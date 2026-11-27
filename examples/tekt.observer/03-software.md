# 03 — Software: pipeline, CLI, APIs, scheduler, agents

This file covers server-side and batch logic. Per-instance values (watchers, lexicons, thresholds) come from [00 SITE_PROFILE](./00-arkitype.md). Persisted shapes are in [02](./02-database.md). The 60 discovery adapters are specified in [03a](./03a-source-adapters.md), and the six agent skills in [03b](./03b-agent-skills.md).

## Component boundaries

```yaml
boundaries:
  deterministic_pipeline: "Python scripts; one stage per script; JSON in, JSON out, under a JOB_AGENT_ROOT. No LLM calls for the shipped watchers."
  discovery_adapters: "plugin registry of SourceAdapter objects keyed by discovery_mode (ATS / career pages / public-sector boards)"
  feed_gatherer: "stdlib RSS/Atom/HN-Algolia reader for watcher registries"
  journal: "immutable_json_store: durable append-only events + snapshots (02 §1)"
  local_app_server: "stdlib HTTP server: static React build + JSON API over the journal (Mode L)"
  legacy_viewer: "server-rendered HTML portfolio + /api/v1 over raw artifacts (serve_html.py)"
  pocketbase_worker: "polls PocketBase operations and shells out to run_track.sh (Mode H scaffold)"
  agents: "Claude Code / Codex / Gemini CLIs, used for track setup, source curation, source repair (coder + reviewer) and LLM digests of ATS-style tracks"
  rule: "Agents never replace deterministic routine fetching; they create or repair the deterministic parts."
```

## Pipeline: discover → enrich → classify → trends → rerank → synthesize → render

The orchestrator is `scripts/run_pipeline.sh --track <slug> [--live] [--today D] [--audience A] [--scratch P] [--registry R] [--append]`. Each stage runs only if its inputs (and, where relevant, its per-watcher script) exist. It discovers per-watcher scripts **by filename**: `scripts/<slug>_discover.py`, `scripts/<slug>_classify.py`, `scripts/<slug>_synthesize_digest.py`.

