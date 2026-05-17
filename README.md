# biff-skills

[Biff](https://biffweb.com/) web framework skills packaged as an [APM](https://github.com/microsoft/apm) plugin. One install deploys the full set to every runtime APM supports: Claude Code, Codex, OpenCode, Cursor, Copilot, Gemini, and Windsurf.

Skills follow the [Agent Skills](https://agentskills.io) open standard. The `biff` style and conventions skill auto-triggers from conversation context; the rest appear as slash commands.

## Companion packages

This package focuses on Biff-specific conventions and layers on top of [clojure-skills](https://github.com/brackendev/clojure-skills). Install both together so the general Clojure style guide, `/clj-check` quality pipeline, and `/clj-smells-review` are available for the underlying Clojure code; this package adds Biff-aware scaffolding, framework guidance, and deployment. For ClojureDart / Flutter work, install [clojuredart-skills](https://github.com/brackendev/clojuredart-skills).

| Package | Focus |
|---------|-------|
| [clojure-skills](https://github.com/brackendev/clojure-skills) | Idiomatic Clojure style, scaffolding, quality checks, and code review. Required companion. |
| [biff-skills](https://github.com/brackendev/biff-skills) (this package) | Biff web framework: scaffolding, conventions, deployment. |
| [clojuredart-skills](https://github.com/brackendev/clojuredart-skills) | ClojureDart / Flutter equivalents for the Clojure toolkit. |

## Install

Install [APM](https://github.com/microsoft/apm) first if you don't already have it. Then, in a project:

```bash
apm install brackendev/biff-skills --target all
```

Globally for your user account:

```bash
apm install brackendev/biff-skills -g --target all
```

Update later with `apm update [-g]`. Remove with `apm uninstall brackendev/biff-skills [-g]`. A local filesystem path can replace the shorthand at either scope.

## Requirements

- [Clojure CLI](https://clojure.org/guides/install_clojure) and Java 17 or higher for any skill in this package.
- [clojure-skills](https://github.com/brackendev/clojure-skills) installed alongside, for `/clj-check`, `/clj-smells-review`, and the general `clojure` style guide.

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
| **biff** | `com.biffweb` namespace, `biff/submit-tx`, `biff/q`, `biff/lookup`, `biff/form`, `doc-schema`, `:db/doc-type`, `:db.id/*`, `resources/config.edn`, `config.env`, `dev/repl.clj`, `dev/tasks.clj`, `src/com/*/app.clj`, `home.clj`, `worker.clj`, `schema.clj`, `hx-*` attributes in Clojure code, `use-jetty`, `use-xtdb`, `use-chime`, `use-beholder`, `biff/submit-job`, `biff/authentication-module`, `clj -M:dev dev`, `clj -M:dev deploy`, `server-setup.sh`, or any mention of Biff or biffweb. Covers the module + system-map architecture, project layout, routes, transactions and queries against XTDB with Malli schema, htmx patterns, authentication, scheduled tasks, transaction listeners, queues, and REPL workflow. Defers to the `clojure` skill (in [clojure-skills](https://github.com/brackendev/clojure-skills)) for general style. |

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).

## License

MIT. See [LICENSE](LICENSE).
