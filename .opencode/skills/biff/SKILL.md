---
name: biff
description: >-
  Use when writing, editing, reviewing, or discussing Biff web applications.
  Triggers: com.biffweb namespace, biff/submit-tx, biff/q, biff/lookup,
  biff/lookup-id, biff/form, doc-schema, :db/doc-type, :db/op, :db.id/* keyword
  ids, :db/now, resources/config.edn, config.env, dev/repl.clj, dev/tasks.clj,
  src/com/*/app.clj, home.clj, worker.clj, schema.clj, hx-* htmx attributes in
  Clojure code, use-jetty, use-xtdb, use-chime, use-beholder, use-xtdb-tx-listener,
  biff/submit-job, biff/submit-job-for-result, biff/authentication-module,
  biff/wrap-render-rum, biff/add-seconds, clj -M:dev dev, clj -M:dev deploy,
  clj -M:dev logs, clj -M:dev generate-config, server-setup.sh, fixtures.edn,
  or any mention of Biff or biffweb. Covers the module + system-map architecture,
  project layout, routes, XTDB transactions with Malli schema, queries, htmx
  patterns, authentication, scheduled tasks, transaction listeners, queues, and
  REPL workflow.
user-invocable: false
---

# Biff

Biff is a Clojure web framework that bundles XTDB (with Malli schema enforcement), htmx with hyperscript, passwordless email authentication, a module-plus-system-map architecture, live REPL with file-watch, and Ubuntu VPS / Docker / uberjar deployment. It targets solo developers and small teams.

