# Compliance

The one-command acceptance gate is:

```bash
make compliance
```

It runs:

- `make test-mcp-contract`
- `make test-tool-golden`
- `make test-security`
- `make test-e2e`
- `make test-codex-compat`
- `make dogfood-mcp`

## Current Result

Latest local report files:

- [reports/compliance/latest.json](reports/compliance/latest.json)
- [reports/compliance/latest.md](reports/compliance/latest.md)

Current status in `latest.json`:

- `passed`: `true`
- `tests_run`: `29`
- P0 required tools: all `passed`
- `security`: `passed`
- `e2e`: `passed`
- `codex_dogfood`: `passed`

Two tests are skipped in the default profile because `view_image` is P1 and not exposed unless explicitly enabled.

## CI Evidence

GitHub Actions workflow:

- [.github/workflows/compliance.yml](.github/workflows/compliance.yml)

Latest verified run:

- https://github.com/ytagent/codex-tool-runtime-mcp/actions/runs/25957272106
- conclusion: `success`
- head SHA at run time: `7cada8b369eef55db6f8df2588d4d1943a62804e`

## Coverage

The suite verifies:

- MCP initialize, tools/list, tools/call, schemas, structured success/failure output, unknown tool behavior, and stdout protocol cleanliness.
- P0 tool golden cases for read/list/search/patch/exec/stdin/kill/git status/git diff.
- Security cases for traversal, absolute paths, symlink escape, command workdir escape, shell attempts to read outside workspace, destructive command policy, network default policy, sensitive environment scrubbing, stdout JSON-RPC pollution, and concurrent read-only calls.
- Deterministic E2E loops for JavaScript bugfix, Python function add, long-running stdin, and workspace escape denial.
- Codex compatibility vectors for patch envelope and exec/session/stdin behavior.
- MCP-only dogfood without direct filesystem or shell bypass during task execution.

## Running Individual Gates

```bash
make test-mcp-contract
make test-tool-golden
make test-security
make test-e2e
make test-codex-compat
make dogfood-mcp
make benchmark-smoke
```

