---
name: dual-agent
description: 主从双 Harness 异构协作编排 Skill（基于「幻觉错位 / 两张有破洞的网叠加」原理）。日常默认主推 Codex + agy 双订阅模式：在 Codex / Codex CLI 或个人环境中绝不主动推送 agy-gcp，首次触发精简为 2 步实时选型（选主控模型 → 锁定 agy 并选子模型）；agy-gcp 降级为 Codex 不可用时的 Gemini 4 旗舰备选方案，仅当在 agy-gcp 环境使用本 Skill（或用户显式点名）时才主动推送。主 Harness 负责规划、定边界、审 git diff 与终验；独立子 Harness 负责可行性验证与意见提出、任务执行与本地编译自检闭环。
---

# Dual-Agent 2.1：双订阅异构双网协作编排

## 一、 核心理念：幻觉错位与「两张有破洞的网叠加」

### 1. 为什么需要两家工具叠加协作
- **「幻觉错位」与「两张有破洞的网叠加」理论**：任何单一模型与单一 Harness（运行框架与工具链）都有固有的盲区、思维惯性与疏漏点，就像一张有破洞的网。不同 Harness（如 OpenAI Codex vs Antigravity CLI）与不同厂商模型家族（如 `gpt-6.1-*` vs `Gemini 4` / `Claude 5.5`）的疏漏点**天然错位、互不重合**。让两套不同体系叠加协作，可使漏洞相互遮盖，大幅压低单模型幻觉穿透与漏检风险。
- **主从职责清晰互补**：
  - **主 Harness（规划与终验）**：负责需求拆解、架构决策、明确目标与不可变边界、集中审查 `git diff` 及最终验收；
  - **子 Harness（可行性验证提意见 + 任务执行）**：在独立新会话中对照真实代码库验证主控方案可行性、提出异议或修正建议，并自主完成代码实现与本地编译自检闭环。

### 2. 日常主推方案与 `agy-gcp` 备选隔离规则
- **日常默认主推：`Codex + agy` 双订阅模式**
  - **主 Harness**：**Codex（桌面端 / Codex CLI）**（如 `gpt-6.1-sol` 等），专注架构把关与终验；
  - **子 Harness**：**`agy`（个人 Google Pro 无头 CLI）**（如 1M 上下文/输出主力 `gemini-4-hydrogen-high`），承担可行性核验、意见反馈与重度编码闭环。既满足跨 Harness/跨厂商互补，又全程走固定双订阅额度，零额外按量账单。
- **`agy-gcp` 降级定位（仅作备选）**：
  - **`agy-gcp`（GCP Vertex AI ADC）** 降级为 **Codex 不可用时的备选方案**——当 Codex 不可用（额度耗尽、网络/账号受限）或处于 GCP 合规环境时，通过 `agy-gcp` 调用 **Gemini 4 旗舰（如 `gemini-4-carbon-medium` / `gemini-4-argon-medium`）** 顶替原 Codex 主控位置，配合独立子会话延续双网隔离。
- **严格按宿主隔离推送（严禁越界推销 `agy-gcp`）**：
  - **在 Codex / Codex CLI（及个人 `agy` CLI / 个人登录态 Antigravity IDE）中使用本 Skill 时**：**绝不主动推送 `agy-gcp`**（不列进默认选型弹窗），直接绑定 `agy` 作为子通道（仅用户主动在对话中明确点名要求 `agy-gcp` 时才切换）。
  - **只有在 `agy-gcp` 环境中使用本 Skill 时**（检测到环境变量 `AGY_ADC_AUTH=true` 等 GCP ADC 激活态）：才主动推送 `agy-gcp` 通道及其旗舰/主力模型组合。
- **主控自绑定 & 单向角色边界**：
  - 在哪个宿主窗口调用，当前宿主即为主控；**`Codex` 与 `Antigravity IDE` 只做主控、绝不做子 Agent**（不从外部反向调用 Codex CLI 或 IDE）。
  - **子 Harness 始终以全新独立 CLI 会话运行**（传 `--project` 而非当前会话 ID），把数百步读码、改码与报错修复隔离在子会话上下文内。

---

## 二、 同一会话首次触发：宿主自适应「2 步实时选型向导」

