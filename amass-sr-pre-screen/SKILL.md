---
name: amass-sr-pre-screen
description: Use when building a PRISMA pre-screen credibility-filter for systematic reviews — batch-resolves a curated PMID dump to canonical Amass IDs, post-filters on JuFo + retraction + citation count, and emits a Rayyan/Covidence-importable RIS file plus an audit-trail CSV.
license: Apache-2.0
metadata:
  author: amass
  version: "0.5.0"
---

# SR Pre-Screen Credibility Filter

For systematic-review researchers running PRISMA 2020 — Cochrane groups, SR-as-a-service consultancies, academic teams. The user pastes a curated PMID dump from the upstream search step (typically ~5,000 PMIDs) and gets back a Rayyan/Covidence-importable RIS file containing only the ~100-500 papers that pass credibility thresholds (JuFo journal-quality tier, retraction flag, citation count). The load-bearing Amass primitives are the trust-filter conjunction on every paper record (`isRetracted` + `journalQualityJufo` + `citationCount` all default fields) plus per-item-error semantics on batch lookup — competitor stacks require Web of Science + Scopus + Retraction Watch + JuFo-table licensing to assemble equivalent signals.

---

## Scaffold target — read FIRST

This starter is built on **TanStack Start + Vite + TanStack Router + TanStack Query** (Lovable's default). THREE TypeScript files; do NOT wrap them in any outer template or placeholder component.

For Next.js App Router users, see the Appendix.

### Load-bearing constraints — DO NOT mutate these (read before transcribing)

Past scaffolds in the kit have failed because LLMs "auto-corrected" the spec. Each rule below is a hard constraint from the live Amass API or from Vite/TanStack Start semantics. Violating any of them causes a runtime error or a build error. Do NOT optimise, normalise, or "improve" them.

1. **Per-item lookup error shape is `{ error: { code, message } }`.** Each element of the `data[]` array returned by `POST /records/lookup` is either `{ amassIds: [...] }` (success) OR `{ error: { code, message } }` (per-item failure). The `error` field is a STRUCTURED OBJECT — empirically verified against the live API, even though the documentation example shows a simpler bare-string form. Always extract `.message` as a string before rendering. NEVER pass the error object directly as a React child (React #31 crash). The reference `extractLookupResult()` helper in `amass.server.ts` handles this correctly — DO NOT rewrite it to assume the error is a bare string.
2. **Trust signals are default fields. Do NOT pass `include=` flags.** `isRetracted`, `journalQualityJufo`, and `citationCount` are returned as default fields on every `GET /biomedcore/records/{amassId}` response. Passing `include=isRetracted,journalQualityJufo,citationCount` is unnecessary and may be ignored or treated as a no-op.
3. **Replace files completely. Do NOT preserve any Lovable blank-template content.** Before declaring the scaffold complete, the following strings MUST appear ZERO times in `src/`:
   - `data-lovable-blank-page-placeholder`
   - `cdn.gpteng.co/blank-app-v1.svg`
   - `"Your App"` (in meta titles)
   - `"Replace this with a one-sentence description"`
   - Any `<div className="flex min-h-screen items-center justify-center">` wrapper around the root `<main>` or around any threshold-control or results-table component.
   And any single JSX element MUST have at most ONE `className=` attribute (duplicate `className` is a JSX syntax error and React silently keeps only one).
4. **Server-only enforcement.** Use the `.server.ts` filename suffix (`src/lib/amass.server.ts`). Do NOT add `import "server-only"` — that's a Next.js package that crashes Vite SSR.
5. **Route component binding.** `createFileRoute("/")` MUST receive `{ head, component: Index }`. Omitting `component: Index` makes SSR render an empty page.
6. **Lookup chunking is per-100-items.** The reference code chunks the input PMID list into batches of ≤100 before issuing `POST /records/lookup`. Do NOT change this to per-1000 or per-1 — 100 is the documented sweet spot for the batch endpoint.

### File 1: `src/lib/amass.server.ts` — server-only API client

The `.server.ts` filename enforces server-only. **Do NOT add `import "server-only"`** — it crashes Vite SSR.

```ts
// src/lib/amass.server.ts
// Server-only via .server.ts filename. DO NOT add `import "server-only"`.

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

### File 2: `src/lib/prescreen.functions.ts` — TanStack Start server function

```ts
// src/lib/prescreen.functions.ts
import { createServerFn } from "@tanstack/react-start";
import { z } from "zod";
import { getAmassClient, extractLookupResult, chunk, type LookupResult } from "./amass.server";

export const PreScreenSchema = z.object({
  pmids: z.array(z.string()).min(1),
  min_jufo: z.number().int().min(1).max(3).default(2),
  allow_retracted: z.boolean().default(false),
  min_citation_count: z.number().int().min(0).default(10),
});

export type AuditRow = {
  pmid: string;
  accessed_at: string;
  AMBC_id: string;
  isRetracted: string;
  journalQualityJufo: string;
  citationCount: string;
  lookup_error: string;
  included_in_prescreen: string;
  threshold_min_jufo: string;
  threshold_min_citation_count: string;
  threshold_allow_retracted: string;
};

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

export const runPreScreen = createServerFn({ method: "POST" })
  .inputValidator((input: unknown) => PreScreenSchema.parse(input))
  .handler(async ({ data }) => {
    try {
      const client = getAmassClient();
      const accessed_at = new Date().toISOString();

      // Chunk PMIDs to ~100 per batch lookup
      const pmidItems = data.pmids.map((pmid) => ({ pmid }));
      const chunks = chunk(pmidItems, 100);
      const chunkResults = await Promise.all(chunks.map((c) => client.batchLookupBiomed(c)));
      const lookupResults: LookupResult = chunkResults.flat() as LookupResult;

      const rows: AuditRow[] = [];
      const ambcToRow = new Map<string, number>();

      extractLookupResult(pmidItems, lookupResults,
        (input, ids) => {
          const id = ids[0];
          const idx = rows.push({
            pmid: input.pmid!,
            accessed_at,
            AMBC_id: id,
            isRetracted: "",
            journalQualityJufo: "",
            citationCount: "",
            lookup_error: "",
            included_in_prescreen: "false",
            threshold_min_jufo: String(data.min_jufo),
            threshold_min_citation_count: String(data.min_citation_count),
            threshold_allow_retracted: String(data.allow_retracted),
          }) - 1;
          ambcToRow.set(id, idx);
          return null;
        },
        (input, msg) => {
          rows.push({
            pmid: input.pmid!,
            accessed_at,
            AMBC_id: "",
            isRetracted: "",
            journalQualityJufo: "",
            citationCount: "",
            lookup_error: msg,
            included_in_prescreen: "false",
            threshold_min_jufo: String(data.min_jufo),
            threshold_min_citation_count: String(data.min_citation_count),
            threshold_allow_retracted: String(data.allow_retracted),
          });
          return null;
        },
      );

      // Per-paper fan-out: default fields cover trust signals (no include= needed).
      async function* walkGen() {
        for (const ambcId of ambcToRow.keys()) {
          try {
            const rec = (await client.getBiomedRecord(ambcId)) as Record<string, unknown>;
            yield { ambcId, rec, error: null as string | null };
          } catch (e) {
            yield { ambcId, rec: null, error: e instanceof Error ? e.message : String(e) };
          }
        }
      }
      for await (const step of withIdleTimeout(walkGen(), 180_000)) {
        const idx = ambcToRow.get(step.ambcId)!;
        if (step.error) { rows[idx].lookup_error = step.error; continue; }
        const rec = step.rec as Record<string, unknown>;
        const retracted = Boolean(rec["isRetracted"]);
        const jufo = (rec["journalQualityJufo"] as number) ?? 0;
        const cites = (rec["citationCount"] as number) ?? 0;
        rows[idx].isRetracted = String(retracted);
        rows[idx].journalQualityJufo = String(jufo);
        rows[idx].citationCount = String(cites);
        // Apply threshold filter
        const included = (!retracted || data.allow_retracted) && jufo >= data.min_jufo && cites >= data.min_citation_count;
        rows[idx].included_in_prescreen = String(included);
      }

      const included = rows.filter((r) => r.included_in_prescreen === "true").length;
      return { rows, summary: { total: rows.length, included, excluded: rows.length - included } };
    } catch (err) {
      throw new Error(err instanceof Error ? err.message : "Pre-screen run failed");
    }
  });
```

### File 3: `src/routes/index.tsx` — TanStack Start route + page component

The `component: Index` binding is REQUIRED. Root is `<main>` — no outer wrapper.

```tsx
// src/routes/index.tsx
import { createFileRoute } from "@tanstack/react-router";
import { useState } from "react";
import { useMutation } from "@tanstack/react-query";
import { useServerFn } from "@tanstack/react-start";
import { runPreScreen } from "@/lib/prescreen.functions";

export const Route = createFileRoute("/")({
  head: () => ({
    meta: [
      { title: "SR Pre-Screen — Amass" },
      { name: "description", content: "PRISMA pre-screen credibility-filter for systematic reviews." },
    ],
  }),
  component: Index, // ← REQUIRED. Without this, SSR fails.
});

function Index() {
  const [pmidsInput, setPmidsInput] = useState("");
  const [minJufo, setMinJufo] = useState(2);
  const [allowRetracted, setAllowRetracted] = useState(false);
  const [minCitationCount, setMinCitationCount] = useState(10);
  const preScreen = useServerFn(runPreScreen);
  const mutation = useMutation({
    mutationFn: async () => {
      const pmids = pmidsInput.split(/\s+/).filter(Boolean);
      return preScreen({ data: { pmids, min_jufo: minJufo, allow_retracted: allowRetracted, min_citation_count: minCitationCount } });
    },
  });

  return (
    <main className="min-h-screen bg-background text-foreground font-sans">
      <header className="border-b border-border">
        <div className="max-w-6xl mx-auto px-6 py-4 flex items-center justify-between">
          <h1 className="text-2xl font-extrabold tracking-tight">amass</h1>
          <span className="text-xs text-muted-foreground font-mono">sr-pre-screen · v0.1</span>
        </div>
      </header>

      <section className="max-w-6xl mx-auto px-6 py-10 space-y-8">
        {/* PMID textarea + threshold controls (min_jufo, allow_retracted, min_citation_count) + Try-sample + Pre-screen button + Results table + Download RIS + Download CSV */}
      </section>
    </main>
  );
}
```

---

## What you build

The user pastes a PMID dump (1,000-5,000 rows, one per line) and configures credibility thresholds. The handler chunks PMIDs into ~100-item batches and runs `POST /records/lookup` concurrently. For each resolved canonical `AMBC_`, it calls `GET /records/{amassId}` (no `include` — `isRetracted`, `journalQualityJufo`, `citationCount` are default fields), then client-side filters against the thresholds. Output: results table + Rayyan/Covidence-importable RIS file (included papers only) + 11-column audit CSV (all papers with `included_in_prescreen` boolean + replayed thresholds per row).

Input surface: textarea + 3 threshold controls. Empty state: Try-sample with the SCLC-DLL3-2026Q2 worked example (10 verified PMIDs spanning NEJM through Cureus, designed to demonstrate the JuFo + citation-count filter cleanly). Output: paginated results table + Download buttons. During fan-out: Cancel + progress indicator.

---

## API contract — load-bearing, follow exactly

**Authoritative source:** https://platform.amass.tech/documentation/for-ai-agents/llm-quick-reference.md — the live single-page reference for every endpoint, param, enum, and response schema across BiomedCore + TrialCore. If anything in this SKILL.md drifts from that doc, the doc wins.

### Endpoints used

- `POST /api/v1/cores/biomedcore/records/lookup` — batch resolve PMID → `AMBC_` (chunked to ~100 items per request)
- `GET /api/v1/cores/biomedcore/records/{amassId}` — fetch paper record with default fields (covers `isRetracted` + `journalQualityJufo` + `citationCount` — do NOT pass `include` flags)

### Critical operational rules (load-bearing — the build fails without them)

- **Auth:** `Authorization: Bearer ${AMASS_API_KEY}` on every request. Get key from https://platform.amass.tech. Rate limit: 60 req / 60 s per user+org. On HTTP 429, read `Retry-After` and back off exponentially. On 401/403, surface the credential error directly.
- **Response envelope:** every endpoint returns `{ "data": ... }` wrapped. Unwrap before consuming.
- **Per-item lookup errors:** each `data[]` element on `POST /records/lookup` is either `{ amassIds: [...] }` (success) OR `{ error: { code, message } }` (per-item failure). The error field is a STRUCTURED OBJECT — empirically verified, even if the live docs example shows the simpler `{ error: "..." }` form. Always extract `.message` as a string before rendering. NEVER render the error object directly as a React child (React #31 crash). Example: `{ "input": { "pmid": "99999999" }, "error": { "code": "NOT_FOUND", "message": "Identifier not found" } }`.
- **Top-level errors:** non-2xx body is `{ "error": { "status", "code", "message" } }`. The reference code's `req<T>` throws with the upstream code+message; the route handler's outer `try/catch` re-throws so TanStack Query surfaces it as `mutation.error.message` on the client. NEVER let an uncaught throw propagate to the route-level error boundary.
- **Lookup before fetch:** `GET /records/{amassId}` accepts ONLY canonical `AMBC_` / `AMTC_` IDs. Passing PMID/DOI/NCT directly returns 404. Always: lookup → resolve canonical → fetch by canonical ID.

### Lookup chunking

`POST /records/lookup` `items[]` accepts up to ~100 items per request. Use the `chunk()` helper.

---

## Worked example

- **Review scope:** SCLC-DLL3-2026Q2 — Tarlatamab and DLL3-targeted bispecifics for small-cell lung cancer.
- **Sample PMIDs (10, verified live against Amass BiomedCore on 2026-05-28):** 37861218, 37355629, 39023700, 39876075, 41532856, 41256964, 41401368, 40884462, 41111642, 40419854
- **Default thresholds:** `min_jufo=2`, `allow_retracted=false`, `min_citation_count=10`

Expected behaviour with default thresholds (verified live):

| PMID | First author / Journal | JuFo | Cites | `isRetracted` | Verdict |
|---|---|---|---|---|---|
| 37861218 | Ahn / NEJM 2023 (DeLLphi-301) | 3 | 390 | false | **INCLUDED** |
| 37355629 | Rudin / J Hematol Oncol 2023 (DLL3 review) | 2 | 135 | false | **INCLUDED** |
| 39023700 | Dhillon / Drugs 2024 (Tarlatamab: First Approval) | 2 | 35 | false | **INCLUDED** |
| 39876075 | Sands / Cancer 2025 (AE management) | 2 | 24 | false | **INCLUDED** |
| 41532856 | Mishra / Cancer Discovery 2026 (CTC biomarker) | 3 | 2 | false | EXCLUDED (cites < 10) |
| 41256964 | Aredo / JTO CRR 2025 (EGFR-mutant DLL3) | 0 | 1 | false | EXCLUDED (jufo < 2) |
| 41401368 | Antoniu / Exp Opin Biol Ther 2025 (review) | 1 | 0 | false | EXCLUDED (jufo < 2) |
| 40884462 | Tomić / Biomol Biomed 2025 (SCLC overview) | 1 | 0 | false | EXCLUDED (jufo < 2) |
| 41111642 | Ismail / Cureus 2025 (heart transplant case) | 0 | 0 | false | EXCLUDED (jufo < 2) |
| 40419854 | Pathak / Ther Innov Reg Sci 2025 (regulatory) | 1 | 0 | false | EXCLUDED (jufo < 2) |

Under defaults: **4 INCLUDED / 6 EXCLUDED**. PMID 41532856 (Mishra / Cancer Discovery) is a useful teaching case — it has the highest JuFo tier (3, alongside the NEJM anchor) but fails the citation floor because it was published in 2026 and hasn't accumulated cites yet. This demonstrates that the threshold filter cuts on a quality dimension that JuFo alone wouldn't catch.

Try-sample tooltip: "SCLC-DLL3-2026Q2 — Tarlatamab and DLL3-targeted bispecifics in small-cell lung cancer. 10 PMIDs verified live against Amass BiomedCore on 2026-05-28."

---

## Per-starter constraints

- Audit CSV has 11 columns: `pmid, accessed_at, AMBC_id, isRetracted, journalQualityJufo, citationCount, lookup_error, included_in_prescreen, threshold_min_jufo, threshold_min_citation_count, threshold_allow_retracted`. The trailing 3 `threshold_*` columns repeat run-time values per row for reviewer reproducibility.
- RIS file: standard RIS field mapping (`TY=JOUR, PMID, TI, AU, JO, JF, PY, DO, AB`). Only INCLUDED papers go in the RIS. Excluded papers only appear in the audit CSV.
- v0.1 ceiling: 5,000 PMIDs per run. Chunking (100 per batch) keeps lookup phase under rate-limit ceiling.
- v0.1 is lookup-only — no BiomedCore search-driven discovery (extension path).

---

## Kit conventions (apply to all Amass starters)

**Visual:**
- External IDs (PMID, DOI, NCT, `AMBC_`, `AMTC_`) in monospace — IBM Plex Mono. Prose in sans-serif — Inter. Both loaded from Google Fonts (`https://fonts.googleapis.com/css2?family=Inter:wght@400;600;700;800;900&family=IBM+Plex+Mono:wght@400;500;700&display=swap`).
- Neutral palette. shadcn-style tokens or Tailwind defaults. Dark mode follows OS `prefers-color-scheme`; no toggle.
- Lucide React for icons.
- Header: wordmark `amass` in Inter weight 800 at top-left.

**UX behavior:**
- Empty state shows a Try-sample button that loads the worked example identifiers verbatim.
- Long-running fan-outs render an analyst-visible Cancel button + progress indicator.
- Wrap fan-out async generators in `withIdleTimeout(gen, 180_000)`.
- Per-row error rendering: invalid identifiers show verbatim upstream error. Errors NEVER crash the surface.
- No `AskUserQuestion` gating on credentials — env-var prompt covers `AMASS_API_KEY` at runtime.

**Stack (TanStack Start + Vite — Lovable's default):**
- Server-only files use `.server.ts` filename. DO NOT add `import "server-only"`.
- Server functions: `src/lib/<name>.functions.ts` with `createServerFn({ method: "POST" }).inputValidator(...).handler(...)`.
- Route + page: `src/routes/index.tsx` with `createFileRoute("/")({ head, component: Index })`. The `component` binding is REQUIRED.
- `https://api.amass.tech` hardcoded. NO `AMASS_API_BASE_URL` env var.

---

## Pre-completion grep checklist — run BEFORE declaring scaffold done

The scaffold is only complete when every grep below returns ZERO. If any returns non-zero, fix the file and re-run before reporting success. These are exact mutations prior scaffolds in the kit have shipped — each one causes runtime failure or a misleading UI.

```
# Lovable blank-template leftovers (must be 0 — JSX errors / wrong page):
grep -rn 'data-lovable-blank-page-placeholder\|cdn\.gpteng\.co' src/   → expect 0
grep -rn '"Your App"\|Replace this with a one-sentence' src/    → expect 0
grep -rn 'flex min-h-screen items-center justify-center' src/   → expect 0

# Per-item lookup error mishandling (must be 0 — React #31 crash):
grep -rn 'r\.error\.code\b' src/ | grep -v '\.message'           → expect 0
grep -rn '{r.error}\|{result.error}\|{lookup_error}' src/        → expect 0  (no raw object renders)

# Server-only crash (must be 0 — crashes Vite SSR):
grep -rn 'import "server-only"\|from "server-only"' src/        → expect 0

# Required positive checks (must be ≥1):
grep -rn 'chunk(.*100)' src/                                    → expect ≥1  (lookup chunking)
grep -rn 'extractLookupResult' src/                             → expect ≥1
grep -rn 'component: Index' src/routes/                         → expect 1
grep -rn 'isRetracted\|journalQualityJufo\|citationCount' src/  → expect ≥1 each
```

If any check fails, fix it and re-run. Do NOT declare the scaffold complete with any non-zero count above.

---

## Validation

After `npm run dev`:

1. **Empty state** renders `<main>` root + amass wordmark + PMID textarea + 3 threshold controls (`min_jufo`, `allow_retracted`, `min_citation_count`) + Try-sample. Console clean (no errors / placeholder warnings / `import "server-only"` crashes).
2. **Try sample** loads the SCLC-DLL3-2026Q2 set (10 PMIDs: 37861218, 37355629, 39023700, 39876075, 41532856, 41256964, 41401368, 40884462, 41111642, 40419854). Click "Pre-screen". Completes in ~5-10 seconds. Results table renders one row per PMID with live JuFo tier + `isRetracted` flag + `citationCount`. Network tab shows 1 batch lookup + 10 per-paper GET calls.
3. **Anchor check** with default thresholds (`min_jufo=2`, `allow_retracted=false`, `min_citation_count=10`):
   - **Exactly 4 PMIDs** flip to `included_in_prescreen=true`: 37861218 (NEJM, JuFo 3, 390 cites), 37355629 (J Hematol Oncol, JuFo 2, 135 cites), 39023700 (Drugs, JuFo 2, 35 cites), 39876075 (Cancer, JuFo 2, 24 cites).
   - **Exactly 6 PMIDs** flip to `included_in_prescreen=false`. Of those, PMID 41532856 is the notable case: JuFo 3 (highest tier) but only 2 cites → excluded by the citation floor, NOT by JuFo.
4. **Threshold adjustment:** change `min_jufo` from 2 to 3 and re-run. Included count drops from 4 to 1 (only PMID 37861218 survives — it's the only JuFo-3 entry that also passes the citation floor). Page does NOT crash.
5. **Invalid PMID test:** add `99999999` to the textarea. Pre-screen completes. That row shows `lookup_error: NOT_FOUND` (or similar message from the live API). Other rows unaffected. Page does NOT crash.
6. **Download RIS:** file downloads. Open in a text editor. Contains valid RIS format (TY=JOUR, PMID, TI, AU, JO/JF, PY, DO, AB). Only INCLUDED papers appear; excluded ones absent.
7. **Download CSV:** file downloads. Has 11 columns: `pmid, accessed_at, AMBC_id, isRetracted, journalQualityJufo, citationCount, lookup_error, included_in_prescreen, threshold_min_jufo, threshold_min_citation_count, threshold_allow_retracted`. Threshold columns repeat run-time values per row for reviewer reproducibility.

---

## Hand-off

Build, lint, typecheck must pass.

**Credentials:** `AMASS_API_KEY` from https://platform.amass.tech. Run `npm run dev`.

**Want to extend it?**
- Cohort persistence — save pre-screen runs against a `project_id`.
- PRISMA-flow-diagram generator consuming the audit CSV.
- Field-specific JuFo overrides (different floor for clinical-trial vs basic-science papers).

Discord: https://discord.com/invite/sEGaBHMhWa.

---

## License

Apache-2.0. Copyright 2026 Amass Technologies. Source: https://github.com/lluisDTU/amass-technologies-amass-starters.

---

## Appendix — Next.js App Router alternative

For Claude Code / Cursor / Next.js production users:

- `lib/amass.ts` — same code as `src/lib/amass.server.ts` above, with `import "server-only"` added at the top.
- `app/api/pre-screen/route.ts` — standard Next.js POST handler with identical pre-screen logic.
- `app/page.tsx` — `"use client"` page POSTing to `/api/pre-screen`. Same `<main>` root, no outer wrapper.

Amass API contract, worked example, validation, kit conventions all apply identically.
