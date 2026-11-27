# 02 — Database: journal, projection, artifacts and config files

tekt.observer has **no single database**. It keeps three layers of persistent state, and a rebuild has to keep them distinct:

1. **Immutable JSON journal** (`state/`). This is the durable record behind the local app. Events are append-only, hash-chained, fsynced and compacted into immutable snapshots.
2. **PocketBase projection** (Mode H). It is the queryable, authorized and realtime copy of the same collections. It's scaffolded only; the migration is the schema of record.
3. **Filesystem pipeline artifacts and config** (`artifacts/`, `tracks/`, `shared/`, `profile/`). These are the outputs of each pipeline stage, plus per-track source config and state. They are the integration boundary between the pipeline and every UI.

The schema is identical for every instance built on this architecture. Only row data and the watcher definitions in [00 SITE_PROFILE](./00-arkitype.md) differ.

**Schema sources of record:** the code (`scripts/immutable_json_store.py`, `scripts/exchange_bundle.py`, `scripts/app_server.py`, `scripts/source_config.py`, `scripts/portfolio_state.py`), `pocketbase/pb_migrations/1755475200_initial_collections.js`, and `shared/digest_schema.md`. Shapes were **confirmed by running** the starter and live pipelines and the local app on 2026-11-26 (see [agent.local.md](./agent.local.md)).

## 1. Immutable JSON journal

```yaml
journal:
  root: "<TEKT_OBSERVER_STORE or <repo>/state>"
  layout:
    events/: "<sequence:020d>-<sha256(canonical event bytes)>.json   one event per file"
    snapshots/: "<sequence:020d>-<sha256(canonical snapshot bytes)>.json"
    CURRENT.json: "the only mutable file; atomic pointer cache {schema_version, sequence, snapshot, sha256}; never required for recovery"
    .writer.lock: "fcntl LOCK_EX for all appends and compactions"
    exports/: "bundle files written by the local app"
    run-logs/: "stdout+stderr of background live runs"
  canonical_bytes: "json.dumps(value, ensure_ascii=False, sort_keys=True, separators=(',', ':')) + '\\n', UTF-8"
  event:
    schema_version: 1
    sequence: "int, monotonically increasing from 1 under the writer lock"
    recorded_at: "UTC ISO-8601 with Z"
    previous_hash: "sha256 hex of the previous event file name hash, or null for the first"
    collection: "non-empty string"
    record_id: "non-empty string"
    operation: "put | delete"
    record: "object; present only on put (whole-record replace, no patches)"
  write_protocol:
    - "open(path, O_WRONLY|O_CREAT|O_EXCL, 0600); if it exists with identical bytes treat as success, else raise 'immutable file collision'"
    - "write, flush, fsync(file), fsync(directory)"
    - "append_many writes N events in one lock hold; each event's previous_hash = prior event hash"
    - "after every append_many, compact if due"
  snapshot:
    fields: {schema_version: 1, sequence: "last included event", created_at: "recorded_at of that event", state: "{collection: {record_id: record}}"}
    due_when: "events since last snapshot >= compact_every OR (any new events AND snapshot age >= compact_interval_seconds); with no snapshot yet, the age rule applies only when interval == 0"
    force: "store compact --force, and seed/ingest force a snapshot"
  read_protocol:
    - "pick the highest-sequence snapshot; verify sha256(file) == name hash, sequence == name, schema_version == 1"
    - "walk ALL event files in order, verifying hash, sequence, schema_version and the previous_hash chain (the full chain is verified on every read)"
    - "apply events with sequence > snapshot.sequence; put = replace record, delete = remove"
  default_collections: [workspaces, memberships, watchers, sources, runs, items, feedback, operations, exports, digests]
  retention: "nothing is ever deleted or overwritten; there is no pruning"
  known_limits:
    - "read() re-verifies every event on each call: O(total events) per request"
    - "a single global lock and log for all collections; not partitioned by tenant or source"
```

### Record shapes the local app writes (Mode L)

