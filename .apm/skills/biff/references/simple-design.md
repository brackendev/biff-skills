# Simple Design in Biff

Part of the `biff` skill, for Biff 2 applications. It shows how Biff's own constructs carry out Key Rule 20 of the host-neutral `clojure` skill ("Choose simple over easy"). Read `clojure/references/simple-design.md` in the `clojure` skill first, and `clojure-jvm/references/simple-design.md` in the `clojure-jvm` skill for the JVM constructs. Their rules and exceptions apply here unchanged.

Biff 2 already keeps apart most of the concerns that those references name. In a Biff project the rule is mostly this: use the Biff construct, and do not build a second mechanism beside it. The example below was run against biff.fx 2.0.3.

## What Biff already keeps apart

| Concern | Biff construct | Do not add |
|---------|----------------|------------|
| A decision and the effect that carries it out | A `defpipeline` or `defmachine` whose state functions are pure and return effect descriptors (Key Rule 4) | I/O inside a state function |
| Who does the work | The system map, merged with the request and passed to every handler | A read of the global `system` atom or of an environment variable (Iron Law BIFF-4) |
| What a feature provides | A module map whose keys the libraries collect | Registration that runs as a side effect when a namespace loads (Iron Law CLJ-8) |
| What data a page needs | A biff.graph query answered by resolvers, and HoneySQL maps for SQL (Key Rule 9) | SQL text built in a handler, or a join resolver where a declared backref would do |
| Why a write is allowed | The `authorize` function in `schema.clj` (Iron Laws BIFF-1 and BIFF-2) | Permission checks spread across handlers |
| When work runs | `biff.background` queues and tasks (Key Rule 10) | A second queue library beside it |
| The time and random values | `:biff.fx/now`, the UUID sequences, and `:biff.fx/seed` in the context | A call to the clock or to a random source inside a state function |
| What the page shows | One handler that renders the page from the database, re-run after every write (Key Rule 5) | Signals for anything other than form inputs (Key Rule 7) |

## Keep state functions pure

A state function decides. It returns data that names the effects, and biff.fx runs them. The decision can then be tested by calling the function, with no database and no queue.

```clojure
;; bad: the step reads the file, decides, and writes, so a test needs the file
(defpipeline increment-file
  (fn [{:keys [path]}]
    (let [new-n (inc (or (some-> (slurp path) parse-long) 0))]
      (spit path (str new-n))
      new-n)))

;; good: the first step asks for the data, and the second decides and names the effect
(defpipeline notify-price-drop
  (fn [_ctx product-id]
    [:biff.sqlite.fx/execute
     {:select [:product/price :product/watch-price :product/watcher-email]
      :from   :product
      :where  [:= :product/id product-id]}])

  (fn [_ctx [{:product/keys [price watch-price watcher-email]}]]
    (when (< price watch-price)
      {:_ [:biff.background.fx/submit-jobs :com.example/email
           [{:to watcher-email :text (str "Now " price)}]]})))
```

Call the pipeline with no arguments to get its step functions, and test each one as a plain function:

```clojure
(deftest notify-price-drop-test
  (let [[query decide] (notify-price-drop)
        row {:product/price 5 :product/watch-price 9 :product/watcher-email "a@example.com"}]
    (is (= :product (:from (second (query {} 7)))))
    (is (= {:_ [:biff.background.fx/submit-jobs :com.example/email
                [{:to "a@example.com" :text "Now 5"}]]}
           (decide {} [row])))
    (is (nil? (decide {} [(assoc row :product/price 9)])))))
```

- The second step returns `nil` when there is nothing to do. The pipeline then returns `nil` and runs no effect.
- Take the time and random values from the context, so the state function returns the same result for the same arguments.
- Keep each application effect handler to the I/O alone, in the project's single `fx.clj` namespace (`references/biff-reference.md`, "Effects"). A handler that also decides puts the rule back where a test cannot reach it without the I/O.
- An atom in the system map changes through the `[:biff.fx/swap! ...]` effect, so that change is also data that a test can read.

## Receive collaborators from the context

Every handler, state function, resolver, task, and queue consumer receives the context: the system map, merged with the request where there is one. That map is how a function gets its collaborators.

- Destructure what the function needs at the top. Pass those values, not the whole context, to the helpers in `lib/`. A helper that takes the context can read anything in it, so it depends on the layout of the whole system. This point is this package's guidance.
- Never reach around the context to the global `system` atom or to an environment variable (Iron Law BIFF-4).
- Module namespaces never require each other (Key Rule 1). They meet only in `modules.clj`, which places them side by side.

## Use biff.background as the queue

`biff.background` is the queue between a handler and work that should not hold up the response. Declare the queue in a module, submit jobs with the `:biff.background.fx/submit-jobs` effect, and let the consumer read the job from the context (`references/biff-reference.md`, "Background Work").

- A job is plain data. Put ids and values in it, not connections or functions.
- The handler knows only the queue's name and the shape of the job, so the consumer can change without a change to the handler.
- Do not add a core.async channel or a `LinkedBlockingQueue` beside it for application work. Two queue mechanisms give the program two sets of rules for failure and shutdown.
- Jobs live in memory, are not retried, and are lost on a restart. Work that must survive a restart needs a database record that a scheduled task submits again (`references/biff-reference.md`, "Background Work").
- For work that the response depends on, keep the direct call or the effect in the pipeline. A queue there only delays the answer.

## Declare data, and let the libraries work out how

- **Queries.** Ask biff.graph for attributes, and declare backrefs in `columns` instead of writing join resolvers (Key Rule 9). A resolver states its `:input` and `:output`, and the graph finds the path.
- **Schema.** The `columns` map is the description, and Biff generates `schema.sql` and the batch resolvers from it (Iron Law BIFF-5).
- **Routes.** A route is a Reitit vector, and every URL comes from a `defpath` function (Key Rule 3).
- **Authorization.** The rules in `authorize` see the diff and the session user and nothing a call site passes in (Iron Law BIFF-2). The policy can then be read in one place.

## When the Biff construct is not the right choice

- A system write, such as one from a background job, uses plain `execute` (Iron Law BIFF-1).
- A read that depends on the request needs a resolver that authorizes it, because biff.sqlite has no read authorization (Iron Law BIFF-3).
- A query that is too slow for a page that re-renders after every write moves into an action that stores its result in tab state (Key Rule 8).
