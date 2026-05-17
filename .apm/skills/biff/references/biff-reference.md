# Biff Reference

Deeper detail for the `biff` skill. Sections are grouped by topic; load whichever applies.

## Routing

Biff uses Ring with Reitit. Routes live inside a module under `:routes` (CSRF protected) or `:api-routes` (CSRF disabled).

### Basics

```clojure
(defn foo [_]
  {:status 200
   :headers {"content-type" "text/plain"}
   :body "foo response"})

(def module
  {:routes [["/foo" {:get foo}]
            ["/bar" {:post bar}]]})
```

### Path parameters

```clojure
(defn click [{:keys [path-params]}]
  (println (:token path-params))
  ...)

(def module {:routes [["/click/:token" {:get click}]]})
```

### Nested routes

```clojure
(def module
  {:routes [["/auth/" ["send"           {:post send-token}]
                      ["verify/:token"  {:get  verify-token}]]]})
```

### Middleware via route data

```clojure
(defn wrap-signed-in [handler]
  (fn [{:keys [session] :as ctx}]
    (if (some? (:uid session))
      (handler ctx)
      {:status 303 :headers {"location" "/"}})))

(def module
  {:routes [["/app" {:middleware [wrap-signed-in]}
             [""         {:get  app}]
             ["/set-foo" {:post set-foo}]]]})
```

### Public API routes (CSRF disabled)

```clojure
(defn echo [{:keys [params]}]
  {:status 200
   :headers {"content-type" "application/json"}
   :body params})

(def module {:api-routes [["/echo" {:post echo}]]})
```

### Rum responses

`biff/wrap-render-rum` renders vector responses to HTML. These two handlers are equivalent:

```clojure
(defn my-handler [_]
  {:status 200
   :headers {"content-type" "text/html"}
   :body (rum/render-static-markup
          [:html [:body [:p "Hello"]]])})

(defn my-handler [_]
  [:html [:body [:p "Hello"]]])
```

### htmx handlers

Return a partial vector when responding to an htmx swap:

```clojure
(defn click [_]
  [:div "Earth will now self-destruct"])

(def module
  {:routes [["/page"  {:get  page}]
            ["/click" {:post click}]]})
```

### Forms with CSRF

`biff/form` injects the anti-forgery token automatically:

```clojure
(defn signin [_]
  [:html [:body
          (biff/form {:action "/signin"}
                     [:input {:type "email" :name "email"}]
                     [:button {:type "submit"} "Sign in"])]])
```

Manual approach (do not use unless you have a reason):

```clojure
[:input {:type "hidden"
         :name "__anti-forgery-token"
         :value csrf/*anti-forgery-token*}]
```

The starter project's `com.example.ui/page` helper also injects the CSRF token into htmx requests triggered by child elements.

### Out-of-band swaps

To update a region outside the htmx target, render a fragment with `hx-swap-oob`:

```clojure
[:div#messages {:hx-swap-oob "beforeend"}
 [:p "new message: " text]]
```

Send these from an `:on-tx` listener when broadcasting across web sockets so every server fans out.

### WebSockets

```clojure
(require '[ring.adapter.jetty9 :as jetty])

(defn chat-ws [{:keys [example/chat-clients]}]
  {:status 101
   :headers {"upgrade" "websocket" "connection" "upgrade"}
   :ws {:on-connect (fn [ws] (swap! chat-clients conj ws))
        :on-text    (fn [_ text]
                      (doseq [ws @chat-clients]
                        (jetty/send! ws
                                     (rum/render-static-markup
                                      [:div#messages {:hx-swap-oob "beforeend"}
                                       [:p "new message: " text]]))))
        :on-close   (fn [ws _ _] (swap! chat-clients disj ws))}})
```

For cross-server fan-out, push the broadcast into an `:on-tx` listener instead of calling `jetty/send!` from the handler.

## Static files

Static HTML files are declared as a map of path to Rum data:

```clojure
(def about-page
  (ui/page {:base/title (str "About " settings/app-name)}
           [:p "This app was made with "
            [:a.link {:href "https://biffweb.com"} "Biff"] "."]))

(def module {:static {"/about/" about-page}})
```

