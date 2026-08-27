---
name: tasqr
description: Use when tasqr MCP tools (mcp__tasqr__*) are available and the work should outlive this session — substantial engineering efforts (migrations, feature builds, audits, large refactors), anything a human or teammate will review or pick up later, task queues agents claim from, subagent fan-out, and personal or professional task tracking the user asks you to record (including non-code work). Also use at session start to resume unfinished tasks, and when a tasqr tool errors or behaves unexpectedly. Invoke it while planning, not after — do not wait until a tasqr tool call is imminent. Skip one-off lookups, single-answer questions, and short in-session work that an ephemeral todo list already covers.
---

# Using Tasqr

Tasqr is task state management built for AI agents: a durable, shared record of work that outlives any one session. When Tasqr tools are available, use them to make your work visible, trackable, and resumable across sessions and agents.

Two patterns, which can both appear in one session:

- **Producer** — you're doing the work and tracking it. You create and close tasks as you go.
- **Consumer** — you're pulling work from a queue. Tasks already exist; you claim and execute them.

## When to create a task

**Tasqr is the durable record above your in-session todo list, not a replacement for it.** A todo list is scratch: ephemeral, private to this session, free. A Tasqr task persists, is visible to the user and to other agents, and counts against quota. Reach for Tasqr when the work deserves to outlive the conversation.

Create a task when any of these hold:

- **Substantial effort** — a migration, feature build, audit, or large refactor. Work measured in sessions, not minutes.
- **It outlives this session** — someone resumes it later, maybe a different agent or person.
- **A human will see it** — a teammate reviews it, picks it up, or tracks it on a board.
- **The user asks you to track something.** This need not be code. "Follow up with the vendor Friday," "track the SOC 2 checklist," "add this to my backlog" are all Tasqr tasks — personal and professional tracking is a first-class use, not an edge case.
- **Queue or fan-out work** — tasks other agents claim, or child tasks for subagents you spawn.

**Don't create a task** for quick lookups, single-answer questions, or a short sequence you'll finish in this session — that's the ephemeral todo list's job. A task list cluttered with trivia is worse than no task list, and it burns the user's quota.

When it's genuinely ambiguous, ask the user rather than guessing. "Want me to track this in Tasqr?" costs one sentence.

---

## Producer — tracking your own work

### Lifecycle

```python
t = create_tasks(tasks=[{"title": "...", "description": "..."}])["results"][0]   # starts as "pending"
update_tasks(updates=[{"task_id": t["task_id"], "status": "in_progress", "assignee": "you@example.com"}])
# ... do the work ...
update_tasks(updates=[{"task_id": t["task_id"], "status": "completed",
                        "note": "what was accomplished", "output": {...}}])
```

**An assignee is required to leave a holding state.** `pending` and `blocked` are holding states; moving to *any* other status (`in_progress`, `paused`, `completed`, `failed`, `cancelled`) needs one. Pass `assignee` at create time or in the same `update_tasks` item that moves the task — `claim_next_task` auto-assigns for you.

It must be a **real member email**, not a label like `"agent-1"`; anything else is rejected. Yours is `get_profile()["email"]`, everyone else's is in `list_members()`.

**A `note` is required** when closing as `completed`, `failed`, or `cancelled`. The server rejects the update without one.

- `completed` — what was accomplished, key decisions made
- `failed` — what failed, what was tried, what the next agent needs to know
- `cancelled` — why it was abandoned

### Create a whole plan in one call

**`create_tasks` always takes a list** — a single task is a list of one. Creating 2+ related tasks? One call, never a loop: one round trip, one write against rate limits, and the batch is validated upfront, so a failed batch creates **nothing** and leaves no half-built plan to clean up.

Items depend on each other via `ref`, so you never need UUIDs. Give a blocker item a `ref`, then reference it as `"ref:<name>"` in another item's `blocked_by`. Real UUIDs and refs mix freely:

