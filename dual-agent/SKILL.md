---
name: dual-agent
description: 主从双 Harness 异构协作编排 Skill（基于「幻觉错位 / 两张有破洞的网叠加」原理）。日常默认主推 Codex + agy 双订阅模式（agy 默认模型锁定 gemini-4-hydrogen-xhigh；Codex 与个人环境中绝不主动推送 agy-gcp，仅在 agy-gcp 环境或用户显式点名时推送）。提供两种独立场景模式：(1) 默认「需求全流程开发模式」：首次触发走 2 步实时选型，随后按「主 Harness 规划 → agy 确认 → 主审核 → 人确认（按需） → agy 执行 → 主终验」动态协作推进，正常开发不插入讨论阶段；(2) 「纯需求沟通讨论模式」（仅用户显式让两个 Agent 沟通讨论需求时启用）：免选模型直通（当前终端模型 + agy 默认 gemini-4-hydrogen-xhigh），Skill 仅约束只读安全边界与 agy 启动方式，讨论流程完全交给主控自主规划。
---

# Dual-Agent 2.2：双订阅异构双网协作编排

## 一、 核心理念与双模式分流

### 1. 幻觉错位与「两张有破洞的网叠加」理论
- 任何单一模型与单一 Harness（运行框架与工具链）都有固有的盲区、思维惯性与疏漏点，如同各自带有破洞的网。
- 不同 Harness（如 OpenAI Codex vs Antigravity CLI）与不同厂商模型家族（如 `gpt-6.1-*` vs `Gemini 4` / `Claude 5.5`）的疏漏点**天然错位、互不重合**。让两套不同体系叠加协作，可相互遮盖盲区，显著压低单模型幻觉穿透与漏检风险。

### 2. 日常主推方案与 `agy-gcp` 备选隔离规则
- **日常默认主推：`Codex + agy` 双订阅模式**
  - **主 Harness**：**Codex（桌面端 / Codex CLI）**（如 `gpt-6.1-sol` 等），负责规划拆解、审核与终验；
  - **子 Harness**：**`agy`（个人 Google Pro 无头 CLI）**，默认主力模型统一锁定为 **`gemini-4-hydrogen-xhigh`**（1M 上下文/输出，深度思考档），兼顾跨 Harness/跨厂商互补与固定双订阅成本优势。
- **`agy-gcp` 降级定位（仅作备选）**：
  - **`agy-gcp`（GCP Vertex AI ADC）** 降级为 **Codex 不可用时的备选方案**——当 Codex 不可用或处于 GCP 合规环境时，通过 `agy-gcp` 调用 **Gemini 4 旗舰（如 `gemini-4-carbon-medium` / `gemini-4-argon-medium`）** 顶替原 Codex 主控大脑，配合独立子会话延续双网隔离。
- **严格按宿主隔离推送（严禁越界推销 `agy-gcp`）**：
  - **在 Codex / Codex CLI（及个人 `agy` CLI / 个人登录态 Antigravity IDE）中使用本 Skill 时**：**绝不主动推送 `agy-gcp`**（不列进默认弹窗），直接绑定 `agy`（仅用户主动点名要求 `agy-gcp` 时才切换）。
  - **只有在 `agy-gcp` 环境中使用本 Skill 时**（检测到环境变量 `AGY_ADC_AUTH=true` 等 GCP ADC 激活态）：才主动推送 `agy-gcp` 通道及其旗舰/主力模型组合。
- **主控自绑定 & 单向角色边界**：
  - 在哪个宿主窗口调用，当前宿主即为主控；**`Codex` 与 `Antigravity IDE` 只做主控、绝不做子 Agent**。子 Harness 始终以**全新独立 CLI 会话**运行（传 `--project` 而非当前会话 ID）。

### 3. 两种独立使用模式（严禁混为一谈）

| 模式 | 触发场景 | 是否弹窗选模型 | Skill 约束重点 |
| :--- | :--- | :--- | :--- |
| **默认模式：需求全流程开发模式** | **正常开发默认走此模式**（实现需求、改代码、修 Bug、重构等；**绝不出现独立的“讨论需求阶段”**） | **首次触发走 2 步弹窗**（同会话后续自动沿用） | 2 步选型 + **「主 Harness 规划 $\rightarrow$ `agy` 确认 $\rightarrow$ 主审核 $\rightarrow$ 人确认（按需） $\rightarrow$ `agy` 执行 $\rightarrow$ 主终验」**动态闭环 |
| **专项模式：纯需求沟通讨论模式** | **仅当用户显式要求让两个 Agent 沟通讨论需求 / 评审想法 / 技术推演**（不写代码）时触发 | **免选模型直通（绝不弹窗）**：当前终端模型 + `agy` 默认 `gemini-4-hydrogen-xhigh` | **仅约束 `agy` 启动方式与只读护栏**，讨论流程完全交给主控（如 Codex）自主规划并给出可靠结论 |

