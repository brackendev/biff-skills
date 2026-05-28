# Skill Conventions

This document defines the argument grammar, scope vocabulary, and mutation defaults that every user-invocable skill in `biff-skills` follows. A reader who learns one skill should be able to predict every other skill. New skills follow this document.

These conventions apply to user-invocable skills (`user-invocable: true` in frontmatter). Model-invocable reference skills with no argument surface (currently `biff`) carry no argument grammar and are exempt from rules 1 and 2.

## The four rules

### Rule 1: argument grammar

User-invocable skills accept natural-language keywords and bare paths. The single sanctioned flag is `--report`. No other `--name` flags exist.

Skill-specific modifiers are bare phrases, not flags. Examples used in this plugin:

- `vps`, `docker`, `uberjar`: deployment-target keywords for `/biff-deploy`
- `<project-name>`: required positional argument for `/biff-new`
- `all`, `<path>`, `<glob>`: shared scope keywords listed in rule 2

Each skill documents its own modifiers in its `## Arguments` table.

Exemptions are listed in the [Exemptions](#exemptions) section with a reason. The standard exemption pattern is a scaffolding skill that needs a required positional argument because no useful default exists.

### Rule 2: scope vocabulary

Skills that operate on files, diffs, pull requests, or commit messages share these core rows in their `## Arguments` table:

| Input             | Target                                                          |
|-------------------|-----------------------------------------------------------------|
| (no argument)     | The skill's narrowest useful default                            |
| `all`             | Widen the selected scope to its maximum                         |
| `<path>` `<glob>` | Operate on those files or directories                           |

Each skill states what `all` resolves to in concrete terms (the default deployment path, the whole project, every detected step). The keyword has one uniform meaning: widen the selected scope to the maximum. The unit varies per skill.

A skill whose narrowest useful default is already maximum scope still accepts `all` for family consistency. The table row reads "Same as (no argument); accepted for family consistency."

Opt-in rows (`#N` or PR URL, `pr`, `commit`) appear only when the skill genuinely supports them. None of the current `biff-skills` skills operate on pull requests or commit messages, so the opt-in rows do not appear in this plugin today.

### Rule 3: mutation is the default

Skills that can mutate the workspace apply changes when invoked. The operator passes `--report` to receive a description of what the skill would do, without modifying any files.

Only the literal token `--report` enables report-only mode. Natural-language phrases ("preview", "dry run", "rehearse") are scope input or step keywords, not mode triggers. A skill that conflates them is wrong.

Command verbs reinforce the default. Skills named `/biff-new` and `/biff-deploy` (verbs that imply action) mutate by default. The plugin currently ships no pure-report user-invocable skill; a future review-style skill would carry a reading verb (`/biff-review-*`, `/biff-audit-*`, `/biff-check-*`) and omit `--report`.

### Rule 4: vendored and generated paths are excluded by default

A mutating skill that walks the workspace excludes vendored, generated, and dependency-locked paths from its scope. The operator opts back in per file by naming the path explicitly. No new `--name` flag is introduced; the override rides on Rule 1's `<path>` `<glob>` row.

The boundary statement: Rule 4 applies to mutating skills that discover candidate files from the workspace. It does not apply to skills whose target set is defined by an explicit project operation, template, dependency model, git operation, or named path argument.

**Exclusion set.** Two filters apply together. A path that matches either filter is excluded.

1. `.gitignore`-matched paths. Anything excluded by the project's `.gitignore`, `.git/info/exclude`, or the global excludes file is out of scope. Resolve membership with `git check-ignore -v -- <path>`.
2. Hardcoded floor (excluded even when the project tracks the path):

   | Category | Patterns |
   |----------|----------|
   | Dependency directories | `node_modules/`, `vendor/`, `third_party/`, `.bundle/` |
   | Build outputs | `target/`, `build/`, `dist/`, `out/`, `.shadow-cljs/`, `cljd-out/` |
   | Lock files | `*.lock`, `package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`, `Gemfile.lock`, `Cargo.lock`, `poetry.lock`, `composer.lock` |

**Override.** When the operator names a vendored or generated file in the arguments through the `<path>` `<glob>` row, the filter does not apply to that target. Naming the path directly is treated as informed consent. The filter remains active for broad scopes: `(no argument)`, `all`, or a directory whose contents include vendored sub-paths.

**Reporting.** The skill includes a single "skipped N vendored or generated paths" line in its results when the filter excluded any path. Under `--report`, the skill emits the full list so the operator can audit scope.

**Scope of Rule 4 in this plugin.** No skill in this plugin discovers candidate files from the workspace today. Both `/biff-new` (scaffolds a new project tree from the official Biff installer) and `/biff-deploy` (runs a deployment command and a `git ls-files` upload manifest) are exempt because their target sets are defined by explicit project operations, not workspace discovery. The rule is recorded so any future fix-style skill in this plugin honors the same contract as its companion packages.

## Classification

Every user-invocable skill falls into one of two classes.

**Mutating skill** -- default behavior. May carry `--report` when preview is useful. May omit `--report` when preview is meaningless (the operator reads `SKILL.md` to see the side effects) or the action is small and reversible.

**Pure report** -- never mutates. No `--report` flag because there is nothing to invert.

A skill is classified by its actual behavior, not by its name. If the name and behavior disagree, the rename is the fix. The classification appears in the skill's own description so the operator knows what to expect.

### Skills in this plugin

| Skill           | Class    | `--report` available? | Notes |
|-----------------|----------|------------------------|-------|
| `/biff-new`     | Mutating | No                     | Scaffolds the project tree via the official `clj -M -e '(load-string ...)'` installer. Preview the side effects by reading `SKILL.md`. |
| `/biff-deploy`  | Mutating | Yes                    | Default path is the managed Ubuntu VPS via `server-setup.sh` and `clj -M:dev deploy`. `--report` prints the deployment path, target host, and the command sequence (including the `git ls-files` upload manifest for the VPS path) without running anything. |

The model-invocable `biff` skill has no argument surface and is not classified here.

## Section structure

User-invocable skills with an argument surface use these section conventions:

- `## Arguments` -- required. Contains the canonical scope table.
- `## Scope` -- optional. Add only when detection order, fallback, or base-branch resolution exceeds what the table can express.
- `## Mutation` -- optional. Add only when mutation behavior needs clarification beyond a single table row (for example, when each command requires per-step operator confirmation in addition to the default mutate-on-invoke contract).

The section name `## Customization` is retired.

## Worked examples

### `/biff-deploy` -- mutating skill with `--report`

```
/biff-deploy                  # default vps path; rsync or git push, restart, tail logs
/biff-deploy docker           # build the starter Dockerfile image
/biff-deploy uberjar          # build the standalone uberjar via clj -T:build uber
/biff-deploy --report         # print the planned commands and upload manifest; no writes
/biff-deploy docker --report  # print the docker build command and image tag; no writes
/biff-deploy all              # synonym for (no argument)
```

The default deploys via the VPS path (`server-setup.sh` + `clj -M:dev deploy`). Each remote-touching command still asks the operator for confirmation, which is layered on top of the standard mutate-by-default contract. With `--report`, the skill prints the planned command sequence (including the resolved target host, the deploy command, the `git ls-files` upload manifest for the VPS path, and the docker image tag or uberjar output path) and writes nothing. The `## Mutation` section in the skill body documents the per-command confirmation behavior.

### `/biff-new` -- mutating skill, exemption from `all`/path rows

```
/biff-new my-app             # scaffolds the project tree at ./my-app
/biff-new                    # prompts the operator for a project name and namespace
```

`<project-name>` is a required positional argument. The skill omits the `all` and `<path>` rows because they would not be meaningful for scaffolding. It also omits `--report` because preview is meaningless: the operator reads the upstream `https://biffweb.com/new.clj` script (and this `SKILL.md`) to see what files will be written. The exemption is listed below.

## Exemptions

`/biff-new` accepts a required positional `<project-name>` because no useful default exists for scaffolding. It omits the `all` and `<path>` scope rows and the `--report` flag for the same reason. The `## Arguments` table in its `SKILL.md` documents the positional grammar.

## Ambiguity notes

**`(no argument)` outside a git worktree.** A skill whose narrowest useful default depends on git state (for example, "deploy the current commit") must define the fallback when no git worktree is present. `/biff-deploy` requires a Biff project on disk (it reads `resources/config.edn` and `config.env`) and runs `git ls-files` for the VPS upload manifest. If the working directory is not a Biff project or not a git worktree, the skill stops and reports the missing prerequisite rather than widening silently.

**`commit` versus staged-and-unstaged state.** When a future skill accepts `commit` as a scope keyword, it must state whether `commit` means the most recent commit, the staged tree, or the staged-plus-unstaged working tree. The expected default is the most recent commit. None of the current skills carry this scope.

**`--report` versus natural-language synonyms.** "Preview", "dry run", "rehearse", and similar phrases are scope input or step keywords (or operator chatter), never mode triggers. Only the literal `--report` token disables writes. `/biff-deploy` accepts the deployment-target keywords `vps`, `docker`, and `uberjar`; none of those are mode-toggle words.

**Step keywords versus scope keywords.** `/biff-deploy` accepts deployment-target keywords (`vps`, `docker`, `uberjar`) that select work to run, and the scope keyword `all` (a synonym for the default VPS path, accepted for family consistency). Target keywords are skill-specific and listed in the skill's own table. Scope keywords are shared and listed here. When a skill has both, the `## Arguments` table lists both with clearly distinct rows.

**Tool-level flags the skill calls internally.** A skill may invoke a tool that itself uses POSIX flags (for example, `clj -M:dev deploy`, `clj -T:build uber`, `docker build -t`, `curl -fsS`, `git ls-files`). Those are tool-level flags, not skill flags, and do not count against rule 1. The skill body should disambiguate when a tool flag could be mistaken for a skill flag.

## Author checklist

When adding or modifying a user-invocable skill, confirm each item before committing.

- [ ] Skill has a `## Arguments` section (or is listed under [Exemptions](#exemptions)).
- [ ] Scope rows match the canonical table; opt-in rows appear only where the skill genuinely supports them.
- [ ] If the skill mutates, the command verb signals it (`/biff-new`, `/biff-deploy`).
- [ ] If the skill mutates and preview is useful, `--report` is documented.
- [ ] If the skill is pure-report, the verb signals it (a future `/biff-review-*`, `/biff-audit-*`, or `/biff-check-*` skill) and the skill has no `--report` flag.
- [ ] No `## Customization` section.
- [ ] No `--name` flags other than `--report`. Tool-level flags the skill calls internally (for example, `clj -M:dev deploy`, `clj -T:build uber`, `docker build -t`) are not skill flags and do not count.
- [ ] Mutating skills that walk the workspace include a one-line Rule 4 reference in their `## Mutation` or `## Scope` section. Exempt skills (scaffolding, deployment, dependency upgrade, fixed targets, git operations, named paths) carry no reference.
- [ ] Frontmatter `name` matches the skill's directory name.
- [ ] OpenCode mirror under `.opencode/skills/<name>/SKILL.md` is byte-identical to the canonical source.
- [ ] `agents/openai.yaml` `default_prompt` references the current command name.
- [ ] `CHANGELOG.md` records the change under `[Unreleased]` when the change is user-facing.
