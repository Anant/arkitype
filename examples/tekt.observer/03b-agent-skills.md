# 03b — Agent skills and the mode router (supplement to 03)

The agent half of tekt.observer is **six SKILL.md files** plus an `AGENTS.md` mode router. The source of truth is `.agents/skills/<name>/SKILL.md`. `scripts/sync_claude_skills.sh` mirrors each file into `.claude/skills/` and adds a `GENERATED FILE` header, so the two copies differ only by that header (observed with `diff`). `GEMINI.md` imports `@AGENTS.md`. `CLAUDE.md` imports `@AGENTS.md` plus `.arkitype/00`, `.arkitype/03`, `.knowledge/README.md` and `.manage/README.md`.

The skills are the most portable part of the repo. They are procedure, not code. Each entry below ends with a `keep_verdict`; [ROADMAP-DELTA](./ROADMAP-DELTA.md) collects the verdicts. Everything here is `verified: static-analysis` from reading the skills. The `track run` path was also observed once through the pipeline artifacts in [03](./03-software.md).

## Mode router (AGENTS.md)

```yaml
mode_router:
  modes:
    track_run:        {trigger: "non-interactive/harness launch, or prompt says run a track / produce a digest / names tracks/<t>/AGENTS.md", contract: "prompt first, then tracks/<t>/AGENTS.md", skills: [organize-filter-items, understand-prioritize-items]}
    track_setup:      {trigger: "create/scaffold/initialize/set up a track, or names explore-start", skills: [explore-start, explore-discover-sources]}
    source_curation:  {trigger: "add/evaluate/confirm/look up a NAMED employer or official source on an EXISTING track", skills: [gather-curate-items], escalate: "broad discovery, removals, cadence or search-term edits -> explore-start"}
    repo_development: {trigger: "anything else", skills: [engineering], enforced_by: "hooks/claude_engineering_gate.py (PreToolUse) and hooks/gemini_engineering_gate.py (BeforeTool) block edits until the engineering skill was loaded; claude_engineering_gate_mark.py records the load"}
    other_non_interactive: {rule: "prompt is the contract; follow literally; no clarifying questions"}
  ask_rule: "ask which mode only if genuinely ambiguous"
  shared_references: [".arkitype/00..04 (repo's own thin extraction)", ".arkitype/watchers/<track>/", ".manage/3.processes.md", ".manage/6.knowledge.md", ".manage/README.md (task docs -> .archive/<YYYY>/<name>.completed.md)", "shared/source_curation.md", "shared/discovery_modes.md"]
```

## Skills