```yaml
pipeline:
  scratch_setup: "see 01 mode_P.scratch_isolation; rm -rf scratch unless --append"
  stages:
    - id: gather
      when: --live
      run: "feed_gather.py --root S --track T --date D --registry shared/schemas/<T>_source_registry.json [--workers 6] [--for-date D]"
      behavior:
        - "load registry {sources:[{id,name,kind,url,topic_hints}]}; fetch each source concurrently (ThreadPoolExecutor, max workers = min(workers, n)); output preserves registry order"
        - "HTTP GET with User-Agent profile.runtime_defaults.user_agent ('feed-gather'), timeout 15s, default TLS context"
        - "parse by kind: rss (channel/item: title, link, description, pubDate or dc:date, dc:creator or author), atom (entry: title, first/alternate link href, summary else content, published else updated, author/name), hn_algolia (hits[]: title|story_title, url|story_url else news.ycombinator.com/item?id=<objectID>, created_at, author; empty description); descriptions HTML-stripped and cut to 400 chars"
        - "strip HTML from descriptions; cap at MAX_PER_SOURCE=20 items per source"
        - "--for-date: hn_algolia gets numericFilters=created_at_i>=start,created_at_i<=end server-side; rss/atom filtered client-side by parsed pubDate; if all filtered, add limitation window_filtered_out:<n>"
        - "per-source degradation: fetch error -> status failed + fetch_error:<msg>; parse error -> failed + parse_error:<msg>; unknown kind -> failed; zero items -> partial; else complete"
        - "candidates: employer=source name, location='content', remote='unknown', matched_terms=topic_hints, notes='kind=<kind>', plus published_at, author, topic_hints"
        - "write artifacts/discovery/T/D.json and latest.json; stderr progress '[gather] <id>: <status> (<n> items)'"
    - id: fixture_discover
      when: "not --live"
      run: "scripts/<T>_discover.py if present (topic_watch has a local-HTML fixture reader) else write an empty discovery artifact"
      known_bug: "topic_watch_discover.py requires tracks/topic_watch/sources.json, which a fresh checkout does not have, so non-live run_pipeline.sh --track topic_watch fails. Use run_starter_workflows.sh (sample seeding) for offline demos."
    - id: enrich
      when: --live
      run: "feed_enrich.py --root S --track T --date D"
      behavior:
        - "for up to MAX_URLS=60 candidate URLs not already in artifacts/enrichment/T/urls.json: GET with Range bytes=0-199999, timeout 12s, concurrent"
        - "parse <meta> og:*, twitter:*, description, article:published_time, author, <link rel=canonical>, <title>"
        - "og_image = og:image or twitter:image; og_title = og:title or <title>; og_description = og:description or description or twitter:description"
        - "merge into the URL cache (shared across dates), and patch each discovery candidate in place with an 'enrichment' object"
    - id: classify
      when: "scripts/<T>_classify.py exists"
      contract: "read discovery; emit organized items (02 §4) with item_key, topic, content_type, categories, audiences, confidence, rationale"
      shared_helpers: {item_key: "<prefix>-<sha1(url|title)[:12]>", substring_match: "case-insensitive substring", word_boundary_match: "regex \\b..\\b for tickers"}
      variants:
        topic_watch:
          topic: "argmax over profile.classifier_lexicons.topic_watch.topic_keywords by count of substring hits in TITLE; ties keep the earlier topic; default news_ai"
          content_type: "first content_type_hints needle found in 'url title' (lowercased); default post"
          audiences: "every audience whose audience_keywords hit the title; default [architects, builders]"
          categories: "[topic]"
          confidence: "min(1, 0.4 + 0.2*topic_hits + 0.1*audience_hits)"
          taxonomy_source: "resolved_classifier_taxonomy(): shipped shared/schemas/<T>_taxonomy.json overlaid by tracks/<T>/taxonomy.json labels"
        market_watch:
          asset_class: "argmax keyword hits over taxonomy.asset_classes[].keywords (title); else the source's first topic_hint; else public_equities"
          event_type: "first event_keyword_map entry hit whose event type is allowed for the class; else the class's first event_type; else news"
          watchlist: "tickers by word boundary, ticker_regex, companies/funds/anchors by substring; if watchlist classes do not contain asset_class, switch to the first watchlist class and recompute event_type"
          is_portfolio_alert: "any watchlist hit"
          confidence: "min(1, 0.4 + 0.15*class_hits + 0.2*min(watchlist_hits,3))"
          audiences: "profile.market_watch_rules.audiences_by_asset_class"
          mirrors: "topic=asset_class, content_type=event_type (so the shared viewer renders unchanged)"
        career_watch:
          role_type: "argmax keyword hits over taxonomy.role_types[].keywords in 'title description'; default ai_augmented_generalist"
          seniority: "first seniority_keywords bucket (taxonomy order) with a hit; default senior_ic"
          remote: "any remote_keywords in 'title description'"
          company: "COMPANY_HINT_RE on title, else text after ' at ' (before '('), else after ' @ ', else source name"
          audiences: "[seniority] + [instructor] when role_type == ai_instructor"
          confidence: "min(1, 0.4 + 0.15*role_hits + 0.1*seniority_hits + 0.1 if remote)"
          mirrors: "topic=role_type, content_type='posting'"
    - id: trends
      when: "organized artifact exists"
      run: "track_trends.py"
      behavior: "counts per topic/source/content_type/audience; velocity = today count - previous organized artifact count per topic; cross_source_urls = URLs (query string and trailing slash stripped) seen from >1 source_id; top 25 title tokens excluding STOPWORDS"
      known_bug: "Stripping the query string merges every news.ycombinator.com/item?id=N into one 'cross-source' URL (seen on 2026-11-26 for all three watchers)."
    - id: rerank
      run: "track_rerank.py --with-feedback"
      behavior:
        - "audiences = shared/schemas/<T>_taxonomy.json .audiences; none -> stage no-op"
        - "for each audience: boost = 1.5 if audience in item.audiences else 0.55; +0.25 if is_portfolio_alert and audience in {investors, portfolio_managers, allocators}"
        - "feedback_delta = per_item_boosts(events.jsonl for T/audience): for each item_key keep the boost of the latest event, unless an earlier event has a strictly stronger |boost| (a later event replaces only if ts is newer AND |boost| >= previous)"
        - "audience_score = clamp(confidence*boost + feedback_delta, 0, 1) rounded to 3dp; sort by (-score, title); rank 1..n"
    - id: synthesize_audience
      run: "synthesize_audience_digests.py --max-top 8"
      behavior: "per audience: top_matches = first 8 ranked items; other_new_roles = the rest; source_notes from discovery; write JSON + markdown"
    - id: synthesize_persona
      when: "scripts/<T>_synthesize_digest.py exists"
      variants:
        topic_watch: "top_matches = items whose audiences include --audience (default architects), sorted by confidence, first 8; other = remaining matched + unmatched; fit_score = confidence*10; recommendation apply_now if confidence >= 0.7 else watch"
        market_watch: "top_matches = all portfolio alerts sorted by confidence (section portfolio_alerts); other = rest (situational_awareness); same fit/recommendation rule"
        career_watch: "top 8 by confidence; rest other"
      executive_summary: "'<n> items surfaced across <k> topics; <m> match the <audience> audience lens.' (topic_watch form)"
    - id: render
      run: "render_digest.py (markdown from digest JSON via digest_json.render_digest_markdown); HTML via render_html.py / serve_html.py / render_multitrack_site.py (04)"
  backfill:
    run: "backfill.sh --track T (--start D --end D | --dates 'D1 D2 ...')"
    per_date: [feed_gather --for-date, feed_enrich, "<T>_classify", track_trends, "track_rerank --with-feedback", synthesize_audience_digests, "<T>_synthesize_digest", render_digest]
    guard: "skips future dates; re-running a date overwrites its artifacts"
  starter_workflows:
    offline: "run_starter_workflows.sh -> seed_starter_workspace.py writes organized artifacts from each spec's samples.json into tests/tmp/starter-workflows"
    live: "run_starter_workflows.sh --live -> run_pipeline.sh --live for each slug from generate_watchers.py --list (first run clears scratch, later ones --append)"
```