---

## 二、 默认模式（需求全流程开发）：2 步选型与动态开发闭环

### 1. 同一会话首次触发：宿主自适应「2 步实时选型向导」

- **首次触发强制手动确认**：进入需求开发（写代码）阶段且本会话尚未完成过 2 步选型时，**严禁跳过弹窗直接全量读码或改业务代码**（注：即便此前同会话曾跑过“纯需求讨论模式”，进入实际写代码阶段仍须走一次 2 步选型确认）。使用原生单选弹窗（Codex 用 `functions.request_user_input_async` / `request_user_input`，Antigravity IDE / CLI 用 `ask_question`）按 **Step 1 $\rightarrow$ Step 2** 串行弹出。
- **显式直达与会话级复用**：若用户首条指令已明确模型或显式点名 `agy-gcp`，跳过已明确步骤；首次选定后持久记住 `{ masterHarness, masterModel, workerChannel, workerModel }`，**本会话后续开发与返工自动沿用，不再重复弹窗**。

#### Step 1：实时查询主控可用模型 $\rightarrow$ 弹出【选择主控模型】
- **当主控为 Codex 时**：运行以下脚本实时读取当前激活模型与本地模型缓存：
  ```bash
  python3 - <<'PY'
  import json, os, re
  cfg = os.path.expanduser("~/.codex/config.toml")
  cache = os.path.expanduser("~/.codex/models_cache.json")
  cur_m, cur_e = "default", "high"
  if os.path.isfile(cfg):
      t = open(cfg, encoding="utf-8", errors="ignore").read()
      m = re.search(r'(?m)^model\s*=\s*"([^"]+)"', t)
      e = re.search(r'(?m)^model_reasoning_effort\s*=\s*"([^"]+)"', t)
      if m: cur_m = m.group(1)
      if e: cur_e = e.group(1)
  print(f"CURRENT={cur_m} ({cur_e})")
  if os.path.isfile(cache):
      for x in json.load(open(cache, encoding="utf-8")).get("models", []):
          if isinstance(x, dict) and (x.get("slug") or x.get("id")):
              effs = ",".join(i.get("effort","") for i in x.get("supported_reasoning_levels",[]) if isinstance(i,dict))
              print(f"{x.get('slug') or x.get('id')}\t{x.get('display_name','')}\t[{effs}]")
  PY
  ```
- **当主控为 Antigravity IDE 或 `agy` / `agy-gcp` CLI 时**：直接调用内置工具 `swarm(action="models", all=true)`。
- 发起单选弹窗：首推 `(Recommended) 保持当前主控模型（<当前模型名>）`（或高复杂度规划推荐对应旗舰：`gpt-6.1-sol` / `agy-gcp` 下 `Gemini 4 Carbon (Medium)` / 个人 Pro `Gemini 4 Argon (Medium)`），并列实时候选；若选其他模型，在 Step 2 弹窗时顺带提示用户切顶栏 / `/model`。

#### Step 2：按宿主自动锁定子通道 $\rightarrow$ 实时查选【子 Harness 模型】 $\rightarrow$ 立即启动
- **子通道自动判定（绝不主动推送 `agy-gcp`）**：
  - **Codex / Codex CLI / 个人 `agy` CLI / 个人 Antigravity IDE**：自动锁定 `workerChannel = "agy"`（仅用户明确点名 `agy-gcp` 时才切）。
  - **仅当宿主自身处于 `agy-gcp` 环境时**（如 `AGY_ADC_AUTH=true`）：自动锁定 `workerChannel = "agy-gcp"`，主动推送 `agy-gcp` 模型列表。