```yaml
skills:
  - name: explore-start
    lines: 548
    role: "Interactive track setup wizard, end to end"
    inputs: [profile/cv.md, profile/prefs_global.md, "user brief", "optional PDF CV in profile/ (pdftotext)"]
    steps:
      0: "Profile readiness: run setup_machine.sh if profile files are missing; a JOB_AGENT_PROFILE_TEMPLATE marker means the file is still a default; draft cv.md from a single PDF; ask a single profile question"
      1: "Minimum prefs.md brief: display name, search area, goals, keep-only keywords, constraints, geography ('none yet' allowed)"
      2: "Ask for known companies and job boards"
      3: "Optional source discovery -> delegates to explore-discover-sources"
      4: "Normalize, probe, integrate sources: canary collection, targeted probing, quality triage; new code only as provider modules"
      5: "Generate files: tracks/<t>/{prefs.md, sources.json, match_rules.json (only if accepted), source_state.json, sources.md, AGENTS.md, CLAUDE.md, GEMINI.md, digests/}; never hand-write ranked_overview.md or shared/ranked_jobs/<t>.json; AGENTS.md template carries 'Check every run / every 3 runs / every month', search-term and filter sections, output discipline"
      6: "First local digest preview"
      7: "Delivery preferences + local config handholding (.env.local presets, *_CMD secret commands, schedule)"
      8: "Validation checklist: discover_jobs --list-sources, --plan-only --due-only, update_ranked_overview, per-source discover run, configure_schedule + install_scheduler, scripts/test.sh if shared code changed, eval_source_quality canary, start_source_integration / integrate_next_source --dry-run, run_track.sh without delivery, send_digest_email/telegram --dry-run"
      9: "Final response"
    interaction_style: "recommend -> confirm or override; delegation phrases ('suggest', 'go ahead') accept the remaining low-risk defaults; one recommended next step per transition"
    boundaries: ["never edit shared/templates/profile/*", "do not invoke source integration from scheduled runs", "read gitignored profile/tracks files with shell when the agent's file tools refuse"]
    keep_verdict: "KEEP as a people skill, generalized from 'job track' to 'observation track'. The recommend->confirm style and profile-readiness gate are the best UX in the repo."

  - name: explore-discover-sources
    lines: 194
    role: "Propose a short tiered source pack for a new track"
    precondition: "minimum brief exists (else return to explore-start)"
    outputs: "primary (about 4-8) + follow_up sources, one recommended package, setup-ready normalized records, suggested search terms, cadence (every_run rare / every_3_runs default / every_month for slow sources), match_rules suggestions only for broad/noisy sources"
    boundaries: ["does NOT write match_rules.json", "does not replace a strong existing list", "track config holds user logic; parsing lives in provider modules"]
    keep_verdict: "KEEP (merge with explore-start as a sub-step); generalize tiers beyond employers to any source type."

  - name: gather-curate-items
    lines: 79
    role: "Add or evaluate ONE named employer/official source on an existing track"
    steps: ["ground in the existing track", "resolve the official URL (homepage-linked careers/ATS first; surface corrections)", "normalize conservatively (default discovery_mode html)", "update sources.json + render_sources_md.py", "run the single-source integration workflow when support is weak"]
    keep_verdict: "KEEP as a people skill ('add a source'). Pair it with the source-integration loop as its automated back end."

  - name: organize-filter-items
    lines: 174
    role: "Track-run stage: enumerate and filter candidates from the scheduler's discovery artifact"
    contract: "artifact path from track instructions is the scheduler->agent handoff; today's artifact, else a fresh latest; a source counts as checked only if the artifact shows enumeration, terms, counts; never rerun discover_jobs.py in a scheduled run; never touch source_state.json"
    outputs: "candidate set + coverage notes ('not checked' with reason)"
    keep_verdict: "REBUILD as deterministic code. It is a filter with an evidence rule and does not need an LLM. Keep the coverage-honesty rule as a spec."

  - name: understand-prioritize-items
    lines: 123
    role: "Track-run stage: score candidates against CV + prefs"
    procedure: "holistic 1-10 score, fit reasons, concerns, recommendation apply_now|watch|skip, ordered strongest first"
    evidence_rule: "description (bounded JD body) is primary evidence; empty description -> say so in concerns; hedge with likely/appears"
    keep_verdict: "KEEP the evidence discipline as a reusable 'judge against a profile' skill; drop the job-specific vocabulary."

  - name: engineering
    lines: 123
    role: "Repo-development rules: public vs private files (git check-ignore), minimal diffs, scope control, verification, non-interactive pivot for unattended coder runs"
    enforced_by: "engineering-gate hooks"
    keep_verdict: "PATTERN ONLY. A later rebuild will have its own engineering conventions; the hook-gated 'load the rules before editing' idea is worth copying."
```

```yaml
open_decisions:
  - id: skill-generality
    question: "Skills hardcode job vocabulary (CV, employer, apply_now) while the shipped watchers are news/market/topic. Generalize or keep as a job vertical?"
    recommendation: "Generalize to observer/source/item; ship 'career' as one preset."
  - id: mirror-mechanism
    question: "Keep .agents -> .claude copy-with-header sync, or use symlinks / a single skills dir?"
    recommendation: "One canonical skills dir plus a generated mirror, with a CI check (the existing sync_claude_skills.sh --check)."
defects_observed:
  - "Skills reference profile/cv.md for every track, including news/topic watchers where a CV is meaningless."
  - "organize-filter-items and understand-prioritize-items describe the run_track.sh path only. The feed-watcher pipeline (03 pipeline) has no skill coverage."
```
