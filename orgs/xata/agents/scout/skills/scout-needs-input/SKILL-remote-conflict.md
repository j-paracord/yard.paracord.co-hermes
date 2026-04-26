---
name: scout-needs-input
description: Ask the operator a question mid-run without blocking forever. XADD a `scout.needs_input` event to the notify stream (surfaces in the website run pane and Slack), XREAD the reply on the per-run input stream with a 30-minute timeout. On timeout, emit `run.paused_awaiting_operator` and exit cleanly — the run is resumable.
---

# scout-needs-input

Use this when you need a human decision *during* a run that you can't reasonably guess at. The mechanism is two Redis streams — the operator sees your question on whichever surface they're already on (website pane or Slack), answers on either, and you unblock.

**Do not use this skill** for anything you could answer by reading the working copy, the payload, or the `reference/` docs. Every question costs operator attention; only ask when the answer is genuinely external.

## When to use

- Ambiguous amount or identifier: "Which HSBC account — checking or savings?", "Confirm the payroll total is $42,318.22 before I post to QBO."
- Missing input the trigger payload didn't include: "The invoice PDF wasn't in the share drive — paste a link."
- Irreversible or high-blast-radius action: "Should I actually approve these 14 Airbase transactions now or stage them for review?"
- Divergence from script: "The runbook says pull from BambooHR but the new source of truth is Deel — should I switch?"

Do **not** use this skill for:
- Asking the operator to debug your code. Figure it out, or fail with `run.failed` and a clean stderr.
- "FYI" messages. Use a `scout.observation` notify event instead — it doesn't block you waiting for a reply.
- Confirmation of every step. Operators ignored frequent pings; batch decisions into one question when possible.

## Stream contract

**Ask — publish to notify stream:**

- Key: `xata:scout:notify:runbook-events`
- Event: `scout.needs_input`
- Fields:
  - `run_id` — the active run
  - `question_id` — a short unique id you generate (e.g. `q-<short-uuid>`); used to correlate the reply
  - `prompt` — the human-readable question
  - `input_type` — one of `text`, `number`, `currency`, `choice`, `url`, `confirm`
  - `choices` — JSON array, required when `input_type=choice` (e.g. `["checking","savings"]`)
  - `allowed_responders` — JSON array of operator emails, or `["any"]` if anyone on the run can answer
  - `context` — optional one-paragraph explainer shown under the prompt

**Wait — subscribe to the per-run input stream:**

- Key: `xata:scout:input:<run_id>`
- Fields on reply: `question_id`, `answer`, `responder`, `_ts`
- You filter by `question_id` — if the operator replied to an earlier question on the same run, ignore it and keep blocking.

## Flow

### Step 1 — Publish the question

Prefer the `yard stream` wrapper (exists on the droplet, handles Upstash auth from `REDIS_URL`):

```bash
QID="q-$(openssl rand -hex 4)"
yard stream publish runbook-events \
  --event scout.needs_input \
  --run-id "$RUN_ID" \
  --data "$(jq -cn \
    --arg qid "$QID" \
    --arg prompt "Which HSBC account did this transfer come from?" \
    --arg type "choice" \
    --argjson choices '["checking","savings"]' \
    --argjson who '["yannis@xatabase.ai"]' \
    '{question_id:$qid, prompt:$prompt, input_type:$type, choices:$choices, allowed_responders:$who}')"
```

Record `QID` — you need it to match the reply.

### Step 2 — Block on the reply stream

```bash
REPLY=$(yard stream read-one "input:$RUN_ID" \
  --filter "question_id=$QID" \
  --timeout 1800)
```

`yard stream read-one` does an XREAD with `BLOCK <ms>` internally and returns the first message whose fields match the filter. Timeout is in seconds — 1800s = 30 min is the default ceiling; pick lower for time-critical decisions (e.g. 600s = 10 min for "is this close to the deadline?").

### Step 3a — Reply arrived

```bash
ANSWER=$(echo "$REPLY" | jq -r '.answer')
```

