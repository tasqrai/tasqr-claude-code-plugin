# Tasqr analysis tools

Four read-mostly tools that answer "how is the fleet doing?" and "how do we usually do this?" from the org's own history. All are tier-gated, and all degrade to a clear unavailable response instead of half-working.

| Tool | Available on | Unavailable response |
|------|--------------|----------------------|
| `get_insights` | dev, pro, enterprise | `{"available": false, "reason": "not_computed_yet"}` |
| `get_standup` | dev, pro, enterprise | same, or `reason: "byok"` |
| `list_runbooks` | pro, enterprise | `{"available": false, "reason": "byok"}` |
| `plan_tasks` | pro, enterprise (metered) | `{"available": false, "reason": "byok" \| "not_enabled"}` |

On a tier that doesn't include the tool, the call **errors** with an upgrade message. That is an answer, not a failure — tell the user their plan doesn't include it and carry on; don't retry.

`reason: "byok"` means the org encrypts its own data client-side, so the server can't read task content. Nothing you do will change that.

---

## `get_insights()` — flow health

Recomputed every ~6 hours across a rolling window. Use it to find work that needs re-queueing, unblocking, or a human.

```python
get_insights()
# {"available": True, "computed_at": ..., "window_days": 30,
#  "summary": {...},
#  "signals": [{"type": "stuck_in_progress", "severity": "medium",
#               "count": 4, "task_ids": [...], "data": {}}, ...],
#  "throughput": {"days": [{"date": "2026-07-25", "created": 9, "completed": 7}, ...]},
#  "cycle_time": {"overall": {"p50_s": ..., "p90_s": ...}, "by_tag": {...}},
#  "failure": {"rate": 0.12, "by_tag": {...}},
#  "agents": [{"email": ..., "completed": 12, "failed": 1, "cycle_p50_s": ...}, ...],
#  "failure_clusters": [{"label": "flaky Redis fixture", "count": 6, "share_pct": 22}]}
```

Signal types: `stuck_in_progress`, `stale_blocked`, `aging_backlog`, `crash_loop`, `churning`, `failure_spike`. Severity ordered high → low.

**`task_ids` on a signal are actionable** — pass them straight to `get_tasks` to see what's wrong, then fix or re-queue. A signal without follow-through is just a number you read out. `task_ids` carries at most the top 20 ids; `count` is the true total, so a large signal lists only its head.

`failure_clusters` (pro/enterprise, weekly) groups recent failures by theme. Absent until the first clustering run.

---

## `get_standup()` — the fleet report

A short natural-language summary of what the fleet did in a period: shipped, failed, stuck, notable outputs, and the change versus the previous period. Cheap, high-value session-opening context.

```python
get_standup()                              # most recent, org-wide, any cadence
get_standup(cadence="weekly")              # most recent weekly
get_standup(scope="team:backend")          # one team's report
get_standup(period="2026-W30", cadence="weekly")
# {"available": True, "scope": "org", "period": "2026-W30", "cadence": "weekly",
#  "computed_at": ..., "headline": {...}, "body": "…prose…", "history": [...]}
```

- `cadence` — `daily` / `weekly` / `monthly`. Omit for the most recent of any cadence. Which cadences exist depends on the org's configuration (set in the dashboard, not here).
- `period` — `2026-07-20` (daily), `2026-W30` (weekly), `2026-06` (monthly). Omit for the most recent.
- `scope` — `org` (default) or `team:<name>`. A team report requires membership on that team, or `admin`/`owner`; otherwise the call errors.
- `history` — the last few reports for the same scope + cadence, for trend context.

`headline` is counts and deltas; `body` is the prose. Reports are generated on a schedule — you cannot trigger one.

---

## `list_runbooks()` — learned how-tos

Runbooks are distilled "how this org does X" guides, learned weekly from completed tasks. Read one before reinventing a procedure.

```python
list_runbooks()
# {"runbooks": [{"runbook_id", "topic", "body", "computed_at", "member_count"}, ...]}
```

`member_count` is how many tasks a runbook was distilled from — higher means better established. Runbooks are generated, not authored, and a stale one ages out on its own.

You often don't need this call at all: `claim_next_task` already attaches the best-matching runbook as `context.runbook`.

---

## `plan_tasks()` — draft a grounded task graph

Decomposes a goal into a dependency-wired task graph in exactly the `ref:` / `blocked_by` shape `create_tasks` accepts, grounded in the org's own history (similar completed tasks, typical cycle time, common failure tags, any matching runbook).

```python
plan_tasks(goal="Migrate billing off the deprecated Stripe API",
           context="Must stay dual-running for one release",
           max_tasks=8)
# {"plan": {"goal": ..., "tasks": [{"ref", "title", "description", "blocked_by",
#                                    "suggested_tags", "priority"}, ...],
#           "grounded_on": {...}},
#  "plan_calls_remaining": 17, "created": None}
```

`suggested_tags` are drawn from the org's tag vocabulary (plain names, no scores; empty when nothing fits) — advisory, like `create_tasks`' own suggestions. `create=True` creates the tasks with their `priority` but **no tags**; applying a suggestion stays a deliberate `update_tasks`.

**Metered separately from your task quota**: 20 calls/month on pro, 100 on enterprise. `plan_calls_remaining` comes back on every call. Don't burn calls iterating on phrasing — put the constraints in `context` the first time.

`create=True` submits the draft through the same validated `create_tasks` path:

- success → `created: [{task_id, ref}, ...]`
- upfront validation failed → `not_created: {reason}` and **nothing is written**
- mid-batch failure → `partial: true` and `failed: [{ref, error}]` alongside `created`

**Review before creating.** The default `create=False` returns the draft so you (or the user) can check it. Prefer that for anything non-trivial: the planner is grounded, not omniscient, and a wrong graph costs quota and cleanup. Editing a task list you drafted yourself is cheaper than deleting one you didn't.
