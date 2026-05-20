# Changelog

## [Unreleased]

## [0.1.3] - 2026-05-20

### Added

- New `biff-lenses` skill (model-invoked, auto-triggered alongside the [code-lenses](https://github.com/brackendev/code-lenses) plugin). Translates the four default code-lenses philosophies (grug, Honest Code, Tidy First, Parse Don't Validate) and the two opt-in philosophies (APOSD, Legacy Code) into Biff-specific patterns. Records only the Biff-specific deltas on top of `clojure-lenses`: XTDB transactions through `biff/submit-tx`, Malli schema enforcement via `:db/doc-type` and `doc-schema`, the module + system-map architecture, htmx response patterns, the authentication module, scheduled tasks via `use-chime`, transaction listeners, queues via `biff/submit-job`, and the live REPL workflow. Closes the Clojure-family gap where `clojure-skills`, `clojurescript-skills`, `clojuredart-skills`, and `fulcro-skills` all ship a `*-lenses` companion but `biff-skills` did not.

### Changed

- Skill bodies and documentation that reference the sibling `clojure-skills` quality pipeline now point at `/clj-fix` instead of `/clj-tidy`, matching the rename in that companion package. The behavior is unchanged; only the command name moves to the family-wide noun-first canonical naming.

## [0.1.2]

### Added

- `CONVENTIONS.md` at the repo root defines the argument grammar, scope vocabulary, and mutation defaults that every user-invocable skill in this plugin follows. The three rules: natural-language keywords and bare paths with `--report` as the only sanctioned flag, a shared scope vocabulary (`(no argument)`, `all`, `<path>`, with opt-in rows for pull requests and commit messages), and mutation as the default.
- `/biff-deploy` now accepts `--report` to print the planned commands for the selected deployment path (resolved target host, `git ls-files` upload manifest, Docker image tag, or uberjar output path) without writing or running anything.
- `/biff-deploy` accepts the scope keyword `all` as a synonym for the default VPS path. Accepted for family consistency with the standard scope vocabulary.

### Changed

- `/biff-new` and `/biff-deploy` now carry the canonical `## Arguments` table documented in `CONVENTIONS.md`. `/biff-new` is exempt from the `all` and `<path>` rows because scaffolding has no useful default scope; `/biff-deploy` operates on deployment-target keywords (`vps`, `docker`, `uberjar`) rather than file paths.
- Skill bodies that reference the sibling `clojure-skills` quality pipeline now point at `/clj-tidy` instead of `/clj-check`, matching the rename in that companion package.

### Migration

- Saved `/biff-deploy` invocations that pass `vps`, `docker`, or `uberjar` continue to work unchanged. Pass `--report` to receive a preview of the planned commands for the selected path without writing or running anything.
- Saved `/biff-new <project-name>` invocations continue to work unchanged.
- Workflows that ran `/clj-check` from the companion `clojure-skills` package now use `/clj-tidy`; the references in this plugin's documentation point at the new name.

## [0.1.1]

### Added

- The `biff` reference now documents `:db.op/upsert` (the recommended replacement for the deprecated `:db/lookup`), the `:db/op :create` operation, and the per-attribute sentinels `:db/union`, `:db/difference`, `:db/add`, `:db/default`, `:db/dissoc`, and `:db/unique`.

### Changed

- The `biff` skill now defers to both [clojure-skills](https://github.com/brackendev/clojure-skills) (host-neutral baseline) and [clojure-jvm-skills](https://github.com/brackendev/clojure-jvm-skills) (JVM-specific Clojure: Java interop, refs / agents / STM, `with-open`, JVM-typed exceptions, `alter-var-root`, Clojure CLI / `tools.build` / `clj-kondo` / `cljfmt` / `test-runner` / nREPL workflow). Install all three packages together for Biff projects.

### 0.1.0

#### Added

- Initial release. Three Biff skills packaged as an APM plugin and deployed to every runtime APM supports (Claude Code, Codex, OpenCode, Cursor, Copilot, Gemini, Windsurf). Designed to layer on top of [clojure-skills](https://github.com/brackendev/clojure-skills) rather than duplicate it.
- **biff** (model-invoked): Project layout, the module + system-map architecture, transactions and queries against XTDB with Malli schema enforcement, routing, htmx response patterns, authentication, scheduled tasks, transaction listeners, queues, and REPL workflow. Defers to the [clojure](https://github.com/brackendev/clojure-skills) skill for general Clojure style. Pulls deeper detail from `references/biff-reference.md` on demand.
- **biff-new** (user-invoked): Scaffold a new Biff project via the official `clj -M -e '(load-string (slurp "https://biffweb.com/new.clj"))'` installer. Confirms namespace and project directory, then verifies that `clj -M:dev dev` starts the app.
- **biff-deploy** (user-invoked): Deployment workflow for the three Biff paths: managed Ubuntu VPS via `server-setup.sh` and `clj -M:dev deploy`, Docker image build, or standalone uberjar. Includes post-deploy verification via `clj -M:dev logs` and a smoke check against the live domain.
