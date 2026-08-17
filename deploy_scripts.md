# Design Patterns — Deploy Scripts

A running record of deployment-script decisions for Rodeo-Org products. Specific rulings are captured
verbatim; where a ruling encodes a reusable principle, the general form is extracted so it can be
applied elsewhere. (This is the library's first *operational* pattern — the repo otherwise holds
UI/UX conventions; the ruling below deliberately extends its scope.)

## Branch-parameterized pull → build → run

**Ruling (Rodeo-Org, 2026-08-17, tony):** There is one deploy script, usable for every target. It takes
a `--branch <name>` flag — **`main` is the default** when the flag is omitted, and **`--branch dev`**
also works. The script **pulls, builds, and runs** the project. (Concrete instance: rodeo-forum
`ops/deploy.sh`.)

**Generalized principle (project- and stack-agnostic):**

A deploy script for a git-backed app should have five properties, in order:

1. **One script, branch-parameterized.** A single `--branch <name>` flag selects the target, defaulting
   to the **stable** branch (here `main`). The same script deploys the stable line and the integration
   line (`dev`) — no forked, drifting per-environment scripts. New long-lived branches cost nothing.

2. **Pull the exact ref, refuse on a dirty tree.** `git fetch` → `git checkout <branch>` →
   `git pull --ff-only`. Abort if the working tree has uncommitted changes rather than clobber them.
   Deploy from a known, clean commit — never from whatever happened to be on disk.

3. **⚠️ Always rebuild artifacts from source — never serve a prebuilt/cached build output.** This is the
   load-bearing rule. Build outputs (`dist/`, bundles, compiled assets) are routinely **gitignored**, so
   a branch checkout does **not** refresh them; a server pointed at a stale prebuilt bundle serves the
   *old* code regardless of the checked-out branch, and it reads as a code regression that costs real
   debugging time. Rebuilding on every deploy makes "what is served" ≡ "what is on the branch,"
   structurally — the checkout and the served artifact can never disagree. (Motivating failure cataloged
   in `pantheon-config/bugs/gitignored-dist-served-stale-looks-like-code-regression-checkout-does-not-refresh.md`.)

4. **Keep per-target runtime config outside the repo, so branches coexist.** Each target's runtime
   config — port, database URL, secrets — lives with *whatever owns the running process*, never in the
   repo (gitignored env file, or the service unit's own `EnvironmentFile`). Distinct port + datastore
   per branch is what lets `dev` and `main` run side-by-side on one host without colliding. The deploy
   script itself needs none of that runtime config for the pull+build phases; it only needs it if the
   script also *runs* the process directly (see 5).

5. **Fail-fast; the run phase comes last and hands off to whatever supervises the process.** `set -euo
   pipefail`; each phase logs a clear banner. How the "run" phase is spelled depends on who owns the
   long-lived process:
   - **A service manager owns it** (systemd/supervisor — the common production case): the run phase is
     `systemctl restart <service>` (service name derived from the branch, e.g. `rodeo-<branch>`,
     overridable). The unit owns env, port, and restart-on-failure; the script's job is just
     *pull → build → restart*. This is rodeo-forum's model (`rodeo-dev` on its droplet).
   - **The script owns it** (foreground / dev box): the run phase loads the per-target env (4) and
     `exec`s the process, so signals reach the app directly rather than the shell wrapper.
   Offer a build-only mode (`--no-run` / `--no-restart`) for when you want to stage a build without
   bouncing the running process.

The intent is a deploy that is **reproducible and self-consistent**: the same command deploys any
branch, from a clean ref, always serving exactly the code on that branch — in any stack the product
is built on.
