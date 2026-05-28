---
name: amass-pipeline-monitor
description: Use when building a weekly pipeline-monitoring digest for biotech competitive-intelligence analysts — takes a sponsor watchlist, surfaces new Phase 2/3 trials via per-sponsor TrialCore search, walks the trial→paper cross-core edge, and emits a 3-panel weekly dashboard.
license: Apache-2.0
metadata:
  author: amass
  version: "0.4.3"
---

# Pipeline Monitor

For competitive-intelligence analysts at biotech and pharma companies tracking a watchlist of pharma sponsors and indication clusters. The user pastes a YAML watchlist of sponsors and gets a weekly 3-panel digest: new Phase 2/3 trials this week, new papers describing watched trials, and retraction-flagged citations. The load-bearing Amass primitives are canonical `AMTC_` IDs (week-over-week diffs don't drift when CT.gov rewrites an NCT) plus the trial→paper cross-core edge (`referencesBiomedCore`) — one Amass call surfaces paper-side context that PubMed + CT.gov + OpenAlex require 3-4 calls to assemble.

---

## Scaffold target — read FIRST

This starter is built on **TanStack Start + Vite + TanStack Router + TanStack Query** (the default for Lovable). The scaffold produces THREE TypeScript files; do NOT wrap them in any outer template, placeholder component, or blank-app `<div>`.

For Next.js App Router users, see the Appendix at the bottom.

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

// Phase enum — MUST match Amass TrialCore API exactly (with slashes, not underscores).
// Verified against https://platform.amass.tech/documentation/for-ai-agents/llm-quick-reference.md
export type Phase =
  | "EARLY_PHASE1"
  | "PHASE1"
  | "PHASE1/PHASE2"
  | "PHASE2"
  | "PHASE2/PHASE3"
  | "PHASE3"
  | "PHASE4"
  | "NA";

export interface TrialSearchOpts {
  phase?: Phase; // SINGLE phase per request — API rejects multiple `phase=` params with 400. Loop client-side.
  minStartDate?: string; // ISO YYYY-MM-DD
  sponsorType?: string;
  limit?: number; // default 300, ceiling 300
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
    if (opts.phase) params.set("phase", opts.phase); // SINGLE phase only — API rejects multi-value with 400
    if (opts.minStartDate) params.set("minStartDate", opts.minStartDate);
    if (opts.sponsorType) params.set("sponsorType", opts.sponsorType);
    params.set("limit", String(opts.limit ?? 300));
    return this.req<unknown[]>("GET", `/api/v1/cores/trialcore/records?${params}`);
  };

  getTrialRecord = (amassId: string, includes: ReadonlyArray<string> = []) => {
    const qs = includes.length ? `?${includes.map((i) => `include=${i}`).join("&")}` : "";
    return this.req<unknown>("GET", `/api/v1/cores/trialcore/records/${amassId}${qs}`);
  };

  getBiomedRecord = (amassId: string, includes: ReadonlyArray<string> = []) => {
    const qs = includes.length ? `?${includes.map((i) => `include=${i}`).join("&")}` : "";
    return this.req<unknown>("GET", `/api/v1/cores/biomedcore/records/${amassId}${qs}`);
  };
}

let _client: AmassClient | null = null;
export const getAmassClient = () =>
  (_client ??= new AmassClient(process.env.AMASS_API_KEY!));

export { AmassClient };
```

### File 2: `src/lib/digest.functions.ts` — TanStack Start server function

```ts
// src/lib/digest.functions.ts
import { createServerFn } from "@tanstack/react-start";
import { z } from "zod";
import { getAmassClient } from "./amass.server";

export const WatchlistSchema = z.object({
  watchlist_id: z.string(),
  sponsors: z.array(z.object({
    canonical: z.string(),
    aliases: z.array(z.string()).default([]),
  })).min(1),
  phases: z.array(z.enum([
    "EARLY_PHASE1", "PHASE1", "PHASE1/PHASE2", "PHASE2",
    "PHASE2/PHASE3", "PHASE3", "PHASE4", "NA",
  ])).default(["PHASE2", "PHASE2/PHASE3", "PHASE3"]),
  last_check: z.string(), // ISO YYYY-MM-DD
});

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

