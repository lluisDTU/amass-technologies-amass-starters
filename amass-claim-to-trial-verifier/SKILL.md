---
name: amass-claim-to-trial-verifier
description: Use when building a marketing-claim-to-trial verifier — takes a marketing claim about a drug + cited DOIs, resolves to canonical Amass IDs, walks paper→trial via referencesTrialCore, retrieves trial primary outcomes, and surfaces whether the claim is supported, unsupported, or contradicts the trial's actual outcomes.
license: Apache-2.0
metadata:
  author: amass
  version: "0.1.2"
---

# Claim-to-Trial Verifier (Amass Cross-core)

Build a working marketing-claim-to-trial verifier on the Amass API. The user is a regulatory consultant reviewing promotional material (FDA OPDP-adjacent) or an investigative journalist auditing pharma marketing claims, who just got Amass API credentials and wants a TypeScript claim-verification tool running locally in under five minutes. Lovable typically scaffolds TanStack Start (TanStack Router + `createServerFn` server-functions); Next.js (App Router with `app/api/<endpoint>/route.ts`) is the alternative path. Both work end-to-end against the same `lib/amass.ts` client pattern — the Amass primitives are stack-agnostic.

The consultant pastes a marketing claim about a drug plus the 1-5 DOIs the promotional copy cites; the tool resolves each cited DOI to a canonical `AMBC_` Amass ID, walks the paper→trial cross-core edge per cited paper (`include=referencesTrialCore`), retrieves the cited trial's registered primary outcome measures (`include=outcomes`), and surfaces a verdict card per claim — the **trial's actual primary endpoint rendered verbatim from the registered protocol** next to the claim text, with a flagged verdict (`supported` / `not_supported` / `contradicts` / `needs_review`). The verifier never paraphrases trial outcomes; it surfaces them verbatim so the analyst makes the final call. The worked example binds the verified Tarlatamab (Imdelltra) / Amgen identifier set: claim "Tarlatamab improves overall survival in previously-treated ES-SCLC" + DOI 10.1056/NEJMoa2307980 (Ahn MJ et al. NEJM 2023, DeLLphi-301 primary publication) → resolves to NCT05060016 → trial primary endpoint surfaces as Objective Response Rate (ORR) per RECIST 1.1, NOT Overall Survival → expected verdict `needs_review`. The identifiers (DOI, NCT, primary endpoint phrasing) were verified against ClinicalTrials.gov and the Amass API on 2026-05-21 before shipping.

---

## Stack — flexible at the framework layer, principle-pinned at the API layer