```yaml
local_records:
  workspaces: {id: local, name: "My observation workspace", revision: "int; +1 on every item status change and on each ingest"}
  watchers:  {id: "<slug>", slug: str, name: "display_name after the last ' · '", description: str, enabled: bool, status: "healthy|attention|paused", sourceCount: int}
  sources:   {id: "<watcher_slug>:<source_id>", watcher: slug, name: str, url: str, discoveryMode: "kind or discovery_mode", cadence: "cadence_group or every_run", status: "ready|attention|failed"}
  items:
    id: "<track>:<item_key>"
    watcher: slug
    title: str
    description: "og_description, else candidate description, else a readable sentence from classification (machine 'x_hits=' strings are suppressed)"
    url: str
    image: "og_image"
    siteName: "og_site_name or source_id"
    author: str
    topic: "topic | role_type | asset_class | 'general'"
    score: "0..100: digest fit_score*10 when the item is a digest top match, else max(40, confidence*100)"
    status: "new | saved | dismissed (preserved across re-ingest)"
    observedAt: "artifact generated_at or <date>T00:00:00Z"
    provenance: ["Live run · <date>", "Source · <source_id>", "<classifier rationale>"]
    sample: "bool; sample items are deleted when the first live ingest lands"
  runs:       {id: "<track>:<date>", watcher: slug, date: str, status: complete, itemCount: int, artifact: "path to organized JSON"}
  digests:    {id: "pipeline:<track>:<date> | digest-NNNN", watcher: "slug (pipeline digests only)", title: str, createdAt: str, summary: str, itemIds: [item ids], status: ready}
  operations: {id: "starter-run | live-run:<YYYYMMDDTHHMMSS> | live-import:<date>", kind: "live_tracks (optional)", label: str, status: "queued|running|complete|failed", progress: 0..100, updatedAt: str, itemCount: "int?", error: "str?", log: "path?"}
  exports:    {id: bundle_id, filename: "<bundle_id>.tekt-observer.json", createdAt: str, workspaceRevision: int, sha256: "of the written file"}
  memberships: "collection exists; never written in local mode (an implicit owner)"
  feedback:   "collection exists; the local app records triage as items.status instead"
```

## 2. PocketBase projection (Mode H)

The migration of record is `pocketbase/pb_migrations/1755475200_initial_collections.js`, which targets unmodified PocketBase 0.39.11. Every domain collection carries a `workspace` relation with `cascadeDelete: true`.

