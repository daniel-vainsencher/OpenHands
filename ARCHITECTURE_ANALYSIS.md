# OpenHands Workspace Architecture Analysis

## Question
Are all agents let loose in the same workspace, or is the sandbox supposed to contain a single git project checkout at a time?

## Answer: Multiple Repositories Supported

**The sandbox workspace is designed to support MULTIPLE git repositories concurrently**, but each conversation/session tracks which repository it's working with.

## Evidence from Code

### 1. Workspace Structure (V0 - Current Implementation)

**Source**: `openhands/runtime/base.py` lines 532-534
```python
@property
def workspace_root(self) -> Path:
    """Return the workspace root path."""
    return Path(self.config.workspace_mount_path_in_sandbox)
```

**Source**: `openhands/core/config/openhands_config.py` lines 92-95
```python
workspace_mount_path_in_sandbox: str = Field(
    default=DEFAULT_WORKSPACE_MOUNT_PATH_IN_SANDBOX  # This is '/workspace'
)
```

The workspace root is `/workspace` by default.

### 2. Repository Placement Pattern

**Source**: `openhands/runtime/base.py` lines 927-929
```python
# In get_microagents_from_selected_repo()
if selected_repository:
    repo_root = self.workspace_root / selected_repository.split('/')[-1]
    microagents_dir = repo_root / '.openhands' / 'microagents'
```

**Source**: `openhands/server/session/agent_session.py` lines 154-155
```python
if self.runtime and runtime_connected and selected_repository:
    repo_directory = selected_repository.split('/')[-1]
```

When a repository like `"daniel-vainsencher/OpenHands"` is selected:
- The repo name is extracted: `"OpenHands"` (everything after the last `/`)
- The repository is placed at: `/workspace/OpenHands`
- Multiple repos can coexist: `/workspace/OpenHands`, `/workspace/pebbles`, etc.

### 3. Conversation Metadata Tracks Selected Repository

**Source**: `openhands/storage/data_models/conversation_metadata.py` lines 21-27
```python
@dataclass
class ConversationMetadata:
    conversation_id: str
    selected_repository: str | None  # Full name like "owner/repo"
    user_id: str | None = None
    selected_branch: str | None = None
    git_provider: ProviderType | None = None
```

Each conversation stores which repository it's working with.

**Source**: `openhands/server/session/agent_session.py` lines 485-488
```python
if selected_repository and repo_directory:
    memory.set_repository_info(
        selected_repository, repo_directory, selected_branch
    )
```

The agent's memory is configured with repository-specific context.

### 4. Architecture Intent: Repository-Specific Operations

Throughout the codebase, when operations need to be repository-specific, they follow this pattern:

**Source**: `openhands/runtime/base.py` lines 1246-1253
```python
def get_workspace_branch(self, primary_repo_path: str | None = None) -> str | None:
    if primary_repo_path:
        # Use the primary repository path
        git_cwd = str(self.workspace_root / primary_repo_path)
    else:
        # Use the workspace root
        git_cwd = str(self.workspace_root)

    self.git_handler.set_cwd(git_cwd)
    return self.git_handler.get_current_branch()
```

The code explicitly supports passing a repository-specific path.

## The Bug: git_changes Endpoint Ignores Repository Selection

**Source**: `openhands/server/routes/files.py` lines 238-249
```python
async def git_changes(
    conversation: ServerConversation = Depends(get_conversation),
    ...
) -> list[dict[str, str]] | JSONResponse:
    runtime: Runtime = conversation.runtime

    # BUG: Always uses workspace root, ignores selected_repository
    cwd = runtime.config.workspace_mount_path_in_sandbox
    logger.info(f'Getting git changes in {cwd}')

    changes = await call_sync_from_async(runtime.get_git_changes, cwd)
```

This endpoint:
1. Does NOT check `conversation.metadata.selected_repository`
2. Always scans the entire `/workspace` directory
3. Returns changes from ALL repositories, not just the selected one

**Note**: The `git_diff` endpoint has the same bug (lines 275-286).

## V0 vs V1 Architecture Differences

### V0 (Current, Legacy)
- **Location**: `openhands/server/routes/files.py`
- **Workspace path**: `/workspace`
- **Repository path**: `/workspace/{repo_name}`
- **Status**: Marked for deprecation (lines 1-7)

### V1 (Future)
- **Location**: `openhands/app_server/`
- **Workspace path**: `/workspace/project` (based on code in app_server)
- **Repository path**: `/workspace/project/{repo_name}` (based on frontend utility)
- **Status**: Still being migrated to

**Source**: `frontend/src/utils/get-git-path.ts` lines 9-22
```typescript
export function getGitPath(selectedRepository: string | null | undefined): string {
  if (!selectedRepository) {
    return "/workspace/project";
  }

  const parts = selectedRepository.split("/");
  const repoName = parts.length > 1 ? parts[1] : parts[0];

  return `/workspace/project/${repoName}`;
}
```

This frontend utility is designed for V1's structure but isn't being used by V0 endpoints.

## Current Directory Structure in Practice

```
/workspace/
├── .git/                    # Empty repo (workspace root git init)
├── OpenHands/
│   └── .git/               # Repository: daniel-vainsencher/OpenHands
├── pebbles/
│   └── .git/               # Repository: {some-owner}/pebbles
└── ...                     # Other potential repositories
```

## Recommended Fix

The `git_changes` (and `git_diff`) endpoints should:

1. Check if `conversation.metadata.selected_repository` is set
2. If set, construct path: `{workspace_root}/{repo_name}`
   - Where `repo_name = selected_repository.split('/')[-1]`
3. If not set, use workspace root (current behavior - scan all repos)

Example fix:
```python
async def git_changes(
    conversation: ServerConversation = Depends(get_conversation),
    ...
) -> list[dict[str, str]] | JSONResponse:
    runtime: Runtime = conversation.runtime

    cwd = runtime.config.workspace_mount_path_in_sandbox

    # Use selected repository path if available
    if conversation.metadata.selected_repository:
        repo_name = conversation.metadata.selected_repository.split('/')[-1]
        cwd = str(Path(cwd) / repo_name)

    logger.info(f'Getting git changes in {cwd}')
    changes = await call_sync_from_async(runtime.get_git_changes, cwd)
```

This matches the pattern used throughout the codebase for repository-specific operations.

## Summary

- **Design**: Sandbox supports multiple repositories under `/workspace`
- **Intent**: Each conversation works with one selected repository
- **Bug**: `git_changes` endpoint ignores repository selection
- **Impact**: Shows changes from all repos instead of just the selected one
- **Solution**: Apply the same repository-specific path pattern used elsewhere
