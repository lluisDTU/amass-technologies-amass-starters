---
name: amass-pipeline-monitor
description: Use when building a weekly pipeline-monitoring digest for biotech competitive-intelligence analysts — takes a sponsor watchlist, surfaces new Phase 2/3 trials via per-sponsor TrialCore search, walks the trial→paper cross-core edge, and emits a 3-panel weekly dashboard.
license: Apache-2.0
metadata:
  author: amass
  version: "0.3.0"
---

# Pipeline Monitor

For competitive-intelligence analysts at biotech and pharma companies tracking a watchlist of pharma sponsors and indication clusters. The user pastes a YAML watchlist of sponsors and gets a weekly 3-panel digest: new Phase 2/3 trials this week, new papers describing watched trials, and retraction-flagged citations. The load-bearing Amass primitives are canonical `AMTC_` IDs (week-over-week diffs don't drift when CT.gov rewrites an NCT) plus the trial→paper cross-core edge (`referencesBiomedCore`) — one Amass call surfaces paper-side context that PubMed + CT.gov + OpenAlex require 3-4 calls to assemble.

---

## Important — Lovable scaffold cleanup

**Before generating the page component**, the AI builder MUST verify there is NO leftover Lovable blank-app placeholder in the final JSX. Specifically, the page root must NOT contain:

- An outer `<div className="flex min-h-screen items-center justify-center">` wrapping the real app content
- `<img data-lovable-blank-page-placeholder="REMOVE_THIS" src="https://cdn.gpteng.co/blank-app-v1.svg" ... />`

If these are present from Lovable's default template, REMOVE them entirely. The real app's root element must be the page's only top-level wrapper. Leaving the placeholder wrapper in place produces malformed JSX (unclosed outer div) and the published deploy renders a generic error page instead of the app.

---

## What you build

The user pastes a YAML sponsor watchlist (canonical sponsors + indication-cluster scope + last-check date). The handler issues a per-sponsor TrialCore search via `GET /records?query=<sponsor>&phase=...&minStartDate=<last_check>` to surface new Phase 2/3 trials since the last check. For each surfaced trial, it walks the trial→paper cross-core edge via `GET /records/{AMTC_}?include=referencesBiomedCore`, then per-paper resolves trust signals (`isRetracted`, `journalQualityJufo`, `citationCount`) via `GET /records/{AMBC_}`. The output is a Markdown digest grouped by sponsor with three sections: new trials, new papers, retraction-flagged citations — rendered inline as a 3-panel dashboard.

The input surface is a single textarea accepting YAML. The empty state offers a Try-sample button that loads the verified SCLC-DLL3-2026Q2 watchlist verbatim. The output renders as a 3-panel dashboard with a Download Markdown digest button. During the per-sponsor + per-trial + per-paper fan-out, the UI shows a Cancel button and a progress indicator.

---

## API contract — load-bearing, follow exactly

### Endpoints used

- `GET /api/v1/cores/trialcore/records?query=<sponsor>&phase=PHASE2&phase=PHASE2_PHASE3&phase=PHASE3&minStartDate=<YYYY-MM-DD>&limit=300` — per-sponsor TrialCore search returning new trials
- `GET /api/v1/cores/trialcore/records/{amassId}?include=referencesBiomedCore` — fetch trial record with paper-side cross-core spine
- `GET /api/v1/cores/biomedcore/records/{amassId}` — fetch paper record with trust signals (`isRetracted`, `journalQualityJufo`, `citationCount` are default fields; no `include` flag needed)

### Auth + rate limit

`Authorization: Bearer ${AMASS_API_KEY}` on every request. 60 requests / 60 seconds per user+org. On HTTP 429, read the `Retry-After` header and back off exponentially. On 401/403, surface the credential error directly — retry won't fix it.

### Response envelope (CRITICAL)

Every endpoint returns `{ "data": ... }` wrapped. Unwrap before consuming. The TrialCore search endpoint returns `{ "data": [...] }` (array of trial records, NOT lookup-result-shaped). Per-record GETs return `{ "data": {...} }` (single object).

### Top-level errors

Non-2xx HTTP response body: `{ "error": { "status": ..., "code": ..., "message": ... } }`. Throw with a message including the upstream `code` + `message`. The outer route handler's `try/catch` surfaces this as a mutation error inline — never let it propagate to the route-level error boundary.

### TrialCore search ceiling

The `limit` parameter caps at 300 records per query. Sponsors returning >300 results in a `query=<sponsor>&phase=PHASE2&phase=PHASE2_PHASE3&phase=PHASE3` probe (without `minStartDate`) need their indication scope narrowed at onboarding before the weekly cron can run reliably.

---

## Reference code

### `lib/amass.ts` — server-only API client

Drop this file in unchanged. Framework-agnostic. The `import "server-only"` enforces server-side-only usage at build time.

```ts
import "server-only";

export type Phase = "PHASE2" | "PHASE2_PHASE3" | "PHASE3";

export interface TrialSearchOpts {
  phase?: ReadonlyArray<Phase>;
  minStartDate?: string;   // ISO YYYY-MM-DD
  sponsorType?: string;
  limit?: number;          // default 300, ceiling 300
}

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

  searchTrialcore = (query: string, opts: TrialSearchOpts = {}) => {
    const params = new URLSearchParams();
    params.set("query", query);
    for (const p of opts.phase ?? []) params.append("phase", p);
    if (opts.minStartDate) params.set("minStartDate", opts.minStartDate);
    if (opts.sponsorType) params.set("sponsorType", opts.sponsorType);
    params.set("limit", String(opts.limit ?? 300));
    return this.req<unknown[]>("GET", `/api/v1/cores/trialcore/records?${params}`);
  };

  getTrialRecord = (
    amassId: string,
    includes: ReadonlyArray<"referencesBiomedCore" | "outcomes" | "detailedDescription"> = [],
  ) => {
    const qs = includes.length ? `?${includes.map((i) => `include=${i}`).join("&")}` : "";
    return this.req<unknown>("GET", `/api/v1/cores/trialcore/records/${amassId}${qs}`);
  };

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

### Server-side route handler

Write the handler in your framework's preferred server-side shape (TanStack Start `createServerFn` OR Next.js `app/api/digest-run/route.ts`). The handler structure:

1. Parse the input body with a zod `WatchlistSchema` (fields: `sponsors[]` of `{ canonical, aliases[] }`, `phases[]`, `last_check`, `watchlist_id`).
2. For each canonical sponsor, run two TrialCore searches concurrently: one with `minStartDate=<last_check>` (the "new since last check" query), one without (the cardinality probe). Wrap the per-sponsor fan-out in `Promise.all`.
3. For each surfaced trial in the "new" query result, call `getTrialRecord(amassId, ["referencesBiomedCore"])` to fetch the trial's paper-side cross-core spine. Deduplicate the union of `AMBC_` IDs across all trials.
4. For each unique paper `AMBC_`, call `getBiomedRecord(amassId)` (no `include` — default fields cover trust signals). Wrap this fan-out in `withIdleTimeout(gen, 180_000)`.
5. Assemble the 3-panel output: new trials (left), new papers (center), retraction-flagged citations (right — papers with `isRetracted=true`).
6. Return `{ digest: {...}, errors: {...} }` as JSON. Include the Markdown digest body for direct download.

**Critical:** wrap the entire handler body in `try { ... } catch (err) { ... }`. On TanStack Start, throw `new Error(err.message ?? "Digest run failed")` from the catch — TanStack Query surfaces it as `mutation.error.message`. On Next.js, return `Response.json({ error: ... }, { status: 500 })`. NEVER let an uncaught throw propagate to the route-level error boundary.

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

Use this verified watchlist in the empty-state Try-sample button. Render all live data from the Amass response — NEVER hardcode trial / paper metadata.

- **Watchlist:** SCLC-DLL3-2026Q2
- **Sponsors:** Amgen, Roche (canonical for Genentech), AbbVie, Boehringer Ingelheim, Harpoon Therapeutics
- **Phases:** PHASE2, PHASE2_PHASE3, PHASE3
- **last_check:** 2026-05-13 (one week prior to today)
- **Expected trial anchor:** NCT05060016 (DeLLphi-301, Amgen) surfaces in the Amgen panel with `phase=PHASE2`
- **Expected paper anchor:** PMID 37861218 (Ahn MJ et al. NEJM 2023) surfaces in the "new papers" center panel via the DeLLphi-301 cross-core walk

Try-sample tooltip: "SCLC-DLL3-2026Q2 — five sponsors tracking DLL3-targeted bispecifics in small-cell lung cancer." (Tarlatamab anchor set verified against PubMed + ClinicalTrials.gov + FDA approval letter at draft time)

---

## Per-starter constraints

- Sponsor-name canonicalisation onboarding step: issue a `GET /records?query=<raw sponsor name>&limit=300` probe and cluster returned `sponsorName` strings. Genentech maps to Roche (subsidiary with its own CT.gov string). Persist the analyst-confirmed canonical→aliases map into the watchlist YAML.
- Sponsor cardinality probe: if a per-sponsor `query=<canonical>&phase=PHASE2&phase=PHASE2_PHASE3&phase=PHASE3` (no `minStartDate`) returns ≥300 records, the limit ceiling is hit — the analyst must narrow the indication scope before the weekly run is reliable.
- 3-panel dashboard structure: new trials (left), new papers via trial→paper cross-core walk (center), retraction-flagged citations (right). Don't collapse panels.
- Markdown digest filename: `digest-<watchlist_id>-<YYYY-Wnn>.md` (ISO week number). The download button emits this file directly from the response payload.
- v0.1 returns a single consolidated response per digest run. No digest-history append-only persistence in v0.1 (that's the primary extension path).

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
- `lib/amass.ts` carries `import "server-only"` at the top. Never imported from a `"use client"` component.
- `https://api.amass.tech` is hardcoded as the base URL in `lib/amass.ts`. Do NOT introduce an `AMASS_API_BASE_URL` env var.
- Pin React 18.3.1, TypeScript 5.4.5, zod 3.23.8, lucide-react 0.453.0, server-only 0.0.1. Add framework packages on top.

**Pre-publish cleanup (DO THIS — Lovable scaffolds may leave these in place):**
- Strip ALL `data-lovable-blank-page-placeholder="REMOVE_THIS"` attributes.
- Strip ALL `<img>` tags pointing at `cdn.gpteng.co`.
- If Lovable left an outer `<div className="flex min-h-screen items-center justify-center">` wrapping the real app + a leftover placeholder image, REMOVE the outer wrapper entirely so the real app's root `<div>` is the only top-level returned element.

---

## Validation

After `npm run dev`, verify in order:

1. **Empty state** renders cleanly with the Try-sample button visible. Browser console clean (no errors / warnings about `data-lovable-blank-page-placeholder` or unclosed JSX).
2. **Try sample** loads the SCLC-DLL3-2026Q2 watchlist and runs end-to-end. The new-trials-this-week panel populates with at least DeLLphi-301 (NCT05060016, Phase 2) under Amgen; the new-papers panel populates from the trial→paper cross-core walk; the retraction-flagged panel populates honestly (may be empty if no walked paper has `isRetracted=true`).
3. **Invalid sponsor test:** add a fictional sponsor name to the watchlist YAML (e.g. `Nonexistent Pharma Inc`) and re-run. The corresponding per-sponsor block renders with zero new trials and a note explaining the empty result. Page does NOT crash; other sponsors continue to populate.
4. **Download Markdown digest** click emits `digest-sclc-dll3-2026q2-<YYYY-Wnn>.md` with per-sponsor sections containing trial cards and paper cards rendered correctly.

---

## Hand-off

Build, lint, typecheck must pass. Apply the Pre-publish cleanup steps from Kit conventions above.

**Credentials:** get your `AMASS_API_KEY` from https://platform.amass.tech and add it to `.env`.

To run: `npm run dev`.

**Want to extend it?**
- Add digest-history append-only persistence — week-over-week diff in the dashboard itself, surfacing newly-added trials vs the previous week.
- Wire to a Slack channel or email distribution list as an MCP slash-command, so the weekly digest delivers itself.
- Add modality-cluster scoping (e.g., "DLL3-targeted bispecifics in SCLC" as a saved indication query) for cross-sponsor competitive analysis.

Questions? Join the Amass Developer Community on Discord: https://discord.com/invite/sEGaBHMhWa.

---

## License

Apache-2.0. Copyright 2026 Amass Technologies. Source: https://github.com/lluisDTU/amass-technologies-amass-starters.
