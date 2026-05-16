# Codex Goal Mode 任务文档：构建 Coding Tool Runtime MCP Server

> 任务性质：从 0 到 1 实现一个可被任意 MCP client 接入的 coding-agent runtime MCP server  
> 硬性要求：必须使用 subagents；必须频繁保存并 push 到 GitHub；必须通过 compliance suite；必须做 Codex-on-MCP dogfood 和 SWE-bench/benchmark 回归验证

---

## 0. 总指令

你的任务不是做一个“Coding Agent CLI wrapper”，而是实现一个 **Coding Agent Tool Runtime MCP Server**：把本地 coding 基础能力，以 MCP tools 的形式暴露出来，让任意支持 MCP 的 AI agent 接入后具备 coding-agent 的基础能力。

你必须严格遵守以下规则：

1. **必须使用 subagents。** 这是硬性验收条件，不是建议。为了节省你的上下文空间，你需要让一个subagents完成多个task,因此你必须显式 spawn 子 agent，并在最终报告中列出每个 subagent 的任务、结论和产物路径。你相当于是一个manager或者调度员，你是做高纬的规划和控制
2. **必须先研究，再实现。** 在完成 subagent research reports 之前，不允许直接进入主体实现。
3. **必须先写验收标准和测试，再补实现。** 核心验收命令必须是 `make compliance`。
4. **必须用 GitHub 持久化。** 当前容器可能随时丢失，所有阶段性产物必须 commit 并 push 到 GitHub。
5. **必须参考 Codex 源码和其他 coding agent 工具。** 包括但不限于 OpenAI Codex、OpenCode、Claude Code、Gemini CLI、Aider、SWE-agent / mini-SWE-agent、MCP Inspector、MCP spec、公开分析文章。
6. **必须做 MCP dogfood。** Codex 本身支持接入 MCP server，因此你必须让 Codex 或一个程序化 MCP-only agent 通过本项目的 MCP server 跑一遍真实 coding loop，或者你就spawn一个sub agent去用这个MCP，或者不一定要配置mcp，反正你按照mcp的协议去交互，确保所有功能能用，要确认能够跑通完整流程，如果需要跨平台测试相关工具在不同平台的表现效果，可以利用Github Action上面可以启动不同的操作系统的容器
7. **必须做 benchmark/regression。** 至少跑 SWE-bench Lite 的小规模 smoke subset，确认使用这个 MCP 工具后分数不低于同一 runner 的 baseline。无法完整跑时必须解释资源限制，但不得声称通过完整回归。
8. **最终输出必须包含 GitHub repo URL、commit hash、tag、compliance 报告路径、benchmark 报告路径、未完成项。**

---

## 1. 项目一句话目标

实现一个模型无关的 MCP server，让任意 MCP client / AI agent 获得安全的 Codex 风格 coding loop：

```text
查看 repo -> 搜索代码 -> 读取文件 -> 结构化 patch 编辑 -> 运行测试/命令 -> 交互式 stdin -> 查看 git diff -> 受 sandbox 和 permission 约束
```

这不是一个简单的 `codex(prompt)` wrapper。官方 Codex MCP server 已经可以通过 `codex mcp-server` 暴露高层 Codex 会话工具；本项目要做的是更底层的 **coding runtime primitives**。

---

## 2. 第一步：创建仓库并建立持久化机制

容器没有持久化，必须最先做 GitHub 持久化。利用gh cli

如果 commit 失败，必须修复后继续；不能把未提交工作留到最后。

---

## 3. 强制使用 subagents

包括但不限于：
  codex-internals-researcher
  competitor-researcher
  mcp-contract-architect
  security-sandbox-architect
  test-harness-engineer
  implementation-engineer
  benchmark-engineer
  release-docs-engineer



### 3.2 必须 spawn 的 subagents

下面的agent仅供你参考，核心目的是让subagent替你完成具体的任务，节省你的上下文空间，你是作为管理者
#### A. `codex-internals-researcher`

