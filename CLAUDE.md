# CLAUDE.md

## Repository

This is a fork of [taylorwilsdon/google_workspace_mcp](https://github.com/taylorwilsdon/google_workspace_mcp). Upstream
is active with frequent contributions.

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
```

## Project Structure

- `gcalendar/calendar_tools.py` - Calendar tools (events, calendar CRUD, sharing)
- `auth/service_decorator.py` - OAuth scope management and `@require_google_service` decorator
- `core/tool_tiers.yaml` - Tool visibility tiers (core/extended/complete)
- `core/tool_registry.py` - Post-registration tier filtering
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

- `calendar_read` → readonly scope (get_events, list_calendars, query_freebusy)
- `calendar_events` → event CRUD scope (create/modify/delete events, move_event)
- `calendar_full` → full access scope (calendar metadata, ACL/sharing)

## Tool Tiers

Tools are tiered to control context window usage:

- **core** - Daily driver tools (4 calendar tools)
- **extended** - Management operations (delete_event, update_calendar, sharing, etc.)
- **complete** - Rare/admin/dangerous operations (delete_calendar, etc.)

Tiers are cumulative: requesting `extended` includes `core` + `extended`.