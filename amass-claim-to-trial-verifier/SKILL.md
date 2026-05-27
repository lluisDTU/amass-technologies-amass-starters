---
name: amass-claim-to-trial-verifier
description: Use when building a marketing-claim-to-trial verifier — resolves cited DOIs to canonical Amass IDs, walks the paper→trial cross-core edge, retrieves the trial's registered primary outcome verbatim from the protocol, and surfaces a flagged verdict naming the claim/outcome match or mismatch.
license: Apache-2.0
metadata:
  author: amass
  version: "0.3.0"
---

# Claim-to-Trial Verifier

For two ICPs sharing the same workflow shape: regulatory consultants reviewing promotional material (FDA OPDP-adjacent compliance reviews, EMA pharmacovigilance audits, internal medical-affairs sign-off) and investigative journalists auditing pharma marketing claims (fact-checking press releases, sales-aid decks, direct-to-consumer ads). The user pastes a marketing claim plus the 1-5 DOIs the promotional copy cites and gets back a verdict card showing the trial's registered primary endpoint verbatim from the protocol next to the claim text. The load-bearing Amass primitives are the paper→trial cross-core walk via `referencesTrialCore` plus the TrialCore `include=outcomes` response delivering the registered primary outcome as a verbatim string — text-mining for NCT identifiers misses methods/supplement references and drifts across registry revisions; LLM paraphrasing trial outcomes against a named pharma sponsor is defamation-adjacent.

---

## Important — Lovable scaffold cleanup

**Before generating the page component**, the AI builder MUST verify there is NO leftover Lovable blank-app placeholder in the final JSX. Specifically, the page root must NOT contain:

- An outer `<div className="flex min-h-screen items-center justify-center">` wrapping the real app content
- `<img data-lovable-blank-page-placeholder="REMOVE_THIS" src="https://cdn.gpteng.co/blank-app-v1.svg" ... />`

If these are present from Lovable's default template, REMOVE them entirely. The real app's root element must be the page's only top-level wrapper. Leaving the placeholder wrapper in place produces malformed JSX (unclosed outer div) and the published deploy renders a generic error page instead of the app.

---

## What you build

The user pastes a marketing claim about a drug plus the 1-5 cited DOIs from the promotional copy. The handler runs a 3-call chain: `POST /records/lookup` to resolve cited DOIs to canonical `AMBC_` IDs; `GET /records/{AMBC_}?include=referencesTrialCore` per resolved paper to walk to the cited trials; `GET /records/{AMTC_}?include=outcomes` per unique walked trial to retrieve the registered primary outcome measures verbatim. The output is a verdict card per claim with five fields: claim text verbatim, cited DOIs verbatim, walked trials with `AMTC_` + NCT + brief title, primary outcome text verbatim from the trial protocol, and a verdict badge (`supported` / `not_supported` / `contradicts` / `needs_review`) with a 1-2 sentence reasoning naming the specific outcome-category match or mismatch.

The input surface is a claim textarea + a cited-DOIs list (comma-or-newline separated, 1-5 entries). The empty state offers a Try-sample button that loads the verified Tarlatamab claim worked example verbatim. The output is the verdict card. During the 3-call fan-out, the UI shows a Cancel button and progress indicator.

---

## API contract — load-bearing, follow exactly

### Endpoints used

- `POST /api/v1/cores/biomedcore/records/lookup` — batch resolve DOI → `AMBC_`
- `GET /api/v1/cores/biomedcore/records/{amassId}?include=referencesTrialCore` — fetch paper record with cross-core trial spine
- `GET /api/v1/cores/trialcore/records/{amassId}?include=outcomes` — fetch trial record with the structured `outcomes` array containing primary outcome measures verbatim

### Auth + rate limit

`Authorization: Bearer ${AMASS_API_KEY}` on every request. 60 requests / 60 seconds per user+org. On HTTP 429, read the `Retry-After` header and back off exponentially. On 401/403, surface the credential error directly — retry won't fix it.

### Response envelope (CRITICAL)

Every endpoint returns `{ "data": ... }` wrapped. Unwrap before consuming.

