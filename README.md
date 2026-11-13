# Dual-Agent 2.0 — 全新升级自由模式

> **全新升级自由模式（2.0 版本）**：打破固定主从与写死模型的束缚——**在哪用谁就是主控**（支持 `Codex`、`Antigravity IDE`、`AGY CLI` 任意主控），同一会话首次触发时通过 **3 步实时模型弹窗**自由选定主从拍档，开启**全新独立子会话**（`agy` 个人 Pro 或 `agy-gcp` GCP ADC）完成阅读、编码与自测闭环，将架构决策与数百步编码上下文彻底物理隔离。

## 核心亮点

- **主控宿主自绑定（免选主控）**：在 `Codex`、`Antigravity IDE` 或终端 `agy` / `agy-gcp` 中发起对话，当前窗口自动锁定为主控（`Codex` 与 `Antigravity IDE` 仅作主控、不做被调子 Agent）。
- **会话首次 3 步实时选型（后续自动复用）**：
  1. **Step 1 选主控模型**：实时拉取当前主控真实可用模型列表并给出 `(Recommended)` 建议；
  2. **Step 2 选子 Agent 通道**：在无头 CLI `agy`（个人 Google Pro）与 `agy-gcp`（GCP Vertex AI ADC）间自由二选一；
  3. **Step 3 选子 Agent 模型**：实时执行子通道 `models` 查询，结合主控模型推荐最佳编码拍档，点选后**立即开跑**，同一会话后续任务自动沿用。
- **同体系异构模型独立双会话**：支持在 Antigravity IDE 或 `agy` CLI 中用旗舰（`Carbon Medium` / `Argon Medium`）当大脑看需求与审 `git diff`，同时拉起全新独立子会话用 1M 主力（`gemini-4-hydrogen-high` / `gemini-3.8-flash-high`）干重活，既享受旗舰智商又绝不烧穿周配额。
- **双宿主零轮询等待**：Codex 下基于 V8 `while` 循环 + `functions.wait` 持续等待，Antigravity IDE / CLI 下基于后台任务反应式自动唤醒，全程零轮询 Token 浪费。

---

## 一键安装（Codex + Antigravity IDE / CLI 全平台生效）

将 [AGENTS.md](AGENTS.md) 中的分工规则合并到目标项目的 `AGENTS.md`。确认本机已安装并登录 `agy` 后，在本仓库根目录执行：

```bash
# 1. 同步安装 Skill 至 Codex 全局目录与 Antigravity (IDE / App / CLI 共享) 全局目录
mkdir -p ~/.codex/skills/dual-agent ~/.gemini/config/skills/dual-agent
cp dual-agent/SKILL.md ~/.codex/skills/dual-agent/SKILL.md
cp dual-agent/SKILL.md ~/.gemini/config/skills/dual-agent/SKILL.md

# 2. （可选推荐）安装 GCP ADC 独立包装脚本与实时彩色看板脚本至 PATH
mkdir -p ~/.local/bin
cp bin/agy-gcp ~/.local/bin/agy-gcp
cp bin/agy-watch ~/.local/bin/agy-watch
```

安装后，在 **Codex** 或 **Antigravity IDE / CLI** 对话中调用 `$dual-agent`（或说明使用 `dual-agent` 自由模式）即可弹出选型向导开始协作：
- [dual-agent/SKILL.md](dual-agent/SKILL.md)：3 步实时选型向导、双通道环境防串号与双宿主子会话调度模板。
- [bin/agy-gcp](bin/agy-gcp)：GCP Vertex AI ADC 独立可执行包装器（自动解析 `~/.gemini/credentials/gcp-adc.json` 或标准 gcloud ADC 凭证与项目 ID）。
- [bin/agy-watch](bin/agy-watch)：支持跨会话自动热切换的终端实时彩色执行大盘。
- [CHANGELOG.md](CHANGELOG.md)：问题排查取证与版本演进记录。

---

## 推荐自由组合速览

| 所在宿主（自动锁定为主控） | 主控推荐模型（定边界 / 审 `git diff`） | 独立子会话通道 & 推荐模型（读码 / 编码 / 自测） | 场景优势 |
| :--- | :--- | :--- | :--- |
| **Codex**（只做主控） | 当前窗口模型（如 `gpt-6.1-sol`） | `agy` + `gemini-4-hydrogen-high` | 零主控切换开销，用个人 Pro 充裕的 1M Hydrogen 主力包揽重度读写与单测 |
| **Antigravity IDE**（只做主控） | `Gemini 4 Carbon (Medium)` 或 `Argon (Medium)` | `agy` + `gemini-4-hydrogen-high` | IDE 可视化审阅 + 旗舰大脑把关，数百步编码隔离在独立 CLI 子会话，不污染 IDE 主会话上下文 |
| **终端 `agy` CLI** | `gemini-4-argon-medium` 或 `hydrogen-high` | `agy` + `gemini-4-hydrogen-high` | 纯个人 Pro 异构双会话：旗舰仅看几千 Token 边界与 `git diff`，彻底告别旗舰周配额击穿 |
| **Antigravity IDE / `agy-gcp`** | `gemini-4-carbon-medium` | `agy-gcp` + `gemini-3.8-flash-high`（CLI 更新白名单后可切 `hydrogen-high`） | 全程运行在企业 GCP Vertex AI 合规边界内，主从独立双会话隔离 |