```yaml
pocketbase:
  rule_macros:
    memberRead:  "@request.auth.id != '' && workspace.memberships_via_workspace.user ?= @request.auth.id"
    editorWrite: "memberRead && (workspace.memberships_via_workspace.role ?= 'owner' || ... ?= 'editor')"
    ownerWrite:  "memberRead && workspace.memberships_via_workspace.role ?= 'owner'"
  collections:
    users:
      type: auth
      fields: {name: text, mode: "text, required"}
      indexes: ["idx_users_mode (mode)"]
    workspaces:
      fields: {name: "text req", slug: "text req", revision: "number req", definition: "json req"}
      indexes: ["UNIQUE idx_workspaces_slug (slug)"]
      rules: {list: "auth && memberships_via_workspace.user ?= auth.id", view: same}
    memberships:
      fields: {workspace: "relation->workspaces req", user: "relation->users req", role: "text req (owner|editor|viewer)"}
      indexes: ["UNIQUE (workspace, user)", "(workspace, role)"]
      rules: {list: "user = auth.id", view: "user = auth.id", create: owner-of-workspace, update: owner-of-workspace, delete: owner-of-workspace}
    watchers:
      fields: {workspace: rel, slug: "text req", name: "text req", enabled: bool, definition: "json req"}
      indexes: ["UNIQUE (workspace, slug)"]
      rules: {list: memberRead, view: memberRead, create: editorWrite, update: editorWrite, delete: ownerWrite}
    sources:
      fields: {workspace: rel, watcher: "rel->watchers req", source_key: "text req", name: "text req", url: "text req", discovery_mode: "text req", status: "text req", configuration: "json req", health: json}
      indexes: ["UNIQUE (watcher, source_key)"]
      rules: {list: memberRead, view: memberRead, create: editorWrite, update: editorWrite, delete: editorWrite}
    runs:
      fields: {workspace: rel, watcher: "rel req", run_key: "text req", status: "text req", started_at: text, finished_at: text, summary: json, provenance: json}
      indexes: ["UNIQUE (watcher, run_key)"]
      rules: {list: memberRead, view: memberRead, create: editorWrite, update: editorWrite, delete: ownerWrite}
    items:
      fields: {workspace: rel, watcher: "rel req", run: "rel optional", item_key: "text req", title: "text req", url: text, status: "text req", score: number, content: "json req", ranking: json, provenance: "json req"}
      indexes: ["UNIQUE (watcher, item_key)"]
      rules: {list: memberRead, view: memberRead, create: editorWrite, update: editorWrite, delete: editorWrite}
    feedback:
      fields: {workspace: rel, item: "rel->items req", user: "rel->users req", kind: "text req", note: text, context: json}
      indexes: ["(workspace, item)"]
      rules: {list: memberRead, view: memberRead, create: editorWrite, update: editorWrite, delete: editorWrite}
    operations:
      fields: {workspace: rel, watcher: "rel optional", kind: "text req", status: "text req", claim_token: text, claimed_at: text, attempt: "number req", max_attempts: "number req", cancel_requested: bool, input: json, progress: json, result: json, error: text}
      indexes: ["(status, created)"]
      rules: {list: memberRead, view: memberRead, create: ownerWrite, update: ownerWrite, delete: ownerWrite}
    exports:
      fields: {workspace: rel, bundle_id: "text req", workspace_revision: "number req", status: "text req", path: text, sha256: text, manifest: "json req", published_to: text, error: text}
      indexes: ["UNIQUE (workspace, bundle_id)"]
      rules: {list: memberRead, view: memberRead, create: ownerWrite, update: ownerWrite, delete: ownerWrite}
  down_migration: "delete exports, operations, feedback, items, runs, sources, watchers, memberships, workspaces, users (that order, ignoring errors)"
  roles:
    owner: [members, configuration, operations, imports, exports, read, curation]
    editor: [watchers, sources, runs, curation, read]
    viewer: [read, export]       # spec intent; the export collection's create rule is actually ownerWrite
    worker: "a trusted service credential (TEKT_OBSERVER_TOKEN)"
  open_decisions:
    - "No 'digests' collection, although the journal and the bundle both carry digests."
    - "The viewer-can-export intent conflicts with exports.createRule = ownerWrite."
    - "No projection job from the journal into PocketBase exists; the two stores aren't synchronized."
```

## 3. Exchange bundle (portable hand-off)

```yaml
exchange_bundle:
  file_name: "<bundle_id>.tekt-observer.json"
  encoding: canonical_bytes (sorted keys, compact separators, trailing newline)
  shape:
    manifest:
      bundle_id: "first 32 hex of sha256(canonical({workspace_id, workspace_revision, schema_version, hashes}))   # content-derived; created_at excluded"
      workspace_id: str
      workspace_revision: int
      schema_version: 1
      producer: "tekt.observer | tekt.observer.local"
      created_at: "UTC ISO Z"
      hashes: "{<collection>: sha256(canonical(data[collection]))} for every collection"
    data:
      collection_order: [workspace, watchers, sources, runs, items, feedback, digests, provenance]
      workspace: object
      others: "arrays of records (PocketBase transient fields collectionId, collectionName, expand, created, updated stripped)"
  validation:
    - "manifest and data are objects; schema_version == 1"
    - "data keys == exactly the eight collections"
    - "no secret-bearing key anywhere: token, password, passwordConfirm, credentials, environment, claim_token, or any key ending _secret, _password, _token (case-insensitive suffix)"
    - "every collection hash matches"
  write: "validate, then write <path>.tmp and rename"
  import: "validate; create workspace with slug=<fork-slug or '<slug>-import'> (refuse if slug exists); recreate records in collection order, remapping watcher/run/item relation ids; never merges"
  publish_guard: "manifest.workspace_revision must equal the current workspace revision, else 'stale export revision'"
  publish_transport: "rclone copy --immutable <bundle> <remote:path>"
```

