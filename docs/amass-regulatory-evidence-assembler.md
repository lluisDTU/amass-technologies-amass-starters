# Regulatory Evidence Assembler

> An auditable literature-evidence assembler for FDA / EMA regulatory submissions — built for the gap between submission day and approval day, when PMIDs and NCTs in the upstream registries shift underneath you.

## Who it's for

Medical writers, regulatory-affairs analysts, and evidence-synthesis CROs (Arriello, Masuu Global, and similar shops) preparing CTD Module 2.5 / 2.7 narratives or PSUR-style ongoing-evidence audits. The user has a curated PMID / DOI / NCT submission scope already — what they need is a tool that turns it into an audit-CSV their reviewer can defend without re-checking every external identifier.

## The job it does

The operator pastes a curated list of external identifiers (PMID, DOI, NCT) exported from upstream discovery — PubHive, Embase, Scopus — and clicks **Assemble audit**. The tool resolves each identifier to a canonical `AMBC_` or `AMTC_` Amass ID, walks the paper→trial cross-core edge (`include=referencesTrialCore,citedBy`), and emits the audit-CSV row set that becomes Appendix B of the submission package. Each cited paper is anchored to its supporting trial via a canonical `AMTC_` ID that doesn't move when ClinicalTrials.gov rewrites an NCT between submission and approval.

The worked example binds to the Tarlatamab (Imdelltra) / Amgen BLA 761344-dlle scope: Ahn MJ et al. NEJM 2023 (PMID 37861218), Rudin et al. J Hematol Oncol 2023 (PMID 37355629), DeLLphi-301 (NCT05060016), DeLLphi-304 (NCT05740566) — all verified against PubMed + ClinicalTrials.gov + FDA approval letter at draft time.

## Why it's built on Amass

**Canonical-ID stability through registry revisions.** Submission packages live for 6-12 months between submission and approval. PMIDs and NCTs can be revised, merged, or superseded in that window. Amass canonical IDs (`AMBC_` / `AMTC_`) are stable across upstream-registry revisions — the audit-CSV's anchor doesn't drift.

**Paper→trial cross-core edge in one call.** The `referencesTrialCore` array on a BiomedCore record delivers all cited trials in the same response as the paper record. No second HTTP call, no separate text-mining pass for NCT identifiers in methods/supplements, no cross-system identifier-mapping table.

**Per-item-error semantics on batch lookup.** An invalid identifier in the curated scope (an outdated PMID, a typo in a DOI) surfaces as a structured `{ code, message }` error on that one row — without crashing the rest of the batch. Audit-CSV row N+1 still populates correctly.

**Trust signals on every record.** `isRetracted` (default field) and `journalQualityJufo` come back wired into the same response — a retracted paper in the citation chain surfaces immediately in the assembled CSV, not on a separate retraction-DB scrape pass.

## What you get out of the box

- One Lovable / Claude Code / Cursor prompt that scaffolds the full TypeScript + React app
- Batch-lookup + per-paper GET + per-trial GET, all with structured error handling
- Verbatim Tarlatamab DeLLphi-301 worked example wired to the empty-state **Try sample** button
- Audit-CSV export with columns covering external identifier → canonical Amass ID → cited trial → retraction flag → JuFo tier
- A 180-second idle-timeout-with-cancel pattern on the long-running fan-out

## Where to take it next

Add submission-scope persistence (save audit runs against a project ID), wire in RegulatoryCore once it lands for FDA approval-letter cross-referencing, or extend to PSUR cadence with month-over-month diff highlighting.

## Build it

Paste this into Lovable (or Claude Code / Cursor / any AI builder):

> Build an auditable regulatory-evidence assembler with the Amass API. Fetch your build skill: `https://raw.githubusercontent.com/lluisDTU/amass-technologies-amass-starters/main/amass-regulatory-evidence-assembler/SKILL.md` — credentials come from the Amass platform console at https://platform.amass.tech.

Sign up for a free API key at [platform.amass.tech](https://platform.amass.tech).

---

[← Back to all starters](../README.md)
