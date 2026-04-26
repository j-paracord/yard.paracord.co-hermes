---
name: heal-pr
description: End-to-end heal-loop playbook. Invoked when a `heal-runbook` trigger arrives carrying a failed runbook and stderr. Finish the operator's intent first, then open a PR against `kb/runbooks/` so the next automated run is green. Uses `yard git open-pr`; requires the scout-git skill.
---

# heal-pr

You are scout. A runbook just failed on the xatabase website and the operator clicked **"Scout: finish the job."** The heal trigger landed with a full failure context. Your job has two parts, in this order:

1. **Complete the operator's intent.** The monthly close still needs to happen; that is the deliverable.
2. **Open a PR** against `kb/runbooks/` with a fix so next month's automated run succeeds. The PR is a byproduct, not the goal.

Read the **scout-git** skill before starting — every git operation here goes through `yard git`.

## When this skill fires

A `xata:scout:trigger:heal-runbook` message arrives. The consumer unpacks it into a prompt for you carrying:

- `run_id` — the original failed run; doubles as your heal-run checkpoint id
- `runbook` — slug (e.g. `monthly/run-payroll`)
- `failing_step` — `{block_index, content}` — the exact `wb` block that died
- `stderr` — last lines of captured error output
- `sha` — the runbook commit SHA that ran (pin this in the PR body)
- `payload` — the original user inputs (form fields, prereq confirmations)
- `triggered_by` — operator email

If any of those fields is missing, emit `run.failed` on notify with `reason: "heal context incomplete"` and stop. Don't guess.

## Flow

### Step 1 — Sync the working copy

```bash
yard git pull --path /home/hm-scout/repos/xatabase-finance-assistant
```

Record the HEAD SHA after pull — you'll need it in the PR body and in progress events.

### Step 2 — Read the runbook

```bash
cat /home/hm-scout/repos/xatabase-finance-assistant/kb/runbooks/<runbook>.md
```

