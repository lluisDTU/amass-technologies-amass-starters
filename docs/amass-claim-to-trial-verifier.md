# Claim-to-Trial Verifier

> A marketing-claim-to-trial verifier for OPDP-adjacent regulatory review and pharma-claim journalism. Surfaces the trial's registered primary endpoint verbatim from the protocol next to the promotional claim — with a flagged verdict naming the specific match or mismatch.

## Who it's for

Two ICPs that share the same workflow shape:

- **Regulatory consultants reviewing promotional material** — FDA Office of Prescription Drug Promotion (OPDP) compliance reviews, EMA pharmacovigilance audits, internal-medical-affairs sign-off on marketing claims.
- **Investigative journalists auditing pharma marketing claims** — fact-checking a press release, a sales-aid deck, or a direct-to-consumer ad against the underlying trial registration.

Both audiences need the same artifact: the trial's actual registered primary endpoint rendered verbatim from the protocol, next to the claim text, with a verdict the analyst can defend.

## The job it does

The consultant pastes a marketing claim about a drug plus the 1-5 DOIs the promotional copy cites, and clicks **Verify claim**. The tool resolves each cited DOI to a canonical `AMBC_` Amass ID, walks the paper→trial cross-core edge per cited paper (`include=referencesTrialCore`), retrieves the cited trial's registered primary outcome measures (`include=outcomes`), and surfaces a verdict card per claim:

- **Claim text verbatim** (Inter)
- **Cited DOIs verbatim** (IBM Plex Mono)
- **Walked trial(s)** — `AMTC_` + NCT + brief title — clickable to the Amass platform's trial-detail surface
- **Primary outcome text VERBATIM from the trial protocol** — never paraphrased, never normalised
- **Verdict badge** — `supported` / `not_supported` / `contradicts` / `needs_review` — with a 1-2 sentence reasoning naming the specific outcome-category match or mismatch.

The worked example binds the verified Tarlatamab (Imdelltra) / Amgen identifier set: claim **"Tarlatamab improves overall survival in previously-treated ES-SCLC"** + DOI **10.1056/NEJMoa2307980** (Ahn MJ et al. NEJM 2023, DeLLphi-301 primary publication) → resolves to NCT05060016 → trial primary endpoint surfaces verbatim as **Objective Response Rate per RECIST 1.1**, NOT Overall Survival → expected verdict `needs_review`. (Identifiers and primary endpoint phrasing verified against ClinicalTrials.gov and the Amass API on 2026-05-21.)

## Why it's built on Amass

**Paper→trial cross-core walk via `referencesTrialCore`.** The load-bearing edge is the paper-side BiomedCore record's `referencesTrialCore` array — DOI → `AMBC_` → walked `AMTC_` IDs in one fetch. No free-tier API exposes paper→trial as a structured field with canonical-ID delivery. Text-mining for NCT identifiers in paper text misses NCTs that appear only in methods or supplements (recall gap) AND drifts across NCT-registry revisions (citation-chain drift).

**TrialCore `include=outcomes` for the registered primary endpoint.** The trial-side GET returns the structured `outcomes` array with primary outcome measures rendered verbatim from the trial protocol. The verifier surfaces the trial's **registered** primary endpoint — not the publication's reported outcome metric, not a marketing paraphrase. The trial registration is the load-bearing ground truth for an OPDP-style claim audit.

**Verbatim-outcome discipline as an API guarantee.** Amass returns the primary outcome measure as a string field — no LLM rewrite, no normalisation, no paraphrase between the trial protocol's wording and the verdict card. A `contradicts` verdict against a named pharma sponsor is defamation-adjacent unless the rendered text is verbatim — the API's verbatim-string delivery makes the verbatim-render discipline possible.

**Per-item-error semantics on the DOI lookup.** An invalid DOI in the promotional-copy cited-DOI list surfaces inline as `lookup_error` on that one row — the rest of the claim verification continues. The `not_supported` verdict on a fully-unresolved paper set says "no trial reference in cross-core spine," not "the tool broke."

## What you get out of the box

- One AI-builder prompt that scaffolds the full TypeScript + React app
- Tarlatamab worked example wired to the empty-state **Try sample** button
- 3-call fan-out — batch DOI lookup → per-paper GET with `referencesTrialCore` → per-trial GET with `outcomes`
- Per-claim verdict card with all five fields verbatim from the live Amass record
- Clickable `AMTC_` chips linking to the Amass platform's trial-detail surface
- LLM-compare step constrained to cite verbatim phrasings (claim + outcome) in the 1-2 sentence reasoning
- Cancel-button + idle-timeout discipline on the fan-out

## Where to take it next

Add multi-claim batching for OPDP enforcement-letter audits (persist verdicts to an append-only audit log keyed by `claim_id + session_id`), add free-text claim extraction from a full promotional-piece PDF (LLM extraction step → batched verification), or extend to trial-results-magnitude integration (when CT.gov has posted results, compare the claim's STATED magnitude against the trial's REPORTED magnitude to flag `partially_supported` cases).

## Build it

Paste this into Lovable (or Claude Code / Cursor / any AI builder):

> Build a marketing-claim-to-trial verifier with the Amass API. Fetch your build skill: `https://raw.githubusercontent.com/lluisDTU/amass-technologies-amass-starters/main/amass-claim-to-trial-verifier/SKILL.md` — credentials come from the Amass platform console at https://platform.amass.tech.

Sign up for a free API key at [platform.amass.tech](https://platform.amass.tech).

---

[← Back to all starters](../README.md)
