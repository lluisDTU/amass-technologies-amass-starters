---
name: amass-pipeline-monitor
description: Use when building a weekly pipeline-monitoring digest for biotech competitive-intelligence analysts — takes a sponsor watchlist, surfaces new Phase 2/3 trials via per-sponsor TrialCore search with client-side phase + sponsorName filtering, and emits a single-panel sponsor-grouped digest. (v0.5: trials-only. Paper cross-core walk + retraction filter are v0.2 extensions.)
license: Apache-2.0
metadata:
  author: amass
  version: "0.5.0"
---

# Pipeline Monitor

For competitive-intelligence analysts at biotech and pharma companies tracking a watchlist of pharma sponsors and indication clusters. The user pastes a YAML watchlist of sponsors and gets a weekly single-panel digest of new Phase 2/3 trials per sponsor since `last_check`. The load-bearing Amass primitive is the canonical `AMTC_` ID — week-over-week diffs don't drift when CT.gov rewrites an NCT.

**v0.5 scope (trials-only).** The handler issues ONE TrialCore search per sponsor and filters results client-side by `phase` (must be in `watchlist.phases`) AND `sponsorName` (must contain the canonical or one of the aliases). This is intentionally minimal: the trial-list view is the headline feature, and per-watchlist response time stays under 5 seconds with one API call per sponsor. The trial→paper cross-core walk (`referencesBiomedCore` → `getBiomedRecord` for retraction/JuFo/citation signals) is a documented v0.2 extension — the API client retains `getTrialRecord` and `getBiomedRecord` methods so the extension is a UI-only addition. Retraction-centric storytelling lives in **P5** (Retraction Cascade Monitor), which is the correct surface for it.

---

## Scaffold target — read FIRST

This starter is built on **TanStack Start + Vite + TanStack Router + TanStack Query** (the default for Lovable). The scaffold produces THREE TypeScript files; do NOT wrap them in any outer template, placeholder component, or blank-app `<div>`.

For Next.js App Router users, see the Appendix at the bottom.

### Load-bearing constraints — DO NOT mutate these (read before transcribing)

Past scaffolds have failed because LLMs "auto-corrected" the spec. Each of the rules below is a hard constraint from the live Amass API or from Vite/TanStack Start semantics. Violating any of them causes a runtime 400 or a build error. Do NOT optimise, normalise, or "improve" them.

1. **Phase enum values contain `/` (slash), not `_` (underscore).** The literal strings `"PHASE1/PHASE2"` and `"PHASE2/PHASE3"` are intentional and valid TypeScript. Slashes inside a string literal are legal. Do NOT replace `/` with `_`. The Amass TrialCore API returns `400 BAD_REQUEST: Request validation failed` if the value is `PHASE2_PHASE3` instead of `PHASE2/PHASE3`. Verified against the live API and against https://platform.amass.tech/documentation/for-ai-agents/llm-quick-reference.md.
2. **The TrialCore `phase` query param accepts ONE value per request.** `TrialSearchOpts.phase` MUST be typed `Phase | undefined`, NOT `Phase[]`, NOT `ReadonlyArray<Phase>`, NOT `Set<Phase>`. The HTTP builder MUST use `params.set("phase", opts.phase)` exactly once, NOT a `for…append("phase", p)` loop. To filter across multiple phases, the CALLER iterates phases via `Promise.all(data.phases.map(...))` and merges results client-side by canonical `amassId`. Sending multiple `phase=` query params OR comma-separated `phase=A,B,C` returns 400.
3. **Replace files completely. Do NOT preserve any Lovable blank-template content.** Specifically, before declaring the scaffold complete, the following strings MUST appear ZERO times in `src/`:
   - `data-lovable-blank-page-placeholder`
   - `cdn.gpteng.co/blank-app-v1.svg`
   - `"Your App"` (in meta titles)
   - `"Replace this with a one-sentence description"`
   - Any `<div className="flex min-h-screen items-center justify-center">` wrapper around the root `<main>` or around any `Panel`/card component.
   And any single JSX element MUST have at most ONE `className=` attribute (duplicate `className` attributes are a JSX syntax error and React silently keeps only one).
