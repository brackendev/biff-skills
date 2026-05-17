---
name: biff-deploy
description: Deploy a Biff app to an Ubuntu VPS, or build a Docker / uberjar artifact
argument-hint: "[vps|docker|uberjar]"
user-invocable: true
disable-model-invocation: true
---

# Deploy a Biff App

Biff supports three deployment paths: a managed Ubuntu VPS (the default), a Docker image, or a standalone uberjar. Run this skill from inside a Biff project.

## Steps

Parse `$ARGUMENTS`. If empty, default to `vps`.

| Argument  | Path |
|-----------|------|
| `vps`     | Ubuntu VPS via `server-setup.sh` and `clj -M:dev deploy` |
| `docker`  | Build a Docker image with the starter `Dockerfile` |
| `uberjar` | Build a standalone uberjar with `clj -T:build uber` |

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
