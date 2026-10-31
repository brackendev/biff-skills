# biff-skills

[Biff](https://biffweb.com/) 2 web framework skills packaged as an [APM](https://github.com/microsoft/apm) plugin. One install deploys the full set to every runtime in APM's default target set: Claude Code, Codex, OpenCode, Cursor, Copilot, Gemini, Windsurf, Kiro, and Grok Build. Antigravity is supported by naming it explicitly with `--target antigravity`.

Skills follow the [Agent Skills](https://agentskills.io) open standard. The `biff` style and conventions skill auto-triggers from conversation context. The other skill, `/biff-deploy`, is a user-only slash command. Runtimes differ in how they offer it: Codex lists it under `/skills` and runs it as `$name`, OpenCode attaches it with `@name`, and Gemini CLI does not read the user-only setting, so its agent can start it.

## Companion packages

Eight sibling APM packages. This package layers on top of [clojure-skills](https://github.com/brackendev/clojure-skills) and [clojure-jvm-skills](https://github.com/brackendev/clojure-jvm-skills). Install all three together for Biff projects: `clojure-skills` covers cross-dialect style, `clojure-jvm-skills` covers Java interop, refs / agents / STM, `with-open`, JVM-typed exceptions, and the Clojure CLI / `tools.build` workflow that Biff runs on, and this package adds Biff framework guidance and deployment. For ClojureScript work, install [clojurescript-skills](https://github.com/brackendev/clojurescript-skills). For Electric or Fulcro full-stack work, install [electric-skills](https://github.com/brackendev/electric-skills) or [fulcro-skills](https://github.com/brackendev/fulcro-skills) instead of this package. For ClojureDart / Flutter work, install [clojuredart-skills](https://github.com/brackendev/clojuredart-skills) instead of (or in addition to) this package. For Babashka scripts and `bb.edn` tasks, install [babashka-skills](https://github.com/brackendev/babashka-skills).

| Package | Focus | Layers on |
|---------|-------|-----------|
| [clojure-skills](https://github.com/brackendev/clojure-skills) | Host-neutral [Clojure](https://clojure.org/) family baseline: style, naming, threading, collections, atoms, dispatch, formatting, namespaces, and testing. Triggers on `.clj`, `.cljs`, `.cljc`, `.cljd`. | — |
| [clojure-jvm-skills](https://github.com/brackendev/clojure-jvm-skills) | [Clojure](https://clojure.org/) on the JVM: Java interop, refs / agents / STM, `with-open`, JVM-typed exceptions, `alter-var-root`, and the Clojure CLI / `tools.build` / `clj-kondo` / `cljfmt` / `test-runner` / nREPL workflow. | `clojure-skills` |
| [clojurescript-skills](https://github.com/brackendev/clojurescript-skills) | [ClojureScript](https://clojurescript.org/) style and tooling: JavaScript interop, externs inference, macro stage separation, `catch :default`, JS-flavored numbers and truthiness, and the `cljs.main` workflow. Triggers on `.cljs`, `.cljc` compiled to JS, `shadow-cljs.edn`, `figwheel-main.edn`. | `clojure-skills` |
| [babashka-skills](https://github.com/brackendev/babashka-skills) | [Babashka](https://babashka.org/) scripting host: built-in namespaces (`babashka.fs`, `babashka.process`, `babashka.http-client`), `bb.edn` tasks, subprocess exit checks, threads and process exit, records and types without Java classes, testing, pods, and packaging. Triggers on `.bb`, `bb.edn`. | `clojure-skills` |
| [biff-skills](https://github.com/brackendev/biff-skills) (this package) | [Biff](https://biffweb.com/) 2 web framework on the JVM: conventions and deployment. | `clojure-skills` + `clojure-jvm-skills` |
| [electric-skills](https://github.com/brackendev/electric-skills) | [Electric](https://github.com/hyperfiddle/electric) v3 full-stack reactive framework: `e/defn` components, `e/client` / `e/server` site transfer, `e/for` over `e/diff-by`, `e/watch` state, `e/Token` transactions, `e/Offload`, the `electric-dom3` and `electric-forms5` APIs, and the single-JVM shadow-cljs workflow. Triggers on `hyperfiddle.electric3`, `hyperfiddle.electric-dom3`, `e/defn`, `e/client`, `e/server`, and `com.hyperfiddle/electric`. | `clojure-skills` + `clojurescript-skills` + `clojure-jvm-skills` |
| [fulcro-skills](https://github.com/brackendev/fulcro-skills) | [Fulcro](https://github.com/fulcrologic/fulcro) 3 full-stack framework: `defsc` components, idents and the normalized client database, mutations, `df/load!`, dynamic routing, forms, UI state machines, Fulcro Inspect, and the Pathom 3 server. Triggers on `com.fulcrologic.fulcro.*`, `com.fulcrologic.rad.*`, `com.wsscode.pathom3.*`, `defsc`, `defmutation`, `defrouter`, `df/load!`, and ident vectors. | `clojure-skills` + `clojurescript-skills` + `clojure-jvm-skills` |
| [clojuredart-skills](https://github.com/brackendev/clojuredart-skills) | [ClojureDart](https://github.com/Tensegritics/ClojureDart) on [Flutter](https://flutter.dev/): Dart interop, type hints, `cljd.flutter` directives, async, FFI, REPL, and the Flutter project workflow. Triggers on `.cljd`, `cljd.flutter`. | `clojure-skills` |

## Install

Install [APM](https://github.com/microsoft/apm) 0.34.0 first. Homebrew offers only the newest release, so remove a Homebrew copy with `brew uninstall apm` and install from PyPI with [uv](https://docs.astral.sh/uv/getting-started/installation/): `uv tool install apm-cli==0.34.0`. APM 0.35.0 is not supported because it can delete installed skill files ([microsoft/apm#3372](https://github.com/microsoft/apm/issues/3372)). Then, in a project:

```bash
apm install brackendev/biff-skills --target all
apm install brackendev/clojure-jvm-skills --target all
apm install brackendev/clojure-skills --target all
```

Globally for your user account:

```bash
apm install brackendev/biff-skills -g --target all
apm install brackendev/clojure-jvm-skills -g --target all
apm install brackendev/clojure-skills -g --target all
```

Update later with `apm update [-g]`. Remove with `apm uninstall brackendev/biff-skills [-g]`. A local filesystem path can replace the shorthand at either scope.

## Requirements

- [Clojure CLI](https://clojure.org/guides/install_clojure) and Java 21 or higher for any skill in this package.
- A Biff 2 project. The skills describe Biff 2 and the project layout of the official [biff-starter](https://github.com/jacobobryant/biff-starter) repository.
- [clojure-skills](https://github.com/brackendev/clojure-skills) installed alongside, for the host-neutral baseline plus `/clj-fix` and `/clj-smells-fix`.
- [clojure-jvm-skills](https://github.com/brackendev/clojure-jvm-skills) installed alongside, for JVM-specific Clojure guidance the `biff` skill defers to (Java interop, JVM exceptions, refs / agents / STM, `alter-var-root`, the CLI workflow).
- [Babashka](https://babashka.org) (`bb`) to run the `biff-rules` hook. Without it, the hook does nothing and prints a notice on standard error.

## Command guide

A quick guide to every slash command. The detailed entries under [Skills](#skills) cover arguments and examples.

| Command | Use it when | What it does |
|---------|-------------------|--------------|
| `/biff-deploy` | Deploying a Biff app | Provisions and deploys to an Ubuntu server, hot-reloads a deployment, or builds an uberjar, then verifies with logs and a smoke check |

For lint, format, and test, run the project's own `clojure -M:run code-quality` task, which the starter's continuous integration workflow also runs before it fails on any changed file. The `/clj-fix` command from [clojure-skills](https://github.com/brackendev/clojure-skills) gives different results in a Biff 2 project. It formats only the source and test directories with whichever cljfmt it finds on `PATH` or as a Clojure CLI tool, and it does not add the blank lines that Biff's `format` task inserts between pairs written across several lines. Biff's own task formats every tracked Clojure and EDN file with the cljfmt version that Biff pins. `/clj-fix` also skips its test step, because Biff 2 projects define no `:test` alias and run Kaocha through `clojure -M:run test`.

## Skills

User-invocable skills follow the conventions documented in [CONVENTIONS.md](CONVENTIONS.md): natural-language keywords and bare paths, no `--name` flags other than `--report`, mutation as the default, and a shared scope vocabulary.

### Deployment

#### `/biff-deploy [vps|soft|uberjar] [--report]`

Deploy a Biff app with the project's `biff.tasks` tasks. The default VPS path provisions a fresh Ubuntu server once with `clojure -M:run prod-setup` and then deploys with `clojure -M:run deploy`, which pushes the current git branch and restarts the app. The `soft` path loads every project source file into the running app instead of restarting it, and the `uberjar` path builds `target/jar/app.jar` for other hosting. The skill verifies a deploy with `clojure -M:run prod-logs` and a smoke check against the configured domain, and it asks for confirmation before each command that touches the server. Pass `--report` to print the planned commands, the resolved domain, the branch and worktree state, and the uploaded untracked files without writing or running anything.

```bash
/biff-deploy
/biff-deploy soft
/biff-deploy uberjar
/biff-deploy --report
```

### Auto-triggered

These skills activate from conversation context. They cannot be invoked directly.

| Skill | Triggers |
|-------|----------|
| **biff** | `com.biffweb.*` namespaces, `biff.core/start`, `:biff.core/id`, `start-order`, `modules.clj`, `defpipeline`, `defmachine`, `defresolver`, `:biff.graph.fx/query`, `:biff.sqlite.fx/*` effects, `authorized-write`, `defpath`, `:biff.ring/routes`, `biff.datastar`, `signals-json`, `signal-name`, Datastar `data-on:` and `@post(` attributes in Clojure code, `:biff.datastar/tab-id`, `:biff.background/tasks` and `:biff.background/queues`, `submit-jobs`, `biff.auth/module`, `resources/schema.sql`, `resources/config.edn`, `config.env`, `config.prod.env`, `repl.clj`, the `-M:run` tasks run with `clojure` or `clj` (`dev`, `code-quality`, `format`, `lint`, `test`, `deploy`, `prod-setup`), `agent-refresh`, `biff-starter`, or any mention of Biff or biffweb. Covers the `biff.core` module system and project layout, routes through `biff.ring`, effects as data with `biff.fx`, `biff.graph` resolvers, SQLite schema, migrations, and authorized writes, Datastar pages and actions, authentication, background jobs, and the task workflow. Defers to the `clojure` skill in [clojure-skills](https://github.com/brackendev/clojure-skills) for host-neutral style and to the `clojure-jvm` skill in [clojure-jvm-skills](https://github.com/brackendev/clojure-jvm-skills) for Java interop, JVM exception handling, nREPL, clj-kondo, cljfmt, and tools.build. The skill opens with Iron Laws (BIFF-1 through BIFF-6), rules that prevent security holes and lost changes. The agent does not write code that breaks them, and it reports a violation it finds in existing code before changing it. |

## Hooks

The package includes one hook, `biff-rules`.

### `biff-rules`

Agents seldom load an auto-triggered skill on their own, so this hook gives the agent the BIFF-1 through BIFF-6 Iron Laws of the `biff` skill as text. In a `deps.edn` that names `com.biffweb`, it puts the condensed laws into the agent's context when a session starts, and again after `/clear` or a compaction, with an instruction to load the `biff` skill along with the `clojure` and `clojure-jvm` skills. When the session starts outside such a project, for example in a parent directory, the hook delivers the laws on the first edit of a `.clj`, `.cljs`, `.cljc`, or `.edn` file in a matching project instead, once per session. The hook never edits a file. Without Babashka it does nothing and prints a notice on standard error.

The hook has been run only under Claude Code, where it registers `SessionStart` and `PostToolUse`. The package ships empty Copilot and Cursor hook files, so APM writes no Copilot hook and no native Cursor hook. APM also installs the generic copy for Codex, Gemini CLI, Kiro, and Windsurf, where its effect has not been tested. OpenCode and Grok Build have no hook support.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).

## License

MIT. See [LICENSE](LICENSE).
