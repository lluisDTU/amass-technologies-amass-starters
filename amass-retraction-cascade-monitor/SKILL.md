---
name: amass-retraction-cascade-monitor
description: Use when building a retraction-cascade tracker that re-audits a stored bibliography for newly retracted citations AND surfaces the downstream cascade — inbound citedBy, outbound references, and the cross-core trial spine — all in one record fetch.
license: Apache-2.0
metadata:
  author: amass
  version: "0.3.0"
---

# Retraction Cascade Monitor

For three ICPs sharing the same workflow shape: systematic-review teams re-auditing a published review's bibliography for citations retracted since publication; regulatory-affairs analysts monitoring a submission's citation chain between submission and approval; medical-affairs teams monitoring a published medical-information answer for citation contamination over time. The user pastes a curated bibliography PMID list and gets a 3-panel cascade dashboard showing retracted papers in the bibliography, the inbound `citedBy` cascade of downstream papers carrying contamination, and the outbound `references` cascade plus cross-core trial spine. The load-bearing Amass primitive is the three-direction-in-one-record-fetch conjunction — `isRetracted` (default field) + `references` + `citedBy` + `referencesTrialCore` all delivered in a single `GET` with one `include=` set, which OpenAlex and Scite cannot match in one HTTP call.

---

## Important — Lovable scaffold cleanup

**Before generating the page component**, the AI builder MUST verify there is NO leftover Lovable blank-app placeholder in the final JSX. Specifically, the page root must NOT contain:

- An outer `<div className="flex min-h-screen items-center justify-center">` wrapping the real app content
- `<img data-lovable-blank-page-placeholder="REMOVE_THIS" src="https://cdn.gpteng.co/blank-app-v1.svg" ... />`

If these are present from Lovable's default template, REMOVE them entirely. The real app's root element must be the page's only top-level wrapper. Leaving the placeholder wrapper in place produces malformed JSX (unclosed outer div) and the published deploy renders a generic error page instead of the app.

---

## What you build

The user pastes a bibliography PMID list (typical SR cohort: 100-500 PMIDs at time t0). The handler batch-resolves all PMIDs to canonical `AMBC_` IDs, then for each resolved paper issues `GET /records/{amassId}?include=references,citedBy,referencesTrialCore` — a single call that delivers four signals: the `isRetracted` default field, the outbound `references` array, the inbound `citedBy` array, and the cross-core `referencesTrialCore` array. The handler then filters to the subset where `isRetracted === true` and assembles a 3-panel cascade dashboard: retracted papers in the bibliography (left), inbound `citedBy` cascade (center — downstream papers carrying contamination), and outbound `references` plus cross-core `referencesTrialCore` (right — what the retracted paper itself cited, and what trials it describes).

The input surface is a multi-line PMID textarea (one PMID per line). The empty state offers a Try-sample button that loads the Wakefield worked example verbatim. The output is the 3-panel dashboard plus a Download cascade report button (Markdown or JSON). During the per-paper fan-out, the UI shows a Cancel button and progress indicator.

---

## API contract — load-bearing, follow exactly

### Endpoints used

- `POST /api/v1/cores/biomedcore/records/lookup` — batch resolve PMID → `AMBC_`
- `GET /api/v1/cores/biomedcore/records/{amassId}?include=references,citedBy,referencesTrialCore` — fetch paper record with all three cascade directions in one call (`isRetracted` is a default field, no include flag needed)

### Auth + rate limit

`Authorization: Bearer ${AMASS_API_KEY}` on every request. 60 requests / 60 seconds per user+org. On HTTP 429, read the `Retry-After` header and back off exponentially. On 401/403, surface the credential error directly — retry won't fix it.

### Response envelope (CRITICAL)

Every endpoint returns `{ "data": ... }` wrapped. Unwrap before consuming.

For `POST /records/lookup`, each `data[]` element is one of:

```json
{ "input": { "pmid": "9500320" }, "amassIds": ["AMBC_..."] }
```

or

```json
{ "input": { "pmid": "99999999" }, "error": { "code": "NOT_FOUND", "message": "Identifier not found" } }
```