- **首次触发强制手动确认**：同一会话**第一次**调用 `$dual-agent` 时，**严禁跳过选型弹窗直接全量读码或修改业务代码**。必须使用宿主原生单选弹窗（Codex 使用 `functions.request_user_input_async` / `request_user_input`，Antigravity IDE / CLI 使用 `ask_question`）按 **Step 1 $\rightarrow$ Step 2** 串行弹出，每步给出 `(Recommended)` 建议，由用户点选确认后立即启动。
- **显式直达与会话级复用**：
  - 若用户在首条消息中已明确指定某一步的模型（或显式点名使用 `agy-gcp`），跳过已明确步骤、仅弹剩余未确认步骤；
  - 首次选定后持久记住 `{ masterHarness, masterModel, workerChannel, workerModel }`，**本会话后续所有新任务与返工续接自动沿用，不再重复弹窗**（除非用户主动要求“重新选模型 / 切换子 Agent”）。

### Step 1：实时查询主控可用模型 $\rightarrow$ 弹出【选择主控模型】
**严禁凭静态记忆硬编模型列表**，先实时获取当前主控宿主的真实可用模型：

- **当主控为 Codex 时**：运行以下轻量脚本实时读取当前激活模型与本地模型缓存：
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
- **当主控为 Antigravity IDE 或 `agy` / `agy-gcp` CLI 时**：直接调用内置工具 `swarm(action="models", all=true)`，毫秒级拿到当前登录态的实时模型 ID 与显示名称。

拿到实时列表后，发起 **Step 1 单选弹窗（选择主控模型）**：
- 首项给出 `(Recommended)` 建议：若当前窗口已选模型适合当前任务，首推 `(Recommended) 保持当前主控模型（<当前模型名>）— 直接进入下一步`；高复杂度跨服务规划场景可推荐对应宿主的旗舰模型（Codex 推荐 `gpt-6.1-sol`；`agy-gcp` 备选环境推荐 `Gemini 4 Carbon (Medium)`；个人 Pro IDE / `agy` 推荐 `Gemini 4 Argon (Medium)` 或 `Gemini 4 Hydrogen (High/XHigh)`）。
- 其余选项列出本次实时查出的核心候选模型（并在问题描述中附上完整实时列表供 `Other` 自由输入）。
- **平滑切换说明**：若用户点选了其他主控模型，在发起 Step 2 弹窗时顺带提示一句：“请顺手在输入框旁模型选择器（或 `/model`）切换为 `<所选主控模型>`，下一轮起自动按该主控模型运行”，无需额外多停一轮。

### Step 2：按宿主自动锁定子通道 $\rightarrow$ 实时查选【子 Harness 模型】 $\rightarrow$ 立即执行

**子通道自动判定规则（不再多弹一步通道选择，绝不主动推销 `agy-gcp`）**：
- **在 Codex / Codex CLI、Antigravity IDE（个人态）或个人 `agy` CLI 中**：自动锁定 `workerChannel = "agy"`（**绝不主动推送 `agy-gcp`**；仅当用户已在指令中显式点名 `agy-gcp` 时才改为 `"agy-gcp"`）。
- **仅当当前宿主处于 `agy-gcp` 环境时**（如环境变量 `AGY_ADC_AUTH=true` 已激活）：自动锁定 `workerChannel = "agy-gcp"`，主动推送 `agy-gcp` 模型列表。

按锁定的 `workerChannel` **实时查询**子通道可用模型清单：

```bash
# 1) 默认个人通道 agy（Codex / 个人环境默认锁定此分支）：
env -u AGY_ADC_AUTH -u GOOGLE_APPLICATION_CREDENTIALS -u CLOUDSDK_CORE_PROJECT -u GOOGLE_CLOUD_PROJECT agy models 2>/dev/null | tr '\r' '\n' | grep -E '^[a-z0-9][a-z0-9._-]+[[:space:]]+'

# 2) 仅当宿主为 agy-gcp（AGY_ADC_AUTH=true）或用户显式点名 agy-gcp 时执行：
if command -v agy-gcp >/dev/null 2>&1; then agy-gcp models; elif [ -x "$HOME/.local/bin/agy-gcp" ]; then "$HOME/.local/bin/agy-gcp" models; else SHELL_SESSIONS_DISABLE=1 /bin/zsh -ic 'agy-gcp models'; fi 2>/dev/null | tr '\r' '\n' | grep -E '^[a-z0-9][a-z0-9._-]+[[:space:]]+'
```

结合 Step 1 主控模型，发起 **Step 2 单选弹窗（选择子 Harness 模型）**：
- **当 `workerChannel === "agy"` 时**：
  - 首推 `(Recommended) gemini-4-hydrogen-high — 1M 上下文/输出主力 workhorse，与 Codex 形成跨 Harness 异构双网互补，双订阅额度充裕`；
  - 同列实时存在的 `gemini-4-hydrogen-xhigh`（超复杂逻辑与深度可行性校验）、`gemini-3.8-flash-high`（高额度保底）、`claude-sonnet-5-5-high`（若主控同为 Gemini 体系，推荐选跨厂商 Claude 以实现模型家族错位互补）、`gemini-4-argon-medium` 等。
