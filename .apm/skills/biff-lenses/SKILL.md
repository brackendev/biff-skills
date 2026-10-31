---
name: biff-lenses
description: >-
  Translate code-lenses design philosophies (grug, Honest Code, Tidy First,
  Parse Don't Validate, and the opt-in APOSD and Legacy Code lenses) into
  Biff 2-specific patterns. Use when a code-lenses review or refactor runs
  against Biff code: biff.core modules and start-order, biff.fx defpipeline
  and defmachine effects, biff.graph resolvers, biff.sqlite columns and
  authorized-write, biff.ring defpath routes, Datastar pages, actions, and tab
  state, biff.background tasks and queues, and the clojure -M:run task workflow.
  Layers on clojure-lenses from clojure-skills.
user-invocable: false
---

# Code Lenses for Biff

Layered on top of [clojure-lenses](https://github.com/brackendev/clojure-skills) (in `clojure-skills`), which already covers the host-neutral translations: namespaced maps as the default data, Malli closed maps and tagged maps at boundaries, system maps for lifecycle, `ex-info` errors, boring `(is (= (f x) y))` tests, and seams through higher-order functions. This skill records only what Biff 2 adds: the `biff.core` module rules, `biff.fx` effects as data, `biff.graph` resolvers, `biff.sqlite` authorization diffs, `defpath` routing, and the Datastar render loop.

The default code-lenses review set is `grug`, `honest-code`, `tidy-first`, and `parse-dont-validate`. `aposd` and `legacy-code` are opt-in (added with `+aposd` or `+legacy-code`, or invoked directly). The sections below apply when the corresponding lens is active.

When the [code-lenses](https://github.com/brackendev/code-lenses) plugin is active in a Biff project, apply the baseline `clojure-lenses` translations first; reach for this skill for the Biff-specific additions below. The `biff` skill and its reference hold the API details these translations assume.

## Grug Brain

- One page, one render handler. Datastar re-renders the whole page from the database after every write, so there is no client state to synchronize and no per-action partial template to maintain. Adding hand-written partial updates or a client-side state library brings back the complexity this design removes.
- Actions update state and return 204. An action handler that also builds HTML is doing the render handler's job twice.
- A single `:start` state is enough for most POST handlers. Reach for `defmachine` with named states only when the logic branches on an effect's result; otherwise `defpipeline` with one or two functions is the simple form.
- Let biff.sqlite generate resolvers from `columns`: per-table lookups by primary key, lookups by `:unique` columns, and reverse joins for `:ref` columns that set `:backref` or `:backref-many`. Hand-writing any of these duplicates what the module already registers.
- New module folders (`work/`, `api/`, `uicomp/`) are cheap; a `lib/` namespace that grows into a framework is not. Keep `lib/` to plain helper functions.

## A Philosophy of Software Design

- A biff.core module is the deep-module boundary. Its only public surface is the `module` map (`:biff.ring/routes`, `:biff.graph/resolvers`, `:biff.background/queues`, and similar); other modules never require its namespace. Judge depth by how much behavior sits behind those keys.
- biff.graph hides where data comes from. A page asks for `{:session/user [:user/display-name]}` without knowing whether the value is a column, a join, or derived data. Moving logic from helper functions into resolvers deepens the data model; calling `execute` from a page handler for data a resolver could provide makes it shallower.
- `schema.clj` concentrates two decisions: the storage shape (`columns`) and the write policy (`authorize`). Scattering per-handler permission checks spreads that policy across every action.
- Effect handlers are deliberately shallow: one effect, input in, output out. Logic belongs in the pure state functions that call them.
- `routes.clj` holds only paths shared across modules. A path used by one module stays in that module, next to its handler.

## Tidy First

- Order a module namespace as the starter does: requires, `defpath` definitions, a `;;;; Page` section (helpers, then the render pipeline), an `;;;; Actions` section, and `(def module ...)` last.
- Extract a resolver before a render pipeline grows a second query. Two `:biff.graph.fx/query` calls or an `execute` inside a page handler signal data-model logic that belongs under `model/`.
- Replace string-built URLs with `defpath` before changing a route. Literal paths duplicated across handlers drift when the route changes.
- Group routes that share middleware under one `["" {:middleware [...]}]` entry instead of repeating the wrapper per route.
- Move a column into `columns` before writing code that reads it. Coercion applies only to declared columns, so undeclared ones come back as raw SQLite values.
- Run `clojure -M:run format` after tidying. The starter's cljfmt settings align map and `let` columns, and its `format` task inserts a blank line between adjacent pairs when either pair puts its key and value on separate lines, so hand-formatted code produces CI diffs.

## Parse, Don't Validate

- `columns` is the parse layer for stored data. Declared types (`:uuid`, `:inst`, `:enum`, `:edn` with `:extra-schema`) are coerced on write and read, so handlers receive `java.time.Instant`, UUIDs, and keywords, never raw strings or integers.
- `defpath` and `wrap-path-param-uuids` parse route identity: UUID path parameters arrive as `java.util.UUID`. Handlers that call `parse-uuid` on path params duplicate that step.
- Request resolvers are the parse-and-authorize boundary for reads. A resolver that returns `{:request/widget {:widget/id ...}}` only for an authorized user hands the page a parsed, permitted identity; pages do not re-check access, and a page that requires `:request/widget` gets a 400 response when the resolver returns nothing.
- `:biff.datastar/tab-id` is already derived from the user id and the untrusted client tab id. Use it as the tab-state key; never read the raw `client-tab-id` signal.
- Signals are untrusted input. Parse values from `:biff.datastar/signals` (for example `(some-> (:tab/background-color signals) keyword)`) and let `authorized-write` and `:extra-schema` reject what does not fit.
- Configuration is parsed once by `biff.config` into the system map. Code that reads environment variables directly bypasses that parse and the secret wrapping.

## Honest Code

Cite principles by number, as `clojure-lenses` describes. Biff adds these translations:

- **P03 (I/O at the Boundary):** Handlers built with `defpipeline` return effects as data. `{:_ [:biff.sqlite.fx/authorized-write {...}] :status 204}` states exactly what the handler does, and a handler that calls `execute` mid-function hides the write inside logic that could have been pure.
- **P03 for time and randomness:** Upstream's list of calls that reach outside the process includes clock, random, and UUID calls. Pure state functions get these from the context: `:biff.fx/now`, `:biff.fx/random-uuid7-seq`, `:biff.fx/random-uuid4-seq`, and `:biff.fx/seed`. Calling `(java.time.Instant/now)` or `random-uuid` inside a state function makes it impure and its tests nondeterministic.
- **P05 (DOM as State):** Biff 2 renders each page on the server and patches it into the browser with Datastar, which is close to the server-rendered design that P05 describes, although P05 names HTMX. The database is the source for what a page shows, and tab state (a table keyed by `tab-id`) is the source for per-tab UI state. Server-side atoms holding UI state are a second copy that the re-render loop cannot see.
- **P11 (Type Declarations Over Imperative Validation):** `authorized-write` checks the diff it builds against one declared policy, so it is the honest write path for user actions. `execute` for a user-initiated write bypasses that policy.
- **P08 (Typed Exceptions at the Boundary):** A failing effect throws `ex-info` with `:biff.fx/trace`, and biff.ring's internal-error middleware, the boundary, turns it into the `on-error` page. Do not catch and swallow around effects. P08 puts retries in the queue infrastructure, and Biff background jobs are not retried, so work that must eventually succeed needs a persisted record that a scheduled task re-submits, not a catch block.
- **P10 (Pure Function Assertions Over Mocks):** Test state functions directly. Call a pipeline with no arguments to get its step functions, and assert on the returned effect descriptors. No database or mocks are needed.

## Legacy Code

- Seams in Biff 2: the `:biff.fx/handlers` map (substitute effect handlers in a test context), the zero-argument call on machines and pipelines (test logic without effects), `:biff.sqlite/db-path` (point a test system at a temporary file), and `(biff.core/start initial-system #'modules start-order)` (override init values).
- Sprout a new module rather than growing an old one. New behavior goes in a new namespace under `app/` or `model/` registered in `modules.clj`; the legacy module stays untouched.
- Convert one handler at a time from a plain function to `defpipeline`: move each side effect into an effect descriptor and keep the function's inputs and response unchanged. Callers and routes do not change.
- Effect sketching: a change to `columns` touches the generated `schema.sql`, every resolver output that includes the column, every `authorize` rule over its table, and every page that queries it. `graph.mmd`, which the starter's `start` writes from `(biff.graph/uml @system)`, maps the resolver graph for this sketch. Destructive column changes need a manual `sqlite3def --enable-drop` run.
- Characterization tests: start a system with a temporary `:biff.sqlite/db-path`, insert known rows with `execute`, and assert on `biff.graph/query` results before changing a resolver.

## Gotchas

- A "tidy" that has one module namespace require another breaks the module rule and creates load-order coupling. Move the shared code to `lib/` instead.
- Extracting a helper from a page handler must keep the `app-page` call, which returns only the wrapper element on SSE re-renders. A helper that returns a full document breaks the re-render.
- Setting both `:biff.fx/next` and `:biff.fx/return` in one output map is an error; effect order within one map is unspecified, so ordering-sensitive refactors need `:biff.fx/seq`.
- `authorized-write` rejects primary-key changes and `REPLACE INTO`. Rewriting an upsert as `REPLACE` to simplify it fails at runtime; use `:on-conflict` with `:do-update-set` as the starter does.
- Expensive queries in a render handler run on every database write for every open page. Moving such a query into the render path in the name of simplicity degrades every user's page; store its results in tab state from an action instead.
- `(refresh)` from `repl.clj` is needed after changing `columns` or `start-order`; file-save evaluation alone does not re-run module start functions.
