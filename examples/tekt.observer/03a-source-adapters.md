# 03a — Source adapters (supplement to 03)

This supplement covers the **60 discovery modes** registered at `xingh/tekt.observer@42534a5`, plus the three feed kinds read by `feed_gather.py`. It is architecture: an adapter knows a *kind of site* (an ATS, a career-page engine, a public-sector board), not a particular watcher. Watchers pick adapters by naming a `discovery_mode` in their `sources.json`. The keyless feed registries for the shipped watchers are in [00 SITE_PROFILE.watchers](./00-arkitype.md).

The list was **observed** by importing `discover.registry.load_registry()` (60 modes across 36 provider modules) and confirmed against the generated catalog `shared/discovery_modes.md`, which `render_discovery_modes_md.py --check` also passed.

## Adapter contract

```yaml
adapter_contract:
  registry: "scripts/discover/registry.py"
  discovery: "import every non-underscore module in scripts/discover/sources/; collect module-level SOURCE (one) and SOURCES (list); a duplicate mode raises ValueError; cached per process"
  SourceAdapter:
    modes: "tuple[str]: one adapter may serve several modes (aliases or sibling sites on the same engine)"
    discover: "(SourceConfig, terms: list[str], timeout_seconds: int) -> Coverage"
    options_schema: "Mapping[str, type] | None (unused by every adapter at this commit)"
    requires: "tuple of importable module names; only browser modes declare 'playwright.sync_api'"
    emits_candidates: "bool; false = coverage/health only (no candidate rows)"
  SourceConfig: {source: name, url: str, discovery_mode: str, last_checked: "date|null", cadence_group: str, filters: "{str: [str]}", source_id: str}
  Coverage: {source, source_url, discovery_mode, cadence_group, last_checked, due_today, status: "complete|partial|failed", listing_pages_scanned, search_terms_tried, result_pages_scanned, direct_job_pages_opened, enumerated_jobs, matched_jobs, limitations: [str], candidates: [Candidate], source_id, filters}
  Candidate: {employer, title, url, source_url, alternate_url: "", location: "unknown", remote: "unknown", matched_terms: [], notes: "", description: "", description_truncated: false}
  browser_helpers: "BrowserPageResult / BrowserEnrichmentResult / BrowserStrategy dataclasses in discover/core.py; Playwright pagination capped at MAX_BROWSER_PAGES=10, 60s per page"
  shared_helpers: "discover/helpers.py: visible-text extraction, JSON-after-marker extraction for inline app state, term matching with aliases, infer_remote_status, looks_like_job_link, merge_candidate (dedupe by URL), should_keep_candidate (03 keep_rule), JD description setter with truncation"
  http: "discover/http.py: shared fetch helpers, default timeout 20s"
  sap_odata: "scripts/sap_odata.py: SAP SuccessFactors OData helper"
  contract_tests: "tests/contract/test_source_contract.py replays tests/fixtures/sources/<mode>/ for each fixture-backed mode"
  catalog_generator: "render_discovery_modes_md.py renders shared/discovery_modes.md from registry metadata; --check guards drift"
  how_to_add: "add or extend a module under scripts/discover/sources/ exposing SOURCE/SOURCES; add fixtures + a contract test; regenerate the catalog. Never grow discover_jobs.py (a CLI compatibility shim)."
```

## Feed kinds (watcher registries)

```yaml
feed_kinds:
  rss:        {parser: "ElementTree channel/item", history: "client-side pubDate window only"}
  atom:       {parser: "ElementTree atom:entry", history: "client-side published/updated window only"}
  hn_algolia: {parser: "JSON hits[]", history: "server-side numericFilters on created_at_i (full history)"}
  extension_rule: "a new kind needs a parser in feed_gather._gather_source AND an entry in generate_watchers.SOURCE_KINDS"
```

## Grouping by provider module

