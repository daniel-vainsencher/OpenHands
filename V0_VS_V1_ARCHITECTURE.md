# OpenHands V0 vs V1 Architecture

## Summary

**Yes, there are TWO implementations of the OpenHands application server, and they coexist in the same codebase in the same branch, mounted at different URL paths.**

- **V0 (Legacy)**: Original implementation, marked for deprecation
- **V1 (Current)**: New implementation based on Software Agent SDK

## Directory Structure

```
openhands/
├── server/                    # V0 - Legacy implementation (DEPRECATED)
│   ├── routes/               # V0 API endpoints
│   │   ├── files.py         # <- Bug is here (git_changes endpoint)
│   │   ├── conversation.py
│   │   ├── settings.py
│   │   └── ...
│   ├── session/             # V0 session management
│   ├── listen.py            # V0 main server entry point
│   └── app.py               # V0 FastAPI app setup
│
└── app_server/               # V1 - New implementation
    ├── app_conversation/    # V1 conversation management
    ├── event/              # V1 event handling
    ├── sandbox/            # V1 sandbox management
    ├── user/               # V1 user management
    └── v1_router.py        # V1 router mounting
```

## How They Coexist

### V0 Routes (Legacy)
**Mounted at**: Root level (`/api/...`)

Examples:
- `/api/conversations/{id}/git/changes` - **This is where the bug is**
- `/api/conversations/{id}/git/diff`
- `/api/conversations/{id}/files`
- `/api/settings`
- `/api/secrets`

**Location**: `openhands/server/routes/*.py`

**Characteristic**: Every file has this header:
```python
# IMPORTANT: LEGACY V0 CODE
# This file is part of the legacy (V0) implementation of OpenHands
# and will be removed soon as we complete the migration to V1.
# Tag: Legacy-V0
```

### V1 Routes (New)
**Mounted at**: `/api/v1/...`

Examples:
- `/api/v1/app-conversations/` - Conversation management
- `/api/v1/events/` - Event handling
- `/api/v1/sandbox/` - Sandbox management
- `/api/v1/users/` - User management
- `/api/v1/webhooks/` - Webhook callbacks

**Location**: `openhands/app_server/*/`

**Characteristic**: Clean, modular architecture with service-based design

## How They're Mounted Together

**File**: `openhands/server/app.py` (lines 91-102)

```python
app = FastAPI(...)

# V0 Routes (root level)
app.include_router(public_api_router)           # V0
app.include_router(files_api_router)            # V0 <- git_changes is here
app.include_router(security_api_router)         # V0
app.include_router(feedback_api_router)         # V0
app.include_router(conversation_api_router)     # V0
app.include_router(manage_conversation_api_router)  # V0
app.include_router(settings_router)             # V0
app.include_router(secrets_router)              # V0
app.include_router(git_api_router)             # V0

# V1 Routes (under /api/v1)
app.include_router(v1_router.router)            # V1 (all mounted under /api/v1/)

app.include_router(trajectory_router)           # V0
```

Both sets of routes are registered in the same FastAPI application!

## Migration Status

### V0 Status
- **Status**: DEPRECATED, marked for removal
- **Usage**: Still actively serving the current frontend
- **Migration goal**: Being phased out as V1 implementations are completed
- **Example files with deprecation notice**:
  - `openhands/server/routes/files.py`
  - `openhands/server/routes/settings.py`
  - `openhands/server/routes/conversation.py`
  - `openhands/server/dependencies.py`
  - `openhands/server/listen.py`
  - Many more...

### V1 Status
- **Status**: Active development, the future
- **Based on**: Software Agent SDK (external repository)
- **Link**: https://github.com/OpenHands/software-agent-sdk
- **Architecture**: More modular, service-based design
- **Completion**: Still being built out, not all V0 endpoints have V1 equivalents yet

## Key Architectural Differences

### V0 Architecture
```
Client → WebSocket/HTTP → V0 Routes → Session → AgentSession → Runtime
```

- Monolithic session management
- Routes directly in `openhands/server/routes/`
- Tight coupling between components
- Uses `ServerConversation` model

### V1 Architecture
```
Client → HTTP/REST → V1 Routes → Services → SDK → Sandbox
```

- Service-based architecture
- Clear separation of concerns:
  - **Router** layer: HTTP endpoints (`*_router.py`)
  - **Service** layer: Business logic (`*_service.py`)
  - **Model** layer: Data models (`*_models.py`)
- More scalable and maintainable
- Uses Agent SDK for core agent functionality

## Workspace Path Differences

This is relevant to the git_changes bug!

### V0
- **Workspace root**: `/workspace`
- **Repository path**: `/workspace/{repo_name}`
- **Example**: `/workspace/OpenHands`

### V1
- **Workspace root**: `/workspace/project`
- **Repository path**: `/workspace/project/{repo_name}`
- **Example**: `/workspace/project/OpenHands`

**Evidence**:
- V1 sandbox services set `working_dir='/workspace/project'`
- Frontend utility `getGitPath()` generates `/workspace/project/{repo_name}` paths
- But V0 still uses `/workspace` directly

## Why Does This Matter for the Bug?

The `git_changes` endpoint is part of **V0** and:

1. Lives in `openhands/server/routes/files.py` (marked LEGACY V0)
2. Serves requests to `/api/conversations/{id}/git/changes`
3. Ignores `conversation.metadata.selected_repository`
4. Always scans entire `/workspace` directory
5. Has not been migrated to V1 yet (no V1 equivalent exists)

**The fix must be applied to the V0 code** because:
- That's what the frontend currently uses
- V1 doesn't have a git_changes endpoint yet
- V0 will continue to be used until migration is complete

## Finding Other V0/V1 Code

### To find V0 code:
```bash
grep -r "Legacy-V0" --include="*.py" openhands/
```

### To identify V1 code:
```bash
ls -R openhands/app_server/
```

### To see all mounted routes:
```bash
grep "include_router" openhands/server/app.py
```

## Timeline Context

From the README in `openhands/app_server/`:
> As of 2025-09-29, much of the code in the OpenHands repository can be regarded
> as legacy, having been superseded by the code in AgentSDK.

The migration to V1 is ongoing, but V0 is still the primary implementation being used by the frontend for many operations.

## Implications for Development

### When fixing bugs:
1. **Check which version**: Look for "Legacy-V0" tags or check the directory
2. **Fix in the right place**: Don't fix V0 bugs in V1 or vice versa
3. **Consider migration**: If fixing V0, consider if V1 needs the same fix

### When adding features:
1. **Prefer V1**: New features should go in V1 if possible
2. **Check compatibility**: Ensure frontend can use the new V1 endpoint
3. **Document migration**: Help track what still needs migration

### When testing:
1. **Test the right version**: Ensure you're testing V0 or V1 as appropriate
2. **Check URL paths**: `/api/...` is V0, `/api/v1/...` is V1
3. **Consider both**: Some features may exist in both versions

## Summary

- **Two implementations**: V0 and V1 coexist in the same codebase
- **Same branch**: Both in `main`, in separate directories
- **Mounted together**: Both registered in same FastAPI app at different paths
- **V0**: `openhands/server/` → root-level routes (`/api/...`)
- **V1**: `openhands/app_server/` → prefixed routes (`/api/v1/...`)
- **Migration status**: V0 is being phased out, V1 is the future
- **Bug location**: The git_changes bug is in V0 code (`openhands/server/routes/files.py`)
