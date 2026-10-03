---
name: tasqr
description: Use when the Tasqr MCP tools (create_tasks, list_tasks, claim_next_task and the rest) are available and the work should outlive this session — substantial engineering efforts (migrations, feature builds, audits, large refactors), anything a human or teammate will review or pick up later, task queues agents claim from, subagent fan-out, and personal or professional task tracking the user asks you to record (including non-code work). Also use at session start to resume unfinished tasks, and when a tasqr tool errors or behaves unexpectedly. Invoke it while planning, not after — do not wait until a tasqr tool call is imminent. Skip one-off lookups, single-answer questions, and short in-session work that an ephemeral todo list already covers.
---

# Using Tasqr

Tasqr is a durable, shared record of work that outlives any one session. Two patterns, which can both appear in one session:

- **Producer** — you're doing the work and tracking it: create and close tasks as you go.
- **Consumer** — you're pulling work from a queue: claim, execute, close.

The tool docstrings cover parameters and response shapes. This skill covers judgment, server rules the docstrings don't state, and mistakes models make.

## When to create a task

**Tasqr is the durable record above your in-session todo list, not a replacement for it.** A todo list is scratch: ephemeral, private, free. A Tasqr task persists, is visible to the user and other agents, and counts against quota.

Create a task when any of these hold:

- **Substantial effort** — a migration, feature build, audit, or large refactor; work measured in sessions, not minutes.
- **It outlives this session** — someone (maybe another agent) resumes it later.
- **A human will see it** — a teammate reviews it, picks it up, or tracks it on a board.
- **The user asks you to track something.** Not just code: "follow up with the vendor Friday" or "track the SOC 2 checklist" are Tasqr tasks. Personal and professional tracking is a first-class use.
- **Queue or fan-out work** — tasks other agents claim, or child tasks for subagents you spawn.

**Don't** for quick lookups, single-answer questions, or a short sequence you'll finish this session. A list cluttered with trivia is worse than none, and it burns quota. If it's genuinely ambiguous, ask: "Want me to track this in Tasqr?"

---

## Session start: resume before you create

Check for work already assigned to you first, or you'll create duplicates:

```python
me = get_profile()["email"]
list_tasks(status="in_progress", assignee=me)   # resume these
list_tasks(status="pending", assignee=me)       # start these before new work
```

On a paid plan `get_standup()` is cheap opening context (`references/intelligence.md`). Before creating a task on an unfamiliar area, `search_tasks` may show prior work to reuse.

---

## Rules the server enforces

**Assignee is required to leave a holding state.** `pending` and `blocked` are holding states; moving to *any* other status (`in_progress`, `paused`, `completed`, `failed`, `canceled`) needs an assignee — set at create time or in the same update item. `claim_next_task` assigns for you. It must be a **real member email**, not a label like `"agent-1"`. Yours is `get_profile()["email"]`; others are in `list_members()`.

**A `note` is required to close** (`completed`, `failed`, `canceled`), per item. Write it for the next reader: `completed` — what was done and key decisions; `failed` — what failed, what was tried, what the next agent needs; `canceled` — why. Put results in `output`, which must be a JSON object (`{"summary": ...}`), not a string; `metadata` too.

**No self-loops.** Re-sending the current status is an error; to change fields on an `in_progress` task, omit `status` (it also writes no history event).

| Status | Valid next states |
|---|---|
| `pending` | `in_progress`, `blocked`, `canceled` |
| `in_progress` | `blocked`, `paused`, `completed`, `failed`, `canceled` |
| `blocked` | `in_progress`, `completed`, `failed`, `canceled` |
| `paused` | `in_progress`, `completed`, `failed`, `canceled` |
| `completed` / `failed` / `canceled` | none (terminal) |
| `feedback` | Used internally by Tasqr; it won't appear in your tasks |

Every move from `pending`/`blocked` to a non-holding status, `canceled` included, needs an assignee; `pending` → `blocked` doesn't.

**`paused` vs `blocked`.** `paused` = waiting on the user (put the question in `note`). `blocked` = waiting on something outside your control (CI, a deploy, another agent) instead of polling.

**Blocking behavior.**
- Adding an open `blocked_by` entry moves a `pending`/`in_progress` task to `blocked` by itself. Blocking on an already-finished task is rejected; max 90 blockers per task.
- Explicit `status: "blocked"` with no `blocked_by` is allowed (for waits outside Tasqr) but **nothing auto-unblocks it**. When the wait ends, move it yourself straight to `in_progress` (with an assignee): `blocked` → `pending` is not valid.
- Auto-unblock, when all blockers are `completed`, returns the task to `pending` — even one that was `in_progress` when blocked — so move it to `in_progress` again to resume.
- If a blocker `failed`, the dependent auto-fails; if `canceled`, it auto-cancels. Propagation is asynchronous: wait a few seconds before reading the dependent back.

**Tags are replaced, not merged** on update: pass the full list, or `[]` to clear.

---

## Producer — tracking your own work

Create the task, move it to `in_progress` with your assignee, close it with a `note`. Put what the executor needs in `description` and `metadata`; use `priority` (1=critical, 5=low, default 3).

**Plans go in one `create_tasks` call, never a loop.** One round trip, one write against rate limits, and a failed batch creates **nothing**, leaving no half-built plan. Wire dependencies with `ref` / `"ref:<name>"` in `blocked_by` (mixing real UUIDs is fine). **`ref:` resolves only in `blocked_by`, not in `parent_task_id`** — `parent_task_id` takes a real task id, so create the parent first and reuse its id. Same for closing: put every transition in one `update_tasks` call. The server topologically sorts a batch and rejects cycles.

**Quota.** A batch of N counts N and is rejected whole if it would breach the limit. Check `get_quota()` before a large batch; if `remaining` is 0, stop and tell the user rather than retrying.