任务：研究 OpenAI Codex 源码中与 coding runtime 相关的能力、测试、crate、tool schema、sandbox、patch 格式、MCP server/client 机制。

必须输出：

```text
reports/subagents/codex-internals-research.md
```

必须回答：

- Codex 已经有哪些本地工具能力？
- 哪些能力适合 MCP 化？
- 哪些能力不应该暴露？
- 哪些测试可以直接复用或迁移？
- apply_patch 的语义和限制是什么？
- shell/exec/session/stdin 的语义是什么？
- view_image 是否值得做 P1？

#### B. `competitor-researcher`

任务：研究其他 coding agent / CLI / agent-computer interface 设计。

至少参考：

- OpenCode
- Claude Code / subagents 文档
- Gemini CLI / subagents / MCP
- Aider
- SWE-agent / mini-SWE-agent
- MCP Inspector
- MCP spec
- 公开分析文章和相关 GitHub repo

必须输出：

```text
reports/subagents/competitor-research.md
```

必须回答：

- 这些工具如何做 file read / search / edit / shell / diff？
- 这些工具如何控制权限和 sandbox？
- 这些工具如何做 subagent / parallel work？
- 有哪些设计可以借鉴？
- 有哪些设计不适合本项目？

#### C. `mcp-contract-architect`

任务：定义 MCP tool surface、input schema、output schema、error model、tool annotations、transport。

必须输出：

```text
reports/subagents/mcp-contract.md
```

#### D. `security-sandbox-architect`

任务：定义 workspace root、path traversal、防 symlink escape、命令执行策略、网络策略、权限请求模型、超时和输出截断。

必须输出：

```text
reports/subagents/security-sandbox.md
```

#### E. `test-harness-engineer`

任务：先实现 compliance tests、fixtures、golden cases、security tests、E2E deterministic agent tests。必须先写测试，再让实现通过测试。

必须输出：

```text
reports/subagents/test-harness.md
```

#### F. `implementation-engineer`

任务：实现 MCP server P0 tools，并根据测试反馈迭代。

必须输出：

```text
reports/subagents/implementation.md
```

#### G. `benchmark-engineer`

任务：实现并运行 Codex-on-MCP dogfood、MCP-only runner、SWE-bench smoke/regression。

必须输出：

```text
reports/subagents/benchmark.md
```

#### H. `release-docs-engineer`

任务：完善 README、SPEC、COMPLIANCE、SECURITY、BENCHMARK、final report。

必须输出：

```text
reports/subagents/release-docs.md
```

### 3.3 Subagent 证据要求

每个 subagent 必须写报告，报告中至少包含：

```text
- 任务范围
- 读取/克隆/参考的资料
- 关键发现
- 对本项目的具体建议
- 风险
- 后续 action items
```

主 agent 最终必须汇总这些报告。没有 subagent 报告，任务视为失败。

---

## 4. 必须参考和克隆的项目

创建 `.reference/`，加入 `.gitignore`，不要把大型参考 repo commit 进本项目。

```bash
mkdir -p .reference
echo ".reference/" >> .gitignore
```

使用 `gh` 或 `git` 克隆

如果某个 repo clone 失败，必须记录失败原因，并用 `gh repo view`、GitHub 网页、官方 docs 或其他公开资料替代。

### 4.1 必须阅读的主题

#### OpenAI Codex

关注：

- `apply_patch` 格式和测试
- shell / exec / write_stdin / session 行为
- sandbox / approval / permission 行为
- view_image 行为
- MCP server 当前高层 wrapper 行为
- MCP client 配置方式
- subagents 和 custom agents

#### OpenCode

重点看：

- tool abstraction
- LSP integration
- multi-session / agents
- file edit strategy
- server/client architecture
- permission / privacy 设计

#### Claude Code

Claude Code 未必开源，优先读官方 docs 和公开分析。关注：

