---
name: amass-trust-filtered-rag
description: Use when building a trust-filtered biomedical RAG with an MCP server — two MCP tools (fetch_credible_evidence_by_id + get_credible_paper_with_trials) wrapped as a TanStack Start chat UI, with JuFo + retraction + citation-count post-filters wired into every fetched evidence record.
license: Apache-2.0
metadata:
  author: amass
  version: "0.4.2"
---

# Trust-Filtered Biomedical RAG (MCP)

For LLM-tooling builders shipping biomedical AI agents — Claude Desktop tool authors, MCP-aware Cursor users, custom-orchestrator developers. The scaffold ships two layers in one repo: an MCP server (`lib/mcp-tools.server.ts`) exposing two production-shaped tools, plus a TanStack Start chat UI consuming the same tool handlers in-process so a Lovable user sees the end-to-end MCP flow on first `npm run dev`. The load-bearing Amass primitive is the trust-filter conjunction (JuFo + retraction + citation count) returned in the same response as the cited evidence, plus paper↔trial cross-core linkage — competitor stacks require 2-3 separate API calls and a join layer on the agent side.

---

## Scaffold target — read FIRST

This starter is built on **TanStack Start + Vite + TanStack Router + TanStack Query** (Lovable's default). FOUR TypeScript files (one more than other starters because the MCP server is architecturally distinct); do NOT wrap them in any outer template or placeholder component.

For Next.js App Router users, see the Appendix.

### File 1: `src/lib/amass.server.ts` — server-only API client

```ts
// src/lib/amass.server.ts
// Server-only via .server.ts filename. DO NOT add `import "server-only"`.

export type LookupResult = ReadonlyArray<{
  input?: { pmid?: string; doi?: string; nctId?: string };
  amassIds?: string[];
  error?: { code?: string; message?: string };
}>;

class AmassClient {
  constructor(private apiKey: string, private baseUrl = "https://api.amass.tech") {}

  private async req<T>(method: "GET" | "POST", path: string, body?: unknown): Promise<T> {
    const headers: Record<string, string> = { Authorization: `Bearer ${this.apiKey}` };
    if (body !== undefined) headers["Content-Type"] = "application/json";

    for (let attempt = 0; attempt <= 3; attempt++) {
      const res = await fetch(`${this.baseUrl}${path}`, {
        method,
        headers,
        body: body === undefined ? undefined : JSON.stringify(body),
      });

      if (res.status === 429 && attempt < 3) {
        const retry = Number(res.headers.get("Retry-After") ?? "1");
        await new Promise((r) => setTimeout(r, Math.max(retry * 1000, 1000 * 2 ** attempt)));
        continue;
      }

      if (!res.ok) {
        const p = (await res.json().catch(() => ({}))) as {
          error?: { code?: string; message?: string };
        };
        throw new Error(
          `Amass ${method} ${path} → ${res.status} ${p.error?.code ?? "UNKNOWN"}: ${p.error?.message ?? res.statusText}`,
        );
      }

      const { data } = (await res.json()) as { data: T };
      return data;
    }
    throw new Error(`Amass ${method} ${path} — retries exhausted`);
  }

  batchLookupBiomed = (items: ReadonlyArray<{ pmid?: string; doi?: string }>) =>
    this.req<LookupResult>("POST", "/api/v1/cores/biomedcore/records/lookup", { items });

  getBiomedRecord = (amassId: string, includes: ReadonlyArray<string> = []) => {
    const qs = includes.length ? `?${includes.map((i) => `include=${i}`).join("&")}` : "";
    return this.req<unknown>("GET", `/api/v1/cores/biomedcore/records/${amassId}${qs}`);
  };
}

let _client: AmassClient | null = null;
export const getAmassClient = () =>
  (_client ??= new AmassClient(process.env.AMASS_API_KEY!));

export { AmassClient };
```

### File 2: `src/lib/mcp-tools.server.ts` — MCP server with the two tool handlers

The tool handlers are exported as plain `async` functions so the chat route can import them directly (in-process) AND a stdio launcher can register them with `@modelcontextprotocol/sdk`. Pin `@modelcontextprotocol/sdk` to an EXACT version (no caret) — the tool-schema serialization shape couples to the SDK's `Server` class.

```ts
// src/lib/mcp-tools.server.ts
// Server-only via .server.ts filename. DO NOT add `import "server-only"`.
import { z } from "zod";
import { getAmassClient } from "./amass.server";

export const FetchCredibleEvidenceInput = z.object({
  ids: z.array(z.string()).min(1).max(50),
  min_jufo: z.number().int().min(1).max(3).optional().default(1),
  allow_retracted: z.boolean().optional().default(false),
  min_citation_count: z.number().int().min(0).optional().default(0),
});

export const GetCredibleEvidenceInput = z.object({
  amass_id: z.string().regex(/^AMBC_/),
  include_fulltext: z.boolean().optional().default(false),
});

function classifyId(id: string): { amassId?: string; doi?: string; pmid?: string } {
  if (/^AMBC_/.test(id)) return { amassId: id };
  if (/^10\./.test(id)) return { doi: id };
  return { pmid: id };
}

export async function fetch_credible_evidence_by_id(
  args: z.infer<typeof FetchCredibleEvidenceInput>,
) {
  const client = getAmassClient();
  const classified = args.ids.map(classifyId);

  const toLookup = classified
    .map((c, i) => ({ ...c, index: i }))
    .filter((c) => !c.amassId);
  const lookupResults = toLookup.length
    ? await client.batchLookupBiomed(toLookup.map(({ pmid, doi }) => ({ pmid, doi })))
    : [];

  const amassIds: string[] = classified.map((c, i) => {
    if (c.amassId) return c.amassId;
    const li = toLookup.findIndex((t) => t.index === i);
    return lookupResults[li]?.amassIds?.[0] ?? "";
  });

  const records = await Promise.all(
    amassIds.filter(Boolean).map((id) => client.getBiomedRecord(id)),
  );
  return records.filter((r: any) => {
    if (r.isRetracted && !args.allow_retracted) return false;
    if ((r.journalQualityJufo ?? 0) < args.min_jufo) return false;
    if ((r.citationCount ?? 0) < args.min_citation_count) return false;
    return true;
  });
}

export async function get_credible_paper_with_trials(
  args: z.infer<typeof GetCredibleEvidenceInput>,
) {
  const client = getAmassClient();
  const includes = args.include_fulltext
    ? (["referencesTrialCore", "citedBy", "fulltext"] as const)
    : (["referencesTrialCore", "citedBy"] as const);
  return client.getBiomedRecord(args.amass_id, includes);
}
```

### File 3: `src/lib/mcp-chat.functions.ts` — TanStack Start server function consuming the MCP tools

```ts
// src/lib/mcp-chat.functions.ts
import { createServerFn } from "@tanstack/react-start";
import { z } from "zod";
import {
  fetch_credible_evidence_by_id,
  get_credible_paper_with_trials,
  FetchCredibleEvidenceInput,
  GetCredibleEvidenceInput,
} from "./mcp-tools.server";

const ChatRequestSchema = z.discriminatedUnion("tool", [
  z.object({ tool: z.literal("fetch_credible_evidence_by_id"), args: FetchCredibleEvidenceInput }),
  z.object({ tool: z.literal("get_credible_paper_with_trials"), args: GetCredibleEvidenceInput }),
]);

export const invokeMcpTool = createServerFn({ method: "POST" })
  .inputValidator((input: unknown) => ChatRequestSchema.parse(input))
  .handler(async ({ data }) => {
    try {
      if (data.tool === "fetch_credible_evidence_by_id") {
        const result = await fetch_credible_evidence_by_id(data.args);
        return { tool: data.tool, result };
      }
      const result = await get_credible_paper_with_trials(data.args);
      return { tool: data.tool, result };
    } catch (err) {
      throw new Error(err instanceof Error ? err.message : "MCP tool call failed");
    }
  });
```

### File 4: `src/routes/index.tsx` — TanStack Start route + chat UI

The `component: Index` binding is REQUIRED. Root is `<main>` — no outer wrapper.

```tsx
// src/routes/index.tsx
import { createFileRoute } from "@tanstack/react-router";
import { useState } from "react";
import { useMutation } from "@tanstack/react-query";
import { useServerFn } from "@tanstack/react-start";
import { invokeMcpTool } from "@/lib/mcp-chat.functions";

export const Route = createFileRoute("/")({
  head: () => ({
    meta: [
      { title: "Trust-Filtered Biomedical RAG (MCP) — Amass" },
      { name: "description", content: "Two MCP tools + chat UI: JuFo + retraction + citation-count post-filters on every fetched evidence record." },
    ],
  }),
  component: Index, // ← REQUIRED. Without this, SSR fails.
});

function Index() {
  const [chatInput, setChatInput] = useState("");
  const [minJufo, setMinJufo] = useState(1);
  const [allowRetracted, setAllowRetracted] = useState(false);
  const [minCitationCount, setMinCitationCount] = useState(0);
  const [includeFulltext, setIncludeFulltext] = useState(false);
  const invoke = useServerFn(invokeMcpTool);
  const mutation = useMutation({
    mutationFn: async () => {
      // Parse identifiers from chatInput (PMID/DOI/AMBC_); call fetch_credible_evidence_by_id
      const ids = chatInput.split(/[,\s]+/).filter(Boolean);
      return invoke({ data: { tool: "fetch_credible_evidence_by_id", args: { ids, min_jufo: minJufo, allow_retracted: allowRetracted, min_citation_count: minCitationCount } } });
    },
  });

  return (
    <main className="min-h-screen bg-background text-foreground font-sans">
      <header className="border-b border-border">
        <div className="max-w-6xl mx-auto px-6 py-4 flex items-center justify-between">
          <h1 className="text-2xl font-extrabold tracking-tight">amass</h1>
          <span className="text-xs text-muted-foreground font-mono">trust-filtered-rag · v0.1</span>
        </div>
      </header>

      <section className="max-w-6xl mx-auto px-6 py-10 space-y-8">
        {/* Chat textarea + threshold chips (min_jufo, allow_retracted, min_citation_count, include_fulltext) + Try-sample + Send + Evidence cards per assistant turn (PMID + JuFo badge + isRetracted chip + citationCount + cross-core trial chips + "MCP tool call" disclosure as JSON) + "Get with trials" button on each card */}
      </section>
    </main>
  );
}
```

---

## What you build

The scaffold ships TWO architecturally distinct layers in one repo:

1. **MCP server in `src/lib/mcp-tools.server.ts`** — Node.js + `@modelcontextprotocol/sdk`. Exposes two tools:
   - `fetch_credible_evidence_by_id(ids[], min_jufo?, allow_retracted?, min_citation_count?)` — resolves PMIDs/DOIs/AMBC_ IDs to canonical, fetches per-paper records, post-filters on trust signals.
   - `get_credible_paper_with_trials(amass_id, include_fulltext?)` — fetches paper record with `referencesTrialCore` + `citedBy` arrays.

2. **TanStack Start chat UI** consuming the same handlers in-process via `src/lib/mcp-chat.functions.ts`. Builders can lift the MCP server out into their own agent stack (Claude Desktop, MCP-aware Cursor, custom orchestrator) by pointing `npx @modelcontextprotocol/inspector` or a stdio launcher at `mcp-tools.server.ts`.

Input surface: chat textarea + 4 threshold chips. Empty state: Try-sample loading "Fetch credible evidence for PMID 37861218". Output: evidence-card panel per assistant turn with "Get with trials" follow-up + "MCP tool call" JSON disclosure.

---

## API contract — load-bearing, follow exactly

**Authoritative source:** https://platform.amass.tech/documentation/for-ai-agents/llm-quick-reference.md — the live single-page reference for every endpoint, param, enum, and response schema across BiomedCore + TrialCore. If anything in this SKILL.md drifts from that doc, the doc wins.

### Endpoints used

- `POST /api/v1/cores/biomedcore/records/lookup` — batch resolve PMID/DOI → `AMBC_`
- `GET /api/v1/cores/biomedcore/records/{amassId}` — fetch paper record with default trust-filter fields
- `GET /api/v1/cores/biomedcore/records/{amassId}?include=referencesTrialCore,citedBy,fulltext` — fetch paper record with cross-core trial linkage + optional fulltext

### Critical operational rules (load-bearing — the build fails without them)

- **Auth:** `Authorization: Bearer ${AMASS_API_KEY}` on every request. Get key from https://platform.amass.tech. Rate limit: 60 req / 60 s per user+org. On HTTP 429, read `Retry-After` and back off exponentially. On 401/403, surface the credential error directly.
- **Response envelope:** every endpoint returns `{ "data": ... }` wrapped. Unwrap before consuming.
- **Per-item lookup errors:** each `data[]` element on `POST /records/lookup` is either `{ amassIds: [...] }` (success) OR `{ error: { code, message } }` (per-item failure). The error field is a STRUCTURED OBJECT — empirically verified, even if the live docs example shows the simpler `{ error: "..." }` form. Always extract `.message` as a string before rendering. NEVER render the error object directly as a React child (React #31 crash). Example: `{ "input": { "pmid": "99999999" }, "error": { "code": "NOT_FOUND", "message": "Identifier not found" } }`.
- **Top-level errors:** non-2xx body is `{ "error": { "status", "code", "message" } }`. The reference code's `req<T>` throws with the upstream code+message; the route handler's outer `try/catch` re-throws so TanStack Query surfaces it as `mutation.error.message` on the client. NEVER let an uncaught throw propagate to the route-level error boundary.
- **Lookup before fetch:** `GET /records/{amassId}` accepts ONLY canonical `AMBC_` / `AMTC_` IDs. Passing PMID/DOI/NCT directly returns 404. Always: lookup → resolve canonical → fetch by canonical ID.

---

## Worked example

- **Sample query text:** `Fetch credible evidence for PMID 37861218`
- **PMID 37861218** (Ahn MJ et al., NEJM 2023, Tarlatamab DeLLphi-301 primary publication) — expected `isRetracted=false`, JuFo tier 3 (NEJM), `referencesTrialCore` populated with `AMTC_` IDs

Try-sample tooltip: "Ahn MJ et al. NEJM 2023 — tarlatamab pivotal trial in ES-SCLC. Tarlatamab worked example shape-illustrative; re-verify before binding to a real RAG production workflow."

---

## Per-starter constraints

- Two MCP tools are the public API. Their Zod input schemas (`FetchCredibleEvidenceInput`, `GetCredibleEvidenceInput`) are the request contract.
- Pin `@modelcontextprotocol/sdk` to EXACT version (no caret). Tool-schema serialization couples to SDK's `Server` class.
- Pin `ai` and `@ai-sdk/react` (caret ranges) for chat-UI streaming layer.
- Evidence card per assistant turn: PMID in monospace + JuFo badge (1-3, ★ icon) + isRetracted chip (red when true, hidden when false) + citationCount + optional fulltext excerpt (~280 chars when `include_fulltext=true`) + clickable `AMTC_` chips for cross-core trial linkage.
- Each card carries an "MCP tool call" disclosure surfacing the underlying tool payload as formatted JSON — the debugging affordance for builders lifting the MCP boundary into their own agent stack.

---

## Kit conventions (apply to all Amass starters)

**Visual:**
- External IDs (PMID, DOI, NCT, `AMBC_`, `AMTC_`) in monospace — IBM Plex Mono. Prose in sans-serif — Inter. Both loaded from Google Fonts (`https://fonts.googleapis.com/css2?family=Inter:wght@400;600;700;800;900&family=IBM+Plex+Mono:wght@400;500;700&display=swap`).
- Neutral palette. shadcn-style tokens or Tailwind defaults. Dark mode follows OS `prefers-color-scheme`; no toggle.
- Lucide React for icons.
- Header: wordmark `amass` in Inter weight 800 at top-left.

**UX behavior:**
- Empty state shows a Try-sample button that loads the worked example identifiers verbatim.
- Long-running fan-outs render an analyst-visible Cancel button + progress indicator.
- Wrap fan-out async generators in `withIdleTimeout(gen, 180_000)`.
- Per-row error rendering: invalid identifiers show verbatim upstream error. Errors NEVER crash the surface.
- No `AskUserQuestion` gating on credentials — env-var prompt covers `AMASS_API_KEY` at runtime.

**Stack (TanStack Start + Vite — Lovable's default):**
- Server-only files use `.server.ts` filename (`src/lib/amass.server.ts`, `src/lib/mcp-tools.server.ts`). DO NOT add `import "server-only"`.
- Server functions: `src/lib/<name>.functions.ts` with `createServerFn({ method: "POST" }).inputValidator(...).handler(...)`.
- Route + page: `src/routes/index.tsx` with `createFileRoute("/")({ head, component: Index })`. The `component` binding is REQUIRED.
- `https://api.amass.tech` hardcoded. NO `AMASS_API_BASE_URL` env var.
- Additional pins for this starter: `@modelcontextprotocol/sdk` EXACT version (no caret); `ai` + `@ai-sdk/react` caret ranges.

---

## Validation

After `npm run dev`:

1. **Empty state** renders `<main>` root + amass wordmark + chat textarea + 4 threshold chips + Try-sample. Console clean (no errors about `data-lovable-blank-page-placeholder`, unclosed JSX, `import "server-only"` crashes, or `@modelcontextprotocol/sdk` import warnings).
2. **Try sample** click invokes `fetch_credible_evidence_by_id` with PMID 37861218. Evidence card renders with canonical `AMBC_` ID in monospace, JuFo tier 3 badge, no retraction chip (live `isRetracted=false`), citation count, clickable `AMTC_` chips for DeLLphi-301 trial.
3. **Get with trials:** click button on the evidence card. `get_credible_paper_with_trials` invokes; trial-side details render inline (NCT05060016 + DeLLphi-301 brief title in monospace).
4. **MCP tool call disclosure:** expand the disclosure on any card. Underlying tool payload renders as formatted JSON.

---

## Hand-off

Build, lint, typecheck must pass.

**Credentials:** `AMASS_API_KEY` from https://platform.amass.tech. Run `npm run dev`.

**Want to extend it?**
- Wire the MCP server to your own agent (Claude Desktop, Cursor, custom orchestrator) via stdio launcher at `mcp-tools.server.ts`.
- Add a third MCP tool for retraction-cascade enumeration (combine with the retraction-cascade-monitor starter).
- Wrap the chat UI in a server-sent-events streaming layer for long fan-outs.

Discord: https://discord.com/invite/sEGaBHMhWa.

---

## License

Apache-2.0. Copyright 2026 Amass Technologies. Source: https://github.com/lluisDTU/amass-technologies-amass-starters.

---

## Appendix — Next.js App Router alternative

For Claude Code / Cursor / Next.js production users:

- `lib/amass.ts` — same code as `src/lib/amass.server.ts` above, with `import "server-only"` added at the top.
- `lib/mcp-tools.ts` — same code as `src/lib/mcp-tools.server.ts` above, with `import "server-only"` added at the top.
- `app/api/mcp-chat/route.ts` — standard Next.js POST handler dispatching to the MCP tools.
- `app/page.tsx` — `"use client"` chat page POSTing to `/api/mcp-chat`. Same `<main>` root, no outer wrapper.

Amass API contract, worked example, validation, kit conventions all apply identically.
