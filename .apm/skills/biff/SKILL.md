---
name: biff
description: >-
  Use when writing, editing, reviewing, or discussing Biff 2 web applications.
  Triggers: com.biffweb.* namespaces (core, config, fx, graph, sqlite, ring,
  datastar, background, authenticate, admin, tasks), biff.core/start,
  :biff.core/id, :biff.core/init, start-order, modules.clj, defpipeline,
  defmachine, defresolver, :biff.graph.fx/query, :biff.sqlite.fx/execute,
  authorized-write, defpath, :biff.ring/routes, biff.datastar, signals-json,
  signal-name, data-on: and @post( attributes in Clojure code,
  :biff.datastar/tab-id, :biff.background/tasks, :biff.background/queues,
  submit-jobs, biff.auth/module, resources/schema.sql, resources/config.edn,
  config.env, config.prod.env, #biff/env, #biff/secret, repl.clj,
  clj -M:run dev, clj -M:run code-quality, clj -M:run deploy,
  clj -M:run prod-setup, agent-refresh, biff-starter, or any mention of Biff or
  biffweb. Covers the biff.core module system, project layout, Reitit routes
  via biff.ring, biff.fx effects, biff.graph resolvers, SQLite schema and
  authorized writes, Datastar pages and actions, authentication, background
  jobs, and the task workflow.
user-invocable: false
---

# Biff

Biff 2 is a Clojure web framework built as a set of independent libraries under `com.biffweb/*`: `biff.core` (module system), `biff.config` (Aero config), `biff.fx` (effects as data), `biff.graph` (resolver graph), `biff.sqlite` (default database), `biff.ring` (Reitit, Jetty, middleware), `biff.datastar` (server-rendered reactive UI), `biff.background` (scheduled tasks and in-memory queues), `biff.authenticate` (email sign-in codes), `biff.admin` (dashboard), and the dev-only `biff.run` / `biff.tasks` task runner. `com.biffweb/defaults` bundles everything except `run` and `tasks`; `biff.xtdb` (XTDB 2) is an optional alternative to SQLite. Projects start from the `jacobobryant/biff-starter` repository. Java 21 or later is required.

This skill covers only what Biff changes. Follow the `clojure` skill from [clojure-skills](https://github.com/brackendev/clojure-skills) for general style (naming, namespaces, `defn-`, threading, Malli at boundaries, testing) and the `clojure-jvm` skill from [clojure-jvm-skills](https://github.com/brackendev/clojure-jvm-skills) for Java interop, JVM exceptions, nREPL, clj-kondo, cljfmt, and tools.build. For design-philosophy reviews, the `biff-lenses` skill layers Biff deltas onto `clojure-lenses`.

## Project Layout

The starter project (main namespace `com.example`, renamed by `clj -M:run init`) looks like this:

```
├── deps.edn                     :run alias (tasks) and :prod alias (server)
├── repl.clj                     REPL scratch space: get-ctx, refresh
├── tests.edn                    Kaocha configuration
├── AGENTS.md                    agent workflow (agent-refresh, code-quality)
├── resources
│   ├── config.edn               all config, committed; uses #biff/env and #biff/secret
│   ├── TEMPLATE.config.env      template for config.env (dev values)
│   ├── TEMPLATE.config.prod.env template for config.prod.env (uploaded as the server's config.env)
│   ├── schema.sql               generated from schema.clj; do not edit
│   ├── tailwind.css
│   └── public/js/main.js
├── src/com
│   ├── example.clj              entry point: system atom, start, stop, -main
│   └── example
│       ├── modules.clj          the modules vector and start-order
│       ├── routes.clj           shared defpath definitions
│       ├── schema.clj           SQLite columns map and authorization rules
│       ├── app/                 user-facing modules (routes and handlers)
│       ├── model/               data-model modules (biff.graph resolvers, sqlite module)
│       └── lib/                 shared functions (ui, middleware, email)
└── test/com/example_test.clj
```

`config.env` and `config.prod.env` hold secrets and are gitignored; `clj -M:run init` (run automatically by `dev`) generates them from the templates.

The starter README places files that define biff.core modules only in module folders (`app/`, `model/`, and any you add, such as `work/` for background jobs, `api/` for external API routes, or `ui_components/` for resolvers that return hiccup). Key Rule 1 covers how module namespaces depend on each other.

## Modules and the System Map

A module is a map. Library modules are created by calling a function; application modules are plain `(def module {...})` maps referenced from `modules.clj`:

```clojure
(def modules
  [{:biff.core/init {:biff.ring/on-error #'lib.ui/on-error}}
   (biff.config/module)
   (biff.ring/module)
   (biff.datastar/module)
   (biff.background/module)
   (biff.fx/module)
   (biff.graph/module)
   model.user/module
   model.sqlite/module
   model.tab/module
   app.admin/module
   app.landing/module
   app.auth/module
   app.demo/module])

(def start-order
  [:biff.config/module
   :biff.sqlite/module
   :biff.admin/module
   :biff.background/module
   :biff.ring/module])

(defonce system (atom {}))

(defn start []
  (reset! system (biff.core/start #'modules start-order)))
```

- `:biff.core/init` is a map, or a function of the modules var, whose result is merged into the system map. Init-only modules (`biff.fx/module`, `biff.graph/module`, `biff.datastar/module`, and most application modules) never appear in `start-order`.
- A module with `:biff.core/start` must have a `:biff.core/id`, and that id must appear in `start-order`. `:biff.core/stop` receives what its own start returned; stops run in reverse order.
- Libraries collect keys from every module: `:biff.ring/routes`, `:biff.graph/resolvers`, `:biff.fx/handlers`, `:biff.sqlite/columns`, `:biff.background/tasks`, `:biff.background/queues`, and `:biff.core/on-tx`.
- Handlers receive the whole system map merged with the Ring request. The starter names that argument `request` or `ctx`.

## Key Rules

1. **Keep module namespaces isolated.** Each exposes a single `module` map. Only `modules.clj` requires module namespaces, and module namespaces never require each other. Put shared helpers in `lib/` and shared paths in `routes.clj`, never in another module namespace.
2. **Reference functions and the modules vector as vars.** Use `#'modules` in `biff.core/start` and `#'fn` for functions stored in module or config maps so that file-save evaluation takes effect without a restart.
3. **Read configuration from the system map.** Never read environment variables or the global `system` atom from application code; the atom is for the REPL only. Declare values in `resources/config.edn` with `#biff/env` and wrap secrets with `#biff/secret`, then unwrap them with `force` (nil-safe), not `@`.
4. **Build every URL with `defpath`.** Call the path function with no arguments in the Reitit route vector and with arguments for `:href` and `:action`. UUID path parameters are encoded and decoded automatically.
5. **Write handlers as `defpipeline` or `defmachine`.** State functions stay pure and return effect descriptors such as `[:biff.sqlite.fx/execute {...}]`. Put effects whose result is not needed under an underscore-prefixed key (`:_`), and use `:biff.fx/seq` when effect order matters.
6. **Route user-initiated writes through `authorized-write`.** Use the `:biff.sqlite.fx/authorized-write` effect so the `authorize` function in `schema.clj` checks the diff. Use plain `execute` only for system writes such as background jobs.
7. **Authorize reads with biff.graph resolvers.** biff.sqlite has no read authorization. Use "params" resolvers that return an entity only when the current user may read it.
8. **Render each page from one handler.** Wrap content in `ui/app-page`, which returns only the `#biff-datastar-content` element when `:biff.datastar/sse-request` is true. Returning a full document on those requests nests `<html>` inside the page. Datastar re-renders the page over SSE after every database write.
9. **Return 204 from Datastar actions.** Action handlers update state and return `{:status 204}` (or `patch-signals` to change client signals). They do not render HTML.
10. **Bind form inputs to namespaced signals.** Initialize signals with `:data-signals__ifmissing` and `signals-json`, bind with `signal-name`, and read them from `:biff.datastar/signals`. Without `__ifmissing`, every server push overwrites the signals and erases what the user is typing. Avoid signals for anything other than form inputs; store UI state the render needs in tab state keyed by `:biff.datastar/tab-id`.
11. **Keep page rendering fast.** Every write re-renders open pages. Move expensive queries into an action that stores results in tab state.
12. **Change the data model in `schema.clj`, never in `resources/schema.sql`.** Edit the `columns` map, and put extra SQL such as multi-column indexes in `extra-init-sql` in `schema.clj`. Biff regenerates `resources/schema.sql` from `columns`, and sqldef migrates on startup. Run `(refresh)` from `repl.clj` after schema or start-order changes.
13. **Query through biff.graph where the data model is involved.** Biff generates one batch resolver per table from `columns`. Add derived-data and session resolvers under `model/`, and request data with `[:biff.graph.fx/query ...]` as a pipeline's initial effect.
14. **Use `biff.background` for scheduled and queued work.** Declare `:biff.background/tasks` and `:biff.background/queues` in a module and submit with `submit-jobs` or the `:biff.background.fx/submit-jobs` effect. Jobs live in memory only. They are not persisted and are not retried, and a consumer that throws logs the error and drops the job. Queues default to one thread (the schema doc's "10" is stale), and tasks and queues are read only at startup.
15. **Run `clj -M:run code-quality` before finishing.** It updates clj-kondo configs, formats, lints, and tests. The starter CI fails when formatting would change files. While the app runs, agents can evaluate changes, lint, and test with `trench -p $(cat .nrepl-port) -e "((requiring-resolve 'com.biffweb.tasks/agent-refresh))"`.

`/clj-fix` from clojure-skills calls `clj -M:cljfmt`, `clj -X:test`, and `clj -M:dry4clj`, which the starter's `deps.edn` does not define (it has only `:run` and `:prod`). Use `clj -M:run code-quality`, `clj -M:run format`, `clj -M:run lint`, and `clj -M:run test` instead. Biff's `format` task also inserts blank lines between multi-line pairs before running cljfmt, so formatting with plain cljfmt produces different output.

## References

Detailed patterns live in `references/biff-reference.md`. Load it when working on:

- Configuration, secrets, and environment files
- Routes, `defpath`, the default middleware stack, CSRF, and sessions
- Datastar pages, signals, actions, and tab state
- biff.fx machines and pipelines, and testing them
- biff.graph resolvers and queries
- SQLite columns, migrations, queries, authorized writes, transactions, and Litestream
- Background tasks and queues, authentication, the admin dashboard
- Project tasks, the REPL workflow, custom tasks, and the XTDB 2 alternative

## Gotchas

- **Java 21 is the minimum.** Biff 2.0.1 raised it from 17; the starter README still says 17. `biff.xtdb` additionally requires Java earlier than 25 and extra JVM options.
- **`defpipe`, `pipe`, and `:biff.fx/test` do not exist.** The changelog and release notes mention them, but the code provides `pipeline` / `defpipeline`, and tests call a machine or pipeline with no arguments to get its state functions.
- **SQLite column types are `:int :real :text :boolean :inst :uuid :enum :edn :blob`.** The biff.sqlite README example uses `:string`, which fails validation.
- **Destructive migrations are skipped silently.** Removing or retyping a column leaves the old column in place. Apply drops by hand with `target/bin/sqlite3def storage/sqlite/main.db --apply --enable-drop -f resources/schema.sql` after startup regenerates `schema.sql`.
- **`:biff.background/job` is the job key.** The biff.background README destructures `{:biff.background/queues [job]}`, which is wrong; use `{:biff.background/keys [job queue]}`.
- **A missing `COOKIE_SECRET` makes sessions fall back to memory.** Sessions then vanish on every restart; `init` generates the secret into both env files.
- **CSRF protection is header based.** biff.ring rejects state-changing requests that carry neither `Sec-Fetch-Site` nor `Origin` (scripts and `curl` get 403). No token is needed in forms; the starter sets `:biff.auth/skip-csrf-protection true` for the same reason.
- **`init` can replace git history.** Answering yes to its prompt deletes `.git` and creates a fresh repository with one commit.
- **Deploy requires a clean worktree and force-pushes.** `clj -M:run deploy` refuses uncommitted changes or a detached HEAD, and it force-pushes the current branch to the server.
- **The starter `Dockerfile` build stage calls `clj -M:dev uberjar`,** but `deps.edn` has no `:dev` alias. The working command is `clj -M:run uberjar`, which writes `target/jar/app.jar`.
