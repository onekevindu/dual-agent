# Dual-Agent 3.0

在 Codex 与 agy 中使用同一份 Skill。双订阅开发沿用 Codex 主控、agy 执行的分工；单宿主工作直接沿用所在会话的模型、工具和上下文。

## 使用示例

| 场景 | 可以这样说 |
| --- | --- |
| Codex 主控、agy 执行 | `$dual-agent 用 Orchestrator (Neon) 排查订单偶现卡住的问题。用线程栈验证原因，再修复。` |
| 只用 Codex | `$dual-agent 只用 Codex，定位这次构建失败并验证修复。` |
| 只用 agy（在 agy 中输入） | `/dual-agent 只用 agy，优化这段算法，用同一组输入比较前后耗时。` |
| 只读交叉评审 | `$dual-agent 让双方评审这个迁移方案，找出会影响上线的分歧。只讨论。` |

双会话首次开发按主控、执行方依次选模型，已有选择直接沿用。主控按需决定是否显式启用专门能力，执行方自行选择日常检索、编辑和验证方法。详细约定见 [Skill 入口](dual-agent/SKILL.md)。

例如，排查偶现故障时，Codex 若判断互相矛盾的两段证据值得独立推演，可以显式启用 `/boost`，把待解释的矛盾交给 agy。如果一次线程栈对照已经能证实原因，agy 可以直接验证修复。类似地，`/swarm` 的价值来自多个执行者确实需要共享进展，每个成员也会消耗配额。两者都是可选手段。

开工前可以按需要澄清需求或制订计划（例如 agy 的 `/grill-me`、`/plan`）。在 Codex CLI，你可以用 `/btw` 临时旁问，或用 `/goal` 明确希望持续追踪的完成条件。交互指令和模型工具的可用性按当前宿主确认。[Codex CLI 命令说明](https://developers.openai.com/codex/cli/slash-commands/)

## 安装

从仓库根目录将 Skill 复制到需要使用的宿主。只用一个宿主时只执行对应行。

```bash
mkdir -p ~/.codex/skills && cp -R dual-agent ~/.codex/skills/
mkdir -p ~/.gemini/config/skills && cp -R dual-agent ~/.gemini/config/skills/
```

需要项目级约定时，将 [AGENTS.md](AGENTS.md) 的协作约定合并进目标项目。启动外部 agy 需要已登录的 CLI。[无头调用说明](dual-agent/references/agy-launch.md) 包含选型、沙箱和会话续接示例。历史版本见 [CHANGELOG.md](CHANGELOG.md)。

可选的终端看板和 GCP 包装器可从仓库直接执行，或复制到 PATH。

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
