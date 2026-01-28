# Hermes DEX - Claude 上下文文档

## 🌐 语言规范 (Language Policy)

**重要**: 所有在本项目工作的 Claude 助手必须遵循以下语言规则:

### 代码与技术层面
- ✅ **代码**: 仅使用英文(变量、函数、类型等)
- ✅ **代码注释**: 使用中文
- ✅ **提交信息**: 使用中文
- ✅ **Git 分支名**: 使用英文

### 文档与沟通层面
- ✅ **问题解释**: 使用中文
- ✅ **分析报告**: 使用中文
- ✅ **用户交流**: 使用中文
- ✅ **技术讨论**: 使用中文

### 示例
```go
// ✅ 正确 - 英文代码 + 中文注释
// 处理订单匹配逻辑
func ProcessOrderMatching(order *Order) error {
    // 验证订单有效性
    if err := validateOrder(order); err != nil {
        return err
    }
    // 执行匹配引擎
    return matchingEngine.Execute(order)
}

// ❌ 错误 - 中文代码或英文注释
// Process order matching logic
func 处理订单匹配(订单 *Order) error { ... }
```

---

## 项目概述

Hermes 是一个基于 **CometBFT** 和 **Cosmos SDK** 构建的高性能区块链去中心化交易所(DEX)。系统实现了链上订单匹配、永续交易和高级风险管理功能。

## 架构设计

### 技术栈

- **CometBFT** (v0.38.15): 拜占庭容错共识引擎
- **Cosmos SDK** (v0.50.11): 模块化区块链框架
- **ABCI**: 应用区块链接口 - 共识层与应用层之间的关键调用路径

### 目录结构

```
hermes/
├── protocol/              # 核心区块链协议
│   ├── app/              # 应用设置与配置
│   ├── cmd/hermesd/      # 节点二进制文件与 CLI
│   ├── x/                # 自定义 Cosmos 模块(业务逻辑)
│   │   ├── clob/         # ⭐ 核心: 订单簿与匹配引擎
│   │   ├── subaccounts/  # 账户与抵押品管理
│   │   ├── perpetuals/   # 永续合约配置
│   │   ├── prices/       # 预言机价格源
│   │   ├── leverage/     # 杠杆与保证金管理
│   │   └── ...           # 其他模块(资产、奖励等)
│   ├── indexer/          # 事件索引用于查询
│   ├── daemons/          # 链下服务(清算、预言机)
│   └── lib/              # 共享工具库
```

## 核心模块

| 模块              | 用途                                             |
|------------------|--------------------------------------------------|
| `x/clob`         | **中央订单簿**: 订单放置、取消、匹配、清算           |
| `x/subaccounts`  | 抵押品跟踪、保证金计算                              |
| `x/perpetuals`   | 永续合约定义与配置                                 |
| `x/prices`       | 预言机价格集成                                     |
| `x/leverage`     | 杠杆限制与去杠杆逻辑                                |

## ABCI 生命周期(关键路径)

ABCI 接口定义了 CometBFT 与应用程序的通信方式:

1. **PreBlock**: 初始化模块状态(`clob.PreBlocker`)
2. **BeginBlock**: 设置每个区块的状态与事件
3. **DeliverTx**: 执行交易(订单放置/取消)
4. **EndBlock**: 处理匹配、清算、状态最终确定
5. **Commit**: 将状态持久化到磁盘

**大部分业务逻辑位于 `protocol/x/` 模块中**,通过 ABCI 钩子访问。

## 开发注意事项

### 订单类型

- **Short-Term**: 单区块订单(立即执行或过期)
- **Long-Term**: 多区块订单,带有 Good-Til-Time (GTT)
- **Conditional**: 由价格条件触发(止损、止盈订单)
- **TWAP**: 时间加权平均价格执行

## 重要约束

1. **确定性**: 所有操作必须是确定性的,以满足共识要求
2. **Gas 限制**: 复杂操作必须遵守区块 Gas 限制
3. **状态管理**: 仔细区分持久化状态与瞬态状态
4. **MEV 保护**: 内置防止抢跑的机制

## 延伸阅读

- `protocol/x/clob/CLAUDE.md` - 订单簿实现深入解析
- `README.md` - 项目设置与部署指南
- Cosmos SDK 文档: https://docs.cosmos.network
- CometBFT 文档: https://docs.cometbft.com

---

## 🤖 Claude Code 使用技巧

### 快速上手命令

#### 1. 代码探索与理解

```bash
# 快速了解订单匹配的实现位置
"订单匹配逻辑在哪里实现?"

# 理解某个模块的作用
"解释一下 x/subaccounts 模块的作用"

# 查看特定功能的代码路径
"展示从订单提交到执行的完整调用链"
```

#### 2. 代码搜索与分析

```bash
# 查找特定函数的实现
"找到 PlaceOrder 函数的所有实现"

# 搜索错误处理相关代码
"搜索所有 liquidation 相关的错误处理"

# 分析测试覆盖
"分析 memclob 模块的测试覆盖情况"
```

#### 3. 测试相关操作

