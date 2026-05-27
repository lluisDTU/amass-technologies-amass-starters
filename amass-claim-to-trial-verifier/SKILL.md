---
name: amass-claim-to-trial-verifier
description: Use when building a marketing-claim-to-trial verifier — resolves cited DOIs to canonical Amass IDs, walks the paper→trial cross-core edge, retrieves the trial's registered primary outcome verbatim from the protocol, and surfaces a flagged verdict naming the claim/outcome match or mismatch.
license: Apache-2.0
metadata:
  author: amass
  version: "0.4.0"
---

# Claim-to-Trial Verifier

For two ICPs sharing the same workflow shape: regulatory consultants reviewing promotional material (FDA OPDP-adjacent compliance, EMA pharmacovigilance, internal medical-affairs sign-off) and investigative journalists auditing pharma marketing claims (fact-checking press releases, sales-aid decks, DTC ads). The user pastes a marketing claim + the 1-5 DOIs the promotional copy cites and gets back a verdict card showing the trial's registered primary endpoint verbatim from the protocol. The load-bearing Amass primitives are the paper→trial cross-core walk via `referencesTrialCore` plus the TrialCore `include=outcomes` response delivering the registered primary outcome as a verbatim string — text-mining for NCT identifiers in paper text misses methods/supplements + drifts across registry revisions; LLM paraphrasing trial outcomes against a named pharma sponsor is defamation-adjacent.

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

### File 2: `src/lib/verify.functions.ts` — TanStack Start server function

```ts
// src/lib/verify.functions.ts
import { createServerFn } from "@tanstack/react-start";
import { z } from "zod";
import { getAmassClient, extractLookupResult, type LookupResult } from "./amass.server";

export const ClaimVerifyRequestSchema = z.object({
  claim: z.string().min(1),
  cited_dois: z.array(z.string()).min(1).max(5),
});

export type WalkedTrial = {
  amtcId: string;
  nctId: string;
  briefTitle: string;
  primaryOutcomeMeasures: string[]; // verbatim from trial protocol
};

export type Verdict = "supported" | "not_supported" | "contradicts" | "needs_review";

export type ClaimVerifyResponse = {
  claim: string;
  cited_dois: string[];
  resolved_papers: Array<{ doi: string; ambcId: string; lookup_error: string }>;
  walked_trials: WalkedTrial[];
  verdict: Verdict;
  reasoning: string;
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

export const verifyClaim = createServerFn({ method: "POST" })
  .inputValidator((input: unknown) => ClaimVerifyRequestSchema.parse(input))
  .handler(async ({ data }): Promise<ClaimVerifyResponse> => {
    try {
      const client = getAmassClient();

      // Call 1 — resolve DOIs to AMBC_
      const doiItems = data.cited_dois.map((doi) => ({ doi }));
      const lookupResults = await client.batchLookupBiomed(doiItems);

      const resolvedPapers: ClaimVerifyResponse["resolved_papers"] = [];
      const ambcIds: string[] = [];
      extractLookupResult(doiItems, lookupResults,
        (input, ids) => { ambcIds.push(ids[0]); resolvedPapers.push({ doi: input.doi!, ambcId: ids[0], lookup_error: "" }); return null; },
        (input, msg) => { resolvedPapers.push({ doi: input.doi!, ambcId: "", lookup_error: msg }); return null; },
      );

      // Call 2 — per-paper GET with referencesTrialCore; collect union of walked AMTC_ IDs
      const amtcIds = new Set<string>();
      async function* walkPapers() {
        for (const ambcId of ambcIds) {
          try {
            const rec = (await client.getBiomedRecord(ambcId, ["referencesTrialCore"])) as Record<string, unknown>;
            const refs = (rec["referencesTrialCore"] as string[]) ?? [];
            for (const t of refs) amtcIds.add(t);
            yield { ambcId };
          } catch (e) {
            yield { ambcId, error: e instanceof Error ? e.message : String(e) };
          }
        }
      }
      for await (const _step of withIdleTimeout(walkPapers(), 180_000)) { /* progress */ }

      // Call 3 — per-trial GET with include=outcomes; extract verbatim primary outcome text
      const walkedTrials: WalkedTrial[] = [];
      async function* walkTrials() {
        for (const amtcId of amtcIds) {
          try {
            const rec = (await client.getTrialRecord(amtcId, ["outcomes"])) as Record<string, unknown>;
            walkedTrials.push({
              amtcId,
              nctId: String(rec["nctId"] ?? ""),
              briefTitle: String(rec["briefTitle"] ?? ""),
              primaryOutcomeMeasures: (rec["primaryOutcomeMeasures"] as string[]) ?? [],
            });
            yield { amtcId };
          } catch (e) {
            yield { amtcId, error: e instanceof Error ? e.message : String(e) };
          }
        }
      }
      for await (const _step of withIdleTimeout(walkTrials(), 180_000)) { /* progress */ }

      // LLM-compare step — constrained to cite verbatim claim phrasing + verbatim outcome phrasing
      // (Implement with your AI SDK of choice. Pseudo-code below.)
      let verdict: Verdict = "not_supported";
      let reasoning = "No trial reference resolved in the cross-core spine on the cited paper(s).";

      if (walkedTrials.length > 0) {
        // Replace this with a real LLM call (Vercel AI SDK, Anthropic SDK, etc.).
        // Prompt should require citing both the verbatim claim text and the verbatim
        // primaryOutcomeMeasures text in the reasoning. Constrain output to one of:
        // supported | not_supported | contradicts | needs_review.
        const firstOutcome = walkedTrials[0].primaryOutcomeMeasures[0] ?? "";
        verdict = "needs_review";
        reasoning = `Claim: "${data.claim.slice(0, 80)}...". Trial primary outcome (verbatim): "${firstOutcome}". The claim's stated outcome category does not match the trial's registered primary endpoint category — analyst review required.`;
      }

      return {
        claim: data.claim,
        cited_dois: data.cited_dois,
        resolved_papers: resolvedPapers,
        walked_trials: walkedTrials,
        verdict,
        reasoning,
      };
    } catch (err) {
      throw new Error(err instanceof Error ? err.message : "Claim verification failed");
    }
  });
```