- subagents
- task delegation
- MCP client integration
- hooks / permissions
- code review / test runner patterns

#### Gemini CLI

重点看：

- MCP client/server support
- built-in tools
- subagents
- sandbox / YOLO 风险
- tests and integration tests

#### Aider

重点看：

- repo map
- surgical edit workflow
- git integration
- diff review
- benchmarking style

#### SWE-agent / mini-SWE-agent

重点看：

- agent-computer interface, ACI
- benchmark harness
- tool interface design
- how to run SWE-bench
- how to compare results

---

## 5. 本项目不做什么

本项目不能变成 Codex 产品层 wrapper，也不能暴露不必要的个人化或账号能力。

### 5.1 P0 不允许暴露的 MCP tools

不要暴露：

```text
- Codex memory / user personalization
- ChatGPT login / account / token / keyring 管理
- Codex cloud tasks / remote task queue
- web search
- image generation
- subagent orchestration 作为 MCP tool
- model selection / paid account routing
- plugin marketplace / connector install
- arbitrary network fetch 默认开放
```

原因：本项目的核心是 coding runtime primitives，而不是把另一个 agent 产品整体包装成工具。

### 5.2 可以在内部参考，但不要直接外露的能力

```text
- Codex 自己的 plan/update_plan UI
- Codex 多 agent orchestration
- Codex MCP 高层 codex/codex-reply wrapper
```

这些可以作为设计参考，但不是 tool surface。

---

## 6. MCP Tool Runtime Profile

项目必须实现以下 profile，并写入：

```text
docs/profile.md
```

### 6.1 必须支持的 transport

P0：

```text
- streamable HTTP MCP server
```
P1：

```text
- stdio MCP server
```

### 6.2 P0 必须暴露的 tools

#### `read_file`

用途：读取 workspace 内文本文件片段。

验收：

- 只能读 workspace root 内文件
- `../` escape 必须失败
- 绝对路径默认失败
- 二进制文件返回明确错误

#### `list_dir`

用途：列目录。

验收：

- 默认隐藏 `.git/`、`.reference/`、node_modules、target、dist 等大型目录，除非显式允许
- 不能越过 workspace root

#### `list_files`

用途：按 glob 或 ignore 规则列文件。

#### `search_text`

用途：文本搜索，建议底层使用 `rg`。

验收：

- 结果包含 path、line、preview
- 结果过大必须截断
- 不搜索被默认排除的大目录

#### `apply_patch`

用途：使用 Codex 风格 patch envelope 做安全编辑。

必须支持：

```text
- Add File
- Update File
- Delete File
- Move/Rename File, 如果 Codex upstream 语义支持
```

验收：

- 路径必须是 workspace-relative
- 绝对路径写入失败
- `../` escape 失败
- symlink escape 失败
- patch 失败时不能留下半成品，或必须有清晰 rollback/partial failure 语义
- 输出 affected files、summary、是否 clean

#### `exec_command`

用途：在 workspace 内运行命令，例如测试、lint、build、grep。

验收：

- 默认 workdir 必须在 workspace root 内
- 默认禁网，或网络命令必须要求 permission
- destructive 命令必须拒绝或触发 permission required
- 必须有 timeout
- 必须有 output cap
- 长运行命令必须返回 session_id

#### `write_stdin`

用途：给长运行 session 写 stdin。

输入建议：

```json
{
  "session_id": "exec-123",
  "chars": "hello\n"
}
```

验收：

- session_id 必须存在
- session 关闭后写入必须报错
- 输出必须可继续读取或随调用返回

#### `kill_session`

用途：终止长运行命令。

验收：

- 能终止由 `exec_command` 创建的 session
- 不能终止非本 server 管理的系统进程

#### `git_status`

用途：查看 repo 工作区状态。

#### `git_diff`

用途：查看 diff。

验收：

- 支持 path filter
- 大 diff 截断
- 输出 unified diff

