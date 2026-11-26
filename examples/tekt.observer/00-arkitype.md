# 00 — Arkitype: tekt.observer (as built at xingh/tekt.observer@42534a5)

This is the identity and site profile for **tekt.observer**, reverse-engineered from the public repo `xingh/tekt.observer` at commit `42534a5` (2026-09-15). Everything specific to *this* instance lives in the single `SITE_PROFILE` block below: names, brand tokens, ports, the three shipped watchers with their full taxonomies, source registries and starter samples, and the keyword lexicons the deterministic classifiers use.

Files [01](./01-infrastructure.md) to [04](./04-interface.md) describe the architecture in generic terms and read their per-instance values from this profile. They also lean on two supplements: [03a — Source adapters](./03a-source-adapters.md) and [03b — Agent skills](./03b-agent-skills.md). To create a sibling observer (say, a "competitor_watch" or a public-intelligence pack), swap this file and keep 01–04.

> Source repo license: MIT (see `xingh/tekt.observer`). This document describes that system so it can be rebuilt; it does not copy source code. See [ROADMAP-DELTA.md § License](./ROADMAP-DELTA.md#license-and-attribution).

## How to read the profile

- `identity`: names, positioning, and the naming rule (`tekt` is reserved, so the product is always written `tekt.observer`).
- `brand`: the tokens for the React app and for the legacy server-rendered viewer. They are two separate systems; see [04](./04-interface.md#design-system).
- `runtime_defaults`: ports, paths and thresholds that the generic layers reference by name (for example `profile.runtime_defaults.app_port`).
- `toggles`: the default state of the capability toggles defined in [03](./03-software.md#capability-toggles).
- `watchers`: the canonical watcher specs. Upstream these live in `.arkitype/watchers/<slug>/{watcher.json,brief.md,taxonomy.json,sources.json,samples.json}`, and `generate_watchers.py` emits `tracks/<slug>/{track.json,prefs.md}` and `shared/schemas/<slug>_{taxonomy,source_registry}.json` from them.
- `classifier_lexicons`: values hardcoded in the upstream Python classifiers (`*_classify.py`). They are lifted here because they are per-instance content, not architecture.
- `constellation`: how tekt.observer sits next to a later rebuild (installer, desktop index, knowledge graph) without naming those projects.

```yaml
SITE_PROFILE:
  identity:
    site_name: tekt.observer
    executable: tekt.observer            # POSIX shell shim -> .venv/bin/python scripts/tekt_observer.py
    naming_rule: "Always 'tekt.observer'. The short name 'tekt' alone is reserved for a separate utility-belt product."
    license: "MIT (see xingh/tekt.observer)"
    fork: {repo: xingh/tekt.observer, commit: 42534a5, commit_date: "2026-09-15", visibility: public}
    tagline: "Watch the sources you care about. Get one briefing a day that's actually worth reading."
    positioning: >-
      Watches primary sources directly, classifies what it finds against a taxonomy you declare, ranks per audience,
      and hands you a browsable daily report. No aggregator deciding what matters; no LLM in the daily loop.
    audience: [CLI-comfortable individuals, analysts, job seekers, investors, managers tracking AI adoption]
    not_a_fit: [users who will not use a CLI, Windows (macOS and Linux supported)]
    method_name: "tekt.observer flow"
    phases:                              # the six-phase pattern the repo implements (README "How it fits")
      - {phase: explore,    meaning: "understand a new source and generate a script to crawl it", implemented_by: [explore-discover-sources skill, probe_career_source.py]}
      - {phase: seek,       meaning: "run that script against known sources",                   implemented_by: [discover_jobs.py, discover/sources/*]}
      - {phase: gather,     meaning: "schedule fetching and collect it in one place",            implemented_by: [feed_gather.py, scheduler]}
      - {phase: organize,   meaning: "classify, categorize, annotate, relate, index",            implemented_by: ["<watcher>_classify.py", track_trends.py]}
      - {phase: understand, meaning: "rerank against a profile, prioritize for impact",          implemented_by: [track_rerank.py]}
      - {phase: generate,   meaning: "produce a digest and a brief on how to act on it",         implemented_by: [synthesize_audience_digests.py, "<watcher>_synthesize_digest.py", render_digest.py, render_html.py]}
    core_entities: [workspace, membership, watcher, source, run, item, feedback, operation, export, digest]
    non_goals:
      - bidirectional cross-instance synchronization (exchange bundles are one-way, fork-on-import)
      - custom PocketBase Go extensions or JS hooks
      - FastAPI or any custom server framework for the hosted mode
      - local model hosting
      - LLM calls in the daily scheduled loop for the three shipped watchers
  brand:
    logo: "README header uses a wordmark image under .knowledge/images/ (height 26, right-aligned)"
    app_mark: {text: "t.", shape: rounded-square 28px, colors: "bg ink / text canvas"}
    react_app:
      fonts: {sans: Inter (400/500/600), serif_display: "Newsreader (opsz 6..72, 500/600)", fallback_serif: Iowan Old Style}
      tokens_hsl:
        light: {canvas: "42 29% 97%", surface: "40 33% 99%", ink: "30 10% 15%", muted: "30 7% 47%", line: "35 16% 87%", accent: "19 67% 48%"}
        dark:  {canvas: "32 10% 10%", surface: "30 9% 13%", ink: "40 20% 91%", muted: "35 8% 62%", line: "30 8% 23%", accent: "24 73% 60%"}
      theme_color_meta: "#f7f5f1"
      shadow_soft: "0 1px 2px rgba(27,25,22,.04), 0 10px 35px rgba(27,25,22,.035)"
      character: "calm, keyboard-friendly, document-centered; restrained neutrals with one warm accent"
      prohibition: "do not copy third-party branding or assets"
    legacy_viewer:
      tokens:
        light: {fg: "#1a1a1a", muted: "#666", bg: "#fdfdfd", card: "#fff", border: "#e2e2e2", accent: "#2e5cff", code_bg: "#f6f6f6"}
        dark:  {fg: "#e8e8e8", muted: "#999", bg: "#121212", card: "#1c1c1c", border: "#333"}
      font: system-ui sans
  runtime_defaults:
    app_host: 127.0.0.1
    app_port: 8091                       # React app + journal API (tekt.observer up)
    legacy_viewer_port: 8765             # serve_html.py
    pocketbase_port: 8090                # hosted mode
    vite_dev_port: 4173                  # proxies /api -> app_port
    store_root: state/                   # immutable journal
    compact_every_events: 100
    compact_every_seconds: 300
    scratch_root: "tests/tmp/<track>"    # run_pipeline.sh default JOB_AGENT_ROOT
    starter_scratch: tests/tmp/starter-workflows
    feed_max_items_per_source: 20
    feed_fetch_timeout_s: 15
    feed_workers: 6
    enrich_max_urls: 60
    enrich_timeout_s: 12
    enrich_max_bytes: 200000
    discovery_timeout_s: 20
    browser_timeout_ms: 60000
    browser_max_pages: 10
    jd_description_char_budget: 4000
    jd_fetch_wall_clock_budget_s: 60
    jd_fetch_hard_ceiling: 200
    feedback_boosts: {save: 0.20, note: 0.10, click: 0.05, hide: -0.35}
    rerank_boost: {selected_audience: 1.5, other_audience: 0.55, portfolio_alert_bonus: 0.25, portfolio_alert_audiences: [investors, portfolio_managers, allocators]}
    inbox_default_visibility_percentile: 75     # "top quarter by default"
    priority_bands: {top_priority: ">=80", worth_review: "60-79", low_confidence: "<60"}
    max_api_body_bytes: 65536
    max_import_body_bytes: 20971520
    legacy_max_write_body_bytes: 262144
    legacy_operation_log_cap: 200
    user_agent: "tekt.observer/<component> (github.com/xingh/tekt.observer)"   # update on any repo move
  toggles:                               # defaults; definitions in 03-software.md#capability-toggles
    live_fetch: false                    # run_pipeline.sh without --live uses fixtures/samples
    enrich_opengraph: true               # whenever live_fetch
    feedback_rerank: true                # track_rerank --with-feedback
    per_audience_digests: true           # any taxonomy with audiences
    llm_digest_for_ats_tracks: true      # run_track.sh agent phase
    browser_adapters: false              # needs Playwright + Chromium
    delivery: {email: false, telegram: false, logseq: false}
    scheduler: false
    hosted_mode: false                   # docker compose PocketBase + worker
    engineering_gate_hooks: false        # installed per agent by setup
  agent_providers: [claude, codex, gemini]
  delivery_presets:
    email_smtp: [gmail, fastmail, outlook_hotmail, proton_business]
  constellation:
    family: tekt
    role: "observation half of the toolkit: public-source watcher, local journal, Catalog Viewer candidate"
    siblings:
      - {role: "utility-belt installer + catalog", relation: "an installer can place agents; tekt.observer runs on them"}
      - {role: "spec method", relation: "the arkitype extract method used for this document and for the in-repo .arkitype/ folder"}
      - {role: "desktop index companion", relation: "proposed shell for a later replicator (see ROADMAP-DELTA)"}
      - {role: "knowledge graph", relation: "a later plan may write items into a graph store"}
      - {role: "alternate public-intelligence engine", relation: "same problem space, different engine; out of scope here"}
    handoff_destination_example: "an rclone remote name used in the source docs for bundle publication"
    license_rule: "Source repo license: MIT (see xingh/tekt.observer). Follow that license if any code is reused."
  watchers:
    topic_watch:
      watcher:
        id: topic_watch
        display_name: Topicwatch · AI in Business
        description: AI adoption, productivity, governance, customer operations, and operating models for business teams.
        status: active
        order: 10
        default_audience: managers
        interest_ids: []
        interest_topic_mappings: {}
      brief_md: |
        # Topicwatch · AI in Business

        Watch practical uses of AI in business: adoption, team productivity, customer and back-office operations, governance, risk, operating-model changes, and evidence of measurable value.

        Prioritize concrete deployments, workflow redesign, adoption lessons, controls, cost or productivity evidence, and implications for managers. De-emphasize benchmark-only model news unless it changes what a business can deploy.

        The generated starter uses the keyless source registry declared beside this spec. Create a private track with the guided setup flow when you are ready to narrow the question.
      taxonomy:
        topics:
        - id: ai_knowledge_memory
          label: AI Knowledge and Memory
          facets: [knowledge_storage, knowledge_graph, context_graph]
        - id: ai_capabilities
          label: AI Capabilities
          facets: [vision, document_vision, document_formulate, agent_fleet_management]
        - id: open_source_ai
          label: Open Source AI
          facets: [applications, harnesses]
        - id: models_intelligence_inference
          label: Models / Intelligence Inference
          facets: [open, local, cloud]
        - id: models_inference_strategy
          label: Models / Inference Strategy
          facets: [open, fusion, edge, cloud]
        - id: models_family
          label: Models
          facets: [open_source, foundation, frontier]
        - id: enterprise_ai_adoption
          label: Enterprise AI Adoption
          facets: [operating_model, change_management, value_measurement, deployment]
        - id: ai_productivity_workflows
          label: AI Productivity and Workflows
          facets: [knowledge_work, automation, copilots, workflow_redesign]
        - id: ai_governance_risk
          label: AI Governance and Risk
          facets: [policy, compliance, security, responsible_ai]
        - id: ai_customer_operations
          label: AI in Customer Operations
          facets: [sales, service, support, marketing]
        - id: news_ai
          label: News / AI
          facets: []
        content_types:
        - id: post
          label: Post
          category_groups: [topics, post_video_podcast_book_categories]
        - id: video
          label: Video
          category_groups: [topics, post_video_podcast_book_categories]
        - id: podcast
          label: Podcast
          category_groups: [topics, post_video_podcast_book_categories]
        - id: book
          label: Book
          category_groups: [topics, book_paper_categories]
        - id: resource
          label: Resource
          category_groups: [topics, resource_categories]
        - id: paper
          label: Paper
          category_groups: [topics, book_paper_categories]
        category_groups:
          post_video_podcast_book_categories:
            derived_from_topics: true
            extra: []
          book_paper_categories:
            derived_from_topics: true
            extra: [energy, policy, machine_learning_ai, quantum, networking]
          resource_categories:
            derived_from_topics: true
            extra: [skills, tools, workflows, connectors, agents, patterns, architectures, applications, harnesses, fleets, company_harness, department_harness,
              team_harness, knowledge, templates, news]
        audiences:
        - id: builders
          label: Builders
          lens: hands-on implementation, code, prototypes
        - id: operators
          label: Operators
          lens: reliability, on-call, cost, throughput
        - id: managers
          label: Managers
          lens: team throughput, roadmap alignment, hiring
        - id: architects
          label: Architects
          lens: system shape, tradeoffs, invariants across layers
        - id: leaders
          label: Leaders
          lens: strategy, portfolio bets, external narrative
      sources:
      - id: omitted_personal_author_feed_1
        kind: atom
        topic_hints: [ai_capabilities, open_source_ai, ai_knowledge_memory]
        note: "Present in the xingh/tekt.observer topic_watch registry. Id, display name, and URL omitted because they contain a personal name."
      - id: omitted_personal_author_feed_2
        kind: rss
        topic_hints: [models_family, ai_capabilities]
        note: "Present in the xingh/tekt.observer topic_watch registry. Id, display name, and URL omitted because they contain a personal name."
      - id: latent_space_rss
        name: Latent Space
        kind: rss
        url: https://www.latent.space/feed
        topic_hints: [ai_capabilities, models_intelligence_inference, open_source_ai]
      - id: arxiv_cs_lg_rss
        name: arXiv cs.LG (Machine Learning)
        kind: rss
        url: http://export.arxiv.org/rss/cs.LG
        topic_hints: [models_family, ai_capabilities]
      - id: arxiv_cs_cl_rss
        name: arXiv cs.CL (Computation and Language)
        kind: rss
        url: http://export.arxiv.org/rss/cs.CL
        topic_hints: [models_family, ai_knowledge_memory]
      - id: lesswrong_rss
        name: LessWrong
        kind: rss
        url: https://www.lesswrong.com/feed.xml?view=curated-rss
        topic_hints: [news_ai, ai_capabilities]
      - id: hn_algolia_enterprise_ai
        name: Hacker News (enterprise AI adoption)
        kind: hn_algolia
        url: https://hn.algolia.com/api/v1/search_by_date?tags=story&query=enterprise%20AI%20adoption&hitsPerPage=20
        topic_hints: [enterprise_ai_adoption]
      - id: hn_algolia_ai_governance
        name: Hacker News (AI governance and risk)
        kind: hn_algolia
        url: https://hn.algolia.com/api/v1/search_by_date?tags=story&query=AI%20governance%20risk&hitsPerPage=20
        topic_hints: [ai_governance_risk]
      samples:
      - item_key: sample-enterprise-adoption
        title: 'Starter brief: measuring enterprise AI adoption beyond pilot counts'
        url: https://example.com/tekt-observer/enterprise-ai-adoption
        topic: enterprise_ai_adoption
        content_type: post
        audiences: [managers, leaders]
      - item_key: sample-customer-operations
        title: 'Starter brief: redesigning customer-support workflows around AI'
        url: https://example.com/tekt-observer/ai-customer-operations
        topic: ai_customer_operations
        content_type: post
        audiences: [operators, managers]
      - item_key: sample-governance-operating-model
        title: 'Starter brief: an operating model for AI governance and accountable deployment'
        url: https://example.com/tekt-observer/ai-governance-operating-model
        topic: ai_governance_risk
        content_type: resource
        audiences: [architects, leaders, managers]
    career_watch:
      watcher:
        id: career_watch
        display_name: Careerwatch · Career opportunities & intelligence
        description: Roles, professional shifts, company signals, skills, and news that can change a person's career decisions.
        status: active
        order: 20
        default_audience: senior_ic
        interest_ids: []
        interest_topic_mappings: {}
      brief_md: |
        # Careerwatch · Career opportunities & intelligence

        Watch professions that build, govern, sell, teach, or operationalize AI: AI and ML engineering, solutions architecture, product management and design, governance and risk, operations and automation, developer relations, education, and customer-facing technical roles.

        Also watch career-relevant news: meaningful company changes, skill demand, labor-market shifts, professional events, and practitioner developments that could affect what the person should learn, pursue, or avoid. Label these separately from open roles.

        Prioritize jobs where AI is part of the actual responsibilities or product surface, not a generic skill keyword added to an otherwise unrelated listing.

        The generated starter uses the keyless source registry declared beside this spec. Use guided setup to replace the broad example with employers, locations, seniority, role preferences, and career-intelligence sources matched to you.
      taxonomy:
        role_types:
        - id: ai_engineer
          label: AI Engineer
          keywords: [ai engineer, llm engineer, agent engineer, rag, retrieval augmented, orchestration, llm ops]
        - id: prompt_engineer
          label: Prompt Engineer
          keywords: [prompt engineer, prompt engineering]
        - id: ml_engineer
          label: ML Engineer
          keywords: [ml engineer, machine learning engineer, mle, training pipeline, model serving, feature engineering]
        - id: ai_researcher
          label: AI Researcher
          keywords: [ai researcher, research scientist, research engineer, applied research, alignment researcher]
        - id: ai_instructor
          label: AI Instructor / Trainer
          keywords: [ai instructor, ai trainer, curriculum, workshop, cohort course, teach ai, instructor]
        - id: ai_devrel
          label: AI Developer Advocate
          keywords: [developer advocate, devrel, community engineer, developer relations, developer experience]
        - id: ai_pm
          label: AI Product Manager
          keywords: [ai product manager, ai pm, product lead ai]
        - id: ai_solutions_architect
          label: AI Solutions Architect
          keywords: [ai solutions architect, genai architect, ai architect, llm solutions architect, ai consultant]
        - id: ai_product_designer
          label: AI Product Designer
          keywords: [ai product designer, ai ux, conversational designer, ai interaction designer]
        - id: ai_governance_risk
          label: AI Governance / Risk
          keywords: [ai governance, responsible ai, ai risk, model risk, ai compliance, ai safety policy]
        - id: ai_operations_automation
          label: AI Operations / Automation
          keywords: [ai operations, ai automation, automation lead, ai transformation, ai enablement]
        - id: ai_customer_success
          label: AI Customer Success / Sales Engineering
          keywords: [ai customer success, ai sales engineer, ai solutions engineer, genai specialist, ai account executive]
        - id: ai_augmented_generalist
          label: AI-augmented Generalist SWE
          keywords: [staff engineer, senior engineer, principal engineer, backend engineer, full stack, generalist]
        audiences:
        - id: individual_contributor
          label: Individual contributor
          lens: shipping, building, learning
        - id: senior_ic
          label: Senior IC / Staff
          lens: architecture reach, tech-lead influence without formal mgmt
        - id: tech_lead
          label: Tech Lead
          lens: small team leadership + hands-on code, roadmap responsibility
        - id: manager
          label: Manager
          lens: people leadership, hiring, cross-team collaboration
        - id: instructor
          label: Instructor / Educator
          lens: teaching, curriculum, live delivery
        seniority_keywords:
          individual_contributor: [junior, mid-level, entry]
          senior_ic: [senior, staff, principal, lead engineer, distinguished]
          tech_lead: [tech lead, engineering lead]
          manager: [manager, director, head of, vp]
          instructor: [instructor, trainer, curriculum lead]
        remote_keywords: [remote, distributed, anywhere, work from home, wfh, remote-first, remote-friendly]
      sources:
      - id: hnrss_jobs
        name: Hacker News — Jobs firehose (hnrss)
        kind: rss
        url: https://hnrss.org/jobs
        topic_hints: [ai_augmented_generalist]
      - id: hn_algolia_ai_engineer
        name: Hacker News (AI Engineer)
        kind: hn_algolia
        url: https://hn.algolia.com/api/v1/search_by_date?tags=story&query=AI%20Engineer%20hiring&hitsPerPage=20
        topic_hints: [ai_engineer]
      - id: ai_jobs_net
        name: ai-jobs.net
        kind: rss
        url: https://ai-jobs.net/feed/
        topic_hints: [ai_engineer, ml_engineer, ai_researcher]
      - id: wwr_programming
        name: We Work Remotely — Programming
        kind: rss
        url: https://weworkremotely.com/categories/remote-programming-jobs.rss
        topic_hints: [ai_engineer, ml_engineer, ai_augmented_generalist]
      - id: hn_algolia_ai_customer_jobs
        name: Hacker News (AI customer success and sales engineering)
        kind: hn_algolia
        url: https://hn.algolia.com/api/v1/search_by_date?tags=story&query=AI%20solutions%20engineer%20customer%20success%20hiring&hitsPerPage=20
        topic_hints: [ai_customer_success]
      samples:
      - item_key: sample-ai-platform-engineer
        title: Senior AI Platform Engineer
        url: https://example.com/tekt-observer/ai-platform-engineer
        topic: ai_engineer
        content_type: posting
        audiences: [senior_ic, tech_lead]
      - item_key: sample-ai-governance
        title: AI Governance and Model Risk Lead
        url: https://example.com/tekt-observer/ai-governance-risk-lead
        topic: ai_governance_risk
        content_type: posting
        audiences: [senior_ic, manager]
      - item_key: sample-ai-solutions-architect
        title: AI Solutions Architect
        url: https://example.com/tekt-observer/ai-solutions-architect
        topic: ai_solutions_architect
        content_type: posting
        audiences: [senior_ic, tech_lead]
    market_watch:
      watcher:
        id: market_watch
        display_name: Marketwatch · AI Markets & Regulation
        description: AI-related public companies, semiconductors, cloud platforms, regulation, and market-moving policy.
        status: active
        order: 30
        default_audience: investors
        interest_ids: []
        interest_topic_mappings: {}
      brief_md: |
        # Marketwatch · AI Markets & Regulation

        Watch public companies exposed to AI compute, cloud platforms, enterprise software, and AI applications. Track earnings, guidance, capital expenditure, launches, customer adoption, supply constraints, and material partnerships alongside AI regulation, antitrust, copyright, safety rules, and semiconductor export controls.

        Starter public-company coverage includes NVIDIA, AMD, Microsoft, Alphabet, Amazon, Meta, Broadcom, TSMC, ASML, Arm, Oracle, Salesforce, Palantir, Snowflake, Micron, and Super Micro Computer. This is an observation list, not investment advice.

        The generated starter uses the keyless source registry declared beside this spec and the classifier's sample watchlist. Add a private watchlist or create a custom track before using the output for personal investment monitoring.
      taxonomy:
        asset_classes:
        - id: public_equities
          label: Public equities
          event_types: [earnings_release, guidance_change, rating_action, regulatory_action, m_and_a, exec_change, insider_transaction, activist_filing, product_launch_with_material_impact,
            major_customer_or_supplier_news]
          keywords: [earnings, guidance, profit warning, beat estimates, downgrade, upgrade, rating, sec charges, settlement, acquires, acquisition, merger,
            ceo, cfo, resigns, activist, insider selling, stock split, buyback]
        - id: private_equity
          label: Private equity / venture
          event_types: [funding_round, exit_or_acquisition, exec_change, product_launch, competitor_or_peer_round, regulatory_action, down_round_signal, board_change]
          keywords: [funding round, series a, series b, series c, series d, seed round, raises, ipo, acquired by, acquisition of, down round, unicorn, startup,
            closed a round]
        - id: fixed_income_macro
          label: Fixed income / macro
          event_types: [central_bank_policy, cpi_pce_print, payrolls_print, yield_curve_shift, credit_spread_shift, sovereign_debt_action, rate_decision,
            qt_qe_change]
          keywords: [fed, fomc, ecb, bank of england, boe, boj, rate decision, rate cut, rate hike, cpi, pce, inflation, payrolls, unemployment, yield curve,
            treasury, credit spread, sovereign debt, quantitative]
        - id: ai_regulation
          label: AI regulation / policy
          event_types: [ai_rulemaking, antitrust_action, copyright_action, export_control, safety_standard, privacy_action, government_procurement_policy]
          keywords: [ai act, artificial intelligence regulation, ai regulation, responsible ai, algorithmic accountability, ai safety, antitrust, copyright,
            export control, chip restrictions, model transparency, foundation model rules]
        digest_sections:
        - id: portfolio_alerts
          order: 1
          label: Portfolio alerts
        - id: situational_awareness
          order: 2
          label: Situational awareness
        audiences:
        - id: investors
          label: Investors
          lens: portfolio P&L over 1-5 sessions, capital allocation, risk exposure
        - id: portfolio_managers
          label: Portfolio managers
          lens: mandate-relative positioning, factor exposures, drawdown control
        - id: allocators
          label: Allocators
          lens: manager selection, cross-asset allocation, benchmark relative
        - id: gps
          label: GPs
          lens: private-market pipeline, exit windows, comp sets for portfolio companies
        - id: lps
          label: LPs
          lens: fund performance drivers, capital call cadence, sector concentration
      sources:
      - id: fed_press_all_rss
        name: Federal Reserve — Press Releases
        kind: rss
        url: https://www.federalreserve.gov/feeds/press_all.xml
        topic_hints: [fixed_income_macro]
      - id: sec_press_rss
        name: SEC — Press Releases
        kind: rss
        url: https://www.sec.gov/news/pressreleases.rss
        topic_hints: [public_equities]
      - id: boe_news_rss
        name: Bank of England — News
        kind: rss
        url: https://www.bankofengland.co.uk/rss/news
        topic_hints: [fixed_income_macro]
      - id: ecb_press_rss
        name: European Central Bank — Press Releases
        kind: rss
        url: https://www.ecb.europa.eu/rss/press.html
        topic_hints: [fixed_income_macro]
      - id: cnbc_top_rss
        name: CNBC — Top News
        kind: rss
        url: https://www.cnbc.com/id/100003114/device/rss/rss.html
        topic_hints: [public_equities, fixed_income_macro]
      - id: yahoo_finance_rss
        name: Yahoo Finance — Top Stories
        kind: rss
        url: https://finance.yahoo.com/news/rssindex
        topic_hints: [public_equities, fixed_income_macro]
      - id: arstechnica_business_rss
        name: Ars Technica — Business
        kind: rss
        url: https://arstechnica.com/business/feed/
        topic_hints: [public_equities, private_equity]
      - id: federal_register_ai_rss
        name: US Federal Register (artificial intelligence)
        kind: rss
        url: https://www.federalregister.gov/documents/search.rss?conditions%5Bterm%5D=artificial%20intelligence
        topic_hints: [ai_regulation]
      - id: hn_algolia_ai_regulation
        name: Hacker News (AI regulation)
        kind: hn_algolia
        url: https://hn.algolia.com/api/v1/search_by_date?tags=story&query=AI%20regulation&hitsPerPage=20
        topic_hints: [ai_regulation]
      - id: hn_algolia_ai_public_companies
        name: Hacker News (AI chips and public companies)
        kind: hn_algolia
        url: https://hn.algolia.com/api/v1/search_by_date?tags=story&query=NVIDIA%20Microsoft%20AI%20chips&hitsPerPage=20
        topic_hints: [public_equities]
      samples:
      - item_key: sample-ai-capex
        title: 'Starter brief: hyperscaler AI capital spending and demand signals'
        url: https://example.com/tekt-observer/hyperscaler-ai-capex
        topic: public_equities
        content_type: guidance_change
        audiences: [investors, portfolio_managers, allocators]
      - item_key: sample-semiconductor-chain
        title: 'Starter brief: reading AI semiconductor supply-chain signals'
        url: https://example.com/tekt-observer/ai-semiconductor-chain
        topic: public_equities
        content_type: major_customer_or_supplier_news
        audiences: [investors, portfolio_managers]
      - item_key: sample-ai-regulation
        title: 'Starter brief: AI rules, copyright, antitrust, and chip export controls'
        url: https://example.com/tekt-observer/ai-regulation
        topic: ai_regulation
        content_type: ai_rulemaking
        audiences: [investors, portfolio_managers, lps]
  classifier_lexicons:
    topic_watch:
      topic_keywords:
        ai_knowledge_memory: [knowledge graph, context graph, agent memory, vector store, retrieval augmented, rag, hnsw, episodic memory, knowledge storage,
          entity resolution, chunking]
        ai_capabilities: [vision, document vision, document formulate, agent fleet, fleet management]
        open_source_ai: [open source, open-source, oss, harness]
        models_intelligence_inference: [inference, local inference, cloud inference, quantized, quantization, gpu]
        models_inference_strategy: [fusion inference, edge inference, hybrid inference, routing tokens, routing, cross-model]
        models_family: [foundation model, frontier model, open weights]
        enterprise_ai_adoption: [enterprise ai, ai adoption, ai transformation, operating model, change management, return on investment, roi, deployment]
        ai_productivity_workflows: [productivity, workflow, copilot, knowledge work, automation, process redesign, time saved]
        ai_governance_risk: [ai governance, responsible ai, ai risk, compliance, audit, security, guardrail, ai act]
        ai_customer_operations: [customer service, customer support, contact center, sales, marketing, customer experience, crm]
        news_ai: [regulator, adoption, roundup, quarterly report, policy, transparency rule]
      default_topic: news_ai
      audience_keywords:
        architects: [tradeoff, invariant, design, architecture, pattern, hierarchy, provenance, schema]
        builders: [build, building, prototype, code, coding agent, implementation, tuning, benchmark]
        operators: [cost, latency, reliability, throughput, observability, cache, caching, production, repair]
        managers: [team, roadmap, hiring, adoption, playbook, workflow, productivity]
        leaders: [strategy, portfolio, policy, regulator, annual, compliance]
      default_audiences: [architects, builders]
      content_type_hints:
      - [/podcast, podcast]
      - [podcast, podcast]
      - [/video, video]
      - ['video:', video]
      - [.pdf, paper]
      - ['paper:', paper]
      - [/book, book]
      - ['book:', book]
      - ['resource:', resource]
      - [catalog, resource]
      - [catalogue, resource]
      default_content_type: post
    market_watch:
      default_watchlist:
        public_equities:
          tickers: [AAPL, MSFT, NVDA, GOOGL, META, AMZN, AMD, AVGO, TSM, ASML, ARM, ORCL, CRM, PLTR, SNOW, MU, SMCI, BRK.B, JPM, GS, CAT, DE, GE, SIE.DE,
            SPY, QQQ, IWM, XLE, XLI]
          ticker_regex: [\b7203\.T\b, \b8306\.T\b, \bASML\.AS\b]
        private_equity:
          companies: [Ravena Robotics, Northwind Health, Kessel Data, Harborlight Bio, Two Bridges Energy]
          funds: [Meridian Growth III, Cardinal Ventures II]
        fixed_income_macro:
          anchors: [Fed, FOMC, ECB, Bank of England, BoE, BoJ, CPI, PCE, payrolls, Treasury, yield curve, credit spread, 2s10s, 3m10y]
        ai_regulation:
          anchors: [EU AI Act, AI Act, FTC, DOJ, SEC, antitrust, copyright, export control, chip restrictions, AI safety, algorithmic accountability, model
              transparency]
  classifier_lexicons_extra:
    market_watch_event_keyword_map:
      earnings: earnings_release
      guidance: guidance_change
      beat estimates: earnings_release
      profit warning: guidance_change
      downgrade: rating_action
      upgrade: rating_action
      rating: rating_action
      sec charges: regulatory_action
      settlement: regulatory_action
      acquires: m_and_a
      acquisition: m_and_a
      merger: m_and_a
      ceo: exec_change
      cfo: exec_change
      resigns: exec_change
      activist: activist_filing
      insider selling: insider_transaction
      buyback: product_launch_with_material_impact
      funding round: funding_round
      series a: funding_round
      series b: funding_round
      series c: funding_round
      series d: funding_round
      seed round: funding_round
      raises: funding_round
      ipo: exit_or_acquisition
      acquired by: exit_or_acquisition
      down round: down_round_signal
      fed: central_bank_policy
      fomc: central_bank_policy
      ecb: central_bank_policy
      rate decision: rate_decision
      rate cut: rate_decision
      rate hike: rate_decision
      cpi: cpi_pce_print
      pce: cpi_pce_print
      inflation: cpi_pce_print
      payrolls: payrolls_print
      yield curve: yield_curve_shift
      credit spread: credit_spread_shift
      quantitative: qt_qe_change
      ai act: ai_rulemaking
      artificial intelligence regulation: ai_rulemaking
      ai regulation: ai_rulemaking
      antitrust: antitrust_action
      copyright: copyright_action
      export control: export_control
      chip restrictions: export_control
      ai safety: safety_standard
      algorithmic accountability: ai_rulemaking
      model transparency: ai_rulemaking
    career_watch_company_hint_regex: ^(?P<co>[A-Z][\w &.-]{1,40}?)\s*(?:\||:| is |,)
  career_watch_rules:
    role_type_default: ai_augmented_generalist
    seniority_default: senior_ic        # first matching seniority bucket wins, in taxonomy order
    instructor_audience_when_role: ai_instructor
  market_watch_rules:
    asset_class_fallback: "first topic_hint of the source, else public_equities"
    watchlist_overrides_asset_class: true
    audiences_by_asset_class: {any: [investors], private_equity: [investors, gps], fixed_income_macro: [investors, allocators]}
    event_type_fallback: "first event_type of the asset class, else news"
```

## Open decisions

```yaml
open_decisions:
  - id: product-name
    question: "Keep 'tekt.observer' for the rebuild, or rename the viewer once it becomes a Catalog Viewer?"
    evidence: "ANALYSIS.md section 7; t76u says rebuild rather than move."
  - id: user-agent-domain
    question: "Every fetcher's User-Agent embeds github.com/xingh/tekt.observer. A rebuild needs an owned contact URL."
  - id: starter-source-health
    question: "7 of 23 starter feeds failed or were empty on 2026-11-26 (arXiv x2 HTTP 500, ai-jobs.net parse error, Yahoo Finance 404, Federal Register XML error, Ars Technica and one HN query empty). Replace or keep?"
```
