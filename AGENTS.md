# CodeMate Repository Guide

## Scope

本文件适用于整个仓库。CodeMate 是面向本地代码仓库的 Coding Agent，核心目标是让模型
能够持续、安全、可恢复地调用工具完成真实开发任务。修改时优先保持现有模块边界和行为，
不要为了局部便利绕过 Runtime、权限、持久化或协议层。

## Before You Change Anything

1. 先运行 `git status --short`，工作区可能包含用户尚未提交的修改。不要覆盖、回滚或格式化
   与当前任务无关的文件。
2. 使用 `rg`、`rg --files` 定位实现和测试。修改某个行为前先阅读对应测试和模块文档。
3. 工作区就是启动时传入的目录。`WorkspaceContext` 不会向父目录搜索仓库根目录，不要重新
   引入向上发现逻辑。
4. 不读取、修改或提交 `.env`、用户密钥、`~/.codemate` 状态、`.codemate` 本地状态、
   `runs/`、缓存、虚拟环境和 `node_modules/`。
5. 仓库中的详细设计文档位于 `docs/设计总览.md` 和 `docs/modules/`。代码行为与文档不一致
   时，以经过测试确认的当前代码为准，并同步修正文档。

## Environment And Commands

项目要求 Python 3.10 及以上，依赖由 `uv` 管理。

```bash
uv sync
uv run codemate --help
uv run pytest -q
uv run ruff check codemate tests
```

优先运行与改动直接相关的测试，例如：

```bash
uv run pytest -q tests/runtime/test_tool_execution.py
uv run pytest -q tests/permissions tests/tools/test_sandbox.py
uv run pytest -q tests/context tests/memory
uv run pytest -q tests/bridge
```

VS Code 扩展使用独立的 Node/TypeScript 工具链：

```bash
cd vscode-extension
npm ci
npm run check
npm run compile
```

- `npm run check` 分别检查 Extension Host 和 Webview TypeScript。
- `npm run compile` 使用 `tsc` 构建 Extension Host，并用 `esbuild` 打包 Webview。
- `vscode-extension/dist/` 是生成产物，不要手工编辑。
- 只修改 Python 时不必运行 Node 构建；修改 Bridge 协议或扩展时必须同时检查两端。

## Architecture Map

- `codemate/cli.py`：CLI 参数、交互循环和 Agent 初始化入口。
- `codemate/runtime/agent.py`：Agent 组合对象、Session 状态和高层生命周期。
- `codemate/runtime/loop.py`：模型请求、commentary、工具调用和 final answer 的主循环。
- `codemate/runtime/tool_execution.py`：工具参数校验、权限 gate、审批、执行和结果 metadata 总闸口。
- `codemate/runtime/planning.py`、`review.py`：Plan Mode 与 Reviewer 子 Agent。
- `codemate/context/`：分层上下文、Token 预算、History 清理和压缩。
- `codemate/models/`：OpenAI、Anthropic、Ollama 等协议适配；Runtime 不应感知供应商细节。
- `codemate/tools/`：工具 schema、handler、权限校验、Shell 分析、沙箱、MCP 和 Todo。
- `codemate/storage/`：Session、Transcript、Task State、Trace、Run 和 ChangeSet 持久化。
- `codemate/memory/`：Legacy 与 Progressive 两套可切换长期记忆后端。
- `codemate/skill_evolution/`：多轮窗口、候选提取、Skill 合并和演化记录。
- `codemate/bridge/`：CLI Runtime 与 VS Code Extension Host 之间的 JSONL Bridge。
- `vscode-extension/src/`：运行于 Node.js Extension Host，负责进程、文件、Diagnostics 和原生 Diff。
- `vscode-extension/webview/`：运行于隔离浏览器环境，负责聊天界面和用户交互。
- `evals/`：Skill Evolution 等评测数据和执行器，不属于线上 Runtime。

## Core Invariants

### Model And History

- 内部只使用统一的 commentary、tool calls 和 final answer 表示。Provider 特有格式转换应留在
  `codemate/models/` 边界。
- 不破坏 Tool Call 与 Tool Result 的配对和顺序。Anthropic 转换要求一组 Tool Result 紧跟
  对应 Tool Use；History 清理与 Compact 必须保持消息组完整。
- 当前用户请求不能被 History Compact 吞掉。Todo、近期 Skill 调用、当前请求和必要的运行
  状态需要按现有恢复规则保留。
