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

4. **Load per-target config so branches coexist.** Source a per-branch env file (`.env.<branch>`, falling
   back to `.env`) before running, and keep deploy config (ports, database URLs, secrets) out of the
   repo (gitignored). Distinct port + datastore per branch is what lets `dev` and `main` run
   side-by-side on one host without colliding. Fail loudly if a required var (e.g. the database URL) is
   absent — don't boot half-configured.

5. **Fail-fast, run last, hand off signals.** `set -euo pipefail`; each phase logs a clear banner; the
   run phase comes last and `exec`s the process so signals (SIGTERM from a service manager, Ctrl-C)
   reach the app directly. Offer a build-only mode (`--no-run`) for the case where a process manager,
   not the script, owns the running process.

The intent is a deploy that is **reproducible and self-consistent**: the same command deploys any
branch, from a clean ref, always serving exactly the code on that branch — in any stack the product
is built on.