This skill covers Biff-specific conventions. For general Clojure style, follow the `clojure` skill from the companion [clojure-skills](https://github.com/brackendev/clojure-skills) package. Run the shared quality pipeline via `/clj-fix` from that package; pass `dev` as a lint path so the Biff `dev/repl.clj` and `dev/tasks.clj` helpers are covered. There is no Biff-specific check command -- the underlying Clojure tooling is identical.

## Project Layout

Biff projects use a `com.<organization>.<app>` namespace under `src/com/`. A new project with `com.example` as the main namespace looks like:

```
├── Dockerfile
├── README.md
├── cljfmt-indents.edn
├── deps.edn
├── dev
│   ├── repl.clj                  REPL helpers and notes
│   └── tasks.clj                 clj -M:dev <task> commands
├── resources
│   ├── config.edn                public configuration (committed)
│   ├── config.template.env       generator for config.env
│   ├── fixtures.edn              dev seed data
│   ├── public/                   static assets served as-is
│   ├── tailwind.config.js
│   └── tailwind.css
├── server-setup.sh               Ubuntu VPS provisioning script
├── src
│   └── com
│       ├── example
│       │   ├── app.clj           routes and handlers
│       │   ├── email.clj
│       │   ├── home.clj
│       │   ├── middleware.clj
│       │   ├── schema.clj        Malli schema
│       │   ├── settings.clj
│       │   ├── ui.clj
│       │   └── worker.clj
│       └── example.clj           entrypoint
└── test
    └── com
        └── example_test.clj
```

`resources/config.edn` holds public configuration and is committed. `config.env` holds secrets and is gitignored; generate it with `clj -M:dev generate-config` (or by running `clj -M:dev dev` once, which generates it on demand).

## Modules and the system map

A module is a plain map. Any of these keys are recognized:

```clojure
(def module
  {:static     {...}     ; static HTML routes (path -> Rum data)
   :routes     [...]     ; CSRF-protected routes
   :api-routes [...]     ; CSRF-disabled routes (public APIs)
   :schema     {...}     ; Malli schema fragments
   :tasks      [...]     ; chime scheduled tasks
   :on-tx      (fn ...)  ; XTDB transaction listener
   :queues     [...]})   ; in-memory concurrent queues
```

Modules are bundled in the system map:

```clojure
(def modules
  [app/module
   (biff/authentication-module {})
   home/module
   schema/module
   worker/module])

(def initial-system {:biff/modules #'modules ...})
```

The system map flows through a sequence of components. Each component is a function `(system) -> system'`:

```clojure
(def components
  [biff/use-config
   biff/use-secrets
   biff/use-xtdb
   biff/use-queues
   biff/use-xtdb-tx-listener
   biff/use-jetty
   biff/use-chime
   biff/use-beholder])
```

After all components run, the system is live. Handlers receive the system map merged with the Ring request as a context map (`ctx`), so resources like `:biff.xtdb/node` and `:biff/db` are available:

```clojure
(defn hello [{:keys [biff/db] :as ctx}]
  (let [n-users (ffirst (xt/q db '{:find [(count user)]
                                   :where [[user :user/email]]}))]
    [:html [:body [:p "There are " n-users " users."]]]))

(def module {:routes [["/hello" {:get hello}]]})
```

Prefer `ctx` over `req` in handler signatures; the context map carries more than just the Ring request.

## Key Rules

1. **Prefer Biff helpers over the underlying library.** Use `biff/submit-tx` over `xt/submit-tx`, `biff/q` over `xt/q`, `biff/lookup` / `lookup-id` over hand-rolled `xt/pull` lookups, and `biff/form` over hand-rolled CSRF token markup. The helpers enforce schema, await transactions, throw on misuse, and accept `ctx` directly.
2. **Tag every Biff transaction document with `:db/doc-type`.** Schema enforcement runs only when this key is present. A transaction that bypasses it can corrupt invariants silently.
3. **Choose `:db/op` deliberately.** The default is `:xtdb.api/put`, which replaces the whole document. Use `:db/op :merge` to create-or-merge and `:db/op :update` when the document must already exist. Use `:db/op :delete` to delete.
4. **Use `:db/now` and `:db.id/<name>` sentinels in transactions.** `:db/now` becomes `(java.util.Date.)`. Any `:db.id/foo` keyword becomes a fresh random UUID; reuse the same `:db.id/foo` within a transaction to link new documents together.
5. **Pass a function to `biff/submit-tx` when contention is possible.** Biff retries on contention up to three times. With a function (rather than a vector), the function is re-invoked between retries with a fresh `:biff/db`. This is required for any read-then-write pattern.
6. **Use `:api-routes` for public JSON APIs.** Only `:api-routes` disables CSRF. Anything under `:routes` requires the anti-forgery token and is intended for browser sessions.
7. **Attach middleware via the Reitit `{:middleware [...]}` route data.** Per-route middleware composes correctly with Biff's late binding. Wrapping the top-level handler defeats live-evaluation of route-local middleware.
8. **Return Rum vectors directly from handlers.** Biff's `wrap-render-rum` middleware renders `[:html ...]` and other Rum vectors to HTML. Do not call `rum/render-static-markup` manually inside a handler.
9. **Use `biff/form` for every POST form.** It injects the CSRF token (`__anti-forgery-token`) and a couple of other conveniences. Manual hidden inputs are allowed but easier to forget.
10. **For htmx responses, return a partial vector, not a full HTML document.** htmx swaps the response into the page via `hx-swap` attributes; returning a full `[:html ...]` document inside a swap target produces nested `<html>` tags.
11. **Use schema aliases for foreign keys.** Define `:user/id :uuid` and reference `:user/id` from `:msg/user`. Biff does not enforce referential integrity, but the alias makes intent obvious to readers and to other tooling.
12. **Use late binding (`#'var`) in modules and component lists.** This is what lets `clj -M:dev dev` pick up file-save evaluations without restarting the system.
13. **Reserve `(refresh)` for component or system-map changes.** Most edits do not need it. Calling `refresh` stops every cleanup function in `:biff/stop`, then runs `clojure.tools.namespace.repl/refresh :after start`.
14. **Push long-running work behind `:queues` or `:tasks`, not into a request handler.** Queues bound concurrency for fan-out; chime tasks run on a schedule. Queues are in-memory and not persisted, and there is no retry for jobs that throw -- jobs that must survive a restart need a different mechanism.
15. **For real-time fan-out across users, broadcast from an `:on-tx` listener.** Calling `jetty/send!` directly from a request handler only reaches clients connected to the same JVM. Routing the broadcast through a transaction listener fans out reliably.

## References

Detailed patterns live in `references/biff-reference.md`. Load it when working on:

- Routes, middleware, htmx responses, out-of-band swaps
- Schema design, `biff/submit-tx`, transaction functions, `biff/q`, `lookup` helpers
- The authentication module, sessions, role-based authorization, CSRF, secret rotation
- Modules, components, the system map, scheduled tasks, transaction listeners, queues, REPL workflow

## Gotchas

- **`config.env` is not generated until the first `clj -M:dev dev` run.** Fresh checkouts of a Biff project will fail config loading until `clj -M:dev generate-config` (or `dev`, which auto-generates) runs once.
- **Tests run automatically on file save during `clj -M:dev dev`.** A failing test you haven't written yet will spam the dev log; triage at the source rather than disabling.
- **htmx pages need the htmx `<script>` in `<head>` once.** Forgetting it makes `hx-*` attributes inert with no error.
- **WebSocket fan-out from a request handler only reaches clients on the same JVM.** Push fan-out into an `:on-tx` listener.
- **`server-setup.sh` is project-local.** Edits to install additional system packages belong in this script so the server stays reprovisionable from scratch.
- **`clj -M:dev deploy` only uploads files tracked by `git ls-files`** (plus `config.env` and `target/resources/public/css/main.css`). Add overrides under `:biff.tasks/deploy-untracked-files` in `resources/config.edn`.
- **`clj -M:dev prod-dev` evaluates local changes against the running production system.** Use it deliberately; it is not a sandbox.
- **Standalone XTDB topology has no built-in backups and cannot run on more than one server.** For production data, point XTDB at a managed Postgres via `config.env`.