## ATS-style tracks: `run_track.sh`

```yaml
run_track:
  usage: "run_track.sh --track <slug> [--delivery logseq|email|telegram]... [--timeout-secs N] [--discovery-timeout-secs N]"
  steps:
    - "load .env.local + secrets (load_runtime_env.sh); macOS: re-exec under caffeinate"
    - "discover_jobs.py --track T --today D --due-only --pretty --progress --output artifacts/discovery/T/D.json --latest-output .../latest.json, with a timeout watchdog and heartbeat log lines"
    - "build a prompt file and run the provider agent non-interactively (codex: exec -s workspace-write; claude: -p --no-session-persistence --output-format stream-json --permission-mode acceptEdits --allowedTools ...; gemini: --approval-mode yolo, sandbox off by default) with JOB_AGENT_ROOT/TRACK/TODAY/PROVIDER/BIN exported; stdout goes through a FIFO that timestamps activity for an inactivity watchdog"
    - "the agent follows tracks/T/AGENTS.md using organize-filter-items then understand-prioritize-items and writes artifacts/digests/T/D.json (digest schema 02 §4)"
    - "update_source_state.py: set last_checked=D for every configured source whose discovery status == complete"
    - "render_digest.py -> tracks/T/digests/D.md"
    - "update_seen_jobs.py -> append top_matches + other_new_roles as {date_seen, company, title, location, url} to tracks/T/seen_jobs.json"
    - "update_ranked_overview.py -> rebuild ranked_overview.md / shared/ranked_jobs/ from markdown digests (parses 'Fit score: x/10', 'Link:', 'Location:')"
    - "optional delivery: sync_to_logseq.sh, send_digest_email.py, send_digest_telegram.py"
  discover_jobs:
    config: "tracks/T/sources.json (validated by source_config.load_sources_config) + source_state.json last_checked"
    due_rule: "every_run -> always; never checked -> due; every_month -> different calendar month; every_3_runs -> >= 3 days since last_checked"
    terms: "normalize_terms(track_terms, source.search_terms): override replaces, append extends; dedupe case-insensitively"
    per_source: "adapter.discover(SourceConfig, terms, timeout) -> Coverage; then filter_coverage_for_track (match_rules.json keep_if_any_text_term); then enrich_candidate_descriptions (JD fetch bounded by 4000 chars, 60s wall clock, 200 fetches)"
    modes: "--list-sources, --plan-only, --source NAME, --cadence-group G, --due-only"
    keep_rule: "should_keep_candidate: reject NON_TECHNICAL title hints; require matched terms; keep if a title term match + technical title, or technical title + specialized term in matched or body"
  known_gap: "The shipped watchers have no tracks/<slug>/sources.json, so run_track.sh and therefore the PocketBase worker's run_watcher operation cannot run them; only run_pipeline.sh can."
```

