# CLAUDE.md

## Repository

This is a fork of [taylorwilsdon/google_workspace_mcp](https://github.com/taylorwilsdon/google_workspace_mcp).
Upstream is active with frequent contributions.

**Remotes:**
- `origin` → `chrisguillory/google-workspace` (our fork - push here)
- `upstream` → `taylorwilsdon/google_workspace_mcp` (source of truth - pull from here)

**PRs go to our fork (`chrisguillory/google-workspace`), NOT upstream.**
Only open PRs to upstream when intentionally contributing back.

## Workflow

**Before starting any feature work, always sync with upstream:**

```bash
git fetch upstream
git rebase upstream/main
```

This prevents duplicate work (e.g., building a tool that already exists upstream) and reduces merge conflicts.

**Feature branches:** Always work on a feature branch, never directly on main.

```bash
git checkout main
git fetch upstream && git rebase upstream/main
git checkout -b feature/<name>
# ... work ...
git push origin feature/<name>
# PR to our fork:
gh pr create --repo chrisguillory/google-workspace
```

## Project Structure

- `gcalendar/calendar_tools.py` - Calendar tools (events, calendar CRUD, sharing, move)
- `auth/service_decorator.py` - OAuth scope management and `@require_google_service` decorator
- `core/tool_tiers.yaml` - Tool visibility tiers (core/extended/complete)
- `core/tool_registry.py` - Post-registration tier / permissions / read-only filtering
- `core/server.py` - Server entrypoint; hosts cross-service tools (e.g. `list_google_accounts`)
- Each Google service has its own subdirectory module

## Tool Pattern

All tools use the triple-decorator pattern:

```python
@server.tool()
@handle_http_errors("tool_name", service_type="calendar")
@require_google_service("calendar", "scope_group_name")
async def tool_name(service, user_google_email: str, ...) -> str:
```

## Scope Groups (Calendar)

Defined in `auth/service_decorator.py::SCOPE_GROUPS`:

- `calendar_read` → readonly scope (list_calendars, get_events, query_freebusy, list_calendar_sharing)
- `calendar_events` → event CRUD scope (manage_event, move_event)
- `calendar` → full calendar access scope (create/update/delete calendars, ACL/sharing)

Prefer `calendar_events` for anything scoped to events; use `calendar` only for calendar metadata
and ACL operations.

## Tool Tiers

Tools are tiered to control context window usage:

- **core** — Daily driver tools
- **extended** — Management operations (calendar/event CRUD, sharing, freebusy, OOO/focus-time, move)
- **complete** — Rare/admin/dangerous operations (e.g. `delete_calendar`)

Tiers are cumulative: requesting `extended` includes `core` + `extended`.

**Special: `list_google_accounts`** lives in the `accounts` YAML section but is force-kept across all
tier/permission filters (`core/tool_registry.py::filter_server_tools`) because multi-account workflows
need it regardless of tier.

## Events are consolidated in upstream

Upstream replaced `create_event`/`modify_event`/`delete_event`/`rsvp_event` with a single
`manage_event(action="create"|"update"|"delete"|"rsvp", ...)` tool. Similarly `manage_out_of_office`
and `manage_focus_time` are action-dispatched.

When adding event-related functionality, extend `manage_event` (or its `_*_impl` helpers); do not
resurrect the old separate tools.