```yaml
modules:   # 36 modules -> 60 modes (observed)
  generic_html:   [html, icims_html, factorial, cybernetica_teamdash, knds_jobboard, secunet_jobboard]
  browser:        [browser, asml_browser, automattic_browser, coinbase_browser, helsing_browser, thales_browser, trailofbits_browser]   # requires playwright
  static_pages:   [pcd_team, qedit_inline, neclab_jobs, leastauthority_careers, qusecure_careers, partisia_site]   # last three: emits_candidates=false (health-only)
  public_service: [verfassungsschutz_rss, auswaertiges_amt_json, bnd_career_search, bosch_autocomplete]
  ashby:          [ashby_api, ashby_html]
  eightfold:      [eightfold_api, infineon_api]
  hackernews:     [hackernews_jobs, hackernews_whoishiring_api]
  service_bund:   [service_bund_search, service_bund_links]
  successfactors: [demant_rss, sennheiser_rss]
  single_mode:    [alphatheta_html, apple_jobs, bamboohr_api, bundeswehr_jobsuche, dover_api, ecb_avature_rss, enbw_phenom, getro_api, greenhouse_api, harman_html,
                   hibob_api, iacr_jobs, ibm_api, jobvite_html, krisp_html, lever_json, lifeatspotify_api, personio_page, recruitee_inline, rheinmetall_html,
                   softgarden_html, teamtailor_api, thales_html, ultipro_api, workable_api, workday_api, yc_jobs_board]
rebuild_note: >-
  Roughly 20 modes are reusable engines (greenhouse, lever, ashby, workday, workable, teamtailor, recruitee, personio, bamboohr, hibob, getro,
  eightfold, successfactors, jobvite, icims, ultipro, softgarden, plus generic html and the Playwright browser driver). The other ~40 are single-employer or single-agency
  scrapers tuned to one job-search corpus (regional public-sector and security employers). Rebuild the engines; regenerate employer-specific ones on demand with
  the explore -> integrate loop instead of hand-porting them.
```

## Full mode catalog

Each entry gives the mode, the provider module and function, whether it emits candidates, the URL shape it expects, the filters it supports, its runtime requirements, its contract fixtures, and its behavior and known limits.