## Scheduler

```yaml
scheduler:
  configure: "configure_schedule.py --track T --cadence daily|weekly|monthly --time HH:MM [--weekday mon..sun] [--month-day 1..31] [--delivery x]... -> create or replace the single line for T in .schedule.local (keeps a generated header)"
  validation: {track: "^[A-Za-z0-9._-]+$", time: "^([01][0-9]|2[0-3]):[0-5][0-9]$", weekly_requires: weekday, monthly_requires: month_day, daily_forbids: [weekday, month_day]}
  dispatcher: "run_scheduled_jobs.sh, invoked every minute by cron/launchd"
  dispatcher_algorithm:
    - "mkdir .scheduler/run.lock or exit 0 (single instance); trap removes it"
    - "for each non-comment line: parse cadence; due if (daily) now >= HH:MM; (weekly) weekday matches and now >= HH:MM; (monthly) day matches and now >= HH:MM"
    - "skip if .scheduler/state/<entry-key>.stamp (or a legacy time-less stamp) starts with today's date: at most one run per entry per day, with catch-up later the same day"
    - "run run_track.sh --track T [--delivery ...]; on success write stamp YYYY-MM-DD-HH:MM; invalid lines set exit status 1 but do not stop other entries"
  gap: "Dispatches only run_track.sh (ATS-style). Feed watchers have no scheduled path except wrapping run_pipeline.sh by hand."
```

## Local app API (Mode L, `scripts/app_server.py`)

