# Mission contracts and proof of work

This runbook describes how to run a fleet of coding/ops agents on Mission Control
so that **every mission carries an owner, a reviewer and observable proof**, and no
mission can be reported as finished on a status message alone.

It is written for operators who currently track agent work in scattered markdown
files, chat transcripts or ad-hoc report folders. That pattern has two failure
modes this platform is built to remove:

- **Competing sources of truth.** The same mission exists in several files, each
  written by a different agent, none of them authoritative.
- **Hollow status.** An agent reports "in progress" or "done" with nothing
  attached that a human can inspect.

Everything below uses primitives that already exist in this repository. No new
service is required.

## What the platform already provides

| Operating need | Where it lives | Notes |
| --- | --- | --- |
| Mission | `tasks` (`title`, `description`) | Board-scoped work item |
| Owner | `tasks.assigned_agent_id`, `tasks.created_by_user_id` | One accountable agent per mission |
| Lifecycle | `tasks.status`, `tasks.priority`, `tasks.in_progress_at` | Status vocabulary is fixed, see below |
| Reviewer gate | `approvals` (`action_type`, `confidence`, `rubric_scores`, `status`, `resolved_at`) | Sensitive actions are gated, with a rubric |
| Proof trail | `activity_events` (`event_type`, `message`, `task_id`, `agent_id`) | Append-only history for audit |
| Contract metadata | `task_custom_field_definitions` + `tasks.custom_field_values` | Typed, validated, optionally required |
| Workforce | `agents`, `gateways` | Identity and runtime location of each agent |
| Structure | `organizations`, `board_groups`, `boards` | One board per division or loop |
| Capability packs | `skills`, skills marketplace | Narrow skill sets instead of one giant agent |

## The status vocabulary is fixed

`backend/app/schemas/tasks.py` declares:

```python
TaskStatus = Literal["inbox", "in_progress", "review", "done"]
```

Four statuses, no more. Do not try to introduce `QUEUED`, `BLOCKED`, `FAILED` or
`VERIFIED` as statuses — the API will reject them. Map them instead:

| Operating state | How to represent it |
| --- | --- |
| Queued | `status = inbox` |
| Running | `status = in_progress` |
| Awaiting review | `status = review` + an open approval |
| Blocked | `status = in_progress` + `blocked_reason` custom field set |
| Failed | `status = in_progress` + `blocked_reason` set + a failure activity event |
| Verified | `status = done`, allowed only under the rule below |

This keeps one vocabulary for everyone. An agent that invents its own status
words is reporting into a channel nobody queries.

## The mission contract as custom fields

Create these once per organization. Field types are constrained by the database
to `text`, `text_long`, `integer`, `decimal`, `boolean`, `date`, `date_time`,
`url`, `json`; `ui_visibility` is one of `always`, `if_set`, `hidden`.

| `field_key` | `field_type` | `required` | Purpose |
| --- | --- | --- | --- |
| `why` | `text_long` | yes | Which business loop this mission improves |
| `source_of_truth` | `text_long` | yes | Paths and data the agent is allowed to trust |
| `files_owned` | `json` | yes | Exact paths this mission may write |
| `do_not_touch` | `json` | no | Paths that are off limits |
| `base_sha` | `text` | yes | Commit the work starts from |
| `acceptance` | `text_long` | yes | Observable result that ends the mission |
| `tests` | `text_long` | yes | Commands a reviewer can re-run |
| `proof_url` | `url` | no | Where the before/after evidence is published |
| `metric` | `text` | no | The number that should move |
| `reviewer` | `text` | yes | Who verifies; never the author |
| `blocked_reason` | `text_long` | no | Set whenever progress stops |

Create one definition:

```bash
curl -fsS -X POST "$API/api/v1/organizations/me/custom-fields" \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{
        "field_key": "base_sha",
        "label": "Base SHA",
        "field_type": "text",
        "required": true,
        "ui_visibility": "always",
        "validation_regex": "^[0-9a-f]{7,40}$",
        "description": "Commit the mission starts from"
      }'
```

`validation_regex` is the part that stops drift: a mission cannot be opened with
a base commit that is not a commit.

Open a mission with its contract attached:

```bash
curl -fsS -X POST "$API/api/v1/boards/$BOARD_ID/tasks" \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{
        "title": "Export web assets from the canonical source",
        "status": "inbox",
        "priority": "high",
        "assigned_agent_id": "'"$AGENT_ID"'",
        "custom_field_values": {
          "why": "Unblocks the first end-to-end client walkthrough",
          "source_of_truth": "the canonical asset file and its recorded checksum",
          "files_owned": ["export/web/v1/"],
          "do_not_touch": ["the canonical source file"],
          "base_sha": "0badc0f",
          "acceptance": "Two exported assets exist with recorded sizes and checksums",
          "tests": "re-run the export script; compare checksums",
          "reviewer": "reviewing-agent"
        }
      }'
```

## The reviewer gate

Use `approvals` for anything you would not let an agent do unattended. An
approval carries `action_type`, a free-form `payload`, a `confidence` score and
`rubric_scores`, which is what lets you run tiered autonomy instead of approving
every single call:

| Tier | Example actions | Handling |
| --- | --- | --- |
| Green | read-only inspection, reports, local builds | No approval; the activity trail is enough |
| Orange | writes inside `files_owned`, restartable service changes | Approval auto-resolved above a confidence threshold, always logged and reversible |
| Red | production deploys, deletions, secrets, money, permissions | Approval held for a human, always |

An agent must never resolve its own approval. `approvals.agent_id` records who
requested; the resolver is recorded separately when the approval is resolved.

```bash
curl -fsS "$API/api/v1/boards/$BOARD_ID/approvals?status=pending" \
  -H "Authorization: Bearer $TOKEN"
```

## The proof rule

One rule makes the whole thing work:

> A mission may only reach `done` when (a) at least one activity event carries
> the evidence, and (b) every approval it opened is resolved.

Evidence means something a reviewer can re-check without trusting the author:
a checksum, a byte size, a test summary, a URL, a before/after pair. "I verified
it" is not evidence. Neither is a promise to verify later.

Read the trail for a mission:

```bash
curl -fsS "$API/api/v1/activity?task_id=$TASK_ID" \
  -H "Authorization: Bearer $TOKEN"
```

Because `activity_events` is append-only, the trail cannot be quietly rewritten
after the fact — which is exactly the property you want when several agents work
on the same board.

## Operating notes

- **One board per loop, not per project.** A board whose name is the whole
  company cannot tell you what is blocked.
- **One canonical branch per deployed surface.** If what runs in production is
  built from a commit that is not an ancestor of your main branch, you have two
  truths; reconcile before adding features.
- **Narrow skill packs beat large ones.** Prefer several agents with a handful of
  skills each over one agent carrying every capability.
- **A report folder is not a control plane.** If the answer to "what is running
  right now" requires reading files by hand, it belongs in `tasks` instead.