Paths ending in `/` get `index.html` appended. Tailwind CSS classes work inline (`[:button.bg-blue-500.text-white ...]`). Anything in `resources/public/` is served as-is.

## Schema (Malli)

XTDB does not enforce schema. Biff enforces it via Malli.

```clojure
(def schema
  {:user/id :uuid
   :user    [:map {:closed true}
             [:xt/id           :user/id]
             [:user/email      :string]
             [:user/joined-at  inst?]
             [:user/foo {:optional true} :string]
             [:user/bar {:optional true} :string]]})
```

For documents that reference a user, use the aliased ID type so the relation is obvious:

```clojure
(def schema
  {:user/id :uuid
   :msg/id  :uuid
   :msg     [:map {:closed true}
             [:xt/id    :msg/id]
             [:msg/user :user/id]
             ...]})
```

For mild convenience, use `biff/doc-schema`:

```clojure
(ns com.example.schema
  (:require [com.biffweb :refer [doc-schema] :rename {doc-schema doc}]))

(def schema
  {:user/id :uuid
   :user    (doc {:required [[:xt/id          :user/id]
                             [:user/email     :string]
                             [:user/joined-at inst?]]
                  :optional [[:user/foo :string]
                             [:user/bar :string]]})})
```

Biff does not enforce that `:msg/user` actually points to an existing user document; only the value type is checked.

## Transactions

`xt/submit-tx` works:

```clojure
(require '[xtdb.api :as xt])

(defn send-message [{:keys [biff.xtdb/node session params]}]
  (xt/submit-tx node
                [[::xt/put {:xt/id       (java.util.UUID/randomUUID)
                            :msg/user    (:uid session)
                            :msg/text    (:text params)
                            :msg/sent-at (java.util.Date.)}]]))
```

`biff/submit-tx` is the higher-level helper. It enforces schema, awaits the transaction, and supports sentinel keywords:

```clojure
(require '[com.biffweb :as biff])

(defn send-message [{:keys [session params] :as ctx}]
  (biff/submit-tx ctx
                  [{:db/doc-type :message
                    :msg/user    (:uid session)
                    :msg/text    (:text params)
                    :msg/sent-at :db/now}]))
```

Behavior:

- `:xt/id` defaults to `(java.util.UUID/randomUUID)`.
- `:db/op` defaults to `:xtdb.api/put`.
- `:db/now` is replaced with `(java.util.Date.)`.
- Any `:db.id/<name>` keyword is replaced with a fresh random UUID; reuse the same keyword in the transaction to link new documents.
- Plain XT operations (non-map elements) are passed through.

### Document operations

| `:db/op` | Behavior |
|----------|----------|
| (default) `:xtdb.api/put` | Replace the entire document. |
| `:merge` | Merge into the existing document, or create it if missing. |
| `:update` | Merge into the existing document. Transaction fails if the document does not exist. |
| `:create` | Create the document. Transaction fails if it already exists. |
| `:delete` | Delete the document. |

Biff inserts `:xtdb.api/match` operations behind the scenes for `:merge` and `:update` so concurrent writes do not silently overwrite each other.

### Upsert by attribute (`:db.op/upsert`)

`:db.op/upsert` (note the namespace: `db.op`, not `db/op`) updates a document matched by the given attribute-value pairs, or creates a new one with a fresh `:xt/id` if no match exists. This is the recommended way to express the "create-if-missing, otherwise merge" pattern:

```clojure
(biff/submit-tx ctx
                [{:db/doc-type     :user
                  :db.op/upsert    {:user/email "hello@example.com"}
                  :user/joined-at  :db/now}])
```

The operation is atomic via a transaction function and requires `:biff/ensure-unique` to be installed. New projects install it by default; if you build a system map yourself, pass `:tx-fns biff/tx-fns` to `use-xtdb`. Prefer `:db.op/upsert` over the older `:db/lookup` form, which is deprecated.

### Per-attribute operations

A handful of sentinels operate on a single attribute and use its previous value:

