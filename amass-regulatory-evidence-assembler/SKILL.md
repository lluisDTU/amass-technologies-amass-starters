---
name: amass-regulatory-evidence-assembler
description: Use when building an auditable literature-evidence assembler for FDA / EMA regulatory submissions (CTD Module 2.5 / 2.7 / PSUR) — resolves PMID/DOI/NCT submission scope to canonical Amass IDs, walks the paper→trial cross-core edge, and emits an audit CSV with trust-filter columns.
license: Apache-2.0
metadata:
  author: amass
  version: "0.4.0"
---

# Regulatory Evidence Assembler

For medical writers, regulatory-affairs analysts, and evidence-synthesis CROs preparing CTD Module 2.5 / 2.7 narratives or PSUR-style ongoing-evidence audits. The user pastes a curated PMID/DOI/NCT submission scope and gets back an audit CSV where each cited paper is anchored to its supporting trial via canonical Amass IDs that don't drift when ClinicalTrials.gov rewrites an NCT between submission and approval. The load-bearing Amass primitive is canonical-ID stability — `AMBC_` and `AMTC_` IDs survive upstream-registry revisions, so the audit chain stays valid across the 6-12 month submission-to-approval window.

---

## Scaffold target — read FIRST

This starter is built on **TanStack Start + Vite + TanStack Router + TanStack Query** (the default for Lovable). The scaffold produces THREE TypeScript files; do NOT wrap them in any outer template, placeholder component, or blank-app `<div>`. The three files below are the COMPLETE structure.

For Next.js App Router users (Claude Code, Cursor, or anyone forking for production with Next.js conventions), see the Appendix at the bottom — it shows the equivalent `lib/amass.ts` + `app/api/audit-run/route.ts` + `app/page.tsx` shape.

### File 1: `src/lib/amass.server.ts` — server-only API client

The `.server.ts` filename is how TanStack Start's bundler enforces server-only — never bundled into the client. **Do NOT add `import "server-only"`** — that's a Next.js-specific package; it crashes Vite's SSR runtime because Vite imports it during SSR and the package throws.

```ts
// src/lib/amass.server.ts
// Server-only by virtue of the .server.ts filename. TanStack Start's bundler
// never includes this in the client bundle. DO NOT add `import "server-only"`.

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

  getBiomedRecord = (amassId: string, includes: ReadonlyArray<string> = []) => {
    const qs = includes.length ? `?${includes.map((i) => `include=${i}`).join("&")}` : "";
    return this.req<unknown>("GET", `/api/v1/cores/biomedcore/records/${amassId}${qs}`);
  };

  getTrialRecord = (amassId: string, includes: ReadonlyArray<string> = []) => {
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

### File 2: `src/lib/audit.functions.ts` — TanStack Start server function

```ts
// src/lib/audit.functions.ts
import { createServerFn } from "@tanstack/react-start";
import { z } from "zod";
import { getAmassClient, extractLookupResult, type LookupResult } from "./amass.server";

export const SubmissionScopeSchema = z.object({
  submission_id: z.string().optional(),
  drug: z.string().optional(),
  client_sponsor: z.string().optional(),
  indication: z.string().optional(),
  pmids: z.array(z.string()).default([]),
  dois: z.array(z.string()).default([]),
  ncts: z.array(z.string()).default([]),
});

