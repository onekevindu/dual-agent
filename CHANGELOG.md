# Changelog

## [2.2.0] - 2026-11-17 — 新增「纯需求沟通讨论模式（免选模型直通）」、全模式默认升级 `gemini-4-hydrogen-xhigh` 与动态开发闭环

1. **全模式 `agy` 默认主力模型升级为 `gemini-4-hydrogen-xhigh`**：
   - 替换原 `gemini-4-hydrogen-high`，统一作为默认开发模式与纯需求沟通讨论模式的默认推荐/锁定模型；同步顺延同通道 429 降级链为 `gemini-4-hydrogen-xhigh` $\rightarrow$ `gemini-4-hydrogen-high` $\rightarrow$ `gemini-3.8-flash-high`。
2. **明确两大独立场景模式（互不混杂，正常开发不插讨论阶段）**：
   - **默认模式（需求全流程开发）**：正常使用 `$dual-agent` 默认走此模式，绝不额外出现“讨论需求阶段”。首次触发走 2 步选型弹窗，随后按**「主 Harness 规划 $\rightarrow$ `agy` 确认 $\rightarrow$ 主审核 $\rightarrow$ 人确认（按需） $\rightarrow$ `agy` 执行 $\rightarrow$ 主终验」**动态闭环推进。
   - **专项模式（纯需求沟通讨论）**：仅当用户显式让两个 Agent 沟通讨论需求（不写代码）时触发。**免弹窗选模型**，直接使用**当前终端模型 + `agy` 默认 `gemini-4-hydrogen-xhigh`**；Skill 仅约束 `agy` 启动与物理只读底线，讨论流程完全交由主控自主规划并产出可靠结论。后续同会话若转入写代码开发，仍按规范走 2 步选型弹窗。
3. **升级 `READONLY_SETTINGS` 只读检索权限**：
   - 修复旧版 `EVAL_SETTINGS` 一刀切封禁 `command(*)` 导致 `agy 1.5.9` 无法运行 `rg`/`ls`/`git diff` 检索代码的问题：改用显式 `deny`（`write_file(*)`、`edit_file(*)`、`mcp(*)`、`read_url(*)`）叠加只读命令白名单（`rg`、`grep`、`ls`、`cat`、`head`、`tail`、`wc`、`git status/diff/log/show`）与 `--sandbox`。

---

## [2.1.0] - 2026-11-17 — 确立「两张网叠加」异构双订阅主推架构、`agy-gcp` 降级备选隔离与可行性核验闭环

1. **明确「幻觉错位 / 两张有破洞的网叠加」理论与双订阅主推定位**：
   - 确立日常默认主推 **`Codex + agy` 双订阅协作模式**：不同 Harness（Codex vs Antigravity CLI）与不同厂商模型（`gpt-6.1-*` vs `Gemini 4` / `Claude 5.5`）的思维盲区与疏漏点互不重合（幻觉错位），叠加协作可互相补漏并降低单模型幻觉穿透风险。
   - **`agy-gcp` 降级为备选方案并严格按宿主隔离推送**：仅作为 Codex 不可用时启用 Gemini 4 旗舰（`Carbon` / `Argon`）顶替的备选退路。在 **Codex / Codex CLI（及个人 `agy` / IDE 环境）中绝不主动推送 `agy-gcp`**；**仅当在 `agy-gcp` 环境（`AGY_ADC_AUTH=true`）中使用 Skill 或用户显式点名时才推送 `agy-gcp`**。
2. **Codex 选型向导精简为 2 步直达（取消默认通道二选一弹窗）**：
   - **Step 1**：实时读取 `~/.codex/config.toml` 与 `~/.codex/models_cache.json` 选定主控模型；
   - **Step 2**：自动锁定 `agy` 子通道（不推送 `agy-gcp`）并实时运行 `agy models` 选定子模型后立即开工（`agy-gcp` 宿主下同理锁定 `agy-gcp`）。
   - 同步核验 `agy 1.5.9` 实测结果：`agy-gcp models` 已全面支持 `gemini-4-hydrogen-{high,xhigh,medium,low}`、`gemini-4-carbon-*`、`gemini-4-argon-*` 与 `claude-*-5-5-*`，更新 `README.md` 过时标注。