## 4. Pipeline artifacts (filesystem, under a JOB_AGENT_ROOT)

```yaml
artifacts:
  discovery: # artifacts/discovery/<track>/<date>.json + latest.json
    schema_version: 1
    track: slug
    today: "YYYY-MM-DD"
    generated_at: "UTC Z"
    mode: discover
    sources:
      - source: name
        source_id: str
        source_url: str
        discovery_mode: "adapter mode or feed kind (rss|atom|hn_algolia)"
        cadence_group: "every_run|every_3_runs|every_month"
        last_checked: "date|null"
        due_today: bool
        status: "complete | partial | failed"      # partial = fetched but 0 items
        listing_pages_scanned: "int|str"
        search_terms_tried: [str]
        result_pages_scanned: str
        direct_job_pages_opened: int
        enumerated_jobs: int
        matched_jobs: int
        limitations: ["fetch_error:<msg>", "parse_error:<msg>", "unknown_kind:<k>", "window_filtered_out:<n>", "track filter removed ...", ...]
        filters: "{name: [values]}"
        candidates:
          - {employer: str, title: str, url: str, source_url: str, alternate_url: str, location: "str ('content' for feeds)", remote: "str", matched_terms: [str],
             notes: str, description: str, description_truncated: bool, published_at: "str? (feeds)", author: "str? (feeds)", topic_hints: "[str]? (feeds)",
             enrichment: "object? (added in place by enrich)"}
  enrichment: # artifacts/enrichment/<track>/urls.json — cache shared across dates
    "<url>": {og_image: str, og_title: str, og_description: str, og_site_name: str, og_type: str, canonical: str, published_time: str, author: str}
  organized: # artifacts/organized/<track>/<date>.json
    schema_version: 1
    track: slug
    date: "YYYY-MM-DD"
    generated_at: "UTC Z"
    items:
      common: {item_key: "<prefix>-<sha1(url|title)[:12]>; prefix ai|mw|jw by watcher", source_id: str, url: str, title: str, topic: str, content_type: str, categories: [str], audiences: [str], confidence: "0..1 (2dp)", rationale: str}
      market_watch_extra: {asset_class: str, event_type: str, watchlist_matches: [str], is_portfolio_alert: bool}
      career_watch_extra: {company: str, role_type: str, seniority: str, is_remote_friendly: bool}
  trends: # artifacts/trends/<track>/<date>.json
    {schema_version: 1, track: slug, date: str, generated_at: str, total_items: int,
     items_per_topic: [{topic, count}], items_per_source: [{source_id, count}], items_per_content_type: [{content_type, count}], items_per_audience: [{audience, count}],
     topic_velocity_vs_previous: [{topic, delta}], cross_source_urls: [{url, sources: [source_id]}], top_keywords: "[{token, count}] top 25"}
  ranked_audience: # artifacts/ranked_audience/<track>/<audience>/<date>.json
    {schema_version: 1, track: slug, audience: id, date: str, generated_at: str, total_items: int, selected_items: int, feedback_applied_items: int,
     ranked: "[organized item + {audience, audience_score 0..1 (3dp), selected: bool, rank: 1.., feedback_delta?}]"}
  digest: # artifacts/digests/<track>/<date>.json (+ latest.json); per-audience at artifacts/digests/<track>/<audience>/<date>.json (+ latest.json, top-level adds audience)
    schema_version: 1
    track: slug
    date: str
    runs:  # non-empty; first kind=initial; same-day reruns append kind=update
      - kind: "initial | update"
        generated_at: str
        executive_summary: str
        recommended_actions: [str]
        top_matches: "[{company*, title*, listing_url*, recommendation*: apply_now|watch|skip, job_key, alternate_url, location, remote, team_or_domain, posted_date, updated_date, source, source_url, fit_score: 0..10|null, why_match: [str], concerns: [str]}]"
        other_new_roles: "[{company*, title*, listing_url*, recommendation*, short_note*, job_key, alternate_url, location, source, fit_score}]"
        filtered_roles: "[{company*, title*, reason_filtered_out*, listing_url}]"
        source_notes: "[{source*, discovery_mode*, status*: complete|partial|failed, listing_pages_scanned, search_terms_tried, result_pages_summary, direct_job_pages_opened, limitations, note}]"
        notes_for_next_run: [str]
        discovery_artifacts: [path]
    note: "Job-shaped field names (company, listing_url, *_roles) are reused for news and market items; a rebuild should rename them to neutral terms."
  markdown_digests: ["tracks/<track>/digests/<date>.md", "tracks/<track>/digests/<audience>/<date>.md"]
  feedback_events: # artifacts/feedback/<track>/<audience>/events.jsonl (append-only)
    {track: slug, audience: id, item_key: str, action: "save|click|note|hide", ts: "UTC Z (default now)", note: "str?"}
  evals: # artifacts/evals/<track>/<source_slug>/
    "<date>.json": "eval result {deterministic checks, reviewer result, final_status: pass|integration_needed|blocked, integration_ticket?}"
    "<date>.source_integration_loop.json": "{phases, attempts, max_attempts, final_status: pass|blocked|retry_limit|running}"
    "<date>.discovery.json": "fresh discovery after a coder attempt"
    "<date>.attempt<N>.coder.stdout.jsonl | .stderr.log | .last_message.txt": "coder transcripts"
    "<date>.attempt<N>.postmortem.json": "failure analysis fed into the next attempt"
  integration_ticket:
    {status: open, track, source, discovery_mode, source_url, canary_title, canary_url, search_terms_tried, configured_filters,
     candidate_counts: {candidates, matched_jobs, enumerated_jobs}, page_evidence: {status, listing_pages_scanned, result_pages_scanned, direct_job_pages_opened, limitations},
     summary, defect_types, failing_checks, reviewer_defects, failure_mode, primary_evidence, target_outcome,
     suggested_strategy: "config_url_correction|config_terms_override|config_terms_append|config_native_filters|provider_filter_support|dedicated_provider_logic",
     suggested_strategy_label, config_suggestion, test_hint, likely_file, success_condition, non_goals}
  html_site: "site/ (render_html.py) or a multitrack folder (render_multitrack_site.py); layout in 04-interface.md"
```

