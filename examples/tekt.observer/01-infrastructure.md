# 01 — Infrastructure

Runtime, packaging, hosting and machine setup, described generically. Ports, paths and thresholds come from `SITE_PROFILE.runtime_defaults` in [00](./00-arkitype.md). Persistent formats are in [02](./02-database.md) and behavior is in [03](./03-software.md).

The system runs in three infrastructure modes. **Mode L** is the local app: one Python process serves a static React build and a JSON API over an immutable on-disk journal. **Mode P** is the pipeline: a deterministic CLI and cron/launchd batch that writes JSON artifacts to a filesystem root. **Mode H** is hosted: Docker Compose with unmodified PocketBase plus a Python worker. Mode H is scaffolded but not at parity (see `open_decisions`).

## Runtime and toolchain

```yaml
runtime:
  os_supported: [macOS, Linux]          # Windows explicitly unsupported (bash, fcntl, cron/launchd)
  python:
    version: ">=3.10 (observed working on 3.13.5)"
    venv: .venv/                         # created by scripts/bootstrap_venv.sh
    stdlib_only_core: true               # feed_gather, feed_enrich, classifiers, trends, rerank, journal, bundle, app server use only the stdlib
    dev_requirements: {pytest: "9.0.2", playwright: "1.58.0"}
    optional_runtime:
      playwright_chromium: "only for *_browser discovery modes (bootstrap_venv.sh installs it unless --no-chromium)"
      crawl4ai: "named as the target repeatable browser extractor; not imported by code at 42534a5"
      browser-use: "named for reconnaissance/repair; roadmap RM-019, not imported"
  node:
    version: ">=20 (observed working on 24.22.1)"
    package_manager: npm (package-lock.json committed)
    frontend_stack:
      react: "^19.1.1"
      react-dom: "^19.1.1"
      "@tanstack/react-query": "^5.85.5"
      pocketbase_js_sdk: "^0.26.2"       # declared, not yet used by src/
      lucide-react: "^0.541.0"
      clsx: "^2.1.1"
      tailwind-merge: "^3.3.1"
      dev: {vite: "^7.1.3", "@vitejs/plugin-react": "^5.0.2", typescript: "^5.9.2", tailwindcss: "^3.4.17", postcss: "^8.5.6", autoprefixer: "^10.4.21", vitest: "^3.2.4", jsdom: "^26.1.0", "@testing-library/react": "^16.3.0", "@testing-library/jest-dom": "^6.8.0"}
    scripts: {dev: "vite --host 127.0.0.1", build: "tsc -b && vite build", test: "vitest run --environment jsdom --globals"}
    build_output: frontend/dist/ (sourcemaps on)
  agent_clis:                            # only for setup, source integration and LLM digests of ATS tracks
    claude: "Claude Code CLI (-p --no-session-persistence --output-format stream-json)"
    codex: "Codex CLI (exec, -s workspace-write); on Ubuntu needs bwrap + AppArmor profile (scripts/install_bwrap_apparmor.sh, sudo)"
    gemini: "Gemini CLI (--output-format stream-json, approval-mode)"
  other_tools:
    rclone: "publish bundles: rclone copy --immutable (never rclone sync)"
    cron_or_launchd: scheduler backend
    caffeinate: "macOS only; run_track.sh re-execs itself under caffeinate -dimsu to stop sleep mid-run"
    docker_compose: "Mode H only"
```

## Mode L — local app (`tekt.observer up`)

```yaml
mode_L:
  entry: "./tekt.observer up [--host <profile.runtime_defaults.app_host>] [--port <profile.runtime_defaults.app_port>]"
  steps:
    - ensure_frontend: "if frontend/dist/index.html is missing or older than any of package.json, package-lock.json, vite.config.ts, src/**: npm install (if node_modules missing) then npm run build; error if npm missing"
    - start: "python scripts/app_server.py --host H --port P (store=state/, static=frontend/dist)"
  bind_rule: "app_server refuses any host other than 127.0.0.1, localhost, ::1"
  process_model: "ThreadingHTTPServer; background live runs on a daemon thread guarded by one process-wide lock"
  persistent_paths: {journal: "<store_root>/events|snapshots|CURRENT.json", exports: "<store_root>/exports/", run_logs: "<store_root>/run-logs/"}
  first_launch: "seeds workspace 'local', the watchers in profile.watchers, their sources, and samples (only if no live items), then forces a snapshot"
```

