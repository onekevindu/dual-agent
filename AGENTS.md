# 双订阅异构双网协作分工（Dual-Agent 2.1）

用户决定业务目标、约束、主从模型选型和最终验收。协作的核心理论是**「幻觉错位 / 两张有破洞的网叠加」**：不同 Harness（运行框架）与不同厂商模型家族的思维盲区和疏漏点互不重合，让两家模型工具协同互补（主 Harness 负责规划审核，子 Harness 负责可行性验证提意见与任务执行），大幅降低单模型幻觉穿透与漏检风险。

- **日常默认主推 `Codex + agy` 双订阅 & `agy-gcp` 备选隔离规则**：
  - **日常主推**：在 **Codex / Codex CLI**（主 Harness，如 `gpt-6.1-sol`）中使用 `$dual-agent` 时，默认直接配合个人 Google Pro 无头 CLI **`agy`**（子 Harness，如 1M 主力 `gemini-4-hydrogen-high`），兼顾跨 Harness/跨厂商互补与固定双订阅成本优势。
  - **绝不越界推送 `agy-gcp`**：在 Codex / Codex CLI 及个人 `agy` / 个人 IDE 环境中调用本 Skill 时，**绝不主动推送 `agy-gcp`**；`agy-gcp` 已降级为 **Codex 不可用时的备选方案**（调用 Vertex AI Gemini 4 旗舰如 `gemini-4-carbon-medium` / `gemini-4-argon-medium` 顶替主控大脑），**只有在 `agy-gcp` 环境（`AGY_ADC_AUTH=true`）中使用 Skill 或用户显式点名时才推送**。
- **主控自绑定 & 首次触发「2 步实时选型」（同会话后续自动复用）**：
  - **在哪用谁就是主控**：在哪个宿主窗口发起任务，该窗口即为唯一主控（`Codex` 与 `Antigravity IDE` 只做主控、不做子 Agent）。
  - 同一会话**第一次**调用 `$dual-agent` 时走 2 步串行弹窗，给出 `(Recommended)` 建议，由用户手动拍板后立即启动：
    1. **实时查询并选主控模型**：Codex 读 `~/.codex/config.toml` 与 `~/.codex/models_cache.json`，Antigravity IDE / `agy` / `agy-gcp` CLI 调内置 `swarm(action="models", all=true)`；
    2. **按宿主自动锁定子通道并实时查选子模型**：Codex 与个人环境直接锁定 `agy`（不推 `agy-gcp`），仅 `agy-gcp` 环境（或用户显式点名）才推送 `agy-gcp`；实时运行对应通道 `models` 查询并结合主控模型推荐最佳互补拍档，选定后立即开启**全新独立子会话**执行。
  - **同会话持久记忆**：选定后的配置在本会话后续轮次与返工中自动沿用，无需重复弹窗（除非用户要求重选）。
- **执行与安全护栏**：
  - **默认禁写单测与禁加测试 POM 依赖**：除非用户在当前对话显式要求编写单元测试，否则主从均严禁新建 `src/test/**` 或在 `pom.xml` 中擅自新增 `test` 依赖；默认采用 `mvn compile`、SQL/表结构对齐或运行态验证。
  - **锁死亲选模型，严禁私自升配**：执行与返工期间严格锁定用户在弹窗中亲选的子模型；仅当遇配额耗尽（429）或端点持续不可用报错时，按同通道降级链（`gemini-4-hydrogen-high` $\rightarrow$ `gemini-3.8-flash-high`）无缝兜底并告知用户。
  - **双通道严格防串号**：调 `agy` 必通过 `env -u` 剥离 GCP ADC 环境变量；调 `agy-gcp` 走可执行 `agy-gcp` 或 `SHELL_SESSIONS_DISABLE=1 /bin/zsh -ic 'agy-gcp "$@"' zsh` 位置参数透传，每次必传 `--model`。
  - **禁加短 `--print-timeout`**：保持默认 `0s`，由宿主等待机制（Codex V8 `while` + `functions.wait`，Antigravity IDE / CLI 后台任务反应式唤醒）零轮询承接。
  - **微小收尾主控兜底**：纯文档拼写或微小单行调整由主控直接改完验证，不拉起子会话。
- **「两张网叠加」四阶段协作闭环**：
  - **委托前（主 Harness 规划）**：检查脏工作区快照备份，明确业务目标、必要设计、不可变边界和验收条件，不对子 Harness 做微观指导。
  - **执行中（子 Harness 核验提意见 + 执行）**：先对照真实代码核验主控方案可行性（若存在核心假设错误或契约冲突则停手提意见交主控拍板），可行则自主在独立子会话完成修改与编译自检，并在摘要中单列「可行性核验与补充意见」；主控零轮询等待结果。
  - **交付后（主 Harness 审 diff 终验）**：主控先审可行性意见，再集中审查 `git diff` 并做独立编译/运行态验证，把遗漏问题一次性交回同一子会话修复。
  - **出现阻塞时**：针对具体阻塞定向处理，主控绝不亲自下场重写业务代码。

完整交互向导与双宿主调用模板见 `$dual-agent` Skill。
