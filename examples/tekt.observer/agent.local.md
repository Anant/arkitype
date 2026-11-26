# agent.local.md — Agent capability record

This is the handoff letter for whoever regenerates or rebuilds from this folder. It is environment-local and not part of the portable 00–04 set.

```yaml
extracting_agent:
  identity: automated extract agent
  host_tool: "isolated Linux shell + headless browser, no access to user machines"
  extraction_date: 2026-11-26 (America/New_York)
  method: "arkitype /extract (Phase 0/1/2; files 00–04 + agent.local.md)"
  subject: "xingh/tekt.observer @ 42534a5 (single commit, 2026-09-15), 502 tracked files"
  note: "transfer/rename of the source repo is ON HOLD and was not touched"

capabilities_used:
  filesystem_and_search: "full tree walk; rg over scripts/, frontend/, pocketbase/, deploy/, shared/, tracks/, .agents/, .claude/, .knowledge/, .manage/, .arkitype/, tests/"
  shell_execution:
    - "copied repo to a scratch tree, created .venv (python 3.13.5), installed pytest + pyyaml"
    - "pytest unit+contract: 556 passed, 28 skipped"
    - "imported discover.registry.load_registry() -> 60 modes / 36 modules; generated the 03a catalog from registry metadata"
    - "render_discovery_modes_md.py --check and sync_claude_skills.sh --check: both clean"
    - "run_starter_workflows.sh offline + run_pipeline.sh --live for the shipped watchers (network fetch of 23 public feeds)"
  dev_server_execution: "app_server.py (port 18091) with built frontend (node 24.22.1, npm ci + vite build); serve_html.py (port 18765); both stopped afterwards"
  browser: "captured assets/inbox.png and legacy viewer pages after POST /api/v1/runs/ingest; the feed-page screenshot was omitted from this published extract because it showed a personal name"
  network_access: "public RSS/Atom/HN Algolia feeds only; no authenticated services"

capabilities_missing:
  - gap: "Playwright browsers not installed"
    compensation: "used load_registry instead of available_registry; browser-mode adapters described from source"
    affects: ["03a browser modes (7)"]
    confidence: medium
  - gap: "Did not run PocketBase / docker compose (Mode H)"
    compensation: "schema and rules read from pb_migrations + compose file"
    affects: ["01 Mode H", "02 PocketBase collections"]
    confidence: medium-high (static, but migrations are explicit)
  - gap: "Did not run LLM agent tracks (run_track.sh with codex/claude/gemini) or the coder self-heal loop"
    compensation: "read prompts, runner scripts and e2e fake_codex/fake_gemini doubles"
    affects: ["03 run_track + source-integration loop", "03b track-run skills"]
    confidence: medium
  - gap: "No delivery sent (email/Telegram/Logseq)"
    compensation: "renderers read; formats documented from code"
    affects: ["04 outbound digests"]
    confidence: medium-high
  - gap: "Did not install the cron/launchd scheduler on a real host"
    compensation: "read install_scheduler.sh / configure_schedule.py / run_scheduled_jobs.sh"
    affects: ["01 Mode P", "03 scheduler"]
    confidence: medium-high
  - gap: "Most ATS/employer adapters not exercised live"
    compensation: "contract fixtures + source; 03a marks behavior from code"
    affects: ["03a"]
    confidence: medium

build_toolchain:
  languages: "Python 3 stdlib-heavy scripts; bash runners; React 19 + Vite 7 + Tailwind 3 + TypeScript 5"
  ai_build_tools_evidenced:
    - "The in-repo .arkitype/ folder includes an earlier agent.local.md dated 2026-08-01"
    - "Codex / Claude / Gemini CLIs: first-class agent providers; engineering-gate hooks for Claude (PreToolUse) and Gemini (BeforeTool); .claude/skills mirror"
    - ".manage/ + .archive/2026/*.completed.md plan discipline (e.g. 2026-08-18 watcher-spec refactor, local-portfolio UI)"
  ci: "none in repo (no .github/); scripts/test.sh is the local gate"
  history: "source history is a single squashed commit"

regeneration_requirements:
  minimum: "read these files; Python >=3.10 and Node >=20 to rebuild Mode L"
  for_watchers_live: "outbound HTTPS to public feeds; fallback = fixtures/samples (toggle live_fetch=false)"
  for_browser_adapters: "Playwright + Chromium; fallback = skip the 7 browser modes (they degrade to coverage status failed)"
  for_agent_tracks: "one of codex / claude / gemini CLIs; fallback = deterministic pipeline only"
  for_hosted: "Docker for PocketBase 0.39.11; fallback = Mode L journal only"
  secrets: "names only in 01; supply via *_CMD commands or a local secrets file"

verification_status:
  observed: ["registry and catalog (03a)", "unit + contract tests", "watcher pipeline live run and artifacts (02, 03)", "journal + ingest + local API (02, 03)", "React app Inbox and views (04)", "legacy viewer pages (04)", "starter feed health (00 open_decisions)"]
  static_analysis: ["PocketBase Mode H", "run_track.sh + agent providers", "scheduler install", "source-integration loop", "delivery formats", "agent skills (03b)", "exchange bundle publish via rclone"]

readme_deviations:
  - what: "Files at the top level of the output folder, not in .arkitype/"
    why: "The published extract lives under examples/tekt.observer/; this matches a flat 00–04 layout. The source repo's own .arkitype/ was left untouched."
  - what: "Added supplements 03a-source-adapters.md and 03b-agent-skills.md"
    why: "60 adapters and six skills would have made 03 unreadable; both are referenced from 03 and 00"
  - what: "Added ROADMAP-DELTA.md and README.md"
    why: "Requested mapping onto a later rebuild plan; index for reviewers"
  - what: "defects_observed keys in 03/03b/04 alongside open_decisions"
    why: "Purpose is documentation for a rebuild; observed bugs are as important as unmade decisions"
  - what: "Rule 1 bent in places: 03a lists ~40 employer-specific adapter modes; 04 defect notes name concrete watcher labels"
    why: "These are site-specific by nature; 03a flags them DROP-for-rebuild, and 00 holds all other watcher data"
  - what: "Specified the as-built system, not the repo's forward-looking .arkitype target"
    why: "The in-repo .arkitype/ extraction describes a PocketBase rebuild that isn't implemented; a rebuild needs the real behavior"
  - what: "Omitted one legacy-viewer screenshot and two individual-author feed identifiers"
    why: "They contained personal names; the feed view is still described in 04, and the two feeds are recorded as omitted slots in 00"
```