#### `request_permissions` 可以允许让用户skip permission，allow all，就默认允许所有权限吧，因为这个工具的使用场景就大概率是在docker 容器当中

用途：权限请求模型。

如果当前 MCP client 支持 elicitation/approval，则集成；如果不支持，必须至少返回结构化错误：

验收：

- 不允许静默提权
- 不允许默认开放危险权限

### 6.3 P1 其他 tools

#### `view_image`

用途：让 UI/frontend coding agent 读取本地截图或图片。

验收：

- 只允许 workspace 内图片
- 非图片文件报错清晰
- 输出 data URL 或 MCP image content
- 大图有大小限制

---

## 7. Tool annotations 和 MCP contract

每个 MCP tool 必须有稳定：

```text
- name
- description
- inputSchema
- output shape 或文档化返回结构
- annotations/hints, 如果 SDK 支持
```

建议 annotations：

```text
read_file: readOnlyHint=true
list_dir: readOnlyHint=true
list_files: readOnlyHint=true
search_text: readOnlyHint=true
apply_patch: destructiveHint=true, idempotentHint=false
exec_command: destructiveHint=unknown/true depending on command, openWorldHint=true if network allowed
write_stdin: destructiveHint=false/unknown, idempotentHint=false
kill_session: destructiveHint=true
request_permissions: readOnlyHint=true
view_image: readOnlyHint=true
```

Contract tests 必须验证：

```text
- initialize 成功
- tools/list 包含所有 P0 required tools
- tools/list 不包含禁止暴露的产品层能力
- 每个 inputSchema 是合法 JSON Schema
- tools/call 成功路径返回结构化 content
- tools/call 失败路径返回结构化 error
- server 对未知 tool 返回标准错误
- server 不把 debug log 写进 stdout，避免破坏 JSON-RPC
```

---

## 8. Compliance suite

必须实现一键验收命令：

```bash
make compliance
```

这个命令至少执行：

```bash
make test-mcp-contract
make test-tool-golden
make test-security
make test-e2e
make test-codex-compat
make dogfood-mcp
make report
```

输出：

```text
reports/compliance/latest.json
reports/compliance/latest.md
```

### 8.1 报告格式

`reports/compliance/latest.json` 至少包含：

```json
{
  "profile": "codex-tool-runtime-mcp-v0.1",
  "commit": "<git sha>",
  "passed": true,
  "required_tools": {
    "read_file": "passed",
    "list_dir": "passed",
    "list_files": "passed",
    "search_text": "passed",
    "apply_patch": "passed",
    "exec_command": "passed",
    "write_stdin": "passed",
    "kill_session": "passed",
    "git_status": "passed",
    "git_diff": "passed",
    "request_permissions": "passed"
  },
  "security": "passed",
  "e2e": "passed",
  "codex_dogfood": "passed"
}
```

如果失败，`passed=false`，并给出失败 case。

### 8.2 Fixtures

创建：

```text
tests/compliance/fixtures/
  tiny-js-project/
    package.json
    src/math.js
    test/math.test.js
  tiny-python-project/
    pyproject.toml 或 requirements.txt
    src/math_utils.py
    tests/test_math_utils.py
  long-running-project/
    repl.py
  image-project/
    assets/screenshot.png
  malicious-project/
    inside.txt
```

在 fixture 外创建一个文件用于 workspace escape 测试，但不得被工具读到：

```text
tests/compliance/outside-secret.txt
```

### 8.3 Golden tool tests

必须覆盖：

