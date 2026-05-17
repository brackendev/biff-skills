# biff-skills

[Biff](https://biffweb.com/) web framework skills packaged as an [APM](https://github.com/microsoft/apm) plugin. One install deploys the full set to every runtime APM supports: Claude Code, Codex, OpenCode, Cursor, Copilot, Gemini, and Windsurf.

Skills follow the [Agent Skills](https://agentskills.io) open standard. The `biff` style and conventions skill auto-triggers from conversation context; the rest appear as slash commands.

## Companion packages

Five sibling APM packages. This package layers on top of [clojure-skills](https://github.com/brackendev/clojure-skills) and [clojure-jvm-skills](https://github.com/brackendev/clojure-jvm-skills). Install all three together for Biff projects: `clojure-skills` covers cross-dialect style, `clojure-jvm-skills` covers Java interop, refs / agents / STM, `with-open`, JVM-typed exceptions, and the Clojure CLI / `tools.build` workflow that Biff runs on, and this package adds Biff-aware scaffolding, framework guidance, and deployment. For ClojureScript work, install [clojurescript-skills](https://github.com/brackendev/clojurescript-skills); for ClojureDart / Flutter work, install [clojuredart-skills](https://github.com/brackendev/clojuredart-skills) instead of (or in addition to) this package.

| Package | Focus | Layers on |
|---------|-------|-----------|
| [clojure-skills](https://github.com/brackendev/clojure-skills) | Host-neutral Clojure family baseline (style, naming, threading, collections, atoms, dispatch, formatting, namespaces, testing). Triggers on `.clj`, `.cljs`, `.cljc`, `.cljd`. | — |
| [clojure-jvm-skills](https://github.com/brackendev/clojure-jvm-skills) | JVM-specific Clojure (Java interop, refs / agents / STM, `with-open`, JVM-typed exceptions, `alter-var-root`, Clojure CLI / `tools.build` / `clj-kondo` / `cljfmt` / `test-runner` / nREPL workflow). | `clojure-skills` |
| [clojurescript-skills](https://github.com/brackendev/clojurescript-skills) | ClojureScript-specific style (JavaScript interop, externs inference, macro stage separation, `catch :default`, JS-flavored numbers and truthiness, the `cljs.main` workflow). Triggers on `.cljs`, `.cljc` compiled to JS, `shadow-cljs.edn`, `figwheel-main.edn`. | `clojure-skills` |
| [biff-skills](https://github.com/brackendev/biff-skills) (this package) | [Biff](https://biffweb.com/) web framework on the JVM: scaffolding, conventions, deployment. | `clojure-skills` + `clojure-jvm-skills` |
| [clojuredart-skills](https://github.com/brackendev/clojuredart-skills) | ClojureDart on Flutter: Dart interop, type hints, `cljd.flutter` directives, async, FFI, REPL, Flutter project workflow. Triggers on `.cljd`, `cljd.flutter`. | `clojure-skills` |

## Install

Install [APM](https://github.com/microsoft/apm) first if you don't already have it. Then, in a project:

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

- [Clojure CLI](https://clojure.org/guides/install_clojure) and Java 17 or higher for any skill in this package.
- [clojure-skills](https://github.com/brackendev/clojure-skills) installed alongside, for the host-neutral baseline plus `/clj-check` and `/clj-smells-review`.
- [clojure-jvm-skills](https://github.com/brackendev/clojure-jvm-skills) installed alongside, for JVM-specific Clojure guidance the `biff` skill defers to (Java interop, JVM exceptions, refs / agents / STM, `alter-var-root`, the CLI workflow).

## Skills

### Scaffolding and quality

#### `/biff-new [project-name]`

Scaffold a new Biff project using the official `clj -M -e '(load-string ...)'` installer. Confirms the namespace and project directory before running, then verifies the first `clj -M:dev dev`.

```bash
/biff-new my-app
```

For lint / format / test / duplicate-form checks, use `/clj-check` from [clojure-skills](https://github.com/brackendev/clojure-skills). Pass `dev` as a lint path so the Biff `dev/repl.clj` and `dev/tasks.clj` helpers are covered.

#### `/biff-deploy [vps|docker|uberjar]`

Deploy a Biff app. Defaults to the managed Ubuntu VPS path (`server-setup.sh` + `clj -M:dev deploy`). Also supports Docker image and standalone uberjar builds. Verifies the deploy with `clj -M:dev logs` and a smoke check against the configured domain.

```bash
/biff-deploy
/biff-deploy docker
/biff-deploy uberjar
```

### Auto-triggered

These skills activate from conversation context. They cannot be invoked directly.

| Skill | Triggers |
|-------|----------|
| **biff** | `com.biffweb` namespace, `biff/submit-tx`, `biff/q`, `biff/lookup`, `biff/form`, `doc-schema`, `:db/doc-type`, `:db.id/*`, `resources/config.edn`, `config.env`, `dev/repl.clj`, `dev/tasks.clj`, `src/com/*/app.clj`, `home.clj`, `worker.clj`, `schema.clj`, `hx-*` attributes in Clojure code, `use-jetty`, `use-xtdb`, `use-chime`, `use-beholder`, `biff/submit-job`, `biff/authentication-module`, `clj -M:dev dev`, `clj -M:dev deploy`, `server-setup.sh`, or any mention of Biff or biffweb. Covers the module + system-map architecture, project layout, routes, transactions and queries against XTDB with Malli schema, htmx patterns, authentication, scheduled tasks, transaction listeners, queues, and REPL workflow. Defers to the `clojure` skill in [clojure-skills](https://github.com/brackendev/clojure-skills) for host-neutral style and to the `clojure-jvm` skill in [clojure-jvm-skills](https://github.com/brackendev/clojure-jvm-skills) for Java interop, JVM exception handling, refs / agents / STM, `alter-var-root`, and the Clojure CLI / `tools.build` / `clj-kondo` / `cljfmt` / `test-runner` / nREPL workflow. |

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).

## License

MIT. See [LICENSE](LICENSE).
