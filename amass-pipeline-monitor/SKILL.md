---
name: amass-pipeline-monitor
description: Use when building a weekly CI-analyst delivery (MCP slash-command + Markdown digest) that takes a sponsor watchlist as input and surfaces new Phase 2/3 trials, new published evidence describing those trials, and retraction-flagged citations. Provides build instructions for the per-sponsor TrialCore search + trial→paper cross-core walk + per-paper trust-filter resolution + 3-panel weekly dashboard surface in a Next.js+TS app.
license: Apache-2.0
metadata:
  author: amass
  version: "0.1.0"
---

# Pipeline Monitor CI Agent (Amass Cross-core)

Build a working weekly CI-analyst delivery (MCP slash-command + Markdown digest + 3-panel dashboard) on the Amass API. The user is a **competitive-intelligence analyst at a mid-cap immuno-oncology biotech** who just got Amass API credentials and wants a per-sponsor weekly pipeline digest — new Phase 2/3 trials, new published evidence describing those trials via the trial→paper cross-core edge, retraction-flagged citations — running locally in **under five minutes**.

The analyst pastes a YAML sponsor watchlist (canonical sponsors + indication-cluster scope) and clicks **Run digest**; the app issues a per-sponsor TrialCore search filtered to Phase 2/3 + recent `minStartDate`, walks each surfaced trial's `referencesBiomedCore` cross-core edge, resolves per-paper trust signals (`isRetracted`, `journalQualityJufo`, `citationCount`), and emits a Markdown digest grouped by sponsor — rendered inline as a 3-panel weekly dashboard. The worked example anchors on a **SCLC-DLL3-2026Q2** watchlist (Amgen + Roche + AbbVie + Boehringer Ingelheim + Harpoon Therapeutics) bound to the verified Tarlatamab identifier set: pivotal **DeLLphi-301 = NCT05060016**, confirmatory **DeLLphi-304 = NCT05740566**, primary publication **PMID 37861218** (Ahn MJ et al., NEJM 2023). `[Tarlatamab anchor set verified against PubMed + ClinicalTrials.gov + FDA approval letter at draft time]`

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

Pin with caret ranges (`^x.y.z`) for compatible patch updates. `zod` is pinned because the watchlist-scope schema (`WatchlistSchema`) is the request contract — silent zod-version drift in default-value or refine semantics shifts what the route accepts. `server-only` is pinned because it is the build-time enforcement Hard Rule #2 depends on (works identically in Next.js and TanStack Start). The per-item-error verbatim preservation (Hard Rule #11) and the 3-panel digest decomposition (Hard Rule #10) are load-bearing for the dashboard semantics; do not paraphrase upstream error strings or collapse the panel decomposition.

After scaffolding, run `npm install`. If any package fails to resolve, check `node_modules/<pkg>/package.json` for the latest stable version and update accordingly.