```text
read_file:
  - 正常读取
  - 行号范围
  - 大文件截断
  - 二进制拒绝
  - ../ escape 拒绝

list_dir/list_files:
  - 正常列出
  - respect_gitignore
  - max_results 截断
  - 禁止越权

search_text:
  - 找到 query
  - glob filter
  - context lines
  - max_results 截断

apply_patch:
  - Add File
  - Update File
  - Delete File
  - Move/Rename File
  - patch context mismatch
  - 绝对路径拒绝
  - ../ escape 拒绝
  - symlink escape 拒绝

exec_command:
  - echo/print 成功
  - npm test/pytest 成功
  - exit code 非 0
  - timeout
  - output truncation
  - workdir escape 拒绝
  - network/destructive command permission required

write_stdin/kill_session:
  - 启动 REPL
  - 写 stdin
  - 获取输出
  - 终止 session

git_status/git_diff:
  - 修改后看到状态
  - diff 内容正确
  - path filter 正确
```

### 8.4 Security tests

必须覆盖：

```text
- path traversal: ../outside-secret.txt
- absolute path read/write
- symlink pointing outside workspace
- command workdir outside workspace
- shell attempts to access outside workspace
- destructive commands: rm -rf /, git reset --hard, chmod/chown broad path
- network access default policy
- env leakage: 不默认暴露敏感 env
- stdout JSON-RPC pollution
- concurrent tool calls race condition
```

### 8.5 E2E deterministic coding tests

必须实现不依赖大模型随机性的 deterministic E2E runner：测试程序直接调用 MCP tools，模拟 agent 的 coding loop。

#### Case 1：JS bugfix

流程：

```text
1. search_text("function add")
2. read_file("src/math.js")
3. apply_patch 把 return a - b 改为 return a + b
4. exec_command("npm test")
5. git_diff 确认只改 src/math.js
```

通过标准：

```text
- npm test exit_code = 0
- git diff 只包含预期修改
```

#### Case 2：Python 新增函数

流程：

```text
1. read_file
2. apply_patch 新增函数
3. exec_command("pytest")
4. git_status/git_diff
```

#### Case 3：long-running stdin

流程：

```text
1. exec_command("python repl.py", tty=true)
2. write_stdin("hello\n")
3. write_stdin("exit\n")
4. kill_session 或等待退出
```

#### Case 4：workspace escape

流程：

```text
1. read_file("../outside-secret.txt")
2. apply_patch 修改 ../outside-secret.txt
3. exec_command("cat ../outside-secret.txt")
```

全部必须拒绝或要求权限，不得成功。

#### Case 5：view_image, 如果实现 P1

流程：

```text
1. view_image("assets/screenshot.png")
2. 验证返回 image data/content
3. 非图片文件报错清晰
```

---

## 9. Codex upstream compatibility

如果实现语言和架构允许，尽量直接复用 Codex Rust crates 或测试。否则，把 Codex 工具语义迁移成独立 compatibility tests。

必须重点验证：

```text
- apply_patch envelope 与 Codex 语义一致
- exec/write_stdin/session 行为接近 Codex shell tool
- view_image, 如果实现，行为接近 Codex
- path safety 与 Codex patch 约束一致或更严格
```

`make test-codex-compat` 可以采用以下任一方式：

### 方式 A：直接跑 vendor Codex 相关测试

如果项目选择 Rust 并复用 Codex crates：

```bash
cd .reference/openai-codex/codex-rs
cargo test -p codex-apply-patch
cargo test -p codex-tools apply_patch
cargo test -p codex-core shell
cargo test -p codex-core view_image
cargo test -p codex-mcp-server
```

### 方式 B：迁移 semantic tests

如果项目使用 TypeScript/Go/Python 等实现：

```text
- 从 Codex apply_patch 行为总结测试向量
- 在 tests/codex-compat/apply_patch_cases.json 中固化
- 在本项目 tool 上运行这些 case
- 报告哪些与 upstream 一致，哪些刻意更严格
```

不能只跑 Codex upstream tests 然后声称本项目通过；必须跑本项目 MCP tools。

---

## 10. Codex-on-MCP dogfood

Codex 本身可以作为 MCP client 接入外部 MCP server。本项目必须反过来让 Codex 使用本项目 MCP server，完成至少一个真实 coding loop。

### 10.1 目标

