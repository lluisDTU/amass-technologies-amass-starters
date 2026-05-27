---
name: amass-retraction-cascade-monitor
description: Use when building a retraction-cascade tracker that re-audits a stored bibliography for newly retracted citations AND surfaces the downstream cascade — inbound citedBy, outbound references, and the cross-core trial spine — all in one record fetch.
license: Apache-2.0
metadata:
  author: amass
  version: "0.4.0"
---

# Retraction Cascade Monitor

For three ICPs sharing the same workflow shape: systematic-review teams re-auditing a published review's bibliography for citations retracted since publication; regulatory-affairs analysts monitoring a submission's citation chain between submission and approval; medical-affairs teams monitoring a published medical-information answer for citation contamination over time. The user pastes a curated bibliography PMID list and gets a 3-panel cascade dashboard. The load-bearing Amass primitive is the three-direction-in-one-record-fetch conjunction — `isRetracted` (default field) + `references` + `citedBy` + `referencesTrialCore` all delivered in a single `GET` with one `include=` set, which OpenAlex and Scite cannot match in one HTTP call.

---

## Scaffold target — read FIRST

This starter is built on **TanStack Start + Vite + TanStack Router + TanStack Query** (Lovable's default). THREE TypeScript files; do NOT wrap them in any outer template or placeholder component.

For Next.js App Router users, see the Appendix.

### File 1: `src/lib/amass.server.ts` — server-only API client

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
```

### File 2: `src/lib/cascade.functions.ts` — TanStack Start server function

```ts
// src/lib/cascade.functions.ts
import { createServerFn } from "@tanstack/react-start";
import { z } from "zod";
import { getAmassClient, extractLookupResult, type LookupResult } from "./amass.server";

export const CascadeRequestSchema = z.object({
  pmids: z.array(z.string()).min(1),
});

export type RetractedPaper = {
  ambcId: string;
  pmid: string;
  journal: string;
  isRetracted: boolean;
  inboundCount: number;
};

export type CascadeResponse = {
  retracted: RetractedPaper[];
  inbound: Array<{ ambcId: string; sourceAmbcId: string }>;
  outbound: Array<{ ambcId: string; sourceAmbcId: string; kind: "reference" | "trial" }>;
  lookup_errors: Array<{ pmid: string; message: string }>;
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

export const runCascadeAudit = createServerFn({ method: "POST" })
  .inputValidator((input: unknown) => CascadeRequestSchema.parse(input))
  .handler(async ({ data }): Promise<CascadeResponse> => {
    try {
      const client = getAmassClient();

      const pmidItems = data.pmids.map((pmid) => ({ pmid }));
      const lookupResults = await client.batchLookupBiomed(pmidItems);

      const ambcIds: string[] = [];
      const ambcToPmid = new Map<string, string>();
      const lookupErrors: CascadeResponse["lookup_errors"] = [];

      extractLookupResult(pmidItems, lookupResults,
        (input, ids) => { ambcIds.push(ids[0]); ambcToPmid.set(ids[0], input.pmid!); return null; },
        (input, msg) => { lookupErrors.push({ pmid: input.pmid!, message: msg }); return null; },
      );

      // Per-paper fan-out with the LOAD-BEARING 3-direction include set.
      const records = new Map<string, Record<string, unknown>>();
      async function* walkGen() {
        for (const ambcId of ambcIds) {
          try {
            const rec = (await client.getBiomedRecord(ambcId, [
              "references", "citedBy", "referencesTrialCore",
            ])) as Record<string, unknown>;
            records.set(ambcId, rec);
            yield { ambcId };
          } catch (e) {
            yield { ambcId, error: e instanceof Error ? e.message : String(e) };
          }
        }
      }
      for await (const _step of withIdleTimeout(walkGen(), 180_000)) { /* progress */ }

      // Filter retracted subset + assemble cascade
      const retracted: RetractedPaper[] = [];
      const inbound: CascadeResponse["inbound"] = [];
      const outbound: CascadeResponse["outbound"] = [];

      for (const [ambcId, rec] of records.entries()) {
        if (!rec["isRetracted"]) continue;
        const citedBy = (rec["citedBy"] as string[]) ?? [];
        const refs = (rec["references"] as string[]) ?? [];
        const refsTC = (rec["referencesTrialCore"] as string[]) ?? [];
        retracted.push({
          ambcId,
          pmid: ambcToPmid.get(ambcId) ?? "",
          journal: String(rec["journal"] ?? ""),
          isRetracted: true,
          inboundCount: citedBy.length,
        });
        for (const c of citedBy) inbound.push({ ambcId: c, sourceAmbcId: ambcId });
        for (const r of refs) outbound.push({ ambcId: r, sourceAmbcId: ambcId, kind: "reference" });
        for (const t of refsTC) outbound.push({ ambcId: t, sourceAmbcId: ambcId, kind: "trial" });
      }

      return { retracted, inbound, outbound, lookup_errors: lookupErrors };
    } catch (err) {
      throw new Error(err instanceof Error ? err.message : "Cascade audit failed");
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
import { runCascadeAudit } from "@/lib/cascade.functions";

export const Route = createFileRoute("/")({
  head: () => ({
    meta: [
      { title: "Retraction Cascade Monitor — Amass" },
      { name: "description", content: "3-direction retraction cascade dashboard with live isRetracted." },
    ],
  }),
  component: Index, // ← REQUIRED. Without this, SSR fails.
});

function Index() {
  const [pmidsInput, setPmidsInput] = useState("");
  const audit = useServerFn(runCascadeAudit);
  const mutation = useMutation({
    mutationFn: async () => audit({ data: { pmids: pmidsInput.split(/\s+/).filter(Boolean) } }),
  });

  return (
    <main className="min-h-screen bg-background text-foreground font-sans">
      <header className="border-b border-border">
        <div className="max-w-6xl mx-auto px-6 py-4 flex items-center justify-between">
          <h1 className="text-2xl font-extrabold tracking-tight">amass</h1>
          <span className="text-xs text-muted-foreground font-mono">retraction-cascade-monitor · v0.1</span>
        </div>
      </header>

      <section className="max-w-6xl mx-auto px-6 py-10 space-y-8">
        {/* PMID textarea + Try-sample + Run cascade audit + 3-panel cascade dashboard (retracted | inbound citedBy | outbound references + cross-core trials) + Download report (Markdown/JSON) */}
      </section>
    </main>
  );
}
```

---

## What you build

The user pastes a bibliography PMID list (typical SR cohort: 100-500 PMIDs). The handler batch-resolves all PMIDs to canonical `AMBC_` IDs, then for each resolved paper issues `GET /records/{amassId}?include=references,citedBy,referencesTrialCore` — a single call that delivers FOUR signals: `isRetracted` (default field), outbound `references`, inbound `citedBy`, and cross-core `referencesTrialCore`. Filter to `isRetracted === true` and assemble a 3-panel cascade dashboard: retracted papers (left), inbound `citedBy` cascade (center — downstream papers carrying contamination), outbound `references` + cross-core `referencesTrialCore` (right).

Input surface: PMID textarea (one per line). Empty state: Try-sample with Wakefield worked example. Output: 3-panel dashboard + Download cascade report (Markdown or JSON). During fan-out: Cancel + progress indicator.

---

## API contract — load-bearing, follow exactly

### Endpoints used

- `POST /api/v1/cores/biomedcore/records/lookup` — batch resolve PMID → `AMBC_`
- `GET /api/v1/cores/biomedcore/records/{amassId}?include=references,citedBy,referencesTrialCore` — fetch paper record with all three cascade directions in one call

### Auth + rate limit

`Authorization: Bearer ${AMASS_API_KEY}`. 60/60s rate limit. 429 → exponential backoff. 401/403 → surface directly.

### Response envelope

`{ "data": ... }` wrapped. `POST /records/lookup` per-item is structured `{ amassIds: [...] }` or `{ error: { code, message } }`. NEVER render the error object directly (React #31 crashes).

### Top-level errors

Non-2xx body: `{ "error": { "code", "message" } }`. The reference code throws; handler `try/catch` surfaces as `mutation.error.message`.

### Three-direction-in-one-call discipline (load-bearing)

The single-record-fetch with `include=references,citedBy,referencesTrialCore` is the LOAD-BEARING discipline. DO NOT split into multiple per-direction calls. Collapsing the include set across two HTTP calls defeats the value proposition.

---

## Worked example

- **Anchor:** PMID 9500320 (Wakefield et al., Lancet 1998, fully retracted 2010-02-06) — expected `isRetracted=true`, expected inbound `citedBy` cascade with several thousand downstream entries
- **Context (10 illustrative PMIDs from autism-vaccination methodology literature — re-verify against PubMed before binding to a real audit):** 18039999, 19720750, 16352714, 16456141, 14738625, 11241983, 15585572, 18207565, 19127374, 19651677

Try-sample tooltip: "Wakefield et al. Lancet 1998 (PMID 9500320) — canonical retracted paper. Fully retracted 2010-02-06. Re-verify identifier set against PubMed + Retraction Watch before binding to a real audit."

---

## Per-starter constraints

- 3-panel structure (retracted | inbound | outbound + cross-core) is the load-bearing UX. All three panels populate from the SAME record fetch.
- `isRetracted` renders directly from live Amass response. NEVER hardcode retraction flags — defamation-adjacent if a real paper is shown with wrong status.
- Empty arrays in any direction render as honest emptiness (no downstream citers, no outbound references, no trial references).
- Downloadable cascade report: Markdown OR JSON selectable from UI.
- v0.1 is bibliography-snapshot-driven (caller's PMID list at t0). No BiomedCore search-driven discovery (extension path).

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

1. **Empty state** renders `<main>` root + amass wordmark + PMID textarea + Try-sample. Console clean.
2. **Try sample** loads Wakefield bibliography. Left panel: 1 retracted paper (live `isRetracted=true`). Center panel: downstream citers. Right panel: outbound + cross-core (may be empty — render as honest emptiness).
3. **Invalid PMID test:** add `99999999`. Goes to `lookup_errors` with verbatim upstream message. Page does NOT crash; rest of cascade completes.
4. **Download cascade report** (Markdown variant): file saves with nested headers per retracted paper + bulleted lists per cascade direction.

---

## Hand-off

Build, lint, typecheck must pass.

**Credentials:** `AMASS_API_KEY` from https://platform.amass.tech. Run `npm run dev`.

**Want to extend it?**
- Cron-based re-audit cadence (weekly/monthly) + append-only audit log + newly-retracted alerts.
- Search-driven discovery mode auto-detecting newly-retracted papers across a topic area.
- Retraction-acknowledgement letter generator using assembled cascade as input.

Discord: https://discord.com/invite/sEGaBHMhWa.

---

## License

Apache-2.0. Copyright 2026 Amass Technologies. Source: https://github.com/lluisDTU/amass-technologies-amass-starters.

---

## Appendix — Next.js App Router alternative

For Claude Code / Cursor / Next.js production users:

- `lib/amass.ts` — same code as `src/lib/amass.server.ts` above, with `import "server-only"` added at the top.
- `app/api/cascade-audit/route.ts` — standard Next.js POST handler with identical cascade logic.
- `app/page.tsx` — `"use client"` page POSTing to `/api/cascade-audit`. Same `<main>` root, no outer wrapper.

Amass API contract, worked example, validation, kit conventions all apply identically.