export const runDigest = createServerFn({ method: "POST" })
  .inputValidator((input: unknown) => WatchlistSchema.parse(input))
  .handler(async ({ data }) => {
    try {
      const client = getAmassClient();

      // Per-sponsor TrialCore search with minStartDate filter → array of new trial records.
      // CRITICAL: the `phase` param accepts ONE value per request. Loop phases client-side,
      // then merge by canonical amassId to dedupe.
      const sponsorResults = await Promise.all(
        data.sponsors.map(async (sponsor) => {
          const perPhase = await Promise.all(
            data.phases.map((phase) =>
              client.searchTrialcore(sponsor.canonical, { phase, minStartDate: data.last_check }),
            ),
          );
          const merged = new Map<string, unknown>();
          for (const batch of perPhase) {
            for (const t of batch as Array<{ amassId?: string }>) {
              if (t?.amassId) merged.set(t.amassId, t);
            }
          }
          return { sponsor: sponsor.canonical, trials: Array.from(merged.values()) };
        }),
      );

      // Per-trial cross-core walk: getTrialRecord with include=referencesBiomedCore → AMBC_ IDs.
      const allAmbcIds = new Set<string>();
      const trialToPapers = new Map<string, string[]>(); // amtcId → [ambcIds...]

      async function* walkTrials() {
        for (const { trials } of sponsorResults) {
          for (const trial of trials) {
            const amtcId = (trial as { amassId?: string }).amassId ?? "";
            if (!amtcId) continue;
            try {
              const rec = (await client.getTrialRecord(amtcId, ["referencesBiomedCore"])) as Record<string, unknown>;
              const refs = (rec["referencesBiomedCore"] as string[]) ?? [];
              trialToPapers.set(amtcId, refs);
              for (const ambc of refs) allAmbcIds.add(ambc);
              yield { amtcId };
            } catch (e) {
              yield { amtcId, error: e instanceof Error ? e.message : String(e) };
            }
          }
        }
      }
      for await (const _step of withIdleTimeout(walkTrials(), 180_000)) { /* progress */ }

      // Per-paper resolution: trust signals from default fields.
      const paperTrust = new Map<string, { isRetracted: boolean; jufo: number | null; cites: number | null }>();
      async function* walkPapers() {
        for (const ambcId of allAmbcIds) {
          try {
            const rec = (await client.getBiomedRecord(ambcId)) as Record<string, unknown>;
            paperTrust.set(ambcId, {
              isRetracted: Boolean(rec["isRetracted"]),
              jufo: (rec["journalQualityJufo"] as number) ?? null,
              cites: (rec["citationCount"] as number) ?? null,
            });
            yield { ambcId };
          } catch (e) {
            yield { ambcId, error: e instanceof Error ? e.message : String(e) };
          }
        }
      }
      for await (const _step of withIdleTimeout(walkPapers(), 180_000)) { /* progress */ }

      // Assemble 3-panel digest
      const newTrials = sponsorResults.flatMap(({ sponsor, trials }) =>
        trials.map((t) => ({ sponsor, trial: t })),
      );
      const newPapers = Array.from(allAmbcIds).map((ambcId) => ({ ambcId, ...paperTrust.get(ambcId) }));
      const retractionFlagged = newPapers.filter((p) => p.isRetracted);

      return {
        digest: { newTrials, newPapers, retractionFlagged, trialToPapers: Object.fromEntries(trialToPapers) },
        errors: {},
      };
    } catch (err) {
      throw new Error(err instanceof Error ? err.message : "Digest run failed");
    }
  });
```

### File 3: `src/routes/index.tsx` — TanStack Start route + page component

The `component: Index` binding is REQUIRED. The root element is `<main>` — no outer wrapper.

```tsx
// src/routes/index.tsx
import { createFileRoute } from "@tanstack/react-router";
import { useState } from "react";
import { useMutation } from "@tanstack/react-query";
import { useServerFn } from "@tanstack/react-start";
import { runDigest } from "@/lib/digest.functions";

export const Route = createFileRoute("/")({
  head: () => ({
    meta: [
      { title: "Pipeline Monitor — Amass" },
      { name: "description", content: "Weekly pipeline-monitoring digest for biotech CI analysts." },
    ],
  }),
  component: Index, // ← REQUIRED. Without this, SSR fails.
});