```yaml
local_api:
  base: "http://<app_host>:<app_port>"
  common: "JSON responses use canonical bytes, Cache-Control: no-store, X-Content-Type-Options: nosniff; HTML gets no-store; unknown non-API paths fall back to /index.html (SPA)"
  routes:
    - {method: GET,   path: /api/v1/workspace, out: "{workspace, watchers (by slug), sources (by watcher,name), items (observedAt desc), operations (updatedAt desc), digests (createdAt desc), exports (createdAt desc)}"}
    - {method: PATCH, path: "/api/v1/items/{id}", in: "{status: new|saved|dismissed} (exactly this key)", effect: "put item with new status; bump workspace revision", errors: {400: bad body, 404: unknown}}
    - {method: PATCH, path: "/api/v1/watchers/{id}", in: "{enabled: bool}", effect: "put watcher enabled + status healthy|paused (does not stop runs)"}
    - {method: POST,  path: /api/v1/digests, effect: "digest-NNNN of the top 5 non-dismissed items by (-score, title); summary '<n> active signals across <w> watchers, with <k> highlighted below.'", status: 201}
    - {method: POST,  path: /api/v1/exports, effect: "build bundle from journal state (producer tekt.observer.local), write <store>/exports/<id>.tekt-observer.json, journal an export record", out: "record + downloadUrl", status: 201}
    - {method: GET,   path: "/api/v1/exports/{filename}", out: "attachment download; filename ^[A-Za-z0-9._-]+$"}
    - {method: POST,  path: /api/v1/runs, effect: "refuse if a live_tracks operation is queued/running; journal queued op; daemon thread under a global lock runs run_starter_workflows.sh --live --scratch tests/tmp/starter-workflows (log in <store>/run-logs/), then ingest; op -> complete|failed", status: 202}
    - {method: POST,  path: /api/v1/runs/ingest, effect: "ingest_run_artifacts(tests/tmp/starter-workflows)", status: 201}
    - {method: POST,  path: /api/v1/imports/validate, in: "bundle JSON <= 20 MiB", out: "{valid, bundleId, workspaceId, workspaceRevision, schemaVersion, counts}", note: "validation only; local import is not implemented"}
  body_limits: {patch: 65536, import: 20971520}
  ingest_run_artifacts:
    - "for each artifacts/organized/<track>/ (latest file, or the given date): load enrichment cache and the persona digest"
    - "score map from digest top_matches: job_key -> min(100, fit_score*10)"
    - "put one item per organized row (02 local_records.items), preserving an existing item's status"
    - "put run <track>:<date>; put digest pipeline:<track>:<date> when the digest has a summary or top matches"
    - "delete all sample items; bump workspace revision; journal operation live-import:<date>; append_many in ONE lock hold; force a snapshot"
    - "observed 2026-11-26: 300 rows -> 292 items (same item_key across sources collapses)"
  auth: "none; loopback bind is the only control"
```

## Legacy viewer API (`scripts/serve_html.py`)

```yaml
legacy_api:
  bind: "127.0.0.1 by default; any non-loopback bind makes it read-only"
  csrf: "secrets.token_urlsafe(32) per process; injected as window.TEKT_CSRF and sent as X-CSRF-Token by a fetch wrapper"
  write_requirements: [loopback bind, "Content-Type: application/json", "same Origin when present", "X-CSRF-Token match", "body <= 256 KiB"]
  read_routes: ["GET /api/v1/state", "GET /api/v1/items?track&interest&audience&topic&q", "GET /api/v1/interests", "GET /api/v1/portfolios", "GET /api/v1/tracks/<t>", "GET /api/v1/tracks/<t>/taxonomy", "GET /api/v1/operations", "GET /api/v1/operations/<id>"]
  write_routes: ["POST /feedback (legacy alias)", "POST /api/v1/feedback", "POST|PUT|DELETE /api/v1/interests", "POST|PUT|DELETE /api/v1/portfolios", "PUT /api/v1/tracks/<t>", "PUT /api/v1/tracks/<t>/taxonomy", "POST /api/v1/operations {track, kind: run|validate_sources|schedule, ...}", "POST /api/v1/operations/<id>/cancel"]
  html_routes: "see 04-interface.md#legacy-server-rendered-viewer"
  unified_items: "union of organized, ranked_audience, digests and discovery artifacts, keyed '<track>:<item_key>'; the highest audience score wins"
  operations: "PortfolioOperations: one active op per track; subprocess with stdout captured (last 64,000 chars); states per 02; persisted to logs/portfolio-operations.json (cap 200); cancel kills the process and keeps state=cancelled"
  delete_guards: "an interest still referenced, and the default portfolio, can't be deleted"
```

## Unified CLI (`tekt.observer`)