4. **Server-only enforcement.** Use the `.server.ts` filename suffix (`src/lib/amass.server.ts`). Do NOT add `import "server-only"` — that's a Next.js package that crashes Vite SSR.
5. **Route component binding.** `createFileRoute("/")` MUST receive `{ head, component: Index }`. Omitting `component: Index` makes SSR render an empty page.

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

// Phase enum — Amass TrialCore API contract. Values are EXACT strings; do NOT modify.
// The slash `/` inside a string literal is valid TypeScript. Do NOT replace `/` with `_`.
// Verified against https://platform.amass.tech/documentation/for-ai-agents/llm-quick-reference.md
export const PHASES_ALL = [
  "EARLY_PHASE1",
  "PHASE1",
  "PHASE1/PHASE2",   // keep slash — API rejects PHASE1_PHASE2 with 400
  "PHASE2",
  "PHASE2/PHASE3",   // keep slash — API rejects PHASE2_PHASE3 with 400
  "PHASE3",
  "PHASE4",
  "NA",
] as const;
export type Phase = (typeof PHASES_ALL)[number];

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

The handler does ONE TrialCore search per sponsor and filters results client-side. No cross-core walk, no paper fetches. Total API call budget per Run digest = `sponsors.length` (typically 5).

```ts
// src/lib/digest.functions.ts
import { createServerFn } from "@tanstack/react-start";
import { z } from "zod";
import { getAmassClient } from "./amass.server";

// IMPORTANT: the slash `/` inside these string literals is intentional and valid TypeScript.
// Do NOT rewrite "PHASE1/PHASE2" or "PHASE2/PHASE3" as "PHASE1_PHASE2" / "PHASE2_PHASE3".
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

export const runDigest = createServerFn({ method: "POST" })
  .inputValidator((input: unknown) => WatchlistSchema.parse(input))
  .handler(async ({ data }) => {
    try {
      const client = getAmassClient();

      // Client-side sponsorName filter — drops fuzzy text-search noise (e.g. trials that
      // merely mention "Amgen" in interventionNames as a positive control comparator).
      const matchesSponsor = (trialSponsor: string, canonical: string, aliases: string[]) => {
        if (!trialSponsor) return false;
        const lower = trialSponsor.toLowerCase();
        return lower.includes(canonical.toLowerCase()) ||
          aliases.some((a) => lower.includes(a.toLowerCase()));
      };

      // ONE TrialCore search per sponsor — NO phase param, filter client-side.
      // Rationale:
      // - The API `phase` query param accepts ONE value per request. Looping client-side
      //   for N phases would be N× the calls per sponsor.
      // - minStartDate already narrows the result set to a 1-2 week window, well under
      //   the limit=300 ceiling for typical watchlists.
      // - Client-side filter on `t.phase ∈ watchlist.phases` AND `t.sponsorName matches
      //   canonical/alias` gives a tight, relevant result set.
      const sponsorResults = await Promise.all(
        data.sponsors.map(async (sponsor) => {
          try {
            const all = await client.searchTrialcore(sponsor.canonical, {
              minStartDate: data.last_check,
            });
            const filtered = (all as Array<Record<string, unknown>>).filter((t) => {
              const phase = (t.phase as string) ?? "";
              const sponsorName = (t.sponsorName as string) ?? "";
              const phaseOk = (data.phases as readonly string[]).includes(phase);
              const sponsorOk = matchesSponsor(sponsorName, sponsor.canonical, sponsor.aliases);
              return phaseOk && sponsorOk;
            });
            return {
              sponsor: sponsor.canonical,
              trials: filtered.map((t) => ({ amassId: (t.amassId as string) ?? "", record: t })),
              error: null as string | null,
            };
          } catch (e) {
            return {
              sponsor: sponsor.canonical,
              trials: [] as Array<{ amassId: string; record: Record<string, unknown> }>,
              error: e instanceof Error ? e.message : String(e),
            };
          }
        }),
      );

      return {
        watchlist_id: data.watchlist_id,
        last_check: data.last_check,
        digest: { sponsors: sponsorResults },
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
          <span className="text-xs text-muted-foreground font-mono">pipeline-monitor · v0.5</span>
        </div>
      </header>

      <section className="max-w-6xl mx-auto px-6 py-10 space-y-8">
        {/* YAML textarea + Try-sample + Run digest + single sponsor-grouped trials panel + Download Markdown digest */}
        {/* Each trial row: AMTC_ id (mono) · NCT_ id (mono) · phase · startDate / title */}
        {/* No paper panel, no retraction panel — those are v0.2 extensions */}
      </section>
    </main>
  );
}
```

---

## What you build

The user pastes a YAML sponsor watchlist. The handler issues ONE TrialCore search per sponsor via `GET /records?query=<sponsor>&minStartDate=<last_check>` (no `phase` filter) and filters the results client-side: `t.phase ∈ watchlist.phases` AND `t.sponsorName` contains the canonical or one of the aliases. Output is a Markdown digest grouped by sponsor rendered as a single sponsor-grouped panel.

Total API call budget per Run digest = `sponsors.length` (5 for the sample watchlist). Response time: 2-5 seconds.

Input surface: single YAML textarea. Empty state: Try-sample with the SCLC-DLL3-2026Q2 watchlist. Output: one sponsor-grouped trials panel + Download Markdown digest button. During fan-out: Cancel button + progress indicator (the fan-out is now small — ≤5 parallel calls — so this is rarely visible).

---

## API contract — load-bearing, follow exactly

**Authoritative source:** https://platform.amass.tech/documentation/for-ai-agents/llm-quick-reference.md — the live single-page reference for every endpoint, param, enum, and response schema across BiomedCore + TrialCore. If anything in this SKILL.md drifts from that doc, the doc wins.

### Endpoints used (v0.5 — trials only)

- `GET /api/v1/cores/trialcore/records?query=<sponsor>&minStartDate=<YYYY-MM-DD>&limit=300` — per-sponsor TrialCore search. Phase is filtered client-side from the result set; `sponsorName` is filtered client-side against canonical + aliases.

### Endpoints available for v0.2 extension (already wired in `amass.server.ts`, not called by v0.5 `runDigest`)

- `GET /api/v1/cores/trialcore/records/{amassId}?include=referencesBiomedCore` — fetch trial record with paper-side cross-core spine. Use for per-trial "Resolve papers" lazy-load in a v0.2 extension.
- `GET /api/v1/cores/biomedcore/records/{amassId}` — fetch paper record (default fields cover trust signals: `isRetracted`, `journalQualityJufo`, `citationCount`). Use for per-paper trust panel.

### Critical operational rules (load-bearing — the build fails without them)

- **Auth:** `Authorization: Bearer ${AMASS_API_KEY}` on every request. Get key from https://platform.amass.tech. Rate limit: 60 req / 60 s per user+org. On HTTP 429, read `Retry-After` and back off exponentially. On 401/403, surface the credential error directly.
- **Response envelope:** every endpoint returns `{ "data": ... }` wrapped. Unwrap before consuming.
- **Per-item lookup errors:** each `data[]` element on `POST /records/lookup` is either `{ amassIds: [...] }` (success) OR `{ error: { code, message } }` (per-item failure). The error field is a STRUCTURED OBJECT — empirically verified, even if the live docs example shows the simpler `{ error: "..." }` form. Always extract `.message` as a string before rendering. NEVER render the error object directly as a React child (React #31 crash). Example: `{ "input": { "pmid": "99999999" }, "error": { "code": "NOT_FOUND", "message": "Identifier not found" } }`.
- **Top-level errors:** non-2xx body is `{ "error": { "status", "code", "message" } }`. The reference code's `req<T>` throws with the upstream code+message; the route handler's outer `try/catch` re-throws so TanStack Query surfaces it as `mutation.error.message` on the client. NEVER let an uncaught throw propagate to the route-level error boundary.
- **Lookup before fetch:** `GET /records/{amassId}` accepts ONLY canonical `AMBC_` / `AMTC_` IDs. Passing PMID/DOI/NCT directly returns 404. Always: lookup → resolve canonical → fetch by canonical ID.
- **TrialCore `phase` param accepts ONE value per request.** Empirically verified: multiple `phase=` query params (or comma-separated `phase=A,B,C`) return `400 BAD_REQUEST: Request validation failed`. The compound enum values (`PHASE1/PHASE2`, `PHASE2/PHASE3`) ARE accepted as a single value — URL-encoded `%2F` is fine. In v0.5 this is moot because the handler does NOT pass a `phase` query param at all; it filters phase client-side from the unfiltered result set. The singleton-phase typing in `TrialSearchOpts` is preserved for v0.2 extension paths where a caller might want per-phase narrowing on the wire.
- **TrialCore `query` is fuzzy text search, NOT sponsor-exact.** A query for `Amgen` returns trials where "Amgen" appears in `briefTitle`, `briefSummary`, `interventionNames`, etc — including unrelated trials that mention Amgen drugs as positive-control comparators. Always filter results client-side on `sponsorName` against the canonical name + aliases to drop noise.

### TrialCore search ceiling

`limit` caps at 300 records. Sponsors returning ≥300 in a no-`minStartDate` probe (`query=<sponsor>&phase=...`) need their indication scope narrowed at onboarding.

---

## Worked example

- **Watchlist:** SCLC-DLL3-2026Q2
- **Sponsors:** Amgen, Roche (canonical for Genentech), AbbVie, Boehringer Ingelheim, Harpoon Therapeutics
- **Phases (client-side filter):** PHASE2, PHASE2/PHASE3, PHASE3
- **last_check:** 2026-05-13
- **Expected trial anchor:** NCT05060016 (DeLLphi-301, Amgen) surfaces in the Amgen panel with `phase: "PHASE2"` and `sponsorName` containing "Amgen"
- **Expected API call count:** 5 (one per sponsor)
- **Expected response time:** 2-5 seconds

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
- Sponsor cardinality probe: if `query=<canonical>&minStartDate=<2-weeks-ago>` returns ≥300 records (the API ceiling), narrow the watchlist's indication scope or shorten the `last_check` window. For typical biotech sponsors and a 1-week digest window this is rarely a problem.
- Single sponsor-grouped trials panel. Do NOT add a separate "papers" or "retraction" panel — those are v0.2 extensions (see Hand-off below) and the retraction story is owned by P5.
- Markdown digest filename: `digest-<watchlist_id>-<YYYY-Wnn>.md` (ISO week number).
- v0.5: single consolidated response per run. No digest-history persistence (extension path).

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
- Per-row error rendering: invalid identifiers show verbatim upstream error. Errors NEVER crash the surface.
- No `AskUserQuestion` gating on credentials — env-var prompt covers `AMASS_API_KEY` at runtime.

**Stack (TanStack Start + Vite — Lovable's default):**
- Server-only files use `.server.ts` filename (`src/lib/amass.server.ts`). DO NOT add `import "server-only"` — crashes Vite SSR.
- Server functions: `src/lib/<name>.functions.ts` with `createServerFn({ method: "POST" }).inputValidator(...).handler(...)`.
- Route + page: `src/routes/index.tsx` with `createFileRoute("/")({ head, component: Index })`. The `component` binding is REQUIRED.
- `https://api.amass.tech` hardcoded in the API client. NO `AMASS_API_BASE_URL` env var.

