# Benchmark And Regression

This project includes deterministic MCP dogfood plus a SWE-bench smoke/regression scaffold.

## Dogfood

Command:

```bash
make dogfood-mcp
```

Report:

- [reports/dogfood/codex-on-mcp.md](reports/dogfood/codex-on-mcp.md)
- [reports/dogfood/codex-on-mcp.json](reports/dogfood/codex-on-mcp.json)

Current conclusion:

```text
PASS
```

The dogfood runner starts the MCP server, then completes coding loops through MCP tool calls only:

- JavaScript bugfix: search, read, patch, `npm test`, diff.
- Python function add: read, patch, unittest, diff.
- Long-running stdin session.
- Workspace escape denial.

The report records tool calls and direct bypass status.

## SWE-bench Smoke Scaffold

Command:

```bash
make benchmark-smoke
```

Artifacts:

- [benchmarks/swebench/subsets/smoke-lite-10.json](benchmarks/swebench/subsets/smoke-lite-10.json)
- [benchmarks/swebench/predictions/baseline_native.jsonl](benchmarks/swebench/predictions/baseline_native.jsonl)
- [benchmarks/swebench/predictions/candidate_mcp.jsonl](benchmarks/swebench/predictions/candidate_mcp.jsonl)
- [reports/benchmark/swebench-regression.md](reports/benchmark/swebench-regression.md)
- [reports/benchmark/swebench-regression.json](reports/benchmark/swebench-regression.json)

Current conclusion:

```text
INCONCLUSIVE
```

The official SWE-bench harness was not run because:

- Docker is not installed in this container.
- The `swebench` Python package is not installed.
- Checked-in predictions are schema-valid placeholders, not real model-generated patches.

The project intentionally does not claim SWE-bench PASS without official harness evidence.

## Official Harness Commands

Baseline:

```bash
python -m swebench.harness.run_evaluation \
  --dataset_name princeton-nlp/SWE-bench_Lite \
  --predictions_path benchmarks/swebench/predictions/baseline_native.jsonl \
  --max_workers 2 \
  --run_id codex_tool_runtime_native_smoke
```

Candidate:

```bash
python -m swebench.harness.run_evaluation \
  --dataset_name princeton-nlp/SWE-bench_Lite \
  --predictions_path benchmarks/swebench/predictions/candidate_mcp.jsonl \
  --max_workers 2 \
  --run_id codex_tool_runtime_mcp_smoke
```

PASS requires:

```text
candidate_mcp_resolved >= baseline_native_resolved
```

