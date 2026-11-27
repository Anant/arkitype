# ROADMAP-DELTA — tekt.observer extraction → later rebuild

This file maps what the extraction ([00](./00-arkitype.md)–[04](./04-interface.md), [03a](./03a-source-adapters.md), [03b](./03b-agent-skills.md)) found onto a later rebuild: ① model the system → ② replicate and keep fresh → ③ partition into a graph → ④ Catalog Viewer. The direction is to rebuild from understanding, ship *people skills*, and provide a way to visualize the results. So each capability gets one verdict:

- **SKILL**: keep it as an agent skill (procedure, portable, shipped to people).
- **PATTERN**: keep the design and re-implement it small. Code may be lifted under the source license with notice.
- **VIEWER**: input to the Catalog Viewer.
- **REBUILD**: the idea is right but the implementation doesn't fit. Rebuild against a graph store and the connector registry.
- **DROP**: specific to the upstream job search.

The transfer/rename of `xingh/tekt.observer` is **on hold**. Nothing in this delta depends on it.

## Capability map

| # | Capability (where documented) | Replicator stage | Verdict | Why |
|---|---|---|---|---|
| 1 | Track setup wizard `explore-start` + `explore-discover-sources` ([03b](./03b-agent-skills.md)) | ② connector setup / people skills | **SKILL** | Best UX in the repo: recommend→confirm, a profile-readiness gate, a tiered source pack. Generalize from "job track" to "observer/connector". |
| 2 | Add-one-source `gather-curate-items` ([03b](./03b-agent-skills.md)) | ② connector registry | **SKILL** | Maps directly onto "add a connector for X". |
| 3 | Source-integration self-heal loop: eval canary → ticket → coder agent ([03](./03-software.md)) | ② freshness + ① schema-drift loop | **SKILL** (+ small runner) | This is the "canary detects a source changed → re-extract" loop. Keep it as an agent procedure with a thin ticket file format. |
| 4 | arkitype extract, as applied here (this folder) | ① REA | **SKILL** | Proves `/extract` works on a full app. Gaps feed `extract-saas` (see [agent.local.md](./agent.local.md)). |
| 5 | Judge-against-profile evidence rules (`understand-prioritize-items`) | ③ classify / rank | **SKILL** (generalized) | Rules like "description is primary evidence, say when it's missing" are reusable for any LLM ranking. |
| 6 | `organize-filter-items` | ③ normalize | **REBUILD** as code | A deterministic filter doesn't need an LLM. Keep the coverage-honesty rule as a spec. |
| 7 | Immutable hash-chained journal ([02](./02-database.md)) | ② replication log | **PATTERN** | Needs per-tenant/source partitioning and content retention. Worker claim CAS is missing; fix that on reuse. |
| 8 | Exchange bundles (8 collections, secret-key rejection, revision guard, `rclone --immutable`) ([02](./02-database.md)) | edge replica | **PATTERN** | Starting point for selective graph-slice replication. |
| 9 | `SourceAdapter` registry: contract, Coverage/Candidate, contract fixtures ([03a](./03a-source-adapters.md)) | ② connector registry | **PATTERN** | Use the contract shape, plus `auth_door` and `cursor`. Rebuild about 20 engine adapters (Greenhouse, Lever, Ashby, Workday, …). |
| 10 | ~40 employer/agency-specific scrapers ([03a](./03a-source-adapters.md)) | — | **DROP** | Tuned to one job-search corpus (regional public-sector and security employers). Regenerate on demand with skill #3. |
| 11 | Feed gather rss/atom/hn_algolia + keyless watcher registries ([00](./00-arkitype.md)) | ② public-web pack | **PATTERN** | Small and solid. Fold into a public-web connector pack. 7 of 23 starter feeds failed or were empty, so add health checks. |
| 12 | enrich (OG/meta) → classify (keyword lexicons) → trends → rerank (feedback) ([03](./03-software.md)) | ③ normalize + classify | **REBUILD** | The lexicon classifier has low discrimination (news_ai took 87 of 105). Replace it with typed documents, taxonomy, and entity resolution. Keep the feedback-rerank idea and the trend/keyword-cloud output. |
| 13 | Taxonomy tracks: topic/market/career watchers, persona synthesis ([00](./00-arkitype.md)) | ③ doc_type schemas | **PATTERN → graph schemas** | Watcher taxonomies become domain-schema drafts (`ai_news`-style), and personas become digest templates. |
| 14 | Two runners (`run_pipeline.sh` watchers vs `run_track.sh` ATS + LLM) and the scheduler/worker ([03](./03-software.md), [01](./01-infrastructure.md)) | ② supervisor | **REBUILD** | The scheduler and PocketBase worker can only run `run_track.sh`, so the shipped watchers can't be scheduled. Build one supervisor for all connectors. |
| 15 | PocketBase Mode H scaffold ([01](./01-infrastructure.md), [02](./02-database.md)) | ④ hosted | **REBUILD / DROP** | Not at parity: read-only mount, no worker image, and the UI doesn't talk to it. Hosted mode should use the graph store plus the platform's own auth. Keep PocketBase only if an offline multi-user mode is wanted. |
| 16 | Three overlapping stores (journal, PocketBase, legacy portfolio JSON) with no projector | ② / ④ | **REBUILD** | One log → one projection (the graph store). Write it as an ADR in the new repo. |
| 17 | Job-centric artifacts (digest, ranked_jobs, seen_jobs, email "job digest") ([02](./02-database.md), [04](./04-interface.md)) | ③ | **REBUILD** | Generalize to Document/Concept/Edge with provenance + ACL. |
| 18 | Delivery: email, Telegram (3900-char parts), Logseq ([04](./04-interface.md)) | ④ outbound awareness | **PATTERN** | Keep it as "digest out" channels driven by the "what changed" view. Copy comes from the profile. |
| 19 | Engineering-gate hooks (Claude `PreToolUse`, Gemini `BeforeTool`) ([03b](./03b-agent-skills.md)) | agent surface | **PATTERN** | Cheap way to enforce "load the rules before editing". |
| 20 | Legacy server-rendered viewer + static multitrack site ([04](./04-interface.md)) | ④ | **VIEWER** (port report + trends, then retire) | The "browsable daily report" is "what changed today". |