function Index() {
  const [yamlInput, setYamlInput] = useState("");
  const digest = useServerFn(runDigest);
  const mutation = useMutation({
    mutationFn: async (yamlText: string) => {
      // Parse YAML with js-yaml; pass to digest({ data: parsed })
      throw new Error("Implement YAML parse + digest call here");
    },
  });

  return (
    <main className="min-h-screen bg-background text-foreground font-sans">
      <header className="border-b border-border">
        <div className="max-w-6xl mx-auto px-6 py-4 flex items-center justify-between">
          <h1 className="text-2xl font-extrabold tracking-tight">amass</h1>
          <span className="text-xs text-muted-foreground font-mono">pipeline-monitor · v0.1</span>
        </div>
      </header>

      <section className="max-w-6xl mx-auto px-6 py-10 space-y-8">
        {/* YAML textarea + Try-sample + Run digest + 3-panel dashboard (new trials / new papers / retraction-flagged) + Download Markdown digest */}
      </section>
    </main>
  );
}
```

---

## What you build

The user pastes a YAML sponsor watchlist. The handler issues a per-sponsor TrialCore search via `GET /records?query=<sponsor>&phase=...&minStartDate=<last_check>` to surface new Phase 2/3 trials since the last check. For each surfaced trial, it walks the trial→paper cross-core edge via `GET /records/{AMTC_}?include=referencesBiomedCore`, then per-paper resolves trust signals via `GET /records/{AMBC_}`. The output is a Markdown digest grouped by sponsor rendered as a 3-panel dashboard.

Input surface: single YAML textarea. Empty state: Try-sample with the SCLC-DLL3-2026Q2 watchlist. Output: 3-panel dashboard + Download Markdown digest button. During fan-out: Cancel button + progress indicator.

---

## API contract — load-bearing, follow exactly

**Authoritative source:** https://platform.amass.tech/documentation/for-ai-agents/llm-quick-reference.md — the live single-page reference for every endpoint, param, enum, and response schema across BiomedCore + TrialCore. If anything in this SKILL.md drifts from that doc, the doc wins.

### Endpoints used

- `GET /api/v1/cores/trialcore/records?query=<sponsor>&phase=<ONE_PHASE>&minStartDate=<YYYY-MM-DD>&limit=300` — per-sponsor TrialCore search (loop phases client-side; see operational rules below)
- `GET /api/v1/cores/trialcore/records/{amassId}?include=referencesBiomedCore` — fetch trial record with paper-side cross-core spine
- `GET /api/v1/cores/biomedcore/records/{amassId}` — fetch paper record (default fields cover trust signals)

### Critical operational rules (load-bearing — the build fails without them)

- **Auth:** `Authorization: Bearer ${AMASS_API_KEY}` on every request. Get key from https://platform.amass.tech. Rate limit: 60 req / 60 s per user+org. On HTTP 429, read `Retry-After` and back off exponentially. On 401/403, surface the credential error directly.
- **Response envelope:** every endpoint returns `{ "data": ... }` wrapped. Unwrap before consuming.
- **Per-item lookup errors:** each `data[]` element on `POST /records/lookup` is either `{ amassIds: [...] }` (success) OR `{ error: { code, message } }` (per-item failure). The error field is a STRUCTURED OBJECT — empirically verified, even if the live docs example shows the simpler `{ error: "..." }` form. Always extract `.message` as a string before rendering. NEVER render the error object directly as a React child (React #31 crash). Example: `{ "input": { "pmid": "99999999" }, "error": { "code": "NOT_FOUND", "message": "Identifier not found" } }`.
- **Top-level errors:** non-2xx body is `{ "error": { "status", "code", "message" } }`. The reference code's `req<T>` throws with the upstream code+message; the route handler's outer `try/catch` re-throws so TanStack Query surfaces it as `mutation.error.message` on the client. NEVER let an uncaught throw propagate to the route-level error boundary.
- **Lookup before fetch:** `GET /records/{amassId}` accepts ONLY canonical `AMBC_` / `AMTC_` IDs. Passing PMID/DOI/NCT directly returns 404. Always: lookup → resolve canonical → fetch by canonical ID.
- **TrialCore `phase` param accepts ONE value per request.** Empirically verified: multiple `phase=` query params (or comma-separated `phase=A,B,C`) return `400 BAD_REQUEST: Request validation failed`. To filter across multiple phases (e.g. PHASE2 + PHASE2/PHASE3 + PHASE3), issue ONE search per phase per sponsor and merge results client-side by canonical `amassId`. The compound enum values (`PHASE1/PHASE2`, `PHASE2/PHASE3`) ARE accepted as a single value — URL-encoded `%2F` is fine.

### TrialCore search ceiling

`limit` caps at 300 records. Sponsors returning ≥300 in a no-`minStartDate` probe (`query=<sponsor>&phase=...`) need their indication scope narrowed at onboarding.

---

## Worked example

- **Watchlist:** SCLC-DLL3-2026Q2
- **Sponsors:** Amgen, Roche (canonical for Genentech), AbbVie, Boehringer Ingelheim, Harpoon Therapeutics
- **Phases:** PHASE2, PHASE2/PHASE3, PHASE3
- **last_check:** 2026-05-13
- **Expected trial anchor:** NCT05060016 (DeLLphi-301, Amgen) surfaces in the Amgen panel with `phase=PHASE2`
- **Expected paper anchor:** PMID 37861218 (Ahn MJ et al. NEJM 2023) surfaces via DeLLphi-301 cross-core walk

Try-sample tooltip: "SCLC-DLL3-2026Q2 — DLL3-targeted bispecifics in small-cell lung cancer." (Tarlatamab anchor set verified against PubMed + ClinicalTrials.gov + FDA approval letter at draft time)

Sample YAML:

```yaml
watchlist_id: sclc-dll3-2026q2
sponsors:
  - canonical: Amgen
    aliases: []
  - canonical: Roche
    aliases: [Genentech]
  - canonical: AbbVie
    aliases: []
  - canonical: Boehringer Ingelheim
    aliases: []
  - canonical: Harpoon Therapeutics
    aliases: []