## 5. Per-track config and state (`tracks/<slug>/`, `shared/`, `profile/`)

```yaml
track_files:
  generated_from_watcher_spec:   # do not hand-edit; generate_watchers.py stamps the owning spec
    tracks/<slug>/track.json: {schema_version: 1, id, display_name, description, status: "draft|active|archived", interest_ids: [], default_audience, interest_topic_mappings: {}, generated_from: path}
    tracks/<slug>/prefs.md: "<!-- Generated from .arkitype/watchers/<slug>/brief.md; do not edit directly. --> + brief"
    shared/schemas/<slug>_taxonomy.json: "taxonomy + source: <spec path>"
    shared/schemas/<slug>_source_registry.json: "{sources: [{id, name, kind: rss|atom|hn_algolia, url, topic_hints: [taxonomy ids]}]}"
  ats_style_track:   # private tracks created by explore-start (e.g. tracks/test_workflow)
    sources.json:
      schema_version: 1
      track: slug
      track_terms: [str]
      sources:
        - {id: "slug, unique (slugify_source_id + _2.. suffix)", name: str, url: str, discovery_mode: "one of the 60 modes", cadence_group: "every_run|every_3_runs|every_month",
           filters: "{name: [str]} (provider-native)", search_terms: "{terms: [str], mode: append|override}?"}
    source_state.json: {schema_version: 1, track: slug, sources: "{<source_id>: {last_checked: 'YYYY-MM-DD'|null, canary?, integration queue fields?...}}"}
    source_state_write_rule: "every writer goes through mutate_source_state: fcntl lock, reload, mutate, ensure all ids present, atomic write"
    sources.md: "generated by render_sources_md.py, grouped by cadence"
    match_rules.json: {schema_version: 1, track: slug, rules: "[{id, source_ids, source_names, keep_if_any_text_term: [str], limitation: 'Track filter removed {removed} candidate(s)...'}]"}
    prefs.md: "free-text brief (from shared/templates/track_prefs.md)"
    AGENTS.md / CLAUDE.md / GEMINI.md: "per-track run instructions (templates in shared/templates/)"
    seen_jobs.json: {schema_version: 1, track: slug, jobs: "[{date_seen, company, title, location, url}]"}
    ranked_overview.md: "rolling ranked list rebuilt from digests/*.md by update_ranked_overview.py"
    digests/: "markdown digests"
  user_overlays:   # gitignored, created only by 'portfolio_state.py init' or the UI
    profile/interests.json: {schema_version: 1, interests: "[{id: ^[a-z][a-z0-9_-]{0,63}$, label, description, keywords: [str]}]"}
    profile/portfolios.json: {schema_version: 1, default_portfolio_id: id, portfolios: "[{id, name, track_ids, interest_ids, archived: bool}]"}
    tracks/<slug>/taxonomy.json: "{schema_version: 1, track: slug, topics/audiences label+description overrides layered over shipped taxonomy}"
    profile/cv.md, profile/prefs_global.md, profile/personas/: "candidate profile for career tracks (templates in shared/templates/profile/)"
  machine_local:
    .env.local: "KEY=VALUE exports (provider, delivery config); gitignored"
    .schedule.local: "lines: 'daily HH:MM track <slug> [--delivery x]...' | 'weekly <mon..sun> HH:MM track ...' | 'monthly <1-31> HH:MM track ...'; '#' comments"
    .scheduler/state/<key>.stamp: "last run stamp YYYY-MM-DD-HH:MM per entry"
    logs/portfolio-operations.json: "{operations: [{id: uuid hex, track, kind: setup|run|validate_sources|schedule, state: queued|running|needs_input|validating|first_run|ready|failed|cancelled, created_at, updated_at, log (<=64000 chars), returncode}]} capped at 200"
```