## Mode P — pipeline and scheduler

```yaml
mode_P:
  roots:
    repo_root: "checkout; tracked config + code"
    job_agent_root: "JOB_AGENT_ROOT (default repo root for run_track.sh; tests/tmp/<track> for run_pipeline.sh)"
  scratch_isolation: >-
    run_pipeline.sh copies scripts/ into the scratch root, symlinks shared/schemas and .venv, copies shared/digest_schema.md,
    track files (prefs.md, track.json, sources.json if present) and profile/personas, so runs never dirty the working tree.
    Without --append the scratch root is deleted first.
  scheduler:
    linux:
      mechanism: "per-checkout block in the user crontab, delimited by BEGIN/END comments tagged with the instance id; runs every minute"
      instance_id: "<lowercased-dir-basename, non-alnum->'-', max 48 chars>-<cksum of canonical root path>"
      install: "bash scripts/install_scheduler.sh (runs setup_machine.sh --quiet --no-logseq-prompt first; strips any previous block, appends new one)"
    macos:
      mechanism: "LaunchAgent plist in ~/Library/LaunchAgents, with a Label parameterized by the instance id, firing the same dispatcher"
    dispatcher: "scripts/run_scheduled_jobs.sh (see 03-software.md#scheduler)"
    machine_local_files: [.env.local, .schedule.local, .scheduler/state/*.stamp, .scheduler/run.lock/, logs/]
  bootstrap:
    - "bash scripts/bootstrap_venv.sh [--no-chromium]   # venv + dev requirements (+ playwright chromium)"
    - "bash scripts/bootstrap_machine.sh --agent {claude,codex,gemini}   # venv + setup_machine + hooks"
    - "bash scripts/setup_machine.sh --agent <name>   # writes JOB_AGENT_PROVIDER/JOB_AGENT_BIN, delivery config to .env.local; 883-line interactive helper"
    - "sudo bash scripts/install_bwrap_apparmor.sh   # Ubuntu + Codex only"
    - "bash scripts/start_setup_agent.sh --agent <name>   # launches the explore-start skill interactively"
```

## Mode H — hosted (scaffold)

Here is the full compose file as committed (`deploy/docker-compose.yml`):

```yaml
services:
  pocketbase:
    image: ghcr.io/muchobien/pocketbase:0.39.11
    command: ["serve", "--http=0.0.0.0:8090"]
    restart: unless-stopped
    ports:
      - "127.0.0.1:${TEKT_OBSERVER_PORT:-8090}:8090"
    volumes:
      - pocketbase_data:/pb_data
      - ../pocketbase/pb_migrations:/pb_migrations:ro
  worker:
    image: python:3.12-slim
    command: ["python", "scripts/tekt_observer.py", "worker", "--poll-seconds", "5"]
    restart: unless-stopped
    working_dir: /workspace
    environment:
      TEKT_OBSERVER_POCKETBASE_URL: http://pocketbase:8090
    volumes:
      - ..:/workspace:ro
      - worker_artifacts:/workspace/artifacts
    depends_on: [pocketbase]
volumes:
  pocketbase_data:
  worker_artifacts:
```

