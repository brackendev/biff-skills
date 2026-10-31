# Biff Reference

Deeper detail for the `biff` skill, written against Biff 2.0.3 and the `biff-starter` project. Sections are grouped by library; load whichever applies. Examples come from the starter and the library READMEs, corrected where the README disagrees with the code.

## Configuration (biff.config)

`(biff.config/module)` (id `:biff.config/module`, first in `start-order`) reads `resources/config.edn` with Aero and merges it into the system map.

```clojure
{:biff.ring/cookie-secret #biff/secret COOKIE_SECRET
 :biff.ring/port          #long #or [#biff/env PORT "8080"]
 :biff.ring/secure        #boolean #or [#biff/env SECURE "true"]
 :biff.sqlite/db-path     #or [#biff/env SQLITE_DB_PATH "storage/sqlite/main.db"]
 :biff.tasks/main-ns      com.example
 :biff.config/system-properties {"clojure.main.report" "stderr"}}
```

- `#biff/env NAME` reads a value; keys whose value is nil are dropped. An empty value counts as unset, so a blank `LITESTREAM_ENDPOINT=` line in an env file leaves the key out. `#biff/secret NAME` wraps the value in a secret delay that prints as `#<SecretDelay: redacted>`.
- Values are looked up in this order: JVM system properties prefixed `biff.env.`, real environment variables, then a `config.env` dotenv file in the working directory.
- Unwrap secrets with `force`: `(force (:mailersend/api-key ctx))`.
- Prefer flat, namespaced keys. Register application config schema with `(biff.core/register {:com.example/api-key :biff.core/secret})`.
- The Aero profile comes from `BIFF_PROFILE`. The author recommends keeping the same variable names in every environment and changing only the values instead of using `#profile`.
- `config.env` holds development values. `config.prod.env` is a local file that `clojure -M:run deploy` uploads as the server's `config.env`. `clojure -M:run init` creates both from `resources/TEMPLATE.config.env` and `resources/TEMPLATE.config.prod.env`, replacing each `{{ new-secret N }}` with N random bytes, base64 encoded.

## Modules (biff.core)

| Key | Meaning |
|-----|---------|
| `:biff.core/id` | Qualified keyword; required when the module has `:biff.core/start` |
| `:biff.core/init` | Map, or `(fn [modules-var]) => map`, merged into the system map before start |
| `:biff.core/start` | `(fn [ctx]) => ctx` |
| `:biff.core/stop` | `(fn [ctx])`, receives what its own start returned |
| `:biff.core/on-tx` | `(fn [ctx])`, called after each database transaction; biff.core combines every module's value |
| `:biff.core/kv-get`, `kv-set`, `kv-list`, `wrap-db-snapshot` | Database adapter functions (provided by biff.sqlite or biff.xtdb) used by other Biff libraries |

`biff.core/start` validates modules against registered schema, fails on duplicate ids, on ids missing from `start-order`, and on two init functions returning the same key. `(biff.core/start initial-system #'modules start-order)` overrides init values. `biff.core/stop` runs stop functions in reverse order.

`biff.core/validate` is a macro that compiles to nothing when `*assert*` is false; use `biff.core/validate-with-ex` where validation must run in production.

## Routes (biff.ring)

`(biff.ring/module)` (id `:biff.ring/module`, last in `start-order`) builds `:biff.ring/handler` from every module's routes and starts Jetty on `:biff.ring/host` / `:biff.ring/port` using virtual threads.