Locate the failing block by index (markdown code fence #N, 0-indexed from the top of the file). Read the blocks *around* it — context matters. Reference docs may live in `reference/` of the same repo; grep for terms in the failing step.

### Step 3 — Reproduce locally

Reproducing before fixing is non-negotiable. Run the failing block's command in a scratch dir with the same env the `wb` run had:

```bash
# payload ships as JSON; unpack into env the same way `wb -e` would have
mkdir -p /tmp/heal-<run_id>
cd /tmp/heal-<run_id>
# export TRIGGER_PAYLOAD and per-field env — match the failing run exactly
```

If reproduction succeeds on retry (transient network, rate limit, etc.), that's your signal this is a flake, not a code bug. Skip to Step 5 (complete the job) and Step 6's PR becomes a retry-wrapper addition, not a logic fix.

If reproduction fails with the same stderr, you have a real bug. Continue.

### Step 4 — Diagnose and fix the *underlying* issue

Read the error; identify the root cause. **Do not** patch around it with a try/except that swallows errors. The PR's point is to make the automated path work next time — masking the failure defeats that.

Typical root causes in finance runbooks:
- Env var missing → add a prereq check at the top of the runbook, or declare it in the runbook's frontmatter
- Tool output format changed (Deel, Gusto, Orb, QBO CSV headers drift) → update the parser
- Auth expired → *not* a code fix; flag for operator and skip PR
- Race between upstream and downstream systems → add a wait/poll with bounded retries

### Step 5 — Complete the operator's intent

This is where you diverge from the script if needed. The script is *how* the job was being done, not *what* was being done. If the script broke halfway, figure out what remains, do it, and report back.

Emit progress as you go so the operator's run pane doesn't look frozen:

```bash
yard stream publish runbook-events \
  --event step.complete \
  --run-id "$RUN_ID" \
  --data '{"step":"heal-resume","note":"pulling PTO balances directly from BambooHR API"}'
```

When the job is actually done, emit:

```bash
yard stream publish runbook-events \
  --event run.complete \
  --run-id "$RUN_ID" \
  --data '{"completed_by":"scout","heal":true,"sha":"<head-sha-after-pull>"}'
```

### Step 6 — Open the fix PR

Only after the job is complete and `run.complete` has been emitted. Create a branch, commit the fix, open the PR:

```bash
BRANCH="scout/heal-<runbook-slug>-<short-run-id>"
cd /home/hm-scout/repos/xatabase-finance-assistant
git checkout -b "$BRANCH"
git add kb/runbooks/<runbook>.md   # or whichever files you touched
git -c user.name="scout[bot]" \
    -c user.email="<app-id>+scout[bot]@users.noreply.github.com" \
    commit -m "fix(runbook): <one-line summary>"
yard git open-pr \
  --runbook <runbook-slug> \
  --branch "$BRANCH" \
  --title "Heal: <runbook-slug> — <one-line summary>" \
  --body-file /tmp/heal-<run_id>-pr.md \
  --label scout/heal
```

The PR body **must** follow this template (write it to `/tmp/heal-<run_id>-pr.md` before calling open-pr):

```markdown
## Heal context

- **Run ID:** <run_id>
- **Runbook:** <slug> (sha `<short-sha-that-ran>`)
- **Failing step:** block #<N>
- **Operator:** <triggered_by>

## What failed

```
<stderr — last ~20 lines, fenced>
```

## What scout did

<transcript: commands run, decisions made, anything that diverged from the
script. Keep this factual — the developer reviewing needs to trust what
actually happened on the droplet.>

## Diff rationale

<why this change fixes the underlying issue, not just the symptom. Cite
the docs/APIs you consulted if the fix depends on them.>
```

Without this body, the developer reviewing the PR has no context. Make the body complete on the first push — don't rely on follow-up comments.

### Step 7 — Reset the working copy

```bash
git -C /home/hm-scout/repos/xatabase-finance-assistant checkout main
git -C /home/hm-scout/repos/xatabase-finance-assistant branch -D "$BRANCH"
```

The working copy is shared state for future triggers; leaving it on a branch strands them. Back to `main`, clean, then exit.

## Guardrails

- **Order matters: complete the job, *then* open the PR.** If the job can't be completed (blocked on creds, upstream API down), skip the PR. A PR without a completed job is noise.
- **No PR for flakes.** If reproduction in Step 3 succeeded on retry, don't open a PR labeled as a fix — there's nothing to fix. Emit `run.complete` with `heal:"flake-resolved"` and stop.
- **Never commit secrets.** Anything from Doppler or `.env` stays out of the diff. If the fix genuinely needs a new env var, declare it in the runbook's frontmatter and document it in the PR body — the operator will add it via Doppler.
- **One fix per PR.** If you find three bugs while diagnosing one, pick the one that caused the failure, open that PR, and surface the other two via notify (`scout.observation` event) for the developer to triage separately. Bundled PRs don't get merged.
- **If you need operator input mid-heal** (e.g. "which HSBC account?", "confirm this amount"), use the **scout-needs-input** skill. Don't guess at payload values.
- **Self-heal recursion is off.** If your heal run itself fails, emit `run.failed` with `reason:"heal-failed"` and stop. Do *not* trigger another heal on yourself.

## Failure modes

- **`yard git open-pr` returns 422 "No commits between main and <branch>"** — you made no changes. Either this was a flake (stop, no PR) or your `git add` missed the file. Re-check the diff.
- **`yard git open-pr` returns 403** — App token couldn't write. Re-mint (`yard git mint-token`) and retry once; if still failing, escalate via Slack (scout-needs-input) — the App installation may have been revoked.
- **`git push` succeeds but the PR doesn't appear** — branch protection or App-permission mismatch. The push-only-no-PR state is survivable; re-run `yard git open-pr` against the existing branch.
- **Reproduction in Step 3 hangs** — timebox to 2 minutes. Beyond that, kill it, emit `run.failed` with `reason:"heal-repro-timeout"`, and leave the runbook untouched.

## Exit checklist

Before this skill considers itself done, all of these must be true:

- [ ] `run.complete` or `run.failed` emitted on notify with the right `run_id`
- [ ] Working copy is back on `main` and clean (`git status` inside it shows nothing)
- [ ] If a PR was opened, the URL was surfaced in a `scout.observation` notify event so the operator can find it
- [ ] No temp files left in `/tmp/heal-<run_id>*` (best-effort cleanup)
