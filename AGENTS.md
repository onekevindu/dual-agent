# 双订阅异构双网协作分工（Dual-Agent 2.2）

用户决定业务目标、约束、模式选择与最终验收。协作的核心理论是**「幻觉错位 / 两张有破洞的网叠加」**：不同 Harness（运行框架）与不同厂商模型家族的思维盲区和疏漏点互不重合，让两家模型工具协同互补，大幅降低单模型幻觉穿透与漏检风险。

- **日常默认主推 `Codex + agy` 双订阅 & `agy` 默认模型锁定 `gemini-4-hydrogen-xhigh`**：
  - **日常主推**：在 **Codex / Codex CLI**（主 Harness，如 `gpt-6.1-sol`）中使用 `$dual-agent` 时，默认配合个人 Google Pro 无头 CLI **`agy`**（子 Harness，默认模型统一设为 **`gemini-4-hydrogen-xhigh`**），兼顾跨 Harness/跨厂商互补与固定双订阅成本优势。
  - **绝不越界推送 `agy-gcp`**：在 Codex / Codex CLI 及个人 `agy` / 个人 IDE 环境中调用本 Skill 时，**绝不主动推送 `agy-gcp`**；`agy-gcp` 仅作为 **Codex 不可用时的备选方案**（调用 Vertex AI Gemini 4 旗舰如 `gemini-4-carbon-medium` / `gemini-4-argon-medium` 顶替主控大脑），**只有在 `agy-gcp` 环境（`AGY_ADC_AUTH=true`）中使用 Skill 或用户显式点名时才推送**。
- **两种独立使用模式（严禁混为一谈，正常开发不插讨论阶段）**：
  1. **默认模式：需求全流程开发模式**（正常使用 `$dual-agent` 的默认模式，不额外插入“讨论需求阶段”）：
     - 同一会话首次进入开发时走 **2 步实时选型弹窗**（1. 查选主控模型；2. 锁定 `agy` 并查选子模型，首推 `gemini-4-hydrogen-xhigh`），选定后同会话开发与返工自动沿用；
     - 协作动态闭环：**主 Harness 规划 $\rightarrow$ `agy` 确认（只读核验可行性/提意见或内嵌核验） $\rightarrow$ 主 Harness 审核 $\rightarrow$ 人确认（仅需业务/风险拍板时动态触发） $\rightarrow$ `agy` 执行 $\rightarrow$ 主 Harness 审查 `git diff` 终验**。
  2. **专项模式：纯需求沟通讨论模式（免选模型直通）**：
     - 仅当用户显式要求让两个 Agent **沟通讨论需求 / 评审想法**（不写代码）时启用；
     - **绝不弹窗选模型**：直接使用**当前终端/窗口正在使用的模型** + **`agy` 默认 `gemini-4-hydrogen-xhigh`**（不推 `agy-gcp`）；
     - **Skill 只约束启动与只读边界，不限制讨论流程**：仅约束 `agy` 防串号启动、只读沙箱边界（禁写文件、放行只读检索）与无轮询等待；具体讨论几轮与如何收敛给出可靠结果，全权交由主控（如 Codex）自主规划。该模式不替代写代码时的 2 步选型弹窗。
- **执行与安全护栏**：
  - **默认禁写单测与禁加测试 POM 依赖**：除非用户在当前对话显式要求编写单元测试，否则主从均严禁新建 `src/test/**` 或在 `pom.xml` 中擅自新增 `test` 依赖；默认采用 `mvn compile`、SQL/表结构对齐或运行态验证。
  - **锁死模型，严禁私自升配**：开发模式锁定 Step 2 亲选模型，讨论模式锁定 `gemini-4-hydrogen-xhigh`；仅当遇配额耗尽（429）或端点持续不可用报错时，按同通道降级链（`gemini-4-hydrogen-xhigh` $\rightarrow$ `gemini-4-hydrogen-high` $\rightarrow$ `gemini-3.8-flash-high`）无缝兜底并告知用户。
  - **双通道严格防串号 & 禁短 `--print-timeout`**：调 `agy` 必通过 `env -u` 剥离 GCP ADC 变量，每次必传 `--model`；保持 `--print-timeout` 默认 `0s`，由宿主零轮询承接等待（Codex V8 `while` + `functions.wait`，IDE / CLI 后台任务反应式唤醒）。
  - **微小收尾主控兜底 & 不亲自重写业务代码**：纯文档拼写或微小单行调整由主控直接改完验证，遇复杂实现问题交回子会话修复，主控绝不亲自下场重写业务代码。

完整交互向导与双宿主调用模板见 `$dual-agent` Skill。