| Module key | Purpose |
|------------|---------|
| `:biff.ring/routes` | Site routes: sessions and CSRF protection |
| `:biff.ring/api-routes` | API routes: no session, no CSRF |
| `:biff.ring/site-middleware`, `:biff.ring/api-middleware`, `:biff.ring/base-middleware` | Extra middleware inserted into the matching default stack |
| `:biff.ring/on-error` | `(fn [{:keys [status] :as ctx}])` rendering 400, 403, 404, 405, 406, and 500 responses. An exception whose `ex-data` (or a cause's) holds `:biff.ring/status` produces that status |

```clojure
(ns com.example.routes
  (:require [com.biffweb.ring :refer [defpath]]))

(defpath home "/")
(defpath app "/app")
(defpath post-page "/posts/:post-id")

(routes/post-page 123) ;=> "/posts/123"
(routes/post-page)     ;=> "/posts/:post-id", for the Reitit route vector
```

Define paths used by one module in that module, and shared paths in `routes.clj`. UUID path parameters are encoded as 22-character base64url strings and decoded by `wrap-path-param-uuids`. A trailing map argument becomes a single `npy` query parameter (nippy-encoded) that `wrap-nippy-params` merges into `:params`. Calling a path function with the wrong number of arguments throws.

Middleware attaches through Reitit route data:

```clojure
(def module
  {:biff.ring/routes
   ["" {:middleware [mid/wrap-signed-in]}
    [(routes/app)                {:get demo-page}]
    [(set-display-name-path)     {:post set-display-name}]
    [(set-background-color-path) {:post set-background-color}]]})
```

Default stacks, outermost first:

- Base: request logging, `wrap-ssl` (HSTS on by default; `:biff.ring/ssl-redirect` off by default), internal-error handling, static resources from `public/` on the classpath.
- Site: ring-defaults site settings (its anti-forgery, session, and static handling disabled), Muuntaja format negotiation, params, nippy params, UUID path params, `wrap-session`, and `wrap-csrf-protection`.
- API: ring-defaults API settings, Muuntaja, params.

`wrap-csrf-protection` lets a state-changing request through when `Sec-Fetch-Site` is `same-origin` or `none`, or, when that header is absent, when `Origin` names the same host and port as `Host`. Everything else gets a 403, including requests with neither header. Browsers send these headers, so forms need no token.

Sessions are encrypted cookies keyed by `:biff.ring/cookie-secret` (base64, 16 bytes), lasting 60 days with SameSite Lax and HttpOnly. `:biff.ring/secure` (true by default; the dev `config.env` sets `SECURE=false`) controls the Secure flag. Without a cookie secret, sessions fall back to the in-memory store that `biff.ring/module` creates, and no warning is logged (the warning in `wrap-session` can never fire).

The signed-in user id is `(:uid session)`. The starter's `lib/middleware.clj` provides `wrap-signed-in` (redirects to `(routes/signin)` with a 303) and `wrap-redirect-signed-in` (redirects signed-in users to `(routes/app)`).

## Pages and Actions (biff.datastar)

`(biff.datastar/module)` (init only, not in `start-order`) adds `wrap-sse-render` to the site middleware and calls `refresh` from `:biff.core/on-tx`. Each page opens a long-lived SSE connection; after every database write the page handler runs again and changed HTML is pushed to the client.

The starter's `lib/ui.clj` loads Datastar in `<head>` and puts `(biff.datastar/init-opts)` on `<body>`. `app-page` returns only the wrapper element on SSE re-render requests:

```clojure
(defn app-page [{:keys [biff.datastar/sse-request] :as ctx} & content]
  (let [content* [:div#biff-datastar-content
                  {:class ["flex min-h-full flex-1 flex-col"]}
                  content]]
    (if sse-request
      (html-response content*)
      (page (assoc ctx :ui/init-datastar true) content*))))
```

A page and its actions, adapted from the starter's `app/demo.clj` (which writes `[:? :user/display-name]`):

```clojure
(defpath set-display-name-path "/app/display-name")

(defpipeline demo-page
  [:biff.graph.fx/query
   [{:session/user [:?user/display-name]}]]

  (fn [request {:keys [session/user]}]
    (let [{:user/keys [display-name]} user]
      (ui/app-page
       request
       [:main {:id "demo-page"
               :data-signals__ifmissing
               (biff.datastar/signals-json
                {:user/display-name (or display-name "")})}
        [:form {:data-action    (set-display-name-path)
                :data-on:submit "@post(el.dataset.action)"}
         [:input {:type      "text"
                  :data-bind (signal-name :user/display-name)}]
         [:button {:type "submit"} "Save"]]]))))

(defpipeline set-display-name
  (fn [{:keys [session] :biff.datastar/keys [signals]}]
    {:_      [:biff.sqlite.fx/authorized-write
              {:update :user
               :set    {:user/display-name (:user/display-name signals)}
               :where  [:= :user/id (:uid session)]}]
     :status 204}))
```

- `signals-json` and `signal-name` convert keywords to Datastar names (periods and slashes become underscores); `(signal-name [:my.form/pets 0 :pet/name])` addresses a nested signal. Signals arrive parsed in `:biff.datastar/signals` on every action request, for the whole page. The client controls every signal value, so treat signals as untrusted input: parse each one before use, as with form input, and send writes through `authorized-write`. Key tab state by the trusted `:biff.datastar/tab-id`, never by an id taken from a signal.
- `(biff.datastar/patch-signals {...})` returns a 200 response that updates client signals, for example to clear an input after submit.
- `:biff.datastar/tab-id` is a trusted UUID derived from the client tab id and the user id (for a visitor who is not signed in, an id that biff.datastar stores in the session). biff.datastar stores no tab state itself; the starter keeps it in a `tab-state` table (`:tab-state/id` uuid, `:tab-state/data` edn) written with `authorized-write` and read by a resolver.
- `(biff.datastar/disconnect @system)` closes every SSE connection so clients reconnect; the starter's entry namespace calls it on load so pages refresh after file evaluation.
- `:biff.datastar/heartbeat-ms` (default 15000) detects dead clients; `:biff.datastar/rate-limit-ms` (default 20) limits re-render frequency per client.
- The library README calls this architecture less well-trodden; measure before relying on it under heavy load.

## Effects (biff.fx)

`(biff.fx/module)` (init only) collects `:biff.fx/handlers` from all modules. biff.sqlite, biff.graph, and biff.background contribute handlers automatically, and biff.fx itself provides two:

- `[:biff.fx/http request]` runs a hato request map, or a vector of them (the starter includes hato).
- `[:biff.fx/swap! get-atom f & args]` calls `swap!` on `(get-atom ctx)`. Put the atom in the system map with a module such as `{:biff.core/init {:com.example/counter (atom 0)}}` and pass the key: `[:biff.fx/swap! :com.example/counter inc]`.

Use those before writing a handler. The upstream review rules put any application handlers in a single `fx.clj` namespace that exposes a module, keep each handler to the IO alone, and forbid handlers in `app`, `model`, `work`, `uicomp`, and `api` namespaces.

```clojure
(defpipeline get-user-id
  (fn [_ctx email]
    [:biff.sqlite.fx/execute
     {:select [:user/id] :from :user :where [:= :user/email email]}])

  (fn [_ctx result]
    (-> result first :user/id)))

(defmachine increment-file
  :start
  (fn [{:keys [path]}]
    {:content      [:example.fx/slurp path]
     :biff.fx/next :increment})

  :increment
  (fn [{:keys [path]} {:keys [content]}]
    (let [new-n (inc (or (some-> content parse-long) 0))]
      {:_              [:example.fx/spit path (str new-n)]
       :biff.fx/return new-n})))
```

- An effect descriptor is `[:effect-key & args]`; its handler is `(fn [ctx & args])`. Descriptors are recognized as top-level values of an output map, inside `:biff.fx/seq`, or as a state function's entire return value. A vector counts as a descriptor only when its first element is a registered handler key. Any other vector, including one with a misspelled key, passes through as plain data with no error and no effect.
- A pipeline step receives `ctx` and the previous result; the first step receives the caller's arguments. A machine's `:start` receives `ctx` and the caller's arguments; later states receive `ctx` and the previous output map.
- `:biff.fx/next` selects the next state; `:biff.fx/return` exits with a value. Setting both is an error. Without either, the output map (with effect results substituted) is returned. Use `:biff.fx/return` only to leave a pipeline early or to return a non-map value.
- Keys starting with `:_` run their effect but are dropped from the output.
- Effects in one output map run in unspecified order. Use `:biff.fx/seq` (a sequence of descriptors and maps processed in order) when order matters. A returned vector of output maps is plain data: its effects do not run.
- An initial effect descriptor before the first state runs first: `(defpipeline demo-page [:biff.graph.fx/query [...]] (fn [request result] ...))`.
- The context also carries `:biff.fx/now` (an `Instant`), `:biff.fx/random-uuid4-seq`, `:biff.fx/random-uuid7-seq`, and `:biff.fx/seed` (a long for `(java.util.Random. seed)`), so state functions stay pure: `(let [[new-id] random-uuid7-seq] ...)`.
- Errors are `ex-info` with `:biff.fx/state`, `:biff.fx/machine-name`, and `:biff.fx/trace`.

Testing: call a machine with no arguments to get its state map, or a pipeline with no arguments to get its vector of step functions, then call them directly without running effects.

```clojure
(deftest increment-file-test
  (let [{:keys [start increment]} (increment-file)]
    (is (= {:content [:example.fx/slurp "n.txt"] :biff.fx/next :increment}
           (start {:path "n.txt"})))
    (is (= {:_ [:example.fx/spit "n.txt" "3"] :biff.fx/return 3}
           (increment {:path "n.txt"} {:content "2"})))))
```

## Resolvers (biff.graph)

`(biff.graph/module)` (init only) collects `:biff.graph/resolvers` and `:biff.graph/middleware` from all modules. biff.graph is a small Pathom-like engine: no query planner, no unions or query parameters, batch resolvers and caching supported.

```clojure
(defresolver session-user
  {:output [{:session/user [:user/id]}]}
  [{:keys [session]} _]
  (when-some [uid (:uid session)]
    {:session/user {:user/id uid}}))

(defresolver tab-state
  {:output [{:request/tab [:tab/background-color]}]}

  (fn [{:keys [biff.datastar/tab-id]} _]
    (when tab-id
      [:biff.sqlite.fx/execute
       {:select :* :from :tab-state :where [:= :tab-state/id tab-id]}]))

  (fn [_ [{:tab-state/keys [data]}]]
    {:request/tab (merge {:tab/background-color :white} data)}))

(def module
  {:biff.graph/resolvers [session-user tab-state]})
```

- `defresolver` takes `{:input [...] :output [...] :batch bool}` and then a single `[ctx input]` body, a series of pipeline functions, or machine states (`:start` plus named states). The resolver id is the var's qualified name.
- Query syntax: `[:a :?optional {:join [:x]} {:?optional-join [:x]} {:k [:*]}]`. `:?ns/attr` is shorthand for `[:? :ns/attr]`, and the review rules prefer it. Output attributes are implicitly optional. Input attributes are required unless marked optional. Keys not declared in `:output` are filtered out.
- Run queries with `(biff.graph/query ctx input query)` or the `[:biff.graph.fx/query input query]` effect (omit `input` for global queries such as `{:session/user [...]}`).
- biff.sqlite generates batch resolvers from `columns` (see Database), so `{:user/id uid}` resolves `:user/display-name` and `{:user/email email}` resolves `:user/id` with no hand-written resolver.
- The review rules give a resolver's top-level input and output keys one shared namespace, and use `global` for a resolver that takes no input and reads nothing from the request.
- For read authorization (Iron Law BIFF-3), write resolvers that take ids from path or query params and return the entity as a `:request/*` join (for example `:output [{:request/widget [:widget/id]}]`) only when the current user may read it. The review rules require the `request` namespace. When a required `:request/...` attribute cannot be resolved, the query throws with `:biff.ring/status 400` in its `ex-data`, and the Ring handler returns a 400 response. Query `:?request/...` to handle the missing case yourself.
- `(biff.graph/uml @system)` returns the resolver graph as a Mermaid class diagram. The starter's `start` writes it to `graph.mmd`.
- Test a resolver with `((:biff.graph/resolve-fn my-resolver) {:biff.graph/input {:user/id 1}})`. Errors carry `:biff.graph/trace`.

## Database (biff.sqlite)

`(biff.sqlite/module {...})` (id `:biff.sqlite/module`, after config in `start-order`) opens the database at `:biff.sqlite/db-path`, migrates it, and provides the fx handlers, resolvers, and key-value functions:

```clojure
(ns com.example.schema
  (:require [com.biffweb.sqlite.schema :as s]))

(def columns
  {:user/id           s/pkey
   :user/email        (s/text :required :unique)
   :user/joined-at    (s/inst :required :index)
   :user/display-name (s/text)
   :user/plan         (s/text {:default "free"})
   :user/pet-id       (s/ref :pet/id {:backref :pet/owner})

   :pet/id      s/pkey
   :pet/species (s/enum {0 :pet.species/iguana
                         1 :pet.species/rock})})

(def module
  (biff.sqlite/module
   {:biff.sqlite/columns        schema/columns
    :biff.sqlite/extra-init-sql schema/extra-init-sql
    :biff.sqlite/authorize      #'schema/authorize}))
```

- `com.biffweb.sqlite.schema` builds the column maps: `pkey` is `{:type :uuid :primary-key true}`, the type functions (`text`, `inst`, `int`, `real`, `bool`, `uuid`, `blob`, `enum`, `edn`, `ref`) take the flags `:required`, `:unique`, and `:index` followed by an options map, and `?` is `{:optional true}` for Malli map entries. Plain maps such as `{:type :text :required true}` remain valid.
- Each keyword namespace becomes a STRICT table (`:tab-state/*` becomes `tab_state`).
- Column options: `:type`, `:primary-key` (implies `:required`), `:required` (NOT NULL), `:default` (a SQL DEFAULT, coerced like any value), `:unique`, `:unique-with`, `:ref` (foreign key), `:backref` and `:backref-many` (on a `:ref` column, generate a reverse join returning one map or a vector), `:index`, `:enum-values`, and `:extra-schema` (Malli, for `:edn` columns).
- Types: `:int :real :text :boolean :inst :uuid :enum :edn :blob`. `:inst` is stored as epoch milliseconds and read as `java.time.Instant`. `:edn` is stored as a nippy blob. Enum keywords must be globally unique.
- Generated resolvers, all batch: one per table (primary key in, every other column out, with a `:ref` column ending in `-id` also returned as a join without the suffix, such as `{:user/pet [:pet/id]}`), one per `:unique` or `:unique-with` column (the unique value in, the primary key out), and one per backref (`:pet/id` in, `{:pet/owner [:user/id]}` out). An `:edn` column whose `:extra-schema` describes a map, or a collection of maps, is output as a join that takes `[:*]`.
- Migrations: on startup the module writes `resources/schema.sql` from `columns` and runs sqldef against the database. Only non-destructive changes apply; see the skill's Gotchas for drops.
- Connections use WAL mode, foreign keys, a busy timeout, a Hikari read pool, and a single locked write connection.

Queries and writes take `ctx` first and accept a SQL string, a next.jdbc vector, or a HoneySQL map. Values are coerced only for columns in `columns`; alias computed columns with qualified keywords to get coercion. Wrap keyword values in `[:lift ...]`, because keyword parameters are treated as enum values.

```clojure
(biff.sqlite/execute ctx {:select :* :from :pet
                          :where [:= :pet/species [:lift :pet.species/iguana]]})
(biff.sqlite/execute ctx {:select [[[:max :user/joined-at] :user/joined-at]] :from :user})
```

| Function | fx effect | Use |
|----------|-----------|-----|
| `execute` | `:biff.sqlite.fx/execute` | Reads and system writes |
| `execute-one` | `:biff.sqlite.fx/execute-one` | Like `execute`, returning the first row or nil |
| `execute-tx` | `:biff.sqlite.fx/execute-tx` | Several statements in one transaction |
| `authorized-write` | `:biff.sqlite.fx/authorized-write` | Writes on behalf of the current user |
| `authorized-write-tx` | `:biff.sqlite.fx/authorized-write-tx` | Several user writes in one transaction |

`authorized-write` computes a diff `[{:table :op :before :after}]` (`:op` is `:create`, `:update`, or `:delete`) and calls `:biff.sqlite/authorize` `(fn [ctx diff]) => boolean`. It rejects `REPLACE INTO` and primary-key changes. The starter's `schema.clj` authorizes per table:

```clojure
(defn authorize-entry [{{:keys [uid]} :session :keys [biff.datastar/tab-id]}
                       {:keys [table op before after]}]
  (case table
    :user      (and (every? #{uid} (keep :user/id [before after]))
                    (case op
                      :create false
                      :update (only-fields-edited? before after editable-user-fields)
                      :delete true))
    :tab-state (every? #{tab-id} (keep :tab-state/id [before after]))
    false))

(defn authorize [ctx diff]
  (every? #(authorize-entry ctx %) diff))
```

Interactive transactions replace the write connection in `ctx`:

```clojure
(jdbc/with-transaction [tx (:biff.sqlite/write-conn ctx)]
  (let [ctx (assoc ctx :biff.sqlite/write-conn tx)]
    ...))
```

Inspect data from `repl.clj` with `(execute (get-ctx) "select * from user")`.

Backups: Litestream runs only when `:biff.sqlite/litestream-access-key-id` is set, and then `:biff.sqlite/litestream-secret-access-key` (a secret) and `:biff.sqlite/litestream-bucket` are required (`-endpoint`, `-region`, and `-replica-path`, the path within the bucket, are optional). The starter's `config.edn` reads them from the `LITESTREAM_*` variables listed in `resources/TEMPLATE.config.prod.env`. With no local database file it restores first, then replicates in the background.

## Background Work (biff.background)

`(biff.background/module)` (id `:biff.background/module`) runs scheduled tasks with chime and in-memory priority queues on thread pools.

```clojure
(defn every-5-minutes []
  (iterate #(.plusSeconds % (* 60 5)) (java.time.Instant/now)))

(defn send-digest [ctx] ...)

(defn handle-email [{:biff.background/keys [job] :as ctx}] ...)

(def module
  {:biff.background/tasks  [{:schedule every-5-minutes :task #'send-digest}]
   :biff.background/queues {:com.example/email {:consumer  #'handle-email
                                                :n-threads 2}}})

(biff.background/submit-jobs ctx :com.example/email [{:to "a@example.com"}])

;; inside a pipeline or machine:
{:_ [:biff.background.fx/submit-jobs :com.example/email [{:to "a@example.com"}]]}
```

- Task maps take `:schedule` (zero-argument function returning a sequence of `Instant`s), `:task` (receives the system map), and the optional chime callbacks `:error-handler` and `:on-finished`.
- Queue options: `:consumer`, `:n-threads` (default 1), `:queue`. Consumers receive `:biff.background/job` and `:biff.background/queue` (use `.drainTo` on the queue to batch).
- `:biff.background/priority` on a job orders processing; lower runs first, default 10.
- Submitting to an unknown queue throws. Jobs are not persisted and a consumer exception drops the job. Work that must survive a restart needs a database-backed record that a scheduled task re-submits.

## Authentication (biff.authenticate)

`(biff.auth/module {...})` (alias `biff.auth` for `com.biffweb.authenticate`) adds email sign-in with six-digit codes:

| Route | Behavior |
|-------|----------|
| `GET /signin` | Default sign-in page (disable with `:biff.auth/include-signin-page false`) |
| `POST /_biff/auth/send-code` | Sends a code; redirects to the sign-in page with `sent-to=` or `error=` |
| `POST /_biff/auth/verify-code` | Sets `(:uid session)` and redirects to `:biff.auth/app-path` (default `/app`) |
| `POST /_biff/auth/signout` | Clears the session and redirects to `/` |

The application supplies `:biff.auth/get-user-id` `(fn [ctx email])`, `:biff.auth/create-user` `(fn [ctx {:keys [email params]}]) => user-id` (must tolerate races; the starter upserts on `:user/email`), and `:biff.auth/send-email` `(fn [ctx {:keys [to subject html text code]}]) => boolean`. The starter writes `get-user-id` and `create-user` as pipelines and merges `biff.auth/turnstile-config` (`recaptcha-config` and `hcaptcha-config` also exist). Without captcha configuration or `:biff.auth/skip-captcha true`, the app still starts, but every request to the sign-in routes, including `GET /signin`, fails the assertion "Captcha is not configured" and returns a 500. The starter's dev `config.env` sets `SKIP_CAPTCHA=true`, and its `send-email` prints the message, including the code, to the console when `:mailersend/api-key` is unset.

Codes expire after 10 minutes (`:biff.auth/code-expiry-minutes`) and lock after 5 failed attempts (`:biff.auth/max-failed-attempts`).

## Admin Dashboard (biff.admin)

`(biff.admin/module {...})` (id `:biff.admin/module`, after the database module and before `:biff.background/module` and `:biff.ring/module`) serves `/_biff/admin`: usage and revenue metrics, a user directory with impersonation, performance profiling, and email alerts for logged errors. It takes `:biff.admin/get-users`, `:biff.admin/get-usage-events`, `:biff.admin/get-revenue-events`, and `:biff.admin/send-email`. Access is limited to `:biff.admin/admin-user-id` (the `ADMIN_USER_ID` variable); when unset, the page shows the current user id so it can be copied into config. `:biff.admin/alert-email` receives error alerts.

## Tasks and REPL Workflow (biff.tasks)

Run tasks with `clojure -M:run <task>`; `clojure -M:run -h` lists them and `clojure -M:run <task> -h` prints a task's documentation (except `test`, which passes `-h` on to Kaocha).

| Task | Purpose |
|------|---------|
| `dev` | Run `init`, watch Tailwind, evaluate changed files and dependants on save, call the main namespace's `-main` (the starter's `-main` starts nREPL and is meant to use `:biff.tasks/nrepl-port`, default 7888, but see the skill's Gotchas) |
| `init` | Rename `com.example` (prompting on stdin), generate `config.env` and `config.prod.env`, copy Biff's agent skills into `.agents/skills/`, download the pinned clj-kondo, cljfmt, and Tailwind versions into `target/bin/` when they are not installed, offer to reinitialize git |
| `nrepl` | Start an nREPL server without starting the app |
| `code-quality` | Update clj-kondo configs, then `format`, `lint`, `test` |
| `format`, `lint`, `test` | cljfmt and clj-kondo over every Git-tracked Clojure and EDN file outside hidden directories, and Kaocha. `format` first inserts a blank line between adjacent pairs when either pair puts its key and value on separate lines |
| `add <lib>` | Add the latest release of a dependency to `deps.edn` |
| `update` | Update dependencies with antq and refresh clj-kondo configs |
| `css` | Run the Tailwind standalone binary |
| `uberjar` | Build `target/jar/app.jar`. It runs `css --minify` unless `:biff.tasks/build-jar-resources` holds a symbol for a zero-argument function that writes the resources instead |
| `prod-setup`, `deploy`, `prod-logs`, `prod-nrepl`, `prod-restart` | Server provisioning and deployment (see `/biff-deploy`) |

- `repl.clj` provides `(get-ctx)` and `(refresh)`. Saving a file evaluates it; `(refresh)` stops the system, reloads namespaces, and starts again, which is needed only for startup-only code such as `columns` or `start-order`.
- Agents evaluate changes, lint, and test against the running app with `trench -p $(cat .nrepl-port) -e "((requiring-resolve 'com.biffweb.tasks/agent-refresh))"` (trenchman from https://github.com/athos/trenchman/releases/latest). It returns `:status :ok` or `:status :error` with `:out` and `:err`.
- The `:biff/impl-visibility` clj-kondo rule (an error in the starter) forbids requiring a namespace with an `impl` segment from outside its parent.
- The starter's `.cljfmt.edn` aligns map and form columns; keep lines within 100 characters (a clj-kondo warning).

Custom tasks merge into `com.biffweb.tasks/app-tasks` behind a project `-main`:

```clojure
;; dev/tasks.clj
(ns tasks
  (:require [com.biffweb.run :as biff.run]
            [com.biffweb.tasks :as biff.tasks]))

(def tasks
  (merge biff.tasks/app-tasks
         {"seed" {:task 'tasks.seed/seed :doc "Insert sample data."}}))

(defn -main [& args]
  (apply biff.run/main tasks args))

;; deps.edn: point the :run alias at it
;; :run {:extra-paths ["dev" "test"] :main-opts ["-m" "tasks"] ...}
```

Keep `dev/tasks.clj` free of heavy requires; tasks are referenced by symbol and loaded on demand.

## XTDB 2 Alternative (biff.xtdb)

`com.biffweb/xtdb` replaces biff.sqlite with XTDB v2. biff.xtdb 2.0.3 depends on XTDB 2.1.0, so it needs Java 21 or later but earlier than 25, plus these JVM options:

```clojure
:jvm-opts ["--add-opens=java.base/java.nio=ALL-UNNAMED"
           "--enable-native-access=ALL-UNNAMED"
           "-Dio.netty.tryReflectionSetAccessible=true"]
```

It provides `module` (id `:biff.xtdb/module`, storage under `storage/xtdb2/` by default, `:biff.xtdb/storage :memory` and `:biff.xtdb/log :memory` for tests), `schema-module`, `q` (HoneySQL), `execute-tx`, `submit-tx`, `authorized-write` (document operations only), the custom transaction operations `:biff/upsert` and `:biff/assert-unique`, and the fx effects `:biff.xtdb.fx/execute-tx`, `:biff.xtdb.fx/submit-tx`, and `:biff.xtdb.fx/authorized-write`. Writes made through `execute-tx`, `submit-tx`, and `authorized-write` trigger `:biff.core/on-tx` as soon as XTDB indexes them, and a listener checks about once per second for transactions made by other means.
