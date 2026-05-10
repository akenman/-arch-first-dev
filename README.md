# arch-first-dev

架构优先的 AI 编程方法论 —— 先画蓝图，再逐格填充，全局逻辑零断裂。

## 一句话

让 AI 在写代码之前，先把整个产品的架构画在一份文件里。然后按依赖顺序从底层往上，一次填一个模块。蓝图就是全局眼睛。

## 快速开始（3 分钟）

```bash
# 1. 安装：复制到 skills 目录
cp -r arch-first-dev ~/.deepseek/skills/      # DeepSeek TUI
cp -r arch-first-dev ~/.claude/skills/        # Claude Code

# 2. 在 AI 编程工具中输入：
"用 arch-first-dev 帮我做一个命令行待办事项工具"

# 3. AI 自动：
#    → 画蓝图（BLUEPRINT.md）
#    → 逐模块填充实现
#    → 运行时验证
#    → 生成人类可读的架构摘要（SUMMARY.md）
```

## 三层采用粒度

不一定要走完整流程，按需选择：

| 粒度 | 适合场景 | 产出 |
|------|---------|------|
| L1 行为契约 | 设计接口、Review 代码 | pre/post/error/side-effect 声明 |
| L2 约束驱动 | 重构、修复接口不一致 | L1 + 7 条约束检查 |
| L3 完整流程 | 新项目、大型重构 | 蓝图 → 填充 → 验证 |

## 核心特性

- **蓝图填充模型**：BLUEPRINT.md 作为全局导航 + 进度追踪 + 空位标记
- **7 条硬约束**：用户确认门、一次一格、依赖先填、变更有痕...
- **3 层幻觉防护**：行为声明 (pre/post/error/side-effect) + 边界矩阵 + 错误链映射
- **上下文压力感知**：4 个可观察信号，自动释放已实现代码
- **规模自适应**：≤10 接口简化模式，>50 接口风险导向验证
- **渐进式采用**：L1/L2/L3 三层粒度，按需选择

## 平台 & 模型适配

- **5 平台**：DeepSeek TUI / Claude Code / Trae / Cursor / Codex CLI
- **8 模型**：DeepSeek V4/V3 / Claude 4.x / GPT-4o / GLM-4 / Qwen 3 / Mistral Large / Llama 4
- **新模型自动适配**：4 问题自分类，未来模型零维护

## Token 收益

| 项目规模 | 不使用 skill | 使用 skill | 节省 |
|---------|------------|-----------|------|
| 4 模块 / 15 接口 | ~22,000 tokens | ~8,150 tokens | **63%** |
| 10 模块 | ~80,000 tokens | ~18,000 tokens | **77%** |
| 20 模块 | ~280,000 tokens | ~35,000 tokens | **87%** |

**DeepSeek V4 专项**：prefix cache 命中率从 ~15% 提升到 ~90%，每次填充成本降低约 10×。

## 目录结构

```
arch-first-dev/
├── SKILL.md                        # L1 核心（始终加载，~2,250 tokens）
└── references/
    ├── platform-adaptation.md      # L2: 平台工具映射表
    ├── model-adaptation.md         # L2: 模型适配 + 能力探针
    └── advanced-features.md        # L3: 高级特性 + 测试 + 逆向蓝图
```

## 实战示例

以下是一个用本 skill 从零构建的 **Task CLI** 项目（命令行任务管理器）：

```markdown
# 项目蓝图 — Task CLI

@PROGRESS
  storage        ██████████ [done]
  tasks          ██████████ [done]
  stats          ██████████ [done]
  cli            ██████████ [done]
  ──────────────────────────────
  总计: 4/4 模块完成 (100%)

@MODULE storage      ← 第 1 层: JSON 文件读写（loadTasks / saveTasks）
@MODULE tasks         ← 第 2 层: 增删改查 + 状态变更（7 接口）
@MODULE stats         ← 第 3 层: 完成率统计 + 逾期检测（2 接口）
@MODULE cli           ← 第 4 层: 参数解析 + 格式化输出（4 接口）

9 条 @FLOW: 添加/列出/查看/更新/删除/完成/开始/统计/逾期
15 个接口全部 pre/post/error/side-effect 行为声明
257 行有效代码，Phase 3 运行时验证全部通过
```

完整蓝图见 [examples/task-cli/BLUEPRINT.md](examples/task-cli/BLUEPRINT.md)。

## License

MIT