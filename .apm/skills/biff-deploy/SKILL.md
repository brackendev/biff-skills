---
name: biff-deploy
description: "Deploy a Biff app to an Ubuntu VPS, or build a Docker / uberjar artifact"
argument-hint: "[vps|docker|uberjar] [--report]"
user-invocable: true
disable-model-invocation: true
---

# Deploy a Biff App

Biff supports three deployment paths: a managed Ubuntu VPS (the default), a Docker image, or a standalone uberjar. Run this skill from inside a Biff project.

## Arguments

| Input             | Target                                                                                  |
|-------------------|-----------------------------------------------------------------------------------------|
| (no argument)     | VPS path: `server-setup.sh` (first-time provisioning) and `clj -M:dev deploy`           |
| `vps`             | Same as (no argument); explicit selection of the VPS path                               |
| `docker`          | Build a Docker image from the starter `Dockerfile` with `clj -T:build uber` and `docker build` |
| `uberjar`         | Build a standalone uberjar with `clj -T:build uber`                                     |
| `all`             | Same as (no argument); accepted for family consistency                                  |
| `--report`        | Print the planned commands (resolved target host for the VPS path, the `git ls-files` upload manifest, the Docker image tag, or the uberjar output path). Write nothing, run nothing. |

This skill operates on deployment-target keywords rather than file paths, so the `<path>` row of the standard scope vocabulary does not apply. The `all` keyword is accepted as a synonym for the default VPS path.

## Mutation

Mutates by default: deploys to a running production system (VPS path) or writes a Docker image / uberjar artifact (`docker` and `uberjar` paths). The VPS path also asks the operator for confirmation before each remote-touching command, which is layered on top of the standard mutate-on-invoke contract; treat any refusal at that prompt as a stop. With `--report`, the skill prints the planned command sequence for the selected path and writes nothing.

## Steps

Parse `$ARGUMENTS`. Extract `--report` if present (report-only mode). Use the remaining token as the deployment target. If the remaining token is empty or `all`, default to `vps`.

| Argument  | Path |
|-----------|------|
| `vps`     | Ubuntu VPS via `server-setup.sh` and `clj -M:dev deploy` |
| `docker`  | Build a Docker image with the starter `Dockerfile` |
| `uberjar` | Build a standalone uberjar with `clj -T:build uber` |

Under `--report`, do not run any of the commands listed in the path sections below. Instead, resolve `DOMAIN` from `config.env`, print the deployment path, the target host (VPS), the command sequence, and the file list that would be uploaded (`git ls-files` plus any `:biff.tasks/deploy-untracked-files` overrides for the VPS path); print the resolved `docker build` tag and `Dockerfile` location for the Docker path; print the uberjar output path under `target/` for the uberjar path.

### VPS path (default)

#### Confirm before acting

Every command in this path either touches a remote host or modifies production. Show the user each command and the target host, and wait for confirmation before running it. Treat refusal as a stop.

#### First-time provisioning

1. Confirm `DOMAIN` is set in `config.env` (for example `DOMAIN=example.com`). If `config.env` does not exist, generate it:

   ```bash
   clj -M:dev generate-config
   ```

2. Confirm an A record on the domain points to the target VPS, and that `ssh root@<domain>` succeeds.
3. Confirm the VPS has at least 1 GB of memory.
4. Provision:

   ```bash
   scp server-setup.sh root@<domain>:
   ssh root@<domain> "bash server-setup.sh"
   ssh root@<domain> "reboot"
   ```

5. If `rsync` is not installed locally, add the remote git remote:

   ```bash
   git remote add prod ssh://app@<domain>/home/app/repo.git
   ```

   If the project's default branch is `main`, edit `:biff.tasks/deploy-cmd` in `resources/config.edn`.

#### Deploy

```bash
clj -M:dev deploy
clj -M:dev logs
```

`deploy` uploads (rsync if available, `git push` otherwise) and restarts the app. `logs` tails the server log so the deploy can be verified.

Only files that `git ls-files` reports are uploaded, plus `config.env` and `target/resources/public/css/main.css`. Add overrides under `:biff.tasks/deploy-untracked-files` in `resources/config.edn`.

### Docker path

```bash
clj -T:build uber
docker build -t <image>:<tag> .
```

The starter project ships a `Dockerfile`. Push the image to a registry, then run it with the same environment variables that `config.env` defines.

### Uberjar path

```bash
clj -T:build uber
```

Run with:

```bash
java -jar target/<app>-standalone.jar
```

The runtime reads `config.env` from the working directory.

## Post-deploy verification

```bash
clj -M:dev logs                       # VPS path: tail server log
curl -fsS https://<domain>/ -o /dev/null
```

Report a successful deploy only after the smoke check passes.

## Gotchas

- **Never run `server-setup.sh` against a live server without explicit confirmation from the user.** It is intended for a fresh Ubuntu VPS.
- **Update `server-setup.sh` whenever a server package or configuration changes.** The script is the only path to reprovisioning a server from scratch.
- **Standalone XTDB topology has no built-in backups and cannot run on more than one server.** For production data, point XTDB at a managed Postgres via `config.env`.
- **`clj -M:dev prod-dev` evaluates local changes against the running production system.** Use it deliberately; it is not a sandbox.
- **`clj -M:dev deploy` ignores untracked files.** Anything not in `git ls-files` (other than `config.env` and the generated Tailwind CSS) will not appear on the server unless listed under `:biff.tasks/deploy-untracked-files`.