- 可压缩的模型 History 与面向用户回放的 append-only Transcript 职责不同。不要因为压缩
  History 而删除 Transcript；Checkpoint 恢复必须同时考虑 Session 和 Transcript 字节位置。

### Tool Execution

- 所有模型工具必须经过 Tool Registry、schema 校验、`validate_tool`、权限 gate 和
  `ToolExecutionMixin.run_tool`。不要从 Runtime 直接调用 handler 绕过审批与 Trace。
- 新增工具时同步维护 schema、handler/registry、validator、风险与只读 metadata、结果摘要、
  测试和用户文档。
- 工具错误作为可消费结果返回模型；只有真正无法继续的 Runtime 错误才应中断主循环。
- 工具结果存在统一大小限制。需要更多内容时应支持更窄范围或分页，而不是取消截断。
- 重复调用拦截用于阻止无新信息的死循环。改变判定逻辑时必须覆盖合理重试和错误拦截两类测试。

### File Safety

- 路径在权限判断前必须展开用户目录、按 Workspace 解析、消解 `..` 和符号链接，并使用真实
  绝对路径判断边界。
- 权限冲突始终 deny 优先；`read deny` 同时阻止写，`write allow` 隐含读取能力。不得通过
  新工具、Shell 包装或 MCP 绕过这些规则。
- 修改已有文件前必须已经读取该文件，且记录的文件状态仍与磁盘一致。创建不存在的新文件
  不要求预读；修改成功后要刷新 Session 中的文件状态。
- 工作区可能是 dirty Git 仓库，也可能完全没有 Git。不要假设 `git diff` 是文件变化的唯一来源。

### Shell And Sandbox

- Shell 静态分析负责识别命令意图、路径、重定向和危险操作；bwrap 沙箱负责运行时约束。
  两层职责不同，不能因为存在沙箱就删除静态审批，也不能把静态分析当成实际隔离。
- 保持 `required`、`optional`、`disabled` 沙箱模式的语义。`required` 不得静默降级为裸 Shell。
- 扩展 Shell 解析时使用结构化 token/shlex 逻辑，并为管道、重定向、`tee`、通配符、复合命令
  和敏感路径增加正反测试。
- 不扩大网络、环境变量或文件系统访问范围，除非任务明确要求并同时补齐权限与测试。

### MCP

- MCP stdio 的 stdout 只能承载协议消息，日志写 stderr。
- 当前同步 Runtime 使用专用后台事件循环管理异步 MCP transport 和 session。连接、调用和
  关闭必须留在同一个长期 loop/thread 中。
- 远程 Tool Call 失败后结果可能已经生效，不要自动重试具有未知副作用的调用。
- MCP 工具来自外部动态服务，不能仅根据名称视为只读；保持保守审批和 Trace metadata。
- 新增 MCP Server 时固定可访问边界，不把 Token 放入工具参数或返回值，并限制远程结果规模。

### Persistence And Concurrency

- 持久化写入优先使用 `codemate/storage/atomic.py` 中的原子写工具和现有文件锁。
- Schema 演进必须兼容旧 Session。读取旧数据时提供默认值，不因新增字段导致恢复失败。
- Session、Task State、Trace 和 Transcript 是不同工件，不要把它们合并为一份大状态文件。
- 后台 Memory/Skill 提取只有在完整成功后才能推进 checkpoint；失败或中断必须允许下次重试。
- Progressive Memory 更新前必须读取目标记忆并检查 revision，避免并发覆盖。

### Bridge And VS Code

- Bridge 使用一行一个完整 JSON 对象的 JSONL 协议。写入必须加锁并立即 flush，日志不能混入 stdout。
- 异步 UI 交互使用 `request_id` 标识请求、`interaction_id` 匹配一次审批或提问。新增事件时同步更新：
  Python protocol/server、TypeScript protocol、Extension Host 转发、Webview 消费和协议文档。
- Webview 不能直接访问文件系统。文件打开、代码跳转、Diagnostics、原生 Diff 和 ChangeSet 快照
  都由 Extension Host 调用 VS Code API 完成，再通过 `postMessage` 传递脱敏数据。
- Webview 渲染 Markdown 后必须继续使用 DOMPurify；不要通过 `innerHTML` 注入未净化内容。
- 修改消息渲染、批注、折叠、审批或滚动逻辑时，同时检查流式中间态、完成态、Session 恢复态
  和窄视口，避免只修正常完成路径。