3. **升级主从 Harness 职责分工：子 Harness 内嵌「可行性验证提意见 + 任务执行」**：
   - 在执行 Prompt 中嵌入单轮内嵌核验契约：子 Harness 动手前先对照真实代码核验主控方案，发现核心假设错误或契约冲突时先停手提意见交主控拍板；可行则直接执行修改与本地编译自检，并在摘要中单列「可行性核验与补充意见」。重大跨服务改造保留独立只读前置评估（`EVAL_SETTINGS`）。
   - 同步对齐全局「默认禁写 `src/test/**` 与禁加测试 `pom.xml` 依赖」规范至仓库版 `dual-agent/SKILL.md`、`AGENTS.md` 与 `README.md`，并重绘 `assets/architecture.png`。

---

## [2.0.0] - 2026-11-13 — Dual-Agent 全新升级自由模式（主控宿主自绑定 + 3 步实时选型 + 独立子会话自由编排）

> 项目由原 1.x 单向固定绑定（`codex-agy`）正式升级为 **`dual-agent` 2.0 全新升级自由模式**，统一兼容 **Codex**、**Antigravity IDE** 与 **AGY CLI**。

1. **「在哪用谁就是主控」宿主自绑定机制（免选主控通道）**：
   - 在 `Codex`、`Antigravity IDE` 或终端 `agy` / `agy-gcp` 中发起对话时，自动锁定当前宿主窗口为唯一主控（`Codex` 与 `Antigravity IDE` 仅做主控、不做子 Agent），直接从**选择主控模型**起步。
2. **同一会话首次触发 3 步串行实时选型弹窗（后续轮次自动复用）**：
   - **Step 1（实时查并选主控模型）**：Codex 实时读取 `~/.codex/config.toml` 与 `~/.codex/models_cache.json`，Antigravity IDE / CLI 直接调用内置 `swarm(action="models", all=true)` 毫秒级获取当前登录态真实模型列表；给出 `(Recommended)` 建议，选择保持当前模型或顺手在顶栏切换主控模型。
   - **Step 2（选子 Agent 通道）**：在无头 CLI `agy`（个人 Google Pro）与 `agy-gcp`（GCP Vertex AI ADC）间自由二选一。
   - **Step 3（实时查并选子 Agent 模型 $\rightarrow$ 立即执行）**：实时跑子通道 `models` 命令，结合主控模型推荐最佳搭档（如旗舰主控配 `gemini-4-hydrogen-high` 1M 主力子会话），手动点选后立即启动全新独立子会话执行，同会话后续任务自动沿用。
3. **跨宿主零轮询调度与同体系异构模型独立双会话**：
   - 补齐 Antigravity IDE / CLI 作主控时的 `run_command` 后台任务反应式自动唤醒模板（与 Codex V8 `while` + `functions.wait` 并列），支持在 IDE 或 `agy` 内实现“旗舰模型主会话定边界审 `git diff` + 主力模型独立子会话跑几百步编码自测”的上下文物理隔离。
   - 同步收录 `Gemini 4 Hydrogen`（1M 上下文 / 1M 输出 Flash 级主力 workhorse）定位与双通道 CLI 白名单实测结论，双端安装至 `~/.codex/skills/dual-agent/` 与 `~/.gemini/config/skills/dual-agent/`。

---

## [2.6.0] - 2026-11-13 — 个人通道升级新一代主力 `gemini-4-hydrogen-high` 与双通道主力差异隔离

