# Tasqr org administration

Tag writes, member roles and seats, and teams. **Nearly everything here requires the `admin` or `owner` role** — roles *within the user's org*, not Tasqr staff. If a call is rejected for role, don't work around it: do the closest thing that doesn't need admin, and tell the user an org admin has to make the change.

If your key carries the `user` role, most of these tools won't appear in your tool list at all. That's the server hiding what you can't call, not a broken connection.

---

## Tag writes

The org's tag vocabulary is a controlled list — see the main skill for how to pick a tag. These three tools change the list itself; each takes up to 25 items and is all-or-nothing.

```python
create_tags(tags=[
    {"name": "backend", "strict": False, "default": False,
     "description": "Server-side work: API handlers, DB schema, background jobs."},
    {"name": "infra", "description": "Terraform, CI/CD, and AWS resource changes."},
])
# {"created": N, "tags": [...]}

# update_tags patches per item by key presence — omitted fields keep their current value.
update_tags(tags=[{"name": "backend", "description": "Server-side work, excluding infra."}])
update_tags(tags=[{"name": "backend", "description": None}])   # clear it ("" also works)
update_tags(tags=[{"name": "backend", "strict": True,
                   "description": "Server-side work owned by the Backend team."}])
# {"updated": N, "tags": [...]}

delete_tags(names=["backend", "infra"])   # leaves existing task tags untouched
# {"deleted": [...], "count": N}
```

- `description` — free text, max 200 chars. **Required whenever `strict=True`**, so a strict tag's description can never be cleared.
- `strict=True` — only members whose effective tags include this tag can claim tasks carrying it. Choose deliberately: a strict tag decides who can ever see the work.
- `default=True` — given to newly provisioned members as a starting `profile_tags` set. It does **not** auto-apply the tag to tasks.
- Per-tier ceiling: free 15, dev 50, pro 200, enterprise 1000.
- Seeded at org creation: `bug`, `feature`, `feedback`, `improvement`, `docs`, `chore`, `question`.

**Prefer reusing a near-miss tag over creating a new one.** Ceilings are tight and a sprawling vocabulary is worse than an imperfect match.

---

## Members, roles, and seats

```python
list_members()
# {"result": [{"email", "role", "profile_tags", "team_tags", "joined_at", "seated"}, ...]}

update_member("teammate@acme.com", role="admin")
update_member("agent@acme.com", profile_tags=["backend", "bug"])
update_member("leaver@acme.com", seated=False)
```

Three roles: `owner`, `admin`, `user`.

- Only an `owner` may promote to `owner` or modify another `owner`.
- The last `owner` cannot be demoted.
- You may lower your own role, never raise it.
- If the org's identity provider manages roles, `update_member` **rejects role changes** — the group membership in the IdP is the source of truth. Profile-tag and seat edits still work.

**`seated=False` immediately revokes that member's API keys.** It is how you off-board someone, not a soft toggle — confirm with the user before revoking anyone.

`profile_tags` set here are the member's default `claim_next_task` filter. Members can set their own with `update_profile`, but only an admin can give someone a **strict** tag.

---

## Teams

A team bundles tags. Every member of a team inherits its tags on top of their own `profile_tags` — that union is what claim filtering and strict-tag checks read. Teams are how you say "everyone on backend can claim backend work" once instead of per person.

Available on dev, pro, and enterprise (free gets 403). Limits: dev 10, pro 50, enterprise 200 teams.

```python
list_teams()
# {"result": [{"name", "tags", "description", "idp_only", "created_by", "created_at"}, ...]}

get_team("backend")
# {..., "members": [{"email", "sources": ["manual"], "added_by", "added_at"}, ...]}

create_team(name="backend", tags=["backend", "bug"],
            description="Server-side crew")
update_team("backend", tags=["backend", "bug", "infra"])   # omitted fields unchanged
update_team("backend", description=None)                   # clear the description
delete_team("backend")                                     # members keep their own profile_tags

add_team_member("backend", "dev@acme.com")
remove_team_member("backend", "dev@acme.com")
```

Rules worth knowing before you call these:

- **Names are lowercased**, max 32 chars, and may not contain `#`.
- **`tags` must already exist** in the org vocabulary. Changing a team's tags recomputes every member's effective tags.
- **`sources`** on a membership records where it came from: `manual` (assigned here) and/or `idp` (driven by the org's identity provider). Both can hold at once, so IdP churn never drops a manual grant.
- **An IdP-granted membership cannot be removed here** — `remove_team_member` only drops the `manual` source. Change the IdP group instead.
- **`idp_only` teams reject all manual membership edits.** Setting `idp_only` is owner-only, requires an active IdP, and enabling it drops existing manual members.
- If the org has SSO, an IdP group with the same name (case-insensitive) drives membership automatically — no mapping to configure.