- Code Diff 的工具级预览与整轮 ChangeSet 是两层能力。不要混淆单次 write/patch 预览、原生完整
  Diff、全部 Undo/Redo 和文件哈希冲突检查。

### Memory And Skills

- `memory.backend` 支持 `legacy`、`progressive` 和关闭模式；不要通过重构删除另一套后端。
- Core Memory 只保存用户明确支持的跨项目事实，更新证据必须来自当前用户输入；总结后的 value
  可以规范化表达，不要求逐字复制。
- Ordinary Memory 保存项目级长期知识，正文按需读取；默认上下文只放排名靠前的 ID 与 title。
- Skill 完整正文只在 `skill_load` 后进入上下文。手工 Skill 和 Agent 创建的 Skill 使用方式一致，
  但演化流程只能修改 Agent 自己创建的 Skill。
- Skill Evolution 的 Ready Window 必须在得到后续用户反馈后形成；工具参数和结果按既定上限裁剪，
  不要把完整 Shell 输出或大文件重新塞入提取上下文。

## Change Workflow

1. 先找行为入口、状态所有者和对应测试，再决定修改位置。避免在 UI、Prompt 或 Provider 适配层
   重复实现 Runtime 已有规则。
2. 保持改动集中。新增抽象只应消除真实重复或维护明确不变量。
3. 修改 Prompt 时同时检查 Runtime 强制约束。安全、顺序、持久化等关键行为不能只依赖提示词。
4. 修改配置时更新默认模板、用户/项目合并逻辑、README 示例和配置测试。
5. 修改协议或持久化格式时添加旧格式兼容测试。
6. 修改权限、安全或路径逻辑时至少添加一个允许用例和一个拒绝用例。
7. 修改前端功能时优先复用 VS Code 主题变量和现有组件，不引入与黑色工作台背景冲突的固定浅色。
8. 完成后检查 `git diff --check` 和 `git status --short`，确认没有缓存、密钥、运行工件或无关文件。

## Testing Matrix

根据改动范围选择测试，跨模块改动应组合运行：

| 改动 | 最低验证 |
| --- | --- |
| Model adapter | `tests/models/test_clients.py`、Runtime loop tests |
| Runtime/tool execution | `tests/runtime/`、对应 `tests/tools/` |
| 权限/Shell/sandbox | `tests/permissions/`、`test_shell_safety.py`、`test_sandbox.py` |
| Context/compact | `tests/context/`、相关 Session/Transcript tests |
| Memory | `tests/memory/`、Context tests |
| Skill evolution | `tests/skill_evolution/`、`tests/evals/` |
| Storage/checkpoint/change set | `tests/storage/`、Bridge retry tests |
| MCP client/server | `tests/tools/test_mcp.py`、`tests/mcp_servers/` |
| Bridge only | `tests/bridge/` |
| VS Code protocol/UI | Python Bridge tests、`npm run check`、`npm run compile` |

任务完成前尽量运行完整 `uv run pytest -q`。如果环境限制导致某项验证无法执行，要明确说明未验证
内容和剩余风险，不要用“应该能通过”代替结果。

## Code And Documentation Style

- 遵循现有 Python 和 TypeScript 风格，使用清晰的小函数和已有 helper；不要引入无必要依赖。
- 标识符使用英文。面向本项目维护者的解释性注释和文档优先使用中文，并说明“为什么”，不要逐行
  翻译代码。
- 用户可见错误信息保持具体、可行动，避免吞掉原始失败原因；同时不要输出 Token、密钥或完整
  敏感路径内容。
- 使用结构化解析器处理 JSON、JSONL、XML、Shell token 和消息对象，不用脆弱的字符串拼接模拟协议。
- README 只保留用户需要的安装、配置和能力概览；实现细节写入对应 `docs/modules/` 或示例文档。
- 不手工修改生成文件、缓存、评测运行结果或本地会话数据。

## Definition Of Done

- 请求行为已经端到端实现，不只修改了界面或提示词。
- 关键成功、失败和兼容路径都有测试。
- 相关文档、schema 和跨语言协议定义已经同步。
- 完整或合理范围的测试、Lint、TypeScript 检查已经实际运行。
- 最终 Diff 中没有用户无关修改、调试输出、密钥、缓存或生成工件。