1. **实测核验 `Gemini 4 Hydrogen` 官方定位与双通道 CLI 适配差异根因**：
   - **官方定位**：Vertex AI 官方明确定义 `gemini-4-hydrogen` 为 *"high-efficiency Gemini 4 workhorse ... at Flash speed and scale"*（1M 上下文窗口 + Long Decode 最高 1M 输出 token），确认是 `gemini-3.8-flash` 的跨代升级版，并提供 `low` / `medium` / `high` / `xhigh` 四档思考力度。
   - **为什么 Vertex AI 已有模型但 `agy-gcp` 暂不可调（逆向取证结论）**：
     - **个人通道 `agy`（Consumer OAuth）** 通过 `daily-cloudcode-pa.googleapis.com/v1internal:fetchAvailableModels` 服务端动态下发模型表，无需升级 CLI 即可直接使用 `gemini-4-hydrogen-{high,xhigh,medium,low}`（实测 `-p` 返回 `SUCCESS`）。
     - **GCP 通道 `agy-gcp`（`AGY_ADC_AUTH`）** 依赖 `agy` 二进制内置的静态 ADC 模型白名单（当前最新版 `agy 1.5.7` 仅内置到 `gemini-4-carbon-*`，且 `settings.json` 的 `customModels` 仅支持 `gemini-api://` / `blade:` / `evergreen:` / `beyond-api://` 网关 URL，不支持原生 ADC 路由），强传 `--model gemini-4-hydrogen-high` 会在本地 `model_config_manager.go:97` 直接拦截报 `not recognized`。需等 `agy` 下一版本更新客户端白名单（`agy-gcp models` 出现 `gemini-4-hydrogen-*`）后方可启用。
2. **分通道默认主力升级与逐级自动降级链**：
   - **个人通道 `agy`（默认）**：日常默认主力由 `gemini-3.8-flash-high` 升级为 **`gemini-4-hydrogen-high`**（用户点名深度推理可用 `gemini-4-hydrogen-xhigh`）；当遇 Hydrogen 配额不足/耗尽或端点异常时，自动无缝回退至同通道 `gemini-3.8-flash-high`。
   - **GCP 通道 `agy-gcp`**：日常默认主力继续保持 **`gemini-3.8-flash-high`**，严禁在未通过 `agy-gcp models` 确认上线前盲目传入 `gemini-4-hydrogen-*`。
   - **V8 调用模板防呆绑定**：`SKILL.md` 调用模板改为 `const model = channel === "agy-gcp" ? "gemini-3.8-flash-high" : "gemini-4-hydrogen-high"`，从代码层杜绝切换 `agy-gcp` 时误带 `hydrogen` 模型 ID。

---

## [2.5.0] - 2026-11-12 — 双通道（`agy` 个人 / `agy-gcp` GCP ADC）支持、分通道旗舰对齐与 Carbon 超时根因修复

1. **固化 `agy`（个人 Google Pro，默认）与 `agy-gcp`（GCP Vertex AI ADC，按需）双通道切换与环境隔离**：
   - **默认个人通道防串号**：默认使用个人通道 `agy`，调用时强制通过 `env -u AGY_ADC_AUTH -u GOOGLE_APPLICATION_CREDENTIALS -u CLOUDSDK_CORE_PROJECT -u GOOGLE_CLOUD_PROJECT` 剥离父终端残留的 GCP 变量，且每次显式传 `--model` 覆盖共享 `settings.json` 缓存。
   - **非交互 Shell 安全调用 `agy-gcp`**：Codex `tools.exec_command` 默认以 `/bin/zsh -lc`（非交互 Shell）运行，无法直接读取 `~/.zshrc` 中的 `agy-gcp` 函数。`SKILL.md` 固化自适配启动器：优先直调可执行 `agy-gcp`（同步新增免硬编码项目的 [`bin/agy-gcp`](bin/agy-gcp) 包装脚本），未安装脚本时自动回退 `SHELL_SESSIONS_DISABLE=1 /bin/zsh -ic 'agy-gcp "$@"' zsh <参数...>`（位置参数透传杜绝多行提示词与 JSON 二次引号转义损坏，并屏蔽 macOS 交互 Zsh 退出时的 `Saving session...` 输出污染）。
2. **解除单一模型硬编码，建立分通道主力/旗舰模型矩阵**：
   - 实测核对两端 `models` 列表：个人通道 `agy` 仅含 `gemini-4-argon-*`（无 `carbon`），而 GCP 通道 `agy-gcp` 最新一代旗舰为 `gemini-4-carbon-*`（`argon` 已退居次旗舰）。
   - 日常实施、修复、单测、文档默认仍用主力 `gemini-3.8-flash-high`；当用户明确要求使用最新旗舰时，按通道精确对齐（`agy` $\rightarrow$ `gemini-4-argon-medium`，`agy-gcp` $\rightarrow$ `gemini-4-carbon-medium`），不确定模型 ID 时先执行对应通道 `models` 核实，保留“新会话启动旗舰（禁继承 Flash 长上下文）”与“超配额/不可用自动降级”锁。