```yaml
adapters:
  - mode: alphatheta_html
    module: discover.sources.alphatheta
    function: discover_alphatheta_jobs
    emits_candidates: true
    url_shape: "https://alphatheta.com/... careers page."
    filters: "none"
    requires: "none"
    contract_fixtures: "tests/fixtures/sources/alphatheta_html/"
    behavior_and_limits: "Recognizes the no-open-positions marker as a complete empty result; otherwise falls back to generic static-HTML link enumeration."
  - mode: apple_jobs
    module: discover.sources.apple
    function: discover_apple_jobs
    emits_candidates: true
    url_shape: "https://jobs.apple.com/<locale>/search?..."
    filters: "encode Apple keyword/team/location filters in the source URL"
    requires: "none"
    contract_fixtures: "tests/fixtures/sources/apple_jobs/"
    behavior_and_limits: "Parses direct Apple job-detail links and visible search-result summaries from the static search page."
  - mode: ashby_api
    module: discover.sources.ashby
    function: discover_ashby_api
    emits_candidates: true
    url_shape: "https://jobs.ashbyhq.com/<organization>"
    filters: "none"
    requires: "none"
    contract_fixtures: "tests/fixtures/sources/ashby_api/"
    behavior_and_limits: "Uses Ashby public GraphQL job-board data; matched candidates are enriched with task/qualification/compensation sections from the per-posting GraphQL detail query."
  - mode: ashby_html
    module: discover.sources.ashby
    function: discover_ashby_api
    emits_candidates: true
    url_shape: "https://jobs.ashbyhq.com/<organization>"
    filters: "none"
    requires: "none"
    contract_fixtures: "tests/fixtures/sources/ashby_html/"
    behavior_and_limits: "Compatibility alias for the Ashby provider."
  - mode: asml_browser
    module: discover.sources.browser
    function: discover_asml_browser
    emits_candidates: true
    url_shape: "https://www.asml.com/en/careers/find-your-job"
    filters: "none"
    requires: "playwright.sync_api"
    contract_fixtures: "none"
    behavior_and_limits: "Requires Playwright and installed Chromium browser binaries."
  - mode: auswaertiges_amt_json
    module: discover.sources.public_service
    function: discover_auswaertiges_amt_json
    emits_candidates: true
    url_shape: "Auswaertiges Amt careers page exposing an /ajax/json-filterlist/... endpoint."
    filters: "none"
    requires: "none"
    contract_fixtures: "tests/fixtures/sources/auswaertiges_amt_json/"
    behavior_and_limits: "Depends on the page retaining its embedded JSON-filter endpoint."
  - mode: automattic_browser
    module: discover.sources.browser
    function: discover_automattic_browser
    emits_candidates: true
    url_shape: "https://automattic.com/work-with-us/"
    filters: "none"
    requires: "playwright.sync_api"
    contract_fixtures: "none"
    behavior_and_limits: "Requires Playwright and extracts visible job-card links only."
  - mode: bamboohr_api
    module: discover.sources.bamboohr
    function: discover_bamboohr_jobs
    emits_candidates: true
    url_shape: "https://<tenant>.bamboohr.com/careers"
    filters: "none"
    requires: "none"
    contract_fixtures: "tests/fixtures/sources/bamboohr_api/"
    behavior_and_limits: "Uses the tenant careers/list and careers/<id>/detail JSON endpoints; fetches every posting detail before term filtering."
  - mode: bnd_career_search
    module: discover.sources.public_service
    function: discover_bnd_career_search
    emits_candidates: true
    url_shape: "BND advanced career-search form URL."
    filters: "encode BND native filters in the source URL"
    requires: "none"
    contract_fixtures: "tests/fixtures/sources/bnd_career_search/"
    behavior_and_limits: "Parses native result cards from the rendered HTML response."
  - mode: bosch_autocomplete
    module: discover.sources.public_service
    function: discover_bosch_autocomplete
    emits_candidates: true
    url_shape: "Bosch careers jobs page."
    filters: "none"
    requires: "none"
    contract_fixtures: "tests/fixtures/sources/bosch_autocomplete/"
    behavior_and_limits: "Uses the public autocomplete/search endpoint discovered for Bosch careers."
  - mode: browser
    module: discover.sources.browser
    function: discover_browser
    emits_candidates: true
    url_shape: "Generic browser strategy keyed by source name; currently Google, Meta, and Bosch strategies are implemented."
    filters: "location and degree for Google sources"
    requires: "playwright.sync_api"
    contract_fixtures: "none"
    behavior_and_limits: "Requires Playwright and installed Chromium browser binaries."
  - mode: bundeswehr_jobsuche
    module: discover.sources.bundeswehr
    function: discover_bundeswehr_jobsuche
    emits_candidates: true
    url_shape: "Bundeswehr jobsuche portal URL."
    filters: "none"
    requires: "none"
    contract_fixtures: "tests/fixtures/sources/bundeswehr_jobsuche/"
    behavior_and_limits: "Combines portal fallback and SAP OData-style discovery where available."
  - mode: coinbase_browser
    module: discover.sources.browser
    function: discover_coinbase_browser
    emits_candidates: true
    url_shape: "https://www.coinbase.com/careers"
    filters: "none"
    requires: "playwright.sync_api"
    contract_fixtures: "none"
    behavior_and_limits: "Currently reports partial coverage when Cloudflare or unsupported listing extraction blocks deterministic enumeration."
  - mode: cybernetica_teamdash
    module: discover.sources.generic_html
    function: discover_cybernetica_teamdash
    emits_candidates: true
    url_shape: "Cybernetica careers page with cyber.teamdash.com/p/job/ links."
    filters: "none"
    requires: "none"
    contract_fixtures: "tests/fixtures/sources/cybernetica_teamdash/"
    behavior_and_limits: "Only direct Teamdash job links visible in the careers-page HTML are enumerated."
  - mode: demant_rss
    module: discover.sources.successfactors
    function: discover_demant_successfactors
    emits_candidates: true
    url_shape: "Demant careers page; enumeration uses careers.demant.com/services/rss/job/ keyword search."
    filters: "none"
    requires: "none"
    contract_fixtures: "tests/fixtures/sources/demant_rss/"
    behavior_and_limits: "One RSS search per track term; postings not matching any searched keyword are not enumerated."
  - mode: dover_api
    module: discover.sources.dover
    function: discover_dover_jobs
    emits_candidates: true
    url_shape: "https://app.dover.com/<company>/careers/<careers-page-uuid>"
    filters: "none"
    requires: "none"
    contract_fixtures: "tests/fixtures/sources/dover_api/"
    behavior_and_limits: "Uses the Dover careers-page JSON API; unpublished and sample postings are skipped."
  - mode: ecb_avature_rss
    module: discover.sources.ecb
    function: discover_ecb_avature_rss
    emits_candidates: true
    url_shape: "ECB Avature careers page or its SearchJobs/feed/ RSS URL."
    filters: "none"
    requires: "none"
    contract_fixtures: "tests/fixtures/sources/ecb_avature_rss/"
    behavior_and_limits: "Uses the official Avature RSS feed and opens matched job-detail pages for role sections."
  - mode: eightfold_api
    module: discover.sources.eightfold
    function: discover_eightfold_api
    emits_candidates: true
    url_shape: "Eightfold-hosted careers URL such as Microsoft careers."
    filters: "none"
    requires: "none"
    contract_fixtures: "tests/fixtures/sources/eightfold_api/"
    behavior_and_limits: "Provider contains host-specific domain mapping for supported Eightfold boards."
  - mode: enbw_phenom
    module: discover.sources.enbw
    function: discover_enbw_phenom
    emits_candidates: true
    url_shape: "https://careers.enbw.com/en_US/careers"
    filters: "none"
    requires: "none"
    contract_fixtures: "tests/fixtures/sources/enbw_phenom/"
    behavior_and_limits: "Parses embedded Phenom search payloads from localized search-result pages."
  - mode: factorial
    module: discover.sources.generic_html
    function: discover_factorial
    emits_candidates: true
    url_shape: "https://<tenant>.factorialhr.com/ company career page."
    filters: "none"
    requires: "none"
    contract_fixtures: "tests/fixtures/sources/factorial/"
    behavior_and_limits: "Enumerates listing links like generic HTML, then opens each /job_posting/ detail page to recover the real title, location, and role detail, since the listing only exposes anonymous apply links."
  - mode: getro_api
    module: discover.sources.getro
    function: discover_getro_api
    emits_candidates: true
    url_shape: "Getro collection jobs URL."
    filters: "none"
    requires: "none"
    contract_fixtures: "tests/fixtures/sources/getro_api/"
    behavior_and_limits: "Uses Getro collection API pagination and provider-side post-filtering."
  - mode: greenhouse_api
    module: discover.sources.greenhouse
    function: discover_greenhouse_api
    emits_candidates: true
    url_shape: "https://job-boards.greenhouse.io/<board>"
    filters: "none"
    requires: "none"
    contract_fixtures: "tests/fixtures/sources/greenhouse_api/"
    behavior_and_limits: "Uses the Greenhouse public job-board API shape."
  - mode: hackernews_jobs
    module: discover.sources.hackernews
    function: discover_hackernews_jobs
    emits_candidates: true
    url_shape: "https://news.ycombinator.com/jobs"
    filters: "none"
    requires: "none"
    contract_fixtures: "tests/fixtures/sources/hackernews_jobs/"
    behavior_and_limits: "Enumerates Hacker News jobs pages and external links visible in listing HTML."
  - mode: hackernews_whoishiring_api
    module: discover.sources.hackernews
    function: discover_hackernews_whoishiring_api
    emits_candidates: true
    url_shape: "https://news.ycombinator.com/user?id=whoishiring"
    filters: "none"
    requires: "none"
    contract_fixtures: "tests/fixtures/sources/hackernews_whoishiring_api/"
    behavior_and_limits: "Parses the active Who Is Hiring thread and infers employer/title fields from post text."
  - mode: harman_html
    module: discover.sources.harman
    function: discover_harman_jobsearch_jobs
    emits_candidates: true
    url_shape: "https://jobsearch.harman.com/<locale>/careers/... listing page."
    filters: "none"
    requires: "none"
    contract_fixtures: "tests/fixtures/sources/harman_html/"
    behavior_and_limits: "Only direct Avature JobDetail links visible in the listing HTML are enumerated."
  - mode: helsing_browser
    module: discover.sources.browser
    function: discover_helsing_browser
    emits_candidates: true
    url_shape: "https://helsing.ai/jobs"
    filters: "none"
    requires: "playwright.sync_api"
    contract_fixtures: "none"
    behavior_and_limits: "Requires Playwright and extracts visible job cards from the jobs page."
  - mode: hibob_api
    module: discover.sources.hibob
    function: discover_hibob_api
    emits_candidates: true
    url_shape: "https://<company>.careers.hibob.com hosted careers page."
    filters: "none"
    requires: "none"
    contract_fixtures: "tests/fixtures/sources/hibob_api/"
    behavior_and_limits: "Reads every published posting from the HiBob /api/job-ad JSON endpoint (requires the companyIdentifier header derived from the careers subdomain); descriptions come from the same response, so no per-posting fetch is needed."
  - mode: html
    module: discover.sources.generic_html
    function: discover_html
    emits_candidates: true
    url_shape: "Any official static careers/listings page with direct job links."
    filters: "none"
    requires: "none"
    contract_fixtures: "tests/fixtures/sources/html/"
    behavior_and_limits: "Best-effort static link extraction; does not execute JavaScript."
  - mode: iacr_jobs
    module: discover.sources.iacr
    function: discover_iacr_jobs
    emits_candidates: true
    url_shape: "https://www.iacr.org/jobs/"
    filters: "none"
    requires: "none"
    contract_fixtures: "tests/fixtures/sources/iacr_jobs/"
    behavior_and_limits: "Parses IACR posting blocks from the jobs page."
  - mode: ibm_api
    module: discover.sources.ibm
    function: discover_ibm_api
    emits_candidates: true
    url_shape: "https://www.ibm.com/careers/search"
    filters: "none"
    requires: "none"
    contract_fixtures: "tests/fixtures/sources/ibm_api/"
    behavior_and_limits: "Uses IBM careers search API with title-scoped query terms."
  - mode: icims_html
    module: discover.sources.generic_html
    function: discover_html
    emits_candidates: true
    url_shape: "Official iCIMS or employer careers page exposing static job links."
    filters: "none"
    requires: "none"
    contract_fixtures: "tests/fixtures/sources/icims_html/"
    behavior_and_limits: "Static HTML fallback only; blocked iCIMS pages may return failed or partial coverage."
  - mode: infineon_api
    module: discover.sources.eightfold
    function: discover_eightfold_api
    emits_candidates: true
    url_shape: "https://jobs.infineon.com/careers"
    filters: "none"
    requires: "none"
    contract_fixtures: "tests/fixtures/sources/infineon_api/"
    behavior_and_limits: "Infineon-specific Eightfold provider wrapper."
  - mode: jobvite_html
    module: discover.sources.jobvite
    function: discover_jobvite_jobs
    emits_candidates: true
    url_shape: "https://jobs.jobvite.com/<company>/... listing page."
    filters: "none"
    requires: "none"
    contract_fixtures: "tests/fixtures/sources/jobvite_html/"
    behavior_and_limits: "Parses jv-job-list rows from the listing HTML and enriches matched candidates from job-detail pages."
  - mode: knds_jobboard
    module: discover.sources.generic_html
    function: discover_knds_jobboard
    emits_candidates: true
    url_shape: "https://jobs.knds.de/content/search/?locale=de_DE"
    filters: "none"
    requires: "none"
    contract_fixtures: "tests/fixtures/sources/knds_jobboard/"
    behavior_and_limits: "Uses visible KNDS job-detail links when static links are present, then falls back to the public Recruiting Solutions search API."
  - mode: krisp_html
    module: discover.sources.krisp
    function: discover_krisp_jobs
    emits_candidates: true
    url_shape: "https://krisp.ai/careers/"
    filters: "none"
    requires: "none"
    contract_fixtures: "tests/fixtures/sources/krisp_html/"
    behavior_and_limits: "Enumerates direct /jobs/<slug> links and enriches from each job-detail page."
  - mode: leastauthority_careers
    module: discover.sources.static_pages
    function: discover_leastauthority_careers
    emits_candidates: false
    url_shape: "Least Authority careers page."
    filters: "none"
    requires: "none"
    contract_fixtures: "tests/fixtures/sources/leastauthority_careers/"
    behavior_and_limits: "Coverage marker provider; may emit no candidates when no current listing links are exposed."
  - mode: lever_json
    module: discover.sources.lever
    function: discover_lever_json
    emits_candidates: true
    url_shape: "https://jobs.lever.co/<organization>"
    filters: "none"
    requires: "none"
    contract_fixtures: "tests/fixtures/sources/lever_json/"
    behavior_and_limits: "Uses Lever public postings JSON."
  - mode: lifeatspotify_api
    module: discover.sources.lifeatspotify
    function: discover_lifeatspotify_jobs
    emits_candidates: true
    url_shape: "https://www.lifeatspotify.com/jobs"
    filters: "none"
    requires: "none"
    contract_fixtures: "tests/fixtures/sources/lifeatspotify_api/"
    behavior_and_limits: "Uses the public api.lifeatspotify.com job search and detail endpoints; details fetched for matched candidates only."
  - mode: neclab_jobs
    module: discover.sources.static_pages
    function: discover_neclab_jobs
    emits_candidates: true
    url_shape: "NEC Laboratories Europe jobs page."
    filters: "none"
    requires: "none"
    contract_fixtures: "tests/fixtures/sources/neclab_jobs/"
    behavior_and_limits: "Static page provider for visible job links."
  - mode: partisia_site
    module: discover.sources.static_pages
    function: discover_partisia_site
    emits_candidates: false
    url_shape: "Partisia careers/company site URLs."
    filters: "none"
    requires: "none"
    contract_fixtures: "tests/fixtures/sources/partisia_site/"
    behavior_and_limits: "Coverage marker provider for pages that may not expose current job links."
  - mode: pcd_team
    module: discover.sources.static_pages
    function: discover_pcd_team
    emits_candidates: true
    url_shape: "https://pcd.team/jd"
    filters: "none"
    requires: "none"
    contract_fixtures: "tests/fixtures/sources/pcd_team/"
    behavior_and_limits: "Static PCD Team provider with detail-section extraction."
  - mode: personio_page
    module: discover.sources.personio
    function: discover_personio_page
    emits_candidates: true
    url_shape: "https://<organization>.jobs.personio.de/"
    filters: "none"
    requires: "none"
    contract_fixtures: "tests/fixtures/sources/personio_page/"
    behavior_and_limits: "Extracts Personio jobs from embedded page payloads, falling back to the tenant /xml feed when the page payload is missing."
  - mode: qedit_inline
    module: discover.sources.static_pages
    function: discover_qedit_inline
    emits_candidates: true
    url_shape: "QEDIT careers page."
    filters: "none"
    requires: "none"
    contract_fixtures: "tests/fixtures/sources/qedit_inline/"
    behavior_and_limits: "Static inline careers-page provider."
  - mode: qusecure_careers
    module: discover.sources.static_pages
    function: discover_qusecure_careers
    emits_candidates: false
    url_shape: "QuSecure careers page."
    filters: "none"
    requires: "none"
    contract_fixtures: "tests/fixtures/sources/qusecure_careers/"
    behavior_and_limits: "Coverage marker provider for pages without direct current listing links."
  - mode: recruitee_inline
    module: discover.sources.recruitee
    function: discover_recruitee_inline
    emits_candidates: true
    url_shape: "Recruitee-hosted careers page exposing embedded app config."
    filters: "none"
    requires: "none"
    contract_fixtures: "tests/fixtures/sources/recruitee_inline/"
    behavior_and_limits: "Depends on the page retaining Recruitee embedded data attributes."
  - mode: rheinmetall_html
    module: discover.sources.rheinmetall
    function: discover_rheinmetall_html
    emits_candidates: true
    url_shape: "Rheinmetall current jobs page, including encoded native filters in the source URL."
    filters: "encode Rheinmetall native filters in the source URL"
    requires: "none"
    contract_fixtures: "tests/fixtures/sources/rheinmetall_html/"
    behavior_and_limits: "Parses Rheinmetall result-card HTML and pagination."
  - mode: secunet_jobboard
    module: discover.sources.generic_html
    function: discover_secunet_jobboard
    emits_candidates: true
    url_shape: "secunet careers or https://jobs.secunet.com/ page with *-j<id>.html job links."
    filters: "none"
    requires: "none"
    contract_fixtures: "tests/fixtures/sources/secunet_jobboard/"
    behavior_and_limits: "Only links matching the secunet job-detail URL pattern are enumerated."
  - mode: sennheiser_rss
    module: discover.sources.successfactors
    function: discover_sennheiser_jobs
    emits_candidates: true
    url_shape: "https://jobs.sennheiser.com/"
    filters: "none"
    requires: "none"
    contract_fixtures: "tests/fixtures/sources/sennheiser_rss/"
    behavior_and_limits: "Reads the full jobs.sennheiser.com RSS feed in a single fetch."
  - mode: service_bund_links
    module: discover.sources.service_bund
    function: discover_service_bund_links
    emits_candidates: true
    url_shape: "Official employer page exposing service.bund.de/.../IMPORTE/Stellenangebote/... links."
    filters: "none"
    requires: "none"
    contract_fixtures: "tests/fixtures/sources/service_bund_links/"
    behavior_and_limits: "Enumerates direct service.bund job-detail links from the source page."
  - mode: service_bund_search
    module: discover.sources.service_bund
    function: discover_service_bund_search
    emits_candidates: true
    url_shape: "https://www.service.bund.de/Content/DE/Stellen/Suche/Formular.html..."
    filters: "encode service.bund native filters in the source URL"
    requires: "none"
    contract_fixtures: "tests/fixtures/sources/service_bund_search/"
    behavior_and_limits: "Parses service.bund search results and next-page tokens."
  - mode: softgarden_html
    module: discover.sources.softgarden
    function: discover_softgarden_html
    emits_candidates: true
    url_shape: "https://<tenant>.softgarden.io/<language>/vacancies"
    filters: "jobcategory, location, and company"
    requires: "none"
    contract_fixtures: "tests/fixtures/sources/softgarden_html/"
    behavior_and_limits: "Parses static Softgarden vacancy cards and opens matched detail pages for role-section snippets."
  - mode: teamtailor_api
    module: discover.sources.teamtailor
    function: discover_teamtailor_jobs
    emits_candidates: true
    url_shape: "Teamtailor-hosted careers site exposing /jobs.json at the site root."
    filters: "none"
    requires: "none"
    contract_fixtures: "tests/fixtures/sources/teamtailor_api/"
    behavior_and_limits: "Reads the standard Teamtailor jobs.json feed; locations come from embedded jobposting address data."
  - mode: thales_browser
    module: discover.sources.browser
    function: discover_thales_browser
    emits_candidates: true
    url_shape: "https://careers.thalesgroup.com/global/en/search-results"
    filters: "none"
    requires: "playwright.sync_api"
    contract_fixtures: "none"
    behavior_and_limits: "Requires Playwright; HTML provider is preferred when embedded payloads are available."
  - mode: thales_html
    module: discover.sources.thales
    function: discover_thales_html
    emits_candidates: true
    url_shape: "https://careers.thalesgroup.com/global/en/search-results"
    filters: "none"
    requires: "none"
    contract_fixtures: "tests/fixtures/sources/thales_html/"
    behavior_and_limits: "Parses embedded Phenom-style search payloads and Thales job links."
  - mode: trailofbits_browser
    module: discover.sources.browser
    function: discover_trailofbits_browser
    emits_candidates: true
    url_shape: "https://trailofbits.com/careers/"
    filters: "none"
    requires: "playwright.sync_api"
    contract_fixtures: "none"
    behavior_and_limits: "Requires Playwright and extracts visible Workable application links."
  - mode: ultipro_api
    module: discover.sources.ultipro
    function: discover_ultipro_jobs
    emits_candidates: true
    url_shape: "https://recruiting2.ultipro.com/<tenant>/JobBoard/<board-guid>/..."
    filters: "none"
    requires: "none"
    contract_fixtures: "tests/fixtures/sources/ultipro_api/"
    behavior_and_limits: "Pages the LoadSearchResults JSON endpoint (up to 10 pages of 100)."
  - mode: verfassungsschutz_rss
    module: discover.sources.public_service
    function: discover_verfassungsschutz_rss
    emits_candidates: true
    url_shape: "Verfassungsschutz jobs source; provider reads the official jobs RSS feed."
    filters: "none"
    requires: "none"
    contract_fixtures: "tests/fixtures/sources/verfassungsschutz_rss/"
    behavior_and_limits: "Uses RSS plus detail-page enrichment where reachable."
  - mode: workable_api
    module: discover.sources.workable
    function: discover_workable_api
    emits_candidates: true
    url_shape: "https://apply.workable.com/<board>/"
    filters: "none"
    requires: "none"
    contract_fixtures: "tests/fixtures/sources/workable_api/"
    behavior_and_limits: "Uses Workable public board endpoints."
  - mode: workday_api
    module: discover.sources.workday
    function: discover_workday_api
    emits_candidates: true
    url_shape: "https://<tenant>.wd<region>.myworkdayjobs.com/<site>"
    filters: "none"
    requires: "none"
    contract_fixtures: "tests/fixtures/sources/workday_api/"
    behavior_and_limits: "Supports standard Workday candidate API search; source URL must expose a compatible Workday host."
  - mode: yc_jobs_board
    module: discover.sources.yc
    function: discover_yc_jobs_board
    emits_candidates: true
    url_shape: "https://www.ycombinator.com/jobs/role/<role>"
    filters: "none"
    requires: "none"
    contract_fixtures: "tests/fixtures/sources/yc_jobs_board/"
    behavior_and_limits: "Parses the YC jobs Next.js payload for the configured role board."
```