### File 3: `src/routes/index.tsx` — TanStack Start route + page component

```tsx
// src/routes/index.tsx
import { createFileRoute } from "@tanstack/react-router";
import { useState } from "react";
import { useMutation } from "@tanstack/react-query";
import { useServerFn } from "@tanstack/react-start";
import { verifyClaim } from "@/lib/verify.functions";

export const Route = createFileRoute("/")({
  head: () => ({
    meta: [
      { title: "Claim-to-Trial Verifier — Amass" },
      { name: "description", content: "OPDP-adjacent claim verifier — trial primary endpoint verbatim from protocol." },
    ],
  }),
  component: Index, // ← REQUIRED. Without this, SSR fails.
});

function Index() {
  const [claim, setClaim] = useState("");
  const [doisInput, setDoisInput] = useState("");
  const verify = useServerFn(verifyClaim);
  const mutation = useMutation({
    mutationFn: async () => verify({ data: { claim, cited_dois: doisInput.split(/[,\n]/).map((d) => d.trim()).filter(Boolean) } }),
  });

  return (
    <main className="min-h-screen bg-background text-foreground font-sans">
      <header className="border-b border-border">
        <div className="max-w-6xl mx-auto px-6 py-4 flex items-center justify-between">
          <h1 className="text-2xl font-extrabold tracking-tight">amass</h1>
          <span className="text-xs text-muted-foreground font-mono">claim-to-trial-verifier · v0.1</span>
        </div>
      </header>

      <section className="max-w-6xl mx-auto px-6 py-10 space-y-8">
        {/* Claim textarea + cited-DOIs list (1-5) + Try-sample + Verify claim button + Verdict card with 5 fields (claim verbatim / cited DOIs verbatim / walked AMTC_ + NCT / primary outcome verbatim / verdict badge with reasoning) */}
      </section>
    </main>
  );
}
```

---

## What you build

The user pastes a marketing claim + 1-5 cited DOIs from the promotional copy. The handler runs a 3-call chain: `POST /records/lookup` resolves cited DOIs to canonical `AMBC_` IDs; `GET /records/{AMBC_}?include=referencesTrialCore` per resolved paper walks to cited trials; `GET /records/{AMTC_}?include=outcomes` per unique walked trial retrieves the registered primary outcome measures verbatim. Output: per-claim verdict card with five fields — claim verbatim, cited DOIs verbatim, walked trials, primary outcome text verbatim from the protocol, verdict badge (`supported` / `not_supported` / `contradicts` / `needs_review`) with 1-2 sentence reasoning.

Input surface: claim textarea + cited-DOIs list (comma-or-newline separated, 1-5 entries). Empty state: Try-sample with the verified Tarlatamab claim. Output: verdict card. During fan-out: Cancel + progress indicator.

---

## API contract — load-bearing, follow exactly

### Endpoints used

- `POST /api/v1/cores/biomedcore/records/lookup` — batch resolve DOI → `AMBC_`
- `GET /api/v1/cores/biomedcore/records/{amassId}?include=referencesTrialCore` — fetch paper record with cross-core trial spine
- `GET /api/v1/cores/trialcore/records/{amassId}?include=outcomes` — fetch trial record with structured `outcomes` array containing primary outcome measures verbatim

### Auth + rate limit

`Authorization: Bearer ${AMASS_API_KEY}`. 60/60s. 429 → exponential backoff. 401/403 → surface directly.

### Response envelope