3. **查明并修复 `agy-gcp` 下 `gemini-4-carbon-low` 90 秒超时空回复问题（禁设短 `--print-timeout`）**：
   - **取证结论**：交叉审计 Codex rollout 与 `cli-20261112_*.log` 证实，`agy -p` 冷启动需经历 ADC 换票（`3~4.5s`）+ 插件 MCP/SessionStart Hook 初始化（`4.6s~18.4s`），而 Vertex AI Global 端点的 `gemini-4-carbon` 首包排队及经本地代理偶发 `unexpected EOF` / `TLS handshake timeout` 自动重试总耗时常达 `75s~160s`。Codex 测试时擅自附加 `--print-timeout 40s / 90s` 导致在流式首包落盘前被 `poll.go:182` 腰斩返回空回复；放宽超时后同命令 `24.1s`（含冷启动全程 `<120s`）正常返回 `SUCCESS`。
   - **规范固化**：严禁在 CLI 命令中附加短 `--print-timeout`（保持默认 `0s`），统一由外层 Codex V8 `while` + `functions.wait` 承接等待。

---

## [2.4.1] - 2026-11-10 — 旗舰模型配额防爆锁与 Flash（High）全面死锁

1. **实施与修复全面锁死 Flash（High）**：
   - 明确 AGY 的“额度充裕”特指拥有 80%+ 每周额度的 `gemini-3.8-flash-high`。所有调用模板与续接返工写死 `--model 'gemini-3.8-flash-high'`。
2. **硬性封锁 Medium 的自发调用（严禁 Codex 擅自升配，配额不足自动降级）**：
   - 复盘证实：Codex 在第 3、4 轮擅自升配 Medium，单任务狂吐 17.1 万深度思考 Token，瞬间将 `gemini-4-argon-medium` 每周配额击穿至 3%。
   - 明文规定：`gemini-4-argon-medium` 属于每周极度稀缺额度，**严禁 Codex 自主决定升配！只有人类用户在对话中明确要求“使用 Medium / 旗舰模型”时方可使用**。
   - **配额不足自动降级**：若用户明确要求使用旗舰 Medium，但遇到配额不足或耗尽报错时，必须立即无缝降级使用 `gemini-3.8-flash-high` 继续执行，不阻塞流程。
3. **禁止将 Flash 长会话喂给旗舰模型（防范 Context 乘积灾难）**：
   - 严禁将 Flash 多步历史会话（如 600+ 步）带入 Medium；即使用户明确授权使用 Medium，也必须开新会话并精炼交接目标，杜绝上下文乘积放大烧光周配额。
4. **微小收尾与拼写校准禁止外包**：
   - 针对纯文档路由拼写、单行常量调整等极小改动，由 Codex 直接修改并验证，严禁调用 AGY。
5. **提供终端彩色监视看板（`bin/agy-watch` 与原始命令）**：
   - 纳入对 `transcript.jsonl` 的 100% 只读、零侵入实时彩色高亮流式监听，支持步骤、命令、读写文件与思考消息语义化着色。
   - `bin/agy-watch` 支持**跨会话自动无缝切换**（自动检测新会话目录并切换追踪，避免一次会话结束后需手动重启），且基于 Python 纯标准库实现，零第三方依赖（无需安装 `jq`）。
   - **新增详细模式（`-v` / `-vv`）**：支持完整展示用户提示词（不截断 80 字符）、工具调用的完整绝对路径与行号范围、完整命令字符串，以及工具执行结果摘要（如退出码、`Showing lines`、文件修改反馈等）。

---

## [2.4.0] - 2026-11-10 — 移除自定义包装脚本，回归 100% 纯原生 `agy` 跨机可移植方案

1. **移除 `bin/agy-delegate` 外部脚本依赖，换机/分享开箱即用**：
   - 撤除自定义包装脚本，回归纯 `SKILL.md` 单文件形态，任何机器只需安装官方 `agy` 并复制 `SKILL.md` 即可直接使用。