- **当 `workerChannel === "agy-gcp"` 时（仅 `agy-gcp` 宿主或用户显式要求触发）**：
  - 若实时列表已含 `gemini-4-hydrogen-high`（实测 `agy 1.5.9` 已支持）则首推之，否则首推 `(Recommended) gemini-3.8-flash-high`；
  - 若当前 `agy-gcp` 主控已选 `Gemini 4 Carbon (Medium)`，同步标注 `claude-sonnet-5-5-high` 可实现跨厂商模型家族互补（“两张网错位”），同列实时存在的 `gemini-4-hydrogen-xhigh`、`gemini-4-carbon-low`、`gemini-4-argon-medium` 等。
- **选完立即启动**：Step 2 返回后**不再追问“是否开始”**，直接打印一行锁定配置摘要：
  `已锁定本会话协同配置：主 Harness = <当前宿主> (<masterModel>) | 子 Harness = <workerChannel> (<workerModel>，独立新会话)`，随即开始执行任务！

---

## 三、 核心协作原则与安全/配额护栏

1. **「两张网」职责分离与双会话物理隔离**：
   - **主控定目标与边界，不做微观管理**：主 Harness 只负责明确**业务目标、必要设计、不可变边界和验收条件**，不替子 Harness 全量预读源码，不在提示词里教子 Harness 怎么调工具。
   - **子 Harness 承担「可行性验证提意见 + 任务执行」**：
     - **先核验再实施**：子会话动手改码前先对照真实代码核验主控方案。若发现**核心假设错误、接口/契约冲突或重大设计隐患**导致方案不可行，必须先停手输出可行性异议与修正建议，交主控拍板；若方案可行但存在局部边界风险或优化空间，正常执行并在交付摘要中单列**「可行性核验与补充意见」**。
     - **强制独立新会话**：无论主控在 Codex、IDE 还是 CLI，子会话均**必须开启全新独立会话**（传 `--project` 而非主控会话 ID），把数百步源码阅读、修改与编译报错修复彻底隔离在子会话上下文内。
   - **默认禁写测试代码与禁改测试 POM 依赖**：除非用户明确要求编写单元测试，否则主从均严禁新建 `src/test/**` 测试类或在 `pom.xml` 中擅自新增 `test` 依赖；验收以 `mvn compile` 编译校验、SQL/表结构对齐检查或运行态验证为主。
   - **生产代码始终由子会话修改**：主控集中审查 `git diff` 与独立验证，发现问题一次性汇总交回同一子会话修复，主控绝不亲自下场重写业务代码（仅纯文档拼写或单行常量微调除外）。
2. **双通道环境防串号与非交互 Shell 铁律**：
   - **个人通道 `agy`（默认）**：必须通过 `env -u AGY_ADC_AUTH -u GOOGLE_APPLICATION_CREDENTIALS -u CLOUDSDK_CORE_PROJECT -u GOOGLE_CLOUD_PROJECT agy ...` 剥离父进程残留的 GCP 变量，防止误走 GCP 计费。
   - **GCP 通道 `agy-gcp`（备选/GCP 宿主）**：优先直调可执行 `agy-gcp`（`command -v agy-gcp` 或 `$HOME/.local/bin/agy-gcp`），否则回退 `SHELL_SESSIONS_DISABLE=1 /bin/zsh -ic 'agy-gcp "$@"' zsh <参数...>`（位置参数 `"$@"` 透传防止多行提示词与 JSON 被二次转义破坏，并屏蔽 `Saving session...` 污染输出）。
   - **每次 CLI 调用必传 `--model '<精确模型ID>'`**，覆盖共享 `~/.gemini/antigravity-cli/settings.json` 缓存。
3. **锁死亲选模型 & 配额耗尽无缝兜底**：
   - 执行与返工续接期间，**严格锁定用户在 Step 2 亲手选定的 `workerModel`，严禁主控私自升配其他模型**。
   - 仅当所选子模型遇到配额耗尽（Quota exhausted / 429）或端点持续不可用报错时，按同通道降级链无缝兜底并在交付报告中如实说明（`agy`：所选模型 $\rightarrow$ `gemini-4-hydrogen-high` $\rightarrow$ `gemini-3.8-flash-high`；`agy-gcp`：所选模型 $\rightarrow$ `gemini-4-hydrogen-high` $\rightarrow$ `gemini-3.8-flash-high`）。
