# arch-first-dev

**架构优先的 AI 编程方法论。先画蓝图，再逐模块实现。**

---

用 AI 写代码有个很常见的坑：代码能跑，逻辑是断的。

比如你让 AI 做一个记账工具。它生成了四个模块——数据存储、账目管理、统计分析、命令行界面——每个看起来都没问题，跑起来也不报错。但统计模块调用的函数返回值结构和账目模块实际返回的不一样，数据流在某一步悄悄断了。不报错，只是结果不对。

这类问题靠"跟 AI 说仔细一点"解决不了。因为上下文一长，AI 自己也会忘。

这份 skill 的思路是：**别让 AI 一上来就写代码。先让它把架构画清楚。**

## 怎么做的

整个过程分三个阶段：

**Phase 1 — 画蓝图。** AI 把所有模块、接口、数据流、依赖关系写进一份 `BLUEPRINT.md`。这不是设计文档，是**可验证的契约**——每个接口必须写明：

- `pre` — 调用前什么条件必须成立
- `post` — 返回后什么条件一定成立
- `error` — 什么情况下抛什么错
- `side-effect` — 这个函数改变了什么状态

**Phase 2 — 逐模块实现。** 按依赖顺序从底层往上填。底层模块没做完，上层不开始。每个模块实现完后，把代码和契约逐行对照——不是问"你觉得对吗"，而是"声明的每一项，代码里有对应行吗"。找不到就是漏了。

**Phase 3 — 验证。** 跑一遍所有数据流，确认没有 import 错误、没有未捕获的异常、接口签名对得上。

全部做完后，`BLUEPRINT.md` 留在项目里，就是你的项目架构文档。

## 一个例子

用这套方法从零构建了一个命令行任务管理器（Task CLI）。蓝图长这样：

```
@PROGRESS
  storage        ██████████ [done]
  tasks          ██████████ [done]
  stats          ██████████ [done]
  cli            ██████████ [done]

@MODULE storage      ← JSON 文件读写
@MODULE tasks         ← 增删改查 + 状态变更
@MODULE stats         ← 完成率统计 + 逾期检测
@MODULE cli           ← 命令行解析 + 格式化输出

9 条 @FLOW，15 个接口，每个都有完整的 pre/post/error/side-effect 声明
```

完整蓝图见 [examples/task-cli/BLUEPRINT.md](examples/task-cli/BLUEPRINT.md)。

## 不是什么项目都要走完整流程

分三层，灵活选用：

| 层级 | 什么时候用 | 产出 |
|------|-----------|------|
| L1 | 设计接口、review 代码 | 每个接口的 pre/post/error/side-effect |
| L2 | 重构、检查接口一致性 | L1 + 7 条约束检查 |
| L3 | 新项目、大型重构 | 完整蓝图 → 逐模块实现 → 验证 |

一个接口级别的 review 用 L1，五分钟的事，不需要画蓝图。

## 7 条硬约束

这些规则是用来防止 AI 在长上下文里自己把自己搞乱的：

1. 蓝图没确认就不写代码
2. 一次只做一个模块
3. 依赖的模块没做完，上层不开始
4. 改蓝图必须记录变更原因
5. 模块之间只通过接口通信，不直接访问内部文件
6. 每个接口必须在至少一条数据流中被使用
7. 依赖模块的源码只读一次，不反复翻

## 跨平台 & 跨模型

这套方法本身不绑定任何 AI 工具。skill 文件里有平台工具映射表（DeepSeek TUI、Claude Code、Trae、Cursor、Codex CLI）和模型适配参数（包括新模型自动分类逻辑），在不同环境下会自动调整执行策略。

## 什么时候不适合

- 改个配置、修个单行 bug、换个颜色——不需要架构设计
- 已经有一个很好的架构文档，只是加个小功能——没必要重新画蓝图
- 原型阶段，想快速试一下可行性——先跑起来再说

## 安装

```bash
cp -r arch-first-dev ~/.deepseek/skills/   # DeepSeek TUI
cp -r arch-first-dev ~/.claude/skills/     # Claude Code
```

然后在 AI 编程工具里输入：

> 用 arch-first-dev 帮我做一个 ________

## 目录

```
arch-first-dev/
├── SKILL.md                      # 核心流程
├── README.md
├── LICENSE
├── examples/
│   └── task-cli/
│       └── BLUEPRINT.md          # 完整蓝图示例
└── references/
    ├── platform-adaptation.md    # 各平台工具映射
    ├── model-adaptation.md       # 模型适配 + 新模型自分类
    └── advanced-features.md      # 逐行对照、并行填充、多会话协作等
```

## License

MIT