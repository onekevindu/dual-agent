---
name: dual-agent
description: 全新升级自由模式（2.0版本）主从双会话协作编排 Skill。主控自动绑定当前宿主（Codex / Antigravity IDE / AGY CLI，在哪用谁就是主控，免选主控通道；Codex 与 Antigravity IDE 仅做主控），同一会话首次触发时走 3 步串行弹窗【实时查询并选主控模型 → 选子 Agent 通道 (agy / agy-gcp) → 实时查询并选子模型】，给出推荐建议但由用户手动拍板后立即启动；强制开启独立子会话完成阅读、编码与自测闭环，主控集中审查 git diff 与终验。
---

# Dual-Agent 2.0：主从双会话全新升级自由模式

## 一、 核心机制：在哪用谁就是主控 + 首次触发 3 步实时选型

### 1. 主控自动绑定当前宿主（免选主控通道）

- **主控无需选择，天然绑定当前对话宿主**：
  - 在 **Codex** 中调用 $\rightarrow$ 主控自动锁定为当前 **Codex 会话**；
  - 在 **Antigravity IDE**（含 Antigravity 2.0 桌面端）中调用 $\rightarrow$ 主控自动锁定为当前 **Antigravity IDE 会话**；
  - 在 **终端 `agy` / `agy-gcp` CLI** 中调用 $\rightarrow$ 主控自动锁定为**当前 CLI 会话**。
- **单向角色边界**：
  - **`Codex` 与 `Antigravity IDE` 只做主控、绝不做子 Agent**（不从外部反向调用 Codex CLI 或 IDE）；
  - **子 Agent（Worker）** 仅在无头 CLI **`agy`（个人 Google Pro 通道）** 与 **`agy-gcp`（GCP Vertex AI ADC 通道）** 中自由选择，且**必须开启全新独立会话**（即使主控自身也是 `agy` / `agy-gcp` 或 Antigravity IDE，也通过独立 CLI 子会话实现“同体系不同模型、双会话上下文物理隔离”）。

---

### 2. 同一会话首次触发必走「3 步串行弹窗向导」

- **首次触发强制手动确认**：同一会话**第一次**调用 `$dual-agent` 时，**严禁跳过弹窗直接全量读码或修改业务代码**。必须使用当前宿主原生单选弹窗工具（Codex 使用 `functions.request_user_input_async` / `request_user_input`，Antigravity IDE / AGY CLI 使用 `ask_question`）严格按 **Step 1 $\rightarrow$ Step 2 $\rightarrow$ Step 3** 串行弹出，每步给出 `(Recommended)` 建议，由用户亲手点选后立即开始任务。
- **显式直达与会话级复用**：
  - 若用户在首条消息中已明确说清某一步（例如已明确写出子通道和子模型），跳过已明确步骤、仅弹剩余未确认步骤；
  - 一旦完成本会话首次选定，即持久记住 `{ masterModel, workerChannel, workerModel }`，**本会话后续所有新任务与返工续接自动沿用，不再重复弹窗**（除非用户主动提出“重新选模型 / 切换子 Agent”）。

#### Step 1：实时查询主控可用模型 $\rightarrow$ 弹出【选择主控模型】
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
- **当主控为 Antigravity IDE 或 AGY CLI 时**：直接调用内置工具 `swarm(action="models", all=true)`，毫秒级拿到当前窗口登录态的实时模型 ID 与显示名称。

拿到实时列表后，发起 **Step 1 单选弹窗（选择主控模型）**：
- 首项给出 `(Recommended)` 建议：若当前窗口已选模型即适合当前任务，首推 `(Recommended) 保持当前主控模型（<当前模型名>）— 直接进入下一步`；若任务属于高复杂度跨服务架构设计，提示推荐旗舰（如 Codex `gpt-6.1-sol`、GCP `Gemini 4 Carbon (Medium)`、个人 Pro `Gemini 4 Argon (Medium)`）；常规任务推荐主力（如 `Gemini 4 Hydrogen (High)`）。
- 其余选项列出本次实时查出的核心候选模型（并在问题说明中附上完整实时列表供 `Other` 自由输入）。
- **平滑切换说明**：主控就是当前宿主窗口本身——若用户选择保持当前模型，直接进入 Step 2；若用户点选了其他主控模型，在发起 Step 2 弹窗时顺带提示一句：“请顺手在输入框旁模型选择器（或 `/model`）切换为 `<所选主控模型>`，下一轮起自动按该主控模型运行”，无需额外多停一轮。

#### Step 2：弹出【选择子 Agent 通道】（`agy` vs `agy-gcp`）
发起 **Step 2 单选弹窗**，选择承担重度源码阅读、编码实现与测试闭环的独立 CLI 子通道：
1. `(Recommended) agy — 个人 Google Pro CLI 通道（拥有 1M 上下文/输出的 Gemini 4 Hydrogen 高速主力且额度充裕，编码自测首选）`
2. `agy-gcp — GCP Vertex AI ADC CLI 通道（适合企业合规环境或需调用 GCP 专属模型编码时选用）`