```yaml
cli:
  up: "[--hosted] [--host] [--port]: Mode L (ensure frontend + app_server) or Mode H (docker compose up -d)"
  app: "[--host] [--port]: same as local up"
  ingest: "[--scratch PATH] [--date D]: ingest_run_artifacts into TEKT_OBSERVER_STORE; prints a JSON summary"
  store: "put <collection> <id> <json|@file> | delete <collection> <id> | compact [--force] | show"
  worker: "[--poll-seconds 5] [--once]: PocketBase operations loop (below)"
  import: "<bundle> [--fork-slug S]: into PocketBase as a new workspace"
  export: "--workspace ID [--output P]: PocketBase -> bundle (default exports/<id>.tekt-observer.json)"
  publish: "--workspace ID --bundle P --remote R: revision guard + rclone copy --immutable"
  seed-local: "PocketBase: create workspace slug=local and the starter watchers"
  errors: "BundleError, StoreError, PocketBaseError, JSON, OS and subprocess errors print 'tekt.observer: <msg>' to stderr and exit 1"
worker_loop:
  - "list operations status='queued' sorted by created; take the first"
  - "update status=running, claimed_at=now, attempt+=1   # no claim_token compare-and-set: two workers can double-claim"
  - "only kind=run_watcher with input.watcher_slug is supported -> bash scripts/run_track.sh --track <slug>"
  - "success -> status=complete, result={watcher_slug}; failure -> queued again if attempt < max_attempts else failed, error=str(exc)"
  - "journal compact_if_due(); sleep poll_seconds"
pocketbase_client: "urllib REST client: authenticate(identity, password) via auth-with-password, list_records (perPage=200, follows totalPages, filter, sort), create_record, update_record, upsert_records(key)"
```

## Watcher specification and generation

```yaml
watcher_generation:
  spec_root: .arkitype/watchers/<slug>/
  files: [watcher.json, brief.md, taxonomy.json, sources.json, samples.json]
  validation:
    - "slug ^[a-z][a-z0-9_]{0,63}$ equals the directory name"
    - "source kinds in {rss, atom, hn_algolia}"
    - "every source topic_hint and every sample topic/topic_ids must exist in the primary dimension (topics | role_types | asset_classes)"
  outputs: ["tracks/<slug>/track.json", "tracks/<slug>/prefs.md", "shared/schemas/<slug>_taxonomy.json", "shared/schemas/<slug>_source_registry.json"]
  modes: {default: write, --check: "diff; also reports stamped generated files whose spec no longer exists", --list: "print slugs in 'order' order"}
  add_a_watcher: "copy a spec dir, edit, run the generator, optionally add <slug>_classify.py and <slug>_synthesize_digest.py; gather, enrich, trends, rerank, audience digests and rendering are generic"
```

## Source integration (self-healing) loop

```yaml
source_integration:
  layering_rule: "1) choose the best existing discovery_mode; 2) tune source search_terms and native filters from CV + prefs; 3) add provider filter support; 4) only then write dedicated provider logic"
  eval: "eval_source_quality.py: deterministic validators + LLM reviewer over the latest discovery for one source"
  validators: [artifact_nonempty, required_fields, no_empty_fields, url_allowlist, listing_kind, canary_present, detail_depth, sparse_extraction, result_volume, "source health (limitations mentioning blocked/captcha/timeout/unstable/failed)"]
  reviewer: "review_source_with_llm via the reviewer CLI; any error -> status blocked (never silently passes)"
  final_status: "pass | integration_needed (an integration_ticket is emitted, 02 §4) | blocked"
  strategies: [config_url_correction, config_terms_override, config_terms_append, config_native_filters, provider_filter_support, dedicated_provider_logic]
  loop: "source_integration.py --track T --source NAME --today D [--canary-title ...]: repeat eval -> (pass: exit | blocked: exit | needs code and attempts < max: run coder agent with a prompt built from the ticket + prior postmortem, rediscover) until pass, blocked or retry_limit"
  queue: "integrate_next_source.py --track T [--source] [--force] [--dry-run] [--max-attempts 2]: picks the next queued source from source_state.json by priority, at most one attempt per source per day unless --force"
  canaries: "update_source_canary.py stores a known-good title/URL per source in source_state; probe_career_source.py probes a candidate source URL"
  scope: "never triggered from scheduled runs; setup (explore-start) integrates at most the top 2 sources, queues the rest one per day"
  ends_at: "a working-tree fix; upstreaming is a normal fork-and-PR"
```

