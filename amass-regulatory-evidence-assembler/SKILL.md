---
name: amass-regulatory-evidence-assembler
description: Use when building an auditable literature-evidence assembler for FDA / EMA regulatory submissions (CTD Module 2.5 / 2.7). Provides build instructions for lookup-and-walk audit-chain assembly, paper→trial cross-core anchoring, and trust-filter rendering in a TypeScript app (Lovable typically scaffolds TanStack Start; Next.js App Router is the alternative path — both work end-to-end).
license: Apache-2.0
metadata:
  author: amass
  version: "0.2.4"
---

# Regulatory Evidence Assembler (Amass Cross-core)

Build a working auditable literature-evidence assembler on the Amass API. The user is a medical writer at an evidence-synthesis CRO (e.g. Arriello / Masuu Global) who just got Amass API credentials and wants a TypeScript audit-chain tool running locally in under five minutes. Lovable typically scaffolds TanStack Start (TanStack Router + `createServerFn` server-functions); Next.js (App Router with `app/api/<endpoint>/route.ts`) is the alternative path. Both work end-to-end against the same `lib/amass.ts` client pattern — the Amass primitives are stack-agnostic.

The CRO operator pastes a curated PMID/DOI/NCT list exported from upstream discovery (PubHive / Embase / Scopus); the tool resolves each external identifier to a canonical `AMBC_` / `AMTC_` Amass ID, walks the paper→trial cross-core edge per paper (`include=referencesTrialCore,citedBy`), and emits the audit-CSV row set that becomes Appendix B of the submission package — anchoring each cited paper to the supporting trial via canonical `AMTC_` IDs that survive NCT-registry revisions between submission (t0) and approval (tN). The worked example binds to the Tarlatamab (Imdelltra) / Amgen BLA 761344-dlle submission scope with the verified identifier set Ahn MJ et al. NEJM 2023 (PMID 37861218) + Rudin et al. J Hematol Oncol 2023 (PMID 37355629) + DeLLphi-301 (NCT05060016) + DeLLphi-304 (NCT05740566). `[Tarlatamab anchor set verified against PubMed + ClinicalTrials.gov + FDA approval letter at draft time]`

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

Pin with caret ranges (`^x.y.z`) for compatible patch updates. `zod` is pinned because the submission-scope schema (`SubmissionScopeSchema`) is the request contract — silent zod-version drift in default-value or refine semantics shifts what the route accepts. `server-only` is pinned because it is the build-time enforcement Hard Rule #2 depends on (works identically in Next.js and TanStack Start). The per-item-error verbatim preservation (Hard Rule #11) and the `lookup_error` vs `walk_error` column split (Hard Rule #10) are load-bearing for the auditable-chain semantics; do not paraphrase upstream error strings or collapse the error columns.

After scaffolding, run `npm install`. If any package fails to resolve, check `node_modules/<pkg>/package.json` for the latest stable version and update accordingly.

