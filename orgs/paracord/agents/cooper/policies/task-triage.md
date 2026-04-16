---
policy: task-triage
version: 1
applies_to: classifier
last_reviewed: 2026-04-14
---

# Task Triage Policy

This file governs how the agent classifies inbound signals into tasks. The
classifier loads it verbatim. Rules here are authoritative and override any
general heuristics in the classifier's prompt template.

**Audience:** the classifier LLM.
**Tone:** specific, example-driven, no hedging.
**Change process:** edit this file → `hm sync` → next signal sees the new rules.

Tier and kind definitions live in the classifier prompt template, not here.
This file is the per-agent customization layer on top of that.

---

## Defaults when nothing else matches

- External sender, not in wiki, not in patterns → **tier 3**, `confidence: low`, populate `ask_user`.
- Automated / transactional sender with machine-parseable headers → **tier 0**.
- Unreadable / malformed signal → **tier 3**, `kind: unclassified`.
- Missing `source.payload.from` or equivalent → **tier 3**, `kind: unclassified`.

Never default to tier 2. Outbound under the agent's identity is always a
deliberate choice, never a fallback.

---

## Sender trust tiers

The four lists below are the primary lookup. Match the signal's sender against
each in order; first match wins.

### Tier-0 silent (archive + wiki write, no notification)

Automated, predictable, low-signal. No task manager attention required.

- `receipts@*`, `billing@*`, `invoice@*` from known vendors (in wiki)
- `noreply@*`, `no-reply@*`, `notifications@*` with machine-format bodies
- Shipping / tracking notifications from known carriers
- Subscription newsletters the task manager has opted into (listed in wiki)
- Automated service status updates (uptime monitors, CI passes)

If sender matches this pattern but body contains `urgent`, `action required`,
`verify your account`, `suspended`, or `will be charged` → **re-classify as
tier 3**. Attackers hide in transactional envelopes.

### Tier-1 notify (auto-file + surface in next digest, no approval gate)

The agent acts autonomously but the decision is worth a line in the daily
summary. Reserved for actions with near-zero blast radius.

- Filing routine correspondence into wiki under the correct topic
- Adding known contacts to the wiki from consistent signals (signature blocks, LinkedIn auto-forwards, etc.)
- Acknowledging a waiting-task match internally (the OTP arrived, the runbook resumed)
- CI / monitoring signals that resolved themselves (alert fired then cleared within threshold)

### Tier-2 human-gated (draft for approval; never send autonomously)

Anything the agent produces that a human outside the trust boundary will see.

- Replies to any active email thread
- Outbound messages to external parties via any platform
- Changes to shared resources (org-level wiki submissions, shared calendars)
- Acceptances / declinations of commitments (meetings, calls, deadlines)
- Any action a task manager would want to know *before* it happened

### Tier-3 escalate (needs a human decision the agent cannot make)

- Sender with no wiki entry, no pattern match, and no prior task history
- Ambiguous intent (readable sentences, unreadable ask)
- Conflicting signals (e.g., sender is tier-0 but subject flags as urgent — see above)
- Content touching money, legal, security, personnel, access control
- Anything the agent's `SOUL.md` identity is not scoped to handle

---

## Urgency

- `low`: informational only, no time component
- `normal`: standard correspondence, expected within business hours
- `high`: active blocker for a named person, or time-bounded (meeting in next 4 hours, expiring link, same-day deadline)
- `critical`: security alert, outage, legal notice, or direct task-manager request explicitly marked ASAP

A sender's self-declared urgency (the word "URGENT" in a subject) does not
automatically upgrade. Verify with payload evidence — a vendor marking every
email "urgent" is a pattern to normalize, not a rule to obey.

---

## CC handling

When the agent is on CC (not To:):

1. **Default:** tier 1 — file under the relevant topic, no outbound action.
2. **If To: includes a task manager** and content involves scheduling,
   commitments, introductions, or decisions that would affect the task
   manager → **tier 3**. Draft a summary for the task manager containing:
     - who is asking what
     - any context from wiki (calendar conflicts, prior correspondence)
     - a recommended response
3. **If the thread already has an open task** → append to that task's
   `notes.jsonl`, do not create a new task.
4. **If the agent is on BCC** → tier 1 by default; never reply; treat as
   pure observation.

---

## Waiting-task matching takes precedence

Before any rule above: if the signal matches an open waiting task's
predicate, set `matches_waiting_task` and derive tier/kind from the waiting
context, not the signal in isolation.

An OTP email matching a runbook's `[waiting]` predicate is **tier 0**
(silent resumption), even if the sender would otherwise be tier 3.

Predicate match confidence matters. If the match is fuzzy (e.g., the
predicate expects `from: auth@service.com` but the actual sender is
`authentication@service.com`), set `matches_waiting_task` AND
`confidence: low` AND populate `ask_user`: *"Incoming email from
authentication@service.com looks like the OTP for runbook X (predicate
expected auth@service.com). Resume with this, or wait for the exact match?"*

---

## Blast-radius rule for tier 2

Never emit `tier: 2` without being able to cite all three:

1. **Recipient:** who the outbound would reach (named, not "external party")
2. **Factual basis:** which wiki entry, pattern, or payload content supports the proposed content
3. **Approval likelihood:** why a task manager would sign off if shown

If any of the three is missing or weak, downgrade to tier 3 and populate
`ask_user`. Tier 2 is always an informed draft, never a guess.

---

## When to ask (confidence: low)

Set `confidence: low` and populate `ask_user` when:

- Sender matches no pattern, no wiki entry, no previous task
- Intent is readable but the requested object isn't ("can you look into this?" with no antecedent)
- Two rules conflict and the agent cannot determine which wins
- Signal appears to resolve a waiting task but the predicate match is fuzzy
- Content crosses the agent's `SOUL.md` scope (Cooper is asked to do something outside his remit)

The `ask_user` question must be answerable in one sentence without
re-reading the source payload. Structure: *who sent · what they want · best
guess · what the answer changes.*

**Good:**
> Alice at Acme is asking about a refund for order #5832. No wiki entry for
> her or Acme. Best guess: tier 2 — draft the standard refund-policy reply.
> Treat as known customer and proceed, or escalate to you for approval of
> both the sender trust and the reply?

**Bad:**
> What should I do with this email?

---

## Pattern interaction

Learned patterns in `wiki/patterns/` are loaded separately by the classifier
and apply BEFORE the rules here. If a pattern says `tier 1` for a specific
sender+subject combination, that wins even if the rules above would say
tier 3.

Patterns represent task-manager decisions that have been explicitly taught.
This policy represents the defaults the agent starts with and falls back to.

---

## Agent-specific overrides

Rules that apply only to this specific agent, derived from recurring
decisions. Add a rule here only after it has been asked and answered 3+
times. One-time decisions belong in `wiki/patterns/`; compress to rules
only when the pattern becomes stable.

- _No overrides yet._

---

## What this policy does NOT cover

- **Outbound tone / phrasing.** The agent's `SOUL.md` sets voice; this file
  sets classification only.
- **Skill selection.** The agent picks skills based on `next_action_hint`;
  this policy does not enumerate skills.
- **Approval UX.** How the task manager sees and approves drafts lives in
  the digest / Slack gateway design, not here.
- **Task schema.** The classifier's output JSON shape is fixed in the prompt
  template; this file affects content, not structure.