| Sentinel | Effect |
|----------|--------|
| `[:db/union & vs]` | Coerce the previous value to a set, then `clojure.set/union` in the new values. |
| `[:db/difference & vs]` | Set-difference the listed values from the previous value. |
| `[:db/add n]` | Add `n` to a numeric attribute. |
| `[:db/default v]` | Set the attribute only if it is currently absent. |
| `:db/dissoc` | Remove the attribute. |
| `[:db/unique v]` | Set the attribute and abort if any other document has the same value. Requires `:biff/ensure-unique`. |

```clojure
[{:db/op       :update
  :db/doc-type :post
  :xt/id       post-id
  :post/tags   [:db/union "clojure" "almonds"]
  :post/views  [:db/add 1]
  :post/draft  :db/dissoc}]
```

### Contention and retries

Biff retries up to three times on contention. To re-read state between retries, pass a function that returns the transaction:

```clojure
(biff/submit-tx ctx
                (fn [{:keys [biff/db]}]
                  [{:db/doc-type :user
                    :db/op       :merge
                    :xt/id       (or (biff/lookup-id db :user/email email)
                                     (java.util.UUID/randomUUID))
                    :user/email  email}
                   [::xt/fn :biff/ensure-unique {:user/email email}]]))
```

The `:biff/ensure-unique` XT function is built in and useful for enforcing uniqueness inside the transaction.

## Queries

XTDB datalog works directly. Biff provides two convenience helpers.

### `biff/q`

A light wrapper around `xt/q`. It throws on an `:in` arity mismatch and unwraps scalars when `:find` is a single symbol (not a vector):

```clojure
(require '[xtdb.api :as xt])
(require '[com.biffweb :as biff])

;; Vector :find -- tuples
(map first
     (xt/q db '{:find  [email]
                :where [[user :user/email email]]}))

;; Scalar :find -- already unwrapped
(biff/q db '{:find  email
             :where [[user :user/email email]]})
```

### `biff/lookup` and friends

Like `xt/pull` but keyed on attribute-value pairs:

```clojure
(biff/lookup db :user/email "bob@example.com")
;; => {:xt/id #uuid "..."
;;     :user/email "bob@example.com"
;;     :user/favorite-color :chartreuse
;;     :user/best-friend #uuid "..."}

(biff/lookup db '[* {:user/best-friend [*]}]
             :user/email "bob@example.com")
```

Related helpers: `biff/lookup-id`, `biff/lookup-all`, `biff/lookup-id-all`.

## Authentication

Biff ships a passwordless authentication module: email links for signup, email codes for signin (the defaults the starter uses). The module provides backend routes; UI and email templates live in your project so they can be customized.

```clojure
(def modules
  [app/module
   (biff/authentication-module {})
   home/module
   schema/module])
```

Email delivery uses MailerSend in the starter. Until MailerSend and Recaptcha API keys are set in `config.env`, signin links and codes are printed to the console instead of emailed.

A successful auth stores the user ID in the session (`{:uid uuid}`). Sessions are encrypted cookies. A user document is auto-created for new sign-ins.

```clojure
(defn whoami [{:keys [session biff/db]}]
  (let [user (xt/entity db (:uid session))]
    [:html [:body
            [:p "Signed in: " (some? user)]
            [:p "Email: "     (:user/email user)]]]))

(def module {:routes [["/whoami" {:get whoami}]]})
```

### Authorization

Use route-level middleware. The starter ships `wrap-signed-in`:

```clojure
(defn wrap-signed-in [handler]
  (fn [{:keys [session] :as ctx}]
    (if (some? (:uid session))
      (handler ctx)
      {:status  303
       :headers {"location" "/signin?error=not-signed-in"}})))
```

For roles, store them on the user document and check in middleware:

```clojure
(defn wrap-admin [handler]
  (fn [{:keys [biff/db session] :as ctx}]
    (let [user (xt/entity db (:uid session))]
      (if (contains? (:user/roles user) :admin)
        (handler ctx)
        {:status 403
         :headers {"content-type" "text/plain"}
         :body "Unauthorized."}))))
```

### Secret rotation

`config.env` contains two secrets for session cookies and JWTs. Rotate them with:

```bash
clj -M:dev generate-secrets
```

## Modules and components