**Three notes for AI builders that may produce either path.** (1) The route file location differs: TanStack Start uses `src/lib/<name>.functions.ts` with `createServerFn({ method: "POST" }).inputValidator(...).handler(...)`; Next.js App Router uses `app/api/<endpoint>/route.ts` with `export async function POST(req: Request)`. The reference snippet below is shown in Next.js shape — if your builder produces TanStack Start, translate the `POST(req)` handler to `createServerFn(...).handler(async ({ data }) => ...)` and the `req.json()` body parse to `.inputValidator((input) => WatchlistSchema.parse(input))`. (2) The page file location differs: TanStack Router uses `src/routes/index.tsx` with `createFileRoute("/")`; Next.js App Router uses `app/page.tsx`. Either way, the page calls into the server function / route handler the same way at the network level. (3) **The outer `try/catch` wrapping the handler body MUST be preserved when translating to TanStack Start.** The Next.js reference snippet wraps the audit logic in `try { ... } catch (err) { return Response.json({error: ...}, {status: 500}) }`. For TanStack Start, wrap the `.handler()` body in `try { ... } catch (err) { throw new Error(err instanceof Error ? err.message : "Digest run failed"); }` — the throw surfaces as `mutation.error.message` on the client (TanStack Query's `useMutation` `isError` path) and renders in the UI's inline error region. Without this wrap, an uncaught throw from any API call propagates to TanStack Router's route-level error boundary and crashes the whole page instead of surfacing as a mutation error. Load-bearing for the kit's "per-item error renders inline without crashing the surface" claim.

---

## What the app should do

The primary analyst-visible payoff is a **3-panel weekly dashboard** rendered inline as the `/api/digest-run` response: **new trials this week** (left panel — surfaced via per-sponsor TrialCore search filtered to Phase 2/3 + `minStartDate=<last_check>`); **new papers describing watched trials** (center panel — surfaced via the trial→paper cross-core walk on `include=referencesBiomedCore` per surfaced trial); **retraction-flagged citations** (right panel — populated when any walked paper carries `isRetracted = true`). The load-bearing Amass primitive the surface demonstrates is the conjunction of **canonical `AMTC_` IDs that survive NCT registry revisions** (a week-over-week diff keyed on `AMTC_` does not drift when CT.gov rewrites the NCT underneath) **plus the trial→paper cross-core edge** (`referencesBiomedCore`) — one Amass call surfaces the paper-side context that on PubMed + CT.gov + OpenAlex requires 3-4 calls across separate entity-key systems.

The input surface is a **YAML sponsor watchlist** with canonical sponsors + indication-cluster scope + cadence. The empty-state shows a **"Try sample"** button loading the SCLC-DLL3-2026Q2 worked example verbatim (`sponsors: [Amgen, Roche, AbbVie, Boehringer Ingelheim, Harpoon Therapeutics]`; `phases: [PHASE2, PHASE2_PHASE3, PHASE3]`; `last_check: 2026-05-13`). PMIDs / DOIs / NCTs / `AMBC_` / `AMTC_` IDs render in IBM Plex Mono throughout; sponsor names and paper titles render in Inter.

**Step 0 and Step 0.5 (executed once at watchlist onboarding).** Sponsor-name canonicalisation pass: issue one `query=<sponsor>` TrialCore search per raw sponsor name; cluster the returned `sponsorName` strings via case-fold + drop common suffixes (`Inc`, `Bio`, `Sciences`, `Therapeutics`, `Pharmaceuticals`, `GmbH`, `Ltd`); present ambiguous clusters with 3-5 sample trials each for analyst-confirm; persist the analyst-confirmed canonical→aliases map into the watchlist YAML. **Genentech ↔ Roche** is the canonical example — Genentech is a Roche subsidiary with its own `sponsorName` string in CT.gov; the analyst confirms during onboarding that Genentech rolls up to Roche for this watchlist. Then Step 0.5 cardinality probe: per canonical sponsor, issue one `query=<canonical>&phase=PHASE2&phase=PHASE2_PHASE3&phase=PHASE3` (no `minStartDate`) and verify the returned array has `< 300` records (the `limit=300` ceiling on the TrialCore search endpoint); sponsors exceeding 300 prompt the analyst to narrow the indication scope before onboarding completes. Both steps are PROSE-described workflow surfaces in the dashboard's onboarding flow — not separate snippet-included routes.

The cron + fan-out + cancel UX runs per-weekly digest. Per-sponsor TrialCore search (5 sponsors × 2 calls for the dual-query date-filter workaround per Hard Rule #10 = 10 calls at start) plus per-trial cross-core walk (~10 new trials per weekly run typical) plus per-paper resolution (~30 unique papers after dedup) totals roughly **50-150 calls per weekly run** with ~2-6 min wall-time on the SCLC-DLL3-2026Q2 watchlist scale. The route wraps the fan-out async generator in `withIdleTimeout(gen, 180_000)` per Hard Rule #7; the dashboard renders an analyst-visible **Cancel** button next to the **Run digest** action while the fan-out is in flight; the final state is a downloadable Markdown digest (`digest-<watchlist>-<YYYY-Wnn>.md`) with copy-citations-to-clipboard buttons on each paper card. **NO digest-history append-only persistence in v0.1** — that lives in the Hand-off "Want to extend it?" follow-on. Lovable produces the dashboard UI shape from this spec; visual variability across builders is fine — the load-bearing constraint is that the 3-panel decomposition (trials / papers / retractions) and the IBM Plex Mono rendering of identifier strings demonstrate the distinctive Amass primitives.

---

## Hard rules

The 8 Amass-universal rules below are inherited verbatim across all 6 prototype SKILL.mds. Per-prototype rules 9-13 follow; each is 2-4 sentences with a one-line rationale covering load-bearing surfaces, not exhaustive coverage.

1. **Credentials never enter committed code.** Generate `.env` and `.gitignore` it. Only ONE secret is asked of the user at hand-off:
   ```env
   AMASS_API_KEY=your_amass_api_key_here
   ```
   Per Amass API conventions: the key reads from `AMASS_API_KEY`. The base URL is **hardcoded** to `https://api.amass.tech` in `lib/amass.ts` (single production base; no per-user variance). To point at staging or a local proxy for development, edit the `AMASS_API_BASE_URL` constant in `lib/amass.ts` directly — do NOT re-introduce an env var for this (adding `process.env.AMASS_API_BASE_URL` to `lib/amass.ts` causes Lovable's secrets-UX to prompt the user for a value they don't have, which is unnecessary friction). Never hardcode the API key, never log it, never commit it.

2. **Server-only `AmassClient`.** Lives in `lib/amass.ts` with `import "server-only"` at the top. Never imported from a `"use client"` component. The browser only ever speaks to `/api/<endpoint>`. The Bearer token must never reach the browser bundle; the `server-only` package enforces this at build time.

3. **Singleton/cached `AmassClient`.** Created once on server boot via `getAmassClient()` factory, cached via `clientPromise ??=` pattern. Per-request instantiation is wasted work; the Bearer header is computed once at module-import time and reused.

4. **`Authorization: Bearer ${AMASS_API_KEY}` on every Amass request.** On `401` / `403`, surface the credential error directly — retry won't fix it. On `429`, read `Retry-After` and back off exponentially (per the `lib/amass.ts` snippet below). Rate limit is **60 requests / 60 seconds, per user+org** — the per-call retry helper IS the rate-limit defence for the per-weekly fan-out.

5. **Read from the `data` key; handle errors at top-level AND per-item.** Two distinct error shapes: top-level (any non-2xx) — `{"error": {"status": ..., "code": ..., "message": ...}}` — caught by the route's outer `try/catch`. Per-item (`POST /records/lookup` returns HTTP 200 with body `{"data": [...]}` — the universal envelope applies here too) — each array element echoes the input alongside the result: `{"input": {...}, "amassIds": ["AMBC_..."]}` for success, or `{"input": {...}, "error": {"code": "NOT_FOUND", "message": "Identifier not found"}}` for failure (error is a structured object with `code` + `message` fields, NOT a bare string — empirically verified via direct curl on 2026-05-26). The `lib/amass.ts` `batchLookupBiomed` / `batchLookupTrial` methods unwrap the `data` envelope; the `mapLookupResult` helper extracts `error.message` to a string for caller-side rendering, preserving positional index alignment with the input items. **Never assume HTTP 200 means every item succeeded. Never render `error` as a React child directly — extract `.message` first or React #31 crashes the surface.**

6. **Lookup endpoints translate PMID/DOI/NCT → canonical `AMBC_`/`AMTC_` before fetching by ID.** Raw HTTP `GET /records/{id}` accepts **only** canonical Amass IDs. The workflow for v0.1's all-from-cross-core path: `searchTrialcore(<sponsor>, ...)` returns `AMTC_` IDs directly (no lookup needed); each trial's `referencesBiomedCore` array returns `AMBC_` IDs directly (no lookup needed). The `resolvePmid` / `resolveDoi` / `resolveNct` helpers are baked into `lib/amass.ts` for analyst-supplied PMID/DOI/NCT seed extensions.

7. **Idle-timeout + analyst-visible cancel button on long-running fan-outs.** Per the kit's handoff Section 6. Three requirements: (a) the route wraps any fan-out async generator in `withIdleTimeout(gen, 180_000)` — if no chunk arrives for 180s, the wrapper throws; (b) the client renders an analyst-visible **Cancel** button next to the **Run digest** action while the digest is in flight; (c) the route wraps body parse + client setup in an outer `try/catch` returning `Response.json({error}, {status: 500})`. Without all three, a single stalled Amass call locks the dashboard indefinitely on the 50-150-call-per-run fan-out.

8. **Ground-truth discipline.** Every URL, type, function name, version, and config shape in this SKILL.md must come from a source the agent can read — a fetched doc page, a `.d.ts` from the installed SDK, or the installed package's exports. If you can't point to a source, don't write it. Forbidden: fabricating endpoints; guessing field names; `as unknown as ...` or `as any` on SDK values; ignoring `tsc` errors. When the runtime contradicts what's written here, fix the cause.

### Per-prototype hard rules

9. **Sponsor-name canonicalisation at watchlist onboarding (Step 0).** Cluster `sponsorName` strings via case-fold + drop common suffixes (`Inc` / `Bio` / `Sciences` / `Therapeutics` / `Pharmaceuticals` / `GmbH` / `Ltd`); present ambiguous clusters with 3-5 sample trials each for analyst-confirm; persist the analyst-confirmed canonical→aliases map into the watchlist YAML. **Genentech ↔ Roche** is the canonical worked example — surface the disambiguation chip in the dashboard whenever Genentech appears in the raw watchlist. **Rationale:** silent N→1 collapse on sponsor name = silent over/under-count of competitor pipeline activity = load-bearing data-quality risk for the analyst; the workflow needs human-in-the-loop confirmation once, not a heuristic guess every week.

10. **`minStartDate` filter-correctness probe (dual-query workaround).** For each per-sponsor TrialCore search, issue the `minStartDate=<last_check>` query AND apply `lastUpdateDate >= <last_check>` + `startDate >= <last_check>` post-filter client-side; diff the two result sets and surface any anomaly into a digest sidecar as an audit entry. Threshold guidance: **`<5/week` is mechanical noise** (sidecar note only); **`5-20/week` triggers an analyst-review sample**; **`>20/week OR >10% per-sponsor` escalates to Amass support**. **Rationale:** phantom-empty digest from a wrong-direction date-semantic erodes the canonical-ID stability moat at runtime; the dual-query is mechanical overhead, not a blocker.

11. **Trial→paper direction only on cross-core walk (v0.1).** v0.1 uses `include=referencesBiomedCore` on the TrialCore-side GET only (Call 2). The paper→trial direction (`referencesTrialCore` on the BiomedCore record) is the regulatory-evidence-assembler's territory, not this prototype's. Honest-frame empty `referencesBiomedCore` arrays (e.g., a brand-new Phase-2 trial registered this week with no published evidence yet) as **honest emptiness** in the center panel, NOT a workflow halt. **Rationale:** foreclosing the reverse direction keeps v0.1 within published API guarantees; at-scale symmetry between trial-side and paper-side cross-core references is not yet a published guarantee and the v0.1 demo does not exercise the symmetry assumption.

12. **Per-item-error verbatim preservation.** Surface any upstream-generated error-reason strings verbatim in the digest's error column — never paraphrased, never elided. The `mapLookupResult` helper in `lib/amass.ts` enforces this. This is load-bearing for analyst-supplied PMID/DOI seed extensions (the lookup endpoint returns per-item error objects that the digest must surface verbatim); non-load-bearing for v0.1's all-`AMBC_`-from-cross-core path; baked in for kit-wide cross-prototype consistency. **Rationale:** the auditable-gap-vs-audit-failure distinction is the load-bearing Amass distinctive — PubMed E-utils silent absence; OpenAlex silent omission; Semantic Scholar null sentinels — Amass surfaces human-readable failure strings the analyst can act on.

13. **Worked-example anchoring (HIGH mixed-mode, verify-then-ship).** The dashboard displays `isRetracted` for cascade-tracked papers AND wraps "retraction-flagged citations" assertion language around them — both display=claim and pure-assertion triggers fire; the worked-example identifier set is verified against authoritative sources before publication. The "Try sample" YAML in the empty state binds:
    - `watchlist_id: SCLC-DLL3-2026Q2`
    - `sponsors: [Amgen, Roche, AbbVie, Boehringer Ingelheim, Harpoon Therapeutics]`
    - Tarlatamab anchor identifiers: **BLA 761344-dlle** / **NCT05060016** (DeLLphi-301, pivotal Phase 2) / **NCT05740566** (DeLLphi-304, confirmatory Phase 3) / **PMID 37861218** (Ahn MJ et al. NEJM 2023; verified `isRetracted=false`) / **PMID 37355629** (Rudin et al. J Hematol Oncol 2023, DLL3-targeted-therapy review; verified `isRetracted=false`) / **DOI 10.1056/NEJMoa2307980** / **FDA accelerated approval 2024-05-16**.
    - **The retraction-cascade panel renders empty (`no retractions this week`) in the SCLC-DLL3-2026Q2 worked example by design** — both walked PMIDs are verified `isRetracted=false`; the panel populates only from LIVE Amass data at digest-run time on any walked paper that is genuinely flagged retracted at that moment; never from a hardcoded fixture, never from a real-PMID-flipped-to-retracted (defamation-adjacent: an unsupported retraction display IS a retraction claim against the named work).
    - Caveat marker at all 3 anchor sites (empty-state YAML; per-paper card example; retraction-cascade panel default-state text): `[Tarlatamab anchor set verified against PubMed + ClinicalTrials.gov + FDA approval letter at draft time]`.

    **Rationale:** the workflow asserts a real paper's retraction status; this is HIGH-stakes mixed-mode and the canonical sequence is verify-then-ship — the worked-example identifier set is verified against authoritative sources before publication rather than shipping with confabulation.

---

## Brand reference

Source of truth for the Amass app look-and-feel. Load before writing `globals.css`.

### Logo

Render the Amass wordmark at the **top-left of the header**, ~24-32px. Use the text `amass` in Inter weight 800 24px as the wordmark. Place it left of the page title (`Pipeline Monitor`).

### Typography

- **Inter** — UI text, labels, headings, body, sponsor names, paper titles. Weights 400/600/700/800/900.
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

Use **Lucide React** for icons (Lucide only). Suggested icons: `Activity` (header), `FlaskConical` (trials panel), `FileText` (papers panel), `AlertTriangle` (retractions panel), `Loader2` (in-flight), `XCircle` (cancel), `Copy` (copy-citations).

---

## Reference snippets — use these

**`lib/amass.ts`** — singleton `AmassClient` with Bearer-auth, 429 retry/backoff (reads `Retry-After`), per-item-error helper (`mapLookupResult`), `{data: ...}` envelope unwrap, and the per-sponsor TrialCore search + per-trial-record + per-paper-record helpers Prototype 2 depends on. Per Hard Rules #2, #3, #4, #5, #6, #9, #12.

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

interface TrialSearchOpts {
  phase?: ReadonlyArray<"PHASE2" | "PHASE2_PHASE3" | "PHASE3">;
  minStartDate?: string;       // ISO YYYY-MM-DD
  sponsorType?: "INDUSTRY";    // optional helper only; not load-bearing
  limit?: number;              // 1-300, default 20; ceiling 300 per the TrialCore search endpoint
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

  // Per-item-error helper per Hard Rule #12.
  // Per empirical observation, each result is either {input?, amassIds: [...]} or
  // {input?, error: {code, message}}. mapLookupResult extracts error.message to a
  // string for the caller's onError handler.
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

  // Lookup: PMID/DOI/NCT → canonical AMBC_/AMTC_ (Hard Rule #6). v0.1 keeps these helpers for analyst-supplied PMID/DOI/NCT seed extensions; v0.1 critical path does not call them.
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

  // Call 1 — per-sponsor TrialCore search (weekly cron entry point).
  async searchTrialcore(
    query: string,
    opts: TrialSearchOpts = {},
  ): Promise<unknown[] /* TrialCore record array; bind to @amass/sdk type once installed */> {
    const params = new URLSearchParams();
    params.set("query", query);
    for (const p of opts.phase ?? []) params.append("phase", p);
    if (opts.minStartDate) params.set("minStartDate", opts.minStartDate);
    if (opts.sponsorType) params.set("sponsorType", opts.sponsorType);
    params.set("limit", String(opts.limit ?? 300));
    return this.unwrapData(this.request<{ data: unknown[] }>("GET", `/api/v1/cores/trialcore/records?${params}`));
  }

  // Call 2 — per-trial cross-core walk (trial→paper direction; per Hard Rule #11).
  async getTrialRecord(
    amassId: string,
    includes: ReadonlyArray<"referencesBiomedCore" | "outcomes" | "detailedDescription"> = ["referencesBiomedCore"],
  ): Promise<unknown /* TrialCore record with referencesBiomedCore: string[] of AMBC_ IDs */> {
    const qs = includes.length > 0 ? `?${includes.map((i) => `include=${i}`).join("&")}` : "";
    return this.unwrapData(this.request<{ data: unknown }>("GET", `/api/v1/cores/trialcore/records/${amassId}${qs}`));
  }

  // Call 3 — per-paper resolution + trust-filter set (isRetracted + journalQualityJufo + citationCount). Default fields are sufficient — do NOT pass include=fulltext.
  async getBiomedRecord(
    amassId: string,
    includes: ReadonlyArray<"fulltext" | "authorsMetadata" | "meshIds" | "substanceIds" | "referencesTrialCore" | "references" | "citedBy"> = [],
  ): Promise<unknown /* BiomedCore record with isRetracted, journalQualityJufo, citationCount default fields */> {
    const qs = includes.length > 0 ? `?${includes.map((i) => `include=${i}`).join("&")}` : "";
    return this.unwrapData(this.request<{ data: unknown }>("GET", `/api/v1/cores/biomedcore/records/${amassId}${qs}`));
  }
}

// Canonicalisation helper for Hard Rule #9. Case-fold + drop common corporate suffixes.
const SPONSOR_SUFFIXES = [
  "inc", "inc.", "incorporated", "bio", "biosciences", "biotechnology", "sciences",
  "therapeutics", "pharmaceuticals", "pharma", "gmbh", "ltd", "ltd.", "limited", "ag", "sa",
];

export function canonicaliseSponsorName(raw: string): string {
  const lower = raw.toLowerCase().trim().replace(/[,.]/g, " ").replace(/\s+/g, " ");
  const tokens = lower.split(" ");
  while (tokens.length > 1 && SPONSOR_SUFFIXES.includes(tokens[tokens.length - 1])) {
    tokens.pop();
  }
  return tokens.join(" ");
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
```

**`app/api/digest-run/route.ts`** — Next.js App Router POST handler. Takes the watchlist + scope JSON body, executes Call 1 → Call 2 → Call 3 fan-out per sponsor, returns the 3-panel digest data structure for the dashboard.

```ts
import { getAmassClient } from "@/lib/amass";

interface DigestRunBody {
  watchlist_id: string;
  sponsors: ReadonlyArray<{ canonical: string; aliases: string[] }>;
  phases: ReadonlyArray<"PHASE2" | "PHASE2_PHASE3" | "PHASE3">;
  last_check: string;          // ISO YYYY-MM-DD
}

interface TrialRow {
  amassId: string;             // AMTC_...
  nctId: string;
  briefTitle: string;
  phase: string;
  overallStatus: string;
  sponsorName: string;
  startDate: string;
  lastUpdateDate: string;
}

interface PaperRow {
  amassId: string;             // AMBC_...
  pmid: string | null;
  doi: string | null;
  title: string;
  journal: string | null;
  publicationDate: string | null;
  citationCount: number | null;
  journalQualityJufo: number | null;
  isRetracted: boolean | null;
  describesTrials: string[];   // AMTC_ IDs of trials whose referencesBiomedCore included this paper
}

interface E015Anomaly {
  sponsor: string;
  nctId: string;
  side: "server-only" | "client-only" | "cardinality-warning";
  note: string;
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

export async function POST(req: Request) {
  try {
    const body = (await req.json()) as DigestRunBody;
    const client = await getAmassClient();

    const newTrials: TrialRow[] = [];
    const newPapers: Map<string, PaperRow> = new Map();   // dedup by AMBC_ across sponsors
    const retractionFlagged: PaperRow[] = [];
    const e015Anomalies: E015Anomaly[] = [];

    async function* runFanOut(): AsyncGenerator<string> {
      for (const sponsor of body.sponsors) {
        // Call 1 — per-sponsor TrialCore search (with dual-query workaround per Hard Rule #10).
        const [withDate, withoutDate] = await Promise.all([
          client.searchTrialcore(sponsor.canonical, { phase: body.phases, minStartDate: body.last_check }),
          client.searchTrialcore(sponsor.canonical, { phase: body.phases }),
        ]);
        yield `searched ${sponsor.canonical}`;

        // Diff for date-filter anomalies (per Hard Rule #10). Both sides typed as the TrialRow shape per the TrialCore search response.
        const serverList = withDate as TrialRow[];
        const clientFiltered = (withoutDate as TrialRow[]).filter(
          (t) => t.lastUpdateDate >= body.last_check && t.startDate >= body.last_check,
        );
        const serverSet = new Map(serverList.map((t) => [t.nctId, t]));
        const clientSet = new Map(clientFiltered.map((t) => [t.nctId, t]));

        for (const [nctId] of serverSet) {
          if (!clientSet.has(nctId)) {
            e015Anomalies.push({ sponsor: sponsor.canonical, nctId, side: "server-only",
              note: `surfaced by server-side minStartDate but does not satisfy client-side lastUpdateDate + startDate ≥ ${body.last_check}` });
          }
        }
        for (const [nctId] of clientSet) {
          if (!serverSet.has(nctId)) {
            e015Anomalies.push({ sponsor: sponsor.canonical, nctId, side: "client-only",
              note: `satisfies client-side filter but missing from server-side minStartDate result` });
          }
        }

        // Cardinality warning — exactly 300 records suggests the dynamic-crossing case (sponsor exceeds the per-scope ceiling).
        if ((withoutDate as TrialRow[]).length === 300) {
          e015Anomalies.push({ sponsor: sponsor.canonical, nctId: "—", side: "cardinality-warning",
            note: `returned exactly 300 records; may exceed limit=300 ceiling; consider narrowing indication scope` });
        }

        // Alias-fold on sponsorName for Genentech↔Roche-style subsidiary cases (Hard Rule #9).
        const aliasSet = new Set([sponsor.canonical, ...sponsor.aliases].map((s) => s.toLowerCase()));
        const sponsorTrials = serverList.filter((t) => aliasSet.has(t.sponsorName.toLowerCase()));
        newTrials.push(...sponsorTrials);

        // Call 2 — per-trial cross-core walk. trial→paper direction only (Hard Rule #11).
        for (const trial of sponsorTrials) {
          const detail = await client.getTrialRecord(trial.amassId, ["referencesBiomedCore"]) as {
            referencesBiomedCore: string[];
          };
          yield `walked ${trial.nctId}`;

          // Call 3 — per-paper resolution. Default fields are sufficient (NO include=fulltext per response-balloon caveat).
          for (const ambcId of detail.referencesBiomedCore ?? []) {
            if (newPapers.has(ambcId)) {
              newPapers.get(ambcId)!.describesTrials.push(trial.amassId);
              continue;
            }
            const paper = await client.getBiomedRecord(ambcId) as {
              pmid: string | null; doi: string | null; title: string; journal: string | null;
              publicationDate: string | null; citationCount: number | null;
              journalQualityJufo: number | null; isRetracted: boolean | null;
            };
            const row: PaperRow = {
              amassId: ambcId, pmid: paper.pmid, doi: paper.doi, title: paper.title,
              journal: paper.journal, publicationDate: paper.publicationDate,
              citationCount: paper.citationCount, journalQualityJufo: paper.journalQualityJufo,
              isRetracted: paper.isRetracted, describesTrials: [trial.amassId],
            };
            newPapers.set(ambcId, row);
            if (paper.isRetracted === true) retractionFlagged.push(row);
            yield `resolved ${paper.pmid ?? ambcId}`;
          }
        }
      }
    }

    for await (const _ of withIdleTimeout(runFanOut(), 180_000)) {
      // progress yielded for the dashboard's optional streaming indicator
    }

    return Response.json({
      data: {
        watchlist_id: body.watchlist_id,
        week_of: body.last_check,
        new_trials: newTrials,                       // left panel
        new_papers: Array.from(newPapers.values()),  // center panel
        retraction_flagged: retractionFlagged,       // right panel (empty array = "no retractions this week")
        e015_anomalies: e015Anomalies,               // sidecar
      },
    });
  } catch (err) {
    return Response.json({ error: err instanceof Error ? err.message : "Failed" }, { status: 500 });
  }
}
```

The two snippets above are the **only** `lib/<file>.ts` and `app/api/<endpoint>/route.ts` mandated by the v0.1 template. **NO `app/api/digest-history/route.ts`** (week-over-week diff persistence is "Want to extend it?" territory). **NO `python_cli/digest_run.py`** (headless cron sidecar is "Want to extend it?" territory). **NO `lib/digest-store.ts`** (no persistence layer in v0.1). The dashboard renders `data.new_trials` / `data.new_papers` / `data.retraction_flagged` into the 3 panels; `data.e015_anomalies` renders into a sidecar.

---

## Required reading

- `https://platform.amass.tech` — Amass Platform docs + API credentials
- `https://github.com/amass-technologies/public-amass-platform-starter-ts` — official Amass Platform Starter (Apache-2.0; Bun + Ink CLI REPL with BiomedCore + TrialCore tools wired up). The engineer-facing complement to this web-app kit.

---

## Hand-off

Build, lint, and typecheck must pass.

**Credentials.** Get your `AMASS_API_KEY` from https://platform.amass.tech and add it to `.env`. Do NOT use `AskUserQuestion` to gate scaffolding on credentials — Lovable's (or any AI builder's) standard env-var prompt covers this at run time. Just scaffold the app with placeholder `.env`, then run `npm run dev` once the key is in place.

Then present the hand-off summary:

> **Your Pipeline Monitor CI Agent is ready.**
>
> Before first push (if your AI builder left scaffolding placeholders): remove any `data-lovable-blank-page-placeholder="REMOVE_THIS"` attributes, any `<img>` tags pointing at `cdn.gpteng.co` or similar builder-placeholder CDNs, and any leftover "your app will live here" boilerplate. These render invisibly but pollute the DOM and serve no purpose in production. Search the codebase for `REMOVE_THIS` and `gpteng` before committing.
>
> To run it: fill in `.env` and run `npm run dev`.
>
> Verify it works:
> - Paste the SCLC-DLL3-2026Q2 watchlist YAML (or click **Try sample**); click **Run digest**; confirm all 5 sponsors expand with their Phase 2/3 trial sets and the sponsor-canonicalisation chip surfaces if Genentech appears in the raw watchlist (the dashboard rolls it up under Roche per Hard Rule #9).
> - In the **new-trials-this-week** panel (left), confirm `NCT05060016` (DeLLphi-301, Amgen, Phase 2) renders with `phase=PHASE2` and a recent `lastUpdateDate` in IBM Plex Mono; confirm `AMTC_` prefix on the trial's canonical Amass ID `[Tarlatamab anchor set verified against PubMed + ClinicalTrials.gov + FDA approval letter at draft time]`.
> - In the **new-papers-describing-watched-trials** panel (center), confirm PMID 37861218 (Ahn MJ et al. NEJM 2023) renders with a **★ credible journal** badge (live `journalQualityJufo` value from Amass — NEJM = JuFo 3) as a paper describing DeLLphi-301; confirm the paper card shows `describesTrials: [NCT05060016]` so the cross-core walk traversal is visible to the analyst.
> - In the **retraction-cascade panel** (right), confirm it renders empty (`no retractions this week`) on the SCLC-DLL3-2026Q2 worked example — both walked PMIDs (37861218, 37355629) are verified `isRetracted=false`; honest emptiness is the correct behaviour per Hard Rule #13. (Run against your own watchlist on a different week to see the panel populate from live Amass `isRetracted=true` records.)
>
> Want to extend it?
> - Add a Python CLI sidecar (`python_cli/digest_run.py`) for headless weekly cron invocation. The CLI takes the watchlist YAML path as an argument, posts to `/api/digest-run`, and writes the returned digest to `~/pipeline-monitor/<watchlist_id>/<YYYY-Wnn>.md` for Slack-paste / Confluence-paste / email-paste by the analyst.
> - Add week-over-week digest-history persistence with append-only ISO-week indexing for diff visualisation. A trial that surfaces in week N+1 but not week N gets a `new-this-week` badge; a trial whose `lastUpdateDate` advanced gets a `changed-this-week` badge.
> - Add cursor pagination for sponsors with >300 active trials per scope. The v0.1 numerical bounding caps each canonical sponsor at <300 trials at Step 0.5 cardinality probe; lifting that cap requires a cursor-pagination surface (pending an upstream Amass API extension).
> - I'm done — just show me the summary.
>
> Have questions? Join the Amass Developer Community on Discord: `https://discord.com/invite/sEGaBHMhWa`. Building a CLI agent instead of a web app? The engineer-facing complement to this kit is the official Amass Platform Starter at `https://github.com/amass-technologies/public-amass-platform-starter-ts` (Bun + Ink + Vercel AI SDK CLI REPL, Apache-2.0).

---

## License

Apache-2.0. Copyright 2026 Amass Technologies. Source at https://github.com/lluisDTU/amass-technologies-amass-starters.
