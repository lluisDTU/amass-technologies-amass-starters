---
name: amass-trust-filtered-rag
description: Use when building a biomedical RAG tool with built-in Amass trust filters (JuFo + retraction + citation count) and paper↔trial linkage — an MCP server exposing fetch_credible_evidence_by_id + get_credible_paper_with_trials tools, wrapped as a Next.js chat UI for Lovable-one-shottability.
license: Apache-2.0
metadata:
  author: amass
  version: "0.1.0"
---

# Trust-Filtered Biomedical RAG (Amass MCP)

Build a working trust-filtered biomedical RAG surface on the Amass API. The user is an LLM-tooling builder who just got Amass API credentials and wants a Next.js + TypeScript chat UI exercising a real MCP tool flow running locally in under five minutes.

The scaffold ships two layers in one repo: (1) an **MCP server** in `lib/mcp-tools.ts` exposing two tools — `fetch_credible_evidence_by_id(ids[], min_jufo?, allow_retracted?, min_citation_count?)` returning canonical-AMBC-resolved + trust-filtered evidence; `get_credible_paper_with_trials(amass_id, include_fulltext?)` returning a single paper with cross-core trial linkage — and (2) a **Next.js chat UI** that consumes the same MCP tool handlers in-process so a Lovable user sees the end-to-end MCP flow on first run. The builder can lift the MCP server boundary into their own agent stack (Claude Desktop, MCP-aware Cursor, custom orchestrator) by pointing `npx @modelcontextprotocol/inspector` or a stdio launcher at `lib/mcp-tools.ts`. The worked example binds to PMID 37861218 (Ahn MJ et al. NEJM 2023 — Tarlatamab DeLLphi-301 primary publication). `[Tarlatamab worked example shape-illustrative; PMID 37861218 verified against PubMed + ClinicalTrials.gov + FDA approval letter at draft time; re-verify before binding to a real RAG production workflow]`

---

## Stack — flexible at the framework layer, principle-pinned at the API layer