The `:biff/modules` value in the system map can be a var (`#'modules`) so that late binding picks up changes without a restart.

### Components are functions

```clojure
(defn use-jetty [{:keys [biff/handler] :as system}]
  (let [server (jetty/run-jetty
                (fn [request] (handler (merge system request)))
                {:host "localhost" :port 8080 :join? false})]
    (update system :biff/stop conj #(jetty/stop-server server))))
```

A component takes the system map, does setup, and returns the modified system (typically with a stop function pushed onto `:biff/stop`).

### Refresh

`refresh` stops every component, then reloads namespaces:

```clojure
(defn refresh []
  (doseq [f (:biff/stop @system)]
    (log/info "stopping:" (str f))
    (f))
  (clojure.tools.namespace.repl/refresh :after `start))
```

Most edits do not need this. Late binding handles handler and module changes. Reach for `refresh` when components or the system map itself change.

## Scheduled tasks

Biff uses [chime](https://github.com/jarohen/chime) for `:tasks`. Each task takes a function, a zero-arity schedule function that returns a (possibly infinite) sequence of times, and an optional error handler:

```clojure
(require '[com.biffweb :as biff :refer [q]])

(defn print-usage [{:keys [biff/db]}]
  (let [n-users (first (q db '{:find (count user)
                               :where [[user :user/email]]}))]
    (println "There are" n-users "users.")))

(defn every-minute []
  (iterate #(biff/add-seconds % 60) (java.util.Date.)))

(defn error-handler [error]
  (println "Uh oh!"))

(def module
  {:tasks [{:task          #'print-usage
            :schedule      every-minute
            :error-handler error-handler}]})
```

## Transaction listeners

`:on-tx` receives every transaction. Useful for fan-out and side effects keyed on writes:

```clojure
(defn alert-new-user [{:keys [biff.xtdb/node]} tx]
  (let [db-before (xt/db node {::xt/tx-id (dec (::xt/tx-id tx))})]
    (doseq [[op & args] (::xt/tx-ops tx)
            :when (= op ::xt/put)
            :let  [[doc] args]
            :when (and (contains? doc :user/email)
                       (nil? (xt/entity db-before (:xt/id doc))))]
      (println "there's a new user"))))

(def module {:on-tx alert-new-user})
```

## Queues

In-memory concurrent queues, each backed by a thread pool. Jobs are not persisted and there is no retry on exception.

```clojure
(defn echo-consumer [{:keys [biff/job]}]
  (prn :echo job)
  (when-some [callback (:biff/callback job)]
    (callback job)))

(def module
  {:queues [{:id        :echo
             :consumer  #'echo-consumer
             :n-threads 1}]})

(biff/submit-job ctx :echo {:foo "bar"})
@(biff/submit-job-for-result ctx :echo {:foo "bar"})
```

Use queues to bound concurrency for things like outbound HTTP. Anything that must survive a restart needs a persistent mechanism.

## Project tasks (`clj -M:dev`)

Tasks live in `dev/tasks.clj` and run as `clj -M:dev <task>`. List them with `clj -M:dev --help`.

Built-in tasks include `clean`, `css`, `dev`, `deploy`, `generate-config`, `generate-secrets`, `logs`, `prod-dev`, and others. Add custom tasks to `custom-tasks` in `dev/tasks.clj`. Keep startup fast by loading libraries via `requiring-resolve`:

```clojure
(defn frobnicate
  "Frobnicates your project."
  []
  ((requiring-resolve 'foo.bar/frobnicate)))

(def custom-tasks
  {"frobnicate" #'frobnicate})
```

Convenience alias:

```bash
alias biff='clj -M:dev'
```

## REPL workflow

`dev/repl.clj` is the entry point for REPL-driven work. With `clj -M:dev dev` running:

- File saves evaluate changed namespaces (and their dependents) and re-run tests.
- `(refresh)` cycles components when the system map or component definitions change.
- `(reset)` (where defined) is a convenience for cycling and replaying fixtures.

Production REPL access is available via `clj -M:dev prod-dev`, which copies saved files to the running production server for evaluation. Use deliberately; this is not a sandbox.
