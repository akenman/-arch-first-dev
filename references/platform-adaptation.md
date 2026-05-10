# 平台适配

> 加载条件：非 DeepSeek TUI 环境，或工具调用失败需要查表时

## 平台适配

本 skill 使用行为描述而非具体工具名，跨平台通用。AI 执行时根据当前环境选择对应工具：

| 行为 | DeepSeek TUI | Claude Code | Trae CN | Cursor | Codex CLI |
|------|-------------|-------------|---------|--------|-----------|
| 列出目录内容 | list_dir | LS / Bash ls | list_dir | list_dir | Bash ls |
| 搜索代码 | grep_files | Grep | grep_files | search | Grep |
| 读取文件 | read_file | Read | read_file | read_file | Read |
| 写入/编辑文件 | write_file / edit_file | Write / Edit | write_file / edit_file | edit_file | Write |
| 运行命令 | exec_shell | Bash | exec_shell | terminal | Bash |
| 跟踪进度 | checklist_write | TodoWrite | checklist_write | —（跳过） | —（跳过） |
| 规划步骤 | update_plan | —（跳过） | update_plan | —（跳过） | —（跳过） |
| 启动子任务 | agent_spawn | Task | —（转串行） | —（转串行） | —（转串行） |
| 压缩上下文 | /compact | /compact | —（释放模块代码替代） | /compact | —（释放模块代码替代） |

工具不可用时回退策略：
- 进度/计划工具不可用 → 用代码注释或文件记录替代
- 子任务不可用 → 串行填充，同层模块依次实现
- 压缩上下文不可用 → 每个模块 [done] 后立即释放实现代码