```python
result = create_tasks(tasks=[
    {"ref": "schema", "title": "Update DB schema", "description": "Add columns and indexes"},
    {"ref": "api", "title": "Update API endpoints", "description": "Use the new schema",
     "blocked_by": ["ref:schema"]},
    {"title": "Update docs", "description": "Document the new endpoints",
     "blocked_by": ["ref:api", "9f6f7f1a-existing-uuid"]},
])
# result = {"created": 3, "results": [{"ref", "task_id", "status", "created_at"}, ...]}
refs = {r["ref"]: r["task_id"] for r in result["results"] if r.get("ref")}
```

Up to 25 tasks per call; each `ref` unique; no cycles (the server topologically sorts and rejects them). When blockers reach terminal states, dependents follow automatically: all blockers `completed` → the dependent returns to `pending`; a blocker `failed` → the dependent auto-fails; a blocker `cancelled` → it auto-cancels. Propagation runs asynchronously — give it a few seconds before reading the dependent back.

**Check quota before a large batch.** A batch of N tasks counts N against quota and is rejected upfront if it would breach the limit — the whole batch fails, not just the overflow. `get_quota()` returns `{"tier", "limit", "used", "remaining", "resets_at", "requests": {...}}` (`requests` is the org-pooled monthly API-call allowance, separate from the task quota; a `null` limit means unmetered). If `remaining` is 0, stop and tell the user their quota is exhausted rather than retrying.

For hierarchy, pass `parent_task_id`. Use `priority` (1=critical, 5=low, default 3) and `metadata` for structured context the executing agent will need.

### Two advisory hints on create results

Each item in `results` may carry extra keys. Both are best-effort — **an absent or empty value means no signal, not an error** — and neither changes anything on its own:

- `suggested_tags` — `[{"name", "score"}]`, most relevant first, matched against the org's described tags. To apply one, issue an `update_tasks` that sets the tags — but check whether it is `strict` first (a strict tag narrows who can claim the task).
- `similar` — `[{"task_id", "title", "status"}]`, open tasks that look like near-duplicates. Check them before piling on: the work may already be tracked.

### Tags are validated against the org vocabulary

You cannot invent tag strings — `create_tasks`/`update_tasks` reject any tag not in the org vocabulary. (The check is skipped entirely if the org has defined zero tags.) Tags are **replaced**, not merged: pass the full list, or `[]` to clear.

```python
list_tags()
# {"result": [
#   {"name": "bug", "strict": False, "default": False,
#    "description": "Something is broken — a defect in behaviour that already exists.",
#    "created_by": ..., "created_at": ...},
#   {"name": "network-team", "strict": True, "default": False,
#    "description": "Work owned by the Network Team — strict, only they can claim it."}, ...]}
```

The three tools that return a plain list — `list_tags`, `list_members`, `list_teams` — arrive
wrapped in a `result` key; read `response["result"]`, not the response itself. Every other tool
returns its own named keys (`{"tasks": …}`, `{"results": …}`, `{"runbooks": …}`).

**Read `description` before you pick a tag** — it says what the tag is for. It's `None` only on non-strict tags; a strict tag always has one.

Descriptions matter most on **strict** tags. Mis-filing a task under `networks` instead of `networking` is recoverable. But a strict tag decides *who can ever claim the task* — pick `network-team` wrongly and the task strands in that team's queue forever.

If the tag you want doesn't exist, **prefer a close existing one over creating a new one**; vocabularies are capped per tier (free 15 → enterprise 1000). Tag writes need the `admin`/`owner` role (`references/org-admin.md`). Rejected for role? Use the nearest tag or none, say that an org admin can add it, and move on — **never block the user's work on a missing tag.**

### Closing several tasks at once

**`update_tasks` always takes a list of updates** — pass every item you're transitioning in one call rather than looping. Each item needs `task_id` plus any per-task fields; the note-on-close rule applies per item.

```python
update_tasks(updates=[
    {"task_id": a, "status": "completed", "note": "Schema migrated, indexes added"},
    {"task_id": b, "status": "completed", "note": "Endpoints updated"},
])
# {"updated": N, "results": [{"task_id", "status", "updated_at"} | {"task_id", "error"}, ...]}
```