---

## Pre-completion grep checklist — run BEFORE declaring scaffold done

The scaffold is only complete when every grep below returns ZERO. If any returns non-zero, fix the file and re-run before reporting success. These are the exact mutations prior scaffolds have shipped — each one causes runtime failure.

```
# Underscore-form phase values (must be 0 — API rejects with 400):
grep -rn 'PHASE1_PHASE2\|PHASE2_PHASE3' src/                    → expect 0

# Multi-value phase typing (must be 0 — API rejects with 400):
grep -rn 'ReadonlyArray<Phase>\|phase?: Phase\[\]\|phase: Phase\[\]' src/   → expect 0
grep -rn 'opts\.phase ?? \[\]\|append("phase"' src/             → expect 0

# Lovable blank-template leftovers (must be 0 — JSX errors / wrong page):
grep -rn 'data-lovable-blank-page-placeholder\|cdn\.gpteng\.co' src/   → expect 0
grep -rn '"Your App"\|Replace this with a one-sentence' src/    → expect 0
grep -rn 'flex min-h-screen items-center justify-center' src/   → expect 0

# Server-only crash (must be 0 — crashes Vite SSR):
grep -rn 'import "server-only"\|from "server-only"' src/        → expect 0

# Required positive checks (must be ≥1):
grep -rn 'PHASE2/PHASE3' src/                                   → expect ≥1
grep -rn 'params\.set("phase", opts\.phase)' src/               → expect 1
grep -rn 'component: Index' src/routes/                         → expect 1
```

