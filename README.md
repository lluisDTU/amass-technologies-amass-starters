# Amass API Starters

> Six end-to-end starter apps that show what's distinctive about the [Amass API](https://api.amass.tech) — built for biomedical literature and clinical-trial workflows that depend on canonical IDs, cross-core links, and trust filters at the non-commercial tier.

**v0.1.0 — initial public release.** Apache-2.0.

---

## What this is

Each starter is a self-contained TypeScript + React app you build by pasting a single prompt into Lovable, Claude Code, Cursor, or any AI builder. The `SKILL.md` file in each starter directory is the prompt — fetch it as a raw URL and the builder scaffolds the working app against the Amass API.

Each starter gives you a running head start: a real workflow, validated against a worked example, with the error semantics, trust filters, and paper↔trial cross-core walks already wired up. Fine-tune them for your data and your users.

---

## The six starters

| Starter | What it does | One-page pitch |
| --- | --- | --- |
| [`amass-regulatory-evidence-assembler`](./amass-regulatory-evidence-assembler/) | Auditable literature-evidence assembler for FDA / EMA regulatory submissions (CTD Module 2.5 / 2.7 / PSUR). Resolves a curated submission scope to canonical Amass IDs and emits an audit-CSV anchoring each cited paper to its supporting trial. | [Read →](./docs/amass-regulatory-evidence-assembler.md) |
| [`amass-pipeline-monitor`](./amass-pipeline-monitor/) | Weekly competitive-intelligence digest for biotech pipeline-watchers. Takes a sponsor watchlist and surfaces new Phase 2/3 trials, new evidence describing them, and retraction-flagged citations — in one 3-panel dashboard. | [Read →](./docs/amass-pipeline-monitor.md) |
| [`amass-sr-pre-screen`](./amass-sr-pre-screen/) | PRISMA pre-screen credibility-filter for systematic reviews. Drops a ~5,000-PMID search dump to ~100-500 papers before title/abstract screening — emits a Rayyan/Covidene RIS plus an audit-trail CSV. | [Read →](./docs/amass-sr-pre-screen.md) |
| [`amass-trust-filtered-rag`](./amass-trust-filtered-rag/) | Trust-filtered biomedical RAG built around an MCP server. Two MCP tools (`fetch_credible_evidence_by_id` + `get_credible_paper_with_trials`) wrapped as a chat UI — lifts cleanly into Claude Desktop, Cursor, or any MCP-aware agent stack. | [Read →](./docs/amass-trust-filtered-rag.md) |
| [`amass-retraction-cascade-monitor`](./amass-retraction-cascade-monitor/) | Retraction-cascade tracker that re-audits a stored bibliography for newly retracted papers AND surfaces the cascade — inbound, outbound, and cross-core trial spine — so SR teams, regulatory affairs, and medical-info teams pull contaminated evidence before downstream harm. | [Read →](./docs/amass-retraction-cascade-monitor.md) |
| [`amass-claim-to-trial-verifier`](./amass-claim-to-trial-verifier/) | Marketing-claim-to-trial verifier for OPDP-adjacent review and pharma-claim journalism. Surfaces the trial's registered primary endpoint verbatim from the protocol next to the promotional claim — with a flagged verdict naming the specific match or mismatch. | [Read →](./docs/amass-claim-to-trial-verifier.md) |

---

## Build it in five minutes

Each starter is built by pasting one prompt into your AI builder. The template:

> Build a **[STARTER NAME]** on the Amass API. Fetch your build skill: **[RAW SKILL.md URL]** — credentials come from the Amass platform console at [platform.amass.tech](https://platform.amass.tech).

For example, to build the regulatory-evidence assembler in Lovable:

> Build an auditable regulatory-evidence assembler on the Amass API. Fetch your build skill: `https://raw.githubusercontent.com/lluisDTU/amass-technologies-amass-starters/main/amass-regulatory-evidence-assembler/SKILL.md` — credentials come from the Amass platform console at https://platform.amass.tech.

The builder fetches the SKILL.md, scaffolds the app, runs `npm install`, and lands you at `npm run dev` with one env-var (`AMASS_API_KEY`) to paste in. Each one-page pitch (linked in the table above) has the exact paste-ready prompt for its starter.

---

## Why build on Amass

The Amass API exposes two live Cores — **BiomedCore** (biomedical publications) and **TrialCore** (clinical trials) — with a third (**RegulatoryCore**) on the way. Four primitives drive what these starters can do:

**Canonical Amass IDs (`AMBC_` / `AMTC_`)** survive PubMed and ClinicalTrials.gov revisions. A week-over-week audit keyed on a canonical ID doesn't drift when the upstream registry rewrites a PMID or an NCT underneath it — load-bearing for any workflow with a submission-to-approval gap or a multi-week diff cycle.

**Lookup endpoints** translate external identifiers (PMID, DOI, NCT) into canonical Amass IDs in one batch call, with **per-item-error semantics** that surface a single bad identifier inline without crashing the batch. Each `POST /records/lookup` array element echoes the input alongside either `amassIds: [...]` for success or `error: { code, message }` for failure — auditable error trails come for free.

**Cross-core walks** — paper→trial via `referencesTrialCore`, trial→paper via `referencesBiomedCore` — are exposed as structured array fields on a single record fetch. Competitors require 2-4 calls across separate entity-key systems (PubMed + ClinicalTrials.gov + OpenAlex) and lose the structured guarantee in the process.

**Trust filters at the non-commercial tier** — `isRetracted` (default field), `journalQualityJufo` (JuFo journal-quality tier 1-3), `citationCount` — wired into the same response that delivers the cited evidence. No per-paper retraction-DB scraping, no JuFo-table lookup, no separate citation-count API.

---

## Get an API key

Sign up at [platform.amass.tech](https://platform.amass.tech) — the non-commercial tier is free and covers all six starters end-to-end.

---

## Community

Questions, bug reports, or pull requests welcome. Join the Amass Developer Community on Discord: [discord.com/invite/sEGaBHMhWa](https://discord.com/invite/sEGaBHMhWa).

---

## License

Apache-2.0. Copyright 2026 Amass Technologies.