## 6. Seed data

```yaml
seed:
  watcher_specs: "three watchers, 23 keyless sources and 9 sample items; full content in 00 SITE_PROFILE.watchers"
  local_app_seed: "workspace 'local' rev 1; one watcher record per spec; one source record per spec source; samples scored max(72, 94 - 5*index); operation 'starter-run' complete"
  starter_workspace: "seed_starter_workspace.py writes organized artifacts for each watcher from samples.json into tests/tmp/starter-workflows"
  ats_example: "tracks/test_workflow: one 'html' source at http://127.0.0.1:18765/test_workflow_board.html, track_terms [cryptography]; used by e2e tests"
  frontend_demo_state: "frontend/src/demo-data.ts: 9 starter signals shown when /api/v1/workspace fails"
```

## Open decisions

```yaml
open_decisions:
  - id: store-of-record
    question: "Journal vs PocketBase vs filesystem artifacts. Three stores overlap today; no projector keeps them in sync."
  - id: neutral-digest-schema
    question: "Rename job-centric digest fields (company, listing_url, other_new_roles) for general observation?"
  - id: journal-scale
    question: "Partition the log per tenant/source and add an index; every read currently verifies every event."
  - id: doc-drift
    question: "architecture.md says shared/seen_jobs.md, but the code writes tracks/<slug>/seen_jobs.json. Trust the code."
```