```yaml
mode_H:
  entry: "tekt.observer up --hosted   # prints a TLS/reverse-proxy warning, then docker compose -f deploy/docker-compose.yml up -d"
  pocketbase: {version_file: pocketbase/VERSION, version: 0.39.11, binary: unmodified upstream, migrations: pocketbase/pb_migrations (JS), topology: single-writer}
  requirements_documented: [TLS reverse proxy, PocketBase auth, persistent PocketBase + artifact volumes, backups]
  open_decisions:
    - "The worker mounts the repo read-only but run_track.sh writes tracks/<slug>/source_state.json, digests and shared/ files. Writes outside artifacts/ will fail."
    - "The python:3.12-slim image has no venv, Node, agent CLI or bash-tool deps that run_track.sh expects. No worker Dockerfile exists."
    - "The 'digests' collection exists in the journal but not in the PocketBase migration."
    - "The frontend talks to the app_server JSON API, not PocketBase, so hosted mode has no UI wired yet."
```

## Environment variables

These are names and purposes only. Values live in `.env.local` (gitignored) or a secrets file outside the repo (`JOB_AGENT_SECRETS_FILE`). Real secrets are better fetched by a `*_CMD` variable (a shell command that prints the secret, such as a keychain lookup).

```yaml
environment:
  tekt_observer:
    TEKT_OBSERVER_POCKETBASE_URL: "PocketBase base URL (default http://127.0.0.1:8090)"
    TEKT_OBSERVER_TOKEN: "PocketBase auth token for worker/import/export (secret)"
    TEKT_OBSERVER_PORT: "host port published for PocketBase in compose"
    TEKT_OBSERVER_STORE: "journal root (default <repo>/state)"
    TEKT_OBSERVER_COMPACT_EVERY: "events between snapshots (default 100)"
    TEKT_OBSERVER_COMPACT_SECONDS: "max snapshot age in seconds (default 300)"
  roots_and_files:
    JOB_AGENT_ROOT: "working root for pipeline/agents"
    JOB_AGENT_ENV_FILE: "override .env.local path"
    JOB_AGENT_SECRETS_FILE: "path to a secrets file outside the repo (sourced by load_runtime_env.sh)"
    JOB_AGENT_RUNTIME_SECRETS_FILE_LOADED: "internal guard so secrets load once"
    JOB_AGENT_PROFILE_DIR: "profile directory override"
    JOB_AGENT_PYTHON: "python override"
    JOB_AGENT_PLATFORM: "platform override for tests (linux|darwin)"
  agent:
    JOB_AGENT_PROVIDER: "claude | codex | gemini"
    JOB_AGENT_BIN: "absolute path to the agent CLI"
    JOB_AGENT_CODER_BIN / JOB_AGENT_REVIEWER_BIN: "separate CLIs for the source-integration coder and the LLM reviewer"
    JOB_AGENT_CLAUDE_PERMISSION_MODE: "default acceptEdits"
    JOB_AGENT_CLAUDE_SCHEDULED_ALLOWED_TOOLS: "default Read,Write,Edit,MultiEdit,Bash,Glob,Grep,LS,TodoWrite"
    JOB_AGENT_CLAUDE_CODER_ALLOWED_TOOLS / JOB_AGENT_CLAUDE_REVIEWER_ALLOWED_TOOLS: "tool allowlists per role"
    JOB_AGENT_GEMINI_APPROVAL_MODE / _SCHEDULED_ / _SETUP_ / _CODER_ / _REVIEWER_APPROVAL_MODE: "gemini approval per role (scheduled default yolo)"
    JOB_AGENT_GEMINI_SANDBOX / GEMINI_SANDBOX: "gemini sandbox flag (default false)"
    JOB_AGENT_GEMINI_SETUP_MODEL / JOB_AGENT_CODEX_SETUP_MODEL / JOB_AGENT_CODEX_SETUP_REASONING_EFFORT: "setup-session model choices"
    JOB_AGENT_CAFFEINATED: "internal re-exec guard (macOS)"
  run_context_passed_to_agents:
    [JOB_AGENT_TRACK, JOB_AGENT_TODAY, JOB_AGENT_DISCOVERY_ARTIFACT, JOB_AGENT_EVAL_ARTIFACT, JOB_AGENT_INTEGRATION_TICKET,
     JOB_AGENT_PRIOR_POSTMORTEM, JOB_AGENT_REDISCOVERY_ARTIFACT, JOB_AGENT_CANARY_URL, JOB_AGENT_CANARY_TITLE, JOB_AGENT_SOURCE, JOB_AGENT_JOURNAL_DATE]
  scheduler:
    [JOB_AGENT_SCHEDULE_FILE, JOB_AGENT_SCHEDULER_DIR, JOB_AGENT_SCHEDULER_STATE_DIR, JOB_AGENT_LAUNCH_AGENTS_DIR,
     JOB_AGENT_SCHEDULE_TIME, JOB_AGENT_SCHEDULE_STAMP, JOB_AGENT_SCHEDULE_DATE, JOB_AGENT_SCHEDULE_WEEKDAY, JOB_AGENT_SCHEDULE_MONTH_DAY]   # the last five are test clocks
  delivery:
    email: [JOB_AGENT_EMAIL_PROVIDER, JOB_AGENT_EMAIL_ACCOUNT, JOB_AGENT_SMTP_HOST, JOB_AGENT_SMTP_PORT, JOB_AGENT_SMTP_TLS, JOB_AGENT_SMTP_USERNAME,
            JOB_AGENT_SMTP_FROM, JOB_AGENT_SMTP_TO, JOB_AGENT_SMTP_PASSWORD (secret), JOB_AGENT_SMTP_PASSWORD_CMD (preferred)]
    telegram: [JOB_AGENT_TELEGRAM_CHAT_ID, JOB_AGENT_TELEGRAM_API_BASE, JOB_AGENT_TELEGRAM_BOT_TOKEN (secret), JOB_AGENT_TELEGRAM_BOT_TOKEN_CMD (preferred)]
    logseq: [LOGSEQ_GRAPH_DIR]
  hooks_and_sandbox:
    [GEMINI_PROJECT_DIR, GEMINI_HOOK_PYTHON, CODEX_CONFIG_PYTHON, JOB_AGENT_BWRAP_APPARMOR_SOURCE, JOB_AGENT_BWRAP_APPARMOR_DEST, JOB_AGENT_BWRAP_APPARMOR_REQUIRE_ROOT]
```

