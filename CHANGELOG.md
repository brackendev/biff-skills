# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [0.2.11] - 2026-10-31

### Changed

- The Honest Code section of `biff-lenses` cites the Honest Code Principles by number (P01-P25), which the code-lenses `honest-code` skill now follows, instead of the construct names code-lenses no longer uses.

## [0.2.10] - 2026-10-29

### Added

- `biff-rules` hook: delivers the condensed BIFF-1 through BIFF-6 Iron Laws to the agent at session start, after `/clear` or a compaction, and on the first matching edit when the session started outside a matching project, with an instruction to load the `biff` skill along with the `clojure` and `clojure-jvm` skills. Verified only under Claude Code. Copilot and Cursor receive no copy.
- Hook test suites in `tests/hooks/` (`bb tests/hooks/run.bb`), and empty `copilot-hooks.json` and `cursor-hooks.json` files.

## [0.2.9] - 2026-10-29

### Changed

- The Install section requires APM 0.34.0 and installs it from PyPI with `uv tool install apm-cli==0.34.0` after removing any Homebrew copy. APM 0.35.0 is not supported because it can delete installed skill files ([microsoft/apm#3372](https://github.com/microsoft/apm/issues/3372)).

## [0.2.8] - 2026-10-25

### Added

- The `biff` skill covers the SQLite additions in Biff 2.0.3: the `com.biffweb.sqlite.schema` column helpers (`s/pkey`, `s/text`, `s/ref`, and others), the `:default` column option, `execute-one` and its `:biff.sqlite.fx/execute-one` effect, and the generated resolvers for unique columns and for `:backref` and `:backref-many` reverse joins. It tells the agent to declare backrefs instead of writing join resolvers.
- The skill covers the `:?ns/attr` shorthand for optional query attributes, the 400 response for a required `:request/*` attribute that cannot be resolved, `biff.graph/uml` and the `graph.mmd` file that the starter writes, the built-in `:biff.fx/swap!` and `:biff.fx/http` effects, `:biff.fx/seed`, the `nrepl` task, and the optional `:biff.sqlite/litestream-replica-path` key.
- The skill describes the agent skills that `clojure -M:run init` writes into `.agents/skills/` (`biff-dev`, `biff-code-quality`, `biff-code-review`, `biff-spec-review`, and `biff-spec-dev`). It tells the agent not to edit them and to use `biff-code-review` for full convention reviews.
- The `biff` skill warns that a misspelled effect key runs nothing and raises no error, and that `init` repeats its namespace prompt forever when it gets no input. It shows how to pipe the answers to `init`. It also warns that the starter's `start` returns nil, so its `-main` starts nREPL on a random port, and that the `graph.mmd` file that `start` writes blocks deploys until it is committed or ignored.
- `/biff-deploy` lists the files that block a deploy (`graph.mmd` and the files a project-scope APM install adds). Before a soft deploy, it checks that `start` returns the system map, because the soft deploy and `prod-nrepl` connect to the configured nREPL port. It also lists the Litestream backup keys in `config.prod.env`.

### Changed

- All skills target Biff 2.0.3 and the matching `biff-starter` project.
- Path examples and rules name path parameters after their entity (`:post-id`, not `:id`), as the upstream review rules require.
- The skills call the module folder for resolvers that return hiccup `uicomp/`, the name that the upstream review rules use.
- The README requirements state that the skills need Java 21 or later and a Biff 2 project created from `biff-starter`.
- Every command that the skills tell the agent to run starts with `clojure` instead of `clj`, for example `clojure -M:run dev` and `clojure -M:run deploy`, matching Biff's own agent skills and continuous integration workflow. The `clj` launcher exits with an error when `rlwrap` is not installed. The skills keep `clj` only where they quote an upstream file: the `ExecStart` line that `prod-setup` writes and the build line of the starter `Dockerfile`.
- The `biff` skill and the README explain how `/clj-fix` from clojure-skills differs from Biff's own tasks in a Biff project. `/clj-fix` formats only the source and test directories with whichever cljfmt it finds, does not add the blank lines that Biff's `format` task inserts, lints with extra warnings turned on, and skips its test and dry steps. The skill tells the agent to use `clojure -M:run code-quality` and to run it after any `/clj-fix` run.
- The `biff` skill activates for the `-M:run` tasks whether they start with `clojure` or `clj`.
- The `biff` skill's trigger description fits the 1,024-character limit of the Agent Skills specification. It keeps every trigger term and leaves out the summary of covered topics, which the skill body already describes.

### Fixed

- `/biff-deploy`, the README, and `CONVENTIONS.md` say that a soft deploy loads every project source file into the running app. They said it loaded only the changed files.
- Without captcha keys, the app starts, and every sign-in request then fails with a 500. The skills said startup fails.
- Without a cookie secret, no warning is logged. The skill said a warning appears.
- The `biff-lenses` skill uses the `:request/*` namespace for read-authorization resolvers. It used `:params/*`.
- The destructive-migration warning names the `-- Skipped:` log line that sqldef prints. The skill said the skip was silent.
- The project layout lists `.agents/skills/` and `graph.mmd`. It listed an `AGENTS.md` file that the starter does not have.
- The CSRF notes say that biff.ring also rejects requests whose `Sec-Fetch-Site` is `same-site` or `cross-site`.
- The XTDB notes give the Java range for biff.xtdb as 21 or later but earlier than 25, and they say that writes through biff.xtdb trigger `on-tx` as soon as XTDB indexes them.
- The README describes this package as framework guidance and deployment. It still listed scaffolding, which release 0.2.6 removed.
- The README names `/clj-smells-fix`, the current name of the clojure-skills smells command.
- The `biff` skill and `biff-lenses` say when Biff's `format` task inserts a blank line: between adjacent pairs in a map, a binding vector, or a `cond` when either pair puts its key and value on separate lines. They said it separated every multi-line pair, but a pair whose value alone spans several lines gets no blank line.

### Removed

- The warning that the biff.sqlite README uses a `:string` column type is removed, because the README uses `:text`.
- Guidance for older Biff versions is removed: Biff 1 migration from the `biff-lenses` skill, the Biff 1 install note from the README, and the Java version history from the skills and the README.

## [0.2.7] - 2026-10-25

### Added

- The `biff` skill opens with Iron Laws (BIFF-1 through BIFF-6): numbered rules that prevent security holes and lost changes. The agent does not write code that breaks a law, and when existing code breaks one, it reports the law ID, the location, and the correct pattern, then asks before changing code outside the task. Key Rules that guarded against bugs or security holes moved into the laws, and the remaining Key Rules are renumbered.

### Fixed

- The read-authorization guidance uses the `:request/*` join namespace that the starter's review rules require, instead of `:params/*`.

## [0.2.6] - 2026-10-25

### Changed

- The Companion packages table describes biff-skills as covering Biff 2 conventions and deployment, because that package no longer scaffolds projects.

### Removed

- The `/biff-new` command is removed. Start a new Biff 2 project from the official [biff-starter](https://github.com/jacobobryant/biff-starter) repository and run `clj -M:run init`. The `biff` skill still describes the starter layout and the `init` task.

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