**Three notes for AI builders that may produce either path.** (1) The route file location differs: TanStack Start uses `src/lib/<name>.functions.ts` with `createServerFn({ method: "POST" }).inputValidator(...).handler(...)`; Next.js App Router uses `app/api/<endpoint>/route.ts` with `export async function POST(req: Request)`. The reference snippet below is shown in Next.js shape — if your builder produces TanStack Start, translate the `POST(req)` handler to `createServerFn(...).handler(async ({ data }) => ...)` and the `req.json()` body parse to `.inputValidator((input) => SubmissionScopeSchema.parse(input))`. (2) The page file location differs: TanStack Router uses `src/routes/index.tsx` with `createFileRoute("/")`; Next.js App Router uses `app/page.tsx`. Either way, the page calls into the server function / route handler the same way at the network level. (3) **The outer `try/catch` wrapping the handler body MUST be preserved when translating to TanStack Start.** The Next.js reference snippet wraps the audit logic in `try { ... } catch (err) { return Response.json({error: ...}, {status: 500}) }`. For TanStack Start, wrap the `.handler()` body in `try { ... } catch (err) { throw new Error(err instanceof Error ? err.message : "Audit run failed"); }` — the throw surfaces as `mutation.error.message` on the client (TanStack Query's `useMutation` `isError` path) and renders in the UI's inline error region. Without this wrap, an uncaught throw from any API call (e.g., a 4xx response) propagates to TanStack Router's route-level error boundary and crashes the whole page instead of surfacing as a mutation error. Load-bearing for the kit's "per-item error renders inline without crashing the surface" claim.

---

## What the app should do

The primary analyst-visible payoff is the **audit-chain CSV row** — one row per cited paper, anchoring it to its supporting trial via the canonical `AMBC_` and `AMTC_` Amass IDs. The load-bearing Amass primitive is canonical-ID stability: where competing data sources expose the NCT directly as the trial identifier (propagating registry-revision drift between submission and approval), the audit-CSV `referencesTrialCore_AMTCs` column carries an `AMTC_` that survives NCT-registry reassignment, withdrawal-and-reissue, or registry merge between submission (t0) and approval (tN). Each row renders the per-paper trust-filter context (`isRetracted`, `journalQualityJufo`, `citationCount`) in IBM Plex Mono next to the canonical IDs.

The input surface is a **submission-scope YAML paste** — a textarea accepting a `submission_id` + `drug` + `client_sponsor` + `indication` + `pmids[]` + `dois[]` + `ncts[]` block. The empty-state "Try sample" loads the verified Tarlatamab worked example verbatim — `submission_id: BLA-761344-PSUR-2026Q1; drug: tarlatamab; client_sponsor: Amgen; pmids: [37861218, 37355629]; ncts: [NCT05060016, NCT05740566]`. The tooltip on the Try-sample button reads "Tarlatamab (Imdelltra) / Amgen — Ahn NEJM 2023 + Rudin J Hematol Oncol 2023; DeLLphi-301 + DeLLphi-304. `[Tarlatamab anchor set verified against PubMed + ClinicalTrials.gov + FDA approval letter at draft time]`".

The audit run fans out per-paper GETs after batch lookup — typically 200-500 papers per submission scope. Wall-time on a 320-paper happy-path is ~6 min on single-flight concurrency at the 60 req / 60 s ceiling; the route wraps the per-paper fan-out generator in `withIdleTimeout(gen, 180_000)` per Hard Rule #7 and the client renders an analyst-visible **Cancel** button next to the primary "Run audit" action while the fan-out is in flight. A progress indicator increments per resolved paper ("audit run in progress — N of M papers walked"); a stalled Amass call past the 180s idle threshold surfaces a "stream stalled — an Amass call likely hung" error without locking the UI indefinitely.

The final state is a **downloadable audit-chain CSV** with the 9-column shape `accessed_at, input_identifier, resolved_amassId, isRetracted, journalQualityJufo, citationCount, referencesTrialCore_AMTCs, lookup_error, walk_error`, plus a "copy citations to clipboard" button that emits NLM-format citations for the resolved papers. The v0.1 returns the assembled rows in-memory in a single consolidated response — there is no persistence layer, no monthly re-audit cron, and no retraction-review-queue ack-event lifecycle in v0.1 (those land under "Want to extend it?" in Hand-off). Render PMIDs / DOIs / NCTs / `AMBC_` / `AMTC_` IDs in IBM Plex Mono; prose in Inter (Brand reference). Use Lucide React for icons. Lovable produces the UI shape from this spec — variability across builders is fine; the load-bearing constraint is that the four paragraphs above demonstrate the canonical-ID audit chain, the paper→trial cross-core walk, the per-item-error column split, and the worked-example anchor.

---

## Hard rules

The 8 Amass-universal rules below are inherited verbatim across all 6 prototype SKILL.mds. Per-prototype rules 9-13 cover the load-bearing UX / error / discipline surfaces specific to the regulatory-evidence-assembler workflow.

1. **Credentials never enter committed code.** Generate `.env` and `.gitignore` it. Only ONE secret is asked of the user at hand-off:
   ```env
   AMASS_API_KEY=your_amass_api_key_here
   ```
   Per Amass API conventions: the key reads from `AMASS_API_KEY`. The base URL is **hardcoded** to `https://api.amass.tech` in `lib/amass.ts` (single production base; no per-user variance). To point at staging or a local proxy for development, edit the `AMASS_API_BASE_URL` constant in `lib/amass.ts` directly — do NOT re-introduce an env var for this (adding `process.env.AMASS_API_BASE_URL` to `lib/amass.ts` causes Lovable's secrets-UX to prompt the user for a value they don't have, which is unnecessary friction). Never hardcode the API key, never log it, never commit it.

2. **Server-only `AmassClient`.** Lives in `lib/amass.ts` with `import "server-only"` at the top. Never imported from a `"use client"` component. The browser only ever speaks to `/api/<endpoint>`. The Bearer token must never reach the browser bundle; the `server-only` package enforces this at build time.

3. **Singleton/cached `AmassClient`.** Created once on server boot via `getAmassClient()` factory, cached via `clientPromise ??=` pattern. Per-request instantiation is wasted work; the Bearer header is computed once at module-import time and reused.

4. **`Authorization: Bearer ${AMASS_API_KEY}` on every Amass request.** On `401` / `403`, surface the credential error directly — retry won't fix it. On `429`, read `Retry-After` and back off exponentially (per the `lib/amass.ts` snippet below). Rate limit is **60 requests / 60 seconds, per user+org** — the per-call retry helper IS the rate-limit defence for the per-submission 200-500-paper audit-chain fan-out.

5. **Read from the `data` key; handle errors at top-level AND per-item.** Two distinct error shapes: top-level (any non-2xx) — `{"error": {"status": ..., "code": ..., "message": ...}}` — caught by the route's outer `try/catch`. Per-item (`POST /records/lookup` returns HTTP 200 with body `{"data": [...]}` — the universal envelope applies here too) — each array element echoes the input alongside the result: `{"input": {...}, "amassIds": ["AMBC_..."]}` for success, or `{"input": {...}, "error": {"code": "NOT_FOUND", "message": "Identifier not found"}}` for failure (error is a structured object with `code` + `message` fields, NOT a bare string — empirically verified via direct curl on 2026-05-26). The `lib/amass.ts` `batchLookupBiomed` / `batchLookupTrial` methods unwrap the `data` envelope; the `mapLookupResult` helper extracts `error.message` to a string for caller-side rendering, preserving positional index alignment with the input items. **Never assume HTTP 200 means every item succeeded. Never render `error` as a React child directly — extract `.message` first or React #31 crashes the surface.**

6. **Lookup endpoints translate PMID/DOI/NCT → canonical `AMBC_`/`AMTC_` before fetching by ID.** Raw HTTP `GET /records/{id}` accepts **only** canonical Amass IDs — passing a PMID directly returns 404. The workflow is always: `POST /records/lookup` with `[{pmid: "..."}]` → read `amassIds[0]` (handling per-item-error per Rule #5) → `GET /records/{amassId}`. The `lib/amass.ts` `resolvePmid` / `resolveDoi` / `resolveNct` helpers encode this discipline.

7. **Idle-timeout + analyst-visible cancel button on long-running fan-outs.** Three requirements: (a) the route wraps any fan-out async generator in `withIdleTimeout(gen, 180_000)` — if no chunk arrives for 180s, the wrapper throws; (b) the client renders an analyst-visible **Cancel** button next to the primary action while a fan-out is in flight; (c) the route wraps body parse + client setup in an outer `try/catch` returning `Response.json({error}, {status: 500})`. Without all three, a single stalled Amass call locks the UI indefinitely on a multi-minute audit-chain run.

8. **Ground-truth discipline.** Every URL, type, function name, version, and config shape in this SKILL.md must come from a source the agent can read — a fetched doc page, a `.d.ts` from the installed SDK, or the installed package's exports. If you can't point to a source, don't write it. Forbidden: fabricating endpoints; guessing field names; `as unknown as ...` or `as any` on SDK values; ignoring `tsc` errors. When the runtime contradicts what's written here, fix the cause.

### Per-prototype hard rules

9. **Lookup-then-fetch for canonical-ID resolution.** The audit-chain v0.1 path is always `POST /records/lookup` with PMID/DOI/NCT → read `amassIds[0]` → `GET /records/{amassId}`. The audit-CSV `resolved_amassId` column carries the canonical `AMBC_` / `AMTC_`, never the input identifier. Rationale: the workflow's load-bearing value is anchoring citations to canonical IDs that survive registry revisions between submission (t0) and approval (tN) — using the input PMID/DOI/NCT in the audit chain defeats the canonical-ID stability moat.

10. **`lookup_error` vs `walk_error` column split — never collapse.** `lookup_error` populates from lookup-side errors on the `POST /records/lookup` calls (PMID retired upstream, DOI revoked, NCT registry merge — the input identifier never canonicalised). `walk_error` populates from per-record `GET /records/{amassId}` errors after a successful lookup (a backend inconsistency or transient backend issue surfaced on the per-record GET despite a clean lookup). Rationale: these are structurally different audit-failure shapes with different remediation paths — the auditable-gap-vs-audit-failure distinction is the load-bearing per-item-error semantic that competitor stacks (PubMed E-utilities' silent absence, OpenAlex's silent omission, Semantic Scholar's null sentinels) do not surface as a single API contract.

11. **Per-item-error verbatim preservation.** Surface the upstream-generated per-item error string verbatim in the `lookup_error` column — never paraphrased, never elided, never normalised. The `mapLookupResult` helper in `lib/amass.ts` enforces this by passing the raw `r.error` string through unchanged. Rationale: for a contractually-auditable chain (per ICH guidelines for regulatory submission documentation), "this PMID is unresolved with reason X" vs "this PMID is silently absent" is the difference between an auditable gap and an audit failure; paraphrasing the upstream reason string collapses the two.

12. **Paper→trial direction only on the cross-core walk (v0.1).** v0.1 sets `include=referencesTrialCore` (plus `include=citedBy` as an auxiliary provenance signal) on the paper-side BiomedCore GET; the trial-side-driven "describe-this-trial" walk (NCT → all supporting publications) is explicitly foreclosed because at-scale symmetry of the cross-core spine is not yet a published guarantee. Empty `referencesTrialCore` arrays (review-article papers — e.g. the worked example's PMID 37355629) render as honest emptiness — `referencesTrialCore_AMTCs = []` with an explicit "no cited trial in cross-core spine" sidecar note — not as a workflow halt. Rationale: foreclosing the reverse direction keeps v0.1 within published guarantees while the audit chain itself (cited-paper → supporting-trial direction) is complete.

13. **Worked-example anchoring (verify-then-ship-strictly).** The "Try sample" YAML in the empty state binds verified Tarlatamab identifiers **verbatim** — BLA 761344-dlle / NCT05060016 (DeLLphi-301) / NCT05740566 (DeLLphi-304) / PMID 37861218 (Ahn MJ et al. NEJM 2023, verified `isRetracted=false`) / PMID 37355629 (Rudin et al. J Hematol Oncol 2023, verified `isRetracted=false`) / DOI 10.1056/NEJMoa2307980 / FDA accelerated approval 2024-05-16. The `isRetracted` column in every rendered row populates from the **live Amass record at run time**, never from a hardcoded fixture, never from a real-PMID-flipped-to-retracted-for-illustration — because the display=claim rule for regulatory audit-chain rendering is defamation-adjacent: an unsupported retraction display IS a retraction claim against the named work. The live-Amass-binding requirement structurally preserves the guard (Lovable scaffolds against the live API, not against fabricated retraction data). Carry the caveat marker `[Tarlatamab anchor set verified against PubMed + ClinicalTrials.gov + FDA approval letter at draft time]` at three anchor sites: the title intro paragraph, the Try-sample tooltip, and the Hand-off verification preamble. Rationale: the worked example was previously confabulated on first attempt (multiple identifier mismatches against ground truth) — the verified-at-draft-time marker keeps the verification surface visible while the live-Amass-binding handles run-time accuracy.

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

  // Audit-chain default includes — paper→trial cross-core spine (referencesTrialCore) +
  // auxiliary provenance signal (citedBy). Caller may override; v0.1 does NOT include `fulltext`.
  async getBiomedRecord(
    amassId: string,
    includes: ReadonlyArray<"fulltext" | "authorsMetadata" | "meshIds" | "substanceIds" | "referencesTrialCore" | "references" | "citedBy"> = ["referencesTrialCore", "citedBy"],
  ): Promise<unknown /* bind to @amass/sdk BiomedRecord type once installed */> {
    const qs = includes.length > 0 ? `?${includes.map((i) => `include=${i}`).join("&")}` : "";
    return this.unwrapData(this.request<{ data: unknown }>("GET", `/api/v1/cores/biomedcore/records/${amassId}${qs}`));
  }

  // TrialCore get-by-ID — used to render the trial-side card when the analyst clicks through an AMTC_
  // anchor in the audit-CSV row.
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

**`app/api/audit-run/route.ts`** — Next.js App Router POST handler. Takes a submission-scope JSON body, runs Call 1 (`batchLookupBiomed` on PMIDs + DOIs) + Call 2 (`batchLookupTrial` on NCTs) concurrently, then fans out Call 3 (`getBiomedRecord` per resolved `AMBC_`) wrapped in `withIdleTimeout(gen, 180_000)` per Hard Rule #7. Returns the assembled audit-CSV rows + the `lookup` / `walk` error counts in one consolidated JSON response. Non-streaming for v0.1.

```ts
import { AmassClient, getAmassClient } from "@/lib/amass";

interface SubmissionScope {
  submission_id: string;
  drug: string;
  indication: string;
  client_sponsor: string;
  pmids?: string[];
  dois?: string[];
  ncts?: string[];
}

interface AuditRow {
  accessed_at: string;
  input_identifier: string;       // "PMID:37861218" | "DOI:10.1056/..." | "NCT:NCT05060016"
  resolved_amassId: string | null;
  isRetracted: boolean | null;
  journalQualityJufo: number | null;
  citationCount: number | null;
  referencesTrialCore_AMTCs: string[];
  lookup_error: string | null;    // per Hard Rule #10 + #11 — verbatim upstream string
  walk_error: string | null;      // per Hard Rule #10 — distinct from lookup_error
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

// Per Hard Rule #12 — paper→trial direction only; review papers surface empty arrays as honest emptiness.
function extractWalk(record: unknown): { isRetracted: boolean | null; jufo: number | null; cites: number | null; refsTC: string[] } {
  const r = record as {
    isRetracted?: boolean | null;
    journalQualityJufo?: number | null;
    citationCount?: number | null;
    referencesTrialCore?: string[];
  };
  return {
    isRetracted: r.isRetracted ?? null,
    jufo: r.journalQualityJufo ?? null,
    cites: r.citationCount ?? null,
    refsTC: r.referencesTrialCore ?? [],
  };
}

export async function POST(req: Request) {
  try {
    const scope = (await req.json()) as SubmissionScope;
    const client = await getAmassClient();
    const accessed_at = new Date().toISOString();

    // Call 1 + Call 2 — concurrent batch lookups (Hard Rule #6 + #9 + #11).
    const pmidItems = (scope.pmids ?? []).map((pmid) => ({ pmid }));
    const doiItems = (scope.dois ?? []).map((doi) => ({ doi }));
    const nctItems = (scope.ncts ?? []).map((nctId) => ({ nctId }));

    const [pmidResults, doiResults, nctResults] = await Promise.all([
      pmidItems.length > 0 ? client.batchLookupBiomed(pmidItems) : Promise.resolve([]),
      doiItems.length > 0 ? client.batchLookupBiomed(doiItems) : Promise.resolve([]),
      nctItems.length > 0 ? client.batchLookupTrial(nctItems) : Promise.resolve([]),
    ]);

    // Assemble lookup-result rows first; resolved AMBC_/AMTC_ feed the Call 3 fan-out.
    // Per Hard Rule #10: lookup-side errors populate lookup_error (NOT walk_error).
    const rows: AuditRow[] = [];
    const amassIdsToWalk: string[] = [];

    AmassClient.mapLookupResult(
      pmidItems,
      pmidResults,
      ({ pmid }, amassIds) => {
        amassIdsToWalk.push(amassIds[0]);
        rows.push({ accessed_at, input_identifier: `PMID:${pmid}`, resolved_amassId: amassIds[0], isRetracted: null, journalQualityJufo: null, citationCount: null, referencesTrialCore_AMTCs: [], lookup_error: null, walk_error: null });
        return null;
      },
      ({ pmid }, msg) => {
        rows.push({ accessed_at, input_identifier: `PMID:${pmid}`, resolved_amassId: null, isRetracted: null, journalQualityJufo: null, citationCount: null, referencesTrialCore_AMTCs: [], lookup_error: msg, walk_error: null });
        return null;
      },
    );
    AmassClient.mapLookupResult(
      doiItems,
      doiResults,
      ({ doi }, amassIds) => {
        amassIdsToWalk.push(amassIds[0]);
        rows.push({ accessed_at, input_identifier: `DOI:${doi}`, resolved_amassId: amassIds[0], isRetracted: null, journalQualityJufo: null, citationCount: null, referencesTrialCore_AMTCs: [], lookup_error: null, walk_error: null });
        return null;
      },
      ({ doi }, msg) => {
        rows.push({ accessed_at, input_identifier: `DOI:${doi}`, resolved_amassId: null, isRetracted: null, journalQualityJufo: null, citationCount: null, referencesTrialCore_AMTCs: [], lookup_error: msg, walk_error: null });
        return null;
      },
    );
    AmassClient.mapLookupResult(
      nctItems,
      nctResults,
      ({ nctId }, amassIds) => {
        rows.push({ accessed_at, input_identifier: `NCT:${nctId}`, resolved_amassId: amassIds[0], isRetracted: null, journalQualityJufo: null, citationCount: null, referencesTrialCore_AMTCs: [], lookup_error: null, walk_error: null });
        return null;
      },
      ({ nctId }, msg) => {
        rows.push({ accessed_at, input_identifier: `NCT:${nctId}`, resolved_amassId: null, isRetracted: null, journalQualityJufo: null, citationCount: null, referencesTrialCore_AMTCs: [], lookup_error: msg, walk_error: null });
        return null;
      },
    );

    // Call 3 — fan out per-paper GETs on the resolved AMBC_ set. Wrapped in withIdleTimeout per Hard Rule #7.
    // Per Hard Rule #12: include=referencesTrialCore (paper→trial direction) + include=citedBy (auxiliary).
    async function* walkPapers(): AsyncGenerator<{ amassId: string; walk: ReturnType<typeof extractWalk> | null; err: string | null }> {
      for (const amassId of amassIdsToWalk) {
        try {
          const record = await client.getBiomedRecord(amassId, ["referencesTrialCore", "citedBy"]);
          yield { amassId, walk: extractWalk(record), err: null };
        } catch (e) {
          yield { amassId, walk: null, err: e instanceof Error ? e.message : "walk failed" };
        }
      }
    }

    for await (const { amassId, walk, err } of withIdleTimeout(walkPapers(), 180_000)) {
      const row = rows.find((r) => r.resolved_amassId === amassId);
      if (!row) continue;
      if (err) {
        // Per Hard Rule #10 — walk-side failure goes to walk_error column, NOT lookup_error.
        row.walk_error = err;
        continue;
      }
      row.isRetracted = walk!.isRetracted;
      row.journalQualityJufo = walk!.jufo;
      row.citationCount = walk!.cites;
      row.referencesTrialCore_AMTCs = walk!.refsTC;
    }

    return Response.json({
      rows,
      errors: {
        lookup: rows.filter((r) => r.lookup_error).length,
        walk: rows.filter((r) => r.walk_error).length,
      },
    });
  } catch (err) {
    return Response.json({ error: err instanceof Error ? err.message : "Audit run failed" }, { status: 500 });
  }
}
```

The two snippets above are the only `lib/<file>.ts` + `app/api/<route>/route.ts` mandated by this prototype. v0.1 returns the assembled audit-CSV rows in-memory in a single consolidated response — there is no persistence module, no `lib/audit-store.ts`, no `app/api/re-audit/route.ts`, no Python CLI sidecar. Download / copy-to-clipboard happens client-side from the response payload. If a downstream extension needs persistence (monthly re-audit history, ack-event lifecycle), the extension SKILL.md includes its own minimal stub per the template's policy — never an `import` from `@/lib/<x>` without a matching snippet.

---

## Required reading

- `https://platform.amass.tech` — Amass Platform docs + API credentials
- `https://github.com/amass-technologies/public-amass-platform-starter-ts` — official Amass Platform Starter (Apache-2.0; Bun + Ink CLI REPL with BiomedCore + TrialCore tools wired up). The engineer-facing complement to this web-app kit.

---

## Hand-off

Build, lint, and typecheck must pass.

**Credentials.** Get your `AMASS_API_KEY` from https://platform.amass.tech and add it to `.env`. Do NOT use `AskUserQuestion` to gate scaffolding on credentials — Lovable's (or any AI builder's) standard env-var prompt covers this at run time. Just scaffold the app with placeholder `.env`, then run `npm run dev` once the key is in place.

Then present the hand-off summary. The four verification steps bind to the Tarlatamab worked-example identifier set `[Tarlatamab anchor set verified against PubMed + ClinicalTrials.gov + FDA approval letter at draft time]`:

> **Your Regulatory Evidence Assembler is ready.**
>
> Before first push (if your AI builder left scaffolding placeholders): remove any `data-lovable-blank-page-placeholder="REMOVE_THIS"` attributes, any `<img>` tags pointing at `cdn.gpteng.co` or similar builder-placeholder CDNs, and any leftover "your app will live here" boilerplate. These render invisibly but pollute the DOM and serve no purpose in production. Search the codebase for `REMOVE_THIS` and `gpteng` before committing.
>
> To run it: fill in `.env` and run `npm run dev`.
>
> Verify it works:
> - Click **Try sample** to load the Tarlatamab YAML (`submission_id: BLA-761344-PSUR-2026Q1; pmids: [37861218, 37355629]; ncts: [NCT05060016, NCT05740566]`); click **Run audit**. Confirm the rendered rows show `isRetracted=false` for both PMIDs (live values from Amass — not hardcoded), `journalQualityJufo=3` on PMID 37861218 (NEJM is a JuFo tier-3 journal), and `referencesTrialCore_AMTCs` populated with the DeLLphi-301 `AMTC_` ID on PMID 37861218's row.
> - Click the linked `AMTC_` ID for DeLLphi-301; confirm the trial card renders with `NCT05060016` displayed in IBM Plex Mono next to the trial title and Phase 2 status.
> - Click **Download audit CSV**; confirm the file emits with the 9-column shape `accessed_at, input_identifier, resolved_amassId, isRetracted, journalQualityJufo, citationCount, referencesTrialCore_AMTCs, lookup_error, walk_error` and that PMID 37355629's row carries an empty `referencesTrialCore_AMTCs` field (review article — honest emptiness per Hard Rule #12).
> - Paste an intentionally-invalid PMID `99999999` into the scope textarea; click **Run audit**; confirm the row renders inline with `lookup_error` populated by the verbatim upstream string (e.g. `"Unknown PMID"`) per Hard Rule #11, without crashing the surface and without populating `walk_error`.
>
> Want to extend it?
> - Add a monthly re-audit cron with append-only audit-CSV history — surfaces newly-retracted citations between submission (t0) and approval (tN) over the PSUR / approval lifecycle, anchored to canonical `AMBC_` IDs so the audit chain remains reconstructible at any prior cycle.
> - Add a Python CLI sidecar (`python_cli/audit_run.py`) for headless cron invocation — same `lib/amass.ts` semantics, called from a cron environment rather than the dev server.
> - Add a retraction review queue panel with `(acked_at, acked_by, ack_decision)` ack-event lifecycle — durably-persisted across worker restarts, ack-without-decision-metadata rejected, ack-after-re-flag preserves both events.
> - I'm done — just show me the summary
>
> Have questions? Join the Amass Developer Community on Discord: https://discord.com/invite/sEGaBHMhWa

---

## License

Apache-2.0. Copyright 2026 Amass Technologies. Source at https://github.com/lluisDTU/amass-technologies-amass-starters.