- **实时查询所锁定子通道可用模型**：
  ```bash
  # 默认个人通道 agy：
  env -u AGY_ADC_AUTH -u GOOGLE_APPLICATION_CREDENTIALS -u CLOUDSDK_CORE_PROJECT -u GOOGLE_CLOUD_PROJECT agy models 2>/dev/null | tr '\r' '\n' | grep -E '^[a-z0-9][a-z0-9._-]+[[:space:]]+'

  # 仅当宿主为 agy-gcp（AGY_ADC_AUTH=true）或用户显式点名 agy-gcp 时执行：
  if command -v agy-gcp >/dev/null 2>&1; then agy-gcp models; elif [ -x "$HOME/.local/bin/agy-gcp" ]; then "$HOME/.local/bin/agy-gcp" models; else SHELL_SESSIONS_DISABLE=1 /bin/zsh -ic 'agy-gcp models'; fi 2>/dev/null | tr '\r' '\n' | grep -E '^[a-z0-9][a-z0-9._-]+[[:space:]]+'
  ```
- 发起 Step 2 单选弹窗：
  - **首推 `(Recommended) gemini-4-hydrogen-xhigh`**：作为 `agy`（及 `agy-gcp` 实测 `agy 1.5.9` 已支持）默认主力模型，1M 上下文/输出 + `xhigh` 深度推理，兼顾方案核验深度与强编码闭环；
  - 同列实时存在的 `gemini-4-hydrogen-high`（高速降级备选）、`gemini-3.8-flash-high`（高额度保底）、`claude-sonnet-5-5-high`（当主控同为 Gemini 时可选跨厂商互补）、`gemini-4-argon-medium` 等。
- **选完立即启动**：不再追问“是否开始”，打印一行配置摘要并立即启动开发流程：
  `已锁定本会话协同配置：主 Harness = <当前宿主> (<masterModel>) | 子 Harness = <workerChannel> (<workerModel>，独立新会话)`

---

### 2. 动态开发协作流程：「主 Harness 规划 $\rightarrow$ `agy` 确认 $\rightarrow$ 主审核 $\rightarrow$ 人确认 $\rightarrow$ `agy` 执行」

正常需求开发直接进入以下动态协作闭环，**不额外插入独立的“需求讨论阶段”**：

1. **主 Harness 规划**：检查工作区未提交改动快照，主控拆解需求并制定技术方案，明确**业务目标、必要设计、不可变边界与验收条件**（不替子 Harness 全量预读源码，不在提示词里教怎么调工具）。
2. **`agy` 确认（可行性核验与意见提出）**：
   - **前置只读确认路径**（涉及多模块联动、跨服务契约、状态机/库表变更或存在不确定边界时优先采用）：使用只读沙箱配置（`READONLY_SETTINGS`）拉起独立子会话，让 `agy` 对照真实代码核验主控规划的可行性，暴露潜在冲突、隐含假设与修正建议。
   - **内嵌核验合并路径**（契约清晰、风险可控的常规任务）：直接派发执行 Prompt，并在 Prompt 中要求**动手改码前先核验方案可行性**——无根本冲突则直接执行并在摘要单列「可行性核验与补充意见」，若发现重大冲突则停手先回报异议，动态转入下一步主审核。
3. **主 Harness 审核**：主控审核 `agy` 返回的可行性确认结果与异议意见，综合两张网交叉校验修订最终技术方案。
4. **人确认（按需动态触发）**：当审核后发现**业务语义/产品取舍待决、重大架构/边界变更、高风险不可逆操作或 `agy` 提出重大可行性异议**时，主控汇总方案与意见交用户拍板确认；若双方核验完全对齐、边界无歧义且在既定授权范围内，直接推进执行，不对用户做多余二次打扰。
5. **`agy` 执行**：沿用原确认子会话（`--conversation '<conversation_id>'`）或新开独立子会话（`--project`）完成源码修改与本地编译自检（默认严禁新增 `src/test/**` 或在 `pom.xml` 中新增 `test` 依赖）。
6. **主 Harness 审查 `git diff` 与终验**：核对 `--output-format json` 返回的 `conversation_id`、`status`、`stderr` 与 `denied_actions`，集中审查 `git diff` 并做独立编译/运行态验证；遇遗漏通过 `--conversation` 一次性交回同一子会话修复，主控绝不亲自下场重写业务代码（仅纯文档拼写或单行常量微调除外）。

---

## 三、 专项模式（纯需求沟通讨论）：免选模型直通与启动约束

