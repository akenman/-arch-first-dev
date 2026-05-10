# arch-first-dev

让 AI 先画架构蓝图再写代码，而不是上来就直接生成一堆文件。

核心思路很简单：AI 在写任何代码之前，先把整个项目的模块、接口、依赖关系画在 `BLUEPRINT.md` 里，然后按依赖顺序从底层往上一层一层填。每个接口要写明前置条件、后置条件、错误处理和副作用——不是"大概对"，而是声明就要可验证。

## 怎么用

把整个目录复制到 skills 目录：

```bash
cp -r arch-first-dev ~/.deepseek/skills/   # DeepSeek TUI
cp -r arch-first-dev ~/.claude/skills/     # Claude Code
```

然后在 AI 编程工具里说：

> 用 arch-first-dev 帮我做一个命令行待办事项工具

AI 会自动走 Phase 1（画蓝图）→ Phase 2（逐模块实现）→ Phase 3（验证）。

## 不是什么场景都要走完整流程

分三层，按需用：

- **L1** — 只定义接口的 pre/post/error/side-effect，不画蓝图。适合设计接口、review 代码
- **L2** — L1 + 7 条约束检查。适合重构、修接口不一致
- **L3** — 完整的蓝图→填充→验证。适合新项目或大型重构

## 解决了什么问题

AI 写代码最常见的翻车方式不是报错，而是**不报错但逻辑断裂**——跨模块的数据流对不上，函数返回值上游和下游预期的不一样。这份 skill 的核心就是让逻辑在写代码之前就被验证。

具体做法：
- 每个接口强制声明 pre/post/error/side-effect，写代码时逐行对照
- 模块按依赖排序，下层没做完上层不动
- 改蓝图必须记录变更原因（@CHANGE），防止隐式修改导致接口漂移

## 目录

```
arch-first-dev/
├── SKILL.md                      # 核心流程，始终加载
└── references/
    ├── platform-adaptation.md    # 不同 AI 工具的命令映射
    ├── model-adaptation.md       # 不同模型的参数适配
    └── advanced-features.md      # Behavior-Code 对照、并行填充、多会话协作等
```

## 示例

[examples/task-cli/BLUEPRINT.md](examples/task-cli/BLUEPRINT.md) 是一个用这套方法从头构建的命令行任务管理器的完整蓝图——4 个模块，15 个接口，9 条数据流。

## License

MIT