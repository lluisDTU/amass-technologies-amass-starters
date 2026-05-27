---
name: amass-trust-filtered-rag
description: Use when building a trust-filtered biomedical RAG with an MCP server — two MCP tools (fetch_credible_evidence_by_id + get_credible_paper_with_trials) wrapped as a Next.js chat UI, with JuFo + retraction + citation-count post-filters wired into every fetched evidence record.
license: Apache-2.0
metadata:
  author: amass
  version: "0.3.0"
---

# Trust-Filtered Biomedical RAG (MCP)

For LLM-tooling builders shipping biomedical AI agents — Claude Desktop tool authors, MCP-aware Cursor users, custom-orchestrator developers. The scaffold ships two layers in one repo: an MCP server (`lib/mcp-tools.ts`) exposing two production-shaped tools, plus a Next.js chat UI consuming the same tool handlers in-process so a Lovable user sees the end-to-end MCP flow on first `npm run dev`. The load-bearing Amass primitive is the trust-filter conjunction (JuFo + retraction + citation count) returned in the same response as the cited evidence, plus paper↔trial cross-core linkage — competitor stacks require 2-3 separate API calls and a join layer on the agent side.

---

## Important — Lovable scaffold cleanup

**Before generating the page component**, the AI builder MUST verify there is NO leftover Lovable blank-app placeholder in the final JSX. Specifically, the page root must NOT contain:

- An outer `<div className="flex min-h-screen items-center justify-center">` wrapping the real app content
- `<img data-lovable-blank-page-placeholder="REMOVE_THIS" src="https://cdn.gpteng.co/blank-app-v1.svg" ... />`

If these are present from Lovable's default template, REMOVE them entirely. The real app's root element must be the page's only top-level wrapper. Leaving the placeholder wrapper in place produces malformed JSX (unclosed outer div) and the published deploy renders a generic error page instead of the app.

---

## What you build

The scaffold ships two architecturally distinct layers in one repo:

1. **MCP server in `lib/mcp-tools.ts`** — framework-agnostic Node.js + `@modelcontextprotocol/sdk`. Exposes two tools:
   - `fetch_credible_evidence_by_id(ids[], min_jufo?, allow_retracted?, min_citation_count?)` — resolves PMIDs/DOIs/AMBC_ IDs to canonical, fetches per-paper records, and post-filters on trust signals. Returns filtered evidence records.
   - `get_credible_paper_with_trials(amass_id, include_fulltext?)` — fetches a single paper with `referencesTrialCore` + `citedBy` arrays populated; optionally includes fulltext excerpt. Returns the paper record plus its cross-core trial linkage.

2. **Next.js chat UI** that consumes the same MCP tool handlers in-process via a `/api/mcp-chat` route. The builder can lift the MCP server out into their own agent stack (Claude Desktop, MCP-aware Cursor, custom orchestrator) by pointing `npx @modelcontextprotocol/inspector` or a stdio launcher at `lib/mcp-tools.ts`.

The input surface is a chat textarea accepting PMIDs / DOIs / `AMBC_` IDs as comma-or-newline-separated identifiers, plus chips above the textarea for per-message threshold overrides (`min_jufo`, `allow_retracted`, `min_citation_count`, `include_fulltext`). The empty state offers a Try-sample button. The output is an evidence-card panel per assistant turn, with a "Get with trials" follow-up button on each card and an "MCP tool call" disclosure showing the underlying tool payload as JSON.

---

## API contract — load-bearing, follow exactly

### Endpoints used

- `POST /api/v1/cores/biomedcore/records/lookup` — batch resolve PMID/DOI → `AMBC_`
- `GET /api/v1/cores/biomedcore/records/{amassId}` — fetch paper record with default trust-filter fields (`isRetracted`, `journalQualityJufo`, `citationCount`)
- `GET /api/v1/cores/biomedcore/records/{amassId}?include=referencesTrialCore,citedBy,fulltext` — fetch paper record with cross-core trial linkage and optional fulltext

### Auth + rate limit

`Authorization: Bearer ${AMASS_API_KEY}` on every request. 60 requests / 60 seconds per user+org. On HTTP 429, read the `Retry-After` header and back off exponentially. On 401/403, surface the credential error directly — retry won't fix it.

### Response envelope (CRITICAL)

Every endpoint returns `{ "data": ... }` wrapped. Unwrap before consuming.

For `POST /records/lookup`, each `data[]` element is one of:

```json
{ "input": { "pmid": "..." }, "amassIds": ["AMBC_..."] }
```

or

```json
{ "input": { "pmid": "..." }, "error": { "code": "NOT_FOUND", "message": "Identifier not found" } }
```