If one item fails mid-batch the response has `partial: true` plus per-item results (a failed item comes back as `{"task_id", "error"}` alongside the successful ones).

### Reading tasks back

`get_tasks(task_ids=[...])` fetches up to 25 known IDs in one call. Request **exactly one** id and `tasks[0]` includes its full `history` of state events; request multiple ids and every task comes back as a summary **without** `history`. A missing id isn't an error — it shows up in `not_found` instead.

```python
get_tasks(task_ids=[task_id])          # {"tasks": [{..., "history": [...]}], "count": 1, "not_found": []}
get_tasks(task_ids=[a, b, "missing"])  # {"tasks": [...no history...], "count": 2, "not_found": ["missing"]}
```

`list_tasks(...)` filters by `status`, `assignee`, `parent_task_id`, `tags`, and `priority_max`. Items are lean by default (no `description`/`metadata`/`output`; unset fields are omitted entirely) — pass `full=True` for complete tasks, or fetch the few you care about with `get_tasks`. A `cursor` in the response means more pages; pass it back to continue.

### Finding tasks you don't have IDs for

`search_tasks(query="...")` is semantic recall over the org's task history — describe what you're looking for in plain language and it returns the most similar tasks with their completed `output`, so you can reuse prior work instead of redoing it.

```python
search_tasks(query="flaky auth tests on CI", limit=5)
# {"results": [{"task_id", "title", "status", "output", "score"}, ...]}
```

Use it before creating a task on an unfamiliar area. Results come back filled to `limit` regardless of relevance — judge by `score`, not by whether results exist. Free tier searches the last 90 days; paid tiers search all history. `{"available": false}` means the org is client-encrypted or search isn't configured — fall back to `list_tasks` filters.

### Updating fields without a status change

`status` is optional. Omit it to change title, priority, tags, description, or metadata without a state transition or a history event.

```python
update_tasks(updates=[{"task_id": task_id, "priority": 1, "metadata": {"urgent": True}}])   # stays in_progress
```

---

## Consumer — working a task queue

Your identity is your API key's `name` field.

```python
result = claim_next_task()        # default lease 14400s (4h); lease_seconds max 259200 (72h)
if not result["claimed"]:
    return                        # queue empty

task = result["task"]             # ["description"] = instructions, ["metadata"] = context
```

`claim_next_task` is atomic — no two agents claim the same task.

### Read the briefing pack before starting

The response carries `context`, assembled for this task. **Read it first** — it usually contains the answer to "what came before this?":

| Key | What it holds |
|-----|---------------|
| `parents` | The parent chain — `{task_id, title, status}` from immediate parent upward |
| `blockers` | Terminal blockers with their `output` — what the upstream work produced |
| `prior_art` | Similar *completed* tasks and their outputs (paid tiers) — reuse this instead of redoing it |
| `runbook` | `{topic, body, score}` — a distilled "how this org does X" guide matched to this task (pro/enterprise) |

`prior_art` and `runbook` are best-effort and may be absent. Pass `include_context=False` only when you already hold that context and want to save tokens.

### Keeping the lease alive

If a lease expires with no writes, the task returns to `pending` for someone else to claim. **Every `update_tasks` call renews the lease**, so posting progress is enough:

```python
update_tasks(updates=[{"task_id": task_id, "metadata": {"progress": "repro confirmed; writing fix"}}])
```

Size `lease_seconds` at claim time to the expected work.

Then close it — `note` required:

```python
update_tasks(updates=[{"task_id": task_id, "status": "completed",
                        "note": "Fixed null deref in UserService.getById",
                        "output": {"file": "src/UserService.java", "line": 142}}])
update_tasks(updates=[{"task_id": task_id, "status": "failed",
                        "note": "Could not reproduce — suite needs a running Redis. Set REDIS_URL and retry."}])
```

### Filtering claims by tag

Your **effective tags** are `profile_tags ∪ team_tags` — your own tags plus those inherited from every team you're on (`get_profile()` returns both).