export type AuditRow = {
  accessed_at: string;
  input_identifier: string;
  resolved_amassId: string;
  isRetracted: string;
  journalQualityJufo: string;
  citationCount: string;
  referencesTrialCore_AMTCs: string;
  lookup_error: string;
  walk_error: string;
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

export const runAudit = createServerFn({ method: "POST" })
  .inputValidator((input: unknown) => SubmissionScopeSchema.parse(input))
  .handler(async ({ data }) => {
    try {
      const client = getAmassClient();
      const accessed_at = new Date().toISOString();

      const pmidItems = data.pmids.map((pmid) => ({ pmid }));
      const doiItems = data.dois.map((doi) => ({ doi }));
      const nctItems = data.ncts.map((nctId) => ({ nctId }));

      const [pmidRes, doiRes, nctRes] = await Promise.all([
        pmidItems.length ? client.batchLookupBiomed(pmidItems) : Promise.resolve([] as LookupResult),
        doiItems.length ? client.batchLookupBiomed(doiItems) : Promise.resolve([] as LookupResult),
        nctItems.length ? client.batchLookupTrial(nctItems) : Promise.resolve([] as LookupResult),
      ]);

      const rows: AuditRow[] = [];
      const biomedToRow = new Map<string, number>();
      const blank = (input_identifier: string, resolved_amassId = "", lookup_error = ""): AuditRow => ({
        accessed_at,
        input_identifier,
        resolved_amassId,
        isRetracted: "",
        journalQualityJufo: "",
        citationCount: "",
        referencesTrialCore_AMTCs: "",
        lookup_error,
        walk_error: "",
      });

      extractLookupResult(pmidItems, pmidRes,
        (input, ids) => {
          const id = ids[0];
          const idx = rows.push(blank(`PMID:${input.pmid}`, id)) - 1;
          if (id.startsWith("AMBC_")) biomedToRow.set(id, idx);
          return null;
        },
        (input, msg) => { rows.push(blank(`PMID:${input.pmid}`, "", msg)); return null; },
      );
      extractLookupResult(doiItems, doiRes,
        (input, ids) => {
          const id = ids[0];
          const idx = rows.push(blank(`DOI:${input.doi}`, id)) - 1;
          if (id.startsWith("AMBC_")) biomedToRow.set(id, idx);
          return null;
        },
        (input, msg) => { rows.push(blank(`DOI:${input.doi}`, "", msg)); return null; },
      );
      extractLookupResult(nctItems, nctRes,
        (input, ids) => { rows.push(blank(`NCT:${input.nctId}`, ids[0])); return null; },
        (input, msg) => { rows.push(blank(`NCT:${input.nctId}`, "", msg)); return null; },
      );

      // Per-paper fan-out — fetch referencesTrialCore + citedBy on each resolved AMBC_
      const ambcIds = Array.from(biomedToRow.keys());
      async function* walkGen() {
        for (const id of ambcIds) {
          try {
            const rec = (await client.getBiomedRecord(id, ["referencesTrialCore", "citedBy"])) as Record<string, unknown>;
            yield { id, rec, error: null as string | null };
          } catch (e) {
            yield { id, rec: null, error: e instanceof Error ? e.message : String(e) };
          }
        }
      }

      for await (const step of withIdleTimeout(walkGen(), 180_000)) {
        const idx = biomedToRow.get(step.id)!;
        if (step.error) { rows[idx].walk_error = step.error; continue; }
        const rec = step.rec as Record<string, unknown>;
        const retracted = rec["isRetracted"];
        rows[idx].isRetracted = typeof retracted === "boolean" ? String(retracted) : retracted == null ? "" : String(retracted);
        rows[idx].journalQualityJufo = rec["journalQualityJufo"] == null ? "" : String(rec["journalQualityJufo"]);
        rows[idx].citationCount = rec["citationCount"] == null ? "" : String(rec["citationCount"]);
        const refs = rec["referencesTrialCore"];
        rows[idx].referencesTrialCore_AMTCs = Array.isArray(refs) && refs.length
          ? refs.map((r) => typeof r === "string" ? r : (r as { amassId?: string })?.amassId ?? "").filter(Boolean).join("|")
          : "[]";
      }

      return { rows };
    } catch (err) {
      throw new Error(err instanceof Error ? err.message : "Audit run failed");
    }
  });
```

### File 3: `src/routes/index.tsx` — TanStack Start route + page component

The `createFileRoute("/")({ ... })` call MUST include `component: Index`. Without this binding, the route has no component, SSR fails with "SSR rendering failed", and the published deploy shows a generic error page.

The page component's root element is the `<main>` below — do NOT wrap it in any outer `<div className="flex min-h-screen items-center justify-center">` or `<img data-lovable-blank-page-placeholder>` template. Those are Lovable scaffold artifacts that must NOT appear in the final file.

```tsx
// src/routes/index.tsx
import { createFileRoute } from "@tanstack/react-router";
import { useState } from "react";
import { useMutation } from "@tanstack/react-query";
import { useServerFn } from "@tanstack/react-start";
import { runAudit, type AuditRow } from "@/lib/audit.functions";
// import lucide-react icons as needed for the UI (FileText, Download, Copy, Play, Loader2, X)

export const Route = createFileRoute("/")({
  head: () => ({
    meta: [
      { title: "Regulatory Evidence Assembler — Amass" },
      {
        name: "description",
        content:
          "Auditable literature-evidence assembler for FDA / EMA submissions. Resolves PMID/DOI/NCT to canonical Amass IDs and emits a 9-column audit CSV.",
      },
    ],
  }),
  component: Index, // ← REQUIRED. Without this, SSR fails.
});

function Index() {
  const [input, setInput] = useState("");
  const [parseError, setParseError] = useState<string | null>(null);
  const audit = useServerFn(runAudit);

  const mutation = useMutation({
    mutationFn: async (yamlText: string) => {
      // Parse YAML with js-yaml; pass to audit({ data: parsed }).
      // See package.json — add "js-yaml" + "@types/js-yaml" dependencies.
      throw new Error("Implement YAML parse + audit call here");
    },
  });

  // Design the rest of the UI freely within the kit conventions:
  // - Try-sample button loading the worked example identifiers verbatim
  // - YAML textarea for the submission scope
  // - Run audit button + Cancel button during fan-out
  // - Results table with 9 columns
  // - Download audit CSV button + Copy citations button
  // - Per-row error rendering: lookup_error + walk_error columns in destructive color
  // - All identifiers (PMID/DOI/NCT/AMBC_/AMTC_) in monospace (font-mono / IBM Plex Mono)
  // - All prose in sans-serif (font-sans / Inter)

  return (
    <main className="min-h-screen bg-background text-foreground font-sans">
      <header className="border-b border-border">
        <div className="max-w-6xl mx-auto px-6 py-4 flex items-center justify-between">
          <h1 className="text-2xl font-extrabold tracking-tight">amass</h1>
          <span className="text-xs text-muted-foreground font-mono">
            regulatory-evidence-assembler · v0.1
          </span>
        </div>
      </header>

      <section className="max-w-6xl mx-auto px-6 py-10 space-y-8">
        {/* Per-starter UI: title + intro + YAML textarea + Try-sample + Run audit + results table */}
      </section>
    </main>
  );
}
```

That's the complete scaffold target. Three files: API client, server function, route + page. NO outer wrapper. NO placeholder img. NO `import "server-only"`. The `component: Index` binding is mandatory.

---

## What you build

The user pastes a YAML submission scope listing PMIDs, DOIs, and NCTs from upstream discovery (PubHive / Embase / Scopus). The handler runs three concurrent batch lookups — `POST /records/lookup` for PMIDs+DOIs against BiomedCore, plus `POST /records/lookup` for NCTs against TrialCore — resolving every external identifier to a canonical `AMBC_` or `AMTC_` ID with per-item error preservation. For each resolved BiomedCore record, it then issues `GET /records/{amassId}?include=referencesTrialCore,citedBy` to walk the paper→trial cross-core edge and retrieve trust-filter signals (`isRetracted`, `journalQualityJufo`, `citationCount`). The output is a 9-column audit CSV anchored to canonical Amass IDs that survive NCT-registry revisions between submission and approval.

The input surface is a single textarea accepting YAML. The empty state offers a Try-sample button that loads the verified Tarlatamab worked example identifiers verbatim. The output renders as a results table with a Download audit CSV button and a Copy citations button. While the per-paper fan-out runs, the UI shows a Cancel button and a progress indicator.

---

## API contract — load-bearing, follow exactly

### Endpoints used

- `POST /api/v1/cores/biomedcore/records/lookup` — batch resolve PMID/DOI → `AMBC_`
- `POST /api/v1/cores/trialcore/records/lookup` — batch resolve NCT → `AMTC_`
- `GET /api/v1/cores/biomedcore/records/{amassId}?include=referencesTrialCore,citedBy` — fetch paper record with cross-core trial spine + citation graph

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

Non-2xx HTTP response body: `{ "error": { "status": ..., "code": ..., "message": ... } }`. The reference code's `req<T>` method throws with the upstream message; the server function's `try/catch` re-throws so TanStack Query surfaces it as `mutation.error.message` on the client. NEVER let an uncaught throw propagate to the route-level error boundary.

### Lookup-before-fetch discipline

`GET /records/{amassId}` accepts ONLY canonical `AMBC_` / `AMTC_` IDs. Passing a PMID / DOI / NCT directly returns 404. Always: lookup first → resolve canonical → fetch by canonical ID.

---

## Worked example

Use these verified identifiers in the empty-state Try-sample button. Render ALL trust-filter values from the live Amass response — NEVER hardcode them.

- **Submission:** BLA 761344-dlle (tarlatamab / Imdelltra, Amgen, ES-SCLC)
- **PMID 37861218** (Ahn MJ et al., NEJM 2023, DeLLphi-301 primary publication) — expected `isRetracted=false`, JuFo tier 3 (NEJM)
- **PMID 37355629** (Rudin et al., J Hematol Oncol 2023, DLL3 review) — expected `isRetracted=false`, expected `referencesTrialCore=[]` (review article — honest emptiness)
- **NCT05060016** (DeLLphi-301)
- **NCT05740566** (DeLLphi-304)

Sample YAML for the Try-sample button:

```yaml
submission_id: BLA 761344-dlle
drug: tarlatamab (Imdelltra)
client_sponsor: Amgen
indication: ES-SCLC
pmids:
  - "37861218"
  - "37355629"
dois: []
ncts:
  - NCT05060016
  - NCT05740566
```

Try-sample tooltip: "Tarlatamab (Imdelltra) / Amgen — Ahn NEJM 2023 + Rudin J Hematol Oncol 2023; DeLLphi-301 + DeLLphi-304." (verified against PubMed + ClinicalTrials.gov + FDA approval letter at draft time)

---

## Per-starter constraints

- The audit CSV has exactly these 9 columns: `accessed_at, input_identifier, resolved_amassId, isRetracted, journalQualityJufo, citationCount, referencesTrialCore_AMTCs, lookup_error, walk_error`. Do not add or rename columns.
- `lookup_error` and `walk_error` are SEPARATE columns — never collapse. Lookup-side failures (PMID retired, DOI revoked, NCT merge) populate `lookup_error`; per-record GET failures after a successful lookup populate `walk_error`.
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

**Stack (TanStack Start + Vite — Lovable's default):**
- Server-only files use the `.server.ts` filename suffix (`src/lib/amass.server.ts`). The bundler enforces server-only based on filename. Do NOT add `import "server-only"` — that's Next.js's mechanism; it crashes Vite SSR.
- Server functions go in `src/lib/<name>.functions.ts` and use `createServerFn({ method: "POST" }).inputValidator(...).handler(...)`.
- Route + page component goes in `src/routes/index.tsx` and MUST include `component: Index` on the `createFileRoute(...)` config.
- `https://api.amass.tech` is hardcoded as the base URL in the client class. Do NOT introduce an `AMASS_API_BASE_URL` env var.
- Pin React 18 or 19 (whichever Lovable defaults to), TypeScript 5.x, zod 3.x, lucide-react, `@tanstack/react-router`, `@tanstack/react-start`, `@tanstack/react-query`. Add `js-yaml` + `@types/js-yaml` for the YAML input.

---

## Validation

After `npm run dev`, verify in order:

1. **Empty state** renders cleanly: the `<main>` root, the header with `amass` wordmark, the YAML textarea, the Try-sample button. Browser console clean (no errors / warnings about `data-lovable-blank-page-placeholder` or unclosed JSX or `import "server-only"`).
2. **Try sample** loads the Tarlatamab YAML and runs end-to-end. Rendered rows show: 4 rows resolved to canonical AMBC_/AMTC_ IDs; PMID 37861218 row has `isRetracted=false`, `journalQualityJufo=3` (NEJM), `referencesTrialCore_AMTCs` populated with the DeLLphi-301 `AMTC_` ID; PMID 37355629 row has empty `referencesTrialCore_AMTCs` (review article).
3. **Invalid PMID test:** paste `99999999` as an additional PMID and re-run. The corresponding row renders inline with `lookup_error` populated by the verbatim upstream message (e.g. `"Identifier not found"`). Page does NOT crash; the rest of the audit completes.
4. **Download audit CSV** click emits a file with the 9-column header line and one row per submitted identifier.

---

## Hand-off

Build, lint, typecheck must pass.

**Credentials:** get your `AMASS_API_KEY` from https://platform.amass.tech and add it to `.env`.

To run: `npm run dev`.

**Want to extend it?**
- Add a monthly re-audit cron with append-only audit-CSV history — catches newly-retracted citations between submission (t0) and approval (tN) across the PSUR lifecycle, anchored to canonical `AMBC_` IDs.
- Add a retraction-review queue with ack-event lifecycle (`acked_at`, `acked_by`, `ack_decision`) durably persisted across worker restarts.
- Add a Python CLI sidecar for headless cron invocation against the same `lib/amass.server.ts` semantics.

Questions? Join the Amass Developer Community on Discord: https://discord.com/invite/sEGaBHMhWa.

---

## License

Apache-2.0. Copyright 2026 Amass Technologies. Source: https://github.com/lluisDTU/amass-technologies-amass-starters.

---

## Appendix — Next.js App Router alternative

For Claude Code / Cursor users or anyone forking for production with Next.js conventions, the equivalent three-file structure:

### `lib/amass.ts` (Next.js variant)

Same code as `src/lib/amass.server.ts` above, with `import "server-only"` added at the top:

```ts
import "server-only";
// ... rest identical to the TanStack Start variant
```

Next.js's `server-only` package guards against accidental client-bundle imports at BUILD time. Required for Next.js; do NOT use it on TanStack Start (Vite SSR crashes).

### `app/api/audit-run/route.ts`

Replace the TanStack Start `createServerFn` with a standard Next.js App Router POST handler:

```ts
import { AmassClient, getAmassClient, extractLookupResult, type LookupResult } from "@/lib/amass";
import { SubmissionScopeSchema } from "@/lib/schemas";

export async function POST(req: Request) {
  try {
    const scope = SubmissionScopeSchema.parse(await req.json());
    // ... identical audit logic from runAudit's .handler() above ...
    return Response.json({ rows });
  } catch (err) {
    return Response.json({ error: err instanceof Error ? err.message : "Audit run failed" }, { status: 500 });
  }
}
```

### `app/page.tsx`

A `"use client"` page component that POSTs to `/api/audit-run`:

```tsx
"use client";
import { useState } from "react";
import { useMutation } from "@tanstack/react-query";

export default function Page() {
  const mutation = useMutation({
    mutationFn: async (yamlText: string) => {
      const res = await fetch("/api/audit-run", { method: "POST", body: JSON.stringify(/* parsed YAML */) });
      if (!res.ok) throw new Error((await res.json()).error ?? "Audit failed");
      return res.json();
    },
  });
  // Same UI as the TanStack Start variant — single <main> root, no outer wrapper.
}
```

The Amass API contract, worked example, validation steps, and kit conventions all apply identically to the Next.js variant.