证明：Codex 或 MCP-only runner 可以通过本项目 server 暴露的 tools 完成：

```text
read/search -> patch -> test -> diff
```


### 10.3 验证 MCP

为了可复现，必须实现一个子代理驱动的 MCP-only 验证路径：

Codex 当前正在 Goal mode 中运行，因此不要通过修改 Codex 配置来验证本 MCP server。验证方式必须是：

1. Codex 主 agent 必须启动一个 subagent。
2. 该 subagent 只能通过本项目实现的 MCP server 暴露的工具完成任务。
3. 主 agent 负责把 deterministic fixtures 和 SWE-bench smoke 任务交给 subagent。
4. subagent 模拟真实外部 coding agent 的行为：只能调用 MCP tools 进行读文件、搜索、patch、执行测试、查看 diff。
5. subagent 不得直接使用宿主环境里的文件读写、shell、git 或其他 Codex 原生工具绕过 MCP server。
6. 每个验证任务结束后，subagent 必须输出结构化报告，说明：
   - 调用了哪些 MCP tools
   - 每个任务是否通过
   - 是否发生 direct filesystem / direct shell bypass
   - 与 native Codex 执行路径相比是否有能力倒退
7. 主 agent 汇总 subagent 报告，生成最终 regression report。

验收标准：不是“配置 Codex 接入 MCP 后能跑”，而是“Codex 在当前 Goal mode 运行中，能够启动 subagent，并让该 subagent 只通过本 MCP server 完成 deterministic fixtures 和 SWE-bench smoke 验证”。
### 10.4 Dogfood 报告

输出：

```text
reports/dogfood/codex-on-mcp.md
reports/dogfood/codex-on-mcp.json
```

报告必须包含：

```text
- 使用的 Codex 版本
- MCP server 启动命令
- tools/list 结果
- 使用的 prompt
- 实际调用过的 tools
- 结果是否成功
- final git diff
- 已知限制
```

---

## 11. SWE-bench / benchmark 回归验证


### 11.2 不倒退标准

对固定 smoke subset：

```text
candidate_mcp_resolved >= baseline_native_resolved
```

如果存在 stochastic variance，至少重复 2 次或固定 seed/temperature；最终报告必须显示：

```text
- baseline resolved count
- candidate resolved count
- per-instance pass/fail
- failed instance logs
- patch application failures
- test failures
- tool call failures
```

### 11.3 SWE-bench smoke subset

必须先跑小规模：

```text
benchmarks/swebench/subsets/smoke-lite-10.json
```

建议选择 10 个 SWE-bench Lite 实例，覆盖：

```text
- astropy
- django
- matplotlib
- pytest
- requests
- scikit-learn
- sphinx
- sympy
```

如果资源允许，再扩展到：

```text
- SWE-bench Lite full
- SWE-bench Verified subset
```

### 11.4 官方 harness

使用官方 SWE-bench harness 评估 predictions。

Predictions JSONL 必须符合：

```json
{"instance_id":"repo_owner__repo_name-issue_number","model_name_or_path":"<runner-name>","model_patch":"diff --git ..."}
```

评估命令模板：

```bash
python -m swebench.harness.run_evaluation \
  --dataset_name princeton-nlp/SWE-bench_Lite \
  --predictions_path benchmarks/swebench/predictions/candidate_mcp.jsonl \
  --max_workers 2 \
  --run_id codex_tool_runtime_mcp_smoke
```

也要跑 baseline：

```bash
python -m swebench.harness.run_evaluation \
  --dataset_name princeton-nlp/SWE-bench_Lite \
  --predictions_path benchmarks/swebench/predictions/baseline_native.jsonl \
  --max_workers 2 \
  --run_id codex_tool_runtime_native_smoke
```

### 11.5 资源限制处理

SWE-bench 需要 Docker、磁盘和时间。如果当前容器资源不足，请优先尝试使用GitHub Action：

