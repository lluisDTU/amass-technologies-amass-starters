---
name: amass-sr-pre-screen
description: Use when building a PRISMA pre-screen credibility-filter for systematic reviews (Cochrane / SR-as-a-service / academic). Provides build instructions for batch-resolving a curated PMID dump to canonical Amass IDs, post-filtering on JuFo + retraction + citation count, and emitting a Rayyan/Covidence-importable RIS file with an audit-trail CSV in a TypeScript app (Lovable typically scaffolds TanStack Start; Next.js App Router is the alternative path — both work end-to-end).
license: Apache-2.0
metadata:
  author: amass
  version: "0.1.6"
---

# SR Pre-Screen Skill (Amass BiomedCore)

Build a working PRISMA pre-screen credibility-filter on the Amass API. The user is a systematic-review researcher running PRISMA 2020 who just got Amass API credentials and wants a TypeScript pre-screen tool running locally in under five minutes. Lovable typically scaffolds TanStack Start (TanStack Router + `createServerFn` server-functions); Next.js (App Router with `app/api/<endpoint>/route.ts`) is the alternative path. Both work end-to-end against the same `lib/amass.ts` client pattern — the Amass primitives are stack-agnostic.

The SR researcher pastes a curated PMID dump from the upstream PRISMA search step (PubMed / Embase / CENTRAL / Scopus produce the standard 5,000-PMID export); the tool resolves each PMID to a canonical `AMBC_` Amass ID via batch lookup, fans out per-paper GETs to retrieve `isRetracted` + `journalQualityJufo` + `citationCount`, post-filters client-side against the configured credibility thresholds, and emits a Rayyan/Covidence-importable RIS file plus an audit-trail CSV — dropping the 5,000-PMID candidate set to a ~100-500-paper screening set before title/abstract screening. The worked example binds to an illustrative SR scope ("GLP-1 receptor agonists in obesity") with ~10 representative PMIDs. `[identifier-verification caveat — example identifiers assembled without live web access; re-verify against PubMed before binding to a real SR workflow]`

---

## Stack — flexible at the framework layer, principle-pinned at the API layer