**TypeScript + React + a server-function-capable framework.** Lovable defaults to **TanStack Start** (`@tanstack/react-router` + `@tanstack/react-start`'s `createServerFn` for server-side handlers). **Next.js (App Router)** with `app/api/<endpoint>/route.ts` POST handlers is the equivalent alternative path. Both work end-to-end against the same `lib/amass.ts` client; pick whichever your AI builder produces by default. The Amass primitives (canonical-ID resolution + per-item-error semantics + cross-core walks + trust filters) are stack-agnostic — the framework only wraps them.

**The MCP-server boundary is architecturally distinct in this prototype.** (i) The MCP server in `lib/mcp-tools.ts` is **framework-agnostic Node.js + `@modelcontextprotocol/sdk`**; its stdio launcher (`node lib/mcp-tools.js`) works identically under both stacks. (ii) Where framework selection bites is the in-process chat route that consumes the same tool handlers: **Next.js App Router** uses `app/api/mcp-chat/route.ts` with `export async function POST(req)`; **TanStack Start** uses `src/lib/mcp-chat.functions.ts` with `createServerFn({ method: "POST" })`. The tool handlers themselves (exported as plain `async` functions from `lib/mcp-tools.ts`) are identical across both — only the route wrapper differs.

What IS pinned (load-bearing):

```json
"react":                     "^18.3.1",
"react-dom":                 "^18.3.1",
"typescript":                "^5.4.5",
"zod":                       "^3.23.8",
"lucide-react":              "^0.453.0",
"server-only":               "^0.0.1",
"ai":                        "^4.0.18",
"@ai-sdk/react":             "^1.0.7",
"@modelcontextprotocol/sdk": "1.29.0"
```

What's framework-specific (one set OR the other, not both):

- **TanStack Start path**: `@tanstack/react-router`, `@tanstack/react-start`, `vite`, `@vitejs/plugin-react`.
- **Next.js (App Router) path**: `next` (latest stable), `@types/node`.

Pin with caret ranges (`^x.y.z`) for compatible patch updates. `zod` is pinned because the tool input schemas (`FetchCredibleEvidenceInput` / `GetCredibleEvidenceInput`) are both the MCP tool-advertisement contract and the chat-route validation contract — silent zod-version drift in default-value or refine semantics shifts what the tools accept. `server-only` is pinned because it is the build-time enforcement Hard Rule #2 depends on (works identically in Next.js and TanStack Start). `ai` + `@ai-sdk/react` are pinned because v0.1 uses them for the chat-UI streaming layer above the MCP tools. **Pin `@modelcontextprotocol/sdk` exactly (no caret)** because the tool-schema serialization shape couples to the SDK's `Server` class + `setRequestHandler` shape; any minor-version bump can break the in-process chat-route wiring.

After scaffolding, run `npm install`. If any package fails to resolve, check `node_modules/<pkg>/package.json` for the latest stable version and update accordingly.

**Three notes for AI builders that may produce either path.** (1) The route file location differs: TanStack Start uses `src/lib/<name>.functions.ts` with `createServerFn({ method: "POST" }).inputValidator(...).handler(...)`; Next.js App Router uses `app/api/<endpoint>/route.ts` with `export async function POST(req: Request)`. The reference snippet below is shown in Next.js shape — if your builder produces TanStack Start, translate the `POST(req)` handler in `app/api/mcp-chat/route.ts` to `createServerFn(...).handler(async ({ data }) => ...)` in `src/lib/mcp-chat.functions.ts` and the `req.json()` body parse to `.inputValidator((input) => FetchCredibleEvidenceInput.parse(input))`. (2) The page file location differs: TanStack Router uses `src/routes/index.tsx` with `createFileRoute("/")`; Next.js App Router uses `app/page.tsx`. Either way, the page calls into the server function / route handler the same way at the network level. (3) **The outer `try/catch` wrapping the handler body MUST be preserved when translating to TanStack Start.** The Next.js reference snippet wraps the chat logic in `try { ... } catch (err) { return Response.json({error: ...}, {status: 500}) }`. For TanStack Start, wrap the `.handler()` body in `try { ... } catch (err) { throw new Error(err instanceof Error ? err.message : "Chat call failed"); }` — the throw surfaces as `mutation.error.message` on the client (TanStack Query's `useMutation` `isError` path) and renders in the chat surface's inline error region. Without this wrap, an uncaught throw from any API call propagates to TanStack Router's route-level error boundary and crashes the whole page instead of surfacing as a chat error. Load-bearing for the kit's "per-item error renders inline without crashing the surface" claim.

---

## What the app should do

The primary builder-visible payoff is a **Next.js chat UI demonstrating the MCP-tool flow end-to-end** — the user pastes a PMID/DOI/AMBC list (or types a natural-language follow-up), the chat surface invokes `fetch_credible_evidence_by_id` against the MCP server's in-process handlers, and the resulting trust-filtered evidence cards render inline below the assistant turn. The load-bearing Amass primitive demonstrated is the **trust-filter conjunction in the same response as the cited evidence** (JuFo journal-quality + retraction flag + citation-count post-filters on a canonical-ID-resolved record set, plus the paper↔trial cross-core walk) — the residual that distinguishes this RAG surface from OpenAlex-MCP and other biomedical MCP layers.

The input surface is a **chat textarea accepting PMIDs / DOIs / `AMBC_` IDs as comma-or-newline-separated identifiers**, plus a toggle for `include_fulltext` and per-message threshold overrides (`min_jufo`, `allow_retracted`, `min_citation_count`) exposed as small chips above the textarea. The empty-state "Try sample" loads a small illustrative biomedical query verbatim: `Fetch credible evidence for PMID 37861218` (Tarlatamab DeLLphi-301 primary publication). The tooltip on the Try-sample button reads "Ahn MJ et al. NEJM 2023 — tarlatamab pivotal trial in extensive-stage SCLC. `[Tarlatamab worked example shape-illustrative; PMID 37861218 verified against PubMed + ClinicalTrials.gov + FDA approval letter at draft time; re-verify before binding to a real RAG production workflow]`".

The chat flow per turn: the user message triggers `fetch_credible_evidence_by_id` on the in-process MCP handler — Call 1 `batchLookupBiomed` (resolve PMIDs / DOIs to canonical `AMBC_`) + Call 2 per-paper `getBiomedRecord` fan-out (read `isRetracted` + `journalQualityJufo` + `citationCount` from the returned record) + client-side trust-filter post-filter — returning filtered evidence records that the assistant turn renders as evidence cards. A follow-up "Get with trials" button on each card triggers `get_credible_paper_with_trials(amass_id, include_fulltext: true)` which returns the full record with `referencesTrialCore` + `citedBy` (+ `fulltext` if toggled) populated for cross-core trial linkage rendering. The `getBiomedRecord` fan-out wraps in `withIdleTimeout(walkPapers(), 180_000)` per Hard Rule #7 with a chat-message-scoped **Cancel** button next to the assistant's pending indicator while the fan-out is in flight; a stalled Amass call past 180s surfaces a "stream stalled — an Amass call likely hung" message bubble rather than locking the chat surface indefinitely.

The final state per assistant turn is a **rendered evidence-card panel** below the assistant message — one card per paper showing PMID (IBM Plex Mono) + JuFo badge (1-3, ★ icon) + isRetracted badge (rendered from the live Amass record; empty when `isRetracted = false`, red `Retracted` chip when `true`) + `citationCount` value + optional `fulltext` excerpt (first ~280 chars when `include_fulltext = true`) + cross-core `referencesTrialCore` chips (clickable `AMTC_` IDs that re-issue `get_credible_paper_with_trials` to render the trial-side details inline). Each card carries an "MCP tool call" disclosure that surfaces the underlying `fetch_credible_evidence_by_id` / `get_credible_paper_with_trials` payload as JSON — the builder's debugging affordance for lifting the MCP boundary into their own agent stack. Render PMIDs / DOIs / NCTs / `AMBC_` / `AMTC_` IDs in IBM Plex Mono; prose in Inter (Brand reference). Use Lucide React for icons. Lovable produces the chat-UI shape from this spec — variability across builders is fine; the load-bearing constraint is that the four paragraphs above demonstrate the MCP-tool flow, the trust-filter post-filter discipline, the cross-core trial linkage, and the worked-example anchor.

---

## Hard rules

The 8 Amass-universal rules below are inherited verbatim across all 6 prototype SKILL.mds. Per-prototype rules 9-12 cover the load-bearing UX / error / discipline surfaces specific to this prototype's MCP-server-wrapped-as-Next.js-chat architecture.

1. **Credentials never enter committed code.** Generate `.env` and `.gitignore` it. Only ONE secret is asked of the user at hand-off:
   ```env
   AMASS_API_KEY=your_amass_api_key_here
   ```
   Per Amass API conventions: the key reads from `AMASS_API_KEY`. The base URL is **hardcoded** to `https://api.amass.tech` in `lib/amass.ts` (single production base; no per-user variance). To point at staging or a local proxy for development, edit the `AMASS_API_BASE_URL` constant in `lib/amass.ts` directly — do NOT re-introduce an env var for this (adding `process.env.AMASS_API_BASE_URL` to `lib/amass.ts` causes Lovable's secrets-UX to prompt the user for a value they don't have, which is unnecessary friction). Never hardcode the API key, never log it, never commit it.

2. **Server-only `AmassClient`.** Lives in `lib/amass.ts` with `import "server-only"` at the top. Never imported from a `"use client"` component. The browser only ever speaks to `/api/<endpoint>`. The Bearer token must never reach the browser bundle; the `server-only` package enforces this at build time. The MCP tool handlers in `lib/mcp-tools.ts` also import `lib/amass.ts` server-side only — when the MCP server is launched as a stdio process for an external MCP client, it runs in a Node.js server context, never a browser bundle.

3. **Singleton/cached `AmassClient`.** Created once on server boot via `getAmassClient()` factory, cached via `clientPromise ??=` pattern. Per-request instantiation is wasted work; the Bearer header is computed once at module-import time and reused. The same singleton serves both the Next.js chat route AND the MCP server's stdio invocation — no double-wrapping.

4. **`Authorization: Bearer ${AMASS_API_KEY}` on every Amass request.** On `401` / `403`, surface the credential error directly — retry won't fix it. On `429`, read `Retry-After` and back off exponentially (per the `lib/amass.ts` snippet below). Rate limit is **60 requests / 60 seconds, per user+org** — the per-call retry helper IS the rate-limit defence for agentic-loop fan-out workflows.

5. **Read from the `data` key; handle errors at top-level AND per-item.** Two distinct error shapes: top-level (any non-2xx) — `{"error": {"status": ..., "code": ..., "message": ...}}` — caught by the route's outer `try/catch` AND by the MCP tool handler's outer `try/catch`. Per-item (`POST /records/lookup` returns HTTP 200 with body `{"data": [...]}` — the universal envelope applies here too) — each array element echoes the input alongside the result: `{"input": {...}, "amassIds": ["AMBC_..."]}` for success, or `{"input": {...}, "error": {"code": "NOT_FOUND", "message": "Identifier not found"}}` for failure (error is a structured object with `code` + `message` fields, NOT a bare string — empirically verified via direct curl on 2026-05-26). The `lib/amass.ts` `batchLookupBiomed` / `batchLookupTrial` methods unwrap the `data` envelope; the `mapLookupResult` helper extracts `error.message` to a string for caller-side rendering, preserving positional index alignment with the input items. **Never assume HTTP 200 means every item succeeded. Never render `error` as a React child directly — extract `.message` first or React #31 crashes the surface.**

6. **Lookup endpoints translate PMID/DOI/NCT → canonical `AMBC_`/`AMTC_` before fetching by ID.** Raw HTTP `GET /records/{id}` accepts **only** canonical Amass IDs — passing a PMID directly returns 404. The workflow is always: `POST /records/lookup` with `[{pmid: "..."}]` → read `amassIds[0]` (handling per-item-error per Rule #5) → `GET /records/{amassId}`. The `lib/amass.ts` `resolvePmid` / `resolveDoi` / `resolveNct` helpers encode this discipline. `fetch_credible_evidence_by_id` accepts mixed input — PMIDs, DOIs, and `AMBC_` IDs in the same `ids[]` array — and routes each to the correct lookup / get path inline.

7. **Idle-timeout + chat-visible cancel button on long-running fan-outs.** Three requirements: (a) the chat-route + MCP-tool handler wrap any fan-out async generator in `withIdleTimeout(gen, 180_000)` — if no chunk arrives for 180s, the wrapper throws; (b) the chat UI renders a **Cancel** button next to the pending-assistant-message indicator while a fan-out is in flight; (c) the route + MCP tool handler wrap body parse + client setup in an outer `try/catch` returning a structured error rather than crashing the chat surface. Without all three, a single stalled Amass call locks the chat UI indefinitely.

8. **Ground-truth discipline.** Every URL, type, function name, version, and config shape in this SKILL.md must come from a source the agent can read — a fetched doc page, a `.d.ts` in `node_modules/@amass/sdk/`, or the installed package's exports. If you can't point to a source, don't write it. Forbidden: fabricating endpoints; guessing field names; `as unknown as ...` or `as any` on `@amass/sdk` or `@modelcontextprotocol/sdk` values; ignoring `tsc` errors. When the runtime contradicts what's written here, fix the cause.

### Per-prototype hard rules

9. **MCP-server-wrapped-as-Next.js-chat architecture.** The MCP server is defined in `lib/mcp-tools.ts` using `@modelcontextprotocol/sdk` and exposes two tools (`fetch_credible_evidence_by_id` + `get_credible_paper_with_trials`) to any MCP-aware client via stdio. The Next.js chat UI in `app/page.tsx` + `app/api/mcp-chat/route.ts` instantiates the same in-process tool handlers (exported as plain `async` functions from `lib/mcp-tools.ts` alongside the MCP `Server` wiring) and routes the chat flow through them — so the Lovable user sees the end-to-end MCP tool flow on first run without configuring an external MCP client. Builders who want to lift the MCP boundary into their own agent stack point `npx @modelcontextprotocol/inspector node lib/mcp-tools.js` at the stdio entry, or configure the file as an MCP server in Claude Desktop / Cursor / a custom orchestrator. Rationale: the two-surface architecture (MCP boundary + Next.js demo) is the load-bearing affordance — Lovable users see the end-to-end MCP flow on first run, AND builders can lift the MCP boundary into a Claude Desktop / MCP-aware Cursor / custom agent stack via `npx @modelcontextprotocol/inspector node lib/mcp-tools.js`.

10. **Trust-filter post-filter discipline.** v0.1 routes around BiomedCore search-driven retrieval — `fetch_credible_evidence_by_id` resolves caller-supplied PMIDs/DOIs/`AMBC_` to canonical AMBC_ via lookup + per-paper GET, then post-filters the returned records on `isRetracted` (server-side default field) + `journalQualityJufo` (client-side post-filter on the returned field) + `citationCount` (client-side post-filter on the returned field — `minCitationCount` is not currently exposed as a server-side filter on BiomedCore search). v1.0 search-driven discovery — adding a `find_credible_evidence(query, ...)` tool with server-side `minJournalQualityJufo` — is deferred until the BiomedCore search surface stabilises around no-token-match fallback behaviour and JuFo filter semantics. Until then, the lookup-and-retrieve v0.1 is the load-bearing surface. Rationale: the v0.1's load-bearing residual is the trust-filter conjunction in the same response as the cited evidence — surfacing the trust signals as post-filters on lookup-and-retrieve preserves the residual.

11. **`include=fulltext` cost-discipline.** `include=fulltext` is opt-in only — the chat-UI toggle defaults to `false` and the `get_credible_paper_with_trials` tool's `include_fulltext` parameter defaults to `false`. Per `CLAUDE.md` API conventions: "use `include=fulltext` only when actually needed; responses balloon otherwise." The chat-UI toggle carries an inline tooltip surfacing the LLM token-cost tradeoff verbatim: "Fulltext responses balloon many KB per paper — token-cost on the downstream LLM scales accordingly. Toggle on only when grounding requires fulltext span citation, not just abstract + metadata." Rationale: LLM-tooling builders pay OpenAI/Anthropic API credits per call — a fulltext-by-default discipline silently 10×s their per-call cost without delivering proportional value when abstract + cross-core trial linkage already supports the answer.

12. **Worked-example anchoring (shape-illustrative; defamation-adjacent guard preserved).** The "Try sample" chat prompt binds the Tarlatamab DeLLphi-301 primary publication identifier verbatim — PMID 37861218 (Ahn MJ et al. NEJM 2023; verified `isRetracted = false` at draft time). The `isRetracted` badge in every rendered evidence card populates from the **live Amass record at run time**, never from a hardcoded fixture — defamation-adjacent fixture patterns are explicitly forbidden across all tiers (an unsupported retraction display IS a retraction claim against the named work). Carry the caveat marker `[Tarlatamab worked example shape-illustrative; PMID 37861218 verified against PubMed + ClinicalTrials.gov + FDA approval letter at draft time; re-verify before binding to a real RAG production workflow]` at three anchor sites: the title intro paragraph, the Try-sample tooltip, and the Hand-off verification preamble. Rationale: the worked example is illustrative of the MCP-tool flow rather than a load-bearing public-facing retraction-or-claim-assertion — the live-Amass-binding requirement structurally preserves the defamation-adjacent guard (Lovable scaffolds against the live API, not against fabricated retraction data).

---

## Brand reference

Source of truth for the Amass app look-and-feel. Load before writing `globals.css`.

### Logo

Render the Amass wordmark at the **top-left of the header**, ~24-32px. Use the text `amass` in Inter weight 800 24px as the wordmark. Place it left of the page title.

### Typography

- **Inter** — UI text, labels, headings, body. Weights 400/600/700/800/900.
- **IBM Plex Mono** — numerics, IDs (PMID / DOI / NCT / `AMBC_` / `AMTC_`), codes, monospace inputs. Weights 400/500/700. Never for prose.

Load both from Google Fonts (`https://fonts.googleapis.com/css2?family=Inter:wght@400;600;700;800;900&family=IBM+Plex+Mono:wght@400;500;700&display=swap`).

### CSS tokens

Add to `globals.css`. Dark mode follows the OS via `prefers-color-scheme` — no toggle, no `next-themes`, no class swap.

```css
:root {
  --background: 0 0% 100%;
  --foreground: 0 0% 6%;
  --card: 0 0% 100%;
  --card-foreground: 0 0% 6%;
  --popover: 0 0% 100%;
  --popover-foreground: 0 0% 6%;
  --primary: 0 0% 6%;
  --primary-foreground: 0 0% 100%;
  --secondary: 0 0% 100%;
  --secondary-foreground: 0 0% 6%;
  --accent: 0 0% 98%;
  --accent-foreground: 222 47% 11%;
  --muted: 0 0% 98%;
  --muted-foreground: 0 0% 41%;
  --destructive: 0 84% 60%;
  --destructive-foreground: 0 0% 100%;
  --border: 0 0% 93%;
  --input: 0 0% 90%;
  --ring: 215 16% 47%;
  --radius: 0.5rem;
  --variant-error-bg: 0 100% 97%;
  --variant-error-border: 0 93% 86%;
  --variant-error-text: 0 72% 37%;
  --amass-accent: 215 80% 50%;
  --amass-accent-foreground: 0 0% 100%;
}

@media (prefers-color-scheme: dark) {
  :root {
    --background: 0 0% 6%;
    --foreground: 0 0% 90%;
    --card: 0 0% 10%;
    --card-foreground: 0 0% 90%;
    --popover: 0 0% 10%;
    --popover-foreground: 0 0% 90%;
    --primary: 0 0% 100%;
    --primary-foreground: 0 0% 6%;
    --secondary: 0 0% 16%;
    --secondary-foreground: 0 0% 90%;
    --accent: 0 0% 16%;
    --accent-foreground: 0 0% 90%;
    --muted: 0 0% 16%;
    --muted-foreground: 0 0% 65%;
    --destructive: 0 72% 50%;
    --destructive-foreground: 0 0% 100%;
    --border: 0 0% 25%;
    --input: 0 0% 25%;
    --ring: 0 0% 100%;
    --variant-error-bg: 0 60% 9%;
    --variant-error-border: 0 52% 28%;
    --variant-error-text: 0 68% 82%;
  }
}
```

Reference all colors via `hsl(var(--token))`. No hardcoded hex outside the accent. The accent is an accent only — never a large background.

Use **Lucide React** for icons (Lucide only).

---

## Reference snippets — use these

**`lib/amass.ts`** — singleton `AmassClient` with Bearer-auth, 429 retry/backoff (reads `Retry-After`), per-item-error helper (`mapLookupResult`), and `{data: ...}` envelope unwrap. Per Hard Rules #2, #3, #4, #5, #6.

```ts
import "server-only";

type LookupResult = ReadonlyArray<{
  input?: { pmid?: string; doi?: string; nctId?: string };
  amassIds?: string[];
  error?: { code?: string; message?: string };
}>;

interface AmassClientConfig {
  apiKey: string;
  baseUrl: string;
  maxRetries?: number;
  retryBaseMs?: number;
}

class AmassClient {
  constructor(private readonly cfg: AmassClientConfig) {}

  private async request<T>(method: "GET" | "POST", path: string, body?: unknown): Promise<T> {
    const url = `${this.cfg.baseUrl}${path}`;
    const headers: Record<string, string> = { Authorization: `Bearer ${this.cfg.apiKey}` };
    if (body !== undefined) headers["Content-Type"] = "application/json";
    const maxRetries = this.cfg.maxRetries ?? 3;
    const baseMs = this.cfg.retryBaseMs ?? 1000;

    for (let attempt = 0; attempt <= maxRetries; attempt++) {
      const res = await fetch(url, {
        method, headers,
        body: body === undefined ? undefined : JSON.stringify(body),
      });
      if (res.status === 429 && attempt < maxRetries) {
        const retryAfter = Number(res.headers.get("Retry-After") ?? "1");
        await new Promise((r) => setTimeout(r, Math.max(retryAfter * 1000, baseMs * 2 ** attempt)));
        continue;
      }
      if (!res.ok) {
        const p = await res.json().catch(() => ({})) as { error?: { code?: string; message?: string } };
        throw new Error(`Amass ${method} ${path} → ${res.status} ${p.error?.code ?? "UNKNOWN"}: ${p.error?.message ?? res.statusText}`);
      }
      return res.json() as Promise<T>;
    }
    throw new Error(`Amass ${method} ${path} — retries exhausted`);
  }

  private async unwrapData<T>(p: Promise<{ data: T }>): Promise<T> {
    return (await p).data;
  }

  // Per-item-error helper. Per empirical observation, each
  // result is either {input?, amassIds: [...]} or {input?, error: {code, message}}.
  // mapLookupResult extracts error.message to a string for the caller's onError handler.
  static mapLookupResult<I, O>(
    inputs: I[],
    results: LookupResult,
    onSuccess: (input: I, amassIds: string[]) => O,
    onError: (input: I, message: string) => O,
  ): O[] {
    return inputs.map((input, i) => {
      const r = results[i];
      return r?.amassIds && r.amassIds.length > 0
        ? onSuccess(input, r.amassIds)
        : onError(input, r?.error?.message ?? "no result");
    });
  }

  // Lookup: PMID/DOI/NCT → canonical AMBC_/AMTC_ (Hard Rule #6).
  // Response is wrapped in {"data": [...]} per the universal envelope rule — unwrap before
  // returning so callers + mapLookupResult see the array directly.
  async batchLookupBiomed(items: ReadonlyArray<{ pmid?: string; doi?: string }>): Promise<LookupResult> {
    return this.unwrapData(this.request<{ data: LookupResult }>("POST", "/api/v1/cores/biomedcore/records/lookup", { items }));
  }
  async batchLookupTrial(items: ReadonlyArray<{ nctId: string }>): Promise<LookupResult> {
    return this.unwrapData(this.request<{ data: LookupResult }>("POST", "/api/v1/cores/trialcore/records/lookup", { items }));
  }
  async resolvePmid(pmid: string): Promise<string | null> {
    return (await this.batchLookupBiomed([{ pmid }]))[0]?.amassIds?.[0] ?? null;
  }
  async resolveDoi(doi: string): Promise<string | null> {
    return (await this.batchLookupBiomed([{ doi }]))[0]?.amassIds?.[0] ?? null;
  }
  async resolveNct(nctId: string): Promise<string | null> {
    return (await this.batchLookupTrial([{ nctId }]))[0]?.amassIds?.[0] ?? null;
  }

  // RAG default includes — referencesTrialCore for cross-core trial linkage + citedBy
  // as auxiliary provenance. fulltext is opt-in only per Hard Rule #11.
  async getBiomedRecord(
    amassId: string,
    includes: ReadonlyArray<"fulltext" | "authorsMetadata" | "meshIds" | "substanceIds" | "referencesTrialCore" | "references" | "citedBy"> = ["referencesTrialCore", "citedBy"],
  ): Promise<unknown /* bind to @amass/sdk BiomedRecord type once installed */> {
    const qs = includes.length > 0 ? `?${includes.map((i) => `include=${i}`).join("&")}` : "";
    return this.unwrapData(this.request<{ data: unknown }>("GET", `/api/v1/cores/biomedcore/records/${amassId}${qs}`));
  }

  // TrialCore get-by-ID — used to render the trial-side card when the chat UI clicks through an AMTC_
  // chip on a paper's referencesTrialCore.
  async getTrialRecord(
    amassId: string,
    includes: ReadonlyArray<"detailedDescription" | "outcomes" | "referencesBiomedCore"> = [],
  ): Promise<unknown /* bind to @amass/sdk TrialRecord type once installed */> {
    const qs = includes.length > 0 ? `?${includes.map((i) => `include=${i}`).join("&")}` : "";
    return this.unwrapData(this.request<{ data: unknown }>("GET", `/api/v1/cores/trialcore/records/${amassId}${qs}`));
  }
}

let clientPromise: Promise<AmassClient> | null = null;

// Production Amass API base URL. Hardcoded by design — single production base, no per-user variance.
// To point at staging or a local proxy for development, edit this constant directly.
// Do NOT replace this with a process.env lookup — Lovable's secrets-UX will prompt the user for a
// value they don't have, which is unnecessary friction. Per Hard Rule #1.
const AMASS_API_BASE_URL = "https://api.amass.tech";

export function getAmassClient(): Promise<AmassClient> {
  return (clientPromise ??= Promise.resolve(new AmassClient({
    apiKey: process.env.AMASS_API_KEY!,
    baseUrl: AMASS_API_BASE_URL,
  })));
}

export { AmassClient };
```

**`lib/mcp-tools.ts`** — MCP server using `@modelcontextprotocol/sdk` exposing the two v0.1 tools, AND exporting the same handlers as plain `async` functions for in-process consumption by the Next.js chat route. Per Hard Rules #5, #6, #9, #10, #11.

```ts
import "server-only";
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { CallToolRequestSchema, ListToolsRequestSchema } from "@modelcontextprotocol/sdk/types.js";
import { z } from "zod";
import { AmassClient, getAmassClient } from "@/lib/amass";

// Tool input schemas — used both for MCP tool advertisement and for chat-route in-process validation.
export const FetchCredibleEvidenceInput = z.object({
  ids: z.array(z.string()).min(1).describe("PMIDs (numeric strings), DOIs (10.x/...), or canonical Amass AMBC_ IDs."),
  min_jufo: z.number().int().min(0).max(3).optional().describe("Post-filter: minimum journalQualityJufo (0-3). Default 2."),
  allow_retracted: z.boolean().optional().describe("Post-filter: include isRetracted=true records. Default false."),
  min_citation_count: z.number().int().min(0).optional().describe("Post-filter: minimum citationCount. Applied client-side (not exposed as a server-side filter on BiomedCore search)."),
});

export const GetCredibleEvidenceInput = z.object({
  amass_id: z.string().regex(/^AMBC_/, "Must be a canonical AMBC_ ID — resolve PMID/DOI via fetch_credible_evidence_by_id first."),
  include_fulltext: z.boolean().optional().describe("Opt-in fulltext (default false; LLM token-cost balloons when true per Hard Rule #11)."),
});

export type FetchCredibleEvidenceArgs = z.infer<typeof FetchCredibleEvidenceInput>;
export type GetCredibleEvidenceArgs = z.infer<typeof GetCredibleEvidenceInput>;

export interface EvidenceRecord {
  amassId: string;
  pmid: string | null;
  doi: string | null;
  title: string | null;
  journalQualityJufo: number | null;
  isRetracted: boolean | null;
  citationCount: number | null;
  lookup_error: string | null;     // per Hard Rule #5 — verbatim upstream string
  filter_reason: string | null;    // populated when record loaded but filtered out (e.g. "JuFo below threshold")
}

// Classify a raw caller-supplied id into PMID / DOI / AMBC_ branch (Hard Rule #6).
function classifyId(raw: string): { kind: "pmid"; pmid: string } | { kind: "doi"; doi: string } | { kind: "ambc"; amassId: string } | { kind: "unknown"; raw: string } {
  const s = raw.trim();
  if (/^AMBC_/.test(s)) return { kind: "ambc", amassId: s };
  if (/^10\.\d{4,9}\//.test(s)) return { kind: "doi", doi: s };
  if (/^\d+$/.test(s)) return { kind: "pmid", pmid: s };
  return { kind: "unknown", raw: s };
}

// Per Hard Rule #10 — trust-filter applied client-side over the returned record.
function applyTrustFilter(rec: { isRetracted?: boolean | null; journalQualityJufo?: number | null; citationCount?: number | null }, opts: { min_jufo: number; allow_retracted: boolean; min_citation_count: number | undefined }): string | null {
  if (!opts.allow_retracted && rec.isRetracted === true) return "isRetracted=true (allow_retracted=false)";
  if ((rec.journalQualityJufo ?? -1) < opts.min_jufo) return `journalQualityJufo=${rec.journalQualityJufo ?? "null"} below min_jufo=${opts.min_jufo}`;
  if (opts.min_citation_count !== undefined && (rec.citationCount ?? 0) < opts.min_citation_count) return `citationCount=${rec.citationCount ?? "null"} below min_citation_count=${opts.min_citation_count}`;
  return null;
}

// Idle-timeout wrapper per Hard Rule #7. Shared with app/api/mcp-chat/route.ts.
export async function* withIdleTimeout<T>(gen: AsyncGenerator<T>, maxIdleMs = 180_000): AsyncGenerator<T> {
  while (true) {
    const timeout = new Promise<{ kind: "timeout" }>((r) => setTimeout(() => r({ kind: "timeout" }), maxIdleMs));
    const result = await Promise.race([
      gen.next().then((r) => ({ kind: "next" as const, r })),
      timeout,
    ]);
    if (result.kind === "timeout") {
      throw new Error(`Stream stalled for ${Math.round(maxIdleMs / 1000)}s — an Amass call likely hung.`);
    }
    if (result.r.done) return;
    yield result.r.value;
  }
}

// === In-process tool handlers — exported for app/api/mcp-chat/route.ts to consume directly. ===

export async function fetchCredibleEvidenceById(args: FetchCredibleEvidenceArgs): Promise<{ records: EvidenceRecord[]; counts: { resolved: number; lookup_errors: number; filtered_out: number } }> {
  const opts = {
    min_jufo: args.min_jufo ?? 2,
    allow_retracted: args.allow_retracted ?? false,
    min_citation_count: args.min_citation_count,
  };
  const client = await getAmassClient();
  const classified = args.ids.map(classifyId);

  // Call 1 — batch lookup PMIDs + DOIs (Hard Rule #6). AMBC_ IDs skip lookup; unknowns go straight to lookup_error.
  const pmidItems = classified.flatMap((c) => (c.kind === "pmid" ? [{ pmid: c.pmid }] : []));
  const doiItems = classified.flatMap((c) => (c.kind === "doi" ? [{ doi: c.doi }] : []));
  const [pmidResults, doiResults] = await Promise.all([
    pmidItems.length > 0 ? client.batchLookupBiomed(pmidItems) : Promise.resolve([]),
    doiItems.length > 0 ? client.batchLookupBiomed(doiItems) : Promise.resolve([]),
  ]);

  // Stitch lookup outcomes back into the original input order, preserving per-item errors verbatim.
  const stub = (): EvidenceRecord => ({ amassId: "", pmid: null, doi: null, title: null, journalQualityJufo: null, isRetracted: null, citationCount: null, lookup_error: null, filter_reason: null });
  const records: EvidenceRecord[] = [];
  const amassIdsToWalk: string[] = [];
  let pi = 0, di = 0;
  for (const c of classified) {
    const r = stub();
    if (c.kind === "pmid") {
      r.pmid = c.pmid;
      const lr = pmidResults[pi++];
      if (lr?.amassIds?.[0]) { r.amassId = lr.amassIds[0]; amassIdsToWalk.push(lr.amassIds[0]); }
      else r.lookup_error = lr?.error ?? "no result";
    } else if (c.kind === "doi") {
      r.doi = c.doi;
      const lr = doiResults[di++];
      if (lr?.amassIds?.[0]) { r.amassId = lr.amassIds[0]; amassIdsToWalk.push(lr.amassIds[0]); }
      else r.lookup_error = lr?.error ?? "no result";
    } else if (c.kind === "ambc") {
      r.amassId = c.amassId;
      amassIdsToWalk.push(c.amassId);
    } else {
      r.lookup_error = `Unrecognised id format: ${c.raw} (expected PMID / DOI / AMBC_)`;
    }
    records.push(r);
  }

  // Call 2 — per-paper GET fan-out for the resolved AMBC_ set, wrapped in withIdleTimeout per Hard Rule #7.
  // Default includes ["referencesTrialCore", "citedBy"] — fulltext is NOT included here (opt-in only per Hard Rule #11).
  async function* walkPapers(): AsyncGenerator<{ amassId: string; rec: { isRetracted: boolean | null; journalQualityJufo: number | null; citationCount: number | null; title: string | null; pmid: string | null; doi: string | null } | null; err: string | null }> {
    for (const amassId of amassIdsToWalk) {
      try {
        const raw = await client.getBiomedRecord(amassId) as { isRetracted?: boolean | null; journalQualityJufo?: number | null; citationCount?: number | null; title?: string | null; pmid?: string | null; doi?: string | null };
        yield { amassId, rec: { isRetracted: raw.isRetracted ?? null, journalQualityJufo: raw.journalQualityJufo ?? null, citationCount: raw.citationCount ?? null, title: raw.title ?? null, pmid: raw.pmid ?? null, doi: raw.doi ?? null }, err: null };
      } catch (e) {
        yield { amassId, rec: null, err: e instanceof Error ? e.message : "walk failed" };
      }
    }
  }

  for await (const { amassId, rec, err } of withIdleTimeout(walkPapers(), 180_000)) {
    const row = records.find((r) => r.amassId === amassId);
    if (!row) continue;
    if (err) { row.lookup_error = err; continue; }
    row.title = rec!.title;
    row.pmid = row.pmid ?? rec!.pmid;
    row.doi = row.doi ?? rec!.doi;
    row.isRetracted = rec!.isRetracted;
    row.journalQualityJufo = rec!.journalQualityJufo;
    row.citationCount = rec!.citationCount;
    // Per Hard Rule #10 — trust-filter client-side post-filter on the returned record.
    row.filter_reason = applyTrustFilter(rec!, opts);
  }

  const counts = {
    resolved: records.filter((r) => r.amassId && !r.lookup_error).length,
    lookup_errors: records.filter((r) => r.lookup_error).length,
    filtered_out: records.filter((r) => r.filter_reason).length,
  };
  return { records, counts };
}

export async function getCrediblePaperWithTrials(args: GetCredibleEvidenceArgs): Promise<{ record: unknown }> {
  const client = await getAmassClient();
  const includes: Array<"referencesTrialCore" | "citedBy" | "fulltext"> = ["referencesTrialCore", "citedBy"];
  if (args.include_fulltext) includes.push("fulltext");
  const record = await client.getBiomedRecord(args.amass_id, includes);
  return { record };
}

// === MCP server wiring — stdio entry point for external MCP clients (Claude Desktop, MCP Inspector). ===

export function buildMcpServer(): Server {
  const server = new Server({ name: "amass-trust-filtered-rag", version: "0.1.0" }, { capabilities: { tools: {} } });

  server.setRequestHandler(ListToolsRequestSchema, async () => ({
    tools: [
      {
        name: "fetch_credible_evidence_by_id",
        description: "Resolve caller-supplied PMIDs / DOIs / AMBC_ IDs to canonical Amass records, post-filter on JuFo + retraction + citation count, return trust-filtered evidence records.",
        inputSchema: { type: "object", properties: { ids: { type: "array", items: { type: "string" } }, min_jufo: { type: "integer", minimum: 0, maximum: 3 }, allow_retracted: { type: "boolean" }, min_citation_count: { type: "integer", minimum: 0 } }, required: ["ids"] },
      },
      {
        name: "get_credible_paper_with_trials",
        description: "Fetch a single paper by canonical AMBC_ ID with cross-core trial linkage (referencesTrialCore) + citedBy. Set include_fulltext=true to include full body (opt-in; LLM token-cost balloons).",
        inputSchema: { type: "object", properties: { amass_id: { type: "string" }, include_fulltext: { type: "boolean" } }, required: ["amass_id"] },
      },
    ],
  }));

  server.setRequestHandler(CallToolRequestSchema, async (req) => {
    try {
      if (req.params.name === "fetch_credible_evidence_by_id") {
        const args = FetchCredibleEvidenceInput.parse(req.params.arguments);
        const result = await fetchCredibleEvidenceById(args);
        return { content: [{ type: "text", text: JSON.stringify(result, null, 2) }] };
      }
      if (req.params.name === "get_credible_paper_with_trials") {
        const args = GetCredibleEvidenceInput.parse(req.params.arguments);
        const result = await getCrediblePaperWithTrials(args);
        return { content: [{ type: "text", text: JSON.stringify(result, null, 2) }] };
      }
      return { content: [{ type: "text", text: `Unknown tool: ${req.params.name}` }], isError: true };
    } catch (e) {
      return { content: [{ type: "text", text: e instanceof Error ? e.message : "tool call failed" }], isError: true };
    }
  });

  return server;
}

// Stdio launcher — invoke via `node lib/mcp-tools.js` or `npx @modelcontextprotocol/inspector node lib/mcp-tools.js`.
// Lovable's scaffold should add a `scripts.mcp` entry to package.json wiring this up.
export async function runStdio(): Promise<void> {
  const server = buildMcpServer();
  const transport = new StdioServerTransport();
  await server.connect(transport);
}
```

**`app/api/mcp-chat/route.ts`** — Next.js App Router POST handler routing the chat-message flow through the in-process MCP tool handlers exported from `lib/mcp-tools.ts`. Returns a consolidated JSON response containing the assistant text + the tool-call results, ready for the chat UI to render evidence cards. Per Hard Rules #7, #9.

```ts
import { fetchCredibleEvidenceById, getCrediblePaperWithTrials, FetchCredibleEvidenceInput, GetCredibleEvidenceInput, EvidenceRecord } from "@/lib/mcp-tools";

interface ChatRequest {
  // The chat-UI assembles the user turn into a structured tool-call request — Lovable's chat surface
  // owns the LLM-side intent extraction (or routes the user's literal id-paste through fetch_*).
  // v0.1 supports two tool routes directly; an LLM-driven router can be layered above this without
  // changing the route.
  tool: "fetch_credible_evidence_by_id" | "get_credible_paper_with_trials";
  args: unknown;
  user_message: string;
}

export async function POST(req: Request) {
  try {
    const body = (await req.json()) as ChatRequest;

    if (body.tool === "fetch_credible_evidence_by_id") {
      const args = FetchCredibleEvidenceInput.parse(body.args);
      const result = await fetchCredibleEvidenceById(args);
      const assistant_text = renderAssistantTextForFetch(result.records, result.counts);
      return Response.json({ assistant_text, tool: body.tool, tool_args: args, tool_result: result });
    }

    if (body.tool === "get_credible_paper_with_trials") {
      const args = GetCredibleEvidenceInput.parse(body.args);
      const result = await getCrediblePaperWithTrials(args);
      const assistant_text = renderAssistantTextForGet(result.record);
      return Response.json({ assistant_text, tool: body.tool, tool_args: args, tool_result: result });
    }

    return Response.json({ error: `Unknown tool: ${body.tool}` }, { status: 400 });
  } catch (err) {
    return Response.json({ error: err instanceof Error ? err.message : "Chat call failed" }, { status: 500 });
  }
}

function renderAssistantTextForFetch(records: EvidenceRecord[], counts: { resolved: number; lookup_errors: number; filtered_out: number }): string {
  const kept = records.filter((r) => r.amassId && !r.lookup_error && !r.filter_reason);
  if (kept.length === 0) {
    return `Resolved ${counts.resolved} of ${records.length} ids; ${counts.filtered_out} filtered out by trust filters; ${counts.lookup_errors} unresolved. Empty result is a real answer — try lowering min_jufo or set allow_retracted=true to widen the corpus.`;
  }
  return `Found ${kept.length} credible record(s) (${counts.resolved} resolved, ${counts.filtered_out} filtered by trust filters, ${counts.lookup_errors} unresolved). Evidence cards below.`;
}

function renderAssistantTextForGet(record: unknown): string {
  const r = record as { title?: string | null; pmid?: string | null; referencesTrialCore?: string[] };
  const trials = r.referencesTrialCore ?? [];
  if (trials.length === 0) {
    return `Loaded ${r.pmid ? `PMID ${r.pmid}` : "record"}: ${r.title ?? "(no title)"} — no cross-core trial linkage on this record (review article or paper without trial references).`;
  }
  return `Loaded ${r.pmid ? `PMID ${r.pmid}` : "record"}: ${r.title ?? "(no title)"} — ${trials.length} cross-core trial linkage(s).`;
}
```

The three snippets above are the only `lib/<file>.ts` + `app/api/<route>/route.ts` mandated by this prototype. v0.1 ships the MCP server stdio entry + the Next.js chat surface in one repo. Download / copy-to-clipboard for evidence cards happens client-side from the chat response payload. If a downstream extension needs persistence (chat-history append-only, ack-event lifecycle for retracted-evidence flagging, per-session conversation memory), the extension SKILL.md includes its own minimal stub per the template's policy — never an `import` from `@/lib/<x>` without a matching snippet.

---

## Required reading

- `https://platform.amass.tech` — Amass Platform docs + API credentials
- `https://github.com/amass-technologies/public-amass-platform-starter-ts` — official Amass Platform Starter (Apache-2.0; Bun + Ink CLI REPL with BiomedCore + TrialCore tools wired up). The engineer-facing complement to this web-app kit.
- `https://modelcontextprotocol.io/docs` — Model Context Protocol spec (tool definition shape, `Server` class, `StdioServerTransport`, `ListToolsRequestSchema` + `CallToolRequestSchema` request handlers); load-bearing for this prototype's MCP server boundary.

---

## Hand-off

Build, lint, and typecheck must pass.

**Credentials.** Get your `AMASS_API_KEY` from https://platform.amass.tech and add it to `.env`. Do NOT use `AskUserQuestion` to gate scaffolding on credentials — Lovable's (or any AI builder's) standard env-var prompt covers this at run time. Just scaffold the app with placeholder `.env`, then run `npm run dev` once the key is in place.

Then present the hand-off summary. The four verification steps bind to the Tarlatamab DeLLphi-301 worked example `[Tarlatamab worked example shape-illustrative; PMID 37861218 verified against PubMed + ClinicalTrials.gov + FDA approval letter at draft time; re-verify before binding to a real RAG production workflow]`:

> **Your Trust-Filtered Biomedical RAG (Amass MCP) is ready.**
>
> Before first push (if your AI builder left scaffolding placeholders): remove any `data-lovable-blank-page-placeholder="REMOVE_THIS"` attributes, any `<img>` tags pointing at `cdn.gpteng.co` or similar builder-placeholder CDNs, and any leftover "your app will live here" boilerplate. These render invisibly but pollute the DOM and serve no purpose in production. Search the codebase for `REMOVE_THIS` and `gpteng` before committing.
>
> To run it: fill in `.env` and run `npm run dev`.
>
> Verify it works:
> - In the chat textarea, click **Try sample** to load `Fetch credible evidence for PMID 37861218`; press Enter. Confirm `fetch_credible_evidence_by_id` returns one evidence card showing the Ahn MJ et al. NEJM 2023 record — title rendered in Inter; PMID 37861218 in IBM Plex Mono; JuFo badge = 3 (NEJM is a JuFo tier-3 journal); `isRetracted` badge empty (live value from Amass — not hardcoded fixture per Hard Rule #12); `citationCount` populated.
> - Click the **Get with trials** button on the resolved evidence card; confirm `get_credible_paper_with_trials` returns the full record with `referencesTrialCore` populated including the DeLLphi-301 `AMTC_` chip (the cross-core trial-linkage primitive that distinguishes this RAG surface from MCP layers without first-class trial entities).
> - Toggle the `include_fulltext` chip above the chat textarea to **on**, then re-issue **Get with trials** on the same card; confirm the evidence card surfaces a fulltext excerpt (first ~280 chars) — OR an inline "fulltext unavailable" note if Amass does not hold fulltext for this PMID (per the `hasFulltext` field semantics). The tooltip on the toggle should warn about LLM token-cost ballooning per Hard Rule #11.
> - Run an MCP-inspector probe against the stdio server in a separate terminal: `npx @modelcontextprotocol/inspector node lib/mcp-tools.js` (after the project compiles). Confirm both tools — `fetch_credible_evidence_by_id` + `get_credible_paper_with_trials` — enumerate with the JSON-schema shape declared in the `ListToolsRequestSchema` handler. Then call `fetch_credible_evidence_by_id` from the inspector with `{ids: ["37861218"]}`; confirm the result mirrors the in-chat result (same canonical AMBC_ + same trust-filter outcome) — verifies the in-process tool handler is identical to the MCP-boundary tool handler per Hard Rule #9.
>
> Want to extend it?
> - **v1.0 search-driven RAG.** Add a `find_credible_evidence(query, min_jufo, allow_retracted, ...)` MCP tool once the BiomedCore search surface stabilises around no-token-match fallback behaviour and JuFo filter semantics. Until then, the lookup-and-retrieve v0.1 is the load-bearing surface.
> - **Lift the MCP boundary into a Claude Desktop / Cursor / custom agent stack.** Configure `lib/mcp-tools.ts` as an MCP server entry in the host agent's MCP configuration (`{"mcpServers": {"amass-trust-filtered-rag": {"command": "node", "args": ["path/to/lib/mcp-tools.js"], "env": {"AMASS_API_KEY": "..."}}}}`) — the same two tools become callable from the host LLM's tool-use loop without the Next.js chat layer.
> - **Extend `fetch_credible_evidence_by_id` to support NCT alongside PMID + DOI + AMBC_.** Pass NCTs through `client.batchLookupTrial` → resolve to `AMTC_` → optionally `getTrialRecord` with `include=referencesBiomedCore` to walk trial→paper for grounding evidence on a trial-led question. The reverse direction (trial → paper) is foreclosed for v0.1 because at-scale symmetry of the cross-core spine is not yet a published guarantee — extending here requires the at-scale-symmetry caveat to surface in the chat-UI tooltip.
> - I'm done — just show me the summary
>
> Have questions? Join the Amass Developer Community on Discord: https://discord.com/invite/sEGaBHMhWa

---

## License

Apache-2.0. Copyright 2026 Amass Technologies. Source at https://github.com/lluisDTU/amass-technologies-amass-starters.