#### Step 3：实时查询子通道可用模型 $\rightarrow$ 弹出【选择子 Agent 模型】 $\rightarrow$ 立即开始执行
先针对 Step 2 选定的 `workerChannel` **实时查询**可用模型清单：

```bash
# 1) 当子通道为 agy（个人 Google Pro）时：
env -u AGY_ADC_AUTH -u GOOGLE_APPLICATION_CREDENTIALS -u CLOUDSDK_CORE_PROJECT -u GOOGLE_CLOUD_PROJECT agy models 2>/dev/null | tr '\r' '\n' | grep -E '^[a-z0-9][a-z0-9._-]+[[:space:]]+'

# 2) 当子通道为 agy-gcp（GCP Vertex AI ADC）时：
if command -v agy-gcp >/dev/null 2>&1; then agy-gcp models; elif [ -x "$HOME/.local/bin/agy-gcp" ]; then "$HOME/.local/bin/agy-gcp" models; else SHELL_SESSIONS_DISABLE=1 /bin/zsh -ic 'agy-gcp models'; fi 2>/dev/null | tr '\r' '\n' | grep -E '^[a-z0-9][a-z0-9._-]+[[:space:]]+'
```

结合 Step 1 已选主控模型，发起 **Step 3 单选弹窗（选择子 Agent 模型）**：
- **当 `workerChannel === "agy"` 时**：首推 `(Recommended) gemini-4-hydrogen-high — Gemini 4 Flash 级 1M 主力 workhorse，高吞吐编码与测试闭环最佳拍档`（尤其主控已选 `Argon` / `Carbon` / `gpt-6.1` 旗舰时，“旗舰主控定边界 + Hydrogen 独立子会话干重活”可防止几百步编码上下文烧穿旗舰周额度）；同列实时存在的 `gemini-4-hydrogen-xhigh`（超复杂逻辑攻坚）、`gemini-3.8-flash-high`（高额度保底）、`gemini-4-argon-medium`、`claude-sonnet-5-5-high` 等。
- **当 `workerChannel === "agy-gcp"` 时**：若实时列表已含 `gemini-4-hydrogen-high` 则首推之，否则首推 `(Recommended) gemini-3.8-flash-high — 当前 GCP CLI 可用的高性价比编码主力`；同列实时存在的 `gemini-4-carbon-low`、`gemini-4-carbon-medium`、`gemini-4-argon-medium`、`claude-sonnet-5-5-high` 等。
- **选完立即启动**：Step 3 返回后**不再追问“是否开始”**，直接打印一行锁定配置摘要：
  `已锁定本会话自由模式配置：主控 = <当前宿主> (<masterModel>) | 子 Agent = <workerChannel> (<workerModel>，独立新会话)`，随即开始执行任务！

---

## 二、 核心协作原则与安全/配额护栏

1. **主从职责分离与双会话物理隔离**：
   - **不要把子 Agent 当傻子**：主控只负责明确**业务目标、必要设计、不可变边界和验收条件**，不替子 Agent 全量预读源码，不在提示词里教子 Agent 怎么调工具。
   - **强制独立新会话**：即使当前主控本身在 Antigravity IDE 或 `agy` CLI 中运行，调用子 Agent 时也**必须开启全新的独立会话**（传 `--project` 而非当前会话 ID），把几百步文件搜索、代码修改与编译报错自修复彻底隔离在子会话上下文内，主控只审 `git diff` 与做最终验收。
   - **生产代码与测试代码始终由子会话修改**：主控遇到返工或环境阻塞时汇总问题交回同一子会话修复，主控绝不亲自下场重写业务代码（仅纯文档拼写或单行常量微调除外）。
2. **双通道环境防串号与非交互 Shell 铁律**：
   - **个人通道 `agy`**：必须通过 `env -u AGY_ADC_AUTH -u GOOGLE_APPLICATION_CREDENTIALS -u CLOUDSDK_CORE_PROJECT -u GOOGLE_CLOUD_PROJECT agy ...` 剥离父进程残留的 GCP 变量，防止误走 GCP 计费。
   - **GCP 通道 `agy-gcp`**：优先直调可执行 `agy-gcp`（`command -v agy-gcp` 或 `$HOME/.local/bin/agy-gcp`），否则回退 `SHELL_SESSIONS_DISABLE=1 /bin/zsh -ic 'agy-gcp "$@"' zsh <参数...>`（位置参数 `"$@"` 透传防止多行提示词与 JSON 被二次转义破坏，并屏蔽 `Saving session...` 污染输出）。
   - **每次 CLI 调用必传 `--model '<精确模型ID>'`**，覆盖共享 `~/.gemini/antigravity-cli/settings.json` 缓存。
3. **锁死亲选模型 & 配额耗尽无缝兜底**：
   - 执行与返工续接期间，**严格锁定用户在 Step 3 亲手选定的 `workerModel`，严禁主控私自升配其他模型**。
   - 仅当所选子模型遇到配额耗尽（Quota exhausted / 429）或端点持续不可用报错时，按同通道降级链无缝兜底并在交付报告中如实说明（`agy`：所选模型 $\rightarrow$ `gemini-4-hydrogen-high` $\rightarrow$ `gemini-3.8-flash-high`；`agy-gcp`：所选模型 $\rightarrow$ 实时可用 `gemini-4-hydrogen-high` 或 `gemini-3.8-flash-high`）。
