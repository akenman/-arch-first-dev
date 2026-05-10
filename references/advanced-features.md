# 高级特性

> 加载条件：显式需要时。各节独立触发条件：
> - Behavior-Code 对照 → 实现完接口需要对照验证时
> - Auto-Checkpoint → 模块标记 [done] 后
> - Design Decision Log → 选择设计方案时
> - 并行填充 → 同层 ≥ 2 模块 [empty]
> - 多会话协作 → 多人协作场景
> - 退出条件 → 用户问"什么时候退出 skill"
> - @CHANGE vs @DECISION → 修改蓝图需要区分变更类型
> - Rollback → 下游模块因上游变更无法对接
> - 测试策略 → 模块 [done] 后生成测试
> - 验收清单 → Phase 3 完成后

## 高级特性（竞品借鉴）

### 1. Behavior-Code 逐行对照（替代 Self-Critique）

研究表明 LLM 对自己的输出有确认偏误——自我批评的检查清单容易被"逐项打勾通过"。替代方案：**把刚实现的代码和 BEHAVIOR 声明逐行对照**，不做主观判断。

实现完每个接口后，打开自己刚写的代码文件，指着每一行对照 BEHAVIOR 声明：

```
# 在代码中找到以下四段证据，找不到就是漏了：

① pre 的落实证据
  蓝图声明: pre: name 非空, type ∈ {income, expense}
  代码证据: if not name: raise ValueError(...)
            if type not in ("income", "expense"): raise ValueError(...)
  
② post 的落实证据
  蓝图声明: post: 返回的 Category.id > 0
  代码证据: cat = {"id": _next_id, ...}  # id 从 1 开始自增
            return cat

③ error 的落实证据
  蓝图声明: error: ValueError
  代码证据: 所有 if...raise 路径覆盖了所有 error 声明

④ side-effect 的落实证据
  蓝图声明: side-effect: _data 追加一条记录
  代码证据: _data.append(cat)
```

对比规则：**不要问"你觉得做得对吗"——只找"声明是否在代码中有对应行"**。找不到就是漏了。

### 2. Auto-Checkpoint（源自 Claude Code Memory）

每个模块标记 [done] 后，自动写入 `.arch/CHECKPOINT.md`：

```
@CHECKPOINT
  时间: Phase-2 / auth 完成
  状态: 2/6 模块 done
  进度: categories [done], auth [done], transactions [in progress]
  最后变更: @CHANGE_003 — auth.login 增加空密码校验
  上下文预算: 🟡 62%
  当前加载: auth 接口签名 + transactions @MODULE
```

作用：checkpoint 文件是 **压缩上下文后的超快速恢复点**。不需要读整个 BLUEPRINT.md，一个 CHECKPOINT.md 就够 AI 知道"我在哪、下一步做什么"。

自动写入规则：
```
每次 [done] 后 → 覆盖 CHECKPOINT.md
每次填充异常处理后 → 追加到 CHECKPOINT.md
Phase 3 完成后 → CHECKPOINT.md 标注 [COMPLETE]
```

### 3. Design Decision Log（源自 Architecture Decision Records）

@CHANGE 记录"改了什么"，@DECISION 记录"为什么这么改"。在 BLUEPRINT.md 中追加：

```
@DECISION_001
  场景: transactions 模块的 filter 参数设计
  选择: 用 dict 而非 SQL-like 字符串
  理由: 保持内存存储和未来数据库存储的接口兼容性
  替代方案考虑过: 字符串表达式、链式调用、builder 模式
  影响: 未来切换到 SQLite 时 filter dict 可以直接映射到 WHERE

@DECISION_002
  场景: budgets.checkBudget 的返回值设计
  选择: 返回 {spent, limit, remaining, percentage} 而非只返回 bool
  理由: 调用方（reports）需要具体的数值来生成报表
  替代方案考虑过: 返回 bool + 单独调用 getBudgetStatus
  影响: 一次调用获取全部信息，减少跨模块调用次数
```

@DECISION 的触发时机：
```
填充循环第 3 步（实现）中，遇到以下情况之一时记录：
  - 从多个可行的实现方案中选了一个
  - 接口设计有合理的替代方案
  - 为了未来可扩展性做了取舍
  - 性能/安全/兼容性方面的权衡
```

### 4. 并行填充（独立模块并行处理）

严格限定：只有同时满足以下两个条件的模块才能并行：

```
1. 在同一 @BUILD_ORDER 层级（彼此不依赖）
2. 双方的依赖都已 [done]

一个 AI 会话内不支持并行（上下文只有一个推理路径）。
并行适用于：用启动子任务工具分别填充独立模块。如平台不支持子任务，则改为串行依次填充同层模块。
每个子 agent 看到的上下文只包含蓝图 + 自己负责的模块。
```

