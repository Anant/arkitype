# tekt.observer — arkitype reverse-engineering (reference extraction)

This folder holds an arkitype `/extract` of [`xingh/tekt.observer`](https://github.com/xingh/tekt.observer) at commit `42534a5`, written 2026-11-26. It documents the system so it can be rebuilt from understanding rather than moved. Source repo license: MIT (see `xingh/tekt.observer`).

| File | What it holds |
|---|---|
| [00-arkitype.md](./00-arkitype.md) | SITE_PROFILE: identity, brand tokens, runtime defaults, toggles, providers, delivery presets, the three shipped watchers (taxonomy, sources, samples), classifier lexicons, open decisions |
| [01-infrastructure.md](./01-infrastructure.md) | Toolchain, Modes L/P/H (local app, scheduled host, PocketBase compose), env var names, test gate |
| [02-database.md](./02-database.md) | Hash-chained journal, PocketBase collections and rules, exchange bundles, every pipeline artifact schema, track config/state files |
| [03-software.md](./03-software.md) | Pipeline stages and algorithms, both runners, scheduler, worker, local and legacy APIs, CLI, self-heal loop, delivery, agent runtime, defects |
| [03a-source-adapters.md](./03a-source-adapters.md) | SourceAdapter contract, feed kinds, and the full 60-mode catalog with rebuild guidance |
| [03b-agent-skills.md](./03b-agent-skills.md) | AGENTS.md mode router and the six skills with keep/rebuild verdicts |
| [04-interface.md](./04-interface.md) | React workspace app, legacy viewer routes, static site layout, email/Telegram formats, accessibility, UI defects |
| [agent.local.md](./agent.local.md) | Who extracted it and how, what was observed vs static, gaps, deviations from the method |
| [ROADMAP-DELTA.md](./ROADMAP-DELTA.md) | Capability mapping: skills to keep, Catalog Viewer needs, what to rebuild, license |
| `assets/` | Observed screenshots: `inbox.png` and the `legacy-*.png` pages (the feed page screenshot was omitted because it showed a personal name) |

Source: `xingh/tekt.observer` @ `42534a5` only.
