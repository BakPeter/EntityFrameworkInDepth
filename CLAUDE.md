# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Current state

A greenfield .NET / Entity Framework learning project. There is **no solution file, no `.csproj`, no source
code, no tests, and no CI** yet — only the local Postgres development stack under `Infrastructure/`. The one
tracked commit holds `README.md` (a title line) and the standard GitHub `VisualStudio.gitignore`; everything
else is still untracked.

So: there are no build, restore, or test commands to run. Once projects exist, read the `.sln`/`.csproj` rather
than assuming a layout, and replace the stale parts of this file (see *Maintaining this file*).

The `VisualStudio.gitignore` plus `Infrastructure/` implies the intended shape — .NET via `dotnet` CLI or Visual
Studio on Windows, with persistence concerns separated into their own layer — but the layering has not been
decided. Confirm it before adding projects instead of inferring it from the one folder name.

## Local database stack

`Infrastructure/docker-compose.yml` runs Postgres 17 and pgAdmin 8.14, both on a named bridge network
(`efindepth-net`). pgAdmin waits on Postgres's `pg_isready` healthcheck via `depends_on: service_healthy`.

Persistence has **two modes, chosen by whether `.env` exists** — the compose file writes the mount source as
`${POSTGRES_DATA_PATH:-postgres-data}`, and Compose reads a `../`-prefixed value as a bind mount but a bare
word as a named volume:

| `.env` present | Mount | Where data lives |
| --- | --- | --- |
| yes (normal) | bind | `Mounts/postgres-data`, `Mounts/pgadmin-data` at the repo root |
| no | named volume | Docker-managed `efindepth-postgres-data`, `efindepth-pgadmin-data` |

The two are **separate datasets**: losing `.env` points the stack at an empty Docker volume and looks exactly
like data loss, though nothing is actually gone. The `Mounts/` paths live only in the env files
(`POSTGRES_DATA_PATH` / `PGADMIN_DATA_PATH`, relative to `Infrastructure/`, hence `../`); the compose file holds
only the named-volume fallbacks.

In bind mode the data is browsable from Windows and dies with the directory — `docker compose down -v` does not
clear it; deleting the directory contents does. `Mounts/.gitignore` ignores everything but itself, so the
directory survives a clone while its contents are never committed. pgAdmin runs as `user: root` because it
chowns `/var/lib/pgadmin` at startup, which fails on a bind mount otherwise.

Pass both the compose file and the env file explicitly, so these run from the repo root:

All commands follow one template — `docker compose --env-file <env> -f <compose file> <subcommand>`:

```powershell
docker compose --env-file Infrastructure/.env -f Infrastructure/docker-compose.yml up -d    # start
docker compose --env-file Infrastructure/.env -f Infrastructure/docker-compose.yml down     # stop (data survives)
docker compose --env-file Infrastructure/.env -f Infrastructure/docker-compose.yml config   # show resolved values
docker compose --env-file Infrastructure/.env -f Infrastructure/docker-compose.yml logs -f postgres
```

Both are global flags, so they go before the subcommand. `-f` sets the project directory to the compose file's
own directory — so `../Mounts/...` resolves to the repo root and the project name stays `infrastructure`, making
these identical to running bare from inside `Infrastructure/`. `--env-file` is resolved against the current
directory instead, hence the `Infrastructure/` prefix on it too. It is redundant while the file is named `.env`
(Compose auto-loads that), but stating it keeps the data location explicit rather than dependent on a filename;
without it, a missing `.env` silently switches the stack to Docker-managed volumes, whereas naming the file
makes Compose fail loudly instead.

Env files: `.env.example` is the committed template; `.env` is the real, git-ignored one, auto-loaded by Compose
from `Infrastructure/`. Both carry the same 10 variables, and `docker-compose.yml` carries a fallback for every
one, so the stack also starts with no `.env` at all. Adding a variable means touching all three places.

**A fresh clone has no `.env`, and that silently changes where data goes** — see below. Run
`Copy-Item Infrastructure/.env.example Infrastructure/.env` first.

Connection details (defaults): database `efindepth`, user/password `postgres`/`postgres`. From the host, EF Core
connects on `localhost:5432`:

```
Host=localhost;Port=5432;Database=efindepth;Username=postgres;Password=postgres
```

From inside pgAdmin (`http://localhost:5050`, login `admin@local.dev` / `admin`) the server host is `postgres`
— the service name — **not** `localhost`, which there resolves to the pgAdmin container itself.

Two traps worth knowing:

- `POSTGRES_DB`/`USER`/`PASSWORD` are applied only when the data directory is first initialized. Changing them
  later does nothing until `Mounts/postgres-data` (repo root) is emptied.
- Repointing `POSTGRES_DATA_PATH`/`PGADMIN_DATA_PATH` does not migrate data; Docker creates the new directory
  empty (owned by root) and the old one is left behind.

`Infrastructure/.gitignore` ignores `.env` and `.env.*` while re-including `.env.example`. That negation is
load-bearing — drop it and the template stops being committed.

## Maintaining this file

When a solution and projects land, add the real restore/build/test commands (including how to run a single
test), the `dotnet ef` migration commands and which project hosts the `DbContext`, and a description of the
project layout and inter-layer dependencies. Delete the "no code yet" framing above at that point.