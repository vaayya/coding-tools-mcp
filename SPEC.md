# Codex Tool Runtime MCP Spec

This repository implements the `codex-tool-runtime-mcp-v0.1` profile defined in [docs/profile-v0.1.md](docs/profile-v0.1.md).

## Scope

The server exposes coding runtime primitives over MCP. It is intentionally lower-level than a product agent wrapper. Clients can inspect, edit, test, and review a workspace, but cannot ask this server to route prompts to Codex, manage accounts, search the web, spawn subagents, or operate cloud tasks.

## Protocol

- MCP protocol version: `2025-06-18`
- P0 transport: Streamable HTTP at `/mcp`
- P1 transport: stdio with newline-delimited JSON-RPC
- Initialize response advertises tools and logging only.
- `tools/list` returns a stable default P0 tool set.
- `tools/call` returns MCP `content`, `structuredContent`, and `isError`.

## Workspace Model

Startup selects one workspace root:

```bash
codex-tool-runtime-mcp --workspace /path/to/repo
```

All path inputs are workspace-relative. The server rejects absolute paths by default, rejects `..`, canonicalizes existing paths, canonicalizes the nearest existing parent for writes, and rejects symlink escapes.

## Tool Set

Default P0 tools:

- `read_file`: read UTF-8 text slices with line and byte limits.
- `list_dir`: list directory entries with default exclusions.
- `list_files`: list files by glob with result caps.
- `search_text`: literal or regex text search with context and truncation.
- `apply_patch`: Codex-style patch envelope for add, update, delete, and move.
- `exec_command`: run bounded workspace commands and return final output or session id.
- `write_stdin`: write to server-managed running sessions.
- `kill_session`: terminate server-managed sessions.
- `git_status`: structured git working tree status.
- `git_diff`: bounded unified diff with path filters.
- `request_permissions`: structured non-granting permission path when elicitation is unavailable.

P1:

- `view_image`: feature-gated image data output.

## Forbidden Capabilities

The implementation must not expose:

- Codex/ChatGPT login, account, token, or keyring management.
- Codex memory or personalization.
- Codex cloud tasks or remote queues.
- Web search or arbitrary network fetch as a direct tool.
- Image generation.
- Subagent orchestration.
- Model routing or paid account selection.
- Plugin marketplace or connector installation.
- High-level `codex(prompt)` or `codex-reply` wrappers.

## Error Model

Tool execution failures return `isError: true` and structured content:

```json
{
  "ok": false,
  "error": {
    "code": "PATH_OUTSIDE_WORKSPACE",
    "message": "Path escapes the configured workspace.",
    "category": "security",
    "retryable": false,
    "details": {}
  }
}
```

Unknown JSON-RPC methods and malformed protocol-level requests use JSON-RPC errors.

## Implementation Notes

- Runtime implementation: [codex_tool_runtime_mcp/server.py](codex_tool_runtime_mcp/server.py)
- Test profile: [tests/compliance](tests/compliance)
- Current profile document: [docs/profile-v0.1.md](docs/profile-v0.1.md)

