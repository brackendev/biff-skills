---
name: biff-new
description: "Scaffold a new Biff 2 web application from the official starter project"
argument-hint: "<project-name>"
user-invocable: true
disable-model-invocation: true
---

# Scaffold a Biff Project

Create a new [Biff](https://github.com/jacobobryant/biff) 2 web application by cloning the official starter project, [jacobobryant/biff-starter](https://github.com/jacobobryant/biff-starter). The starter is a small app with email sign-in, a Datastar demo page, an admin dashboard, and SQLite storage (see the `biff` skill's Project Layout section).

## Arguments

| Input             | Target                                                                       |
|-------------------|------------------------------------------------------------------------------|
| `<project-name>`  | Required. The directory to clone the starter into. The main namespace is collected separately. |
| (no argument)     | Prompt the operator for a project directory and main namespace before cloning. |

This skill is exempt from the `all` and `<path>` rows of the standard scope vocabulary because scaffolding has no useful default scope.

## Mutation

Mutates by default: clones the starter into `<project-name>`, then runs `clj -M:run init`, which rewrites the `com.example` namespace throughout the project, generates `config.env` and `config.prod.env` with fresh secrets, downloads clj-kondo, cljfmt, and Tailwind into `target/bin/`, and (if the operator agrees) replaces the starter's git history with a single new commit. It then starts `clj -M:run dev` once to verify the app responds. No `--report` flag; preview the side effects by reading this `SKILL.md` and the starter repository.

## Prerequisites

Verify before proceeding. If anything is missing, stop and tell the user.

- Java 21 or higher: `java -version` (Biff 2.0.1 raised the minimum from 17; the starter README still says 17)
- Clojure CLI: `clj --version`
- Git: `git --version`
- Outbound network access to GitHub, Maven Central, and Clojars
- The target directory does not already exist

## Steps

### 1. Collect and confirm the inputs

Use `$ARGUMENTS` as the project directory if provided. Otherwise ask for one. Then ask for:

- The main namespace (for example `com.example.my-app`). It must differ from `com.example`, which `init` treats as "not yet renamed".
- Whether to replace the starter's git history with a fresh repository. Recommend yes for a new project; the starter's history is not useful to the new app.

Confirm all three values with the user before running anything.

### 2. Clone the starter

```bash
git clone https://github.com/jacobobryant/biff-starter <project-dir>
cd <project-dir>
```

### 3. Initialize the project

`init` prompts for the namespace and then for the git choice. Pipe the confirmed answers in:

```bash
printf '%s\n%s\n' '<namespace>' '<Y-or-n>' | clj -M:run init
```

`init` renames `com.example` in every tracked file, moves `src/com/example*` and `test/com/example_test.clj` to the new path, and updates `:biff.tasks/main-ns` in `resources/config.edn`. The namespace keeps hyphens (`com.my-app`) and the directory path uses underscores (`src/com/my_app/`). `init` writes both shapes, so do not rename either by hand. With `Y`, it deletes `.git`, runs `git init`, and commits everything as "First commit".

If `init` fails with `Failed to install correct version of binary` for `tailwindcss`, check whether macOS killed the downloaded binary:

```bash
target/bin/tailwindcss --help; echo $?
```

An exit status of 137 means macOS rejected the binary's code signature. Re-sign it locally with an ad hoc signature and re-run `init`, which skips the prompts once the namespace is set:

```bash
codesign --force -s - target/bin/tailwindcss
clj -M:run init </dev/null
```

Report this step to the user; it changes a downloaded executable.

### 4. Verify the project starts

Start the dev server in the background and capture its log:

```bash
clj -M:run dev > dev.log 2>&1 &
```

Port 8080 is the default. If another process already listens there (`lsof -nP -iTCP:8080 -sTCP:LISTEN`), set `PORT` for this run (for example `PORT=8081 clj -M:run dev`) instead of stopping the other process. Wait for `System started.` in the log, then check:

```bash
curl -fsS -o /dev/null http://localhost:8080/
```

If the server never responds, show the relevant part of `dev.log` and report the failure rather than proceeding. After a successful check, stop the dev server you started (only that process) and delete `dev.log`, or leave the server running if the user wants to explore the app.

### 5. Report

Print the project location, the main namespace, and next steps:

```
Created Biff project at <path>.
Main namespace: <namespace>

Next steps:
  cd <project-dir>
  clj -M:run dev            # Start the app on localhost:8080
  clj -M:run -h             # List project tasks
  clj -M:run code-quality   # Format, lint, and test

Sign in at http://localhost:8080/signin. In development the sign-in code is
printed to the dev console instead of emailed.

Edit src/<namespace_path>/schema.clj to change the data model.
Add pages and actions under src/<namespace_path>/app/.
Read AGENTS.md for the agent workflow (agent-refresh and code-quality).
```

## Gotchas

- **`clj -M:run dev` runs `init` itself when needed.** Running `dev` first on a fresh clone prompts for the namespace inside the long-running dev process; run `init` first so the prompts finish before the server starts.
- **`config.env` and `config.prod.env` are gitignored.** Fresh clones of the new project need `clj -M:run init` (or `dev`) to generate them.
- **Generated files live under `target/` and `storage/`.** CSS builds to `target/resources/public/css/main.css` and the SQLite database lives at `storage/sqlite/main.db`; both are gitignored.