### 5. 多会话协作

适用场景：多人并行开发同一蓝图的不同模块。

**模块锁协议**（`.arch/LOCKS.md`）：

```
@LOCKS
  storage        [claimed]  session-abc123  (2026-05-10 14:30)
  tasks          [claimed]  session-def456  (2026-05-10 14:31)
  stats          [free]
  reports        [free]
```

规则：
  - 填充模块前，检查 LOCKS.md 中该模块状态
  - [free] → 标记 [claimed] + 会话 ID + 时间戳 → 开始填充
  - [claimed] 且距上次更新时间 > 30 分钟 → 可抢占（原会话可能已失效）
  - [done] → 不可抢占，已完成
  - 每次 CHECKPOINT 写入时同时更新 LOCKS.md 的时间戳（续约）

**合并协议**：

两个会话完成各自模块后：
  1. 任一会话读取完整 BLUEPRINT.md + LOCKS.md
  2. 检查所有 [done] 模块的接口签名是否有冲突（同名不同签名）
  3. 无冲突 → 合并，运行 Phase 3 全量验证
  4. 有冲突 → 冲突模块重置为 [in progress]，按构建顺序重新填充
  5. 合并完成后，LOCKS.md 归档到 .arch/LOCKS_ARCHIVE.md

**CHECKPOINT 协调**：

每个会话的 CHECKPOINT.md 追加同行信息：
```
@PEERS
  session-abc123: storage [done]       (最后更新: 14:35)
  session-def456: tasks [in progress]  (最后更新: 14:33)
```
读取 BLUEPRINT.md 时同时读取所有活跃会话的 CHECKPOINT.md，避免重复工作。

**限制**：多会话协作要求 BLUEPRINT.md 和 LOCKS.md 共享（如通过 Git），本 skill 不负责文件同步机制。

### 6. 退出条件

skill 的生效范围是**从 Phase 1 开始到 Phase 3 完成**，外加后续的**迭代模式**。

以下情况应**退出**本 skill：

```
✓ Phase 3 验证全部通过 → 项目已完整实现，退出 skill
✓ 用户说"改一下 UI 颜色"、"重命名一个函数"等单文件改动 → 不触发 skill
✓ 用户说"帮我写个注释"、"格式化代码" → 不触发 skill
```

以下情况应**继续**使用本 skill：

```
✓ 用户说"加一个新功能" → 迭代模式
✓ 用户说"重构模块 X" → 迭代模式（标记 X 为 [in progress]）
✓ 用户说"修这个 bug" → 迭代模式
```

退出后，BLUEPRINT.md 保留在项目 .arch/ 目录中作为**项目架构文档存档**，不再占用上下文。

### 7. @CHANGE 与 @DECISION 的边界

两者都记录修改，但用途不同：

```
@CHANGE 记录"改了什么"（事实）
  触发: 任何对 BLUEPRINT.md 的修改
  格式: 阶段 / 触发 / 原因 / 修改 / 影响 / 约束
  数量: 可能很多（每个修改都要记）
  保留: 约束 4 强制要求

@DECISION 记录"为什么这样设计"（理由）
  触发: 只记有替代方案的、非显而易见的设计选择
  格式: 场景 / 选择 / 理由 / 替代方案考虑过 / 影响
  数量: 少（只有关键选择才记）
  保留: 建议性（不需要 @DECISION 也可以通过验证）

举一个具体场景来说明区别：
  场景: transactions.filter 从 SQL-like 改为 dict
  @CHANGE_005: 改了什么 → filter 参数类型变了
  @DECISION_002: 为什么这样改 → 为了未来兼容 SQLite

两者不冲突，同一个修改可以同时有 @CHANGE 和 @DECISION。
但不必每次都记 @DECISION——只记"别人会问为什么"的情况。
```

### 8. Rollback 协议（源自 Cursor Rules 实践）

当 @CHANGE 的修改引发下游模块的连锁问题时，执行 rollback：

```
触发条件：下游模块填充时发现上游模块的接口变更导致无法对接

Rollback 步骤：
  1. 标记当前模块为 [blocked]
  2. 确认引发问题的上游 @CHANGE 编号
  3. 回滚该 @CHANGE：将上游模块的接口恢复为变更前的签名
  4. 恢复后上游模块状态重置为 [in progress]
  5. 在 @CHANGE 中追加 rollback 记录：
     @CHANGE_004
       触发: Rollback from budgets.checkBudget
       原因: auth.validateToken 返回了额外的 role 字段，budgets 未预期
       操作: auth.validateToken 返回值恢复为 {userId}（去掉 role）
       影响: auth 回到 [in progress]，需要重新实现
       约束: 全部通过

不触发 rollback 的条件：
  - 下游模块尚未开始填充（直接修改上游接口，无成本）
  - 下游模块可以直接适配新接口且受影响文件 ≤ 2 个（> 2 个时执行 rollback）
```