2. **原生参数实现跨机免配置完整权限 + 破坏性命令拦截**：
   - 实施与续接命令直接使用 `agy` 原生 `--mode accept-edits --disable-slash-commands --dangerously-skip-permissions` 配合 `--settings '{"permissions":{"deny":["command(git reset)","command(git checkout)","command(git restore)","command(git clean)"]}}'`（`agy` 内核保证 `--dangerously-skip-permissions` 自动放行工作区内外读写与构建命令，同时严格生效显式 `deny` 规则，无需手动修改每台电脑的 `settings.json`）。
3. **保留 Codex V8 内置 `while` 循环 + `functions.wait` 持续等待机制**：
   - 在 `functions.exec` 中通过 `// @exec: {"yield_time_ms": 300000}` 与 `while (res.session_id !== undefined)` 串联 `exec_command`（30s）与 `write_stdin`（300s），5 分钟内 0 次中途唤醒；超过 5 分钟返回 `Script running with cell ID <cell_id>` 时，通过 `functions.wait` 继续等待原 `cell_id`。

---

## [2.3.0 / 2.3.1] - 2026-11-10 — 定位 Codex V8 `exec` 30s 唤醒根因与 `functions.wait` 续等机制

- 排查 `rollout-2026-11-09T09-53-04.jsonl` 的 `additional_tools` 声明与 205 次 `exec` 调用源码，确认 `tools.exec_command` 单次等待物理上限为 `30000 ms`，而 `tools.write_stdin` 空轮询支持 `300000 ms`（5 分钟）；确立 V8 内置 `while` 循环 + 超时后 `functions.wait` 绑定 `cell_id` 的标准等待范式。
- 修正只读评估（`deny: ["write_file(*)", "command(*)", "mcp(*)", "read_url(*)"]`）与 `--project` 省略时归属 `default-cli-project` 的机制说明。

---

## [2.2.0 / 2.2.1] - 2026-11-10 — 彻底去微观管理化（不把 AGY 当傻子：只定目标与边界）

- 移除提示词中对 AGY 工具用法的琐碎说教（如“≤800行一次读完”、“必须传绝对路径”、“核实真实签名”、“日志写到 `target/`”等），回归“委托前定目标与边界 $\rightarrow$ 执行中 AGY 自治闭环 $\rightarrow$ 交付后一次集中终验 $\rightarrow$ 阻塞时定向排障”的四阶段极简契约。

---

## [2.1.0] - 2026-11-10 — 额度重平衡与执行自治架构（解决“Codex 额度耗尽 100%、Gemini 仅耗 6%”）

- **根因总结**：11 月 9 日全天 Codex 共触发 323 次模型调用、累计 8,441 万输入 tokens；其中仅最后一轮“四表实施”就包含 49 次 `yield_time_ms: 30000` 空等待返回，直接触发的下一次模型唤醒累计携带 1,832 万输入 tokens。再加上 Codex 委托前自己做了 76 次代码搜索读取、通过负责人双进程传话催写、用单文件白名单和 `/private/tmp` 日志重定向制造沙箱冲突，导致省下的编码消耗被监工和返工消耗成倍抵消。
- **核心改进**：砍掉外部“AGY 负责人”传话层，放开工作区完整权限与构建自检权限，让 AGY 端到端修到构建全绿再返回。

---

## [2.0.1 / 2.0.2] - 2026-11-10 — 交叉审计底层机制取证

- **`PublicHealthIngestService` 11 处编译错误的真实因果链**：`15:02` AGY 分段读取了 `1–450/580` 行 $\rightarrow$ `15:09–15:15` Codex 仅凭“已读 46 次”误判读完并下令停止全量阅读、立即写入 $\rightarrow$ `15:17` AGY 在缺失 `451–580` 行原文时整体重写该类 $\rightarrow$ `15:24` AGY 解压基线试图补读 `420–580` 行被 `/private/tmp` 权限拒绝 $\rightarrow$ `15:36` 触发 256K 自动压缩。
- **弃用 `--mode plan`**：反编译 `/Users/du/.local/bin/agy` 证实 `--mode plan` 底层自动注入 `/plan` 宏；只读评估改用 `--mode accept-edits --disable-slash-commands` 搭配 `--settings` 物理禁写。
- **脏工作区基线保护**：防止在存在未提交改动时误用 `git checkout/restore` 清空前序实现。