### 1. 适用边界与免选直通
- **触发条件**：仅当用户明确要求两个 Agent **沟通讨论需求、评审方案思路、做技术选型推演或排查可行性**（不修改代码）时触发。正常开发全流程任务绝不误触发此模式。
- **免选模型直通（严禁弹窗）**：
  - **主 Harness**：直接使用**当前终端/宿主正在使用的模型**；
  - **子 Harness**：在 Codex / 个人环境中**绝不主动推送 `agy-gcp`**，直接使用 `agy` 默认模型 **`gemini-4-hydrogen-xhigh`**（仅当当前宿主自身是 `agy-gcp` 时才用 `agy-gcp` 的 `gemini-4-hydrogen-xhigh`）。
- **不绑定后续写代码选型**：讨论阶段结束后，若用户在同一会话后续要求“按方案开始写代码”，且本会话尚未走过第二节的 2 步选型弹窗，仍须走 2 步选型弹窗确认开发主从模型。

### 2. Skill 只约束启动与只读底线，不限制讨论流程
**讨论需求的具体流程（沟通几轮、如何拆解议题、从什么角度追问反驳、如何收敛并向用户呈现可靠结论）完全交给主 Harness（如 Codex）自主规划**，Skill 不把讨论过程写死成固定步骤，仅约束以下 4 条启动与安全红线：
1. **防串号与默认模型锁定**：`agy` 必须通过 `env -u ...` 剥离 GCP ADC 环境变量并显式传入 `--model 'gemini-4-hydrogen-xhigh'`（覆盖共享 `settings.json` 缓存；仅当遇 429/不可用时自动降级 `gemini-4-hydrogen-high` $\rightarrow$ `gemini-3.8-flash-high`）。
2. **物理只读安全边界**：统一使用下方 `READONLY_SETTINGS`（不加 `--dangerously-skip-permissions`），物理禁止 `write_file(*)` / `edit_file(*)` / `mcp(*)` / `read_url(*)`，同时在沙箱内放行 `read_file` 与只读检索命令（`rg`、`grep`、`ls`、`cat`、`head`、`tail`、`wc`、`git status/diff/log/show`），保证子 Harness 能基于真实代码事实参与讨论。
3. **零轮询等待机制与禁短超时**：Codex 下使用 V8 `while` + `functions.wait`，IDE / CLI 下使用 `run_command` 反应式唤醒；严禁附加短 `--print-timeout`（保持默认 `0s`）。
4. **独立会话与多轮续聊**：首轮讨论传 `--project '<项目ID或名称>'` 开启独立子会话，保留返回的 `conversation_id`；主控需要进一步追问、辩论或对齐细节时，直接将 `--project` 替换为 `--conversation '<conversation_id>'` 续聊。

---

## 四、 安全/配额护栏与双宿主调度模板

### 通用硬性护栏
1. **默认禁写单测与禁改测试 POM 依赖**：除非用户明确要求编写单元测试，否则主从均严禁新增 `src/test/**` 或在 `pom.xml` 中擅自新增 `test` 依赖；验收以 `mvn compile` 编译校验、SQL/表结构对齐检查或运行态验证为主。
2. **锁死模型 & 同通道降级兜底**：开发模式下严格锁定 Step 2 亲选的 `workerModel`，讨论模式下锁定 `gemini-4-hydrogen-xhigh`，**严禁主控私自升配其他高耗配额模型（如 `Argon` / `Carbon`）**。仅当遇配额耗尽（429）或端点持续不可用时按同通道降级链（`gemini-4-hydrogen-xhigh` $\rightarrow$ `gemini-4-hydrogen-high` $\rightarrow$ `gemini-3.8-flash-high`）兜底并告知用户。
3. **双通道防串号 & 禁短超时**：`agy` 必用 `env -u AGY_ADC_AUTH -u GOOGLE_APPLICATION_CREDENTIALS -u CLOUDSDK_CORE_PROJECT -u GOOGLE_CLOUD_PROJECT`；`agy-gcp` 优先直调可执行 `agy-gcp`，否则回退 `SHELL_SESSIONS_DISABLE=1 /bin/zsh -ic 'agy-gcp "$@"' zsh`；每次调用必传 `--model`；严禁附加短 `--print-timeout`。
4. **基线快照保护**：开发执行前如有未提交改动须先做快照备份；执行配置通过 `--settings` 物理拦截 `git reset / checkout / restore / clean`。

---

### 1. 当主控为 Codex 时：V8 循环 + `functions.wait` 零轮询调度模板

在 Codex 的 `functions.exec` 工具中运行（5 分钟内完成则 0 次中途唤醒）：

