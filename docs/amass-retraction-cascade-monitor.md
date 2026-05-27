# Retraction Cascade Monitor

> A retraction-cascade tracker that re-audits a stored bibliography for newly retracted citations AND surfaces the downstream cascade — inbound `citedBy`, outbound `references`, and the cross-core trial spine — so SR teams, regulatory affairs, and medical-info teams pull contaminated evidence from their dossiers before downstream harm.

## Who it's for

Three ICPs that share the same workflow shape:

- **Systematic-review teams** re-auditing a published Cochrane-style review's bibliography for citations retracted since publication.
- **Regulatory-affairs analysts** monitoring a submission's citation chain — a citation that goes retracted between submission and approval needs to be caught before the FDA's reviewer does.
- **Medical-affairs teams** monitoring a published medical-information answer for citation contamination — the literature behind a fielded medical-info response can rot quietly over months.

A single deployment serves all three.

## The job it does

The user pastes a curated bibliography PMID list (the caller's stored snapshot at time t0) into a textarea and clicks **Run cascade audit**. The tool resolves each PMID to a canonical `AMBC_` Amass ID, fetches the per-paper record with `include=references,citedBy,referencesTrialCore` (the retraction-flag default field + both-direction citation graph + cross-core trial spine — all in one record fetch), filters to `isRetracted === true`, and surfaces the three-direction cascade as a 3-panel dashboard:

- **Left panel — retracted papers in bibliography.** The `isRetracted=true` subset with journal + retraction-flag + downstream-contamination cardinality.
- **Center panel — inbound `citedBy` cascade.** What downstream papers cite the retracted work and now carry contamination.
- **Right panel — outbound `references` + cross-core `referencesTrialCore` cascade.** What the retracted work itself cited (often revealing the shaky evidence chain) and what trials it describes.

The worked example binds to Wakefield et al. Lancet 1998 (PMID 9500320) — the canonical retracted paper for demonstration, with ~3,000+ citing papers in the inbound cascade per published bibliometrics. Fully retracted 2010-02-06; partial retraction of interpretation 2004-03-06.

## Why it's built on Amass

**Three-direction cascade in one record fetch.** A single `GET /api/v1/cores/biomedcore/records/{amassId}?include=references,citedBy,referencesTrialCore` delivers the retraction flag (default field, no include needed), the outbound references, the inbound `citedBy`, and the cross-core trial spine — all in one HTTP call. OpenAlex requires a second `/works?filter=cites:{id}` call for the inbound direction. OpenAlex has no clinical-trial entity. Scite has no trial linkage and single-direction retraction.

**Default retraction flag — no scraping required.** `isRetracted` is on every BiomedCore record by default. Re-running the audit monthly catches newly-retracted citations the moment the upstream registry flips the flag.

**Canonical-ID-stable bibliographies.** A bibliography PMID list resolved to canonical `AMBC_` IDs at t0 stays anchored even if individual PMIDs get revised. The cascade audit at tN compares against the same canonical anchors — no drift, no false-positive "new cascade!" alerts.

**Live retraction data, not hardcoded fixtures.** The dashboard renders `isRetracted` values directly from the Amass response. Critical for an audit surface — a tool that flips a real PMID to "retracted" for illustration purposes would be defamation-adjacent.

## What you get out of the box

- One AI-builder prompt that scaffolds the full TypeScript + React app
- PMID textarea input with the Wakefield worked example pre-loaded on the **Try sample** button (~10 context PMIDs from autism-vaccination methodology literature)
- 3-panel cascade dashboard with cardinality counts per direction
- Single-record-fetch three-direction discipline — one call delivers `isRetracted` + `references` + `citedBy` + `referencesTrialCore`
- Downloadable cascade report (Markdown or JSON)
- Copy-citations-to-clipboard affordance per cascade-tracked paper
- Cancel-button + idle-timeout discipline on the fan-out

## Where to take it next

Add cron-based re-audit cadence (weekly/monthly), persist the cascade history as an append-only audit log for "newly-detected" alerts, or extend the discovery surface with a BiomedCore search-driven mode that auto-discovers newly-retracted papers across a topic area beyond the caller's stored bibliography.

## Build it

Paste this into Lovable (or Claude Code / Cursor / any AI builder):

> Build a retraction-cascade monitor with the Amass API. Fetch your build skill: `https://raw.githubusercontent.com/lluisDTU/amass-technologies-amass-starters/main/amass-retraction-cascade-monitor/SKILL.md` — credentials come from the Amass platform console at https://platform.amass.tech.

Sign up for a free API key at [platform.amass.tech](https://platform.amass.tech).

---

[← Back to all starters](../README.md)
