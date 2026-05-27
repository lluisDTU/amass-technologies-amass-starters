---
name: amass-retraction-cascade-monitor
description: Use when building a retraction-cascade tracker that re-audits a stored bibliography for newly retracted papers AND surfaces the cascade — inbound (citedBy), outbound (references), and cross-core (referencesTrialCore) — so SR teams, regulatory affairs, and medical-info teams pull contaminated evidence from their dossiers before downstream harm.
license: Apache-2.0
metadata:
  author: amass
  version: "0.1.0"
---

# Retraction Cascade Monitor (Amass BiomedCore + TrialCore)

Build a working retraction-cascade tracker on the Amass API. The user is a systematic-review researcher (Cochrane-style) re-auditing a published review's bibliography for newly-retracted citations, OR a regulatory-affairs analyst monitoring a submission's citation chain, OR a medical-affairs analyst monitoring a published medical-info answer for citation contamination (the three ICPs share the same audit-trail workflow shape — same lookup, same record-fetch, same cascade panels — and a single deployment serves all three). They just got Amass API credentials and want a Next.js + TypeScript cascade dashboard running locally in under five minutes.

The user pastes a curated bibliography PMID list (caller's stored snapshot at time t0); the tool resolves each PMID to a canonical `AMBC_` Amass ID, fetches the per-paper record with `include=references,citedBy,referencesTrialCore` (the retraction-flag default field + both-direction citation graph + cross-core trial spine — all in one record fetch per Hard Rule #10), filters to `isRetracted === true`, and surfaces the three-direction cascade: inbound (`citedBy` — what downstream papers cite the retracted work and now carry contamination), outbound (`references` — what the retracted work itself cited, often revealing the shaky evidence chain), and cross-core (`referencesTrialCore` — what trials the retracted paper describes). The worked example binds to Wakefield et al. Lancet 1998 (PMID 9500320) — the canonical retracted paper for demonstration purposes, ~3,000+ citing papers in the inbound cascade per published bibliometrics. `[identifier-verified per WebFetch against PubMed + Retraction Watch on 2026-05-21; PMID 9500320 = Wakefield et al. Lancet 1998 351(9103):637-41, fully retracted 2010-02-06 (PMID 20137807); partial retraction of interpretation 2004-03-06 (PMID 15016483)]`

---

## Stack — flexible at the framework layer, principle-pinned at the API layer

**TypeScript + React + a server-function-capable framework.** Lovable defaults to **TanStack Start** (`@tanstack/react-router` + `@tanstack/react-start`'s `createServerFn` for server-side handlers). **Next.js (App Router)** with `app/api/<endpoint>/route.ts` POST handlers is the equivalent alternative path. Both work end-to-end against the same `lib/amass.ts` client; pick whichever your AI builder produces by default. The Amass primitives (canonical-ID resolution + per-item-error semantics + cross-core walks + retraction-flag default field) are stack-agnostic — the framework only wraps them.

What IS pinned (load-bearing):

```json
"react":                  "^18.3.1",
"react-dom":              "^18.3.1",
"typescript":             "^5.4.5",
"zod":                    "^3.23.8",
"lucide-react":           "^0.453.0",
"server-only":            "^0.0.1"
```

What's framework-specific (one set OR the other, not both):

- **TanStack Start path**: `@tanstack/react-router`, `@tanstack/react-start`, `vite`, `@vitejs/plugin-react`.
- **Next.js (App Router) path**: `next` (latest stable), `@types/node`.

Pin with caret ranges (`^x.y.z`) for compatible patch updates. `zod` is pinned because the input-validator schema (`CascadeRequestSchema`) is the request contract — silent zod-version drift in default-value or refine semantics shifts what the route accepts. `server-only` is pinned because it is the build-time enforcement Hard Rule #2 depends on (works identically in Next.js and TanStack Start). The per-item-error verbatim preservation (Hard Rule #11) and the three-direction-in-one-call discipline (Hard Rule #10) are load-bearing; do not paraphrase upstream error strings, do not collapse the include set across two HTTP calls.

After scaffolding, run `npm install`. If any package fails to resolve, check `node_modules/<pkg>/package.json` for the latest stable version and update accordingly.

**Three notes for AI builders that may produce either path.** (1) The route file location differs: TanStack Start uses `src/lib/<name>.functions.ts` with `createServerFn({ method: "POST" }).inputValidator(...).handler(...)`; Next.js App Router uses `app/api/<endpoint>/route.ts` with `export async function POST(req: Request)`. The reference snippet below is shown in Next.js shape — if your builder produces TanStack Start, translate the `POST(req)` handler to `createServerFn(...).handler(async ({ data }) => ...)` and the `req.json()` body parse to `.inputValidator((input) => CascadeRequestSchema.parse(input))`. (2) The page file location differs: TanStack Router uses `src/routes/index.tsx` with `createFileRoute("/")`; Next.js App Router uses `app/page.tsx`. Either way, the page calls into the server function / route handler the same way at the network level. (3) **The outer `try/catch` wrapping the handler body MUST be preserved when translating to TanStack Start.** The Next.js reference snippet wraps the cascade logic in `try { ... } catch (err) { return Response.json({error: ...}, {status: 500}) }`. For TanStack Start, wrap the `.handler()` body in `try { ... } catch (err) { throw new Error(err instanceof Error ? err.message : "Cascade audit failed"); }` — the throw surfaces as `mutation.error.message` on the client (TanStack Query's `useMutation` `isError` path) and renders in the UI's inline error region. Without this wrap, an uncaught throw from any API call propagates to TanStack Router's route-level error boundary and crashes the whole page instead of surfacing as a mutation error. Load-bearing for the kit's "per-item error renders inline without crashing the surface" claim.

---

## What the app should do

The primary analyst-visible payoff is a **3-panel cascade dashboard** — retracted-papers-in-bibliography (left), inbound `citedBy` cascade (center), outbound `references` + cross-core `referencesTrialCore` cascade (right). The load-bearing Amass primitive is the **three-direction-in-one-record-fetch conjunction** per Hard Rule #10: a single `GET /api/v1/cores/biomedcore/records/{amassId}?include=references,citedBy,referencesTrialCore` delivers the retraction flag (`isRetracted` is a default field — no include flag needed), the both-direction citation graph (`references` outbound + `citedBy` inbound), and the cross-core trial spine (`referencesTrialCore`) — competitors do not expose this conjunction in a single call (OpenAlex requires a second HTTP call for the cited-by direction; OpenAlex has no clinical-trial entity; Scite has no trial linkage and single-direction retraction). The retracted-papers panel renders the `isRetracted` flag, journal, and the count of downstream-contaminated papers (inbound cascade cardinality) per retracted paper.

The input surface is a **stored-bibliography PMID list textarea** — a multi-line text input accepting one PMID per line (typical SR cohort size: 100–500 PMIDs; v0.1 ceiling per Hard Rule #9 below). The empty-state "Try sample" loads the W2-verified Wakefield example verbatim — `9500320` (Wakefield Lancet 1998, full retraction 2010-02-06) plus ~10 context PMIDs from the autism-vaccination methodology literature so the cascade panels demonstrate honest cardinality (the inbound `citedBy` panel populates with the live-from-Amass list of papers that cite Wakefield; the outbound `references` panel populates with what Wakefield itself cited). The tooltip on the Try-sample button reads "Wakefield et al. Lancet 1998 (PMID 9500320), the canonical retracted paper. `[identifier-verified per WebFetch against PubMed + Retraction Watch on 2026-05-21; PMID 9500320 = Wakefield et al. Lancet 1998 351(9103):637-41, fully retracted 2010-02-06 (PMID 20137807)]`".

The audit run fans out per-paper GETs after batch lookup. On a 500-PMID bibliography the first stage is ~5 batch-lookup calls (`POST /records/lookup` with `items[].pmid`, batched to stay under any prudent per-call payload size). The second stage is the per-retracted-paper GET — typically <20 retracted papers in a 500-PMID SR cohort (real-world retraction rate ~0.04–4% depending on field, per published retraction-residue bibliometrics), so the second-stage fan-out is at most a few dozen GETs. The route wraps the per-paper fan-out generator in `withIdleTimeout(gen, 180_000)` per Hard Rule #7 and the client renders an analyst-visible **Cancel** button next to the primary "Run cascade audit" action while the fan-out is in flight. A progress indicator increments per resolved paper ("cascade audit in progress — N of M papers walked"); a stalled Amass call past the 180s idle threshold surfaces a "stream stalled — an Amass call likely hung" error without locking the UI indefinitely.

The final state is a **downloadable cascade report** (Markdown or JSON) plus a "copy citations to clipboard" affordance for the cascade-tracked papers. The cascade panels surface real retraction data from **live Amass `isRetracted` values and live `references` / `citedBy` / `referencesTrialCore` arrays** — never from hardcoded fixtures, never from a real-PMID-flipped-to-retracted-for-illustration (the display=claim rule for retraction rendering makes hardcoded retraction values defamation-adjacent per Hard Rule #13). Render PMIDs / DOIs / NCTs / `AMBC_` / `AMTC_` IDs in IBM Plex Mono; prose in Inter (Brand reference). Use Lucide React for icons. Lovable produces the UI shape from this spec — variability across builders is fine; the load-bearing constraint is that the four paragraphs above demonstrate the single-record-fetch three-direction conjunction (`isRetracted` default field + `references` + `citedBy` + `referencesTrialCore` in one call), the live-Amass retraction display, and the worked-example anchor.

---

## Hard rules

Per-prototype rules 9-13 cover the load-bearing UX, error, and discipline surfaces specific to the retraction-cascade workflow.

1. **Credentials never enter committed code.** Generate `.env` and `.gitignore` it. Ask the user to paste credentials at hand-off:
   ```env
   AMASS_API_KEY=your_amass_api_key_here
   ```
   Per API conventions: the key reads from `AMASS_API_KEY`. The base URL is **hardcoded** to `https://api.amass.tech` in `lib/amass.ts` (single production base; no per-user variance). To point at staging or a local proxy for development, edit the `AMASS_API_BASE_URL` constant in `lib/amass.ts` directly — do NOT re-introduce an env var for this (adding `process.env.AMASS_API_BASE_URL` to `lib/amass.ts` causes Lovable's secrets-UX to prompt the user for a value they don't have, which is unnecessary friction). Never hardcode the API key, never log it, never commit it.

2. **Server-only `AmassClient`.** Lives in `lib/amass.ts` with `import "server-only"` at the top. Never imported from a `"use client"` component. The browser only ever speaks to `/api/<endpoint>`. The Bearer token must never reach the browser bundle; the `server-only` package enforces this at build time.

3. **Singleton/cached `AmassClient`.** Created once on server boot via `getAmassClient()` factory, cached via `clientPromise ??=` pattern. Per-request instantiation is wasted work; the Bearer header is computed once at module-import time and reused.

4. **`Authorization: Bearer ${AMASS_API_KEY}` on every Amass request.** On `401` / `403`, surface the credential error directly — retry won't fix it. On `429`, read `Retry-After` and back off exponentially (per the `lib/amass.ts` snippet below). Rate limit is **60 requests / 60 seconds, per user+org** — the per-call retry helper IS the rate-limit defence for fan-out workflows.

5. **Read from the `data` key; handle errors at top-level AND per-item.** Two distinct error shapes: top-level (any non-2xx) — `{"error": {"status": ..., "code": ..., "message": ...}}` — caught by the route's outer `try/catch`. Per-item (`POST /records/lookup` returns HTTP 200 with body `{"data": [...]}` — the universal envelope applies here too) — each array element echoes the input alongside the result: `{"input": {...}, "amassIds": ["AMBC_..."]}` for success, or `{"input": {...}, "error": {"code": "NOT_FOUND", "message": "Identifier not found"}}` for failure (error is a structured object with `code` + `message` fields, NOT a bare string — empirically verified via direct curl on 2026-05-26). The `lib/amass.ts` `batchLookupBiomed` / `batchLookupTrial` methods unwrap the `data` envelope; the `mapLookupResult` helper extracts `error.message` to a string for caller-side rendering, preserving positional index alignment with the input items. **Never assume HTTP 200 means every item succeeded. Never render `error` as a React child directly — extract `.message` first or React #31 crashes the surface.**

6. **Lookup endpoints translate PMID/DOI/NCT → canonical `AMBC_`/`AMTC_` before fetching by ID.** Raw HTTP `GET /records/{id}` accepts **only** canonical Amass IDs — passing a PMID directly returns 404. The workflow is always: `POST /records/lookup` with `[{pmid: "..."}]` → read `amassIds[0]` (handling per-item-error per Rule #5) → `GET /records/{amassId}`. The `lib/amass.ts` `resolvePmid` / `resolveDoi` / `resolveNct` helpers encode this discipline.

7. **Idle-timeout + analyst-visible cancel button on long-running fan-outs.** Three requirements: (a) the route wraps any fan-out async generator in `withIdleTimeout(gen, 180_000)` — if no chunk arrives for 180s, the wrapper throws; (b) the client renders an analyst-visible **Cancel** button next to the primary action while a fan-out is in flight; (c) the route wraps body parse + client setup in an outer `try/catch` returning `Response.json({error}, {status: 500})`. Without all three, a single stalled Amass call locks the UI indefinitely.

8. **Ground-truth discipline.** Every URL, type, function name, version, and config shape in this SKILL.md must come from a source the agent can read — a fetched doc page, a `.d.ts` in `node_modules/@amass/sdk/`, the installed package's exports, or a verbatim quote from the Amass API documentation. If you can't point to a source, don't write it. Forbidden: fabricating endpoints; guessing field names; `as unknown as ...` or `as any` on `@amass/sdk` values; ignoring `tsc` errors. When the runtime contradicts what's written here, fix the cause.

### Per-prototype hard rules

9. **Caller-supplied bibliography snapshot — lookup + get-by-ID exclusively (v0.1).** The tool takes a curated PMID list (caller's stored bibliography snapshot at time t0); periodic re-audit re-resolves the same PMIDs to detect retractions added between t0 and tN. v0.1 uses `POST .../records/lookup` + `GET .../records/{amassId}?include=...` exclusively — NOT BiomedCore search. Rationale: the lookup-only path keeps v0.1 strictly within documented Amass API guarantees; a v1.0 search-driven extension that auto-discovers newly-retracted papers across a topic area would be valuable but lives outside v0.1's scope (it is queued under "Want to extend it?" in Hand-off as a search-driven discovery follow-up).

10. **Three-direction cascade in ONE record fetch.** Every per-paper walk uses `GET /api/v1/cores/biomedcore/records/{amassId}?include=references,citedBy,referencesTrialCore` to retrieve outbound (`references`) + inbound (`citedBy`) + cross-core (`referencesTrialCore`) directions in a single HTTP call. NO second HTTP call for the inbound direction (vs OpenAlex's `cited_by` requiring a separate `/works?filter=cites:{id}` call); NO second vendor for the cross-core direction (vs OpenAlex's missing trial entity). Rationale: the single-record-fetch three-direction conjunction (with `isRetracted` as a default field) is the load-bearing distinctive the cascade workflow exists to demonstrate — collapsing it to multiple calls or multiple vendors defeats the value proposition that justifies the prototype.

11. **Per-item-error verbatim preservation.** Surface the upstream-generated per-item error string verbatim in the `lookup_error` column of the cascade-report row — never paraphrased, never elided, never normalised. The `mapLookupResult` helper in `lib/amass.ts` enforces this by passing the raw `r.error` string through unchanged. Rationale: for an auditable retraction-cascade chain where the SR / regulatory-affairs / medical-info reviewer is making a pull-from-dossier decision, "this PMID is unresolved with reason X" vs "this PMID is silently absent" is the difference between an auditable gap and an audit failure; paraphrasing the upstream reason string collapses the two.

12. **Paper→trial direction only on the cross-core walk (v0.1).** v0.1 sets `include=referencesTrialCore` on the paper-side BiomedCore GET; the trial-side-driven "describe-this-trial" reverse walk (NCT → all retracted-or-not supporting publications via `referencesBiomedCore`) is explicitly foreclosed because at-scale symmetry of the cross-core spine is not yet a published guarantee. Empty `referencesTrialCore` arrays (a retracted paper that describes no clinical trial — common for retracted review articles, retracted commentary, retracted basic-science papers) render as **honest emptiness** in the cross-core panel — an explicit "no cited trial in cross-core spine" sidecar note — not as a workflow halt and not as confabulated trial linkages. Rationale: foreclosing the reverse direction keeps v0.1 within published guarantees while the cascade workflow itself (retracted-paper → cited-trial direction) is complete; pipelines wanting both-direction completeness can union with `referencesBiomedCore` (a second `GET /api/v1/cores/trialcore/records/{amtcId}?include=referencesBiomedCore` call per surfaced trial) — out-of-scope for v0.1.

13. **Worked-example anchoring (verify-then-ship-strictly).** The cascade dashboard DISPLAYS `isRetracted` for cascade-tracked papers (the displayed field IS the claim — display=claim rule) AND the workflow ASSERTS retraction events as cascade triggers ("this paper's retraction caused these N downstream papers to carry contamination"). Both display and assertion appear within one workflow, so the workflow is defamation-adjacent. The "Try sample" PMID list MUST use a web-verified retracted paper — Wakefield Lancet 1998 PMID 9500320 (verified via WebFetch on 2026-05-21 against PubMed at https://pubmed.ncbi.nlm.nih.gov/9500320/ and against Retraction Watch at https://retractionwatch.com/?s=wakefield+lancet+1998). Carry the caveat marker `[identifier-verified per WebFetch against PubMed + Retraction Watch on 2026-05-21; PMID 9500320 = Wakefield et al. Lancet 1998 351(9103):637-41, fully retracted 2010-02-06 (PMID 20137807); partial retraction of interpretation 2004-03-06 (PMID 15016483)]` at three anchor sites: the title intro paragraph, the Try-sample tooltip, and the Hand-off verification preamble. The `isRetracted` column in every rendered row populates from the **live Amass record at run time**, NEVER from a hardcoded fixture, NEVER from a real-PMID-flipped-to-retracted-for-illustration — because an unsupported retraction display IS a retraction claim against the named work. The live-Amass-binding requirement structurally preserves the guard (Lovable scaffolds against the live API, not against fabricated retraction data). Rationale: the cascade UI demonstrates Amass's value by surfacing a REAL retraction with a REAL three-direction cascade — the Wakefield paper is the canonical demonstration because its retraction is documented for 14+ years, undisputed, and the cascade is bibliographically well-characterised (~3,000+ citing papers per published bibliometrics). Confabulating any of the load-bearing identifiers would be defamation-adjacent shipping.

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

**`lib/amass.ts`** — singleton `AmassClient` with Bearer-auth, 429 retry/backoff (reads `Retry-After`), per-item-error helper (`mapLookupResult`), and `{data: ...}` envelope unwrap. The cascade default include set on `getBiomedRecord` is `["references", "citedBy", "referencesTrialCore"]` per Hard Rule #10 — outbound `references` + inbound `citedBy` + cross-core trial spine in one call. Per Hard Rules #2, #3, #4, #5, #6, #10, #11.

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

  // Per-item-error helper. Each result is either {amassIds: [...]} or {error: {code, message}}.
  // Per empirical observation, each result is either {input?, amassIds: [...]} or
  // {input?, error: {code, message}}. Per Hard Rule #11, the upstream-generated
  // error.message string passes through verbatim — never paraphrased.
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

  // Lookup: PMID/DOI/NCT → canonical AMBC_/AMTC_ (Hard Rule #6 + #9).
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

  // Cascade default includes per Hard Rule #10: outbound references + inbound citedBy +
  // cross-core referencesTrialCore — three directions in one GET. isRetracted is a default
  // field so no include flag needed for the retraction signal itself.
  async getBiomedRecord(
    amassId: string,
    includes: ReadonlyArray<"fulltext" | "authorsMetadata" | "meshIds" | "substanceIds" | "referencesTrialCore" | "references" | "citedBy"> = ["references", "citedBy", "referencesTrialCore"],
  ): Promise<unknown /* bind to @amass/sdk BiomedRecord type once installed */> {
    const qs = includes.length > 0 ? `?${includes.map((i) => `include=${i}`).join("&")}` : "";
    return this.unwrapData(this.request<{ data: unknown }>("GET", `/api/v1/cores/biomedcore/records/${amassId}${qs}`));
  }

  // TrialCore get-by-ID — used to render trial-side cards when the analyst clicks through an AMTC_
  // anchor in the cross-core cascade panel. v0.1 does NOT call this on the reverse-direction walk
  // (per Hard Rule #12).
  async getTrialRecord(
    amassId: string,
    includes: ReadonlyArray<"referencesBiomedCore" | "outcomeMeasures" | "armsInterventions"> = [],
  ): Promise<unknown /* bind to @amass/sdk TrialRecord type once installed */> {
    const qs = includes.length > 0 ? `?${includes.map((i) => `include=${i}`).join("&")}` : "";
    return this.unwrapData(this.request<{ data: unknown }>("GET", `/api/v1/cores/trialcore/records/${amassId}${qs}`));
  }
}

const AMASS_API_BASE_URL = "https://api.amass.tech";

let clientPromise: Promise<AmassClient> | null = null;

export function getAmassClient(): Promise<AmassClient> {
  return (clientPromise ??= Promise.resolve(new AmassClient({
    apiKey: process.env.AMASS_API_KEY!,
    baseUrl: AMASS_API_BASE_URL,
  })));
}

export { AmassClient };
```

**`app/api/cascade-audit/route.ts`** — Next.js App Router POST handler. Takes a `{pmids: string[]}` body (the stored bibliography snapshot per Hard Rule #9), runs batch lookup to resolve `AMBC_` IDs, fans out the per-paper GET with `include=references,citedBy,referencesTrialCore` (per Hard Rule #10) wrapped in `withIdleTimeout(gen, 180_000)` per Hard Rule #7, filters to `isRetracted === true`, and returns the cascade keyed by retracted `AMBC_`. Non-streaming consolidated JSON response for v0.1.

```ts
import { AmassClient, getAmassClient } from "@/lib/amass";

interface CascadeRequest {
  pmids: string[];
}

interface PaperRecord {
  amassId: string;
  pmid: string | null;
  title: string | null;
  journal: string | null;
  publicationDate: string | null;
  isRetracted: boolean | null;
  references: string[];                // AMBC_ IDs — outbound
  citedBy: string[];                   // AMBC_ IDs — inbound
  referencesTrialCore: string[];       // AMTC_ IDs — cross-core
}

interface CascadeRow {
  input_pmid: string;
  resolved_amassId: string | null;
  lookup_error: string | null;         // per Hard Rule #11 — verbatim upstream string
  walk_error: string | null;
  paper: PaperRecord | null;
}

interface CascadeReport {
  rows: CascadeRow[];
  retracted: PaperRecord[];            // filtered subset where isRetracted === true
  cascades: Record<string, {           // keyed by retracted paper's AMBC_
    inbound: string[];                 // AMBC_ IDs from citedBy
    outbound: string[];                // AMBC_ IDs from references
    cross_core: string[];              // AMTC_ IDs from referencesTrialCore
  }>;
  errors: { lookup: number; walk: number };
}

async function* withIdleTimeout<T>(gen: AsyncGenerator<T>, maxIdleMs = 180_000): AsyncGenerator<T> {
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

// Extract the cascade-relevant fields from the BiomedCore record (per Hard Rule #10 + #12).
// Empty referencesTrialCore arrays render as honest emptiness — not as a workflow halt.
function extractPaper(amassId: string, record: unknown): PaperRecord {
  const r = record as {
    pmid?: string | null;
    title?: string | null;
    journal?: string | null;
    publicationDate?: string | null;
    isRetracted?: boolean | null;
    references?: string[];
    citedBy?: string[];
    referencesTrialCore?: string[];
  };
  return {
    amassId,
    pmid: r.pmid ?? null,
    title: r.title ?? null,
    journal: r.journal ?? null,
    publicationDate: r.publicationDate ?? null,
    isRetracted: r.isRetracted ?? null,
    references: r.references ?? [],
    citedBy: r.citedBy ?? [],
    referencesTrialCore: r.referencesTrialCore ?? [],
  };
}

export async function POST(req: Request) {
  try {
    const { pmids } = (await req.json()) as CascadeRequest;
    if (!Array.isArray(pmids) || pmids.length === 0) {
      return Response.json({ error: "Request body requires { pmids: string[] } with at least one PMID." }, { status: 400 });
    }
    const client = await getAmassClient();

    // Stage 1 — batch lookup PMID → AMBC_ (Hard Rule #6 + #9 + #11).
    const pmidItems = pmids.map((pmid) => ({ pmid }));
    const lookupResults = await client.batchLookupBiomed(pmidItems);

    // Per Hard Rule #11: lookup-side errors populate row.lookup_error verbatim.
    const rows: CascadeRow[] = [];
    const amassIdsToWalk: string[] = [];

    AmassClient.mapLookupResult(
      pmidItems,
      lookupResults,
      ({ pmid }, amassIds) => {
        amassIdsToWalk.push(amassIds[0]);
        rows.push({ input_pmid: pmid!, resolved_amassId: amassIds[0], lookup_error: null, walk_error: null, paper: null });
        return null;
      },
      ({ pmid }, msg) => {
        rows.push({ input_pmid: pmid!, resolved_amassId: null, lookup_error: msg, walk_error: null, paper: null });
        return null;
      },
    );

    // Stage 2 — fan out per-paper GET with the three-direction include set (Hard Rule #10).
    // Wrapped in withIdleTimeout per Hard Rule #7.
    async function* walkPapers(): AsyncGenerator<{ amassId: string; paper: PaperRecord | null; err: string | null }> {
      for (const amassId of amassIdsToWalk) {
        try {
          const record = await client.getBiomedRecord(amassId, ["references", "citedBy", "referencesTrialCore"]);
          yield { amassId, paper: extractPaper(amassId, record), err: null };
        } catch (e) {
          yield { amassId, paper: null, err: e instanceof Error ? e.message : "walk failed" };
        }
      }
    }

    for await (const { amassId, paper, err } of withIdleTimeout(walkPapers(), 180_000)) {
      const row = rows.find((r) => r.resolved_amassId === amassId);
      if (!row) continue;
      if (err) {
        row.walk_error = err;
        continue;
      }
      row.paper = paper;
    }

    // Filter to retracted subset (isRetracted is a default field — no include flag needed).
    const retracted = rows
      .map((r) => r.paper)
      .filter((p): p is PaperRecord => p !== null && p.isRetracted === true);

    // Assemble the cascade keyed by retracted AMBC_ — three-direction arrays straight from the live record.
    const cascades: CascadeReport["cascades"] = {};
    for (const p of retracted) {
      cascades[p.amassId] = {
        inbound: p.citedBy,
        outbound: p.references,
        cross_core: p.referencesTrialCore,
      };
    }

    const report: CascadeReport = {
      rows,
      retracted,
      cascades,
      errors: {
        lookup: rows.filter((r) => r.lookup_error).length,
        walk: rows.filter((r) => r.walk_error).length,
      },
    };
    return Response.json(report);
  } catch (err) {
    return Response.json({ error: err instanceof Error ? err.message : "Cascade audit failed" }, { status: 500 });
  }
}
```

The two snippets above are the only `lib/<file>.ts` + `app/api/<route>/route.ts` mandated by this prototype. v0.1 returns the assembled cascade report in-memory in a single consolidated response — there is no persistence module, no `lib/cascade-store.ts`, no monthly re-audit cron, and no diff-against-prior-pass module. Download / copy-to-clipboard happens client-side from the response payload. If a downstream extension needs persistence (monthly re-audit history, week-over-week diff), the extension SKILL.md includes its own minimal stub per the template's policy — never an `import` from `@/lib/<x>` without a matching snippet.

---

## Required reading

- `https://platform.amass.tech` — Amass Platform docs + API credentials
- `https://github.com/amass-technologies/public-amass-platform-starter-ts` — official Amass Platform Starter (Apache-2.0; Bun + Ink CLI REPL with BiomedCore + TrialCore tools wired up). The engineer-facing complement to this web-app kit.

---

## Hand-off

Build, lint, and typecheck must pass.

**Credentials.** Get your `AMASS_API_KEY` from https://platform.amass.tech and add it to `.env`. Do NOT use `AskUserQuestion` to gate scaffolding on credentials — Lovable's (or any AI builder's) standard env-var prompt covers this at run time. Just scaffold the app with placeholder `.env`, then run `npm run dev` once the key is in place.

Then present the hand-off summary. The four verification steps bind to the WebFetch-verified Wakefield Lancet 1998 worked example `[identifier-verified per WebFetch against PubMed + Retraction Watch on 2026-05-21; PMID 9500320 = Wakefield et al. Lancet 1998 351(9103):637-41, fully retracted 2010-02-06 (PMID 20137807); partial retraction of interpretation 2004-03-06 (PMID 15016483)]`:

> **Your Retraction Cascade Monitor is ready.**
>
> Before first push (if your AI builder left scaffolding placeholders): remove any `data-lovable-blank-page-placeholder="REMOVE_THIS"` attributes, any `<img>` tags pointing at `cdn.gpteng.co` or similar builder-placeholder CDNs, and any leftover "your app will live here" boilerplate. These render invisibly but pollute the DOM and serve no purpose in production. Search the codebase for `REMOVE_THIS` and `gpteng` before committing.
>
> To run it: fill in `.env` and run `npm run dev`.
>
> Verify it works:
> - Click **Try sample** to load the seed PMID list (PMID 9500320 — Wakefield Lancet 1998 — plus ~10 context PMIDs from the autism-vaccination methodology literature); click **Run cascade audit**. Confirm the retracted-papers panel (left) surfaces PMID 9500320 with `isRetracted=true` rendered in IBM Plex Mono next to the title "Ileal-lymphoid-nodular hyperplasia, non-specific colitis, and pervasive developmental disorder in children" and journal "Lancet" — the `isRetracted` value reads from the live Amass record at run time per Hard Rule #13, never from a hardcoded fixture.
> - Confirm the inbound cascade panel (center) populates with a non-empty list of `AMBC_` IDs from PMID 9500320's `citedBy` array — these are papers in Amass that cite Wakefield (expect dozens to hundreds depending on Amass's PubMed coverage of the autism-vaccination citation literature; the cardinality is live-from-Amass, never hardcoded). Click any cited paper to view its title/journal/date in IBM Plex Mono.
> - Confirm the outbound cascade panel (right) populates with the `AMBC_` IDs from PMID 9500320's `references` array — these are the papers Wakefield itself cited. The combined inbound + outbound + cross-core delivery in ONE GET call (`include=references,citedBy,referencesTrialCore`) is the load-bearing distinctive per Hard Rule #10 — there is no second HTTP call for the inbound direction, vs OpenAlex's `cited_by` requiring a separate `/works?filter=cites:{id}` call.
> - Confirm the cross-core panel (right-bottom) renders either the `AMTC_` IDs of clinical trials Wakefield's paper describes — OR renders honest emptiness (`"no cited trial in cross-core spine"` per Hard Rule #12) if Wakefield's paper does not describe a registered clinical trial. The honest-emptiness frame is load-bearing: a v0.1 surface that confabulated trial linkages on a retracted paper would be defamation-adjacent shipping per Hard Rule #13.
>
> Want to extend it?
> - Add a monthly re-audit cron with diff against the prior pass — surfaces newly-retracted citations between t0 (the bibliography snapshot) and tN (the re-audit cycle); persists the cascade report append-only so SR teams / regulatory affairs / medical-info teams can reconstruct the contamination state at any prior cycle. The cron lives at `app/api/cascade-cron/route.ts` with append-only persistence in a `lib/cascade-store.ts` stub.
> - Add a v1.0 search-driven discovery extension — uses BiomedCore search with `isRetracted=true` + topic filters to auto-discover newly-retracted papers across an indication or a sponsor's literature rather than against a stored bibliography. Out of scope for v0.1 — gated on additional Amass API support for retraction-filtered search.
> - Add a cross-core sponsor-level cascade — "all trials by sponsor X — any retraction-cascaded via cited literature?" — using `TrialCore search` by `sponsorName` + per-trial `referencesBiomedCore` walk + retraction check on the cited papers. Inherits the cross-core symmetry caveat per Hard Rule #12 (both-direction completeness via the reverse `referencesBiomedCore` walk is not yet a published guarantee).
> - I'm done — just show me the summary
>
> Have questions? Join the Amass Developer Community on Discord: https://discord.com/invite/sEGaBHMhWa

---

## License

Apache-2.0. Copyright 2026 Amass Technologies. Source at https://github.com/lluisDTU/amass-technologies-amass-starters.