If any check fails, fix it and re-run. Do NOT declare the scaffold complete with any non-zero count above.

---

## Validation

After `npm run dev`:

1. **Empty state** renders the `<main>` root + amass wordmark + YAML textarea + Try-sample. Console clean (no errors / placeholder warnings / `import "server-only"` crashes).
2. **Try sample** loads SCLC-DLL3-2026Q2 watchlist. Click Run digest → completes in 2-5 seconds. The single trials panel populates with sponsor-grouped trial cards; under Amgen, **DeLLphi-301 (NCT05060016)** appears with `phase: "PHASE2"` and `sponsorName: "Amgen"`. Total API calls = 5 (verifiable in browser DevTools Network tab — one `GET /trialcore/records?query=<sponsor>&minStartDate=...` per sponsor, no `phase=` param).
3. **Sponsor name filter sanity:** under each sponsor card, every trial's `sponsorName` field actually contains the canonical sponsor name or alias. No "Amgen-mentioned-as-Neulasta-positive-control" Chinese trials appear under Amgen.
4. **Invalid sponsor test:** add `- canonical: Nonexistent Pharma Inc` to the watchlist YAML. Its per-sponsor block renders with zero new trials + a "No new trials since `<last_check>`" note. Page does NOT crash.
5. **Download Markdown digest** emits `digest-sclc-dll3-2026q2-<YYYY-Wnn>.md` with per-sponsor sections, one bullet per trial.

---

## Hand-off

Build, lint, typecheck must pass.

**Credentials:** get `AMASS_API_KEY` from https://platform.amass.tech and add to `.env`. Run `npm run dev`.

**Want to extend it? (v0.2 surface area)**
- **Per-trial paper resolution (lazy).** Add a "Resolve papers" button to each trial card. On click, call `client.getTrialRecord(amtcId, ["referencesBiomedCore"])` to get the AMBC_ paper IDs, then `client.getBiomedRecord(ambcId)` per paper for trust signals (`isRetracted`, `journalQualityJufo`, `citationCount`). The client methods are already wired — this is a UI-only addition. Cost: O(per-click), bounded by paper count for one trial.
- **Retraction watch alongside trials.** Filter the resolved papers by `isRetracted: true` and highlight the trial card. (Note: the broader retraction-cascade story is owned by **P5** — Retraction Cascade Monitor.)
- **Digest-history append-only persistence** — week-over-week diff in the dashboard.
- **Slack / email distribution** as an MCP slash-command.
- **Modality-cluster scoping** (e.g., "DLL3-targeted bispecifics in SCLC") as a saved indication query that supplements the per-sponsor search.

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
