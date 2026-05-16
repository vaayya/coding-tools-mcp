# Security Sandbox Architecture Report

## Task Scope

Define the security model for the Coding Tool Runtime MCP Server before implementation. This report covers:

- Workspace root definition and confinement boundary.
- Path traversal defenses for read, search, edit, patch, image, and git tools.
- Symlink escape behavior.
- Command execution policy.
- Network and destructive-operation permission model.
- Timeout and output truncation behavior.
- Environment scrubbing.
- Interactive session lifecycle.
- Residual risks and implementation action items.

This is a design report only. No implementation files were changed.

## Sources

- `CODEX_GOAL_MODE_MCP_RUNTIME_TASK.md`
- Repository state in `/root/codex-tool-mcp` as of 2026-05-16: no runtime implementation files exist yet.

No external sources were consulted for this report.

## Key Findings

1. The project currently has no implemented MCP runtime, so security must be expressed as hard acceptance criteria for the first implementation rather than as a patch to existing behavior.
2. Workspace confinement is the central invariant. Every filesystem and process primitive must resolve through one shared path policy instead of each tool implementing its own checks.
3. Path traversal protection must handle existing paths, not-yet-existing write targets, symlinked parents, symlinked files, case-insensitive filesystems, and time-of-check/time-of-use races.
4. Shell execution cannot be made safe through command-string parsing alone. It needs layered controls: workspace-bound cwd, sandboxing, permission classes, environment scrubbing, timeout limits, output caps, and audit logging.
5. Network access and destructive operations should be denied by default for untrusted clients and require explicit policy grants.
6. Interactive sessions create longer-lived risk than one-shot commands. They need ownership, idle cleanup, strict stdin routing, bounded output buffering, and process-tree termination.

## Concrete Recommendations

### Workspace Root

The server must establish a single canonical workspace root before exposing tools.

