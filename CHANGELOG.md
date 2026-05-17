# Changelog

## [Unreleased]

### 0.1.0

#### Added

- Initial release. Three Biff skills packaged as an APM plugin and deployed to every runtime APM supports (Claude Code, Codex, OpenCode, Cursor, Copilot, Gemini, Windsurf). Designed to layer on top of [clojure-skills](https://github.com/brackendev/clojure-skills) rather than duplicate it.
- **biff** (model-invoked): Project layout, the module + system-map architecture, transactions and queries against XTDB with Malli schema enforcement, routing, htmx response patterns, authentication, scheduled tasks, transaction listeners, queues, and REPL workflow. Defers to the [clojure](https://github.com/brackendev/clojure-skills) skill for general Clojure style. Pulls deeper detail from `references/biff-reference.md` on demand.
- **biff-new** (user-invoked): Scaffold a new Biff project via the official `clj -M -e '(load-string (slurp "https://biffweb.com/new.clj"))'` installer. Confirms namespace and project directory, then verifies that `clj -M:dev dev` starts the app.
- **biff-deploy** (user-invoked): Deployment workflow for the three Biff paths: managed Ubuntu VPS via `server-setup.sh` and `clj -M:dev deploy`, Docker image build, or standalone uberjar. Includes post-deploy verification via `clj -M:dev logs` and a smoke check against the live domain.