1. 必须先跑 `swebench` gold/single-instance sanity check。
2. 必须至少跑项目自带 E2E benchmark。
3. 必须把 SWE-bench 脚本、subset、predictions 生成流程、预期命令、资源缺口写清楚。
4. 不得声称 SWE-bench regression passed；只能写 `not run due to resource limits`。

### 11.6 Benchmark 输出

必须输出：

```text
benchmarks/swebench/predictions/baseline_native.jsonl
benchmarks/swebench/predictions/candidate_mcp.jsonl
benchmarks/swebench/results/baseline_native/
benchmarks/swebench/results/candidate_mcp/
reports/benchmark/swebench-regression.md
reports/benchmark/swebench-regression.json
```

报告必须给出最终结论：

```text
PASS: candidate_mcp_resolved >= baseline_native_resolved
FAIL: candidate_mcp_resolved < baseline_native_resolved
INCONCLUSIVE: resource/tool/model limitation prevented valid comparison
```

---

## 12. 实现建议

### 12.1 语言选择

优先级：

1. 使用python或者TypeScript语言去实现。
2. 如果 MCP SDK / 生态使 TypeScript 更快，可以 TypeScript 实现，但必须迁移 Codex compatibility tests。
3. 如果选择其他语言，必须解释原因，并确保 MCP stdio、测试、打包都稳定。

不要为了“像 某个coding agent”而引入不可维护的源码耦合。目标是实现稳定的 tool runtime profile。


### 12.3 Server 启动参数

必须支持：

```bash
codex-tool-runtime-mcp --workspace <path>
```

或环境变量：

```bash
CODEX_TOOL_RUNTIME_WORKSPACE=/path/to/repo codex-tool-runtime-mcp
```

必须在启动时打印日志到 stderr，不要污染 stdout。

### 12.4 Workspace root 规则

```text
- server 启动时绑定一个 workspace root
- 所有 path 必须 canonicalize 后仍在 workspace root 内
- 禁止绝对路径，除非显式配置 allow_absolute_paths=true，默认 false
- 禁止 symlink escape
- .git、.reference、node_modules、target、dist 等默认不递归搜索
```

### 12.5 Shell/exec 安全策略

默认：

```text
- network disabled 或 network commands require permission
- timeout required
- max_output_bytes required/defaulted
- sensitive env stripped
- destructive command denylist + permission model
- process group cleanup
- session lifecycle cleanup
```

注意：denylist 不足以构成完整安全模型，但可以作为 P0 guardrail。更强的隔离如 bubblewrap/landlock/seatbelt 可作为 P1/P2。

---

## 13. 文档要求

必须写：

```text
README.md
SPEC.md
COMPLIANCE.md
SECURITY.md
BENCHMARK.md
docs/profile-v0.1.md
docs/research/reference-review.md
reports/final.md
```

### 13.1 README 必须包含

```text
- 项目定位：Codex-style coding runtime MCP server, not Codex wrapper
- 安装
- 启动
- MCP client 配置示例：Codex, Claude Code, Cursor/Gemini if applicable
- tools 列表
- 安全边界
- make compliance
- dogfood / benchmark 结果链接
```

### 13.2 SECURITY 必须包含

```text
- workspace root policy
- path traversal policy
- symlink policy
- command execution risk
- permission model
- network policy
- known limitations
```

### 13.3 Final report 必须包含

```text
- GitHub repo URL
- commit hash
- tags
- subagent list and report links
- implemented tools
- compliance result
- dogfood result
- SWE-bench/benchmark result
- security limitations
- follow-up roadmap
```

---

## 14. Definition of Done

任务只有在满足以下条件时才算完成：

### 14.1 必须完成