```bash
# 运行特定测试
"运行 x/clob 模块的所有测试"

# 分析测试失败原因
"分析为什么 TestPlaceOrder 测试失败"

# 创建新测试
"为 CancelOrder 函数添加边界条件测试"
```

#### 4. 代码修改与重构

```bash
# 添加功能
"在订单匹配时添加手续费计算逻辑"

# 修复 bug
"修复 memclob 中的订单取消逻辑 bug"

# 代码重构
"重构 process_operations.go 中的清算逻辑,提高可读性"
```

#### 5. 文档与注释

```bash
# 为复杂函数添加注释
"为 MatchOrders 函数添加详细的中文注释"

# 生成函数文档
"为 keeper.go 中的公开函数生成文档"

# 更新 README
"更新 CLOB 模块的 README,说明新的订单类型"
```

### 高级使用技巧

#### 🔍 使用 Skills 快速定位

Hermes 项目配置了专门的 skills,可以快速访问特定领域的知识。

##### Skills 总览

| Skill | 触发词 | 用途 | 位置 |
|-------|--------|------|------|
| **clob-module** | 订单匹配、订单类型、CLOB、匹配引擎、清算 | 深入理解 CLOB 模块的订单簿实现、匹配算法、状态管理 | `.claude/skills/clob-module/` |
| **cosmos-patterns** | keeper、模块结构、消息处理、Cosmos SDK | 学习 Cosmos SDK 开发模式、keeper 模式、状态管理 | `.claude/skills/cosmos-patterns/` |
| **dex-system** | DEX 架构、ABCI、区块链、共识、模块交互 | 理解整体 DEX 系统架构、ABCI 生命周期、模块协作 | `.claude/skills/dex-system/` |

##### 手动调用 Skills

```bash
# 方式 1: 使用斜杠命令(推荐)
/clob-module
/cosmos-patterns
/dex-system

# 方式 2: 在问题中包含触发词(自动)
"解释订单匹配引擎的实现" → 自动加载 clob-module
"如何实现 keeper 模式?" → 自动加载 cosmos-patterns
"ABCI 生命周期是什么?" → 自动加载 dex-system
```

##### Skill 组合使用

某些复杂任务可能需要多个 skills 协同:

```bash
# 示例 1: 添加新的订单类型
"实现一个新的止损订单类型"
# 会自动加载:
# - clob-module (订单类型和匹配逻辑)
# - cosmos-patterns (如何在 keeper 中实现)

# 示例 2: 理解模块间通信
"解释 CLOB 模块如何与 Subaccounts 模块交互"
# 会自动加载:
# - clob-module (CLOB 侧的逻辑)
# - dex-system (模块间通信机制)
# - cosmos-patterns (keeper 依赖注入)
```

##### Skills 目录结构

```
.claude/skills/
├── clob-module/
│   └── SKILL.md              # 订单簿深度解析
├── cosmos-patterns/
│   └── SKILL.md              # SDK 开发模式
└── dex-system/
    └── SKILL.md              # 系统架构总览
```

##### 添加自定义 Skills

如果需要添加项目特定的 skill:

1. **创建目录和文件**
   ```bash
   mkdir -p .claude/skills/my-skill
   touch .claude/skills/my-skill/SKILL.md
   ```

2. **编写 SKILL.md** (必须包含 YAML 头)
   ```yaml
   ---
   name: my-skill
   description: 简短描述功能和触发词。Use when [用户会说的关键词]
   ---

   # Skill 标题

   ## 使用说明
   详细的指导内容...

   ## 示例
   具体的代码示例...
   ```

3. **自动生效** - 无需额外配置,Claude 会自动发现并加载

##### Skill 开发最佳实践

**✅ 好的 Skill description**
```yaml
description: Deep dive into CLOB order matching, order types, and liquidations. Use when working with order placement, matching engine, or trading logic.
# 优点:
# - 列举具体能力(order matching, order types, liquidations)
# - 包含真实触发词(order placement, matching engine, trading logic)
# - 清晰定义使用场景
```

**❌ 差的 Skill description**
```yaml
description: Help with CLOB module development
# 问题:
# - 太模糊,Claude 不知道何时加载
# - 缺少具体触发词
# - 没有明确使用场景
```

#### 📊 代码分析工作流

```bash
# 1. 先探索代码结构
"展示 x/clob 模块的目录结构和主要文件"

# 2. 理解关键数据流
"解释订单从提交到成交的完整数据流"

# 3. 分析依赖关系
"分析 CLOB 模块依赖了哪些其他模块"

# 4. 检查测试覆盖
"列出 keeper 目录下所有测试文件及其测试内容"
```

#### 🛠️ 开发工作流优化

```bash
# 任务规划模式
"我需要添加一个新的订单类型(Stop-Limit),帮我规划实现步骤"

# 并行探索
"同时分析 PlaceOrder 和 CancelOrder 的实现逻辑"

# 增量开发
"先实现基础的 Stop-Limit 订单验证逻辑,不要添加匹配功能"
```

#### 🧪 测试驱动开发