**Create results** may carry `similar` (check it before piling on — the work may already be tracked) and `suggested_tags` (check `strict` before applying one).

### Tags

Tags must exist in the org vocabulary (the check is skipped if the org has zero tags), so `list_tags()` first. Three tools return a plain list — `list_tags`, `list_members`, `list_teams` — wrapped as `{"result": [...]}`; read `response["result"]`. Every other tool returns its own named keys.

**Read a tag's `description` before choosing it** (`None` only on non-strict tags). It matters most for **strict** tags: a strict tag decides *who can ever claim the task*, so a wrong pick strands it in that team's queue. A wrong non-strict pick is recoverable.

If the tag you want doesn't exist, prefer a close existing one; vocabularies are capped per tier (free 15 → enterprise 1000). Tag writes need `admin`/`owner` (`references/org-admin.md`). If rejected for role, use the nearest tag or none, say an org admin can add it, and move on — **never block the user's work on a missing tag.**

### Reading back

- `get_tasks` with **exactly one** id returns `history`; several ids return summaries without it. A missing id lands in `not_found`, not an error.
- `list_tasks` items are lean; use `full=True` or `get_tasks` for description/metadata/output. A `cursor` only works with the exact parameters that produced it; on `Invalid pagination cursor`, restart from page 1. Unfiltered lists put open work first, so page 1 is the queue.
- `search_tasks` returns `{"available": false}` if the org is client-encrypted or search isn't configured; fall back to `list_tasks` filters.

---

## Consumer — working a task queue

`claim_next_task()` is atomic (no two agents get the same task) and assigns it to your key's identity. `claimed: false` means the queue is empty for your filter.

**Read `context` before starting** — it usually answers "what came before this?": `parents` (chain up from the immediate parent), `blockers` (terminal blockers' `output`), `prior_art` (similar completed tasks, paid tiers, best-effort), `runbook` (how this org does X, pro/enterprise, best-effort). Skip it with `include_context=False` only if you already hold it.

**Leases.** An expired lease with no writes returns the task to `pending` for someone else. Every `update_tasks` call renews it, so posting progress in `metadata` is enough. Size `lease_seconds` to the expected work at claim time.

### Filtering claims by tag

Your **effective tags** are `profile_tags ∪ team_tags` (`get_profile()` returns both).

- With effective tags set and no `tags` argument, they become the filter; an explicit `tags=[]` means no filter.
- **Matching is all-of.** A broad tag set can match nothing against a full queue. If a claim comes back empty against a queue you can see, re-claim with the one or two tags that describe the work, or `tags=[]`.
- **Strict tags.** A task carrying a strict tag can only be claimed by agents whose effective tags include it, even with an explicit filter. A non-admin's `update_profile` can neither add nor drop a strict tag: strict entries are **silently removed** from what you send, existing ones are kept, and the call still succeeds. Read the returned `profile_tags` for what was stored, and ask an admin for strict-tag changes.

---

## Multi-agent coordination

When spawning subagents, model each as a child task (`parent_task_id`) so the tree is visible and each updates its own status. Create the parent first, then the children with its real id. Give each child everything it needs in `description` + `metadata`; a subagent may not share your context. Poll `list_tasks(parent_task_id=parent)` until all children are terminal.

**Subagents on your key share your email**, so `assignee` cannot tell them apart. Have each pass a distinct `agent_id` (say `"auth-auditor"`) on its `update_tasks` items; it is recorded on every state event in `get_tasks` history. `agent_id` is who made the change; `assignee` is who owns the task. Pre-assign children to your own email (`get_profile()["email"]`) so they can move to `in_progress`.

---

## Talking to the human about tasks

Task ids are for tool calls, not conversation. Identify a task to the user by what it *is* — its title or a short summary. An id may ride along in parentheses next to the title, never in place of it. This applies everywhere you address a human: status updates, `paused` questions, close-out summaries, lists of open work.

- Good: "The database upgrade task (efab5678) is blocked by the Java version upgrade task (cdef9012)."
- Bad: "Task efab5678 is blocked by task cdef9012."

---

## Reporting problems with Tasqr itself

`submit_feedback` tells the Tasqr team something is wrong. Send it **before** working around: a tool that errors unexpectedly or contradicts this skill, a workflow the tools don't support, or anything here that's confusing, wrong, or missing. Write it as a bug report (what you tried, expected, got). The first line becomes the title; keep it short. It goes to the Tasqr team, **not** your org — problems with your own tasks belong in `update_tasks` notes.

Submissions pass a content filter: write it the way you'd write to a real person. Abusive text is rejected as a tool error, and repeated rejections revoke the member's keys. If the filter is unavailable that's also a tool error and nothing is written; retry shortly.

---

## More tools

Batch-taking tools (`create_tasks`, `update_tasks`, `get_tasks`, `create_tags`, `update_tags`, `delete_tags`) cap at **25 items** per call. `update_profile(profile_tags=[...])` sets your default claim filter — worth doing once if you work a queue.

Two more surfaces, each in its own reference — **read the file before calling those tools**, don't guess the semantics:

- **`references/org-admin.md`** — tag writes (`create_tags`, `update_tags`, `delete_tags`), roles and seats (`update_member`), and teams (`list_teams`, `get_team`, `create_team`, `update_team`, `delete_team`, `add_team_member`, `remove_team_member`). Nearly all require `admin`/`owner`.
- **`references/intelligence.md`** — `get_insights` (flow health), `get_standup` (fleet report), `list_runbooks` (learned how-tos), `plan_tasks` (draft a grounded task graph). All tier-gated; each degrades to a clear "unavailable" response rather than failing.

`get_org_dek` / `put_org_dek` are client-encryption plumbing handled by the MCP client — never call them yourself.
