# 从主控启动独立 agy

在目标仓库根目录运行。个人通道的查询、首次启动和续接都用同一份隔离环境。以下示例使用 Bash 或 zsh 数组，参数逐项传递。

```bash
runner=(env -u AGY_ADC_AUTH -u GOOGLE_APPLICATION_CREDENTIALS -u CLOUDSDK_CORE_PROJECT -u GOOGLE_CLOUD_PROJECT agy)
"${runner[@]}" models
```

仅在当前为 GCP 宿主（`AGY_ADC_AUTH=true`）或用户点名时使用项目的 `agy-gcp` 包装器替换 runner（可执行文件在 PATH 或 ~/.local/bin；仅有 shell 函数时用交互 zsh 调用）。选择当前通道实际列出的完整模型 ID，包括 Orchestrator 的具体变体。每次启动和续接均传入所选 ID。配额耗尽或端点持续失败时，仅按原有同通道降级约定使用可用的 Hydrogen High、Flash High，并告知用户；用户明确要求固定某模型时报告阻塞。

用 ~/.gemini/config/projects/*.json 的 folderUri 对齐仓库并取得项目 ID。新会话用 --project，未注册项目可用默认项目或 --new-project；返工用输出中的 conversation_id 配合 --conversation。勿续接主控自己的 ID。

```bash
target=(--project "$project_id")   # 续接时用 target=(--conversation "$conversation_id")
exec_settings='{"permissions":{"deny":["command(git reset)","command(git checkout)","command(git restore)","command(git clean)"]}}'
readonly_settings='{"allowNonWorkspaceAccess":false,"enableTerminalSandbox":true,"toolPermission":"request-review","permissions":{"allow":["command(rg)","command(grep)","command(ls)","command(cat)","command(head)","command(tail)","command(wc)","command(git status)","command(git diff)","command(git log)","command(git show)"],"deny":["write_file(*)","edit_file(*)","mcp(*)","read_url(*)"]}}'

# 二选一，将选出的 flags 用在下方启动命令中
flags=(--sandbox --settings "$readonly_settings")     # 只读核验或讨论
# flags=(--mode accept-edits --settings "$exec_settings")  # 已授权的开发执行
"${runner[@]}" "${target[@]}" --model "$worker_model" "${flags[@]}" --output-format json -p "$prompt"
```

执行前检查未提交改动，有改动先备份基线。只读会话仅允许只读检索，子任务也须遵守相同限制。开发会话沿用宿主授权，遇权限拒绝先检查结果中的 denied_actions。保留 print-timeout 默认值。普通派工直接传任务正文。主控已经选定且确认 print 入口支持某条命令时，把它放在任务正文之前，例如蜂群任务可在调用前设置 `prompt="/swarm $prompt"`。保持命令展开开启，并从执行事件或结果核对是否启动。将引用的用户文本或日志置于任务说明之后。

用 `agy --help` 查参数，`agy -p '/help'` 查 print 模式可直接回答的查询命令，`agy changelog` 可按具体功能检索。交互菜单、只读查询、可展开的技能及模型工具各有入口。只有确实需要该能力时才核对相应入口。

Codex 用 exec_command 承接进程，有 session_id 就以 write_stdin 等待同一进程；若外层 functions.exec 已返回 cell_id，则用 functions.wait 续等该 cell。agy 宿主可用原生后台任务的完成通知。等待期间可以处理独立问题，避免反复读取完整日志。最后检查退出码、JSON 状态、拒绝的操作、现场变更和验收结果。