## Delivery

```yaml
delivery:
  email:
    script: "send_digest_email.py --track T --date D [--dry-run]"
    content: "digest_email.render_digest_email: subject from display name + new-role count; plain + HTML bodies with an executive summary, recommended actions with priority, new roles sorted by fit score (score color bands), source status counts; ranked overview as an attachment"
    transport: "SMTP; all four presets (gmail, fastmail, outlook_hotmail, proton_business) use STARTTLS (JOB_AGENT_SMTP_TLS can override); password from JOB_AGENT_SMTP_PASSWORD_CMD or the secrets file"
  telegram:
    script: "send_digest_telegram.py --track T --date D [--dry-run]"
    transport: "Bot API message send to JOB_AGENT_TELEGRAM_CHAT_ID; long digests split into '(part i/n)' chunks under the message limit; token from *_CMD or the secrets file; telegram_chat_id.py helps find the chat id"
  logseq:
    script: "sync_to_logseq.sh --track T"
    behavior: "copy tracks/T/digests/<today>.md to pages/'<Track Display> Job Digest <stamp>.md', copy the ranked overview page, and append '- New [[page]]' to journals/<YYYY_MM_DD>.md idempotently"
```

## Agent integration and guardrails

```yaml
agent_runtime:
  mode_router: "AGENTS.md routes every prompt to: track run | track setup (explore-start) | existing-track source curation (gather-curate-items) | repo development (engineering skill first) | non-interactive (prompt is the contract)"
  provider_selection: "agent_provider.py resolves JOB_AGENT_PROVIDER/JOB_AGENT_BIN and per-role binaries and flags"
  skill_mirroring: ".agents/skills/* is canonical; sync_claude_skills.sh copies it to .claude/skills/* with a GENERATED header; test.sh checks it"
  engineering_gate:
    claude: "PreToolUse hook (claude_engineering_gate.py) denies Edit/Write/NotebookEdit on tracked files until a session marker file shows the engineering skill was loaded; gitignored paths stay editable"
    gemini: "equivalent BeforeTool hook (gemini_engineering_gate.py)"
    codex: "install_codex_project_config.py writes project config"
    session_hook: "install_claude_session_hook.py"
  phase_protocol: "scripts/phases: Phase {name, inputs, outputs, run(PhaseContext)->PhaseResult}; only 'explore' exists (roadmap RM-018: /explore … /generate)"
```

## Capability toggles

```yaml
toggles:
  live_fetch: {enables: "feed_gather + enrich instead of fixtures/samples", default: "profile.toggles.live_fetch", switch: "--live"}
  enrich_opengraph: {enables: "OG image/description cache", default: on with live_fetch}
  feedback_rerank: {enables: "feedback deltas in audience_score", default: on, switch: "--with-feedback"}
  per_audience_digests: {enables: "rerank + audience digests + feedback", default: "on when the taxonomy declares audiences"}
  watcher_specific_stage: {enables: "<slug>_discover/_classify/_synthesize_digest", default: "present-by-filename"}
  llm_digest_for_ats_tracks: {enables: "agent phase in run_track.sh", default: on}
  browser_adapters: {enables: "*_browser modes", default: "only if playwright.sync_api importable"}
  delivery_email / delivery_telegram / delivery_logseq: {default: off, switch: "--delivery x"}
  scheduler: {default: off, switch: "install_scheduler.sh"}
  hosted_mode: {default: off, switch: "up --hosted"}
  backfill_window: {enables: "per-date historical fetch", switch: "--for-date"}
```

## Business rules and invariants

