# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [0.2.5] - 2026-10-24

### Added

- The Companion packages table lists [electric-skills](https://github.com/brackendev/electric-skills), the new companion package for the [Electric](https://github.com/hyperfiddle/electric) v3 full-stack framework, and `CONTRIBUTING.md` cites its Hyperfiddle lavender brand color (`#B7A4C2`).

### Changed

- The Companion packages table names the major version each framework package targets: Biff 2, Electric v3, and Fulcro 3. The skills in those packages do not apply to other major versions.
- `CONTRIBUTING.md` asks for a changelog entry and a version increment for every change to a published file, including documentation-only changes, which matches the release history. It asked for entries only for user-facing changes.

### Fixed

- The `CONTRIBUTING.md` Layout table names the local `AGENTS.md`, `GEMINI.md`, and `TODO.md` files. It named `CLAUDE.md`, which the package does not use.
- The package description in `apm.yml`, which APM shows at install, names Biff 2 and lists what the skills cover. It still described the package from before the Biff 2 rewrite and claimed quality checks, for which the package has no command. Format, lint, and test run through the project's `clj -M:run code-quality` task.
- `CONVENTIONS.md` lists `biff-lenses` with `biff` as the model-invoked skills that have no argument surface. It named only `biff`.
- The README explains how Codex, OpenCode, and Gemini CLI offer the commands: Codex runs a skill as `$name` from its `/skills` list, OpenCode attaches one with `@name`, and Gemini CLI does not read the user-only setting. It said every command appears as a slash command and that user-only commands run only when typed.

## [0.2.4] - 2026-10-24

### Changed

- The `biff` skill's Key Rules carry the symptoms that its Gotchas repeated: a full document returned for an SSE re-render nests `<html>` inside the page, signals without `__ifmissing` erase what the user is typing, extra SQL belongs in `extra-init-sql`, and background jobs are not persisted or retried. `/biff-deploy` and `/biff-new` fold the Gotchas that repeated their steps into those steps.

## [0.2.3] - 2026-10-24

### Removed

- OpenCode 1 is no longer supported. The repository no longer keeps a copy of each `SKILL.md` under `.opencode/skills/` or a per-skill allow list in `opencode.jsonc`. `.opencode/skills` is now a symlink to `.apm/skills/` instead, so an OpenCode 2 session started anywhere in the repository loads the canonical skills together with their supporting files. The `.opencode/package.json` file, which pinned the OpenCode 1 plugin package, is removed as well. APM deploys the same files as before.

### Changed

- The `CONTRIBUTING.md` acceptance steps list the skills OpenCode discovers with `opencode api GET "/api/skill?location[directory]=$PWD"`, because OpenCode 2 has no `--pure` flag or `debug skill` command.

## [0.2.2] - 2026-10-24

### Fixed

- The `CONTRIBUTING.md` acceptance steps install with `apm install --target all` in the clean temporary project. On APM 0.34.0, a bare `apm install` in that project stops with "Multiple harnesses detected" because the steps create several runtime folders first. `apm update` reuses the targets that the install records in the project's `apm.yml`.

## [0.2.1] - 2026-10-17

### Fixed

- `CONTRIBUTING.md` no longer says that omitting `user-invocable` makes a skill model-only. Claude Code defaults `user-invocable` to `true`, so a skill that omits the field also appears as a slash command. A model-only skill sets `user-invocable: false`.

## [0.2.0] - 2026-10-04

### Breaking

- Every skill now targets Biff 2 (written against Biff 2.0.2 and the `biff-starter` project) and no longer describes Biff 1.x. Biff 2 replaces the single `com.biffweb` namespace with separate libraries, makes SQLite the default database, replaces htmx with Datastar, and moves project tasks from `clj -M:dev` to `clj -M:run`. Projects that still run Biff 1.x should stay on release 0.1.11 of this package.
- Java 21 is now the minimum, matching Biff 2.0.1 and later.

### Changed

- The `biff` skill and its reference now cover the `biff.core` module system and start order, the starter's `app/`, `model/`, and `lib/` layout, Reitit routes and `defpath` through `biff.ring`, effects as data with `biff.fx`, `biff.graph` resolvers, SQLite columns, migrations, and authorized writes through `biff.sqlite`, Datastar pages, actions, and tab state, `biff.background` tasks and queues, `biff.authenticate`, `biff.admin`, the `clj -M:run` task workflow, and XTDB 2 through `biff.xtdb` as an alternative database. The skill also lists places where the upstream documentation and code disagree.
- `/biff-new` now clones the official `biff-starter` repository and runs `clj -M:run init` with the confirmed namespace and git-history choice, then verifies that `clj -M:run dev` serves the app. It also documents how to recover when macOS rejects the downloaded Tailwind binary.
- `/biff-deploy` now uses the Biff 2 tasks: `clj -M:run prod-setup` to provision an Ubuntu server, `clj -M:run deploy` to push the current branch and restart, and `clj -M:run uberjar` to build `target/jar/app.jar`. A new `soft` mode runs `clj -M:run deploy --soft`, which evaluates changed files in the running app without a restart.
- The `biff-lenses` skill now translates the code-lenses philosophies into Biff 2 patterns. It leaves the general Clojure translations to `clojure-lenses` instead of restating them.
- The `biff` skill now points format, lint, and test work to `clj -M:run code-quality`, because `/clj-fix` from `clojure-skills` expects aliases that Biff 2 projects do not define.

### Removed

- The `docker` mode of `/biff-deploy` is removed. Biff 2 has no Docker support, and the starter's `Dockerfile` calls a `:dev` alias that the starter does not define.

## [0.1.11] - 2026-09-26

### Fixed

- The `biff-lenses` skill no longer tells the agent to rely on queue retries after swallowing an exception. Biff queues do not retry a job that throws, which the `biff` skill and its reference already stated.
- The `biff-lenses` skill now describes `:db.op/upsert` correctly: it matches on the given attribute values and updates that document, or creates a new one, instead of overwriting.
- The `biff-deploy` and `biff-new` skills no longer point the agent at a `CONVENTIONS.md` file. That file ships only with this package repository, so it is absent from the Biff project where the skills run.

### Changed

- The `biff-lenses` skill now triggers when a code-lenses review or refactor runs against Biff code, rather than on every Biff symbol that also triggers the `biff` skill.

## [0.1.10] - 2026-09-16

### Changed

- The README's runtime sentence now names Grok Build after Kiro. APM 0.28.0 added `grok-build` as a canonical target that `apm install --target all` includes and that APM auto-detects from a `.grok/` directory, so the default target set now has nine runtimes. Grok Build keeps a target-native skill directory (`.grok/skills/` at project scope, `~/.grok/skills/` at user scope), like Claude Code and Kiro, rather than the shared `.agents/skills/` directory.

## [0.1.9] - 2026-07-29

### Changed

- The README's runtime sentence now scopes its list to APM's default target set rather than claiming every runtime APM supports. APM 0.26.0 also supports Antigravity, IntelliJ, and several experimental runtimes, none of which `apm install --target all` includes. The eight runtimes named are unchanged, and the sentence adds that Antigravity works when named explicitly with `--target antigravity`.

## [0.1.8] - 2026-07-29

### Removed

- The package manifest no longer declares the top-level `target: all` field. The APM manifest schema deprecates the `all` value: a parser treats the field as though it were absent and falls through to the `--target` flag or filesystem auto-detection, and the value is scheduled to become a hard parse error in a future APM release. Removing the field makes that fall-through behavior permanent. Installation behavior is unchanged, because APM already resolved targets by auto-detection rather than from this field. The separate `compilation.target` setting is not affected.

## [0.1.7] - 2026-06-15

### Changed

- Add Kiro to the README's runtime list. APM 0.20.0 added Kiro as a first-class install target included in `apm install --target all`, so the README now lists it alongside Claude Code, Codex, OpenCode, Cursor, Copilot, Gemini, and Windsurf.

## [0.1.6] - 2026-05-28

### Added

- `CONVENTIONS.md` gains Rule 4: vendored and generated paths are excluded by default from mutating skills that walk the workspace. The rule sits alongside the existing three rules (renamed from "The three rules" to "The four rules"). Two filters apply together (`.gitignore` matches plus a hardcoded floor of dependency directories, build outputs, and lock files). The override rides on Rule 1's existing `<path>` `<glob>` grammar; no new flag is introduced. No skill in this plugin discovers candidate files from the workspace today: both `/biff-new` and `/biff-deploy` are exempt because their target sets are defined by explicit project operations. The rule is recorded so any future fix-style skill honors the same contract as its companion packages.

## [0.1.5] - 2026-05-26

### Fixed

- Quote the YAML `description` and `argument-hint` frontmatter in all skills that had unquoted values. Prevents potential YAML misinterpretation of special characters (angle brackets in `argument-hint`).

## [0.1.4] - 2026-05-20

### Changed

- `CONTRIBUTING.md` is aligned to the family-wide structural template. A `CONVENTIONS.md` row is added to the Layout table, and the H3 "Argument grammar, scope, and mutation" subsection is collapsed back into the parent H2 Skill conventions section (its content moves to the canonical pointer paragraph that now opens that section).

## [0.1.3] - 2026-05-20

### Added

- New `biff-lenses` skill (model-invoked, auto-triggered alongside the [code-lenses](https://github.com/brackendev/code-lenses) plugin). Translates the four default code-lenses philosophies (grug, Honest Code, Tidy First, Parse Don't Validate) and the two opt-in philosophies (APOSD, Legacy Code) into Biff-specific patterns. Records only the Biff-specific deltas on top of `clojure-lenses`: XTDB transactions through `biff/submit-tx`, Malli schema enforcement via `:db/doc-type` and `doc-schema`, the module + system-map architecture, htmx response patterns, the authentication module, scheduled tasks via `use-chime`, transaction listeners, queues via `biff/submit-job`, and the live REPL workflow. Closes the Clojure-family gap where `clojure-skills`, `clojurescript-skills`, `clojuredart-skills`, and `fulcro-skills` all ship a `*-lenses` companion but `biff-skills` did not.

### Changed

- Skill bodies and documentation that reference the sibling `clojure-skills` quality pipeline now point at `/clj-fix` instead of `/clj-tidy`, matching the rename in that companion package. The behavior is unchanged; only the command name moves to the family-wide noun-first canonical naming.

## [0.1.2] - 2026-05-19

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

## [0.1.1] - 2026-05-17

### Added

- The `biff` reference now documents `:db.op/upsert` (the recommended replacement for the deprecated `:db/lookup`), the `:db/op :create` operation, and the per-attribute sentinels `:db/union`, `:db/difference`, `:db/add`, `:db/default`, `:db/dissoc`, and `:db/unique`.

### Changed

- The `biff` skill now defers to both [clojure-skills](https://github.com/brackendev/clojure-skills) (host-neutral baseline) and [clojure-jvm-skills](https://github.com/brackendev/clojure-jvm-skills) (JVM-specific Clojure: Java interop, refs / agents / STM, `with-open`, JVM-typed exceptions, `alter-var-root`, Clojure CLI / `tools.build` / `clj-kondo` / `cljfmt` / `test-runner` / nREPL workflow). Install all three packages together for Biff projects.

## [0.1.0] - 2026-05-17

### Added

- Initial release. Three Biff skills packaged as an APM plugin and deployed to every runtime APM supports (Claude Code, Codex, OpenCode, Cursor, Copilot, Gemini, Windsurf). Designed to layer on top of [clojure-skills](https://github.com/brackendev/clojure-skills) rather than duplicate it.
- **biff** (model-invoked): Project layout, the module + system-map architecture, transactions and queries against XTDB with Malli schema enforcement, routing, htmx response patterns, authentication, scheduled tasks, transaction listeners, queues, and REPL workflow. Defers to the [clojure](https://github.com/brackendev/clojure-skills) skill for general Clojure style. Pulls deeper detail from `references/biff-reference.md` on demand.
- **biff-new** (user-invoked): Scaffold a new Biff project via the official `clj -M -e '(load-string (slurp "https://biffweb.com/new.clj"))'` installer. Confirms namespace and project directory, then verifies that `clj -M:dev dev` starts the app.
- **biff-deploy** (user-invoked): Deployment workflow for the three Biff paths: managed Ubuntu VPS via `server-setup.sh` and `clj -M:dev deploy`, Docker image build, or standalone uberjar. Includes post-deploy verification via `clj -M:dev logs` and a smoke check against the live domain.