## What the Catalog Viewer needs

**Reuse from the React app** ([04](./04-interface.md)): the shell (sidebar, sticky header, theme, skip link, focus ring), Inbox mechanics (status tabs, search on `/`, list/grid toggle, 75th-percentile default relevance floor with "Top 25% / Show all", save/dismiss/restore, provenance drawer), Watchers→Connectors health cards, Operations polling (3 s while a job is running), Exports (hash-verified bundle + validate), the demo-data fallback, and the design tokens in [00 brand](./00-arkitype.md).

**Fix on the way over:**
- Watcher chip labels use the first word of the name, so two chips read "AI". Use slugs or explicit short labels.
- Provenance shows machine rationale strings. Render real `SOURCED_FROM` evidence instead.
- Import is validate-only. Add apply-import, or drop the button.
- Members is a placeholder. Hosted mode needs platform auth + per-user ACL.
- Exactly one API surface. Today there are three: `app_server` `/api/v1/workspace`, legacy `serve_html` `/api/v1/*` and PocketBase.

**New views and types:** a Catalog API backed by the graph store (`list documents`, `query concepts`, `neighbors`, `explore`, `recall`) and the connector registry; views for Catalog (connectors → documents → freshness), Entity page (three-clock timeline), Relationship graph, What changed (port legacy report + trends), and Schema (doc_types + taxonomy). Generalize the types from `Signal` to `Document` + `Concept` + `Edge` with ACL and provenance.

The observer frontend is small (`App.tsx` is 118 lines), so a clean rebuild that keeps its interaction spec costs little more than forking it.

## Better rebuilt than ported (summary)

Runners/scheduler/worker (#14), the classifier (#12), the hosted PocketBase mode (#15), the store layout (#16), job-centric schemas and copy (#17), and the ~40 single-employer scrapers (#10). Carry over the **specs** (this folder) and the **skills** (#1–5). Carry over code only for the journal, bundles, feed gather and adapter contract (#7–9, #11), with notice (below).

## Phase fit

- **Phase 0:** add "rebuild, not move" as an option next to the on-hold transfer. With these docs, the repo isn't needed as a code base. Licensing note below.
- **Phase 1** (chat exports → graph): reuse journal pattern #7. Use the viewer reuse list above for Catalog + Entity + "new since last sync".
- **Phase 2** (a second internal source + cursors): adapter contract #9 extended with `cursor`, plus supervisor #14.
- **Phase 3** (extract-saas + public sources): skill #4 → `extract-saas`; feed pack #11; self-heal skill #3 as the drift loop; port the "what changed" view (#20).
- **Phase 4** (hosted): bundles #8 for edge slices; hosted viewer replaces #15.

## License and attribution

Source repo license: MIT (see `xingh/tekt.observer`).

- Specs and prose in this folder describe the system; they do not copy source code.
- If any **code** is later lifted from `xingh/tekt.observer` (journal, bundle, feed gather, adapter contract, any adapter, the React app), keep the MIT notice from that repo with the copy, for example in a `THIRD_PARTY_NOTICES` file plus a header comment on lifted files.
- A clean-room rebuild from these specs has no such notice requirement.
- Third-party deps seen: PocketBase server + JS SDK (MIT), React (MIT), TanStack Query (MIT), Tailwind (MIT), clsx/tailwind-merge (MIT), lucide-react (ISC), Playwright (Apache-2.0, optional), pytest (MIT, dev). No copyleft in the direct dependencies. Transitive npm licenses were not audited.
- The brand note "do not copy third-party branding or assets" ([00 brand](./00-arkitype.md)) still applies to any visual reuse.
