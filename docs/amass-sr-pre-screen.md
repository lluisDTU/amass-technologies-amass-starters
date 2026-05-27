# SR Pre-Screen Credibility Filter

> A PRISMA pre-screen credibility-filter for systematic reviews — drops a ~5,000-PMID search dump to a ~100-500-paper screening set in one batch run, with auditable trust-filter thresholds the SR team can defend in the methods section.

## Who it's for

Systematic-review researchers running PRISMA 2020 — Cochrane review groups, SR-as-a-service consultancies, academic teams with a thesis-scale review on their hands. The user has the upstream PubMed / Embase / CENTRAL / Scopus exports (the canonical 5,000-PMID dump from a sensitive search) and faces a week of title/abstract screening to cut it down to the screening set.

## The job it does

The SR researcher pastes the curated PMID dump from the upstream PRISMA search step into a textarea, sets the credibility thresholds (`min_jufo`, `allow_retracted`, `min_citation_count`), and clicks **Pre-screen**. The tool resolves each PMID to a canonical `AMBC_` Amass ID via batch lookup, fans out per-paper GETs to retrieve `isRetracted` + `journalQualityJufo` + `citationCount`, post-filters client-side against the configured thresholds, and emits a Rayyan/Covidence-importable RIS file plus an audit-trail CSV.

Result: the 5,000-PMID candidate set drops to a ~100-500-paper screening set before title/abstract screening begins. The audit-CSV repeats the threshold values on every row so a downstream reviewer can reproduce the inclusion/exclusion verdicts without needing to remember what the screener set.

The worked example binds to an illustrative SR scope ("GLP-1 receptor agonists in obesity") with ~10 representative PMIDs.

## Why it's built on Amass

**JuFo journal-quality tier in the same response.** JuFo is the Finnish publication-forum quality rating (tier 1-3) used by SR teams as a credibility proxy. Amass exposes it as a default field on every BiomedCore record — no separate JuFo-table lookup, no per-paper journal-name normalisation, no "is this Lancet or The Lancet?" matching headache.

**Retraction flag on every record.** `isRetracted` comes back in the same call. A retracted paper in the upstream search dump gets filtered out at pre-screen time — not at full-text review three months later, when the SR has already invested screening hours on it.

**Per-item-error semantics on batch lookup.** A bad PMID in the 5,000-row dump (typo, deprecated identifier) surfaces inline with a verbatim upstream error message on that one row — without crashing the batch. The audit-CSV's `lookup_error` column carries the verbatim string per Hard Rule #5 — auditable for the PRISMA methods section.

**Free credibility filters that elsewhere require paid APIs.** JuFo (journal quality) + retraction + citation count, all on the Amass free tier — no Web of Science, Scopus, or Crossref Citation API subscriptions required. SR-as-a-service consultancies running on academic budgets can ship this without procuring three separate licensed APIs.

## What you get out of the box

- One AI-builder prompt that scaffolds the full TypeScript + React app
- PMID textarea input with the GLP-1 worked example pre-loaded on the **Try sample** button
- Configurable threshold UI: `min_jufo` (1-3), `allow_retracted` (toggle), `min_citation_count` (numeric)
- Batched per-paper GET with per-item-error preservation
- RIS export ready for Rayyan / Covidence import
- Audit-trail CSV with threshold values replayed on every row
- Cancel-button + idle-timeout discipline on the fan-out

## Where to take it next

Add cohort persistence (save pre-screen runs against a project ID), wire to a PRISMA-flow-diagram generator that consumes the audit CSV, or extend the thresholds with field-specific JuFo overrides (different floor for clinical-trial papers vs basic-science papers).

## Build it

Paste this into Lovable (or Claude Code / Cursor / any AI builder):

> Build a PRISMA pre-screen credibility-filter for systematic reviews with the Amass API. Fetch your build skill: `https://raw.githubusercontent.com/lluisDTU/amass-technologies-amass-starters/main/amass-sr-pre-screen/SKILL.md` — credentials come from the Amass platform console at https://platform.amass.tech.

Sign up for a free API key at [platform.amass.tech](https://platform.amass.tech).

---

[← Back to all starters](../README.md)