**TypeScript + React + a server-function-capable framework.** Lovable defaults to **TanStack Start** (`@tanstack/react-router` + `@tanstack/react-start`'s `createServerFn` for server-side handlers). **Next.js (App Router)** with `app/api/<endpoint>/route.ts` POST handlers is the equivalent alternative path. Both work end-to-end against the same `lib/amass.ts` client; pick whichever your AI builder produces by default. The Amass primitives (canonical-ID resolution + per-item-error semantics + the paper→trial cross-core walk + verbatim-outcome render) are stack-agnostic — the framework only wraps them. The reference snippets below happen to use the Next.js shape (`route.ts` + `POST(req)`); the load-bearing constraints (server-only client, per-item-error handling, verbatim-outcome discipline) transfer cleanly to TanStack Start.

These package versions are load-bearing — do not upgrade without testing:

```json
"@amass/sdk":             "{{amass-sdk-version}}",
"@ai-sdk/react":          "{{ai-sdk-react-version}}",
"ai":                     "{{ai-version}}",
"react":                  "{{react-version}}",
"react-dom":              "{{react-dom-version}}",
"react-markdown":         "{{react-markdown-version}}",
"zod":                    "{{zod-version}}",
"lucide-react":           "{{lucide-react-version}}",
"server-only":            "{{server-only-version}}"
```

Add the framework packages on top of the above — one set OR the other, not both: **Next.js path** adds `next`; **TanStack Start path** adds `@tanstack/react-router`, `@tanstack/react-start`, `vite`, `@vitejs/plugin-react`.

Pin as **exact versions** (no `^`). `@amass/sdk` is pinned exactly because the cross-core walk semantics (the paper-side `referencesTrialCore` array shape) and the TrialCore `include=outcomes` structured response shape are load-bearing for the verifier's verbatim-outcome rendering — any SDK version that paraphrases, normalises, or restructures the per-trial primary outcome text breaks the verbatim-outcome discipline (Hard Rule #12 below). `ai` / `@ai-sdk/react` are co-pinned for kit-wide consistency with the other 5 prototypes in the kit; v0.1 returns a consolidated JSON payload (non-streaming) so that pinning is not load-bearing for this prototype, but kept aligned so future streaming-progress work re-uses the kit-wide stack.

**Resolve `{{version}}` placeholders before `npm install`** — per Hard Rule #8 (ground-truth discipline), read the installed `node_modules/<pkg>/package.json` for actual versions. A literal `{{amass-sdk-version}}` in `package.json` triggers `npm ERR! EINVALIDTAGNAME` and aborts the install.

After scaffolding, run `npm install`. If any package fails to resolve, check `node_modules/<pkg>/package.json` for the latest stable version and update accordingly.

**Three notes for AI builders that may produce either path.** (1) The route file location differs: TanStack Start uses `src/lib/<name>.functions.ts` with `createServerFn({ method: "POST" }).inputValidator(...).handler(...)`; Next.js App Router uses `app/api/<endpoint>/route.ts` with `export async function POST(req: Request)`. The reference snippet below is shown in Next.js shape — if your builder produces TanStack Start, translate the `POST(req)` handler to `createServerFn(...).handler(async ({ data }) => ...)` and the `req.json()` body parse to `.inputValidator((input) => ClaimVerifyRequestSchema.parse(input))`. (2) The page file location differs: TanStack Router uses `src/routes/index.tsx` with `createFileRoute("/")`; Next.js App Router uses `app/page.tsx`. Either way, the page calls into the server function / route handler the same way at the network level. (3) **The outer `try/catch` wrapping the handler body MUST be preserved when translating to TanStack Start.** The Next.js reference snippet wraps the verification logic in `try { ... } catch (err) { return Response.json({error: ...}, {status: 500}) }`. For TanStack Start, wrap the `.handler()` body in `try { ... } catch (err) { throw new Error(err instanceof Error ? err.message : "Claim verification failed"); }` — the throw surfaces as `mutation.error.message` on the client (TanStack Query's `useMutation` `isError` path) and renders in the UI's inline error region. Without this wrap, an uncaught throw from any API call propagates to TanStack Router's route-level error boundary and crashes the whole page instead of surfacing as a mutation error. Load-bearing for the kit's "per-item error renders inline without crashing the surface" claim.

---

## What the app should do

The primary analyst-visible payoff is the **per-claim verdict card** — one card per submitted claim, surfacing four things together: the claim text verbatim, the cited DOIs verbatim, the walked trial(s) with their **registered primary outcome text rendered verbatim from the trial protocol**, and the verdict (`supported` / `not_supported` / `contradicts` / `needs_review`) with a 1-2 sentence reasoning. The load-bearing Amass primitive is the paper→trial cross-core walk via `referencesTrialCore` chained into TrialCore `include=outcomes` — a 3-call chain (DOI→`AMBC_` lookup, BiomedCore GET with `referencesTrialCore`, TrialCore GET with `outcomes`) that no free-tier API exposes as a structured walk. Text-mining the paper for NCT identifiers is the workaround the verifier forecloses — that approach misses NCTs that appear only in methods or supplements (recall gap) AND drifts across NCT registry revisions (citation-chain drift).

The input surface is a **claim textarea + cited-DOI list** — a textarea accepting a free-text claim ("Tarlatamab improves overall survival in previously-treated ES-SCLC") plus a comma-or-newline-separated list of 1-5 cited DOIs. The empty-state "Try sample" loads the verified Tarlatamab worked example verbatim — claim `"Tarlatamab improves overall survival in previously-treated ES-SCLC"` + cited DOI `10.1056/NEJMoa2307980`. The tooltip on the Try-sample button reads "Tarlatamab (Imdelltra) / Amgen — Ahn MJ et al. NEJM 2023, DeLLphi-301 primary publication. Expected verdict: needs_review (DeLLphi-301's primary endpoint is ORR per RECIST 1.1, not OS)."

The verification run is a 3-call fan-out — Call 1 (`batchLookupBiomed` on the cited DOIs) + Call 2 per resolved `AMBC_` (`getBiomedRecord` with `include=referencesTrialCore`) + Call 3 per unique walked `AMTC_` (`getTrialRecord` with `include=outcomes`). Typical per-claim wall-time is 5-15 seconds for 1-5 DOIs (well below the rate-limit ceiling); the route wraps the per-paper + per-trial fan-out generator in `withIdleTimeout(gen, 180_000)` per Hard Rule #7 and the client renders an analyst-visible **Cancel** button next to the primary "Verify claim" action while the fan-out is in flight. A progress indicator increments per resolved DOI then per walked trial ("resolving DOI 2 of 3", then "walking trial 1 of 2"); a stalled Amass call past the 180s idle threshold surfaces a "stream stalled — an Amass call likely hung" error without locking the UI indefinitely.

The final state is a **per-claim verdict card** rendered in-page with five fields visible: (a) the claim text verbatim in Inter; (b) the cited DOIs in IBM Plex Mono; (c) the walked trial cards — `AMTC_` ID + `NCT` ID + brief title + **primary outcome text VERBATIM from `outcomeMeasures` / structured `outcomes` response**, never paraphrased; (d) the verdict badge (color-coded — green for supported, amber for needs_review, red for contradicts, grey for not_supported); (e) the 1-2 sentence reasoning naming the specific outcome-category match-or-mismatch. Each walked trial card's `AMTC_` ID and `NCT` ID render as **clickable links to the Amass platform's trial-detail surface** (link shape `https://platform.amass.tech/records/trialcore/{amassId}`) so the analyst can open and inspect the registered trial record directly behind the verbatim outcome text — the verdict card's value depends on the analyst being able to reach the trial record the verdict rests on. The v0.1 returns the assembled verdict in a single consolidated JSON response — there is no multi-claim batching, no persistence layer, and no v1.0 trial-results-magnitude comparison (those land under "Want to extend it?" in Hand-off). Render PMIDs / DOIs / NCTs / `AMBC_` / `AMTC_` IDs in IBM Plex Mono; prose in Inter (Brand reference). Use Lucide React for icons. Lovable produces the UI shape from this spec — variability across builders is fine; the load-bearing constraint is that the four paragraphs above demonstrate the paper→trial cross-core walk, the verbatim-outcome rendering, the verdict-with-reasoning shape, and the worked-example anchor.

---

## Hard rules

The 8 Amass-universal rules below are inherited verbatim across all 6 prototype SKILL.mds. Per-prototype rules 9-13 cover the load-bearing UX / error / discipline surfaces specific to the claim-verifier workflow.

1. **Credentials never enter committed code.** Generate `.env` and `.gitignore` it. Only ONE secret is asked of the user at hand-off:
   ```env
   AMASS_API_KEY=your_amass_api_key_here
   ```
   Per Amass API conventions: the key reads from `AMASS_API_KEY`. The base URL is **hardcoded** to `https://api.amass.tech` in `lib/amass.ts` (single production base; no per-user variance). To point at staging or a local proxy for development, edit the `AMASS_API_BASE_URL` constant in `lib/amass.ts` directly — do NOT re-introduce an env var for this (adding `process.env.AMASS_API_BASE_URL` to `lib/amass.ts` causes Lovable's secrets-UX to prompt the user for a value they don't have, which is unnecessary friction). Never hardcode the API key, never log it, never commit it.

2. **Server-only `AmassClient`.** Lives in `lib/amass.ts` with `import "server-only"` at the top. Never imported from a `"use client"` component. The browser only ever speaks to `/api/<endpoint>`. The Bearer token must never reach the browser bundle; the `server-only` package enforces this at build time.

3. **Singleton/cached `AmassClient`.** Created once on server boot via `getAmassClient()` factory, cached via `clientPromise ??=` pattern. Per-request instantiation is wasted work; the Bearer header is computed once at module-import time and reused.

4. **`Authorization: Bearer ${AMASS_API_KEY}` on every Amass request.** On `401` / `403`, surface the credential error directly — retry won't fix it. On `429`, read `Retry-After` and back off exponentially (per the `lib/amass.ts` snippet below). Rate limit is **60 requests / 60 seconds, per user+org** — the per-call retry helper IS the rate-limit defence for fan-out workflows.

5. **Read from the `data` key; handle errors at top-level AND per-item.** Two distinct error shapes: top-level (any non-2xx) — `{"error": {"status": ..., "code": ..., "message": ...}}` — caught by the route's outer `try/catch`. Per-item (`POST /records/lookup` returns HTTP 200 with body `{"data": [...]}` — the universal envelope applies here too) — each array element echoes the input alongside the result: `{"input": {...}, "amassIds": ["AMBC_..."]}` for success, or `{"input": {...}, "error": {"code": "NOT_FOUND", "message": "Identifier not found"}}` for failure (error is a structured object with `code` + `message` fields, NOT a bare string — empirically verified via direct curl on 2026-05-26). The `lib/amass.ts` `batchLookupBiomed` / `batchLookupTrial` methods unwrap the `data` envelope; the `mapLookupResult` helper extracts `error.message` to a string for caller-side rendering, preserving positional index alignment with the input items. **Never assume HTTP 200 means every item succeeded. Never render `error` as a React child directly — extract `.message` first or React #31 crashes the surface.**

6. **Lookup endpoints translate PMID/DOI/NCT → canonical `AMBC_`/`AMTC_` before fetching by ID.** Raw HTTP `GET /records/{id}` accepts **only** canonical Amass IDs — passing a DOI directly returns 404. The workflow is always: `POST /records/lookup` with `[{doi: "..."}]` → read `amassIds[0]` (handling per-item-error per Rule #5) → `GET /records/{amassId}`. The `lib/amass.ts` `resolvePmid` / `resolveDoi` / `resolveNct` helpers encode this discipline.

7. **Idle-timeout + analyst-visible cancel button on long-running fan-outs.** Three requirements: (a) the route wraps any fan-out async generator in `withIdleTimeout(gen, 180_000)` — if no chunk arrives for 180s, the wrapper throws; (b) the client renders an analyst-visible **Cancel** button next to the primary action while a fan-out is in flight; (c) the route wraps body parse + client setup in an outer `try/catch` returning `Response.json({error}, {status: 500})`. Without all three, a single stalled Amass call locks the UI indefinitely.

8. **Ground-truth discipline.** Every URL, type, function name, version, and config shape in this SKILL.md must come from a source the agent can read — a fetched doc page, a `.d.ts` in `node_modules/@amass/sdk/`, the installed package's exports, or a verbatim quote from the Amass API documentation. If you can't point to a source, don't write it. Forbidden: fabricating endpoints; guessing field names; `as unknown as ...` or `as any` on `@amass/sdk` values; ignoring `tsc` errors. When the runtime contradicts what's written here, fix the cause.

### Per-prototype hard rules

9. **Single-claim-with-cited-DOIs input discipline (v0.1).** The v0.1 path is exactly 1 claim + 1-5 cited DOIs per verification run — the typical FDA OPDP / journalistic single-promotional-piece review unit. Multi-claim batching across multiple promotional pieces is a v1.0 extension surfaced in Hand-off's "Want to extend it?" follow-on. Rationale: bounded fan-out (~5-15 API calls per claim) sits well below the 60 req / 60 s rate-limit ceiling, and per-claim manual review is the regulatory-consultant workflow shape — batching across claims dilutes the verdict-card-as-single-decision-unit pattern that the analyst's review process needs.

10. **Paper→trial cross-core walk via `referencesTrialCore` (load-bearing edge).** The load-bearing edge is `referencesTrialCore` on the paper-side BiomedCore GET — `AMBC_` (resolved from cited DOI) → `referencesTrialCore` array of `AMTC_` IDs. The v0.1 walks in the paper→trial direction only; the reverse direction (`NCT` → all supporting publications via TrialCore `referencesBiomedCore`) is explicitly out of scope for v0.1. Empty `referencesTrialCore` arrays (paper has no cited trials — review article shape) render as **honest emptiness** — verdict surfaces as `not_supported` with reasoning "no trial reference in cross-core spine on cited paper" — not as a workflow halt. Rationale: no free-tier API exposes paper→trial as a structured field with canonical-ID delivery; text-mining for NCT identifiers in paper text is the workaround that misses NCTs appearing only in methods or supplements (recall gap) AND drifts across NCT registry revisions (citation-chain drift).

11. **TrialCore primary outcome retrieval via `include=outcomes`.** The trial-side `GET /api/v1/cores/trialcore/records/{amassId}?include=outcomes` returns the structured `outcomes` object array with the registered primary outcome measures rendered verbatim from the trial protocol. The default `primaryOutcomeMeasures` field (TrialCore default-fields table) is a string[] of outcome-measure titles — sufficient for the verdict-card render. The verifier surfaces the trial's REGISTERED primary endpoint, not the publication's reported outcome metric — the trial registration is the load-bearing ground truth for an OPDP-style claim audit. Rationale: the verifier's value depends on showing the analyst the TRIAL'S ACTUAL registered outcome (not a publication summary or a marketing paraphrase); the LLM-compare step is bespoke claim-vs-registered-outcome verification, NOT Scite-style citation-context classification on the paper-citation graph.

12. **Verdict discipline — verbatim outcome text + flagged verdict + analyst-as-final-arbiter framing.** The verdict card surfaces FIVE fields together, each with non-negotiable discipline: (a) the **claim text verbatim** as submitted by the user — never paraphrased, never normalised; (b) the **cited DOIs verbatim** as submitted by the user; (c) the **walked `AMTC_`(s) with `nctId` + `briefTitle`** rendered from the live TrialCore record; (d) the **trial primary outcome text VERBATIM from `outcomeMeasures` / structured `outcomes` response** — surfaced as-rendered from the Amass record, with no LLM rewrite, paraphrase, or summarisation between Amass's response and the analyst's screen; (e) the **verdict badge with 1-2 sentence reasoning** naming the specific outcome-category match-or-mismatch. The reasoning text is LLM-authored but constrained — it cites the verbatim claim phrasing AND the verbatim trial outcome phrasing AND names the specific direction-or-category alignment that drove the verdict; it does NOT assert truth or falsity of the claim beyond what the trial's REGISTERED primary endpoint supports. Rationale: the regulatory consultant or journalist needs the TRIAL DATA as the authoritative anchor to make the final call (a `contradicts` verdict on a real promotional piece against a real pharma sponsor is defamation-adjacent — see Hard Rule #13); the LLM-compare provides initial framing and a 1-2 sentence reasoning prompt, NOT the final published verdict.

13. **Worked-example anchoring (HIGH pure-assertion, verified-then-shipped).** This prototype's worked example is a **pure-assertion** workflow: it asserts that a real marketing claim is unsupported by a real trial registration, with no display of underlying status fields to soften the assertion. The "Try sample" claim + DOI in the empty state binds verified Tarlatamab identifiers **verbatim** — claim "Tarlatamab improves overall survival in previously-treated ES-SCLC" + DOI 10.1056/NEJMoa2307980 (Ahn MJ et al. NEJM 2023, DeLLphi-301 primary publication) → walks to NCT05060016 (DeLLphi-301) → primary outcome surfaces verbatim as "Part 1 Only: Objective Response (OR) per Response Evaluation Criteria in Solid Tumors (RECIST) version 1.1 by Investigator" (the trial's other primary outcomes are TEAEs, serum concentrations, and ORR by BICR — none is Overall Survival). Expected verdict on the worked example: `needs_review` (claim's stated outcome category = Overall Survival does not match trial's registered primary endpoint category = Objective Response Rate per RECIST 1.1). The trial primary outcome text in the verdict card populates from the **live Amass record at run time**, never from a hardcoded fixture, never from a paraphrased summary, never from a real-trial-flipped-to-different-outcome-for-illustration — that variant would be defamation-adjacent under the display=claim rule. Rationale: the prototype ships a pure-assertion workflow against named pharma sponsors + named promotional claims; the value-moment is correctly flagging a real claim/outcome mismatch against a registered trial, which requires the registered-trial side to be empirically verified before binding.

---

## Brand reference

Source of truth for the Amass app look-and-feel. Load before writing `globals.css`.

### Logo

Render the Amass wordmark at the **top-left of the header**: the text **`amass`** in Inter weight 800, ~24-32px. Place it left of the page title. No `<img>` tag, no remote logo URL — the text wordmark is the canonical render for the prototype kit.

### Typography

- **Inter** — UI text, labels, headings, body. Weights 400/600/700/800/900.
- **IBM Plex Mono** — numerics, IDs (PMID / DOI / NCT / `AMBC_` / `AMTC_`), codes, monospace inputs. Weights 400/500/700. Never for prose.

Load both from Google Fonts (`https://fonts.googleapis.com/css2?family=Inter:wght@400;600;700;800;900&family=IBM+Plex+Mono:wght@400;500;700&display=swap`).

### CSS tokens

Add to `globals.css`. Dark mode follows the OS via `prefers-color-scheme` — no toggle, no `next-themes`, no class swap.

```css
:root {
  --background: 0 0% 100%;
  --foreground: 0 0% 6%;
  --card: 0 0% 100%;
  --card-foreground: 0 0% 6%;
  --popover: 0 0% 100%;
  --popover-foreground: 0 0% 6%;
  --primary: 0 0% 6%;
  --primary-foreground: 0 0% 100%;
  --secondary: 0 0% 100%;
  --secondary-foreground: 0 0% 6%;
  --accent: 0 0% 98%;
  --accent-foreground: 222 47% 11%;
  --muted: 0 0% 98%;
  --muted-foreground: 0 0% 41%;
  --destructive: 0 84% 60%;
  --destructive-foreground: 0 0% 100%;
  --border: 0 0% 93%;
  --input: 0 0% 90%;
  --ring: 215 16% 47%;
  --radius: 0.5rem;
  --variant-error-bg: 0 100% 97%;
  --variant-error-border: 0 93% 86%;
  --variant-error-text: 0 72% 37%;
  --amass-accent: 215 80% 50%;
  --amass-accent-foreground: 0 0% 100%;
}

@media (prefers-color-scheme: dark) {
  :root {
    --background: 0 0% 6%;
    --foreground: 0 0% 90%;
    --card: 0 0% 10%;
    --card-foreground: 0 0% 90%;
    --popover: 0 0% 10%;
    --popover-foreground: 0 0% 90%;
    --primary: 0 0% 100%;
    --primary-foreground: 0 0% 6%;
    --secondary: 0 0% 16%;
    --secondary-foreground: 0 0% 90%;
    --accent: 0 0% 16%;
    --accent-foreground: 0 0% 90%;
    --muted: 0 0% 16%;
    --muted-foreground: 0 0% 65%;
    --destructive: 0 72% 50%;
    --destructive-foreground: 0 0% 100%;
    --border: 0 0% 25%;
    --input: 0 0% 25%;
    --ring: 0 0% 100%;
    --variant-error-bg: 0 60% 9%;
    --variant-error-border: 0 52% 28%;
    --variant-error-text: 0 68% 82%;
  }
}
```

Reference all colors via `hsl(var(--token))`. No hardcoded hex outside the accent. The accent is an accent only — never a large background.

Use **Lucide React** for icons (Lucide only).

---

## Reference snippets — use these

**`lib/amass.ts`** — singleton `AmassClient` with Bearer-auth, 429 retry/backoff (reads `Retry-After`), per-item-error helper (`mapLookupResult`), and `{data: ...}` envelope unwrap. Per Hard Rules #2, #3, #4, #5, #6, #10, #11.

```ts
import "server-only";

type LookupResult = ReadonlyArray<{
  input?: { pmid?: string; doi?: string; nctId?: string };
  amassIds?: string[];
  error?: { code?: string; message?: string };
}>;

interface AmassClientConfig {
  apiKey: string;
  baseUrl: string;
  maxRetries?: number;
  retryBaseMs?: number;
}

class AmassClient {
  constructor(private readonly cfg: AmassClientConfig) {}

  private async request<T>(method: "GET" | "POST", path: string, body?: unknown): Promise<T> {
    const url = `${this.cfg.baseUrl}${path}`;
    const headers: Record<string, string> = { Authorization: `Bearer ${this.cfg.apiKey}` };
    if (body !== undefined) headers["Content-Type"] = "application/json";
    const maxRetries = this.cfg.maxRetries ?? 3;
    const baseMs = this.cfg.retryBaseMs ?? 1000;

    for (let attempt = 0; attempt <= maxRetries; attempt++) {
      const res = await fetch(url, {
        method, headers,
        body: body === undefined ? undefined : JSON.stringify(body),
      });
      if (res.status === 429 && attempt < maxRetries) {
        const retryAfter = Number(res.headers.get("Retry-After") ?? "1");
        await new Promise((r) => setTimeout(r, Math.max(retryAfter * 1000, baseMs * 2 ** attempt)));
        continue;
      }
      if (!res.ok) {
        const p = await res.json().catch(() => ({})) as { error?: { code?: string; message?: string } };
        throw new Error(`Amass ${method} ${path} → ${res.status} ${p.error?.code ?? "UNKNOWN"}: ${p.error?.message ?? res.statusText}`);
      }
      return res.json() as Promise<T>;
    }
    throw new Error(`Amass ${method} ${path} — retries exhausted`);
  }

  private async unwrapData<T>(p: Promise<{ data: T }>): Promise<T> {
    return (await p).data;
  }

  // Per-item-error helper per capability-map.md:121. Each result is either {amassIds: [...]} or {error: "..."}.
  // Per empirical observation, each result is either {input?, amassIds: [...]} or
  // {input?, error: {code, message}}. Per Hard Rule #5: upstream-generated error.message
  // strings pass through verbatim.
  static mapLookupResult<I, O>(
    inputs: I[],
    results: LookupResult,
    onSuccess: (input: I, amassIds: string[]) => O,
    onError: (input: I, message: string) => O,
  ): O[] {
    return inputs.map((input, i) => {
      const r = results[i];
      return r?.amassIds && r.amassIds.length > 0
        ? onSuccess(input, r.amassIds)
        : onError(input, r?.error?.message ?? "no result");
    });
  }

  // Lookup: PMID/DOI/NCT → canonical AMBC_/AMTC_ (Hard Rule #6).
  // Response is wrapped in {"data": [...]} per the universal envelope rule — unwrap before
  // returning so callers + mapLookupResult see the array directly.
  async batchLookupBiomed(items: ReadonlyArray<{ pmid?: string; doi?: string }>): Promise<LookupResult> {
    return this.unwrapData(this.request<{ data: LookupResult }>("POST", "/api/v1/cores/biomedcore/records/lookup", { items }));
  }
  async batchLookupTrial(items: ReadonlyArray<{ nctId: string }>): Promise<LookupResult> {
    return this.unwrapData(this.request<{ data: LookupResult }>("POST", "/api/v1/cores/trialcore/records/lookup", { items }));
  }
  async resolvePmid(pmid: string): Promise<string | null> {
    return (await this.batchLookupBiomed([{ pmid }]))[0]?.amassIds?.[0] ?? null;
  }
  async resolveDoi(doi: string): Promise<string | null> {
    return (await this.batchLookupBiomed([{ doi }]))[0]?.amassIds?.[0] ?? null;
  }
  async resolveNct(nctId: string): Promise<string | null> {
    return (await this.batchLookupTrial([{ nctId }]))[0]?.amassIds?.[0] ?? null;
  }

  // BiomedCore GET — defaults to include=referencesTrialCore per Hard Rule #10 (load-bearing cross-core edge for
  // the claim-verifier workflow). Caller may override.
  async getBiomedRecord(
    amassId: string,
    includes: ReadonlyArray<"fulltext" | "authorsMetadata" | "meshIds" | "substanceIds" | "referencesTrialCore" | "references" | "citedBy"> = ["referencesTrialCore"],
  ): Promise<unknown /* bind to @amass/sdk BiomedRecord type once installed */> {
    const qs = includes.length > 0 ? `?${includes.map((i) => `include=${i}`).join("&")}` : "";
    return this.unwrapData(this.request<{ data: unknown }>("GET", `/api/v1/cores/biomedcore/records/${amassId}${qs}`));
  }

  // TrialCore GET — defaults to include=outcomes per Hard Rule #11 (load-bearing for primary-endpoint
  // verbatim render in the verdict card). Caller may override.
  async getTrialRecord(
    amassId: string,
    includes: ReadonlyArray<"detailedDescription" | "outcomes" | "referencesBiomedCore"> = ["outcomes"],
  ): Promise<unknown /* bind to @amass/sdk TrialRecord type once installed */> {
    const qs = includes.length > 0 ? `?${includes.map((i) => `include=${i}`).join("&")}` : "";
    return this.unwrapData(this.request<{ data: unknown }>("GET", `/api/v1/cores/trialcore/records/${amassId}${qs}`));
  }
}

let clientPromise: Promise<AmassClient> | null = null;

// Production Amass API base URL. Hardcoded by design — single production base, no per-user variance.
// To point at staging or a local proxy for development, edit this constant directly.
// Do NOT replace this with a process.env lookup — Lovable's secrets-UX will prompt the user for a
// value they don't have, which is unnecessary friction. Per Hard Rule #1.
const AMASS_API_BASE_URL = "https://api.amass.tech";

export function getAmassClient(): Promise<AmassClient> {
  return (clientPromise ??= Promise.resolve(new AmassClient({
    apiKey: process.env.AMASS_API_KEY!,
    baseUrl: AMASS_API_BASE_URL,
  })));
}

export { AmassClient };
```

**`app/api/verify-claim/route.ts`** — Next.js App Router POST handler. Takes a JSON body `{claim: string, cited_dois: string[]}`, runs Call 1 (`batchLookupBiomed` on cited DOIs) → fans out Call 2 (`getBiomedRecord` per resolved `AMBC_`) → de-duplicates the union of walked `AMTC_` IDs → fans out Call 3 (`getTrialRecord` per unique `AMTC_`) wrapped in `withIdleTimeout(gen, 180_000)` per Hard Rule #7 → runs the LLM-compare step to author the verdict + reasoning → returns the assembled verdict card in one consolidated JSON response. Non-streaming for v0.1.

```ts
import { AmassClient, getAmassClient } from "@/lib/amass";

interface ClaimVerifyRequest {
  claim: string;
  cited_dois: string[];
}

interface WalkedTrial {
  amassId: string;
  nctId: string | null;
  briefTitle: string | null;
  primaryOutcomeMeasures: string[];   // verbatim from Amass — Hard Rule #12
  walk_error: string | null;
}

type Verdict = "supported" | "not_supported" | "contradicts" | "needs_review";

interface ClaimVerifyResponse {
  claim: string;
  cited_dois: string[];
  resolved_papers: Array<{ doi: string; amassId: string | null; lookup_error: string | null }>;
  walked_trials: WalkedTrial[];
  verdict: Verdict;
  reasoning: string;
}

async function* withIdleTimeout<T>(gen: AsyncGenerator<T>, maxIdleMs = 180_000): AsyncGenerator<T> {
  while (true) {
    const timeout = new Promise<{ kind: "timeout" }>((r) => setTimeout(() => r({ kind: "timeout" }), maxIdleMs));
    const result = await Promise.race([
      gen.next().then((r) => ({ kind: "next" as const, r })),
      timeout,
    ]);
    if (result.kind === "timeout") {
      throw new Error(`Stream stalled for ${Math.round(maxIdleMs / 1000)}s — an Amass call likely hung.`);
    }
    if (result.r.done) return;
    yield result.r.value;
  }
}

// Per Hard Rule #10 — extract referencesTrialCore array from BiomedCore record verbatim. Empty array
// renders as honest emptiness on the verdict card (no trial reference in cross-core spine).
function extractRefsTrialCore(record: unknown): string[] {
  const r = record as { referencesTrialCore?: string[] };
  return r.referencesTrialCore ?? [];
}

// Per Hard Rule #11 + #12 — surface registered primary outcome verbatim. v0.1 reads the default
// `primaryOutcomeMeasures` string[] field; `include=outcomes` adds the structured `outcomes` object array
// for finer-grained renders in v1.0 extensions.
function extractTrialFields(record: unknown): { nctId: string | null; briefTitle: string | null; primaryOutcomeMeasures: string[] } {
  const r = record as { nctId?: string | null; briefTitle?: string | null; primaryOutcomeMeasures?: string[] };
  return {
    nctId: r.nctId ?? null,
    briefTitle: r.briefTitle ?? null,
    primaryOutcomeMeasures: r.primaryOutcomeMeasures ?? [],
  };
}

// LLM-compare step — authors a 1-2 sentence reasoning citing the verbatim claim phrasing AND the
// verbatim primary outcome phrasing. Per Hard Rule #12: does NOT assert truth/falsity beyond what
// the trial's REGISTERED primary endpoint supports. Replace the stub with your AI SDK call shape.
async function classifyClaim(
  claim: string,
  walked: WalkedTrial[],
): Promise<{ verdict: Verdict; reasoning: string }> {
  // If no trial walks at all → not_supported.
  const trialsWithOutcomes = walked.filter((t) => t.primaryOutcomeMeasures.length > 0);
  if (trialsWithOutcomes.length === 0) {
    return {
      verdict: "not_supported",
      reasoning: "No trial reference resolved in the cross-core spine on the cited paper(s); the claim cannot be anchored to a registered trial primary endpoint via the walked path.",
    };
  }

  // LLM call here. The prompt instruction (paraphrased): given the verbatim claim text and the verbatim
  // primary outcome measure(s) from each walked trial, return one of {supported, not_supported,
  // contradicts, needs_review} + a 1-2 sentence reasoning citing the verbatim phrasings. Do NOT assert
  // truth/falsity beyond what the registered primary endpoint supports.
  //
  // Example prompt shape (wire to your AI SDK of choice — Vercel AI SDK, Anthropic SDK, etc.):
  //
  //   const result = await generateObject({
  //     model: anthropic("claude-sonnet-4-6"),
  //     schema: z.object({ verdict: z.enum([...]), reasoning: z.string().max(400) }),
  //     prompt: buildPrompt(claim, trialsWithOutcomes),
  //   });
  //
  // Returning a deterministic stub here so the scaffolded app compiles and runs end-to-end on the
  // worked example without an LLM API key wired up — Lovable will replace this with a real LLM call
  // during the build pass.
  return {
    verdict: "needs_review",
    reasoning: `Claim references "${claim.slice(0, 60)}${claim.length > 60 ? "..." : ""}". Walked trial primary outcome: "${trialsWithOutcomes[0].primaryOutcomeMeasures[0]}". The claim's stated outcome category does not match the trial's registered primary endpoint category — analyst review required.`,
  };
}

export async function POST(req: Request) {
  try {
    const body = (await req.json()) as ClaimVerifyRequest;
    const client = await getAmassClient();

    if (!body.claim || !Array.isArray(body.cited_dois) || body.cited_dois.length === 0) {
      return Response.json({ error: "Request must include a non-empty claim and at least one cited DOI." }, { status: 400 });
    }
    if (body.cited_dois.length > 5) {
      return Response.json({ error: "v0.1 accepts at most 5 cited DOIs per claim — see Hard Rule #9." }, { status: 400 });
    }

    // Call 1 — batch lookup the cited DOIs (Hard Rule #6).
    const doiItems = body.cited_dois.map((doi) => ({ doi }));
    const doiResults = await client.batchLookupBiomed(doiItems);

    const resolvedPapers: ClaimVerifyResponse["resolved_papers"] = [];
    const amassIdsToWalk: string[] = [];
    AmassClient.mapLookupResult(
      doiItems,
      doiResults,
      ({ doi }, amassIds) => {
        amassIdsToWalk.push(amassIds[0]);
        resolvedPapers.push({ doi, amassId: amassIds[0], lookup_error: null });
        return null;
      },
      ({ doi }, msg) => {
        // Per-item lookup error — verbatim upstream string per Hard Rule #5.
        resolvedPapers.push({ doi, amassId: null, lookup_error: msg });
        return null;
      },
    );

    // Call 2 — per-paper GET with include=referencesTrialCore (Hard Rule #10). Collect the union of
    // walked AMTC_ IDs across the resolved papers; de-duplicate before Call 3.
    const walkedAmtcIds = new Set<string>();
    const paperWalkErrors = new Map<string, string>();
    async function* walkPapers(): AsyncGenerator<{ amassId: string }> {
      for (const amassId of amassIdsToWalk) {
        try {
          const record = await client.getBiomedRecord(amassId, ["referencesTrialCore"]);
          for (const amtc of extractRefsTrialCore(record)) walkedAmtcIds.add(amtc);
          yield { amassId };
        } catch (e) {
          paperWalkErrors.set(amassId, e instanceof Error ? e.message : "paper walk failed");
          yield { amassId };
        }
      }
    }
    for await (const _ of withIdleTimeout(walkPapers(), 180_000)) { /* progress hook — wire to streaming if added in v1.0 */ }

    // Call 3 — per-trial GET with include=outcomes (Hard Rule #11). Fan out across the de-duplicated
    // AMTC_ set; assemble walked-trial records carrying verbatim primaryOutcomeMeasures.
    const walkedTrials: WalkedTrial[] = [];
    async function* walkTrials(): AsyncGenerator<WalkedTrial> {
      for (const amtcId of walkedAmtcIds) {
        try {
          const record = await client.getTrialRecord(amtcId, ["outcomes"]);
          const { nctId, briefTitle, primaryOutcomeMeasures } = extractTrialFields(record);
          yield { amassId: amtcId, nctId, briefTitle, primaryOutcomeMeasures, walk_error: null };
        } catch (e) {
          yield { amassId: amtcId, nctId: null, briefTitle: null, primaryOutcomeMeasures: [], walk_error: e instanceof Error ? e.message : "trial walk failed" };
        }
      }
    }
    for await (const trial of withIdleTimeout(walkTrials(), 180_000)) walkedTrials.push(trial);

    // LLM-compare step → verdict + reasoning (Hard Rule #12).
    const { verdict, reasoning } = await classifyClaim(body.claim, walkedTrials);

    return Response.json<ClaimVerifyResponse>({
      claim: body.claim,
      cited_dois: body.cited_dois,
      resolved_papers: resolvedPapers,
      walked_trials: walkedTrials,
      verdict,
      reasoning,
    });
  } catch (err) {
    return Response.json({ error: err instanceof Error ? err.message : "Claim verification failed" }, { status: 500 });
  }
}
```

The two snippets above are the only `lib/<file>.ts` + `app/api/<route>/route.ts` mandated by this prototype. v0.1 returns the assembled verdict card in-memory in a single consolidated response — there is no persistence module, no multi-claim batching store, no `app/api/batch-verify/route.ts`, no Python CLI sidecar. Render happens client-side from the response payload. If a downstream extension needs persistence (audit history of past verifications, multi-claim batch sessions), the extension SKILL.md includes its own minimal stub per the template's policy — never an `import` from `@/lib/<x>` without a matching snippet.

---

## Required reading

- `https://api.amass.tech` — Amass API root (BiomedCore + TrialCore endpoints — lookup, get-by-ID, include flags including `referencesTrialCore` and `outcomes`).
- `https://clinicaltrials.gov/study/NCT05060016` — DeLLphi-301 registration record; primary outcome verbatim source for the worked example.
- `https://github.com/amass-technologies/public-amass-platform-starter-ts` — **official Amass Platform Starter** (Apache-2.0; Bun + Ink + Vercel AI SDK CLI REPL with BiomedCore + TrialCore tools wired up). Use for tool-definition shape reference + ground-truth env-var conventions (`AMASS_API_KEY` required; `AMASS_API_BASE_URL` overrides default `https://api.amass.tech`); use this kit's `lib/amass.ts` for web app API calls. Stack diverges (CLI vs Next.js+TS) — the starter is the engineer-facing complement.

---

## Hand-off

Build, lint, and typecheck must pass.

**Credentials.** Get your `AMASS_API_KEY` from https://platform.amass.tech and add it to `.env`. Do NOT use `AskUserQuestion` to gate scaffolding on credentials — Lovable's (or any AI builder's) standard env-var prompt covers this at run time. Just scaffold the app with placeholder `.env`, then run `npm run dev` once the key is in place.

Then present the hand-off summary. The four verification steps bind to the Tarlatamab worked-example identifier set (DOI, NCT, primary endpoint phrasing) verified against ClinicalTrials.gov and the Amass API on 2026-05-21:

> **Your Claim-to-Trial Verifier is ready.**
>
> To run it: fill in `.env` and run `npm run dev`.
>
> Verify it works:
> - Click **Try sample** to load the Tarlatamab claim ("Tarlatamab improves overall survival in previously-treated ES-SCLC") plus DOI `10.1056/NEJMoa2307980`; click **Verify claim**. Confirm the 3-call chain resolves: DOI → `AMBC_` (Ahn MJ et al. NEJM 2023) → `referencesTrialCore` → `AMTC_DJIWhoYfJnhExqO0I7e7y17JTSJ` (DeLLphi-301, NCT05060016) → primary outcome surfaces verbatim from Amass as "Part 1 Only: Objective Response (OR) per Response Evaluation Criteria in Solid Tumors (RECIST) version 1.1 by Investigator" (or the BICR-assessed companion outcome — both are registered primary endpoints).
> - Confirm the verdict surfaces as **`needs_review`** with the reasoning naming the claim/outcome category mismatch — claim says Overall Survival; trial's registered primary endpoint is Objective Response Rate per RECIST 1.1. The verdict card renders the claim text in Inter, the DOI in IBM Plex Mono, the walked `NCT05060016` in IBM Plex Mono, and the primary-outcome text verbatim in Inter — no LLM paraphrase between Amass's response and the screen.
> - Run a different claim against the same DOI — paste "Tarlatamab achieves an objective response in previously-treated DLL3-positive ES-SCLC" and click **Verify claim**. Confirm the verdict shifts to **`supported`** (or close approximation) with reasoning naming the outcome-category match (claim says ORR; trial primary is ORR per RECIST 1.1).
> - Paste an intentionally-invalid DOI `10.9999/this-doi-does-not-exist` into the cited-DOIs field; click **Verify claim**; confirm the row renders inline with `lookup_error` populated by the verbatim upstream string per Hard Rule #5 — without crashing the surface, and with the verdict surfacing as `not_supported` because no trial walked from the unresolved paper.
>
> Want to extend it?
> - Add multi-claim batching for OPDP enforcement-letter audits — surface a saved-session UI that runs a list of claim-DOI pairs against a single regulatory-consultant engagement, persisting verdicts to an append-only audit log keyed by `claim_id + session_id` for the engagement's submission file.
> - Add v1.0 free-text claim extraction — accept a full promotional-piece PDF or text block, run an LLM extraction step to surface the list of distinct assertions, then verify each assertion against the cited DOI set as a batched run.
> - Add trial-results-magnitude integration — for trials where CT.gov reports posted results (`hasResults=true`), retrieve the results via TrialCore `include=outcomes` (structured `outcomes` object array) and compare the claim's STATED magnitude (e.g. "30% objective response") against the trial's REPORTED magnitude, surfacing `partially_supported` for direction-correct-but-magnitude-overstated cases.
> - I'm done — just show me the summary
>
> Have questions? Join the Amass Developer Community on Discord: https://discord.com/invite/sEGaBHMhWa

---

## License

Apache-2.0. Copyright 2026 Amass Technologies. Source at https://github.com/lluisDTU/amass-technologies-amass-starters.