```bash
# 先写测试
"为新的 Stop-Limit 订单类型编写测试用例"

# 运行并分析测试
"运行测试并分析失败原因"

# 迭代修复
"根据测试失败结果修复实现代码"
```

### 最佳实践

#### ✅ 推荐做法

1. **明确任务边界**
   ```bash
   # 好的提问方式
   "只添加订单验证逻辑,不要修改匹配引擎"

   # 避免的提问方式
   "优化订单处理" (太模糊)
   ```

2. **利用项目上下文**
   ```bash
   # Claude 会自动读取 CLAUDE.md 文件
   "根据项目规范,为新函数添加中文注释"
   ```

3. **分步骤处理复杂任务**
   ```bash
   # 第一步
   "分析现有订单类型的实现模式"

   # 第二步
   "基于分析结果,设计 Stop-Limit 订单的数据结构"

   # 第三步
   "实现 Stop-Limit 订单的验证逻辑"
   ```

4. **充分利用并行能力**
   ```bash
   "同时读取 msg_server_place_order.go 和 msg_server_cancel_orders.go,对比实现差异"
   ```

#### ❌ 避免的做法

1. **不要过度工程化**
   ```bash
   # 避免
   "重构整个 CLOB 模块的架构"

   # 推荐
   "重构 PlaceOrder 函数,提取订单验证逻辑"
   ```

2. **不要跳过测试**
   ```bash
   # 避免
   "快速实现功能,跳过测试"

   # 推荐
   "实现功能并添加单元测试"
   ```

3. **不要忽略项目约定**
   ```bash
   # 避免
   "用英文写注释" (违反项目规范)

   # 推荐
   "按照项目规范用中文写注释"
   ```

### 常见场景示例

#### 场景 1: 理解现有代码

```bash
Q: "解释 memclob.go 中的订单匹配算法"
→ Claude 会分析代码并用中文解释算法原理

Q: "PlaceOrder 函数的调用链是什么?"
→ Claude 会追踪完整的函数调用路径
```

#### 场景 2: 添加新功能

```bash
Q: "我想添加订单优先级功能,帮我规划实现方案"
→ Claude 会进入 Plan Mode,详细规划实现步骤

Q: "实现订单优先级验证逻辑"
→ Claude 会编写代码并添加中文注释
```

#### 场景 3: 调试问题

```bash
Q: "为什么 TestLiquidation 测试失败了?"
→ Claude 会分析测试代码和错误日志

Q: "修复订单取消时的状态不一致问题"
→ Claude 会定位问题并提供修复方案
```

#### 场景 4: 代码审查

```bash
Q: "审查 process_operations.go 中的清算逻辑"
→ Claude 会分析代码质量、性能和安全性

Q: "检查是否有潜在的确定性问题"
→ Claude 会重点检查共识相关的代码
```

### 提示词模板

#### 代码探索模板

```
[任务]: 分析 [文件/模块名]
[目标]: 理解 [具体功能]
[关注点]: [性能/安全性/可维护性等]
```

#### 功能开发模板

```
[需求]: 实现 [功能描述]
[约束]:
- 不修改现有 [某模块] 逻辑
- 保持与 [某功能] 兼容
- 遵循项目代码规范
[测试]: 需要覆盖 [场景1], [场景2]
```

#### 问题诊断模板

```
[问题]: [具体问题描述]
[复现]: [如何复现]
[期望]: [期望的正确行为]
[日志]: [相关错误日志]
```

### 调试技巧

#### 使用 Git 集成

```bash
# 查看当前改动
"显示我修改了哪些文件"

# 创建提交
"创建一个提交,包含所有 CLOB 相关的修改"

# 创建 PR
"创建 PR 到 main 分支,标题为'添加订单优先级功能'"
```

#### 日志分析

```bash
"分析测试失败的日志,定位问题原因"

"从错误堆栈追踪调用链"
```

#### 性能分析

```bash
"分析 MatchOrders 函数的时间复杂度"

"找出可能的性能瓶颈"
```

### 项目特定技巧

#### ABCI 生命周期调试

```bash
"在 PreBlocker 阶段添加调试日志"

"追踪订单在 BeginBlock 到 EndBlock 之间的状态变化"
```

#### 确定性验证

```bash
"检查 MatchOrders 函数是否满足确定性要求"

"验证订单排序逻辑是否稳定"
```

#### 状态管理

```bash
"分析哪些状态存储在 KVStore,哪些在 MemClob"

"检查状态迁移逻辑是否正确"
```

---

## 📝 提交规范

### Commit Message 格式

```
<类型>(<模块>): <简短描述>

<详细描述>

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>
```

### 类型说明

- `feat`: 新功能
- `fix`: Bug 修复
- `refactor`: 代码重构
- `test`: 测试相关
- `docs`: 文档更新
- `perf`: 性能优化
- `style`: 代码格式调整

### 示例

```
feat(clob): 添加订单优先级功能

- 在 Order 结构中添加 Priority 字段
- 更新匹配算法支持优先级排序
- 添加相关测试用例

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>
```