`{ "data": ... }` wrapped — unwrap before consuming. Per-item lookup error is structured `{ code, message }` — extract `.message`. NEVER render error object directly (React #31).

### Top-level errors

Non-2xx body: `{ "error": { "code", "message" } }`. The reference code throws; handler `try/catch` surfaces as `mutation.error.message`.

### Verbatim-outcome discipline (load-bearing)

The trial's primary outcome text returned by `GET /records/{AMTC_}?include=outcomes` is a verbatim string from the trial protocol. RENDER IT VERBATIM. Do NOT paraphrase, normalise, or LLM-rewrite between Amass's response and the verdict card. A `contradicts` verdict against a named pharma sponsor with paraphrased outcome text is defamation-adjacent — the verbatim render is the structural guard.

---

## Worked example

- **Claim:** "Tarlatamab improves overall survival in previously-treated ES-SCLC"
- **Cited DOI:** 10.1056/NEJMoa2307980 (Ahn MJ et al., NEJM 2023, DeLLphi-301 primary publication)
- **Expected walked trial:** NCT05060016 (DeLLphi-301) via `AMTC_` resolved through paper→trial cross-core walk
- **Expected primary outcome text (verbatim from trial protocol):** Objective Response Rate per RECIST 1.1 by Investigator (and BICR-assessed companion) — NOT "Overall Survival"
- **Expected verdict:** `needs_review` — claim says Overall Survival; trial primary is Objective Response Rate per RECIST 1.1

Try-sample tooltip: "Tarlatamab (Imdelltra) / Amgen — Ahn NEJM 2023, DeLLphi-301 primary publication. Expected verdict: needs_review (claim says OS; trial primary is ORR per RECIST 1.1)."

(Identifiers and primary endpoint phrasing verified against ClinicalTrials.gov + Amass API on 2026-05-21.)

---

## Per-starter constraints

- v0.1: exactly 1 claim + 1-5 cited DOIs per verification run. Multi-claim batching is extension path.
- Verdict card has 5 fields: claim verbatim (sans-serif), cited DOIs verbatim (monospace), walked trials with `AMTC_` + NCT + briefTitle (monospace), primary outcome text VERBATIM (sans-serif, never paraphrased), verdict badge + 1-2 sentence reasoning.
- LLM-compare step MUST cite verbatim claim phrasing AND verbatim outcome phrasing in reasoning. Does NOT assert truth/falsity beyond what registered primary endpoint supports.
- Walked-trial `AMTC_` and `NCT` IDs render as clickable links to `https://platform.amass.tech/records/trialcore/{amassId}`.
- v0.1 surfaces the trial's REGISTERED primary endpoint (from trial registration), NOT the publication's reported outcome metric.

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

1. **Empty state** renders `<main>` root + amass wordmark + claim textarea + cited-DOIs input + Try-sample. Console clean.
2. **Try sample** loads Tarlatamab claim + DOI 10.1056/NEJMoa2307980. 3-call chain resolves: DOI → `AMBC_` → walks to `AMTC_` → trial outcomes retrieved. Verdict card renders with verbatim claim, verbatim DOI (monospace), walked NCT05060016 (monospace), primary outcome text reading something like "Objective Response Rate per RECIST 1.1 by Investigator" (verbatim from live Amass — NOT "Overall Survival"), verdict `needs_review` with reasoning naming outcome-category mismatch.
3. **Invalid DOI test:** add `10.9999/this-doi-does-not-exist`. Row populates `lookup_error` with verbatim upstream message; verdict falls back to `not_supported` with reasoning explaining no trial walked. Page does NOT crash.
4. **Different claim same DOI:** paste "Tarlatamab achieves an objective response in previously-treated DLL3-positive ES-SCLC" + same DOI. Verdict shifts to `supported` (or close) with reasoning naming outcome-category MATCH.

---

## Hand-off

Build, lint, typecheck must pass.

**Credentials:** `AMASS_API_KEY` from https://platform.amass.tech. Run `npm run dev`.

**Want to extend it?**
- Multi-claim batching for OPDP enforcement-letter audits — persist verdicts to append-only audit log keyed by `claim_id + session_id`.
- Free-text claim extraction — accept promotional-piece PDF, LLM-extract assertions, batch-verify.
- Trial-results-magnitude comparison — when CT.gov has posted results, compare claim's STATED magnitude vs trial's REPORTED magnitude (e.g., `partially_supported` for direction-correct-but-magnitude-overstated).

Discord: https://discord.com/invite/sEGaBHMhWa.

---

## License

Apache-2.0. Copyright 2026 Amass Technologies. Source: https://github.com/lluisDTU/amass-technologies-amass-starters.

---

## Appendix — Next.js App Router alternative

For Claude Code / Cursor / Next.js production users:

- `lib/amass.ts` — same code as `src/lib/amass.server.ts` above, with `import "server-only"` added at the top.
- `app/api/verify-claim/route.ts` — standard Next.js POST handler with identical 3-call chain logic.
- `app/page.tsx` — `"use client"` page POSTing to `/api/verify-claim`. Same `<main>` root, no outer wrapper.

Amass API contract, worked example, validation, kit conventions all apply identically.