4. **脏工作区快照保护与禁短超时**：
   - **基线快照保护**：若委托前工作区存在未提交改动，先做好快照备份；子会话调用一律通过 `--settings` 物理拦截 `git reset / checkout / restore / clean`。
   - **严禁附加短 `--print-timeout`（如 `40s`/`90s`）**：保持原生默认 `0s`，避免冷启动（`8s~22s`）及旗舰流式首包排队/网络重试（`75s~160s`）被提前截断。

---

## 四、 双宿主自适应子会话调度模板

### 1. 当主控为 Codex 时：V8 循环 + `functions.wait` 零轮询派发子会话

在 Codex 的 `functions.exec` 工具中运行（日常默认锁定 `workerChannel = "agy"`，**绝不主动推送 `agy-gcp`**；5 分钟内完成则 0 次中途唤醒）：

```javascript
// @exec: {"yield_time_ms": 300000, "max_output_tokens": 10000}
const workerChannel = "agy"; // Codex 默认锁定 "agy" 双订阅；仅用户显式点名备选时才改为 "agy-gcp"
const workerModel = "<Step 2 用户选定的 workerModel>";
const prompt = `目标：<要实现的业务目标与关键技术决策>
不可变边界：<必须保留的既有行为、接口与协议；禁止执行 git checkout/restore/reset/clean 丢弃未提交改动；严禁未经用户明确要求新增 src/test/** 测试代码或在 pom.xml 中新增 test 依赖>
验收条件：<mvn compile 编译检查或代码自检通过标准>
执行与核验要求：
1. 先对照真实源码核验上述方案的可行性：若发现核心假设错误、接口契约冲突或重大遗漏导致不宜直接改动，立即停手并输出可行性异议与修正建议；
2. 若方案可行，自主完成代码修改与本地编译自检（默认禁写单测、禁加测试 pom 依赖），修复发现的问题后返回改动摘要，并单列「可行性核验与补充意见」（边界提醒、潜在风险或优化建议）。`;
const settings = JSON.stringify({
  permissions: {
    deny: [
      "command(git reset)",
      "command(git checkout)",
      "command(git restore)",
      "command(git clean)"
    ]
  }
});
const q = s => "'" + s.replaceAll("'", "'\\''") + "'";
const runner = workerChannel === "agy-gcp"
  ? `if command -v agy-gcp >/dev/null 2>&1; then agy-gcp "$@"; elif [ -x "$HOME/.local/bin/agy-gcp" ]; then "$HOME/.local/bin/agy-gcp" "$@"; else SHELL_SESSIONS_DISABLE=1 /bin/zsh -ic 'agy-gcp "$@"' zsh "$@"; fi`
  : `env -u AGY_ADC_AUTH -u GOOGLE_APPLICATION_CREDENTIALS -u CLOUDSDK_CORE_PROJECT -u GOOGLE_CLOUD_PROJECT agy "$@"`;
const args = [
  "--project", "<项目ID或名称>",
  "--model", workerModel,
  "--mode", "accept-edits",
  "--disable-slash-commands",
  "--dangerously-skip-permissions",
  "--settings", settings,
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

### 2. 当主控为 Antigravity IDE / AGY CLI / `agy-gcp` 时：`run_command` 反应式唤醒派发子会话

主控直接通过原生 `run_command` 工具（设置 `WaitMsBeforeAsync: 10000`，`IsDaemon: false`）启动**全新独立子会话**（严禁传入主控当前会话 ID）：

```bash
SETTINGS='{"permissions":{"deny":["command(git reset)","command(git checkout)","command(git restore)","command(git clean)"]}}'
PROMPT='目标：<要实现的业务目标与关键技术决策>
不可变边界：<必须保留的既有行为、接口与协议；禁止执行 git checkout/restore/reset/clean 丢弃未提交改动；严禁未经用户明确要求新增 src/test/** 测试代码或在 pom.xml 中新增 test 依赖>
验收条件：<mvn compile 编译检查或代码自检通过标准>
执行与核验要求：
1. 先对照真实源码核验上述方案的可行性：若发现核心假设错误、接口契约冲突或重大遗漏导致不宜直接改动，立即停手并输出可行性异议与修正建议；
2. 若方案可行，自主完成代码修改与本地编译自检（默认禁写单测、禁加测试 pom 依赖），修复发现的问题后返回改动摘要，并单列「可行性核验与补充意见」（边界提醒、潜在风险或优化建议）。'

# 默认个人通道 agy（Antigravity IDE 个人态 / agy CLI 默认走此分支）：
env -u AGY_ADC_AUTH -u GOOGLE_APPLICATION_CREDENTIALS -u CLOUDSDK_CORE_PROJECT -u GOOGLE_CLOUD_PROJECT \
  agy --project '<项目ID或名称>' --model '<workerModel>' \
  --mode accept-edits --disable-slash-commands --dangerously-skip-permissions \
  --settings "$SETTINGS" --output-format json -p "$PROMPT"

# 仅当当前宿主就是 agy-gcp（AGY_ADC_AUTH=true）或用户显式点名 agy-gcp 备选时使用：
if command -v agy-gcp >/dev/null 2>&1; then
  agy-gcp --project '<项目ID或名称>' --model '<workerModel>' --mode accept-edits --disable-slash-commands --dangerously-skip-permissions --settings "$SETTINGS" --output-format json -p "$PROMPT"
elif [ -x "$HOME/.local/bin/agy-gcp" ]; then
  "$HOME/.local/bin/agy-gcp" --project '<项目ID或名称>' --model '<workerModel>' --mode accept-edits --disable-slash-commands --dangerously-skip-permissions --settings "$SETTINGS" --output-format json -p "$PROMPT"
else
  SHELL_SESSIONS_DISABLE=1 /bin/zsh -ic 'agy-gcp "$@"' zsh --project '<项目ID或名称>' --model '<workerModel>' --mode accept-edits --disable-slash-commands --dangerously-skip-permissions --settings "$SETTINGS" --output-format json -p "$PROMPT"
fi
```

- **反应式零轮询等待铁律**：命令运行满 10 秒自动转为后台任务后，主控立即向用户简短同步一句“已派发独立子会话执行，等待完成唤醒”并**直接结束当前轮次（不调用更多工具）**；严禁使用 `sleep` 或循环调 `manage_task` 轮询，子会话结束时系统会自动唤醒主控并交付完整 JSON 输出。

---

### 3. 集中终验、意见处理与续接返工

1. 从 `--output-format json` 返回结果中提取并保存子会话 `conversation_id`，同时核查进程退出码、`status`、`stderr` 及 `denied_actions`（不能仅凭退出码 `0` 认定完成）。
2. **先审「可行性异议 / 补充意见」再审 `git diff`**：
   - 若子会话因发现根本性冲突而停手提出可行性异议，主控先基于真实反馈修订方案（必要时向用户确认决策），再将派发命令中的 `--project '<项目ID或名称>'` 替换为 `--conversation '<子会话 conversation_id>'` 下发修订方案继续执行；
   - 若子会话已完成修改，主控审阅其「可行性核验与补充意见」，并集中审查 `git diff` 与执行独立编译/运行态验证；发现遗漏问题同样通过 `--conversation` 保持同一 `workerChannel` 与 `workerModel` 一次性汇总打回修复。
3. 若多轮返工后子会话步数过多出现上下文衰减，开启全新独立子会话（传 `--project`）并精炼交接当前基线成果与剩余问题。

---

### 4. 重大架构方案独立只读前置评估（按需触发）

常规任务直接按上文单轮内嵌核验模式派发；仅当涉及跨服务契约重构、核心状态机改造或高风险数据库迁移时，主控可先向子 Harness 派发一轮**物理只读**可行性评估，拿回验证意见与边界清单后再拍板派发实施：

```bash
EVAL_SETTINGS='{"allowNonWorkspaceAccess":false,"enableTerminalSandbox":true,"toolPermission":"request-review","permissions":{"deny":["write_file(*)","command(*)","mcp(*)","read_url(*)"]}}'
# 按已锁定 workerChannel 与模型传入：--project '<项目ID或名称>' --model '<workerModel>' --mode accept-edits --disable-slash-commands --sandbox --settings "$EVAL_SETTINGS" --output-format json -p '请对照现有代码做只读可行性验证：指出以下方案的潜在冲突、遗漏边界、隐含风险及修正建议：<方案摘要>'
```

---

## 五、 项目归属约定

始终在目标仓库根目录启动子会话 `agy` / `agy-gcp`：
- 优先从 `~/.gemini/config/projects/*.json` 查找 `folderUri` 匹配当前仓库根目录的项目 `id` 传入 `--project '<项目ID>'`；
- 若无匹配项目，可传 `--new-project '<仓库目录名>'` 创建对应项目，或省略 `--project`（省略时会话归属至 `default-cli-project`）。