```javascript
// @exec: {"yield_time_ms": 300000, "max_output_tokens": 10000}
const mode = "exec"; // "exec"（代码执行）| "readonly"（agy只读确认 / 纯需求讨论模式）
const workerChannel = "agy"; // Codex 默认锁定 "agy" 双订阅，绝不主动推送 "agy-gcp"；仅用户显式点名时改 "agy-gcp"
const workerModel = "gemini-4-hydrogen-xhigh"; // 开发模式填 Step 2 所选模型（默认推荐 gemini-4-hydrogen-xhigh）；讨论模式固定 gemini-4-hydrogen-xhigh

const prompt = mode === "readonly"
  ? `<由 Codex 自主规划的只读确认或需求讨论议题：明确背景、疑问、边界与期望反馈>`
  : `目标：<要实现的业务目标与关键技术决策>
不可变边界：<必须保留的既有行为、接口与协议；禁止执行 git checkout/restore/reset/clean 丢弃未提交改动；严禁未经用户明确要求新增 src/test/** 测试代码或在 pom.xml 中新增 test 依赖>
验收条件：<mvn compile 编译检查或代码自检通过标准>
执行与核验要求：
1. 动手改码前先对照真实源码核验方案可行性：若发现核心假设错误、接口契约冲突或重大遗漏导致不宜直接改动，立即停手并输出可行性异议与修正建议；
2. 若方案可行，自主完成代码修改与本地编译自检（默认禁写单测、禁加测试 pom 依赖），修复发现的问题后返回改动摘要，并单列「可行性核验与补充意见」。`;

const EXEC_SETTINGS = JSON.stringify({
  permissions: {
    deny: ["command(git reset)", "command(git checkout)", "command(git restore)", "command(git clean)"]
  }
});
const READONLY_SETTINGS = JSON.stringify({
  allowNonWorkspaceAccess: false,
  enableTerminalSandbox: true,
  toolPermission: "request-review",
  permissions: {
    allow: [
      "command(rg)", "command(grep)", "command(ls)", "command(cat)",
      "command(head)", "command(tail)", "command(wc)",
      "command(git status)", "command(git diff)", "command(git log)", "command(git show)"
    ],
    deny: ["write_file(*)", "edit_file(*)", "mcp(*)", "read_url(*)"]
  }
});

const q = s => "'" + s.replaceAll("'", "'\\''") + "'";
const runner = workerChannel === "agy-gcp"
  ? `if command -v agy-gcp >/dev/null 2>&1; then agy-gcp "$@"; elif [ -x "$HOME/.local/bin/agy-gcp" ]; then "$HOME/.local/bin/agy-gcp" "$@"; else SHELL_SESSIONS_DISABLE=1 /bin/zsh -ic 'agy-gcp "$@"' zsh "$@"; fi`
  : `env -u AGY_ADC_AUTH -u GOOGLE_APPLICATION_CREDENTIALS -u CLOUDSDK_CORE_PROJECT -u GOOGLE_CLOUD_PROJECT agy "$@"`;

const sessionTarget = ["--project", "<项目ID或名称>"]; // 续接确认/多轮讨论/返工时改为 ["--conversation", "<conversation_id>"]
const modeFlags = mode === "readonly"
  ? ["--sandbox", "--settings", READONLY_SETTINGS]
  : ["--dangerously-skip-permissions", "--settings", EXEC_SETTINGS];

const args = [
  ...sessionTarget,
  "--model", workerModel,
  "--mode", "accept-edits",
  "--disable-slash-commands",
  ...modeFlags,
  "--output-format", "json",
  "-p", prompt
].map(q).join(" ");

let res = await tools.exec_command({
  cmd: `sh -c ${q(runner)} sh ${args}`,
  yield_time_ms: 30000,
  max_output_tokens: 10000
});
let out = res.output || "";
while (res.session_id !== undefined) {
  res = await tools.write_stdin({
    session_id: res.session_id,
    chars: "",
    yield_time_ms: 300000,
    max_output_tokens: 10000
  });
  out += res.output || "";
}
text(out);
```

- **超 5 分钟纯续等铁律**：当 `functions.exec` 返回 `Script running with cell ID <cell_id>` 时，原 V8 循环仍在运行，**唯一允许的操作是直接调用 `functions.wait({ "cell_id": "<cell_id>", "yield_time_ms": 300000, "max_tokens": 10000 })`**，严禁另开新的 `functions.exec` 调 `write_stdin` 或顺手跑任何检查命令。

---

### 2. 当主控为 Antigravity IDE / AGY CLI / `agy-gcp` 时：`run_command` 反应式唤醒模板

主控通过原生 `run_command`（`WaitMsBeforeAsync: 10000`，`IsDaemon: false`）启动**全新独立子会话**（严禁传入主控当前会话 ID）：

```bash
EXEC_SETTINGS='{"permissions":{"deny":["command(git reset)","command(git checkout)","command(git restore)","command(git clean)"]}}'
READONLY_SETTINGS='{"allowNonWorkspaceAccess":false,"enableTerminalSandbox":true,"toolPermission":"request-review","permissions":{"allow":["command(rg)","command(grep)","command(ls)","command(cat)","command(head)","command(tail)","command(wc)","command(git status)","command(git diff)","command(git log)","command(git show)"],"deny":["write_file(*)","edit_file(*)","mcp(*)","read_url(*)"]}}'

# [A] 默认个人通道 agy — 1) 只读确认 / 纯需求讨论模式（去掉 --dangerously-skip-permissions，默认模型 gemini-4-hydrogen-xhigh）：
env -u AGY_ADC_AUTH -u GOOGLE_APPLICATION_CREDENTIALS -u CLOUDSDK_CORE_PROJECT -u GOOGLE_CLOUD_PROJECT \
  agy --project '<项目ID或名称>' --model 'gemini-4-hydrogen-xhigh' \
  --mode accept-edits --disable-slash-commands --sandbox \
  --settings "$READONLY_SETTINGS" --output-format json -p '<由主控规划的确认/讨论提示词>'

# [A] 默认个人通道 agy — 2) 代码执行模式（续接可改 --project 为 --conversation '<id>'）：
env -u AGY_ADC_AUTH -u GOOGLE_APPLICATION_CREDENTIALS -u CLOUDSDK_CORE_PROJECT -u GOOGLE_CLOUD_PROJECT \
  agy --project '<项目ID或名称>' --model '<workerModel>' \
  --mode accept-edits --disable-slash-commands --dangerously-skip-permissions \
  --settings "$EXEC_SETTINGS" --output-format json -p "$PROMPT"

# [B] 备选通道 agy-gcp（仅当当前宿主就是 agy-gcp [AGY_ADC_AUTH=true] 或用户显式点名时按需运行）：
if command -v agy-gcp >/dev/null 2>&1; then
  agy-gcp --project '<项目ID或名称>' --model '<workerModel>' --mode accept-edits --disable-slash-commands --dangerously-skip-permissions --settings "$EXEC_SETTINGS" --output-format json -p "$PROMPT"
elif [ -x "$HOME/.local/bin/agy-gcp" ]; then
  "$HOME/.local/bin/agy-gcp" --project '<项目ID或名称>' --model '<workerModel>' --mode accept-edits --disable-slash-commands --dangerously-skip-permissions --settings "$EXEC_SETTINGS" --output-format json -p "$PROMPT"
else
  SHELL_SESSIONS_DISABLE=1 /bin/zsh -ic 'agy-gcp "$@"' zsh --project '<项目ID或名称>' --model '<workerModel>' --mode accept-edits --disable-slash-commands --dangerously-skip-permissions --settings "$EXEC_SETTINGS" --output-format json -p "$PROMPT"
fi
# （注：在 agy-gcp 下跑只读确认/讨论时，同样将 --dangerously-skip-permissions --settings "$EXEC_SETTINGS" 替换为 --sandbox --settings "$READONLY_SETTINGS"）
```

- **反应式零轮询等待铁律**：命令运行满 10 秒自动转为后台任务后，主控立即向用户简短同步一句“已派发独立子会话执行，等待完成唤醒”并**直接结束当前轮次（不调用更多工具）**；严禁 `sleep` 或循环调 `manage_task` 轮询。

---

## 五、 项目归属约定

始终在目标仓库根目录启动子会话 `agy` / `agy-gcp`：
- 优先从 `~/.gemini/config/projects/*.json` 查找 `folderUri` 匹配当前仓库根目录的项目 `id` 传入 `--project '<项目ID>'`；
- 若无匹配项目，可传 `--new-project '<仓库目录名>'` 创建对应项目，或省略 `--project`（省略时会话归属至 `default-cli-project`）。