Validate against `input_type`:
- `number` / `currency` → parse as float; reject NaN
- `choice` → confirm it's in the `choices` array
- `confirm` → expect literal `"yes"` or `"no"`
- `url` → must start with `https://`

On a malformed reply, emit `scout.observation` with `reason:"invalid reply to <QID>"` and re-ask once (new `question_id`). If the second reply is also malformed, escalate with `run.failed`.

### Step 3b — Timed out

The operator walked away. Emit:

```bash
yard stream publish runbook-events \
  --event run.paused_awaiting_operator \
  --run-id "$RUN_ID" \
  --data "$(jq -cn --arg qid "$QID" \
    '{question_id:$qid, pending_since:now|todate, resumable:true}')"
```

Then exit cleanly — do **not** guess at the answer. The heal-loop's resume path (future work: `xata:scout:trigger:resume-runbook`) will pick the run back up when the operator finally answers.

## Recipes

**Yes/no confirmation before an irreversible action:**
```bash
QID="q-$(openssl rand -hex 4)"
yard stream publish runbook-events \
  --event scout.needs_input \
  --run-id "$RUN_ID" \
  --data "$(jq -cn --arg qid "$QID" \
    '{question_id:$qid,
      prompt:"About to post a $42,318.22 journal entry to QBO. Proceed?",
      input_type:"confirm",
      allowed_responders:["yannis@xatabase.ai"]}')"

REPLY=$(yard stream read-one "input:$RUN_ID" --filter "question_id=$QID" --timeout 600)
case "$(echo "$REPLY" | jq -r '.answer')" in
  yes) ;; # continue
  no)  echo "operator declined"; exit 0 ;;
  *)   echo "bad reply"; exit 1 ;;
esac
```

**Ask for a missing link:**
```bash
yard stream publish runbook-events \
  --event scout.needs_input \
  --run-id "$RUN_ID" \
  --data "$(jq -cn --arg qid "$QID" \
    '{question_id:$qid,
      prompt:"Paste the Google Drive link to this month'\''s German payroll PDFs.",
      input_type:"url",
      allowed_responders:["any"]}')"
```

**Batch decision to avoid three pings:**
```bash
# Instead of three separate confirms, ask once with a combined payload.
yard stream publish runbook-events \
  --event scout.needs_input \
  --run-id "$RUN_ID" \
  --data "$(jq -cn --arg qid "$QID" \
    '{question_id:$qid,
      prompt:"Review before I post: Payroll $42,318 / PTO 128.5 hrs accrued / Expense sync 412 txns. Proceed with all?",
      input_type:"confirm",
      context:"If any line is wrong, reply no and I will pause for manual review.",
      allowed_responders:["yannis@xatabase.ai"]}')"
```

## Failure modes

- **`yard stream publish` returns a non-2xx** — Upstash may be rate-limited or `REDIS_URL` is stale. Retry once with exponential backoff; on second failure, emit `run.failed` with `reason:"notify-publish-failed"`. Do not proceed without publishing — the operator has no surface for your question.
- **Reply arrives for an older `question_id`** — ignore it, keep blocking. `yard stream read-one --filter` should already discard it, but verify the filter is set.
- **Responder not in `allowed_responders`** — the relay layer (website / Slack bot) enforces this before XADDing, so if you see an unauthorized reply, that's a relay bug. Emit `scout.observation` with `reason:"unexpected_responder"` and treat the run as compromised — exit with `run.failed`.
- **Two replies to the same `question_id`** — take the first; `read-one` returns on first match. Don't process duplicates.

## Guardrails

- One open question per run at a time. If you're already blocking on a reply and need another input, wait for the first to resolve — nested Q&A confuses operators.
- Never log the answer if it contains sensitive data (amounts, account numbers, secrets). The notify stream is readable by anyone with Upstash creds; assume your question is semi-public and size the prompt accordingly.
- Timeouts are features, not failures. A 30-min silence isn't a bug — the operator is at lunch. Exit cleanly and let the run resume later.
