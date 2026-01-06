# Diagnosis: Git Changes Tab Showing Wrong Repository

## Summary
The OpenHands UI "Changes" tab is showing git changes from the wrong repository (showing `pebbles` changes instead of `OpenHands` changes).

## Root Cause
The backend endpoint `/api/conversations/{id}/git/changes` ignores the `selected_repository` setting from the conversation metadata and always checks ALL repositories under `/workspace`.

## Technical Details

### Current Behavior

1. **Frontend** (`frontend/src/utils/get-git-path.ts`):
   - Calculates the git path based on `selectedRepository` from conversation metadata
   - Returns `/workspace/project/{repo-name}` when a repository is selected
   - However, this path is NOT passed to the V0 backend endpoint

2. **Backend** (`openhands/server/routes/files.py` line 238-255):
   ```python
   async def git_changes(
       conversation: ServerConversation = Depends(get_conversation),
       ...
   ) -> list[dict[str, str]] | JSONResponse:
       runtime: Runtime = conversation.runtime

       # BUG: Always uses workspace root, ignores selected_repository
       cwd = runtime.config.workspace_mount_path_in_sandbox  # This is '/workspace'
       logger.info(f'Getting git changes in {cwd}')

       changes = await call_sync_from_async(runtime.get_git_changes, cwd)
   ```

3. **Git Changes Script** (`openhands/runtime/utils/git_changes.py`):
   - When called with `/workspace`, it finds ALL git repositories under that directory
   - Returns changes from ALL repos: `pebbles`, `OpenHands`, etc.
   - The frontend then shows changes from ALL repos mixed together

4. **Conversation Metadata** (`openhands/storage/data_models/conversation_metadata.py`):
   - Has `selected_repository: str | None` field
   - This field is set when the user clones or selects a repository
   - But it's NOT being used by the git_changes endpoint

### Current Directory Structure
```
/workspace/
├── .git/                    # Empty repo (no commits)
├── OpenHands/
│   └── .git/               # The repo user is working on
└── pebbles/
    └── .git/               # Another repo (showing its changes by mistake)
```

### Why This Happens

1. When `git_changes.py` runs from `/workspace`, it:
   - First tries `/workspace` itself (has `.git` but no commits, returns empty)
   - Then scans for subdirectories with `.git`: finds `OpenHands/.git` and `pebbles/.git`
   - Returns changes from BOTH subdirectories

2. The UI doesn't filter these results by `selected_repository`, so it shows all changes

## The Fix

The backend needs to:
1. Check `conversation.metadata.selected_repository`
2. If set, construct the path: `{workspace_mount_path_in_sandbox}/{selected_repository}`
3. Pass that specific path to `runtime.get_git_changes()`

### Example Fix (pseudocode):
```python
async def git_changes(
    conversation: ServerConversation = Depends(get_conversation),
    ...
) -> list[dict[str, str]] | JSONResponse:
    runtime: Runtime = conversation.runtime

    # Use selected repository if available
    cwd = runtime.config.workspace_mount_path_in_sandbox
    if conversation.metadata.selected_repository:
        # Extract just the repo name (e.g., "OpenHands" from "daniel-vainsencher/OpenHands")
        repo_name = conversation.metadata.selected_repository.split('/')[-1]
        cwd = os.path.join(cwd, repo_name)

    logger.info(f'Getting git changes in {cwd}')
    changes = await call_sync_from_async(runtime.get_git_changes, cwd)
```

## Additional Context

### Why `/workspace` has a `.git` directory
The workspace root `/workspace` has been initialized as a git repo but never committed to. This creates confusion but doesn't break the core functionality.

### Frontend vs Backend Path Handling
- **Frontend (V1 API)**: Uses `getGitPath()` which respects `selectedRepository`
- **Backend (V0 API)**: Ignores `selectedRepository` and always uses workspace root
- This inconsistency is likely why the bug exists

## Testing the Fix

After implementing the fix:

1. Open a conversation with `selected_repository` set to "daniel-vainsencher/OpenHands"
2. Navigate to the Changes tab
3. Verify it shows changes only from `/workspace/OpenHands`, not from other repos
4. Test with no repository selected (should show changes from all repos)
