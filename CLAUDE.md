# CLAUDE.md

## Repository

Fork of [taylorwilsdon/google_workspace_mcp](https://github.com/taylorwilsdon/google_workspace_mcp),
synced to upstream **v1.21.1** with a small set of local customizations layered on top.
This checkout runs as the live Google Workspace MCP server.

**Remotes:**
- `origin` → `chrisguillory/google-workspace` (our fork — push here)
- `upstream` → `taylorwilsdon/google_workspace_mcp` (source of truth — pull from here)

**PRs go to our fork (`chrisguillory/google-workspace`), NOT upstream.** Only open PRs to
upstream when intentionally contributing back.

## How `main` relates to upstream

`main` is **an upstream release commit plus our customization commits on top**:

```
<our commits>      ← "Re-port <feature> onto upstream vX.Y.Z"
chore: release X.Y.Z   ← upstream release
```

To move to a newer upstream release: back up the current state to a `backup/…` branch
(see **Backups**), reset onto the new upstream release, then re-apply (cherry-pick) the
self-contained "Re-port …" commits. This rewrites history, so it ends in a
`git push --force-with-lease origin main`.

For ordinary feature work, branch off `main` and PR to the fork:

```bash
git checkout -b feature/<name>
# … work …
gh pr create --repo chrisguillory/google-workspace
```

## Our customizations (carry-overs)

Re-applied on top of each upstream release. Everything else is stock upstream.

| Tool | File | Tier | Notes |
|------|------|------|-------|
| `list_google_accounts` | `core/server.py` | `gmail.core` | Lists authenticated accounts from the credential store. |
| `update_calendar` | `gcalendar/calendar_tools.py` | `calendar.extended` | Patch calendar metadata (summary/description/timezone/location). |
| `manage_calendar_sharing` | `gcalendar/calendar_tools.py` | `calendar.extended` | Consolidated ACL tool: `action` ∈ {list, add, update, remove}. |
| `move_event` | `gcalendar/calendar_tools.py` | `calendar.extended` | Move an event between calendars (`events().move()`). Standalone because upstream's `manage_event` has no move action. |
| `delete_calendar` | `gcalendar/calendar_tools.py` | `calendar.complete` | Delete a secondary calendar (guards against `primary`). |

Discrete event CRUD (create/modify/delete) is upstream's consolidated `manage_event(action=…)`;
the old discrete calendar-sharing tools are subsumed by `manage_calendar_sharing`.

## Project structure

- `gcalendar/calendar_tools.py` — calendar tools (events, calendar CRUD, sharing, move)
- `core/server.py` — FastMCP `server` singleton + cross-service tools (`list_google_accounts`, auth)
- `auth/service_decorator.py` — OAuth scope groups + `@require_google_service`
- `auth/port_resolver.py`, `auth/oauth_callback_server.py` — OAuth callback port resolution (see below)
- `core/tool_tiers.yaml` — tool → tier mapping (core/extended/complete)
- `core/tool_registry.py` — post-registration tier filtering
- Each Google service has its own subdirectory module

## Tool pattern (v1.21.1)

Triple-decorator; the tool name is the function name:

```python
@server.tool(
    title="Move Event",
    annotations=ToolAnnotations(
        readOnlyHint=False, destructiveHint=False, idempotentHint=False, openWorldHint=True,
    ),
)
@handle_http_errors("move_event", is_read_only=False, service_type="calendar")
@require_google_service("calendar", "calendar_events")
async def move_event(service, user_google_email: str, ...) -> str:
    ...
```

(Older upstream used a bare `@server.tool()`. Current code passes `title=` + `ToolAnnotations`,
and `handle_http_errors` takes `is_read_only=`.)

## Calendar scope groups (`auth/service_decorator.py`)

- `calendar_read` → readonly (`get_events`, `list_calendars`, `query_freebusy`)
- `calendar_events` → event CRUD + move (`manage_event`, `move_event`)
- `calendar` → full access incl. calendar metadata + ACL/sharing (`update_calendar`, `delete_calendar`, `manage_calendar_sharing`)

(Upstream renamed the old `calendar_full` group to `calendar`.)

## Tool tiers

Cumulative — `extended` includes `core`; `complete` includes everything. Map a new tool to a
tier in `core/tool_tiers.yaml`. The live server loads the full set:

```bash
uv run main.py --tool-tier complete
```

## OAuth / callback port

`auth/port_resolver.py` + `auth/oauth_callback_server.py` resolve the OAuth callback port
(preferred port + fallback range, detecting a self-owned port). This fixes the recurring
"Port 8000 already in use" re-auth wall. After re-auth or code changes, reconnect from the
Claude client: `/mcp reconnect google-workspace`.

## Dev checks

Before committing:

```bash
.venv/bin/ruff format <files>          # house style (CI enforces it)
.venv/bin/ruff check <files>
.venv/bin/python -m py_compile <files>
```

Dependencies are managed with `uv` (`uv.lock`).

## Backups & preservation

Pre-/cross-sync states are preserved as branches on `origin` (GitHub), so local copies are
disposable:

- `backup/custom-main-pre-upstream` — our pre-upstream custom main (`098d03b`)
- `backup/origin-v1.19-sync` — a prior independent v1.19.0 sync that had been on `origin/main`
- `backup/pre-sync-main` — earlier pre-sync snapshot (`098d03b`)
- `backup/m4-uncommitted-e902e89`, `backup/m4-sync-upstream-stash` — M4 local WIP captured during the mesh sync

**Local backup branches can be pruned after ≈ June 15, 2026** (two weeks from the June 1, 2026
sync) — they live on `origin` regardless:

```bash
git branch -D backup/custom-main-pre-upstream backup/origin-v1.19-sync
```

## Mesh

`origin` is the source of truth; the repo is synced across the Mac mesh (M2/M3/M4) via
`claude-remote-bash`. M5 is not a workspace-mcp host. After a force-push to `main`, sync a
machine with `git fetch origin && git reset --hard origin/main` (a plain pull won't work
across rewritten history), then restart/reconnect its MCP server to load the new code.