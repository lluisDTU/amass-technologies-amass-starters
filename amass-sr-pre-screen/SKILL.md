---
name: amass-sr-pre-screen
description: Use when building a PRISMA pre-screen credibility-filter for systematic reviews — batch-resolves a curated PMID dump to canonical Amass IDs, post-filters on JuFo + retraction + citation count, and emits a Rayyan/Covidence-importable RIS file plus an audit-trail CSV.
license: Apache-2.0
metadata:
  author: amass
  version: "0.4.0"
---

# SR Pre-Screen Credibility Filter

For systematic-review researchers running PRISMA 2020 — Cochrane groups, SR-as-a-service consultancies, academic teams. The user pastes a curated PMID dump from the upstream search step (typically ~5,000 PMIDs) and gets back a Rayyan/Covidence-importable RIS file containing only the ~100-500 papers that pass credibility thresholds (JuFo journal-quality tier, retraction flag, citation count). The load-bearing Amass primitives are the trust-filter conjunction on every paper record (`isRetracted` + `journalQualityJufo` + `citationCount` all default fields) plus per-item-error semantics on batch lookup — competitor stacks require Web of Science + Scopus + Retraction Watch + JuFo-table licensing to assemble equivalent signals.

---

## Scaffold target — read FIRST

This starter is built on **TanStack Start + Vite + TanStack Router + TanStack Query** (Lovable's default). THREE TypeScript files; do NOT wrap them in any outer template or placeholder component.

For Next.js App Router users, see the Appendix.

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

Input surface: textarea + 3 threshold controls. Empty state: Try-sample with ~10 illustrative GLP-1 PMIDs. Output: paginated results table + Download buttons. During fan-out: Cancel + progress indicator.

---

## API contract — load-bearing, follow exactly

### Endpoints used

- `POST /api/v1/cores/biomedcore/records/lookup` — batch resolve PMID → `AMBC_` (chunked to ~100 items per request)
- `GET /api/v1/cores/biomedcore/records/{amassId}` — fetch paper record with default fields (covers `isRetracted` + `journalQualityJufo` + `citationCount` — do NOT pass `include` flags)

### Auth + rate limit

`Authorization: Bearer ${AMASS_API_KEY}`. 60/60s. 429 → exponential backoff via `Retry-After`. 401/403 → surface directly.

### Response envelope

`{ "data": ... }` wrapped — unwrap before consuming. `POST /records/lookup` per-item:

```json
{ "input": { "pmid": "..." }, "amassIds": ["AMBC_..."] }
```

or

```json
{ "input": { "pmid": "..." }, "error": { "code": "NOT_FOUND", "message": "Identifier not found" } }
```

Per-item `error` is a STRUCTURED OBJECT — extract `.message` before rendering. NEVER render the error object directly (React #31 crashes).

### Top-level errors

Non-2xx body: `{ "error": { "code": ..., "message": ... } }`. The reference code's `req<T>` throws; handler `try/catch` re-throws so TanStack Query surfaces as `mutation.error.message`.

### Lookup chunking

`POST /records/lookup` `items[]` accepts up to ~100 items per request. Use the `chunk()` helper.

---

## Worked example

- **Review scope:** GLP-1 receptor agonists in obesity
- **Sample PMIDs (10 illustrative — re-verify against PubMed before binding to a real SR):** 36720262, 35658024, 34614329, 36331190, 35658026, 34170647, 35470291, 32109013, 32966830, 35658028
- **Default thresholds:** `min_jufo=2`, `allow_retracted=false`, `min_citation_count=10`

Try-sample tooltip: "GLP-1 receptor agonists in obesity — 10 illustrative PMIDs. Re-verify against PubMed before binding to a real SR."

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

## Validation

After `npm run dev`:

1. **Empty state** renders `<main>` root + amass wordmark + PMID textarea + 3 threshold controls + Try-sample. Console clean.
2. **Try sample** loads 10 GLP-1 PMIDs. Results table renders one row per PMID with live JuFo tier + `isRetracted` flag + `citationCount`.
3. **Threshold adjustment:** change `min_jufo` from 2 to 3 and re-run. Included count drops; rows with `journalQualityJufo < 3` flip to `included_in_prescreen=false`. Page does NOT crash.
4. **Download RIS + CSV:** both files emit cleanly. RIS contains only included papers in valid RIS format. CSV has all 11 columns with `threshold_*` replayed per row.

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
