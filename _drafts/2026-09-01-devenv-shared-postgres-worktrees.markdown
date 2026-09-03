---
tags: [devenv, nix, postgresql, git, worktree, developer-experience]
author: Nicolas Mugnier
categories: tooling
title: "Sharing a PostgreSQL Cluster Across Git Worktrees with devenv"
description: "Git worktrees give you one directory per branch, but devenv gives each of them its own empty database. Here is how to make them share a single cluster, and what it costs."
image: /assets/img/devenv-shared-postgres.webp
locale: en_US
---

I have been using `git worktree` heavily for a while now: one directory per ticket, which means no more `git stash` juggling and several branches open side by side. On my machine that adds up to five worktrees plus the main checkout.

The development environment is managed by [devenv](https://devenv.sh), which spins up PostgreSQL, Redis, Caddy and PHP-FPM from a single `devenv.nix` file. That combination works beautifully, right up until you run your first command in a freshly created worktree:

```
$ make reload_db

SQLSTATE[08006] [7] connection to server at "localhost" (127.0.0.1), port 5432 failed:
Connection refused
```

Here is what causes it, the three ways out, and the one I settled on.

---

## The problem

devenv derives the PostgreSQL data directory from `DEVENV_STATE`, which is itself computed from the current directory:

```
PGDATA = ${DEVENV_STATE}/postgres = <worktree>/.devenv/state/postgres
```

Every worktree therefore gets **its own cluster**, empty on creation. You have to run `devenv up` there once, which triggers a full `initdb` followed by a fixtures import. On my project that is roughly 10 minutes. Multiply by the number of worktrees and it gets old fast.

What makes it particularly annoying is where the time actually goes. The fixtures import itself takes 7 seconds. The other 9 minutes and change are `initdb` plus framework cache warmup, pure setup cost, paid again and again for a result that is identical every time.

There is a second constraint that shapes every possible solution. The PostgreSQL port is hardcoded:

```nix
services.postgres = {
  enable = true;
  package = pkgs.postgresql_15;
  listen_addresses = "127.0.0.1";
  # no port setting, so devenv's default: 5432
};
```

and `devenv.yaml` declares `strictPorts: true`. As a result, **only one stack can run at a time on the machine**, whichever worktree it was started from. This is not something to work around, it is a fact to design with.

---

## What is already shared, and what is not

This distinction is the key to the whole thing, so it is worth being precise about it.

Your `.env` contains something like `APP_DATABASE_HOST=localhost` and `APP_DATABASE_PORT=5432`, identical in every checkout. So **data access goes through the port, not through the directory**. Any command, a fixtures reload, a test suite, a console command, run from any worktree, talks to whichever postmaster owns 5432.

What is *not* shared is `PGDATA`. It only decides which cluster the postmaster opens at startup. That single variable is the entire problem.

---

## Three options

### Option A: only ever start the stack from one checkout

Zero configuration. You run `devenv up` from the main checkout only, and in worktrees you never start the stack, you just run CLI commands.

This works as is, today, for free. The one limitation: your web server and application server serve the code of the checkout that started the stack. So the site you browse is the main checkout's code, not the worktree's. For CLI work and test suites, it makes no difference at all.

If your workflow is mostly "write code, run tests, occasionally check the app", this is genuinely enough. Do not over-engineer past it.

### Option B: share the `PGDATA`

Force `PGDATA` to a fixed path outside the worktrees. Every checkout then opens the same cluster.

This is possible because the `setup-postgres` script devenv generates reads `$PGDATA` from the environment, with nothing hardcoded, and crucially **skips `initdb` when the directory already exists**:

```bash
if [[ ! -d "$PGDATA" ]]; then
  initdb --locale=C --encoding=UTF8
  ...
else
  echo "PostgreSQL database directory appears to contain a database; Skipping initialization"
fi
```

One `initdb` for the whole machine, one fixtures import, shared by every worktree.

### Option C: option B, plus one database per worktree

Same shared cluster, but a distinct database name per worktree through your `APP_DATABASE_NAME` environment variable. One `initdb` for the machine, but one fixtures import per worktree.

This is the option that fixes B's real weakness, which I will get to.

---

## What I actually did: option B

### The override file

The goal was to leave `devenv.nix` untouched, since it is version controlled and shared with the rest of the team. devenv has exactly the right escape hatch for this: `devenv.local.nix`, merged automatically with `devenv.nix` through the Nix module system. It is conventionally gitignored, and devenv watches for it even when it does not exist yet (it is listed in `.devenv/input-paths.txt`).

The file, at the root of each checkout:

```nix
{ lib, ... }:

{
  env.PGDATA = lib.mkForce "${builtins.getEnv "HOME"}/.local/state/myapp-devenv/postgres";
}
```

Two details matter here:

- `lib.mkForce` is required. It gives the definition priority 50 against 100 for devenv's own postgres module. Without it the module system reports a conflict instead of picking a winner.
- `builtins.getEnv "HOME"` works because devenv evaluates impurely. It avoids hardcoding an absolute path, which means the exact same file can be copied to every checkout without editing.

Check that it evaluates before going further:

```bash
$ devenv print-dev-env | grep '^PGDATA='
PGDATA='/Users/you/.local/state/myapp-devenv/postgres'
```

### Migrating an existing cluster

If you already have a provisioned cluster somewhere, move it rather than rebuilding it:

```bash
devenv down
mkdir -p ~/.local/state/myapp-devenv
mv .devenv/state/postgres ~/.local/state/myapp-devenv/postgres
devenv up -d
```

The `devenv down` first is not optional. It guarantees a clean shutdown with no leftover `postmaster.pid`. On restart, PostgreSQL comes back up on the existing cluster with no `initdb`.

Verify from the server itself rather than trusting the config:

```sql
SHOW data_directory;
--  /Users/you/.local/state/myapp-devenv/postgres
```

Then copy the file everywhere else:

```bash
for wt in ~/Projects/worktrees/*/; do cp devenv.local.nix "$wt"; done
cp devenv.local.nix ~/Projects/myapp/
```

The now orphaned clusters, one per checkout at `<checkout>/.devenv/state/postgres` and roughly 100 MB each, can be deleted once you are confident.

### Is this risky?

No, and it is worth being explicit about why, because "several processes, one data directory" sounds alarming.

Two postmasters cannot open the same `PGDATA` at the same time: the second one fails on the `postmaster.pid` lock. And with `strictPorts: true`, a second `devenv up` while a stack is already running fails cleanly on the port. There is no silent corruption scenario, only clear error messages.

### Two traps I hit along the way

**A bootstrap process that checks the wrong database.** My project has a `database-bootstrap` step that counts tables in the *development* database to decide whether to load fixtures. I had only provisioned the *test* database, so it re-ran a full import on every single `devenv up`. Populating the dev database once fixed it permanently. If you have a similar guard, check which database it actually inspects.

**A stale environment variable in the shell.** `devenv up` inherits your shell environment, so the bootstrap runs with whatever `APP_ENV` happens to be exported. A leftover `APP_ENV=test` from an earlier command meant the test database got provisioned and the dev one stayed empty, which is exactly what triggered the trap above. When in doubt, start clean:

```bash
env -u APP_ENV devenv up -d
```

---

## The catch, and when to move to option C

A shared cluster means **a shared schema**. As long as your worktrees stay close to the main branch, this is a non issue. The moment two branches carry diverging migrations, they step on each other and you are reloading fixtures on every context switch, which cancels out a good part of the benefit.

Option C removes that limitation. Keep the shared server, isolate the data by database name. In each worktree's local environment file:

```
APP_DATABASE_NAME=myapp_feature_a
```

Most frameworks already read the database name from an environment variable, so this usually requires no code change at all.

The tradeoff:

| | Option B | Option C |
|---|---|---|
| `initdb` | one per machine | one per machine |
| Fixtures import | one, shared | one per worktree |
| Diverging migrations | conflict, reload on switch | isolated |
| Setup | one file to copy | one file plus one line per worktree |
| Disk usage | one cluster | one cluster, N databases |

My take after living with it: B is plenty as long as you work on short lived branches cut from the main branch. Move to C the day you catch yourself reloading fixtures several times a day.

---

## Summary

1. Create `devenv.local.nix` at the root of each checkout with the `PGDATA` override.
2. Move an existing cluster to the shared path, or let devenv create one on the first `up`.
3. Populate the database your bootstrap logic inspects, so it stops reloading fixtures on every start.
4. Only start the stack from one checkout at a time, the one whose code you want served over HTTP.

Because the file is not version controlled, this stays a personal choice. Teammates who do not use worktrees have nothing to do, and `devenv.nix` is never touched.

The broader lesson, and the reason I wrote this down: when a tool computes a path from your working directory, moving to worktrees quietly multiplies your state. Databases are the visible case because they fail loudly with a connection error. Caches, generated certificates and package stores do the same thing more discreetly, and they are worth auditing the same way.

---

## References

- [devenv documentation](https://devenv.sh/)
- [devenv: PostgreSQL service](https://devenv.sh/supported-services/postgres/)
- [git worktree documentation](https://git-scm.com/docs/git-worktree)
- [NixOS module system: option definitions and priorities](https://nixos.org/manual/nixos/stable/#sec-option-definitions)