For `POST /records/lookup`, each `data[]` element is one of:

```json
{ "input": { "doi": "10.1056/NEJMoa2307980" }, "amassIds": ["AMBC_..."] }
```

or

```json
{ "input": { "doi": "10.9999/invalid" }, "error": { "code": "NOT_FOUND", "message": "Identifier not found" } }
```

The per-item `error` field is a STRUCTURED OBJECT with `code` and `message` — NOT a bare string. Always extract `.message` as a string before rendering. NEVER render the error object directly as a React child (crashes the surface with React error #31).

### Top-level errors

Non-2xx HTTP response body: `{ "error": { "status": ..., "code": ..., "message": ... } }`. Throw with a message including the upstream `code` + `message`. The outer route handler's `try/catch` surfaces this as a mutation error inline — never let it propagate to the route-level error boundary.

### Verbatim-outcome discipline

The trial's primary outcome text is returned by `GET /records/{AMTC_}?include=outcomes` as a verbatim string from the trial protocol. RENDER IT VERBATIM. Do NOT paraphrase, normalise, or LLM-rewrite the outcome text between Amass's response and the verdict card. A `contradicts` verdict against a named pharma sponsor with paraphrased outcome text is defamation-adjacent — the verbatim render is the structural guard.

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

Write the handler in your framework's preferred server-side shape (TanStack Start `createServerFn` OR Next.js `app/api/verify-claim/route.ts`). The handler structure:

1. Parse the input body with a zod `ClaimVerifyRequestSchema` (fields: `claim: string`, `cited_dois: string[]` of length 1-5). Reject empty claim or 0-DOI input with HTTP 400.
2. Run `batchLookupBiomed(doiItems)`. Use `extractLookupResult` to split successes (collect `AMBC_` IDs) from per-item failures (push to a `resolved_papers` array with `lookup_error` populated).
3. For each resolved `AMBC_`, call `getBiomedRecord(amassId, ["referencesTrialCore"])`. Wrap this fan-out in `withIdleTimeout(gen, 180_000)`. Collect the union of walked `AMTC_` IDs across all papers; deduplicate.
4. For each unique walked `AMTC_`, call `getTrialRecord(amassId, ["outcomes"])`. Extract `nctId`, `briefTitle`, and `primaryOutcomeMeasures` (string array) — render these verbatim.
5. Run the LLM-compare step against the verbatim claim + verbatim outcome text. The LLM authors a verdict (`supported` / `not_supported` / `contradicts` / `needs_review`) plus a 1-2 sentence reasoning citing the specific outcome-category match or mismatch. Constrain the LLM prompt to CITE the verbatim phrasings — do not assert truth/falsity beyond what the registered primary endpoint supports.
6. If no trials walked at all (review-article papers with empty `referencesTrialCore`), surface verdict as `not_supported` with reasoning "no trial reference in cross-core spine on cited paper" — not a workflow halt.
7. Return `{ claim, cited_dois, resolved_papers, walked_trials, verdict, reasoning }` as JSON.

**Critical:** wrap the entire handler body in `try { ... } catch (err) { ... }`. On TanStack Start, throw `new Error(err.message ?? "Claim verification failed")`. On Next.js, return `Response.json({ error: ... }, { status: 500 })`. NEVER let an uncaught throw propagate to the route-level error boundary.

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

Use these verified identifiers in the empty-state Try-sample button. Render the primary outcome text verbatim from the live Amass response — NEVER paraphrase, never hardcode.

- **Claim:** "Tarlatamab improves overall survival in previously-treated ES-SCLC"
- **Cited DOI:** 10.1056/NEJMoa2307980 (Ahn MJ et al., NEJM 2023, DeLLphi-301 primary publication)
- **Expected walked trial:** NCT05060016 (DeLLphi-301) via `AMTC_` resolved through paper→trial cross-core walk
- **Expected primary outcome text (verbatim from trial protocol):** Objective Response Rate per RECIST 1.1 by Investigator (and BICR-assessed companion) — NOT "Overall Survival"
- **Expected verdict:** `needs_review` — claim's stated outcome category (Overall Survival) does not match trial's registered primary endpoint category (Objective Response Rate per RECIST 1.1)

Try-sample tooltip: "Tarlatamab (Imdelltra) / Amgen — Ahn NEJM 2023, DeLLphi-301 primary publication. Expected verdict: needs_review (claim says OS; trial primary endpoint is ORR per RECIST 1.1)."

(Identifiers and primary endpoint phrasing verified against ClinicalTrials.gov + Amass API on 2026-05-21.)

---

## Per-starter constraints

- v0.1 accepts exactly 1 claim + 1-5 cited DOIs per verification run. Multi-claim batching across multiple promotional pieces is an extension path.
- Verdict card has exactly five fields rendered in this order: claim verbatim (sans-serif), cited DOIs verbatim (monospace), walked trials (`AMTC_` + NCT + briefTitle, monospace), primary outcome text VERBATIM (sans-serif, NEVER paraphrased), verdict badge with 1-2 sentence reasoning.
- The LLM-compare step is constrained to CITE the verbatim claim phrasing AND the verbatim outcome phrasing in its reasoning. It does NOT assert truth/falsity beyond what the registered primary endpoint supports.
- Walked-trial cards' `AMTC_` and `NCT` IDs render as clickable links to the Amass platform's trial-detail surface (`https://platform.amass.tech/records/trialcore/{amassId}`) so the analyst can inspect the registered trial behind the verdict.
- v0.1 surfaces the trial's REGISTERED primary endpoint (from the trial registration), NOT the publication's reported outcome metric. The registered endpoint is the load-bearing ground truth for OPDP-style claim audits.

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

1. **Empty state** renders cleanly with the claim textarea + cited-DOIs input + Try-sample button visible. Browser console clean (no errors / warnings about `data-lovable-blank-page-placeholder` or unclosed JSX).
2. **Try sample** loads the Tarlatamab claim + DOI 10.1056/NEJMoa2307980 and runs end-to-end. The 3-call chain resolves: DOI → `AMBC_` → walks to `AMTC_` → trial outcomes retrieved. Verdict card renders with the verbatim claim, the verbatim DOI in monospace, the walked NCT05060016 in monospace, primary outcome text reading something like "Objective Response Rate per RECIST 1.1 by Investigator" (verbatim from live Amass response — NOT "Overall Survival"), and verdict `needs_review` with reasoning naming the outcome-category mismatch.
3. **Invalid DOI test:** add `10.9999/this-doi-does-not-exist` to the cited DOIs and re-verify. The corresponding row populates `lookup_error` with the verbatim upstream message; verdict falls back to `not_supported` with reasoning explaining no trial walked. Page does NOT crash.
4. **Different claim same DOI:** paste "Tarlatamab achieves an objective response in previously-treated DLL3-positive ES-SCLC" + same DOI. Verdict shifts to `supported` (or close) with reasoning naming the outcome-category MATCH (claim says ORR; trial primary IS ORR per RECIST 1.1).

---

## Hand-off

Build, lint, typecheck must pass. Apply the Pre-publish cleanup steps from Kit conventions above.

**Credentials:** get your `AMASS_API_KEY` from https://platform.amass.tech and add it to `.env`.

To run: `npm run dev`.

**Want to extend it?**
- Add multi-claim batching for OPDP enforcement-letter audits — persist verdicts to an append-only audit log keyed by `claim_id + session_id` so a regulatory consultant can run a whole promotional piece through.
- Add free-text claim extraction — accept a full promotional-piece PDF, LLM-extract the list of distinct assertions, then verify each as a batched run.
- Add trial-results-magnitude comparison — when CT.gov has posted results, compare the claim's STATED magnitude (e.g. "30% objective response") against the trial's REPORTED magnitude, surfacing `partially_supported` for direction-correct-but-magnitude-overstated cases.

Questions? Join the Amass Developer Community on Discord: https://discord.com/invite/sEGaBHMhWa.

---

## License

Apache-2.0. Copyright 2026 Amass Technologies. Source: https://github.com/lluisDTU/amass-technologies-amass-starters.