The per-item `error` field is a STRUCTURED OBJECT with `code` and `message` — NOT a bare string. Always extract `.message` as a string before rendering. NEVER render the error object directly as a React child (crashes the surface with React error #31).

### Top-level errors

Non-2xx HTTP response body: `{ "error": { "status": ..., "code": ..., "message": ... } }`. Throw with a message including the upstream `code` + `message`. The chat route's `try/catch` surfaces this as a chat-bubble error inline — never let it propagate to the route-level error boundary.

---

## Reference code

### `lib/amass.ts` — server-only API client

Drop this file in unchanged. Framework-agnostic.

```ts
import "server-only";

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

  getBiomedRecord = (
    amassId: string,
    includes: ReadonlyArray<"fulltext" | "referencesTrialCore" | "references" | "citedBy"> = [],
  ) => {
    const qs = includes.length ? `?${includes.map((i) => `include=${i}`).join("&")}` : "";
    return this.req<unknown>("GET", `/api/v1/cores/biomedcore/records/${amassId}${qs}`);
  };
}

let _client: AmassClient | null = null;
export const getAmassClient = () =>
  (_client ??= new AmassClient(process.env.AMASS_API_KEY!));

export { AmassClient };
```

### `lib/mcp-tools.ts` — MCP server with the two tool handlers

The tool handlers are exported as plain `async` functions so the Next.js chat route can import them directly (in-process) AND a stdio launcher can register them with `@modelcontextprotocol/sdk`. Pin `@modelcontextprotocol/sdk` to an exact version (no caret) — the tool-schema serialization shape couples to the SDK's `Server` class.

```ts
import "server-only";
import { z } from "zod";
import { getAmassClient } from "./amass";

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

function classifyId(id: string) {
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

### Server-side chat route

Write the chat route in your framework's preferred server-side shape (TanStack Start `createServerFn` OR Next.js `app/api/mcp-chat/route.ts`). The route accepts a user message + per-message overrides, parses identifiers out of the message, invokes the appropriate MCP tool handler, and returns the assistant response with the resulting evidence cards.

**Critical:** wrap the entire handler body in `try { ... } catch (err) { ... }`. Surface errors as a chat-bubble message, never a route-level crash.

For the stdio launcher (lifting the MCP server out to Claude Desktop, Cursor, etc.), use a separate entry-point file that registers the tools with `@modelcontextprotocol/sdk`'s `Server` class. The tool handlers themselves are the SAME functions imported from `lib/mcp-tools.ts`.

### Idle-timeout helper

```ts
async function* withIdleTimeout<T>(gen: AsyncGenerator<T>, maxIdleMs = 180_000): AsyncGenerator<T> {
  while (true) {
    const timeout = new Promise<{ kind: "timeout" }>((r) =>
      setTimeout(() => r({ kind: "timeout" }), maxIdleMs),
    );
    const next = gen.next().then((r) => ({ kind: "next" as const, r }));
    const result = await Promise.race([next, timeout]);
    if (result.kind === "timeout")
      throw new Error(`Stream stalled for ${Math.round(maxIdleMs / 1000)}s — an Amass call likely hung.`);
    if (result.r.done) return;
    yield result.r.value;
  }
}
```

---

## Worked example

Use this verified PMID in the empty-state Try-sample button. Render ALL trust-filter values from the live Amass response — NEVER hardcode them.

- **Sample query text:** `Fetch credible evidence for PMID 37861218`
- **PMID 37861218** (Ahn MJ et al., NEJM 2023, Tarlatamab DeLLphi-301 primary publication) — expected `isRetracted=false`, JuFo tier 3 (NEJM), `referencesTrialCore` populated with `AMTC_` IDs

Try-sample tooltip: "Ahn MJ et al. NEJM 2023 — tarlatamab pivotal trial in extensive-stage SCLC. Tarlatamab worked example shape-illustrative; re-verify before binding to a real RAG production workflow."

---

## Per-starter constraints

- The two MCP tools (`fetch_credible_evidence_by_id` + `get_credible_paper_with_trials`) are the public API of this starter. Their Zod input schemas (`FetchCredibleEvidenceInput`, `GetCredibleEvidenceInput`) are the request contract — silent changes break MCP client integrations.
- Pin `@modelcontextprotocol/sdk` to an EXACT version (no caret). The tool-schema serialization shape couples to the SDK's `Server` class; minor-version bumps break in-process wiring.
- Pin `ai` and `@ai-sdk/react` (caret ranges) for the chat-UI streaming layer.
- Evidence card rendering per assistant turn: PMID in monospace + JuFo badge (1-3, ★ icon) + isRetracted chip (rendered red when `isRetracted=true`, hidden when `false`) + citationCount + optional fulltext excerpt (~280 chars when `include_fulltext=true`) + clickable `AMTC_` chips for cross-core trial linkage.
- Each card carries an "MCP tool call" disclosure surfacing the underlying tool payload as formatted JSON — the builder's debugging affordance for lifting the MCP boundary into their own agent stack.

---

## Kit conventions (apply to all Amass starters)

**Visual:**
- External IDs (PMID, DOI, NCT, `AMBC_`, `AMTC_`) render in monospace — IBM Plex Mono. Prose in sans-serif — Inter. Both loaded from Google Fonts (`https://fonts.googleapis.com/css2?family=Inter:wght@400;600;700;800;900&family=IBM+Plex+Mono:wght@400;500;700&display=swap`).
- Neutral, professional palette. shadcn-style design tokens or Tailwind defaults are fine — no marketing colors. Dark mode follows OS `prefers-color-scheme`; no toggle.
- Lucide React for icons.
- Header: wordmark `amass` in Inter weight 800 at top-left.

**UX behavior:**
- Empty state shows a Try-sample button that loads the worked example identifiers verbatim.
- Long-running fan-outs render an analyst-visible Cancel button + progress indicator.
- Wrap fan-out async generators in `withIdleTimeout(gen, 180_000)`; on stall, surface the error inline.
- Per-row error rendering: invalid identifiers show the verbatim upstream error message on that one row. Errors NEVER crash the surface.
- No `AskUserQuestion` to gate scaffolding on credentials — the standard env-var prompt covers `AMASS_API_KEY` at runtime.

**Stack:**
- TypeScript + React + a server-function-capable framework. Lovable defaults to TanStack Start; Next.js App Router is the alternative.
- `lib/amass.ts` and `lib/mcp-tools.ts` carry `import "server-only"` at the top. Never imported from a `"use client"` component.
- `https://api.amass.tech` is hardcoded as the base URL in `lib/amass.ts`. Do NOT introduce an `AMASS_API_BASE_URL` env var.
- Pin React 18.3.1, TypeScript 5.4.5, zod 3.23.8, lucide-react 0.453.0, server-only 0.0.1, `@modelcontextprotocol/sdk` 1.29.0 (exact, no caret), `ai` ^4.0.18, `@ai-sdk/react` ^1.0.7. Add framework packages on top.

**Pre-publish cleanup (DO THIS — Lovable scaffolds may leave these in place):**
- Strip ALL `data-lovable-blank-page-placeholder="REMOVE_THIS"` attributes.
- Strip ALL `<img>` tags pointing at `cdn.gpteng.co`.
- If Lovable left an outer `<div className="flex min-h-screen items-center justify-center">` wrapping the real app + a leftover placeholder image, REMOVE the outer wrapper entirely so the real app's root `<div>` is the only top-level returned element.

---

## Validation

After `npm run dev`, verify in order:

1. **Empty state** renders the chat UI cleanly with the Try-sample button + threshold chips visible. Browser console clean (no errors about `data-lovable-blank-page-placeholder` or unclosed JSX, no `@modelcontextprotocol/sdk` import warnings).
2. **Try sample** click invokes `fetch_credible_evidence_by_id` with PMID 37861218. An evidence card renders below the assistant turn showing the canonical `AMBC_` ID in monospace, JuFo tier 3 badge, no retraction chip (live `isRetracted=false`), citation count, and clickable `AMTC_` chips for the DeLLphi-301 trial.
3. **Get with trials:** click the "Get with trials" button on the evidence card. `get_credible_paper_with_trials` invokes; the trial-side details render inline below the card (NCT05060016 + DeLLphi-301 brief title in monospace).
4. **MCP tool call disclosure:** expand the disclosure on any card. The underlying tool payload renders as formatted JSON, showing the full request + response shape — the debugging affordance for builders.

---

## Hand-off

Build, lint, typecheck must pass. Apply the Pre-publish cleanup steps from Kit conventions above.

**Credentials:** get your `AMASS_API_KEY` from https://platform.amass.tech and add it to `.env`.

To run: `npm run dev`.

**Want to extend it?**
- Wire the MCP server to your own agent (Claude Desktop, Cursor, or a custom orchestrator) via the stdio launcher at `lib/mcp-tools.ts`.
- Add a third MCP tool for retraction-cascade enumeration (combine with the retraction-cascade-monitor starter).
- Wrap the chat UI in a server-sent-events streaming layer for long fan-outs and incremental evidence rendering.

Questions? Join the Amass Developer Community on Discord: https://discord.com/invite/sEGaBHMhWa.

---

## License

Apache-2.0. Copyright 2026 Amass Technologies. Source: https://github.com/lluisDTU/amass-technologies-amass-starters.