---

## 架构与执行流程（2.0 自由模式）

![Dual-Agent 2.0 自由模式架构及执行流程](assets/architecture.png)

[查看完整尺寸 PNG](assets/architecture.png)

### 协作四原则

1. **委托前**：检查脏工作区快照备份，主控讲清楚**业务目标、必要设计、不可变边界和验收条件**，不对子 Agent 做过度限制或手把手指导。复杂方案可按需派发单轮只读评估。
2. **执行中**：直派全新独立子会话（**禁设短 `--print-timeout`**），由子会话自主完成阅读、编码、运行测试与局部纠错；主控零轮询静默等待完成唤醒。
3. **交付后**：主控核对返回 JSON（`conversation_id`、`status`、`denied_actions`），集中审查 `git diff` 并执行独立验证，把遗漏问题一次性交回同一子会话修复。
4. **出现阻塞时**：针对具体阻塞定向处理，主控绝不亲自下场重写业务代码。

---

## 实时查看子会话执行状态（可选彩色大盘）

若想在另一个终端窗口实时观察独立子会话正在调用的工具、修改的文件与思考过程（100% 只读旁路，零侵入）：

### 方式 1：执行仓库内置脚本（推荐，支持跨会话自动热切换与详细模式）
`bin/agy-watch` 基于 Python 纯标准库实现，无需安装 `jq`，**新子会话拉起时自动无缝切换追踪**：
```bash
./bin/agy-watch        # 默认紧凑看板模式（防刷屏）
./bin/agy-watch -v     # 详细模式（不截断长指令，显示完整文件路径、全量命令与执行结果摘要）
./bin/agy-watch -vv    # 全量详细模式（包含命令输出全文）
```

### 方式 2：终端原始单行命令（单会话静态跟踪，免安装）
```bash
tail -n 30 -f "$(ls -t ~/.gemini/antigravity-cli/brain/*/.system_generated/logs/transcript.jsonl | head -n 1)" | jq --unbuffered -r '
  def c_dim(s): "\u001b[90m" + s + "\u001b[0m";
  def c_step(s): "\u001b[1;33m" + s + "\u001b[0m";
  def c_cmd(s): "\u001b[1;32m" + s + "\u001b[0m";
  def c_edit(s): "\u001b[1;35m" + s + "\u001b[0m";
  def c_read(s): "\u001b[1;36m" + s + "\u001b[0m";
  def c_tool(s): "\u001b[1;34m" + s + "\u001b[0m";
  def c_arg(s): "\u001b[37m" + s + "\u001b[0m";
  def c_msg(s): "\u001b[96m" + s + "\u001b[0m";
  def c_user(s): "\u001b[1;45;97m" + s + "\u001b[0m";

  if .type == "USER_INPUT" then
    c_dim("[\(.created_at[11:19])]") + " " + c_user(" 🚀 USER ") + " " + c_msg((.content // "") | gsub("\n"; " ") | .[0:80])
  elif .type == "PLANNER_RESPONSE" then
    c_dim("[\(.created_at[11:19])]") + " " + c_step("#\(.step_index)") + " " +
    (if (.tool_calls | length) > 0 then
      ([.tool_calls[] |
        (if .name == "run_command" then c_cmd(.name)
         elif (.name == "edit_file" or .name == "write_file") then c_edit(.name)
         elif .name == "read_file" then c_read(.name)
         else c_tool(.name) end) +
        "(" + c_arg(
          if .args.Path then (.args.Path | gsub("^\"|\"$"; "") | split("/") | last)
          elif .args.CommandLine then (.args.CommandLine | gsub("^\"|\"$"; "") | .[0:60])
          else (.args.Query // .args.SearchPath // "" | gsub("^\"|\"$"; ""))
          end
        ) + ")"
      ] | join(", "))
    else
      "\u001b[35m💬\u001b[0m " + c_msg((.content // "") | gsub("\n"; " ") | .[0:80])
    end)
  else empty end
'
```