**TypeScript + React + a server-function-capable framework.** Lovable defaults to **TanStack Start** (`@tanstack/react-router` + `@tanstack/react-start`'s `createServerFn` for server-side handlers). **Next.js (App Router)** with `app/api/<endpoint>/route.ts` POST handlers is the equivalent alternative path. Both work end-to-end against the same `lib/amass.ts` client; pick whichever your AI builder produces by default. The Amass primitives (canonical-ID resolution + per-item-error semantics + cross-core walks + trust filters) are stack-agnostic — the framework only wraps them.

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

Pin with caret ranges (`^x.y.z`) for compatible patch updates. `zod` is pinned because the input-validator schema (`PreScreenSchema`) is the request contract — silent zod-version drift in default-value or refine semantics shifts what the route accepts. `server-only` is pinned because it is the build-time enforcement Hard Rule #2 depends on (works identically in Next.js and TanStack Start). The per-item-error verbatim preservation (Hard Rule #11) and audit-trail completeness are load-bearing; do not paraphrase upstream error strings.

After scaffolding, run `npm install`. If any package fails to resolve, check `node_modules/<pkg>/package.json` for the latest stable version and update accordingly.

**Three notes for AI builders that may produce either path.** (1) The route file location differs: TanStack Start uses `src/lib/<name>.functions.ts` with `createServerFn({ method: "POST" }).inputValidator(...).handler(...)`; Next.js App Router uses `app/api/<endpoint>/route.ts` with `export async function POST(req: Request)`. The reference snippet below is shown in Next.js shape — if your builder produces TanStack Start, translate the `POST(req)` handler to `createServerFn(...).handler(async ({ data }) => ...)` and the `req.json()` body parse to `.inputValidator((input) => PreScreenSchema.parse(input))`. (2) The page file location differs: TanStack Router uses `src/routes/index.tsx` with `createFileRoute("/")`; Next.js App Router uses `app/page.tsx`. Either way, the page calls into the server function / route handler the same way at the network level. (3) **The outer `try/catch` wrapping the handler body MUST be preserved when translating to TanStack Start.** The Next.js reference snippet wraps the audit logic in `try { ... } catch (err) { return Response.json({error: ...}, {status: 500}) }`. For TanStack Start, wrap the `.handler()` body in `try { ... } catch (err) { throw new Error(err instanceof Error ? err.message : "Pre-screen run failed"); }` — the throw surfaces as `mutation.error.message` on the client (TanStack Query's `useMutation` `isError` path) and renders in the UI's inline error region. Without this wrap, an uncaught throw from any API call propagates to TanStack Router's route-level error boundary and crashes the whole page instead of surfacing as a mutation error. Load-bearing for the kit's "per-item error renders inline without crashing the surface" claim.

---

## What the app should do

The primary analyst-visible payoff is the **credibility-filtered RIS file** — importable into Rayyan or Covidence as the title/abstract screening corpus, dropping a 5,000-PMID PRISMA upstream-search dump to a ~100-500-paper screening set. The load-bearing Amass primitive is the trust-filter conjunction at non-commercial tier: `isRetracted` as a server-side default field on `GET /records/{amassId}`; `journalQualityJufo` as a client-side post-filter on the returned field (the JuFo Nordic/EU national-evaluation tier is distinctive — not exposed by PubMed natively, not exposed by OpenAlex); and `citationCount` as a client-side post-filter on the returned field. The composition in one workflow against a single canonical-AMBC-ID record is the residual moat.

The input surface is a **paste-PRISMA-export textarea** — one PMID per line, matching the standard PubMed `Send to → File → PMID` export format that PRISMA upstream searches produce (PubMed / Embase / CENTRAL / Scopus). The empty-state "Try sample" loads the GLP-1 RA in obesity illustrative SR scope verbatim — `~10 representative PMIDs for the GLP-1 receptor agonist in obesity literature` (tooltip: "GLP-1 RA in obesity SR pre-screen — illustrative scope; small representative PMID set. `[identifier-verification caveat — example identifiers assembled without live web access; re-verify against PubMed before binding to a real SR workflow]`"). A "thresholds" form sits next to the textarea showing the active `min_jufo` + `allow_retracted` + `min_citation_count` values (tunable per Hard Rule #10).

The pre-screen run fans out batch-lookups + per-paper GETs at the 60 req / 60 s ceiling. A 5,000-PMID input chunks into ~50 batch-lookup calls (~100 items each per conservative items-array ceiling heuristic) + ~500 per-paper GETs at the post-filter step = ~550 calls per pre-screen run; wall-time ~15 min at the rate-limit ceiling. The route wraps the per-paper fan-out async generator in `withIdleTimeout(gen, 180_000)` per Hard Rule #7 and the client renders an analyst-visible **Cancel** button next to the primary "Run pre-screen" action while the fan-out is in flight. A progress indicator increments per resolved paper ("pre-screen run in progress — N of M papers post-filtered"); a stalled Amass call past the 180s idle threshold surfaces a "stream stalled — an Amass call likely hung" error without locking the UI indefinitely.

The final state offers two downloadable artifacts side-by-side: a **Rayyan/Covidence-importable RIS file** (`TY=JOUR, PMID, TI, AU, JO, JF, PY, DO, AB` per the RIS field mapping the screening tools expect) emitted from the filtered set, and an **audit-trail CSV** with the 11-column shape `pmid, accessed_at, AMBC_id, isRetracted, journalQualityJufo, citationCount, lookup_error, included_in_prescreen, threshold_min_jufo, threshold_min_citation_count, threshold_allow_retracted` — the three trailing `threshold_*` columns repeat the run-time threshold values per row so a reviewer downstream can reproduce the inclusion/exclusion verdicts without needing to remember what the screener set. The v0.1 returns the assembled rows in-memory in a single consolidated response — there is no monthly re-audit cron (lands under "Want to extend it?" in Hand-off), no v1.0 BiomedCore-search-driven discovery, no Python CLI sidecar. Render PMIDs / DOIs / `AMBC_` IDs in IBM Plex Mono; prose in Inter (Brand reference). Use Lucide React for icons. Lovable produces the UI shape from this spec — variability across builders is fine; the load-bearing constraint is that the four paragraphs above demonstrate the trust-filter conjunction at non-commercial tier, the lookup-then-fetch canonical-ID flow, the per-item-error column split, and the worked-example anchor.

---

## Hard rules

The 8 Amass-universal rules below are inherited verbatim across all kit prototype SKILL.mds. Per-prototype rules 9-12 cover the load-bearing UX / error / discipline surfaces specific to PRISMA pre-screen.

1. **Credentials never enter committed code.** Generate `.env` and `.gitignore` it. Only ONE secret is asked of the user at hand-off:
   ```env
   AMASS_API_KEY=your_amass_api_key_here
   ```
   Per `CLAUDE.md` API conventions: the key reads from `AMASS_API_KEY`. The base URL is **hardcoded** to `https://api.amass.tech` in `lib/amass.ts` (single production base; no per-user variance). To point at staging or a local proxy for development, edit the `AMASS_API_BASE_URL` constant in `lib/amass.ts` directly — do NOT re-introduce an env var for this (adding `process.env.AMASS_API_BASE_URL` to `lib/amass.ts` causes Lovable's secrets-UX to prompt the user for a value they don't have, which is unnecessary friction). Never hardcode the API key, never log it, never commit it.

2. **Server-only `AmassClient`.** Lives in `lib/amass.ts` with `import "server-only"` at the top. Never imported from a `"use client"` component. The browser only ever speaks to `/api/<endpoint>`. The Bearer token must never reach the browser bundle; the `server-only` package enforces this at build time.

3. **Singleton/cached `AmassClient`.** Created once on server boot via `getAmassClient()` factory, cached via `clientPromise ??=` pattern. Per-request instantiation is wasted work; the Bearer header is computed once at module-import time and reused.

4. **`Authorization: Bearer ${AMASS_API_KEY}` on every Amass request.** On `401` / `403`, surface the credential error directly — retry won't fix it. On `429`, read `Retry-After` and back off exponentially (per the `lib/amass.ts` snippet below). Rate limit is **60 requests / 60 seconds, per user+org** — the per-call retry helper IS the rate-limit defence for the 5,000-PMID-driven ~550-call pre-screen fan-out.

5. **Read from the `data` key; handle errors at top-level AND per-item.** Two distinct error shapes: top-level (any non-2xx) — `{"error": {"status": ..., "code": ..., "message": ...}}` — caught by the route's outer `try/catch`. Per-item (`POST /records/lookup` returns HTTP 200 with body `{"data": [...]}` — the universal envelope applies here too) — each array element echoes the input alongside the result: `{"input": {...}, "amassIds": ["AMBC_..."]}` for success, or `{"input": {...}, "error": {"code": "NOT_FOUND", "message": "Identifier not found"}}` for failure (error is a structured object with `code` + `message` fields, NOT a bare string — empirically verified via direct curl on 2026-05-26). The `lib/amass.ts` `batchLookupBiomed` / `batchLookupTrial` methods unwrap the `data` envelope; the `mapLookupResult` helper extracts `error.message` to a string for caller-side rendering, preserving positional index alignment with the input items. **Never assume HTTP 200 means every item succeeded. Never render `error` as a React child directly — extract `.message` first or React #31 crashes the surface.**

6. **Lookup endpoints translate PMID/DOI/NCT → canonical `AMBC_`/`AMTC_` before fetching by ID.** Raw HTTP `GET /records/{id}` accepts **only** canonical Amass IDs — passing a PMID directly returns 404. The workflow is always: `POST /records/lookup` with `[{pmid: "..."}]` → read `amassIds[0]` (handling per-item-error per Rule #5) → `GET /records/{amassId}`. The `lib/amass.ts` `resolvePmid` / `resolveDoi` / `resolveNct` helpers encode this discipline; for the SR pre-screen the batch form (`batchLookupBiomed`) is used directly with chunks of ~100 PMIDs.

7. **Idle-timeout + analyst-visible cancel button on long-running fan-outs.** Three requirements: (a) the route wraps any fan-out async generator in `withIdleTimeout(gen, 180_000)` — if no chunk arrives for 180s, the wrapper throws; (b) the client renders an analyst-visible **Cancel** button next to the primary action while a fan-out is in flight; (c) the route wraps body parse + client setup in an outer `try/catch` returning `Response.json({error}, {status: 500})`. Without all three, a single stalled Amass call locks the UI indefinitely on a 15-minute pre-screen job.

8. **Ground-truth discipline.** Every URL, type, function name, version, and config shape in this SKILL.md must come from a source the agent can read — a fetched doc page, a `.d.ts` from the installed SDK, or the installed package's exports. If you can't point to a source, don't write it. Forbidden: fabricating endpoints; guessing field names; `as unknown as ...` or `as any` on SDK values; ignoring `tsc` errors. When the runtime contradicts what's written here, fix the cause.

### Per-prototype hard rules

9. **Caller-supplied PRISMA upstream-search input.** The tool consumes a curated PMID list (a PubMed / Embase / CENTRAL / Scopus export from the PRISMA upstream-search step); it does NOT replace the upstream-search step. SR researchers run PRISMA upstream discovery themselves via the standard PRISMA tools as part of PRISMA 2020 protocol — the tool slots **between** the upstream-search step (which produces the 5,000-PMID dump) and the title/abstract screen step (Rayyan / Covidence / Elicit, which consumes the credibility-filtered RIS this app produces). The v0.1 tool does NOT invoke BiomedCore search (`GET .../records?query=...`); in-tool search-driven discovery is out of scope for this version.

10. **Trust-filter post-filter discipline.** Three of the three trust signals come back on the per-paper GET, but they reach the filter at different layers — and conflating them collapses the discipline. `isRetracted` is a server-side default field on `GET /records/{amassId}` (the default-field behaviour, not a query parameter). `journalQualityJufo` and `citationCount` are returned-field client-side post-filters — server-side `minCitationCount` is not currently available as a query parameter. The audit-trail CSV records all three trust signals verbatim from the live record so the SR researcher can re-tune the thresholds post-hoc; never invent JuFo or citation values from PubMed-side metadata.

11. **Per-item error verbatim preservation.** Surface the upstream-generated per-item error string verbatim in the `lookup_error` column — never paraphrased, never elided, never normalised. The `mapLookupResult` helper in `lib/amass.ts` enforces this by passing the raw `r.error` string through unchanged. Rationale: for an SR audit trail (per PRISMA 2020 reporting standards), "this PMID is unresolved with reason X" vs "this PMID is silently absent" is the difference between an auditable gap and an audit failure; paraphrasing the upstream reason string collapses the two.

12. **Worked-example anchoring (caveat-and-ship).** The workflow is a screening-recommendation surface (the tool emits a candidate set for downstream title/abstract screening) — NOT a defamation-adjacent assertion surface (the tool does not assert any specific PMID is retracted; it displays the live `isRetracted` field from the Amass record verbatim alongside the credibility-filter decision). The "Try sample" loads the GLP-1 RA in obesity illustrative SR scope with ~10 representative PMIDs covering the literature shape (RCTs of semaglutide, liraglutide, tirzepatide in obesity / weight loss endpoints). Carry the caveat marker `[identifier-verification caveat — example identifiers assembled without live web access; re-verify against PubMed before binding to a real SR workflow]` at three anchor sites: the title intro paragraph, the Try-sample tooltip, and the Hand-off verification preamble. `isRetracted` + JuFo + citation count columns populate from the **live Amass record at runtime**, never from a hardcoded fixture — the display=claim defamation-adjacent guard structurally holds (Lovable scaffolds against the live API, not against fabricated retraction data). Rationale: the workflow is screening-recommendation (analyst makes the final inclusion decision in Rayyan/Covidence downstream), not retraction-claim-assertion; the live-Amass-binding requirement preserves the structural defamation-adjacent guard while the caveat marker keeps the verification-stakes audit visible.

---

## Brand reference

Source of truth for the Amass app look-and-feel. Load before writing `globals.css`.

### Logo

Render the Amass wordmark at the **top-left of the header**, ~24-32px. Use the text `amass` in Inter weight 800 24px as the wordmark. Place it left of the page title.

### Typography

- **Inter** — UI text, labels, headings, body. Weights 400/600/700/800/900.
- **IBM Plex Mono** — numerics, IDs (PMID / DOI / `AMBC_` / `AMTC_`), codes, monospace inputs. Weights 400/500/700. Never for prose.

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

**`lib/amass.ts`** — singleton `AmassClient` with Bearer-auth, 429 retry/backoff (reads `Retry-After`), per-item-error helper (`mapLookupResult`), and `{data: ...}` envelope unwrap. Per Hard Rules #2, #3, #4, #5, #6, #11.

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

  // Per-item-error helper. Each result is either {input?, amassIds: [...]} or
  // {input?, error: {code, message}} per the empirically observed API shape.
  // Per Hard Rule #11, the upstream-generated error.message string passes through
  // verbatim — never paraphrased.
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

  // Lookup: PMID/DOI → canonical AMBC_ (Hard Rule #6). The SR pre-screen uses the batch form
  // directly with chunks of ~100 PMIDs.
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

  // Per-paper GET — default includes []. The pre-screen post-filter reads the default-fields
  // (`isRetracted`, `journalQualityJufo`, `citationCount` returned by default) without
  // an explicit include flag. Opt-in `include=referencesTrialCore` for the cross-core sub-workflow
  // when Step 1 dispatch enables it.
  async getBiomedRecord(
    amassId: string,
    includes: ReadonlyArray<"fulltext" | "authorsMetadata" | "meshIds" | "substanceIds" | "referencesTrialCore" | "references" | "citedBy"> = [],
  ): Promise<unknown /* bind to @amass/sdk BiomedRecord type once installed */> {
    const qs = includes.length > 0 ? `?${includes.map((i) => `include=${i}`).join("&")}` : "";
    return this.unwrapData(this.request<{ data: unknown }>("GET", `/api/v1/cores/biomedcore/records/${amassId}${qs}`));
  }

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

**`app/api/prescreen-curated-ids/route.ts`** — Next.js App Router POST handler. Takes a JSON body `{pmids: string[], thresholds: {min_jufo, allow_retracted, min_citation_count}, include_referencesTrialCore?: boolean}`, chunks PMIDs into batches of 100 for `batchLookupBiomed`, runs the batch lookups concurrently, fans out per-paper GETs on the resolved `AMBC_` set wrapped in `withIdleTimeout(gen, 180_000)` per Hard Rule #7, post-filters client-side per Hard Rule #10, returns the filtered set + the audit-trail rows + lookup/walk error counts in one consolidated JSON response. Non-streaming for v0.1.

```ts
import { AmassClient, getAmassClient } from "@/lib/amass";

interface Thresholds {
  min_jufo: number;          // 0-3; JuFo tier post-filter threshold (default 2)
  allow_retracted: boolean;  // false = drop retracted; true = keep
  min_citation_count: number; // post-filter on returned citationCount (default 5)
}

interface PreScreenRequest {
  pmids: string[];
  thresholds: Thresholds;
  include_referencesTrialCore?: boolean; // opt-in cross-core sub-workflow
}

interface AuditRow {
  pmid: string;
  accessed_at: string;
  AMBC_id: string | null;
  isRetracted: boolean | null;
  journalQualityJufo: number | null;
  citationCount: number | null;
  referencesTrialCore_AMTCs: string[] | null;  // null when opt-in not enabled; [] when enabled but empty
  lookup_error: string | null;                  // per Hard Rule #11 — verbatim upstream string
  included_in_prescreen: boolean;
  // Run-time thresholds — same value repeats per row so a downstream reviewer can reproduce the
  // inclusion/exclusion verdicts without needing to remember what the screener set.
  threshold_min_jufo: number;
  threshold_min_citation_count: number;
  threshold_allow_retracted: boolean;
}

interface FilteredPaper {
  pmid: string;
  AMBC_id: string;
  title?: string;
  authors?: string[];
  journal?: string;
  publicationDate?: string;
  doi?: string;
  abstract?: string;
  isRetracted: boolean | null;
  journalQualityJufo: number | null;
  citationCount: number | null;
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

// Per Hard Rule #10 — `isRetracted` is the server-side default field; `journalQualityJufo` + `citationCount`
// are returned-field client-side post-filters per the quality-gate-papers MCP-vs-HTTP gap. Decision lives here.
function passesCredibilityFilter(
  record: { isRetracted: boolean | null; journalQualityJufo: number | null; citationCount: number | null },
  t: Thresholds,
): boolean {
  if (record.isRetracted === true && !t.allow_retracted) return false;
  if ((record.journalQualityJufo ?? -1) < t.min_jufo) return false;
  if ((record.citationCount ?? 0) < t.min_citation_count) return false;
  return true;
}

function chunk<T>(arr: T[], size: number): T[][] {
  const out: T[][] = [];
  for (let i = 0; i < arr.length; i += size) out.push(arr.slice(i, i + size));
  return out;
}

export async function POST(req: Request) {
  try {
    const body = (await req.json()) as PreScreenRequest;
    const client = await getAmassClient();
    const accessed_at = new Date().toISOString();
    const includes = body.include_referencesTrialCore ? (["referencesTrialCore"] as const) : ([] as const);

    // Step 1 — chunk PMIDs into batches of ~100 (conservative items[]-ceiling — the upper bound
    // is not yet quantified in published Amass docs, so 100 is a safe default) and run lookups
    // concurrently. Per-item-error rows populate even when the batch returns HTTP 200.
    const pmidItems = body.pmids.map((pmid) => ({ pmid }));
    const lookupChunks = chunk(pmidItems, 100);
    const lookupResults = await Promise.all(
      lookupChunks.map((items) => client.batchLookupBiomed(items)),
    );

    const auditRows: AuditRow[] = [];
    const amassIdsToWalk: string[] = [];

    lookupChunks.forEach((items, chunkIdx) => {
      AmassClient.mapLookupResult(
        items,
        lookupResults[chunkIdx],
        ({ pmid }, amassIds) => {
          amassIdsToWalk.push(amassIds[0]);
          auditRows.push({
            pmid: pmid!,
            accessed_at,
            AMBC_id: amassIds[0],
            isRetracted: null,
            journalQualityJufo: null,
            citationCount: null,
            referencesTrialCore_AMTCs: body.include_referencesTrialCore ? [] : null,
            lookup_error: null,
            included_in_prescreen: false,
            threshold_min_jufo: body.thresholds.min_jufo,
            threshold_min_citation_count: body.thresholds.min_citation_count,
            threshold_allow_retracted: body.thresholds.allow_retracted,
          });
          return null;
        },
        ({ pmid }, msg) => {
          // Per Hard Rule #11 — upstream `error` string passes through verbatim.
          auditRows.push({
            pmid: pmid!,
            accessed_at,
            AMBC_id: null,
            isRetracted: null,
            journalQualityJufo: null,
            citationCount: null,
            referencesTrialCore_AMTCs: null,
            lookup_error: msg,
            included_in_prescreen: false,
            threshold_min_jufo: body.thresholds.min_jufo,
            threshold_min_citation_count: body.thresholds.min_citation_count,
            threshold_allow_retracted: body.thresholds.allow_retracted,
          });
          return null;
        },
      );
    });

    // Step 2 — fan out per-paper GETs on the resolved AMBC_ set. Wrapped in withIdleTimeout per Hard Rule #7.
    // Per Hard Rule #10: default-fields fetch is enough for the three trust signals (no include flag
    // mandatory; opt-in include=referencesTrialCore only when Step 1 dispatch enabled it).
    const filtered: FilteredPaper[] = [];

    async function* walkPapers(): AsyncGenerator<{ amassId: string; record: unknown | null; err: string | null }> {
      for (const amassId of amassIdsToWalk) {
        try {
          const record = await client.getBiomedRecord(amassId, includes);
          yield { amassId, record, err: null };
        } catch (e) {
          yield { amassId, record: null, err: e instanceof Error ? e.message : "walk failed" };
        }
      }
    }

    for await (const { amassId, record, err } of withIdleTimeout(walkPapers(), 180_000)) {
      const row = auditRows.find((r) => r.AMBC_id === amassId);
      if (!row) continue;
      if (err) {
        // Per Hard Rule #10/11 — walk-side failure goes to lookup_error column (single error column
        // in the v0.1 audit-trail schema; collapsed to one column for Rayyan/Covidence ingestion-shape parity).
        row.lookup_error = err;
        continue;
      }
      const r = record as {
        title?: string; authors?: string[]; journal?: string; publicationDate?: string;
        doi?: string; abstract?: string;
        isRetracted?: boolean | null; journalQualityJufo?: number | null; citationCount?: number | null;
        referencesTrialCore?: string[];
      };
      row.isRetracted = r.isRetracted ?? null;
      row.journalQualityJufo = r.journalQualityJufo ?? null;
      row.citationCount = r.citationCount ?? null;
      if (body.include_referencesTrialCore) {
        row.referencesTrialCore_AMTCs = r.referencesTrialCore ?? [];
      }
      const credible = passesCredibilityFilter(
        { isRetracted: row.isRetracted, journalQualityJufo: row.journalQualityJufo, citationCount: row.citationCount },
        body.thresholds,
      );
      row.included_in_prescreen = credible;
      if (credible) {
        filtered.push({
          pmid: row.pmid,
          AMBC_id: amassId,
          title: r.title,
          authors: r.authors,
          journal: r.journal,
          publicationDate: r.publicationDate,
          doi: r.doi,
          abstract: r.abstract,
          isRetracted: row.isRetracted,
          journalQualityJufo: row.journalQualityJufo,
          citationCount: row.citationCount,
        });
      }
    }

    return Response.json({
      filtered,
      audit_trail: auditRows,
      errors: {
        lookup: auditRows.filter((r) => r.lookup_error).length,
      },
      summary: {
        input_count: body.pmids.length,
        resolved_count: auditRows.filter((r) => r.AMBC_id !== null).length,
        filtered_count: filtered.length,
        thresholds: body.thresholds,
      },
    });
  } catch (err) {
    return Response.json({ error: err instanceof Error ? err.message : "Pre-screen run failed" }, { status: 500 });
  }
}
```

**Client-side RIS emission.** The Rayyan/Covidence-importable RIS file is emitted client-side from the `filtered` array in the response. The RIS field mapping is `TY=JOUR` per record + `PMID` (mono) + `TI` (title) + `AU` (one per author) + `JO` (journal) + `JF` (full journal name) + `PY` (publication year — first 4 chars of `publicationDate`) + `DO` (DOI) + `AB` (abstract) + `ER` (end-of-record). The audit-trail CSV is emitted client-side from `audit_trail` with the 11-column shape `pmid, accessed_at, AMBC_id, isRetracted, journalQualityJufo, citationCount, lookup_error, included_in_prescreen, threshold_min_jufo, threshold_min_citation_count, threshold_allow_retracted` (`referencesTrialCore_AMTCs` appears as a 12th column when the Step 1 opt-in is enabled). The three trailing `threshold_*` columns repeat the same run-time threshold values per row — the redundancy is deliberate; a single CSV file is self-contained for reproducibility without needing the screener to remember what thresholds were in force at run time.

The two snippets above are the only `lib/<file>.ts` + `app/api/<route>/route.ts` mandated by this prototype. v0.1 returns the assembled response in-memory in a single consolidated JSON — there is no persistence module, no `lib/prescreen-store.ts`, no monthly re-audit cron, no Python CLI sidecar. RIS download / CSV download happens client-side from the response payload. If a downstream extension needs persistence (e.g. monthly re-audit history, in-tool BiomedCore-search discovery), include its minimal stub directly in the extension code — never an `import` from `@/lib/<x>` without a matching snippet.

---

## Required reading

- `https://platform.amass.tech` — Amass Platform docs + API credentials
- `https://github.com/amass-technologies/public-amass-platform-starter-ts` — official Amass Platform Starter (Apache-2.0; Bun + Ink CLI REPL with BiomedCore + TrialCore tools wired up). The engineer-facing complement to this web-app kit.

---

## Hand-off

Build, lint, and typecheck must pass.

**Credentials.** Get your `AMASS_API_KEY` from https://platform.amass.tech and add it to `.env`. Do NOT use `AskUserQuestion` to gate scaffolding on credentials — Lovable's (or any AI builder's) standard env-var prompt covers this at run time. Just scaffold the app with placeholder `.env`, then run `npm run dev` once the key is in place.

Then present the hand-off summary. The four verification steps bind to the GLP-1 RA in obesity illustrative SR scope `[identifier-verification caveat — example identifiers assembled without live web access; re-verify against PubMed before binding to a real SR workflow]`:

> **Your SR Pre-Screen Skill is ready.**
>
> Before first push (if your AI builder left scaffolding placeholders): remove any `data-lovable-blank-page-placeholder="REMOVE_THIS"` attributes, any `<img>` tags pointing at `cdn.gpteng.co` or similar builder-placeholder CDNs, and any leftover "your app will live here" boilerplate. These render invisibly but pollute the DOM and serve no purpose in production. Search the codebase for `REMOVE_THIS` and `gpteng` before committing.
>
> To run it: fill in `.env` and run `npm run dev`.
>
> Verify it works:
> - Click **Try sample** to load the GLP-1 RA in obesity illustrative PMID set (~10 representative PMIDs covering semaglutide / liraglutide / tirzepatide in obesity / weight-loss endpoints). Click **Run pre-screen** with the default thresholds (`min_jufo: 2`, `allow_retracted: false`, `min_citation_count: 5`). Confirm the audit-trail table renders ~10 rows with `isRetracted` + `journalQualityJufo` + `citationCount` populated from live Amass (not hardcoded) and `included_in_prescreen` set per the threshold conjunction.
> - Tighten the threshold to `min_jufo: 3` and re-run. Confirm any PMID whose returned `journalQualityJufo` is 2 (or null) flips to `included_in_prescreen=false` in the audit trail; the filtered-paper count drops; the dropped rows still appear in the audit trail with `included_in_prescreen=false` (audit-trail completeness preserved — never silent omission).
> - Click **Download RIS**. Confirm the file emits in Rayyan-importable format (`TY=JOUR` per record + `PMID` + `TI` + `AU` + `JO` + `JF` + `PY` + `DO` + `AB` + `ER`); open in Rayyan or Covidence to confirm the import succeeds end-to-end. Click **Download audit CSV**; confirm the 11-column shape `pmid, accessed_at, AMBC_id, isRetracted, journalQualityJufo, citationCount, lookup_error, included_in_prescreen, threshold_min_jufo, threshold_min_citation_count, threshold_allow_retracted` renders verbatim (12 columns when `include_referencesTrialCore` opt-in is enabled).
> - Paste an intentionally-invalid PMID `99999999` into the textarea alongside the GLP-1 RA sample; click **Run pre-screen**. Confirm the audit-trail row for `99999999` renders inline with `lookup_error` populated by the verbatim upstream string (e.g. `"Unknown PMID"`) per Hard Rule #11 and `included_in_prescreen=false`, without crashing the surface and without omitting the row.
>
> Want to extend it?
> - Add a monthly re-audit cron — re-resolves each `AMBC_` via the original PMID and flags newly-retracted records between SR publication (t0) and SR update cycle (tN); surfaces in the SR update workflow as a "X papers in your published bibliography were retracted since the last cycle" alert. Anchors to canonical `AMBC_` IDs so the audit chain is reconstructible at any prior cycle.
> - Add BiomedCore-search-driven in-tool discovery — exposes `GET /api/v1/cores/biomedcore/records?query=...&minJournalQualityJufo=2&isRetracted=false&minPublicationDate=...` as an in-tool seed-PMID-list generator for SR teams that want to bootstrap an upstream-search-equivalent corpus inside the tool. Out of scope for v0.1 — gated on additional Amass API support.
> - Add the cross-core walk sub-workflow via `include=referencesTrialCore` on the per-paper GET — for SRs whose inclusion criteria intersect literature with a target trial set (e.g. SR scope "RCTs of semaglutide in obesity where the published paper references a registered NCT"). Renders the `referencesTrialCore_AMTCs` column in the audit CSV; empty arrays for review-article papers render as honest emptiness. If your scaffold trimmed `lib/amass.ts` to only the methods this v0.1 actually calls (`batchLookupBiomed` + `getBiomedRecord` + `mapLookupResult`), re-add `getTrialRecord` and optionally `batchLookupTrial` / `resolvePmid` / `resolveDoi` / `resolveNct` helpers to walk papers→trials and back.
> - I'm done — just show me the summary
>
> Have questions? Join the Amass Developer Community on Discord: https://discord.com/invite/sEGaBHMhWa

---

## License

Apache-2.0. Copyright 2026 Amass Technologies. Source at https://github.com/lluisDTU/amass-technologies-amass-starters.