## 逆向蓝图（8 步完整流程）

> 加载条件：现有项目需要逆向蓝图时

```
第 1 步：列出项目文件
    用列出目录内容工具递归列出所有源文件。
    排除测试文件、配置文件、生成文件。

第 2 步：逐个模块识别
    按目录结构判断模块边界（通常一个子目录就是一个模块）。
    如果一个目录下只有一个 index.ts/__init__.py，它的导出就是模块接口。
    如果所有文件在一个 src/ 下没有子目录 → 按功能聚类：用 grep 找核心功能的关键字。

第 3 步：提取接口签名
    对每个模块，读入口文件（index.ts、__init__.py）的导出。
    只收集函数签名和类型定义，不读内部实现。
    产出：@MODULE 列表（接口名、参数、返回类型）。

第 4 步：推断依赖
    从 import/require/use 语句推断依赖。
    只收集跨模块引用（不收集标准库和第三方包）。
    产出：每个 @MODULE 的「依赖」字段。

第 5 步：生成 @DATA
    从类型定义、数据类、接口定义中提取数据结构。
    如果一个类型在多个模块中出现 → 它是 @DATA（跨模块共享）。
    如果一个类型只在单个模块内部使用 → 不是 @DATA，不写进蓝图。

第 6 步：生成 @FLOW
    从入口文件（路由、CLI 命令、事件处理器）出发，追踪调用链。
    每一条从用户操作到数据返回的路径就是一条 @FLOW。
    不需要精确到每一步，核心路径即可。

第 7 步：标注 [done] 和 @UNCLEAR
    提取出接口的模块 → 状态: [done]
    推导出接口但不确定实现是否完整的模块 → 状态: [done] 标注 @UNCLEAR
    完全看不出来的部分 → 状态: [empty]（需要补充实现）

第 8 步：整理为 BLUEPRINT.md
    与新建项目同样的蓝图格式。
    全部模块初始状态为 [done] 或带 @UNCLEAR 标记的 [done]。
```

逆向蓝图完成后，修改需求进入迭代模式。并非从 Phase 1 重新开始。

---

## @EXTERNAL 完整格式示例

```
@EXTERNAL PostgreSQL
  类型: 关系数据库
  连接: DATABASE_URL 环境变量
  契约:
    - 所有持久化操作通过此数据库，不使用 ORM（直接 SQL）
    - 表结构与 @DATA 一一对应
    - 连接池: min=2, max=10
  故障模式: 连接失败 → 重试 3 次 → 抛 DatabaseError

@EXTERNAL Stripe API
  类型: 第三方支付
  认证: STRIPE_SECRET_KEY
  契约:
    - createPayment(amount, currency) → PaymentIntent
    - 仅 payments 模块可以调用
  故障模式: API 超时 → 重试 1 次 → 返回 pending 状态
```

---

## 测试策略（最小化）

Phase 2 每模块 [done] 后：
  - 在模块目录下创建 test_<module>.py
  - 每个对外的接口至少一个 happy-path 测试用例
  - 边界矩阵中标记 ✓ 的维度 → 至少 1 个对应边界测试
  - 错误链中的每条路径 → 至少 1 个触发测试

  测试文件豁免约束 5（模块边界）：测试可以跨模块调用，用于集成验证。

Phase 3 验证通过后：
  - 运行全量测试套件：`python -m pytest src/ -q`
  - 覆盖率不做硬性要求（重点在关键路径和边界，不在数字）

## 验收清单

### 蓝图质量
- [ ] 所有功能点已覆盖
- [ ] @BUILD_ORDER 传递性正确，无循环依赖
- [ ] 每个模块 1-7 个接口，职责单一

### 约束遵守
- [ ] 7 条硬规则无违规
- [ ] 每次蓝图修改都有 @CHANGE

### 逻辑幻觉防护
- [ ] 每个接口有 pre/post/error/side-effect 四项
- [ ] 边界矩阵全 ✓ 或 —
- [ ] @ERROR_CHAIN 覆盖所有跨模块错误路径

### 上下文健康
- [ ] 压力等级 🟢 或 🟡
- [ ] 已完成模块的实现代码已释放
- [ ] @CHANGE ≤ 10 条