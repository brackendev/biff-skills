---
name: biff-lenses
description: >-
  Translate code-lenses design philosophies into Biff-specific patterns.
  Auto-triggers when working on Biff code (com.biffweb namespace,
  biff/submit-tx, biff/q, biff/lookup, biff/form, doc-schema, :db/doc-type,
  :db.id/* keyword ids, resources/config.edn, dev/repl.clj, dev/tasks.clj,
  src/com/*/app.clj, home.clj, worker.clj, schema.clj, hx-* htmx attributes,
  use-jetty, use-xtdb, use-chime, use-beholder, biff/submit-job,
  biff/authentication-module) alongside the code-lenses plugin. Layers on top of
  clojure-lenses (in clojure-skills); records only the Biff-specific deltas that
  come from XTDB transactions with Malli schema enforcement, the module +
  system-map architecture, htmx response patterns, the authentication module,
  scheduled tasks via use-chime, transaction listeners, queues via
  biff/submit-job, and the REPL workflow. Covers the four default code-lenses
  philosophies (grug, Honest Code, Tidy First, Parse Don't Validate) and the two
  opt-in philosophies (APOSD, Legacy Code) that activate when their lens is
  added with `+aposd` or `+legacy-code` or invoked directly.
user-invocable: false
---

# Code Lenses for Biff

Layered on top of [clojure-lenses](https://github.com/brackendev/clojure-skills) (in `clojure-skills`) for host-neutral translations and the JVM Clojure deltas that the `clojure-jvm` skill covers. This skill records only the Biff-specific additions: XTDB transactions through `biff/submit-tx`, Malli schema enforcement via `:db/doc-type` and `doc-schema`, the module + system-map architecture, htmx response patterns, the authentication module, scheduled tasks via `use-chime`, transaction listeners, queues via `biff/submit-job`, and the live REPL workflow.

The default code-lenses review set is `grug`, `honest-code`, `tidy-first`, and `parse-dont-validate`. `aposd` and `legacy-code` are opt-in (added with `+aposd` or `+legacy-code`, or invoked directly). The sections below apply when the corresponding lens is active; APOSD and Legacy Code translations are present so the lens can use them when explicitly invoked, but they are not part of the default trigger set.

When the [code-lenses](https://github.com/brackendev/code-lenses) plugin is active in a Biff project, apply the baseline `clojure-lenses` translations first; reach for this skill for the Biff-specific additions below.

## Grug Brain

- One module per concern. A Biff module is a map of `{:routes ... :schema ... :tasks ... :on-tx ...}`; resist the urge to bundle authentication, billing, and worker scheduling into a single sprawling module.
- `biff/submit-tx` is the simple thing. Reach for it before reaching for raw `xtdb.api/submit-tx`. The wrapper handles defaults (`:db/op`), schema validation, transaction listeners, and the rest.
- One `:db/doc-type` per logical entity. Mixing two doc-types in one document, or omitting `:db/doc-type` to "save a keyword," is the kind of cleverness that surfaces six months later as a schema violation.
- htmx responses are tiny. A handler returns a hiccup fragment; htmx swaps it into the DOM. Reaching for a full client-side state library because "we might need it later" is a complexity demon's invitation.
- `clj -M:dev dev` reloads on save. Lean on it. The REPL plus file-watch is the simple workflow; spinning up a separate test harness for every change is over-engineering.
- One `use-*` component per system concern. `use-xtdb`, `use-jetty`, `use-chime`, `use-beholder` each compose; resist the urge to write a "biff-everything" component that hides the wiring.
- `biff/q` and `biff/lookup` are the simple queries. Reach for them before raw XTQL or Datalog. The wrappers handle pull projection, schema-aware coercion, and the common ergonomic cases.

## A Philosophy of Software Design

- The Biff system map is a deep module. It hides Jetty configuration, XTDB connection management, scheduled-task wiring, transaction listeners, and the queue worker behind a small surface: `(biff/start-system ...)`. Application code should not reach into the system map's internals.
- A module is a deep module. The module map (`{:routes :schema :tasks :on-tx}`) is the public interface; the namespace's private helpers are not. Other modules see only what the map exposes.
- The doc-type registry is information hiding. `(biff/submit-tx ctx tx)` looks up the doc-type's Malli schema and applies it; callers do not assemble the schema themselves.
- Errors at the right layer. Schema errors belong in the doc-schema definition (Malli rejects malformed input at submit time). Authentication errors belong in `biff/authentication-module`. HTTP-level errors belong in route handlers via Ring response maps. Business-logic errors belong in module functions.
- htmx response handlers are shallow modules: route → hiccup. Resist the urge to interpose a service layer between the handler and the DOM fragment unless three handlers share the same orchestration.
- `use-xtdb-tx-listener` is the right place for derived data. Compute projections, send notifications, or enqueue jobs in response to transactions, rather than scattering those side effects through the handler that submitted the transaction.

## Tidy First

- Extract a helper before a handler grows a third side effect. A route handler that submits a transaction, enqueues a job, and renders a fragment is doing too much; lift the transaction + job pair into a function and let the handler render.
- Extract a Malli schema before its third doc-type variant. If three doc-types share fields, lift the shared shape into a `:map` variable in `schema.clj` and reference it.
- Read order in a module file: namespace, requires, schema, routes, handlers (in route order), on-tx listeners, tasks. Files that interleave handlers and listeners make the dependency graph harder to read.
- Guard clauses in handler bodies: bail early on missing session, missing path params, or unauthorized roles before doing work. `(when-not (some? session) ...)` reads better than a deeply nested `if-let`.
- Threading transactions through `biff/submit-tx`: prefer one `submit-tx` per logical change rather than several small `submit-tx` calls in a handler. XTDB is transactional per submit; multiple submits make the intermediate state observable.
- Tidying a route table: if three handlers share `wrap-csrf`, `wrap-session`, and `wrap-html`, group them under a route prefix and apply the middleware once at the prefix.
- Tidying a worker namespace: when a `:tasks` map grows past five entries, split into worker namespaces by domain (`com.app.worker.email`, `com.app.worker.billing`).

## Parse, Don't Validate

- `doc-schema` is the parse layer for transactions. A `:db/doc-type :user` document arriving at `biff/submit-tx` is parsed against the registered schema before XTDB sees it. Validation does not belong in handler code.
- Malli `:closed true` on doc-schema rejects unexpected keys at the boundary. Use it for every doc-type; the parse is more useful than a permissive map that drifts.
- Form parsing via `biff/form` and a Malli schema turns raw form parameters into a typed map. The handler receives parsed data; it does not coerce strings into integers itself.
- `:db.id/*` keyword ids are parsed identities. The id `:db.id/system` is the parsed form of "the system user"; handlers consume the parsed id, not raw strings.
- Tagged maps for sum types: a job is `{:job/type :send-email :job/args {...}}` or `{:job/type :generate-report :job/args {...}}`. The `:job/type` tag is the discriminator; the worker dispatches on it.
- The session map is parsed at the authentication boundary. The handler receives `{:uid uuid :email "alice@example.com"}` already verified; it does not re-verify the session contents.
- Route data via Reitit is a parse step. Path parameters are coerced to UUIDs, integers, or keywords through the Reitit coercion middleware; handlers receive parsed types.

## Honest Code

- Transactions are data. `[{:db/doc-type :user :user/email "alice@example.com"}]` is a vector of doc maps; `biff/submit-tx` interprets it. The vector is honest; hidden side effects in handler bodies that bypass `submit-tx` are dishonest.
- Handlers return Ring response maps. The response map is the honest output; routes that mutate global state (resetting an atom, calling `set!`) outside the response map are dishonest.
- XTDB is the single source of truth. State that lives outside XTDB (top-level `defonce` atoms, in-memory caches not derived from a transaction listener) is dishonest because the framework cannot reason about it, the deploy story does not survive a restart, and queries cannot find it.
- The system map is honest about dependencies. A module declares its needs (Jetty, XTDB, chime) by composing `use-*` components; nothing happens by hidden side effect at namespace load time.
- `biff/submit-tx` is honest data flow. The call declares "this transaction goes through schema validation, transaction listeners, and the queue"; raw `xtdb.api/submit-tx` calls bypass that flow and are dishonest.
- htmx attributes are honest about intent. `hx-post="/login"` and `hx-target="#card"` declare exactly what the button does and where the response goes; client-side JavaScript handlers that intercept the request and reshape the payload are dishonest.
- Let It Crash, Biff edition: surface errors as Ring `500` responses or push them to the job-failure path. Do not swallow exceptions in `try`/`catch` around `submit-tx`; let the listener or queue handler retry.
- The Biff app state at any moment is the XTDB index plus the system map. Tests that build a known XTDB fixture, submit a transaction, and assert the resulting query result are the honest characterization.

## Legacy Code

- Seams in Biff: the system map (substitute components for tests), `use-xtdb`'s database (point at an in-memory XTDB for tests), the transaction listener registry (replace listeners per test), Reitit's route data (override middleware in test environments), and `biff/submit-job` (substitute a test queue).
- Extract Interface translates to extracting a module. A handler that touches XTDB, the queue, and the authentication module becomes a handler that depends on a `feature.profile` module exposing `(save-profile ctx params)`.
- Parameterize Constructor: pass the system map (`ctx`) to functions that need it. Code that reaches for a top-level `(defonce sys ...)` cannot be tested without that singleton; code that takes `ctx` substitutes a test system.
- Sprout method translates to a new handler or transaction listener. New business logic goes in a new function called from a new route; legacy handlers stay frozen.
- Wrap method translates to wrapping a legacy handler in a higher-order Ring handler. Useful when migrating from an old route shape to a new doc-type one route at a time.
- Scratch refactoring maps to `clj -M:dev dev` plus the REPL. Edit a handler, save, hit the route in the browser, observe. Source-mapped stack traces from JVM Clojure land back in source.
- Effect sketching: a change to a `:db/doc-type` schema ripples through every `biff/submit-tx` site, every `biff/q` that pulls that doc-type, every form that produces it, and every transaction listener that reacts to it. Use a project-wide search before touching a schema.
- Hard dependencies in Biff projects: top-level `defonce` system maps, ad-hoc `xtdb.api/submit-tx` calls that bypass schema, route handlers that reach into the queue worker directly, transaction listeners that hard-code the queue identifier.
- Characterization tests via fixture XTDB: build a known transaction set, submit a new transaction, assert the query result. The "before" and "after" snapshots are the characterization.
- Pre-Biff Clojure web apps (Ring + Compojure + Datomic) look similar but the conventions differ. When migrating, port one module at a time: route table to Reitit route data, Compojure handlers to Biff handlers, Datomic transactions to XTDB through `submit-tx`.

## Gotchas

- A handler that mutates the system atom directly (`(reset! (:biff/system ctx) new-state)`) bypasses the module structure and breaks reload. "Tidy first" should never collapse a module function into a system-atom mutation.
- "Extract helper" inside a route handler must keep the response map together. Pulling the hiccup body into a helper that returns a string and assembling the response map by hand loses the htmx response shape that the rest of the codebase expects.
- `biff/submit-tx` runs synchronously on the calling thread; submitting a 10,000-document transaction inside a handler blocks the response. Reach for `biff/submit-job` and a worker for bulk transactions.
- `:db/op :create` rejects documents whose id already exists; `:db.op/upsert` overwrites. Confusing the two surfaces as silent "the create succeeded but the document looks wrong" bugs.
- Transaction listeners run after the transaction is durable. Side effects in listeners (sending email, enqueuing jobs) cannot be rolled back if the listener throws; idempotent listener bodies are the way.
- `use-beholder` reloads namespaces on file change. Bare `def` forms that compute expensive values at load time will recompute on every save; wrap them in `defonce` or move the computation to a runtime function.
- htmx responses must include the `hx-*` attributes the caller expects. A fragment returned without `hx-swap-oob` or the right `id` will swap silently into the wrong place; check the network tab when the UI does not update.
- `with-redefs` on Biff internals is fragile. The system map captures component references at startup; redefining a component after `(biff/start-system)` does not retroactively change the running system. Pass alternate components into the system map instead.