```yaml
rules:
  - "Daily loop is deterministic and keyless for the shipped watchers; identical inputs give identical artifacts except timestamps."
  - "Durable collection keeps every collected item, including low-score ones; the UI hides by threshold and never deletes."
  - "Item identity = <track>:<item_key>, where item_key hashes url|title (a title edit makes a new item)."
  - "Workspace revision increments on every portable mutation; publication requires an exact revision match."
  - "Exports never contain credentials, auth records, environment values, operation logs or claim tokens."
  - "Import always forks; there is no merge."
  - "One active operation per track (legacy) and one live run per app process (local app)."
  - "Sources degrade individually; a failed source never fails the run."
  - "Agents may edit tracked code only after loading the engineering skill (hook-enforced)."
```

## Open decisions and defects found

```yaml
open_decisions:
  - {id: two-runners, question: "run_pipeline.sh (feeds) and run_track.sh (ATS + LLM) are separate, and the scheduler and worker call only run_track.sh. Unify behind one phase runner?"}
  - {id: llm-classifier, question: "Classification is keyword-on-title; news_ai absorbed 87 of 105 topic_watch items on 2026-11-26. Add an LLM or embedding classifier (I5) with the keyword baseline as the bar to beat?"}
  - {id: worker-claim, question: "The PocketBase worker has no claim_token compare-and-set, so concurrent workers can double-run."}
  - {id: cancel, question: "operations.cancel_requested is never read by the worker."}
defects_observed:
  - "trends.cross_source_urls strips query strings, so all HN item links collapse into one URL."
  - "available_registry() raises ModuleNotFoundError when Playwright isn't installed: find_spec('playwright.sync_api') throws instead of returning None."
  - "Non-live topic_watch pipeline fails on a fresh checkout (missing tracks/topic_watch/sources.json)."
  - "Classifier rationale strings (e.g. 'class_hits=[]; ...') are shown to users as provenance."
  - "career_watch company falls back to the feed name for most HN/hnrss rows."
  - "capabilities.md claims 10/16/13 feeds for topic/market/career; the registries have 8/10/5."
```

## Module index (completeness walk)

Every tracked script maps to a section above. These are listed by name because they are otherwise referenced only by pattern:

```yaml
module_index:
  per_watcher_stages: "topic_watch_/market_watch_/career_watch_ {classify, synthesize_digest}.py -> pipeline classify / synthesize; topic_watch_taxonomy.py loads + validates the topic taxonomy"
  shared_helpers: "track_common.py (helpers shared by the per-watcher classifiers/synthesizers, e.g. iso_utc_now)"
  feedback: "track_feedback.py: append-only artifacts/feedback/<track>/<audience>/events.jsonl, actions save|hide|click|note, BOOST_PER_ACTION consumed by track_rerank --with-feedback (02 feedback_events)"
  legacy_operations: "portfolio_operations.py: bounded background subprocesses for serve_html /manage; states queued|running|needs_input|validating|first_run|ready|failed|cancelled; log capped at 64000 chars"
  discovery_internals: "discover/constants.py (timeouts 20s / browser 60s, MAX_BROWSER_PAGES 10, JD budget 4000 chars), discover/track_filters.py (match_rules.json keep_if_any_text_term), discover/__init__.py"
  phases: "phases/explore.py + phases/__init__.py (phase protocol above)"
  hooks: "hooks/{claude,gemini}_engineering_gate.py + *_mark.py + install_{claude,gemini}_engineering_gate_hook.py (03b mode router)"
  wrappers: "run_topic_watch_e2e.sh, run_market_watch_e2e.sh: back-compat wrappers for run_pipeline.sh --track <T>"
  launchd_template: "LaunchAgent plist template under scripts/, with a parameterized Label for the scheduler instance id (01 Mode P)"
  watcher_registries: "shared/schemas/<slug>_{taxonomy,source_registry}.json for topic/market/career are generated from the watcher specs (00 watchers, 02 schemas)"
```