4. **脏工作区快照保护与禁短超时**：
   - **基线快照保护**：若委托前工作区存在未提交改动，先做好快照备份；子会话调用一律通过 `--settings` 物理拦截 `git reset / checkout / restore / clean`。
   - **严禁附加短 `--print-timeout`（如 `40s`/`90s`）**：保持原生默认 `0s`，避免冷启动（`8s~22s`）及旗舰流式首包排队/网络重试（`75s~160s`）被提前截断。

---

## 三、 双宿主自适应子会话调度模板

### 1. 当主控为 Codex 时：V8 循环 + `functions.wait` 零轮询派发子会话

在 Codex 的 `functions.exec` 工具中运行（5 分钟内完成则 0 次中途唤醒）：

```javascript
// @exec: {"yield_time_ms": 300000, "max_output_tokens": 10000}
const workerChannel = "<Step 2 用户选定的 workerChannel: agy 或 agy-gcp>";
const workerModel = "<Step 3 用户选定的 workerModel>";
const prompt = `目标：<要实现的业务目标与关键技术决策>
不可变边界：<必须保留的既有行为、接口与协议；禁止执行 git checkout/restore/reset/clean 丢弃未提交改动>
验收条件：<构建或测试通过标准>
请自主完成代码修改、测试编写与本地构建自检，修复发现的问题后返回改动摘要。`;
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

### 2. 当主控为 Antigravity IDE 或 AGY CLI 时：`run_command` 反应式唤醒派发子会话

主控直接通过原生 `run_command` 工具（设置 `WaitMsBeforeAsync: 10000`，`IsDaemon: false`）启动**全新独立子会话**（严禁传入主控当前会话 ID）：

```bash
SETTINGS='{"permissions":{"deny":["command(git reset)","command(git checkout)","command(git restore)","command(git clean)"]}}'
PROMPT='目标：<要实现的业务目标与关键技术决策>
不可变边界：<必须保留的既有行为、接口与协议；禁止执行 git checkout/restore/reset/clean 丢弃未提交改动>
验收条件：<构建或测试通过标准>
请自主完成代码修改、测试编写与本地构建自检，修复发现的问题后返回改动摘要。'

# workerChannel === "agy"（个人 Google Pro 通道）：
env -u AGY_ADC_AUTH -u GOOGLE_APPLICATION_CREDENTIALS -u CLOUDSDK_CORE_PROJECT -u GOOGLE_CLOUD_PROJECT \
  agy --project '<项目ID或名称>' --model '<workerModel>' \
  --mode accept-edits --disable-slash-commands --dangerously-skip-permissions \
  --settings "$SETTINGS" --output-format json -p "$PROMPT"

# workerChannel === "agy-gcp"（GCP Vertex AI ADC 通道）：
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

### 3. 集中终验与续接返工

1. 从 `--output-format json` 返回结果中提取并保存子会话 `conversation_id`，同时核查进程退出码、`status`、`stderr` 及 `denied_actions`（不能仅凭退出码 `0` 认定完成）。
2. 主控集中审查 `git diff` 并做独立构建/测试验证；若发现遗漏问题，将派发命令中的 `--project '<项目ID或名称>'` 替换为 `--conversation '<子会话 conversation_id>'`，保持同一 `workerChannel` 与 `workerModel` 一次性汇总打回修复。
3. 若多轮返工后子会话步数过多出现上下文衰减，开启全新独立子会话（传 `--project`）并精炼交接当前基线成果与剩余问题。

---

### 4. 复杂方案单轮只读可行性评估（按需触发）

常规任务直接实施；仅当涉及复杂跨服务契约或状态机改造时，主控可先向子通道派发单轮只读评估（物理禁写文件与一切终端命令，仅使用原生文件读取与内置 `ripgrep` 搜索）：

```bash
EVAL_SETTINGS='{"allowNonWorkspaceAccess":false,"enableTerminalSandbox":true,"toolPermission":"request-review","permissions":{"deny":["write_file(*)","command(*)","mcp(*)","read_url(*)"]}}'
# 按已选 workerChannel 与模型传入：--project '<项目ID或名称>' --model '<workerModel>' --mode accept-edits --disable-slash-commands --sandbox --settings "$EVAL_SETTINGS" --output-format json -p '评估以下方案的可行性，对照现有代码指出潜在冲突、遗漏边界和修正建议：<方案摘要>'
```

---

## 四、 项目归属约定

始终在目标仓库根目录启动子会话 `agy` / `agy-gcp`：
- 优先从 `~/.gemini/config/projects/*.json` 查找 `folderUri` 匹配当前仓库根目录的项目 `id` 传入 `--project '<项目ID>'`；
- 若无匹配项目，可传 `--new-project '<仓库目录名>'` 创建对应项目，或省略 `--project`（省略时会话归属至 `default-cli-project`）。
