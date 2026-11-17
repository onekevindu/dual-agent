# Dual-Agent 2.2 — 双订阅异构双网协作编排

> **核心理念（「幻觉错位 / 两张有破洞的网叠加」理论）**：不同 Harness（Agent 运行框架）与各家模型家族的思维盲区、疏漏点天然不一致。**日常默认主推 `Codex + agy` 双订阅模式**，并将 **`agy` 默认模型全面升级为 `gemini-4-hydrogen-xhigh`**。**`agy-gcp` 降级为 Codex 不可用时的 Gemini 4 旗舰备选方案**，在 Codex / Codex CLI 中**绝不主动推送 `agy-gcp`**，仅在 `agy-gcp` 环境（`AGY_ADC_AUTH=true`）中使用本 Skill 或用户显式点名时才主动推送。

## 两种独立使用模式（各司其职，互不混杂）

1. **默认模式：需求全流程开发模式**（正常使用 `$dual-agent` 的默认行为，不额外插入讨论阶段）
   - **首次 2 步实时选型（同会话自动复用）**：Step 1 实时查选主控模型 $\rightarrow$ Step 2 自动锁定 `agy`（不推 `agy-gcp`）并实时查选子模型（首推 **`gemini-4-hydrogen-xhigh`**）。
   - **动态开发闭环**：**主 Harness 规划 $\rightarrow$ `agy` 确认（可行性核验与提意见） $\rightarrow$ 主 Harness 审核 $\rightarrow$ 人确认（仅需业务/风险拍板时触发） $\rightarrow$ `agy` 执行 $\rightarrow$ 主 Harness 审查 `git diff` 终验**。
2. **专项模式：纯需求沟通讨论模式（免选模型直通）**
   - **触发场景**：仅当用户明确要求两个 Agent **沟通讨论需求 / 评审想法**（而非开发写代码）时启用。
   - **免选模型直通**：**绝不弹窗选模型**，直接使用**当前终端正在使用的模型** + **`agy` 默认 `gemini-4-hydrogen-xhigh`**。
   - **自由规划、仅约束启动与只读底线**：Skill 只约束 `agy` 防串号启动、`READONLY_SETTINGS` 物理只读边界（禁写文件、放行只读代码检索）与零轮询等待；具体讨论几轮、怎样辩论收敛并给出可靠结论，**全权交给主控（如 Codex）自主规划**。若同一会话后续转为写代码开发，仍须走 2 步选型弹窗。

---

## 核心亮点

- **日常主推 `Codex + agy (xhigh)` 双订阅，防越界推送 `agy-gcp`**：
  - 在 **Codex / Codex CLI**（及个人 `agy` / 个人 Antigravity IDE）中使用时，Skill **绝不主动推送 `agy-gcp`**，直接锁定 `agy` 个人 Pro + `gemini-4-hydrogen-xhigh`。
  - 仅当 **Codex 不可用**或直接在 **`agy-gcp` 环境（`AGY_ADC_AUTH=true`）** 中触发 Skill 时，才主动推送 `agy-gcp` 调用 **Gemini 4 旗舰（`Carbon` / `Argon`）** 作为备选方案。
- **动态开发闭环 + 只读检索能力**：
  - 支持将“只读确认”与“代码执行”按需求风险动态组合，且只读配置放行 `rg`/`grep`/`ls`/`cat`/`git diff/log/show` 等只读检索命令，确保 `agy` 基于真实源码核验与研讨，而非靠路径猜测。
- **双宿主零轮询等待**：Codex 下基于 V8 `while` 循环 + `functions.wait` 持续等待，Antigravity IDE / CLI 下基于后台任务反应式自动唤醒，全程零轮询 Token 浪费。

---

## 一键安装（Codex + Antigravity IDE / CLI 全平台生效）

将 [AGENTS.md](AGENTS.md) 中的分工规则合并到目标项目的 `AGENTS.md`。确认本机已安装并登录 `agy` 后，在本仓库根目录执行：

```bash
# 1. 同步安装 Skill 至 Codex 全局目录与 Antigravity (IDE / App / CLI 共享) 全局目录
mkdir -p ~/.codex/skills/dual-agent ~/.gemini/config/skills/dual-agent
cp dual-agent/SKILL.md ~/.codex/skills/dual-agent/SKILL.md
cp dual-agent/SKILL.md ~/.gemini/config/skills/dual-agent/SKILL.md

# 2. （可选推荐）安装备选通道 agy-gcp 包装脚本与实时彩色看板脚本至 PATH
mkdir -p ~/.local/bin
cp bin/agy-gcp ~/.local/bin/agy-gcp
cp bin/agy-watch ~/.local/bin/agy-watch
```

安装后，在 **Codex**（主推）或 **Antigravity IDE / `agy` / `agy-gcp` CLI** 对话中调用 `$dual-agent`：
- [dual-agent/SKILL.md](dual-agent/SKILL.md)：全流程开发与纯需求讨论双模式规范、宿主隔离选型向导、只读/执行双配置调度模板。
- [bin/agy-gcp](bin/agy-gcp)：备选通道 GCP Vertex AI ADC 独立可执行包装器（自动解析 `~/.gemini/credentials/gcp-adc.json` 或标准 gcloud ADC 凭证与项目 ID）。
- [bin/agy-watch](bin/agy-watch)：支持跨会话自动热切换的终端实时彩色执行大盘。
- [CHANGELOG.md](CHANGELOG.md)：问题排查取证与版本演进记录。

---

## 推荐搭配与模式速览

| 定位 / 模式 | 所在宿主（主 Harness） | 主 Harness 模型 | 独立子 Harness & 默认模型 | 核心特性 |
| :--- | :--- | :--- | :--- | :--- |
| **默认开发模式（主推）** | **Codex / Codex CLI**（不推 `agy-gcp`） | 首次 2 步弹窗选定（如 `gpt-6.1-sol`） | **`agy`** + **`gemini-4-hydrogen-xhigh`** | 「规划 $\rightarrow$ `agy` 确认 $\rightarrow$ 主审 $\rightarrow$ 人确认 $\rightarrow$ 执行」动态闭环 |
| **纯需求沟通讨论模式** | **Codex / Codex CLI**（不推 `agy-gcp`） | **免选直通**（当前终端正在使用的模型） | **`agy`** + **`gemini-4-hydrogen-xhigh`**（物理只读） | **不弹窗选模型**，Skill 仅约束只读启动，讨论流程交由主控自主规划 |
| **同体系隔离** | **Antigravity IDE / 个人 `agy` CLI** | `Gemini 4 Argon (Medium)` 或 `Hydrogen (XHigh)` | **`agy`** + **`gemini-4-hydrogen-xhigh`**（或 `claude-sonnet-5-5-high`） | 旗舰大脑把关、数百步编码隔离在独立 CLI 子会话，不污染主会话 |
| **备选降级（仅 `agy-gcp` 内或点名推送）** | **`agy-gcp` CLI**（`AGY_ADC_AUTH=true`） | `gemini-4-carbon-medium` 或 `gemini-4-argon-medium` | **`agy-gcp`** + **`gemini-4-hydrogen-xhigh`**（或 `claude-sonnet-5-5-high`） | Codex 不可用时的 Gemini 4 旗舰退路，实测 `agy 1.5.9` 已全量支持 |

---

## 架构与执行流程

![Dual-Agent 2.2 双订阅异构双网与双模式协作架构](assets/architecture.png)

[查看完整尺寸 PNG](assets/architecture.png)

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
