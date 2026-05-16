# CI Verification Report

## Task Scope

Add and verify GitHub Actions CI for the MCP runtime compliance gate.

The dedicated CI subagents were started, but the platform returned usage-limit errors before they produced files. The manager completed this CI-only task because CI was a hard release gate.

## Artifacts

- `.github/workflows/compliance.yml`

The workflow installs the package and runs:

- `make compliance`
- `make benchmark-smoke`

## Runs

Initial run:

- URL: https://github.com/ytagent/codex-tool-runtime-mcp/actions/runs/25957243214
- conclusion: failure
- cause: GitHub runner took longer than local machine for `npm test`; tests expected an immediate exit instead of allowing the documented `exec_command` running-session behavior.

Fix:

- Commit: `a862f69d3c2f07adc263e45d59e031089d794db0`
- Change: JS fixture command tests now use a longer `yield_time_ms` so CI waits for final exit.

Verified run:

- URL: https://github.com/ytagent/codex-tool-runtime-mcp/actions/runs/25957272106
- conclusion: success
- head SHA: `7cada8b369eef55db6f8df2588d4d1943a62804e`

## Remaining Notes

GitHub emitted a Node.js 20 action deprecation warning for `actions/checkout@v4` and `actions/setup-python@v5`. It does not affect the current pass.

