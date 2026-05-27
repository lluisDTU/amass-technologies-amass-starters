---
name: amass-regulatory-evidence-assembler
description: Use when building an auditable literature-evidence assembler for FDA / EMA regulatory submissions (CTD Module 2.5 / 2.7 / PSUR) — resolves PMID/DOI/NCT submission scope to canonical Amass IDs, walks the paper→trial cross-core edge, and emits an audit CSV with trust-filter columns.
license: Apache-2.0
metadata:
  author: amass
  version: "0.3.0"
---

# Regulatory Evidence Assembler

For medical writers, regulatory-affairs analysts, and evidence-synthesis CROs preparing CTD Module 2.5 / 2.7 narratives or PSUR-style ongoing-evidence audits. The user pastes a curated PMID/DOI/NCT submission scope and gets back an audit CSV where each cited paper is anchored to its supporting trial via canonical Amass IDs that don't drift when ClinicalTrials.gov rewrites an NCT between submission and approval. The load-bearing Amass primitive is canonical-ID stability — `AMBC_` and `AMTC_` IDs survive upstream-registry revisions, so the audit chain stays valid across the 6-12 month submission-to-approval window.

---

## Important — Lovable scaffold cleanup

**Before generating the page component**, the AI builder MUST verify there is NO leftover Lovable blank-app placeholder in the final JSX. Specifically, the page root must NOT contain:

- An outer `<div className="flex min-h-screen items-center justify-center">` wrapping the real app content
- `<img data-lovable-blank-page-placeholder="REMOVE_THIS" src="https://cdn.gpteng.co/blank-app-v1.svg" ... />`

If these are present from Lovable's default template, REMOVE them entirely. The real app's root element must be the page's only top-level wrapper. Leaving the placeholder wrapper in place produces malformed JSX (unclosed outer div) and the published deploy renders a generic error page instead of the app.

---

## What you build

The user pastes a YAML submission scope listing PMIDs, DOIs, and NCTs from upstream discovery (PubHive / Embase / Scopus). The handler runs three concurrent batch lookups — `POST /records/lookup` for PMIDs+DOIs against BiomedCore, plus `POST /records/lookup` for NCTs against TrialCore — resolving every external identifier to a canonical `AMBC_` or `AMTC_` ID with per-item error preservation. For each resolved BiomedCore record, it then issues a `GET /records/{amassId}?include=referencesTrialCore,citedBy` to walk the paper→trial cross-core edge and retrieve trust-filter signals (`isRetracted`, `journalQualityJufo`, `citationCount`). The output is an audit CSV with one row per submitted identifier, anchored to canonical Amass IDs that survive NCT-registry revisions between submission and approval.

The input surface is a single textarea accepting YAML. The empty state offers a Try-sample button that loads the verified Tarlatamab worked example identifiers verbatim. The output renders as a results table with a Download audit CSV button and a Copy citations button. While the per-paper fan-out runs, the UI shows a Cancel button and a progress indicator.

---

## API contract — load-bearing, follow exactly

### Endpoints used

- `POST /api/v1/cores/biomedcore/records/lookup` — batch resolve PMID/DOI → `AMBC_`
- `POST /api/v1/cores/trialcore/records/lookup` — batch resolve NCT → `AMTC_`
- `GET /api/v1/cores/biomedcore/records/{amassId}?include=referencesTrialCore,citedBy` — fetch paper record with cross-core trial spine and citation graph

### Auth + rate limit

`Authorization: Bearer ${AMASS_API_KEY}` on every request. 60 requests / 60 seconds per user+org. On HTTP 429, read the `Retry-After` header and back off exponentially. On 401/403, surface the credential error directly — retry won't fix it.

### Response envelope (CRITICAL)

Every endpoint returns `{ "data": ... }` wrapped. Unwrap before consuming.

For `POST /records/lookup`, each `data[]` element is one of:

```json
{ "input": { "pmid": "37861218" }, "amassIds": ["AMBC_..."] }
```

or

```json
{ "input": { "pmid": "99999999" }, "error": { "code": "NOT_FOUND", "message": "Identifier not found" } }
```