phases: [PHASE2, PHASE2/PHASE3, PHASE3]
last_check: "2026-05-13"
```

---

## Per-starter constraints

- Sponsor-name canonicalisation: persist analyst-confirmed canonical→aliases map in the watchlist YAML. Genentech maps to Roche.
- Sponsor cardinality probe: for each phase, if `query=<canonical>&phase=<ONE_PHASE>` (no `minStartDate`) returns ≥300 records on any single phase, narrow the indication scope before the weekly run. (Reminder: the API rejects multi-value `phase=`; one phase per probe.)
- 3-panel dashboard: new trials (left), new papers via trial→paper cross-core walk (center), retraction-flagged citations (right). Do NOT collapse panels.
- Markdown digest filename: `digest-<watchlist_id>-<YYYY-Wnn>.md` (ISO week number).
- v0.1: single consolidated response per run. No digest-history persistence (extension path).

---

## Kit conventions (apply to all Amass starters)

**Visual:**
- External IDs (PMID, DOI, NCT, `AMBC_`, `AMTC_`) render in monospace — IBM Plex Mono. Prose in sans-serif — Inter. Both loaded from Google Fonts (`https://fonts.googleapis.com/css2?family=Inter:wght@400;600;700;800;900&family=IBM+Plex+Mono:wght@400;500;700&display=swap`).
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
- Server-only files use `.server.ts` filename (`src/lib/amass.server.ts`). DO NOT add `import "server-only"` — crashes Vite SSR.
- Server functions: `src/lib/<name>.functions.ts` with `createServerFn({ method: "POST" }).inputValidator(...).handler(...)`.
- Route + page: `src/routes/index.tsx` with `createFileRoute("/")({ head, component: Index })`. The `component` binding is REQUIRED.
- `https://api.amass.tech` hardcoded in the API client. NO `AMASS_API_BASE_URL` env var.

---

## Validation

After `npm run dev`:

1. **Empty state** renders the `<main>` root + amass wordmark + YAML textarea + Try-sample. Console clean (no errors / placeholder warnings / `import "server-only"` crashes).
2. **Try sample** loads SCLC-DLL3-2026Q2 watchlist. New-trials panel populates with at least DeLLphi-301 (NCT05060016) under Amgen. New-papers panel populates from cross-core walks. Retraction-flagged panel populates honestly.
3. **Invalid sponsor test:** add `Nonexistent Pharma Inc` to the watchlist. Its per-sponsor block renders with zero new trials + a note explaining empty result. Page does NOT crash.
4. **Download Markdown digest** emits `digest-sclc-dll3-2026q2-<YYYY-Wnn>.md` with per-sponsor sections.

---

## Hand-off

Build, lint, typecheck must pass.

**Credentials:** get `AMASS_API_KEY` from https://platform.amass.tech and add to `.env`. Run `npm run dev`.

**Want to extend it?**
- Add digest-history append-only persistence — week-over-week diff in the dashboard.
- Wire to Slack / email distribution as an MCP slash-command.
- Add modality-cluster scoping (e.g., "DLL3-targeted bispecifics in SCLC") as a saved indication query.

Discord: https://discord.com/invite/sEGaBHMhWa.

---

## License

Apache-2.0. Copyright 2026 Amass Technologies. Source: https://github.com/lluisDTU/amass-technologies-amass-starters.

---

## Appendix — Next.js App Router alternative

For Claude Code / Cursor users or anyone forking for production with Next.js conventions:

### `lib/amass.ts` (Next.js variant)

Same code as `src/lib/amass.server.ts` above, with `import "server-only"` added at the top. Next.js's `server-only` package guards against client-bundle imports at build time. Required for Next.js; do NOT use it on TanStack Start.

### `app/api/digest-run/route.ts`

```ts
import { getAmassClient } from "@/lib/amass";
import { WatchlistSchema } from "@/lib/schemas";

export async function POST(req: Request) {
  try {
    const watchlist = WatchlistSchema.parse(await req.json());
    // ... identical digest logic from runDigest's .handler() above ...
    return Response.json({ digest });
  } catch (err) {
    return Response.json({ error: err instanceof Error ? err.message : "Digest run failed" }, { status: 500 });
  }
}
```

### `app/page.tsx`

`"use client"` page POSTing to `/api/digest-run`. Same UI as the TanStack Start variant — single `<main>` root, no outer wrapper.

The Amass API contract, worked example, validation, and kit conventions apply identically.
