# 主从双会话自由编排分工（Dual-Agent 2.0 自由模式）

用户决定业务目标、约束、主从模型选型和最终验收。协作的核心是**“在哪用谁就是主控”**：由当前主控会话（`Codex` / `Antigravity IDE` / `AGY CLI`）负责定边界与审查 `git diff` 终验，把海量源码阅读、编码实现与本地自测闭环交给**全新独立子会话**（`agy` 或 `agy-gcp`），实现主从上下文物理隔离。

- **主控自绑定 & 会话首次 3 步实时选型（同会话后续自动复用）**：
  - **主控无需特意选择**：在哪个宿主窗口中发起任务，该窗口即为唯一主控（`Codex` 与 `Antigravity IDE` 只做主控、不做子 Agent）。
  - 同一会话**第一次**调用 `$dual-agent` Skill 时，必须先走 3 步串行弹窗，给出 `(Recommended)` 建议，**由用户手动点选确认后立即启动任务**：
    1. **实时查询并选主控模型**：实时拉取当前主控可用模型列表（Codex 读本地配置与模型缓存，Antigravity IDE / AGY CLI 调内置 `swarm(action="models", all=true)`），选择保持当前模型或顺手在顶栏切换目标主控模型；
    2. **选子 Agent 通道**：在无头 CLI `agy`（个人 Google Pro 通道）与 `agy-gcp`（GCP Vertex AI ADC 通道）中自由二选一；
    3. **实时查询并选子 Agent 模型**：实时跑所选子通道 `models` 命令，结合主控模型给出最佳搭档建议（如主控用旗舰 `Carbon` / `Argon` / `gpt-6.1` 时，推荐子会话用 1M 高速主力 `gemini-4-hydrogen-high` 或 `gemini-3.8-flash-high`），选定后立即开启**全新独立子会话**执行。
  - **支持同体系异构模型双会话**：即使在 Antigravity IDE 或 `agy` CLI 内做主控，子 Agent 也开启全新独立会话运行不同模型（如主控 `Argon Medium` / `Carbon Medium` + 子会话 `Hydrogen High`），绝不把几百步编码上下文塞进旗舰主会话。
  - **同会话持久记忆**：选定后的配置在本会话后续轮次与返工中自动沿用，无需重复弹窗（除非用户要求重选）。
- **执行与安全护栏**：
  - **锁死亲选模型，严禁私自篡改**：执行与返工期间严格锁定用户在弹窗中亲选的子模型，**严禁私自升配其他模型**；仅当遇配额耗尽（429）或端点持续不可用报错时，按同通道降级链（`agy` $\rightarrow$ `gemini-4-hydrogen-high` $\rightarrow$ `gemini-3.8-flash-high`；`agy-gcp` $\rightarrow$ 实时可用 `gemini-4-hydrogen-high` 或 `gemini-3.8-flash-high`）无缝兜底并告知用户。
  - **双通道严格防串号**：调 `agy` 必通过 `env -u` 剥离 GCP ADC 环境变量；调 `agy-gcp` 走可执行 `agy-gcp` 或 `SHELL_SESSIONS_DISABLE=1 /bin/zsh -ic 'agy-gcp "$@"' zsh` 位置参数透传，每次必传 `--model`。
  - **禁加短 `--print-timeout`**：保持默认 `0s`，由宿主等待机制（Codex V8 `while` + `functions.wait`，Antigravity IDE / CLI 后台任务反应式唤醒）零轮询承接。
  - **微小收尾主控兜底**：纯文档拼写或微小单行调整由主控直接改完验证，不拉起子会话。
- **四阶段极简协作闭环**：
  - **委托前**：检查脏工作区快照备份，明确业务目标、必要设计、不可变边界和验收条件，不对子 Agent 做过度指导或细碎限制。
  - **执行中**：直派独立子会话完成阅读、编码、运行测试与局部纠错；主控零轮询等待结果。
  - **交付后**：主控集中审查 `git diff` 并做独立构建/测试终验，把遗漏问题一次性交回同一子会话修复。
  - **出现阻塞时**：针对具体阻塞定向处理，主控不亲自下场重写业务代码。

完整交互向导与双宿主调用模板见 `$dual-agent` Skill。