The per-item `error` field is a STRUCTURED OBJECT with `code` and `message` — NOT a bare string. Always extract `.message` as a string before rendering. NEVER render the error object directly as a React child (crashes the surface with React error #31).

### Top-level errors

Non-2xx HTTP response body: `{ "error": { "status": ..., "code": ..., "message": ... } }`. Throw with a message including the upstream `code` + `message`. The outer route handler's `try/catch` surfaces this as a mutation error inline — never let it propagate to the route-level error boundary.

### Three-direction-in-one-call discipline

The single-record-fetch with `include=references,citedBy,referencesTrialCore` is the LOAD-BEARING discipline. DO NOT split this into multiple per-direction calls. Collapsing the include set across two HTTP calls defeats the value proposition that justifies this starter.

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

export function extractLookupResult<I, T>(
  inputs: ReadonlyArray<I>,
  results: LookupResult,
  onSuccess: (input: I, amassIds: string[]) => T,
  onError: (input: I, message: string) => T,
): T[] {
  return inputs.map((input, i) => {
    const r = results[i];
    return r?.amassIds?.length
      ? onSuccess(input, r.amassIds)
      : onError(input, r?.error?.message ?? "no result");
  });
}
```

### Server-side route handler

Write the handler in your framework's preferred server-side shape (TanStack Start `createServerFn` OR Next.js `app/api/cascade-audit/route.ts`). The handler structure:

1. Parse the input body with a zod `CascadeRequestSchema` (fields: `pmids[]`).
2. Run `batchLookupBiomed(pmidItems)`. Use `extractLookupResult` to split successes (collect `AMBC_` IDs) from per-item failures (push to a `lookup_errors` array).
3. For each resolved `AMBC_`, call `getBiomedRecord(amassId, ["references", "citedBy", "referencesTrialCore"])`. Wrap this fan-out in `withIdleTimeout(gen, 180_000)`.
4. Filter the returned records to the subset where `isRetracted === true`.
5. Assemble the 3-panel cascade structure:
   - **Left:** retracted papers (the filtered subset) with journal, retraction flag, and inbound-cascade cardinality count.
   - **Center:** flat list of papers cited BY each retracted paper (from `citedBy[]`). These are downstream papers carrying contamination.
   - **Right:** flat list of papers each retracted paper itself cited (from `references[]`) PLUS the cross-core trials (from `referencesTrialCore[]`).
6. Return `{ retracted, inbound, outbound, lookup_errors }` as JSON.

**Critical:** wrap the entire handler body in `try { ... } catch (err) { ... }`. On TanStack Start, throw `new Error(err.message ?? "Cascade audit failed")`. On Next.js, return `Response.json({ error: ... }, { status: 500 })`. NEVER let an uncaught throw propagate to the route-level error boundary.

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

Use these verified PMIDs in the empty-state Try-sample button. Render ALL retraction signals from the live Amass response — NEVER hardcode them (defamation-adjacent if a real paper is shown with a wrong retraction flag).

- **Anchor:** PMID 9500320 (Wakefield et al., Lancet 1998, fully retracted 2010-02-06; the canonical retracted paper for demonstration purposes) — expected `isRetracted=true`, expected inbound `citedBy` cascade with several thousand downstream entries per published bibliometrics
- **Context (10 illustrative PMIDs from autism-vaccination methodology literature — re-verify against PubMed before binding to a real audit):** 18039999, 19720750, 16352714, 16456141, 14738625, 11241983, 15585572, 18207565, 19127374, 19651677
- **Retraction notice:** PMID 20137807 (the retraction notice itself, published 2010-02-06)

Try-sample tooltip: "Wakefield et al. Lancet 1998 (PMID 9500320) — canonical retracted paper. Fully retracted 2010-02-06. Re-verify identifier set against PubMed + Retraction Watch before binding to a real audit."

---

## Per-starter constraints

- The 3-panel cascade structure (retracted | inbound | outbound + cross-core) is the load-bearing UX. All three panels populate from the SAME record fetch — do not split into multiple calls.
- `isRetracted` values render directly from the live Amass response. NEVER hardcode retraction flags. Showing a real paper with a wrong retraction status is defamation-adjacent — the live-data binding structurally prevents this failure mode.
- Empty arrays in any direction render as honest emptiness (no downstream citers, no outbound references, no trial references). Do not hide empty panels or fabricate placeholder cards.
- Downloadable cascade report: Markdown OR JSON format selectable from the UI. The Markdown variant uses nested headers per retracted paper with bulleted lists for each cascade direction.
- v0.1 is bibliography-snapshot-driven: the caller's stored PMID list at time t0. v0.1 does NOT include BiomedCore search-driven discovery (auto-detect newly retracted papers across a topic area beyond the bibliography). That's the primary extension path.

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

1. **Empty state** renders cleanly with the Try-sample button + PMID textarea visible. Browser console clean (no errors / warnings about `data-lovable-blank-page-placeholder` or unclosed JSX).
2. **Try sample** loads the Wakefield bibliography (~10 PMIDs including 9500320) and runs end-to-end. The left panel renders 1 retracted paper (Wakefield, Lancet 1998, live `isRetracted=true`). The center panel populates with downstream citers. The right panel populates with outbound references + cross-core trial references (may be empty for the Wakefield record specifically — render as honest emptiness).
3. **Invalid PMID test:** add `99999999` to the bibliography and re-run. The corresponding row goes to the `lookup_errors` collection with the verbatim upstream message rendered inline. Page does NOT crash; the rest of the cascade completes.
4. **Download cascade report:** click the Markdown variant. File saves with nested headers per retracted paper and bulleted lists for each cascade direction.

---

## Hand-off

Build, lint, typecheck must pass. Apply the Pre-publish cleanup steps from Kit conventions above.

**Credentials:** get your `AMASS_API_KEY` from https://platform.amass.tech and add it to `.env`.

To run: `npm run dev`.

**Want to extend it?**
- Add a cron-based re-audit cadence (weekly/monthly) with an append-only audit log keyed by `bibliography_id`, surfacing newly-retracted citations alerts.
- Extend the discovery surface with a BiomedCore search-driven mode that auto-discovers newly-retracted papers across a topic area beyond the caller's stored bibliography.
- Add a "draft retraction-acknowledgement letter" generator using the assembled cascade as input — for SR teams who need to formally acknowledge retracted citations in their review's published correction.

Questions? Join the Amass Developer Community on Discord: https://discord.com/invite/sEGaBHMhWa.

---

## License

Apache-2.0. Copyright 2026 Amass Technologies. Source: https://github.com/lluisDTU/amass-technologies-amass-starters.
