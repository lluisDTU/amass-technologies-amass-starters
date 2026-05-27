# Trust-Filtered Biomedical RAG (MCP)

> A trust-filtered biomedical RAG built around an MCP server — two MCP tools wrapped as a Next.js chat UI for the first-run experience, framework-agnostic stdio launcher for production agent stacks. JuFo + retraction + citation-count post-filters wired into the same response that delivers the cited evidence.

## Who it's for

LLM-tooling builders shipping biomedical AI agents — Claude Desktop tool authors, MCP-aware Cursor users, custom-orchestrator developers — who need a credibility-filtered evidence layer their agent can call without composing PubMed + retraction-DB + JuFo lookups by hand on every tool invocation.

## The job it does

The scaffold ships two layers in one repo:

1. **An MCP server** in `lib/mcp-tools.ts` exposing two tools:
   - `fetch_credible_evidence_by_id(ids[], min_jufo?, allow_retracted?, min_citation_count?)` — returns canonical-AMBC-resolved + trust-filtered evidence records.
   - `get_credible_paper_with_trials(amass_id, include_fulltext?)` — returns a single paper with cross-core trial linkage (and optional full text).

2. **A Next.js chat UI** consuming the same MCP tool handlers in-process, so a Lovable / Claude Code / Cursor user sees the end-to-end MCP flow on first `npm run dev`.

The builder can lift the MCP server boundary out into their own agent stack by pointing `npx @modelcontextprotocol/inspector` or a stdio launcher (`node lib/mcp-tools.js`) at the same tool definitions — the chat UI is a first-run convenience, not a production constraint.

The worked example binds to PMID 37861218 (Ahn MJ et al. NEJM 2023 — Tarlatamab DeLLphi-301 primary publication).

## Why it's built on Amass

**Trust-filter conjunction in one response.** `journalQualityJufo` (JuFo tier 1-3) + `isRetracted` (default field) + `citationCount` all wired into the same record — three credibility signals an agent can post-filter on in one round-trip. Competitors require 2-3 separate API calls (PubMed for metadata, Retraction Watch DB scrape, JuFo-table lookup) and a join layer on the agent side.

**Canonical-ID resolution as a primitive.** The MCP tool accepts PMID, DOI, or `AMBC_` interchangeably. Lookup translates external identifiers in one batch call. The agent doesn't need to maintain its own identifier-mapping cache.

**Paper↔trial cross-core walk.** The `get_credible_paper_with_trials` tool returns `referencesTrialCore` + `citedBy` arrays on the paper record — the agent can follow up with structured trial-side enrichment without text-mining NCT identifiers out of full-text passages.

**Apache-2.0 framework-agnostic MCP server.** `lib/mcp-tools.ts` is a plain Node.js + `@modelcontextprotocol/sdk` module. It works under Next.js, TanStack Start, Bun, Deno, or as a standalone stdio launcher under Claude Desktop. The chat-UI wrapper is the only framework-specific layer.

## What you get out of the box

- One AI-builder prompt that scaffolds the full app (MCP server + chat UI in one repo)
- Two production-shaped MCP tools with Zod-validated input schemas
- Per-message threshold overrides exposed as chips above the chat input (`min_jufo`, `allow_retracted`, `min_citation_count`, `include_fulltext`)
- Tarlatamab worked example wired to the empty-state **Try sample** button
- Evidence-card rendering per assistant turn — PMID + JuFo badge + isRetracted chip + citationCount + cross-core trial chips
- "MCP tool call" disclosure showing the underlying tool payload as JSON — the builder's debugging affordance for lifting the boundary into their own agent stack
- Cancel-button + idle-timeout discipline on the per-message fan-out

## Where to take it next

Wire the MCP server to your own agent (Claude Desktop, Cursor, or a custom orchestrator) via the stdio launcher. Add a third MCP tool for retraction-cascade enumeration, or wrap the chat-UI in a server-sent-events streaming layer for long fan-outs. The starter is explicit about where the MCP boundary sits — extension is a path, not a refactor.

## Build it

Paste this into Lovable (or Claude Code / Cursor / any AI builder):

> Build a trust-filtered biomedical RAG MCP server with chat UI on the Amass API. Fetch your build skill: `https://raw.githubusercontent.com/lluisDTU/amass-technologies-amass-starters/main/amass-trust-filtered-rag/SKILL.md` — credentials come from the Amass platform console at https://platform.amass.tech.

Sign up for a free API key at [platform.amass.tech](https://platform.amass.tech).

---

[← Back to all starters](../README.md)