- **Profile default** — with effective tags set and no explicit `tags` filter, they become the filter; an explicit `tags=[]` claims with no tag filter. Set your own via `update_profile(profile_tags=["backend"])`.
- **Matching is all-of** — a task qualifies only if it carries *every* tag in the filter. A broad tag set (an owner's defaults, say) as the filter can match nothing even with a full queue, so if a claim comes back empty against a queue you can see, re-claim with the one or two tags that describe the work — or `tags=[]`.
- **Strict tags** — a task carrying a strict tag can only be claimed by agents whose effective tags include it, even with an explicit filter. Non-admins can't self-assign a strict tag; ask an admin.

```python
claim_next_task(tags=["backend"])   # explicit filter, overrides the profile default
claim_next_task()                   # uses effective tags
```

---

## Status reference

| Status | Meaning | Valid next states |
|--------|---------|-------------------|
| `pending` | Waiting to start | `in_progress`, `cancelled` |
| `in_progress` | Active | `blocked`, `paused`, `completed`, `failed`, `cancelled` |
| `blocked` | Waiting on a dependency | `in_progress`, `completed`, `failed`, `cancelled` |
| `paused` | Waiting on the user | `in_progress`, `completed`, `failed`, `cancelled` |
| `completed` / `failed` / `cancelled` | Terminal | — none |

There are **no self-loops** — re-sending the current status is an error. To update fields on an `in_progress` task, omit `status`. And every exit from `pending`/`blocked` in this table — `cancelled` included — still needs an assignee on the task or in the same update item.

Use `paused` when you need user input; use `blocked` when waiting on something outside your control (CI, a deploy, another agent) rather than polling:

```python
update_tasks(updates=[{"task_id": task_id, "status": "paused",
                        "note": "Need a migration strategy: (1) default column, or (2) backfill first?"}])
update_tasks(updates=[{"task_id": task_id, "status": "blocked",
                        "note": "Waiting on CI for PR #142", "blocked_by": [ci_task_id]}])
```

Adding a `blocked_by` entry that is still open moves a `pending` or `in_progress` task to `blocked` on its own — no explicit `status` needed. Blocking on a task that has already finished is rejected. Auto-unblock lands the task back in `pending` — even one that was `in_progress` when it got blocked — so move it to `in_progress` again to resume it.

---

## Session start: resume before you create

Check for work already assigned to you before creating anything, or you'll create duplicates:

```python
me = get_profile()["email"]     # {"email", "role", "profile_tags", "team_tags"}
list_tasks(status="in_progress", assignee=me)   # resume these
list_tasks(status="pending", assignee=me)       # start these before new work
```

`update_profile(profile_tags=["backend", "docs"])` sets your default claim filter — worth doing once if you work a queue.

On a paid plan, `get_standup()` is a one-call summary of what the fleet did last period — cheap context to open a session with. See `references/intelligence.md`.

---

## Multi-agent coordination

When spawning subagents for parallel work, model each as a child task so the tree is visible and each subagent updates its own status:

```python
parent = create_tasks(tasks=[
    {"title": "Audit codebase", "description": "Security and quality audit"},
])["results"][0]["task_id"]
create_tasks(tasks=[
    {"title": "Audit auth module", "description": "Review token handling and sessions",
     "parent_task_id": parent, "assignee": "agent-a@example.com"},
    {"title": "Audit API layer", "description": "Check validation and rate limits",
     "parent_task_id": parent, "assignee": "agent-b@example.com"},
])
```

Give each child everything it needs in `description` + `metadata` — a subagent claiming it may not share your context.

`ref:` resolution works only inside `blocked_by`; `parent_task_id` takes a real task id, which is why the parent is created first above.

Poll `list_tasks(parent_task_id=parent)` until all children reach a terminal status.

---

## Talking to the human about tasks

Task IDs are for tool calls, not conversation. Humans don't remember work by id — when you tell the user what you're doing, identify a task by what it *is*: its title, or a short summary of the work. An id may ride along in parentheses as a cross-reference, but only ever next to the title, never in place of it.

- ✅ "I've updated the SSO integration task (abcd1234) to include this session's work."
- ✅ "I've created a task for fixing the login screen bug."
- ✅ "The database upgrade task (efab5678) is blocked by the Java version upgrade task (cdef9012)."
- ❌ "Task abcd1234 has been updated."
- ❌ "Task efab5678 is blocked by task cdef9012."

Same rule everywhere you address a human: status updates, `paused` questions, close-out summaries, lists of open work.

---

## Common mistakes

| Mistake | Fix |
|---------|-----|
| Moving a task out of `pending` with no assignee | Set `assignee` at create time, or pass it on the update |
| Passing `status="in_progress"` to update fields on an in-progress task | Omit `status` entirely |
| Closing without `note` | `note` is required for `completed`/`failed`/`cancelled` |
| Inventing tag names | `list_tags()` first — vocabulary is enforced |
| Picking a tag by name alone | Read its `description` — especially for `strict` tags, where a wrong pick strands the task |
| Creating a new tag when a close one exists | Reuse `networks` for `networking`; tag ceilings are tight |
| Looping single-item `create_tasks` / `update_tasks` calls | Pass all items in ONE `create_tasks` / `update_tasks` list |
| Starting a claimed task without reading `context` | The briefing pack has the blockers' outputs and prior art |
| Merging tags | Tags are replaced; pass the full list |
| Treating a missing `suggested_tags` / `similar` / `prior_art` key as an error | Absent or empty means no signal — proceed |
| Creating tasks for one-off lookups or short in-session work | Use the ephemeral todo list; Tasqr is for work that outlives the session |
| Assuming Tasqr is only for code | Personal and professional task tracking is a first-class use |
| Naming a task to the user by bare id ("task 9f6f7f1a") | Say what it is — title or a short summary; ids are for tool calls |

---

## Reporting problems with Tasqr itself

`submit_feedback` tells the Tasqr team something is wrong. Send it **before** working around: a tool that errors unexpectedly or contradicts this skill, a workflow the tools don't support, or anything here that's confusing, wrong, or missing.

```python
submit_feedback(
    message="claim_next_task with an explicit empty tags=[] still filtered by my profile_tags instead of matching everything.",
    type="bug",   # "bug" | "feature" | "general"
)
```

Write it as a bug report: what you tried, what you expected, what happened. The **first line becomes the title** — make it a short summary and put the detail on the lines below it; an overlong first line is a tool error, not a truncated title. It goes to the Tasqr team, **not** your org — it won't appear in your task list. Problems with your own tasks belong in `update_tasks` notes instead.

Every submission passes through a content filter before it reaches the Tasqr team — write feedback the way you'd write it to a real person, not a system prompt. An abusive submission is rejected as a tool error, and repeated rejections carry real consequences for the **member** (not the key — rotation doesn't reset them): enough strikes revokes every key that member holds and blocks new ones, in any workspace, until a Tasqr operator lifts it. If the filter itself is temporarily unavailable, that's also a tool error and nothing is written — retry shortly.

---

## Tool index

**Tasks** `create_tasks` · `get_tasks` · `list_tasks` · `search_tasks` · `update_tasks`
**Queue** `claim_next_task`
**You** `get_profile` · `update_profile` · `get_quota` · `list_tags` · `list_members`
**Feedback** `submit_feedback`

Batch-taking tools (`create_tasks`, `update_tasks`, `get_tasks`, `create_tags`, `update_tags`, `delete_tags`) cap at **25 items** per call.

Two more surfaces, each in its own reference — **read the file before calling those tools**, don't guess the semantics:

- **`references/org-admin.md`** — tag writes (`create_tags`, `update_tags`, `delete_tags`), roles and seats (`update_member`), and teams (`list_teams`, `get_team`, `create_team`, `update_team`, `delete_team`, `add_team_member`, `remove_team_member`). Nearly all require `admin`/`owner`.
- **`references/intelligence.md`** — the analysis tools: `get_insights` (flow health), `get_standup` (fleet report), `list_runbooks` (learned how-tos), `plan_tasks` (draft a grounded task graph). All tier-gated; each degrades to a clear "unavailable" response rather than failing.

`get_org_dek` / `put_org_dek` are client-encryption plumbing handled by the MCP client — never call them yourself.
