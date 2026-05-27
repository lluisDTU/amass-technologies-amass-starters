# Pipeline Monitor CI Agent

> A weekly pipeline-monitoring digest for biotech competitive-intelligence analysts — surfaces new Phase 2/3 trials, new evidence describing them, and retraction-flagged citations, all keyed on canonical sponsor and trial identifiers so week-over-week diffs don't drift.

## Who it's for

Competitive-intelligence analysts at mid-cap immuno-oncology biotechs (or any team tracking a watchlist of pharma sponsors and indication clusters). The user maintains a sponsor watchlist — Amgen, Roche, AbbVie, and friends — and needs a weekly digest that doesn't take half a day to assemble from PubMed + ClinicalTrials.gov + OpenAlex by hand.

## The job it does

The analyst pastes a YAML sponsor watchlist (canonical sponsors + indication-cluster scope + last-check date) and clicks **Run digest**. The app issues a per-sponsor TrialCore search filtered to Phase 2/3 + recent `minStartDate`, walks each surfaced trial's `referencesBiomedCore` cross-core edge to find the papers describing it, resolves per-paper trust signals (`isRetracted`, `journalQualityJufo`, `citationCount`), and emits a Markdown digest grouped by sponsor — rendered inline as a 3-panel weekly dashboard.

The worked example anchors on a **SCLC-DLL3-2026Q2** watchlist (Amgen + Roche + AbbVie + Boehringer Ingelheim + Harpoon Therapeutics) bound to the verified Tarlatamab identifier set: pivotal **DeLLphi-301 = NCT05060016**, confirmatory **DeLLphi-304 = NCT05740566**, primary publication **PMID 37861218** (Ahn MJ et al., NEJM 2023).

## Why it's built on Amass

**Canonical `AMTC_` IDs that survive NCT-registry revisions.** A week-over-week diff keyed on a canonical `AMTC_` doesn't drift when ClinicalTrials.gov rewrites the NCT identifier underneath — the diff stays honest. PMID-keyed or NCT-keyed diffs on PubMed + CT.gov give you false-positive "new trial!" alerts every time an identifier gets revised.

**Trial→paper cross-core edge in one call.** `include=referencesBiomedCore` on a TrialCore record delivers the paper-side context that on PubMed + CT.gov + OpenAlex requires 3-4 calls across separate entity-key systems. One Amass call → all papers describing the trial.

**Sponsor-name canonicalisation at onboarding.** The starter's Step 0 issues a per-sponsor `query` against TrialCore, clusters the returned `sponsorName` strings, and asks the analyst to confirm the canonical mapping. **Genentech ↔ Roche** is the canonical example — caught once at onboarding, stable for every subsequent weekly run.

**Retraction flags wired into the digest.** `isRetracted` is a default field on every paper record — a retracted citation in a new publication describing one of the watched trials surfaces immediately in the right-panel "retraction-flagged citations" pane, not on a separate audit pass.

## What you get out of the box

- One AI-builder prompt that scaffolds the full app + watchlist onboarding flow
- YAML sponsor-watchlist input with the SCLC-DLL3-2026Q2 worked example pre-loaded on the **Try sample** button
- Per-sponsor TrialCore search + per-trial cross-core walk + per-paper trust filter resolution
- 3-panel weekly dashboard rendering inline (new trials | new papers | retraction-flagged citations)
- Downloadable Markdown digest (`digest-<watchlist>-<YYYY-Wnn>.md`)
- Cancel-button + idle-timeout discipline on the ~2-6 minute fan-out

## Where to take it next

Add digest-history append-only persistence (week-over-week diff in the dashboard itself), wire to a Slack channel or email distribution list as an MCP slash-command, or extend the watchlist to include modality-cluster scoping (e.g., "DLL3-targeted bispecifics in SCLC" as a saved indication query).

## Build it

Paste this into Lovable (or Claude Code / Cursor / any AI builder):

> Build a weekly competitive-intelligence pipeline monitor with the Amass API. Fetch your build skill: `https://raw.githubusercontent.com/lluisDTU/amass-technologies-amass-starters/main/amass-pipeline-monitor/SKILL.md` — credentials come from the Amass platform console at https://platform.amass.tech.

Sign up for a free API key at [platform.amass.tech](https://platform.amass.tech).

---

[← Back to all starters](../README.md)
