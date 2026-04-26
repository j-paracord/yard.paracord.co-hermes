---
name: scout-git
description: Use `yard git` — not raw git or gh — for every operation on the xatabase-finance-assistant working copy. Handles App-token auth, pulls, and path conventions. Required reading before any repo-touching flow (heal-pr, runbook edits, diagnostics).
---

# scout-git

All git activity against the runbook source repo flows through a single primitive: the `yard git` subcommand family. You shell out to it with the `terminal` tool — the binary lives on `$PATH` on this droplet.

`yard git` owns GitHub App token minting, auth-header injection, and the working-copy path. **Don't** use raw `git` for fetches/pulls on this repo (auth will fail) and **don't** use `gh auth` (this profile has no user token — only an App installation).

## When to use

- Any step in the **heal-pr** flow (reading / modifying / PR-ing the runbook source).
- Any runbook that needs to read `reference/*.md`, `kb/runbooks/*.md`, or repo-level config to answer a user question.
- Diagnostics: "is the working copy fresh?", "what SHA am I on?", "is the token valid?".
- Before running a workbook that references files in the repo by absolute path.

Do **not** use this skill for:
- Pushing changes to repos *other than* `paracord-clients/xatabase-finance-assistant`. The App is scoped to that one repo; anything else will 404.
- Reading a single file you could retrieve inline from the trigger payload (`workbook_md` ships with every runbook trigger). Don't re-fetch what you already have.

## Path convention — always absolute

The working copy is at:

```
/home/hm-scout/repos/xatabase-finance-assistant/
```

**Always use the absolute path. Never `~/repos/…`.**

Reason: when hermes-agent spawns a subprocess via the `terminal` tool, it overrides `$HOME` to `/home/hm-scout/.hermes/profiles/scout/home/`. So `~` inside a shell you launch resolves to the profile's sandboxed home, not your user home. A symlink bridges the two (`.../profiles/scout/home/repos` → `/home/hm-scout/repos`), but relying on it is fragile. Absolute paths are canonical.

## Commands

### `yard git pull` — force-sync to origin/main

```bash
yard git pull --path /home/hm-scout/repos/xatabase-finance-assistant
```

Runs `git fetch` + `git reset --hard origin/main` with a freshly-minted App token. Reports the before/after SHAs and whether anything changed. Idempotent — safe to run whenever freshness matters.

Flags:
- `--branch <name>` — default `main`. Only change this for a branch-specific flow (you almost never do).
- `--remote <name>` — default `origin`.

### `yard git mint-token` — get an installation access token

```bash
yard git mint-token
```

Prints a GitHub installation token to stdout. Tokens auto-expire after 1 hour — mint fresh rather than caching.

Use when you need to call the GitHub REST API directly (e.g. list PRs, read an issue comment) for something `yard git` doesn't wrap yet:

```bash
TOKEN=$(yard git mint-token)
curl -sH "Authorization: Bearer $TOKEN" \
  https://api.github.com/repos/paracord-clients/xatabase-finance-assistant/pulls?state=open
```

### `yard git status` — working-copy health check

```bash
yard git status --path /home/hm-scout/repos/xatabase-finance-assistant
```

Shows branch, HEAD SHA, clean/dirty state, and whether the App credentials can mint a token right now. First command to run when anything git-related feels off.

### `yard git open-pr` — land a heal or fix

Covered by the **heal-pr** skill. Not part of general scout-git use; don't invoke it outside that flow.

## Recipes

**Read a runbook from the working copy** (for context, not execution):
```bash
cat /home/hm-scout/repos/xatabase-finance-assistant/kb/runbooks/monthly/run-payroll.md
```

**Confirm you're on the same SHA the trigger payload pinned:**
```bash
# payload_sha is in the trigger fields (TRIGGER_PAYLOAD env or --checkpoint id)
# HEAD should match after a successful `yard git pull`
git -C /home/hm-scout/repos/xatabase-finance-assistant rev-parse HEAD
```
(Plain `git` is fine for *local* read-only ops like `rev-parse`, `log`, `diff`, `grep`, `show`. The App token is only required for network ops.)

**Search the repo for a reference the runbook text doesn't include:**
```bash
grep -rn "HSBC" /home/hm-scout/repos/xatabase-finance-assistant/reference/
```

**Re-sync after a known-good PR was merged upstream:**
```bash
yard git pull --path /home/hm-scout/repos/xatabase-finance-assistant
```

## Failure modes

- **`not a git repo: /home/hm-scout/repos/...`** — the clone hasn't been bootstrapped on this droplet yet. Stop and surface this via `scout.needs_input` or a `run.failed` event; don't attempt to `git clone` yourself (the App token flow isn't wired for clone).
- **`remote: Bad credentials` / 401** — token mint failed. Re-run `yard git mint-token` to confirm. If it still fails, `GITHUB_APP_PRIVATE_KEY` in the env is stale or malformed. Escalate; don't patch around it.
- **`Your branch is ahead of 'origin/main' by N commits`** — someone committed locally on the droplet (shouldn't happen). Surface it; don't reset without a decision trail.
- **Post-pull SHA unchanged when you expected a change** — the PR you're waiting on may not be merged to `main` yet, or Cloudflare/raw.githubusercontent caching is surfacing old content. Check `gh pr view <n>` via a minted token.

## Guardrails

- Never `git push` directly to `main`. Branch protection will reject it; even if it didn't, the App is scoped R/W but we land changes only through PRs.
- Never commit secrets or Doppler-pulled values into the working copy. Runbook fixes are code-only.
- Never `git clean -fd` or `git reset --hard` to a non-origin ref. The working copy is shared state; destroying it strands future triggers until someone reclones.
- If you're unsure whether to run a destructive git op, **stop** and ask via `scout-needs-input`.