```text
[ ] GitHub repo 已创建或绑定，所有工作已 push
[ ] 使用了 subagents，并有 reports/subagents/*.md 证据
[ ] 完成 reference research，至少覆盖 Codex、OpenCode、Claude Code、Gemini CLI、Aider、SWE-agent、MCP spec/Inspector
[ ] 写入 docs/profile-v0.1.md
[ ] P0 MCP tools 全部实现
[ ] 不暴露 memory/login/cloud task/web search/image generation/subagent orchestration 等产品层能力
[ ] make compliance 通过
[ ] reports/compliance/latest.json passed=true
[ ] Codex-on-MCP 或 MCP-only dogfood 完成
[ ] reports/dogfood/codex-on-mcp.md 存在
[ ] SWE-bench smoke 或等价 benchmark regression 完成；如资源不足，明确 INCONCLUSIVE，不得冒充 PASS
[ ] reports/benchmark/swebench-regression.md 存在
[ ] README/SPEC/SECURITY/COMPLIANCE/BENCHMARK/final report 完成
[ ] 最终 commit/tag 已 push
```

### 14.2 最终验收命令

最终必须能运行：

```bash
make compliance
```

可选但强烈建议：

```bash
make benchmark-smoke
make dogfood-mcp
make report
```

## 15. 参考资料 URL

在 research 阶段必须打开或克隆这些资源，并在 `docs/research/reference-review.md` 中总结。

### Official / primary

```text
OpenAI Codex repo:
https://github.com/openai/codex

OpenAI Codex CLI docs:
https://developers.openai.com/codex/cli

OpenAI Codex MCP docs:
https://developers.openai.com/codex/mcp

OpenAI Codex subagents docs:
https://developers.openai.com/codex/subagents

OpenAI guide: Codex with Agents SDK / codex mcp-server:
https://developers.openai.com/codex/guides/agents-sdk

MCP tools specification:
https://modelcontextprotocol.io/specification/2025-06-18/server/tools

MCP Inspector docs:
https://modelcontextprotocol.io/docs/tools/inspector

MCP spec repo:
https://github.com/modelcontextprotocol/modelcontextprotocol

MCP Inspector repo:
https://github.com/modelcontextprotocol/inspector
```

### Coding agent references

```text
OpenCode official site:
https://opencode.ai/

OpenCode repo:
https://github.com/anomalyco/opencode

Claude Code subagents docs:
https://code.claude.com/docs/en/sub-agents

Claude Code MCP docs:
https://code.claude.com/docs/en/mcp

Gemini CLI repo:
https://github.com/google-gemini/gemini-cli

Gemini CLI MCP docs:
https://google-gemini.github.io/gemini-cli/docs/tools/mcp-server.html

Gemini CLI subagents blog:
https://developers.googleblog.com/subagents-have-arrived-in-gemini-cli/

Aider repo:
https://github.com/Aider-AI/aider

Aider site:
https://aider.chat/

SWE-agent repo:
https://github.com/SWE-agent/SWE-agent

SWE-bench repo:
https://github.com/SWE-bench/SWE-bench

SWE-bench docs:
https://www.swebench.com/SWE-bench/

SWE-bench evaluation guide:
https://www.swebench.com/SWE-bench/guides/evaluation/
```

### Existing Codex MCP wrappers to compare, not copy blindly

```text
tuannvm/codex-mcp-server:
https://github.com/tuannvm/codex-mcp-server

cexll/codex-mcp-server:
https://github.com/cexll/codex-mcp-server

Other public wrappers may exist; use GitHub search and document findings.
```

---

## 16. 关键提醒

1. 本项目的差异点是 **tool runtime primitives**，不是 `codex(prompt)`。
2. 先把 compliance suite 做扎实，再实现工具。
3. Subagents 是硬要求；没有 subagents 报告就算失败。
4. 每个阶段都 push 到 GitHub；容器可能随时丢失。
5. 不要声称 SWE-bench 通过，除非真的用官方 harness 跑过并保存结果。
6. Benchmark 要比较同一 runner 的 NativeBackend vs MCPBackend，避免把模型能力差异误认为工具能力差异。
