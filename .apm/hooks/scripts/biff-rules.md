# Biff rules from biff-skills

This is a Biff 2 project. The Iron Laws below come from the `biff` skill and add to the `clojure` baseline laws. Never write code that breaks one. Before writing, reviewing, or refactoring code in this project, load the `biff` skill along with the `clojure` and `clojure-jvm` skills. The skills hold the full laws, their exceptions, and the references.

## Iron Laws

- BIFF-1: Route user writes through `:biff.sqlite.fx/authorized-write` with HoneySQL maps, and add an `authorize` case for every new table. Use plain `execute` only for system writes, or for a user write the rules cannot validate, and then check authorization in the handler.
- BIFF-2: Authorize from the diff and the session user only, never from information a call site passes in.
- BIFF-3: Authorize every request-derived read in a biff.graph resolver whose output is a `:request/*` join, because biff.sqlite has no read authorization.
- BIFF-4: Read configuration only from the system map. Declare values in `resources/config.edn` with `#biff/env`, wrap secrets with `#biff/secret`, and unwrap them with `force`.
- BIFF-5: Edit `schema.clj`, never the generated `resources/schema.sql`, and run `(refresh)` after schema changes.
- BIFF-6: Serve browser endpoints from `:biff.ring/routes`, which have sessions and CSRF protection. `:biff.ring/api-routes` are for external APIs only.
