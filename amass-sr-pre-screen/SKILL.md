---
name: amass-sr-pre-screen
description: Use when building a PRISMA pre-screen credibility-filter for systematic reviews — batch-resolves a curated PMID dump to canonical Amass IDs, post-filters on JuFo + retraction + citation count, and emits a Rayyan/Covidence-importable RIS file plus an audit-trail CSV.
license: Apache-2.0
metadata:
  author: amass
  version: "0.3.0"
---

# SR Pre-Screen Credibility Filter

For systematic-review researchers running PRISMA 2020 — Cochrane groups, SR-as-a-service consultancies, academic teams. The user pastes a curated PMID dump from the upstream search step (typically ~5,000 PMIDs from PubMed + Embase + CENTRAL + Scopus) and gets back a Rayyan/Covidence-importable RIS file containing only the ~100-500 papers that pass credibility thresholds (JuFo journal-quality tier, retraction flag, citation count). The load-bearing Amass primitives are the trust-filter conjunction on every paper record (`isRetracted` + `journalQualityJufo` + `citationCount` all default fields) plus per-item-error semantics on batch lookup — competitor stacks require Web of Science + Scopus + Retraction Watch + JuFo-table licensing to assemble equivalent signals.

---

## Important — Lovable scaffold cleanup

**Before generating the page component**, the AI builder MUST verify there is NO leftover Lovable blank-app placeholder in the final JSX. Specifically, the page root must NOT contain:

- An outer `<div className="flex min-h-screen items-center justify-center">` wrapping the real app content
- `<img data-lovable-blank-page-placeholder="REMOVE_THIS" src="https://cdn.gpteng.co/blank-app-v1.svg" ... />`

If these are present from Lovable's default template, REMOVE them entirely. The real app's root element must be the page's only top-level wrapper. Leaving the placeholder wrapper in place produces malformed JSX (unclosed outer div) and the published deploy renders a generic error page instead of the app.

---

## What you build

The user pastes a PMID dump (one PMID per line, typical 1,000-5,000 rows) and configures credibility thresholds (`min_jufo`, `allow_retracted`, `min_citation_count`). The handler chunks the PMID list into ~100-item batches and runs `POST /records/lookup` concurrently across chunks. For each resolved canonical `AMBC_`, it calls `GET /records/{amassId}` (no `include` — `isRetracted`, `journalQualityJufo`, `citationCount` are default fields), then client-side filters against the configured thresholds. The output is a results table + two downloadable artifacts: a Rayyan/Covidence-importable RIS file containing only the included papers, and an audit-trail CSV with one row per submitted PMID showing why it was included or excluded.

The input surface is a textarea for the PMID list + three threshold controls (JuFo dropdown 1-3, retraction toggle, citation count number input). The empty state offers a Try-sample button that loads ~10 illustrative GLP-1 receptor agonist PMIDs. The output is a paginated results table with download buttons. During the per-paper fan-out, the UI shows a Cancel button and a progress indicator.

---

## API contract — load-bearing, follow exactly

### Endpoints used

- `POST /api/v1/cores/biomedcore/records/lookup` — batch resolve PMID → `AMBC_` (chunked to ~100 items per request)
- `GET /api/v1/cores/biomedcore/records/{amassId}` — fetch paper record with default fields (covers `isRetracted` + `journalQualityJufo` + `citationCount` — do NOT pass `include` flags)

### Auth + rate limit

`Authorization: Bearer ${AMASS_API_KEY}` on every request. 60 requests / 60 seconds per user+org. On HTTP 429, read the `Retry-After` header and back off exponentially. On 401/403, surface the credential error directly — retry won't fix it.

### Response envelope (CRITICAL)

Every endpoint returns `{ "data": ... }` wrapped. Unwrap before consuming.

For `POST /records/lookup`, each `data[]` element is one of:

```json
{ "input": { "pmid": "32109013" }, "amassIds": ["AMBC_..."] }
```

or

```json
{ "input": { "pmid": "99999999" }, "error": { "code": "NOT_FOUND", "message": "Identifier not found" } }
```

