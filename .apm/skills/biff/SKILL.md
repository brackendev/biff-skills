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
  config.env, config.prod.env, #biff/env, #biff/secret, repl.clj, the -M:run
  tasks run with clojure or clj (dev, code-quality, format, lint, test,
  deploy, prod-setup), agent-refresh, biff-starter, or any mention of Biff or
  biffweb.
user-invocable: false
---

# Biff

Biff 2 is a Clojure web framework built as a set of independent libraries under `com.biffweb/*`: `biff.core` (module system), `biff.config` (Aero config), `biff.fx` (effects as data), `biff.graph` (resolver graph), `biff.sqlite` (default database), `biff.ring` (Reitit, Jetty, middleware), `biff.datastar` (server-rendered reactive UI), `biff.background` (scheduled tasks and in-memory queues), `biff.authenticate` (email sign-in codes), `biff.admin` (dashboard), and the dev-only `biff.run` / `biff.tasks` task runner. `com.biffweb/defaults` bundles everything except `run` and `tasks`; `biff.xtdb` (XTDB 2) is an optional alternative to SQLite. Projects start from the `jacobobryant/biff-starter` repository. Java 21 or later is required.

This skill covers only what Biff changes. Follow the `clojure` skill from [clojure-skills](https://github.com/brackendev/clojure-skills) for general style (naming, namespaces, `defn-`, threading, Malli at boundaries, testing) and the `clojure-jvm` skill from [clojure-jvm-skills](https://github.com/brackendev/clojure-jvm-skills) for Java interop, JVM exceptions, nREPL, clj-kondo, cljfmt, and tools.build.

## Iron Laws

These rules prevent security holes and lost changes in Biff 2 applications. They follow the upstream review rules in `.agents/skills/biff-code-review/rules.md`, which `clojure -M:run init` writes into every Biff 2 project, and the Biff library READMEs. The `clojure` skill's Iron Laws (CLJ-1 through CLJ-12) and the `clojure-jvm` skill's (JVM-1 through JVM-6) also apply. Never write code that breaks one. When existing code breaks a law, stop and report the law ID, the location, the offending code, and the correct pattern, then ask before changing code outside the current task. A law overrides any Key Rule or reference that seems to conflict with it.

1. **BIFF-1: ROUTE USER WRITES THROUGH `authorized-write`.** Writes made for the current user go through `:biff.sqlite.fx/authorized-write` (or `-tx`) with HoneySQL maps, so the `authorize` function in `schema.clj` checks the diff. Add an `authorize` case for every new table, because the starter's `authorize-entry` returns `false` for any table it does not name. Use plain `execute` only for system writes such as background jobs, or for a user write the rules cannot validate, and then check authorization in the handler. Authorization rules add to authentication middleware and do not replace it (`references/biff-reference.md`).
2. **BIFF-2: AUTHORIZE FROM THE DIFF AND THE SESSION USER ONLY.** An `authorize` rule must not use information that a call site passes in. Upstream frames the question as "if the user could submit arbitrary transactions from the frontend, what would we allow them to do?" When a record does not show its owner, query `:biff.sqlite/before-conn` or `:biff.sqlite/after-conn` (`references/biff-reference.md`).
3. **BIFF-3: AUTHORIZE EVERY REQUEST-DERIVED READ IN A RESOLVER.** biff.sqlite has no read authorization. Extract ids from path or query parameters in a biff.graph resolver whose output is a `:request/*` join, such as `{:request/widget [:widget/id]}`, and return it only when the session user may read the entity (`references/biff-reference.md`).
4. **BIFF-4: READ CONFIGURATION ONLY FROM THE SYSTEM MAP.** Never read environment variables or the global `system` atom from application code. The atom is for the REPL only. Declare values in `resources/config.edn` with `#biff/env` and wrap secrets with `#biff/secret`, then unwrap them with `force` (nil-safe), not `@`.
5. **BIFF-5: EDIT `schema.clj`, NEVER THE GENERATED `schema.sql`.** On every start Biff rewrites `resources/schema.sql` from the `columns` map, marked "Auto-generated; do not edit," so an edit there is lost. Edit `columns`, put extra SQL such as multi-column indexes in `extra-init-sql`, and run `(refresh)` from `repl.clj` after schema changes. sqldef applies only non-destructive migrations, so drops need the manual step in Gotchas.
6. **BIFF-6: SERVE BROWSER ENDPOINTS FROM `:biff.ring/routes`.** Site routes get sessions and CSRF protection. `:biff.ring/api-routes` have neither and are for external APIs only, so a browser form or Datastar action served from them has no CSRF protection (`references/biff-reference.md`).

## Project Layout

The starter project (main namespace `com.example`, renamed by `clojure -M:run init`) looks like this:

```
├── deps.edn                     :run alias (tasks) and :prod alias (server)
├── repl.clj                     REPL scratch space: get-ctx, refresh
├── tests.edn                    Kaocha configuration
├── graph.mmd                    resolver graph (Mermaid), written by start, not gitignored
├── .agents/skills/              Biff's own agent skills, rewritten by init
├── .claude/skills               symlink to ../.agents/skills
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
│       ├── schema.clj           SQLite columns map (com.biffweb.sqlite.schema helpers) and authorization rules
│       ├── app/                 user-facing modules (routes and handlers)
│       ├── model/               data-model modules (biff.graph resolvers, sqlite module)
│       └── lib/                 shared functions (ui, middleware, email)
└── test/com/example_test.clj
```

`config.env` and `config.prod.env` hold secrets and are gitignored; `clojure -M:run init` (run automatically by `dev`) generates them from the templates.

`init` also writes Biff's own agent skills into `.agents/skills/`: `biff-dev` (the `agent-refresh` loop), `biff-code-quality`, `biff-code-review` (with `rules.md` and `checklist.txt`), `biff-spec-review`, and `biff-spec-dev`. Each is marked autogenerated and is overwritten whenever it differs from the copy in the biff.tasks jar, so never edit them. `:biff.tasks/skip-project-files true` stops the copying. Run `biff-code-review` for a full convention review against `rules.md`. Its pointer to `.biff/docs/` leads nowhere: the biff.tasks 2.0.3 jar bundles no docs, so `init` never creates that directory. Read the library READMEs in `jacobobryant/biff` instead.

The starter README places files that define biff.core modules only in module folders (`app/`, `model/`, and any you add, such as `work/` for background jobs, `api/` for external API routes, or `uicomp/` for resolvers that return hiccup). The review rules name that folder `uicomp`, and the README calls it `ui_components`. Key Rule 1 covers how module namespaces depend on each other.

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
3. **Build every URL with `defpath`.** Call the path function with no arguments in the Reitit route vector and with arguments for `:href` and `:action`. UUID path parameters are encoded and decoded automatically. Name a path parameter after its entity (`:post-id`, never `:id`), and add a second parameter only when it cannot be inferred from the first.
4. **Write handlers as `defpipeline` or `defmachine`.** State functions stay pure and return effect descriptors such as `[:biff.sqlite.fx/execute {...}]`. Put effects whose result is not needed under an underscore-prefixed key (`:_`), and use `:biff.fx/seq` when effect order matters.
5. **Render each page from one handler.** Wrap content in `ui/app-page`, which returns only the `#biff-datastar-content` element when `:biff.datastar/sse-request` is true. Returning a full document on those requests nests `<html>` inside the page. Datastar re-renders the page over SSE after every database write.
6. **Return 204 from Datastar actions.** Action handlers update state and return `{:status 204}` (or `patch-signals` to change client signals). They do not render HTML.
7. **Bind form inputs to namespaced signals.** Initialize signals with `:data-signals__ifmissing` and `signals-json`, bind with `signal-name`, and read them from `:biff.datastar/signals`. Without `__ifmissing`, every server push overwrites the signals and erases what the user is typing. Avoid signals for anything other than form inputs; store UI state the render needs in tab state keyed by `:biff.datastar/tab-id`.
8. **Keep page rendering fast.** Every write re-renders open pages. Move expensive queries into an action that stores results in tab state.
9. **Query through biff.graph where the data model is involved.** biff.sqlite generates batch resolvers from `columns`: one per table that takes the primary key, one per `:unique` or `:unique-with` column that returns the primary key, and a reverse join for each `:ref` column that sets `:backref` or `:backref-many`. Declare backrefs instead of writing join resolvers. Add derived-data and session resolvers under `model/`, and request data with `[:biff.graph.fx/query ...]` as a pipeline's initial effect.
10. **Use `biff.background` for scheduled and queued work.** Declare `:biff.background/tasks` and `:biff.background/queues` in a module and submit with `submit-jobs` or the `:biff.background.fx/submit-jobs` effect. Jobs live in memory only. They are not persisted and are not retried, and a consumer that throws logs the error and drops the job. Queues default to one thread (the code uses 1, although `docs/schema.md` says 10), and tasks and queues are read only at startup.
11. **Run `clojure -M:run code-quality` before finishing.** It updates clj-kondo configs, formats, lints, and tests. The starter CI fails when formatting would change files. While the app runs, agents can evaluate changes, lint, and test with `trench -p $(cat .nrepl-port) -e "((requiring-resolve 'com.biffweb.tasks/agent-refresh))"`.

In a Biff project, format, lint, and test with Biff's own tasks: `clojure -M:run code-quality` for all three, or `clojure -M:run format`, `clojure -M:run lint`, or `clojure -M:run test` for one. The starter's CI workflow runs `clojure -M:run code-quality` and then `git diff --quiet --exit-code`, so it passes only on files that Biff's tasks formatted. `/clj-fix` from clojure-skills gives different results in a Biff project, because the starter's `deps.edn` defines only the `:run` and `:prod` aliases:

- **Format.** With no `:cljfmt` alias and no `project.clj`, `/clj-fix` runs a `cljfmt` binary from `PATH` or a `cljfmt` tool installed with `clojure -Ttools`, at whatever version the user installed, and reports `SKIPPED` when neither exists. Biff's own cljfmt lives in `target/bin/`, which is not on `PATH`, so `/clj-fix` never finds it. `/clj-fix` formats only the source and test directories. Biff's `format` task runs cljfmt 0.16.4 (`:biff.tasks/cljfmt-version`, downloaded into `target/bin/` when no cljfmt of that version is installed) over every Git-tracked `.clj`, `.cljc`, `.cljs`, and `.edn` file outside hidden directories, which includes `repl.clj`, `deps.edn`, and `resources/config.edn`. Before it runs cljfmt, it inserts a blank line between two adjacent pairs in a map, a binding vector, or a `cond` when either pair puts its key and value on separate lines.
- **Lint.** `/clj-fix` adds the clj-smells-fix overlay, which keeps linters such as `:unused-binding` and `:refer-all` at warning level even where the project configuration turns them off, adds checks for internal `clojure.lang.RT` calls, and applies its safe rewrites. Biff's `lint` task runs its pinned clj-kondo (2026.05.25) with only the project configuration over every tracked Clojure and EDN file, and `code-quality` first refreshes the clj-kondo dependency configurations.
- **Test and dry.** The starter has no `:test` alias, no `bin/kaocha`, and no `:dry4clj` alias, so `/clj-fix` reports both steps `SKIPPED`. `clojure -M:run test` runs Kaocha with `tests.edn`.

When the user runs `/clj-fix` in a Biff project anyway, run `clojure -M:run code-quality` afterward so the files match what CI checks.

## References

Detailed patterns live in `references/biff-reference.md`. Load it when working on:

- Configuration, secrets, and environment files
- Routes, `defpath`, the default middleware stack, CSRF, and sessions
- Datastar pages, signals, actions, and tab state
- biff.fx machines and pipelines, and testing them
- biff.graph resolvers and queries
- SQLite columns and column helpers, generated resolvers, migrations, queries, authorized writes, transactions, and Litestream
- Background tasks and queues, authentication, the admin dashboard
- Project tasks, the REPL workflow, custom tasks, and the XTDB 2 alternative

## Gotchas

- **Java 21 or later is required.** The Biff README requires it, and biff.ring runs Jetty on virtual threads. The starter README says Java 17, which is wrong. `biff.xtdb` needs Java 21 or later but earlier than 25, plus extra JVM options.
- **`defpipe`, `pipe`, and `:biff.fx/test` do not exist.** Upstream `CHANGELOG.md` names `pipe` and `defpipe`, and `RELEASES/2.0.0-rc23.md` describes `:biff.fx/test`, but the code provides `pipeline` / `defpipeline`, and tests call a machine or pipeline with no arguments to get its state functions.
- **An unregistered effect key fails silently.** biff.fx runs a descriptor only when its keyword is a registered handler. A misspelled key such as `[:biff.sqlite.fx/authorized-writ ...]` runs nothing, raises no error, and stays in the returned map as plain data, so a handler can return 204 without writing. A leftover `:_` key in a handler's result is the sign.
- **Destructive migrations are skipped.** Removing or retyping a column leaves the old column in place, and the startup log shows only a `-- Skipped: ALTER TABLE ... DROP COLUMN ...` line. Apply drops by hand with `target/bin/sqlite3def storage/sqlite/main.db --apply --enable-drop -f resources/schema.sql` after startup regenerates `schema.sql`.
- **`:biff.background/job` is the job key.** The biff.background README destructures `{:biff.background/queues [job]}`, which is wrong; use `{:biff.background/keys [job queue]}`.
- **A missing `COOKIE_SECRET` makes sessions fall back to memory.** Sessions then vanish on every restart; `init` generates the secret into both env files.
- **CSRF protection is header based.** biff.ring rejects state-changing requests that carry neither `Sec-Fetch-Site` nor `Origin` (scripts and `curl` get 403), and requests whose `Sec-Fetch-Site` is `same-site` or `cross-site`, so a subdomain cannot post to the app. No token is needed in forms; the starter sets `:biff.auth/skip-csrf-protection true` for the same reason.
- **`init` prompts on a project still named `com.example`.** It asks for the main namespace and repeats that prompt forever when stdin is closed, which hangs an agent that runs `dev` or `init` in a fresh clone. Pipe the answers instead: `printf 'com.myapp\nn\n' | clojure -M:run init`.
- **`init` can replace git history.** Its second prompt defaults to yes: pressing Enter or answering yes deletes `.git` and creates a fresh repository with one commit.
- **Deploy requires a clean worktree and force-pushes.** `clojure -M:run deploy` refuses uncommitted changes, untracked files, or a detached HEAD, and it force-pushes the current branch to the server. The starter's `start` writes `graph.mmd` to the project root and `.gitignore` does not list it, so after the first local run the file blocks every deploy until it is committed or ignored.
- **The starter's `-main` ignores `:biff.tasks/nrepl-port`.** Its `start` returns the result of `spit "graph.mmd"` (nil), and `-main` passes `(:biff.tasks/nrepl-port (start))` to nREPL, which then listens on a random port. Locally `.nrepl-port` holds the real port, so `trench -p $(cat .nrepl-port)` still works. In production, `prod-nrepl` and `deploy --soft` connect to the configured port and fail. Make `start` return `@system` after the `spit` to restore the configured port.
- **The starter `Dockerfile` build stage runs `RUN clj -M:dev uberjar`** (quoted from the starter `Dockerfile`), but `deps.edn` has no `:dev` alias. The working command is `clojure -M:run uberjar`, which writes `target/jar/app.jar`.