## CI and verification

```yaml
ci:
  hosted_ci: none                       # no .github/ directory at 42534a5
  local_gate: "bash scripts/test.sh"
  test_sh_runs:                         # in this order; set -euo pipefail
    - "bash -n on run_track, bootstrap_machine, setup_machine, install_bwrap_apparmor, install_scheduler, run_scheduled_jobs, sync_to_logseq, test_track_workflow, sync_claude_skills, tests/e2e/fake_codex.sh"
    - "python -m compileall -q scripts"
    - "bash scripts/sync_claude_skills.sh --check          # .claude/skills must mirror .agents/skills (+ GENERATED header)"
    - "render_discovery_modes_md.py --check               # shared/discovery_modes.md must match the registry"
    - "generate_watchers.py --check                        # generated watcher files must match .arkitype/watchers specs"
    - "if frontend/node_modules exists: npm --prefix frontend run build && npm --prefix frontend test"
    - "python -m pytest tests -q   # unit + contract + integration + e2e (fake_codex.sh / fake_gemini.sh agent doubles)"
  suite_size: "389 test functions across tests/; 556 passed + 28 skipped for unit+contract when run on 2026-11-26"
  fixtures: "tests/fixtures/sources/<mode>/ per adapter (contract tests replay saved HTML/JSON)"
```

## Open decisions

```yaml
open_decisions:
  - id: hosted-parity
    question: "Finish Mode H (PocketBase plus a worker image) or drop PocketBase and keep the journal as the only store?"
  - id: windows
    question: "fcntl locks, bash and cron make it POSIX-only. Is that acceptable for the rebuild?"
  - id: browser-runtime
    question: "Playwright (in use) vs Crawl4AI vs browser-use (named in specs but not wired)."
```