- Prefer explicit configuration: `workspace_root` supplied at server startup or session creation.
- If no explicit root is supplied, use the server launch cwd only after canonicalization.
- Resolve the root with filesystem canonicalization, store it as `workspace_root_real`, and use that value for all containment checks.
- Reject unsafe roots unless an operator explicitly overrides them:
  - `/`
  - user home directory
  - system directories such as `/tmp`, `/var`, `/etc`, `/usr`, `/bin`, `/sbin`
  - Windows drive roots such as `C:\`
- Treat the root as immutable for a session. Changing root requires a new session.
- Return user-facing paths as workspace-relative paths to avoid leaking host layout unnecessarily.

Containment rule:

```text
resolved_target must be equal to workspace_root_real
or must be a descendant of workspace_root_real by path components.
String-prefix checks are not sufficient.
```

### Path Traversal Protection

All tools that accept paths must call one shared resolver. This includes file read, directory listing, search, patch application, image view, git diff helpers, fixture access, and command cwd.

Required resolver behavior:

- Accept relative workspace paths by default.
- Allow absolute paths only if the resolved path remains inside `workspace_root_real`; otherwise reject.
- Reject NUL bytes, empty path components where invalid, and platform-specific alternate path syntaxes that bypass normal resolution.
- Normalize `.` and `..`, then verify containment after resolving symlinks.
- For existing paths, resolve the full path with canonicalization and verify containment.
- For new write targets, canonicalize the nearest existing parent directory, verify that parent is inside the root, then validate the final basename.
- Reject paths whose normalized display form escapes the workspace, even if later components would re-enter it.
- Enforce a maximum path length and maximum component count to avoid pathological inputs.
- Do not special-case `.git`; tools may read git metadata only through approved git commands or explicit read tools subject to the same policy.

For recursive operations:

- Walk with APIs that expose file type metadata.
- Never follow a symlink before checking the resolved target.
- Apply maximum file count, byte count, and depth limits.
- Skip ignored or configured-excluded directories consistently for search and listing.

### Symlink Escape Behavior

Symlinks are allowed only when they do not escape the workspace.

Read behavior:

- A symlink inside the workspace may be read only if its final target resolves inside `workspace_root_real`.
- A symlink to a parent directory, sibling repository, home directory, temporary directory, or system path must be rejected.
- Broken symlinks may be listed as metadata but not opened as files.

Write and patch behavior:

- Do not write through symlinks by default.
- `apply_patch` must operate on regular files inside the workspace.
- Creating, deleting, or replacing symlinks should be denied unless a future explicit symlink tool is introduced with a higher permission level.
- Directory symlinks may be traversed for read/search only when their final target remains inside the workspace and recursive limits still apply.

Race protection:

- Use anchored filesystem operations where available, such as opening relative to a workspace directory handle.
- For final file opens, prefer no-follow semantics where supported.
- Re-check resolved metadata after opening sensitive files.
- Treat detected path changes during operation as an error.

### Command Execution Policy

Command execution is necessary for a coding runtime but must be policy-gated.

Default execution constraints:

- Commands run with cwd resolved and confined inside the workspace.
- If no cwd is supplied, use the workspace root.
- Spawn processes in a new process group so timeout cancellation can kill descendants.
- Capture stdout and stderr separately.
- Disable stdin unless an interactive session was explicitly requested.
- Use non-TTY execution by default; allocate a TTY only for tools that require it and only after policy allows it.
- Limit concurrent commands per session and per server.

Command input:

- Prefer structured argv over shell strings for internal tools.
- If a public exec tool accepts shell strings, mark it as higher risk and subject it to the permission model.
- Do not attempt to guarantee safety by parsing command text. Pattern detection can raise permission level, but sandbox and policy are the enforcement boundary.

Default allowed command class:

- Local read-only inspection commands such as `pwd`, `ls`, `find`, `rg`, `sed -n`, `git status`, `git diff`, `git show`, language format checks, and test commands that do not require network.

Permission-required command class:

- Commands that install dependencies, modify the repository, modify git history, start long-running servers, open network sockets, or access package registries.
- Examples: `npm install`, `pip install`, `cargo publish`, `git push`, `git clean`, `git reset --hard`, `rm -rf`, `chmod -R`, `docker`, `sudo`, package-manager commands, and cloud CLIs.

Denied command class:

- Privilege escalation commands.
- Host account management.
- Direct writes outside the workspace.
- Mount, kernel, firewall, and service-manager operations.
- Commands that intentionally disable sandboxing or exfiltrate secrets.

### Network and Destructive Permission Model

Use deny-by-default policy for risky capabilities.

Permission dimensions:

- `filesystem_read`: read/list/search inside workspace.
- `filesystem_write`: edit/create/delete regular files inside workspace.
- `command_readonly`: run local inspection commands.
- `command_write`: run commands expected to change files.
- `network`: make outbound network connections.
- `destructive`: delete files, rewrite git history, discard changes, or remove ignored artifacts.
- `credential_access`: read environment variables, config files, SSH keys, tokens, package credentials, or cloud credentials.

Default policy for untrusted MCP clients:

| Capability | Default |
| --- | --- |
| Workspace file read | Allow |
| Workspace file write | Require explicit grant |
| Read-only local commands | Allow with timeout |
| File-modifying commands | Require explicit grant |
| Network | Deny |
| Destructive commands | Require explicit grant and confirmation |
| Credential access | Deny |
| Outside-workspace access | Deny |

Approval decisions must be auditable:

- Record requested tool, arguments after redaction, cwd, permission class, decision, timestamp, and requester identity if available.
- Approval should be scoped to one operation by default.
- Broader grants must be time-limited and workspace-scoped.

Destructive operations:

- Never infer approval from natural language alone.
- Require structured permission state.
- Before running destructive commands, surface the current git status and the exact command or file operation.
- Do not permit irreversible deletion outside the workspace under any mode.

Network:

- Network access should be disabled in the sandbox by default.
- If enabled, prefer allowlisted domains or package registries instead of unrestricted egress.
- Treat package installation as both `network` and `command_write`.
- Redact credentials from network-related logs and outputs.

### Timeout and Output Truncation

Every operation needs bounded resource usage.

Recommended defaults:

| Operation | Default timeout | Hard max | Output cap |
| --- | ---: | ---: | ---: |
| File read | 5s | 30s | 1 MiB per file |
| Search | 15s | 60s | 512 KiB or 2,000 matches |
| Patch apply | 10s | 60s | 256 KiB |
| One-shot command | 30s | 10m | 1 MiB stdout + 1 MiB stderr |
| Interactive session idle | 10m | 60m | Ring buffer per stream |
| Git diff/status | 10s | 60s | 1 MiB |

Output truncation requirements:

- Preserve beginning and end of output when truncating where feasible.
- Report truncation explicitly with original byte count if known.
- Keep stdout and stderr truncation markers separate.
- Avoid splitting invalid UTF-8 in returned text; replace invalid bytes safely.
- Provide a follow-up read mechanism for active sessions, not unbounded tool responses.

Timeout behavior:

- On timeout, terminate the process group.
- Wait briefly for graceful shutdown, then force kill.
- Return timeout status, elapsed time, exit signal if available, and truncated output.
- Clean up temporary files created by the runtime itself.

### Environment Scrubbing

Commands must run with a scrubbed environment by default.

Allowed by default:

- Minimal runtime variables such as `PATH`, `HOME` pointing to a sandbox-safe home, `TMPDIR` inside a runtime temp directory, `LANG`, `LC_ALL`, and toolchain variables needed by configured tests.
- Workspace-specific variables explicitly set by the server policy.

Denied by default:

- API keys and tokens.
- Cloud credentials.
- SSH agent and private key paths.
- Package registry credentials.
- Git credential helper variables.
- Host proxy variables unless network permission is granted.
- Shell startup file injection variables.

Implementation notes:

- Use an allowlist, not a denylist.
- Set `HOME` to an isolated per-session directory where feasible.
- Set temp directories inside a controlled runtime area, not global host temp when possible.
- Redact secret-looking values from tool output and audit logs as a defense-in-depth measure.
- Avoid loading user shell profiles for non-interactive commands.

### Session Lifecycle

The MCP runtime should distinguish one-shot tool calls from persistent exec sessions.

Session creation:

- Associate each session with a workspace root, permission state, environment policy, timeout limits, and client identity if available.
- Assign an opaque session id.
- Start with no running process unless a command is explicitly launched.

Interactive stdin:

- `write_stdin` must require a valid live session id.
- Stdin must be routed only to the process that owns that session.
- Reject stdin after process exit, timeout, cancellation, or session close.
- Bound bytes per write and total bytes per session.

Output buffering:

- Maintain bounded ring buffers for stdout and stderr.
- Return incremental output with sequence numbers or cursors to avoid duplicates and unbounded payloads.
- Mark dropped output clearly.

Cleanup:

- Close sessions on process exit after a short retention window for final output.
- Reap child processes.
- Kill process groups on timeout, cancellation, client disconnect, or server shutdown.
- Remove runtime-owned temp directories.
- Enforce max session count per client and global max session count.

Audit:

- Log session start, command, cwd, permission class, approval decision, timeout, exit status, and cleanup result.
- Redact arguments and output fragments that match configured secret patterns.

### Git and Patch Safety

Git operations are frequent in coding loops and need explicit policy.

- `git status`, `git diff`, `git show`, and read-only history inspection are `command_readonly`.
- `git add`, `commit`, `merge`, `rebase`, `reset`, `clean`, `checkout`, `switch`, `push`, and tag operations are write or destructive depending on effect.
- `git reset --hard`, `git clean -fdx`, force push, and branch deletion are destructive.
- Patch application must validate every touched path before applying changes.
- Patch application must reject absolute paths and any target outside the workspace.
- Patch application should not change file mode, owner, group, xattrs, or symlinks in P0.

## Risks

- Path checks can be bypassed if each tool implements its own resolver. The resolver must be centralized and heavily tested.
- Symlink race protection is platform-dependent. Even with canonicalization, an attacker with write access can swap paths between check and open unless anchored/no-follow operations are used.
- Shell commands can perform indirect writes or network access that static command classification will miss.
- Dependency installers and test runners may execute arbitrary package scripts.
- Secret redaction is best effort and can miss encoded, split, or transformed secrets.
- Sandboxing behavior differs across Linux, macOS, and Windows. Cross-platform support needs separate test coverage.
- Long-running interactive sessions can accumulate hidden state and consume resources if lifecycle cleanup is incomplete.
- MCP clients may not provide reliable requester identity, limiting audit attribution.

## Action Items

1. Implement a single workspace path resolver and require all file-facing tools to use it.
2. Add compliance tests for `..`, absolute paths, symlink-to-outside, broken symlink, symlinked parent, new-file parent resolution, and prefix confusion such as `/repo` versus `/repo2`.
3. Add command policy tests for cwd escape, timeout kill, output truncation, environment scrubbing, network denial, and destructive command approval.
4. Add session lifecycle tests for stdin after exit, idle timeout, process-tree cleanup, output cursoring, and max-session enforcement.
5. Add audit-log tests with secret redaction fixtures.
6. Document operator configuration for trusted versus untrusted clients.
7. Treat security policy failures as `make compliance` blockers.