The per-item `error` field is a STRUCTURED OBJECT with `code` and `message` — NOT a bare string. Always extract `.message` as a string before rendering. NEVER render the error object directly as a React child (crashes the surface with React error #31).

### Top-level errors

Non-2xx HTTP response body: `{ "error": { "status": ..., "code": ..., "message": ... } }`. Throw with a message including the upstream `code` + `message`. The outer route handler's `try/catch` surfaces this as a mutation error inline — never let it propagate to the route-level error boundary.

### Lookup chunking

The `items[]` array on `POST /records/lookup` accepts up to ~100 items per request (conservative ceiling pending upstream quantification). Chunk inputs above ~100 PMIDs into multiple concurrent lookup calls.

---

## Reference code

### `lib/amass.ts` — server-only API client

Drop this file in unchanged. Framework-agnostic. The `import "server-only"` enforces server-side-only usage at build time.

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

  getBiomedRecord = (amassId: string, includes: ReadonlyArray<string> = []) => {
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

export function chunk<T>(arr: ReadonlyArray<T>, size: number): T[][] {
  const out: T[][] = [];
  for (let i = 0; i < arr.length; i += size) out.push(arr.slice(i, i + size));
  return out;
}
```

### Server-side route handler

Write the handler in your framework's preferred server-side shape (TanStack Start `createServerFn` OR Next.js `app/api/pre-screen/route.ts`). The handler structure:

1. Parse the input body with a zod `PreScreenSchema` (fields: `pmids[]`, `min_jufo`, `allow_retracted`, `min_citation_count`).
2. Chunk the PMID list into 100-item batches via the `chunk` helper. Run all `batchLookupBiomed` calls concurrently via `Promise.all`.
3. Use `extractLookupResult` on each chunk. Successes feed the per-paper fan-out; per-item failures populate `lookup_error` on that row.
4. For each resolved `AMBC_`, call `getBiomedRecord(amassId)` (no `include`). Wrap this fan-out in `withIdleTimeout(gen, 180_000)`. Extract `isRetracted`, `journalQualityJufo`, `citationCount` from the returned record.
5. Apply client-side threshold filter: include the paper if `(isRetracted === false || allow_retracted) && journalQualityJufo >= min_jufo && citationCount >= min_citation_count`.
6. Assemble: results table rows, RIS file body (only included papers), audit CSV (all papers with `included_in_prescreen` boolean and replayed thresholds).
7. Return `{ rows, ris, csv, summary: { total, resolved, included, excluded, lookup_errors } }` as JSON.

**Critical:** wrap the entire handler body in `try { ... } catch (err) { ... }`. On TanStack Start, throw `new Error(err.message ?? "Pre-screen run failed")`. On Next.js, return `Response.json({ error: ... }, { status: 500 })`. NEVER let an uncaught throw propagate to the route-level error boundary.

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

Use these illustrative PMIDs in the empty-state Try-sample button. Render ALL trust-filter values from the live Amass response — NEVER hardcode them.

- **Review scope:** GLP-1 receptor agonists in obesity
- **Sample PMIDs (10 illustrative — re-verify against PubMed before binding to a real SR):** 36720262, 35658024, 34614329, 36331190, 35658026, 34170647, 35470291, 32109013, 32966830, 35658028
- **Default thresholds:** `min_jufo=2`, `allow_retracted=false`, `min_citation_count=10`
- **Expected behavior:** all 10 PMIDs resolve to canonical `AMBC_` IDs; the included subset depends on the threshold filter applied to live data

Try-sample tooltip: "GLP-1 receptor agonists in obesity — 10 illustrative PMIDs. Re-verify against PubMed before binding to a real SR."

---

## Per-starter constraints

- Audit CSV has exactly these 11 columns: `pmid, accessed_at, AMBC_id, isRetracted, journalQualityJufo, citationCount, lookup_error, included_in_prescreen, threshold_min_jufo, threshold_min_citation_count, threshold_allow_retracted`. The trailing three `threshold_*` columns repeat the run-time threshold values on every row so a downstream reviewer can reproduce the inclusion/exclusion verdicts without remembering what the screener set.
- RIS file output uses the standard RIS field mapping for journal articles: `TY=JOUR, PMID, TI, AU, JO, JF, PY, DO, AB`. Each emitted RIS record represents one INCLUDED paper. Excluded papers are NOT in the RIS file (only in the audit CSV).
- v0.1 ceiling: 5,000 PMIDs per pre-screen run. The chunking strategy (100 per batch) keeps the lookup phase under the rate-limit ceiling. For larger SR corpora, run multiple sessions or extend with a streaming worker.
- v0.1 path is lookup-only — no BiomedCore search-driven discovery. Adding search-driven auto-expansion of the review scope is the primary extension.

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

1. **Empty state** renders cleanly with the Try-sample button + 3 threshold controls visible. Browser console clean (no errors / warnings about `data-lovable-blank-page-placeholder` or unclosed JSX).
2. **Try sample** loads the 10 GLP-1 PMIDs and runs end-to-end. Results table renders with one row per PMID, each showing the canonical `AMBC_` ID, the live JuFo tier, the live `isRetracted` flag, and the live `citationCount`.
3. **Threshold adjustment:** change `min_jufo` from 2 to 3 and re-run. The included count drops accordingly; rows with `journalQualityJufo < 3` flip to `included_in_prescreen=false`. The page does NOT crash.
4. **Download RIS + CSV:** both files emit cleanly. The RIS file contains only included papers in valid RIS format (opens in Rayyan or Covidence). The CSV contains all 11 columns with `threshold_*` values replayed on every row.

---

## Hand-off

Build, lint, typecheck must pass. Apply the Pre-publish cleanup steps from Kit conventions above.

**Credentials:** get your `AMASS_API_KEY` from https://platform.amass.tech and add it to `.env`.

To run: `npm run dev`.

**Want to extend it?**
- Add cohort persistence — save pre-screen runs against a `project_id` so multiple SR teams can collaborate on the same corpus over time.
- Add a PRISMA-flow-diagram generator that consumes the audit CSV and emits the standard PRISMA 2020 flow figure.
- Add field-specific JuFo overrides — different credibility floor for clinical-trial papers vs basic-science papers.

Questions? Join the Amass Developer Community on Discord: https://discord.com/invite/sEGaBHMhWa.

---

## License

Apache-2.0. Copyright 2026 Amass Technologies. Source: https://github.com/lluisDTU/amass-technologies-amass-starters.