The per-item `error` field is a STRUCTURED OBJECT with `code` and `message` — NOT a bare string. Always extract `.message` as a string before rendering. NEVER render the error object directly as a React child (crashes the surface with React error #31).

### Top-level errors

Non-2xx HTTP response body: `{ "error": { "status": ..., "code": ..., "message": ... } }`. Throw with a message including the upstream `code` + `message`. The outer route handler's `try/catch` surfaces this as a mutation error inline — never let it propagate to the route-level error boundary.

### Lookup-before-fetch discipline

`GET /records/{amassId}` accepts ONLY canonical `AMBC_` / `AMTC_` IDs. Passing a PMID / DOI / NCT directly returns 404. Always: lookup first → resolve canonical → fetch by canonical ID.

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

  batchLookupTrial = (items: ReadonlyArray<{ nctId: string }>) =>
    this.req<LookupResult>("POST", "/api/v1/cores/trialcore/records/lookup", { items });

  getBiomedRecord = (
    amassId: string,
    includes: ReadonlyArray<
      "fulltext" | "authorsMetadata" | "meshIds" | "substanceIds" | "referencesTrialCore" | "references" | "citedBy"
    > = [],
  ) => {
    const qs = includes.length ? `?${includes.map((i) => `include=${i}`).join("&")}` : "";
    return this.req<unknown>("GET", `/api/v1/cores/biomedcore/records/${amassId}${qs}`);
  };

  getTrialRecord = (
    amassId: string,
    includes: ReadonlyArray<"detailedDescription" | "outcomes" | "referencesBiomedCore"> = [],
  ) => {
    const qs = includes.length ? `?${includes.map((i) => `include=${i}`).join("&")}` : "";
    return this.req<unknown>("GET", `/api/v1/cores/trialcore/records/${amassId}${qs}`);
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

Write the handler in your framework's preferred server-side shape (TanStack Start `createServerFn({ method: "POST" }).inputValidator(...).handler(...)` OR Next.js `app/api/audit-run/route.ts` with `export async function POST(req)`). The handler structure:

1. Parse the input body with a zod `SubmissionScopeSchema` (fields: `submission_id`, `drug`, `client_sponsor`, `indication`, `pmids[]`, `dois[]`, `ncts[]`).
2. Run three concurrent batch lookups via `Promise.all`: `batchLookupBiomed(pmidItems)`, `batchLookupBiomed(doiItems)`, `batchLookupTrial(nctItems)`.
3. Use `extractLookupResult` to walk each input array against its result array. For successes, push a row with `resolved_amassId` and collect the canonical ID for the per-paper fan-out. For per-item errors, push a row with `lookup_error` populated by the verbatim upstream message.
4. For each collected `AMBC_`, call `getBiomedRecord(amassId, ["referencesTrialCore", "citedBy"])` and merge the live `isRetracted`, `journalQualityJufo`, `citationCount`, and `referencesTrialCore` array into the corresponding row. Wrap this fan-out in `withIdleTimeout(gen, 180_000)` (helper below); on stall, throw.
5. Per-record GET failures populate `walk_error` on that row (NOT `lookup_error` — these are structurally different columns).
6. Return `{ rows, errors: { lookup, walk } }` as JSON.

**Critical:** wrap the entire handler body in `try { ... } catch (err) { ... }`. On TanStack Start, throw `new Error(err.message ?? "Audit run failed")` from the catch — TanStack Query surfaces it as `mutation.error.message` on the client. On Next.js, return `Response.json({ error: ... }, { status: 500 })`. NEVER let an uncaught throw propagate to the route-level error boundary.

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

Use these verified identifiers in the empty-state Try-sample button. Render ALL trust-filter values from the live Amass response — NEVER hardcode them.

- **Submission:** BLA 761344-dlle (tarlatamab / Imdelltra, Amgen, ES-SCLC)
- **PMID 37861218** (Ahn MJ et al., NEJM 2023, DeLLphi-301 primary publication) — expected `isRetracted=false`, JuFo tier 3 (NEJM)
- **PMID 37355629** (Rudin et al., J Hematol Oncol 2023, DLL3 review) — expected `isRetracted=false`, expected `referencesTrialCore=[]` (review article — honest emptiness)
- **NCT05060016** (DeLLphi-301)
- **NCT05740566** (DeLLphi-304)

Try-sample tooltip text: "Tarlatamab (Imdelltra) / Amgen — Ahn NEJM 2023 + Rudin J Hematol Oncol 2023; DeLLphi-301 + DeLLphi-304." (verified against PubMed + ClinicalTrials.gov + FDA approval letter at draft time)

---

## Per-starter constraints

- The audit CSV has exactly these 9 columns: `accessed_at, input_identifier, resolved_amassId, isRetracted, journalQualityJufo, citationCount, referencesTrialCore_AMTCs, lookup_error, walk_error`. Do not add or rename columns.
- `lookup_error` and `walk_error` are SEPARATE columns — never collapse. Lookup-side failures (PMID retired, DOI revoked, NCT merge) populate `lookup_error`; per-record GET failures after a successful lookup populate `walk_error`. The distinction is auditable for ICH submission documentation.
- Walk direction is paper→trial only. Empty `referencesTrialCore` arrays render as `[]` (honest emptiness — review articles like PMID 37355629), not as workflow halts.
- The `referencesTrialCore_AMTCs` column joins multi-element arrays with `|` for single-cell CSV compatibility.
- v0.1 returns assembled rows in a single response. No persistence layer, no monthly re-audit cron, no ack-event lifecycle (those are extension paths).

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
2. **Try sample** loads the Tarlatamab YAML and runs end-to-end. Rendered rows show: 4 rows resolved to canonical AMBC_/AMTC_ IDs; PMID 37861218 row has `isRetracted=false`, `journalQualityJufo=3` (NEJM), `referencesTrialCore_AMTCs` populated with the DeLLphi-301 `AMTC_` ID; PMID 37355629 row has empty `referencesTrialCore_AMTCs` (review article).
3. **Invalid PMID test:** paste `99999999` as an additional PMID and re-run. The corresponding row renders inline with `lookup_error` populated by the verbatim upstream message (e.g. `"Identifier not found"`). Page does NOT crash; the rest of the audit completes.
4. **Download audit CSV** click emits a file with the 9-column header line and one row per submitted identifier.

---

## Hand-off

Build, lint, typecheck must pass. Apply the Pre-publish cleanup steps from Kit conventions above.

**Credentials:** get your `AMASS_API_KEY` from https://platform.amass.tech and add it to `.env`.

To run: `npm run dev`.

**Want to extend it?**
- Add a monthly re-audit cron with append-only audit-CSV history — catches newly-retracted citations between submission (t0) and approval (tN) across the PSUR lifecycle, anchored to canonical `AMBC_` IDs.
- Add a retraction-review queue with ack-event lifecycle (`acked_at`, `acked_by`, `ack_decision`) durably persisted across worker restarts.
- Add a Python CLI sidecar for headless cron invocation against the same `lib/amass.ts` semantics.

Questions? Join the Amass Developer Community on Discord: https://discord.com/invite/sEGaBHMhWa.

---

## License

Apache-2.0. Copyright 2026 Amass Technologies. Source: https://github.com/lluisDTU/amass-technologies-amass-starters.
