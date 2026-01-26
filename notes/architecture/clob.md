# CLOB 模块架构设计

## 1. 模块概述

### 1.1 职责定义

CLOB (中央限价订单簿) 模块是 Hermes DEX 交易系统的**核心引擎**,负责管理订单的完整生命周期:

- **订单管理**: 订单放置、取消、过期清理
- **订单匹配**: 基于价格-时间优先的确定性匹配算法
- **清算执行**: 保证金不足的强制平仓和去杠杆
- **MEV 保护**: 速率限制、权益层级限制、确定性匹配

### 1.2 核心功能

**8 种订单类型支持**:
1. **Short-Term Order** (短期订单): 单区块有效,减少链上存储
2. **Long-Term Order** (长期订单): GTT 多区块有效,限价单场景
3. **Conditional Order** (条件订单): 止损/止盈,价格触发执行
4. **TWAP Order** (时间加权平均价格订单): 大额订单拆分执行
5. **Reduce-Only Order**: 仅减仓,避免反向开仓
6. **Post-Only Order**: 仅做 Maker,获取手续费优惠
7. **IOC (Immediate-Or-Cancel)**: 立即成交或取消
8. **FOK (Fill-Or-Kill)**: 全部成交或取消

**核心机制**:
- **价格-时间优先匹配**: 最佳价格优先,相同价格时间早的优先
- **MemClob 内存订单簿**: 纯内存引擎,高性能匹配
- **清算与去杠杆**: 自动风险控制,保护系统安全
- **速率限制**: 防止垃圾订单攻击

### 1.3 关键约束

**确定性要求** (共识关键):
- 所有节点必须产生完全相同的匹配结果
- 订单排序必须稳定且确定性
- 不能使用随机数、当前时间戳等不确定性因素

**Gas 限制约束**:
- 复杂操作必须在区块 Gas 限制内完成
- 订单匹配批量处理,避免单笔交易消耗过多 Gas
- 状态修剪,防止状态膨胀

**共识要求**:
- 订单匹配在 FinalizeBlock 阶段执行,确保共识
- 通过 ProposedOperations 机制保证提议者与验证者一致性

---

## 2. 架构设计

### 2.1 组件结构

```
CLOB 模块架构层次

┌─────────────────────────────────────────────────┐
│         gRPC Query Server (查询接口)             │
│  ClobPair, StatefulOrder, Orderbook Updates    │
└─────────────────────────────────────────────────┘
                       ↕
┌─────────────────────────────────────────────────┐
│         Msg Server (消息处理器)                  │
│  PlaceOrder, CancelOrder, UpdateLeverage, etc  │
└─────────────────────────────────────────────────┘
                       ↕
┌─────────────────────────────────────────────────┐
│              Keeper Layer (业务逻辑层)           │
│  ├─ Order Placement Logic                       │
│  ├─ Order Cancellation Logic                    │
│  ├─ Process Operations (匹配执行)               │
│  ├─ Liquidations (清算逻辑)                     │
│  └─ Rate Limiter (速率限制器)                   │
└─────────────────────────────────────────────────┘
                       ↕
┌─────────────────────────────────────────────────┐
│           MemClob (内存订单簿引擎)               │
│  ├─ Orderbook (买卖订单簿)                      │
│  ├─ Operations Queue (待提议操作队列)           │
│  ├─ Matching Engine (价格-时间优先匹配)         │
│  └─ Snapshot Mechanism (快照机制)               │
└──────────────────────────────────────────────────┘
                       ↕
┌─────────────────────────────────────────────────┐
│              Storage Layer (存储层)              │
│  ├─ StateStore (持久化链上状态)                 │
│  ├─ MemStore (内存缓存)                         │
│  ├─ TransientStore (区块级瞬态存储)             │
│  └─ MemClob (纯内存,不持久化)                   │
└─────────────────────────────────────────────────┘
```

**组件职责说明**:

1. **gRPC Query Server**:
   - 提供查询接口 (ClobPair, StatefulOrder, OrderbookUpdates 等)
   - 支持流式查询 (StreamOrderbookUpdates)
   - 只读操作,不修改状态

2. **Msg Server**:
   - 处理所有用户发起的消息 (PlaceOrder, CancelOrder, UpdateLeverage 等)
   - 验证消息格式和权限
   - 委托给 Keeper 执行业务逻辑

3. **Keeper Layer**:
   - 核心业务逻辑实现
   - 与其他模块 Keeper 协作 (Subaccounts, Perpetuals, Prices)
   - 管理状态读写

4. **MemClob**:
   - 纯内存订单簿引擎
   - 价格-时间优先匹配算法
   - 操作队列管理

5. **Storage Layer**:
   - 4 种存储类型,分层管理不同类型的数据

### 2.2 存储架构 (4 层存储设计)

#### 为什么需要 4 层存储?

CLOB 模块需要处理大量订单数据,不同类型的订单有不同的生命周期和访问模式,因此设计了 4 层存储架构:

```
存储层次                   持久化       作用域       用途
┌────────────────┐
│  StateStore    │  ✅ 是     跨区块    长期订单、配置参数
├────────────────┤
│  MemStore      │  ✅ 是     跨区块    性能优化的内存缓存
├────────────────┤
│ TransientStore │  ❌ 否     单区块    短期订单、临时事件
├────────────────┤
│   MemClob      │  ❌ 否     跨区块*   纯内存订单簿
└────────────────┘
* MemClob 在每个区块 PreBlocker 重建,不持久化
```

#### StateStore (链上持久化存储)

**存储内容** (12 个数据结构):
1. CLOB 交易对配置 (ClobPair)
2. Long-Term 订单 (LongTermOrderPlacement)
3. Conditional 订单 - 已触发 (TriggeredConditionalOrder)
4. Conditional 订单 - 未触发 (UntriggeredConditionalOrder)
5. TWAP 订单 (TWAPOrderPlacement)
6. 订单成交量 (OrderAmountFilled)
7. 清算配置 (LiquidationsConfig)
8. 权益层级限制配置 (EquityTierLimitConfiguration)
9. 区块速率限制配置 (BlockRateLimitConfiguration)
10. 杠杆设置 (Leverage,存储在 CLOB 但属于 Leverage 模块)
11. 已触发条件订单时间窗口 (TriggeredConditionalOrdersTimeWindow)
12. CLOB 模块参数 (Params)

**为什么某些订单需要持久化?**

- **Long-Term 订单**: 有效期跨多个区块,必须持久化才能在多个区块中持续匹配
- **Conditional 订单**: 触发条件可能需要等待多个区块,必须持久化保存触发状态
- **TWAP 订单**: 执行时间窗口跨多个区块,需要持久化保存拆分逻辑
- **订单成交量**: 部分成交的订单需要跨区块记录已成交数量

**权衡分析**:
- ✅ 优势: 数据持久化,节点重启后状态不丢失
- ❌ 劣势: 读写开销大,影响性能
- 决策理由: 长期订单和配置必须持久化,短期数据可用其他存储

#### MemStore (内存缓存)

**存储内容** (6 个数据结构):
1. 已初始化标志 (Initialized)
2. 提议者匹配事件 (ProcessProposerMatchesEvents)
3. 已交付 Long-Term 订单 ID (DeliveredLongTermOrderIds)
4. 已交付 Conditional 订单 ID (DeliveredConditionalOrderIds)
5. 已交付 TWAP 订单 ID (DeliveredTWAPOrderIds)
6. 下一个 CLOB Pair ID (NextClobPairID)

**为什么需要 MemStore?**

- **性能优化**: 从 StateStore 同步到内存,读取速度快
- **共识状态的镜像**: 确保内存缓存与链上状态一致
- **临时标记**: 记录当前区块已交付的订单 ID,避免重复处理

**权衡分析**:
- ✅ 优势: 读取性能优于 StateStore
- ❌ 劣势: 需要同步逻辑,增加复杂度
- 决策理由: 频繁读取的配置数据适合缓存,提升性能

#### TransientStore (区块级瞬态存储)

**存储内容** (8 个数据结构):
1. Short-Term 订单放置 (ShortTermOrderPlacement)
2. 未提交订单 (未成交的 Short-Term 订单)
3. 子账户清算信息 (SubaccountLiquidationInfo)
4. 最低成交价格 (LowestFillPrice)
5. 最高成交价格 (HighestFillPrice)
6. 处理提议者匹配事件(ProcessProposerMatchesEvents,瞬态副本)
7. 已取消的条件订单 (CancelledConditionalOrder)
8. 子账户去杠杆信息 (SubaccountDeleveragingInfo)

**为什么不持久化这些数据?**

- **Short-Term 订单**: 单区块有效,区块结束自动过期,无需持久化
- **区块级临时状态**: 仅在当前区块有效的中间状态
- **性能优化**: 减少链上状态膨胀,降低存储成本

**权衡分析**:
- ✅ 优势: 避免状态膨胀,自动清理
- ❌ 劣势: 每个区块结束后数据丢失
- 决策理由: 单区块有效的数据无需持久化,节省存储成本

#### MemClob (纯内存订单簿)

**存储内容** (3 个核心结构):
1. **Orderbook** (订单簿):
   - Buy Orders (买单列表,按价格降序排列)
   - Sell Orders (卖单列表,按价格升序排列)
2. **Operations Queue** (操作队列):
   - 待提议的订单操作 (放置、取消、匹配)
3. **Snapshot** (快照):
   - 订单簿快照,用于恢复和验证

**为什么需要纯内存订单簿?**

**性能需求**:
- 订单匹配是高频操作,每秒可能处理数千笔订单
- 内存访问速度比磁盘快 1000 倍以上
- 匹配算法需要频繁遍历订单簿,内存结构更高效

**灵活性需求**:
- 支持复杂的订单匹配逻辑 (价格-时间优先、部分成交)
- 需要快速插入、删除、查找订单
- 需要支持订单簿快照和回滚

**权衡分析**:
- ✅ 优势:
  - 极高的匹配性能
  - 灵活的数据结构
  - 支持复杂的匹配算法
- ❌ 劣势:
  - 内存消耗大 (每个节点都需要维护)
  - 状态重建复杂 (PreBlocker 需要重建)
  - 不持久化,节点重启需要重建

**决策理由**:
- 订单匹配是核心性能瓶颈,必须优先保证性能
- 通过 PreBlocker 重建机制确保状态一致性
- 内存消耗在可接受范围内 (订单数量有限)

### 2.3 ABCI 生命周期集成

#### 完整 ABCI 流程图

```
┌──────────────────────────────────────────────────────────┐
│                      CometBFT 共识流程                    │
└──────────────────────────────────────────────────────────┘
                            │
                    ┌───────┴───────┐
                    │ PreBlocker    │ ← CLOB 初始化
                    └───────┬───────┘
                            │
                    ┌───────┴───────┐
                    │ BeginBlocker  │ ← CLOB 重置状态
                    └───────┬───────┘
                            │
    ┌───────────────────────┼───────────────────────┐
    │                       │                       │
    ▼                       ▼                       ▼
DeliverTx[0]         DeliverTx[1]            DeliverTx[N]
UpdateMarketPrices   AddPremiumVotes         User Transactions
    │                       │                       │
    └───────────────────────┼───────────────────────┘
                            │
                    ┌───────┴───────┐
                    │   DeliverTx   │ ProposedOperations
                    │  (匹配执行)    │ ← CLOB 核心
                    └───────┬───────┘
                            │
                    ┌───────┴───────┐
                    │  EndBlocker   │ ← CLOB 清理过期订单
                    └───────┬───────┘
                            │
                    ┌───────┴───────┐
                    │   Precommit   │ ← CLOB 索引器事件
                    └───────┬───────┘
                            │
                    ┌───────┴───────┐
                    │    Commit     │
                    └───────┬───────┘
                            │
                    ┌───────┴───────┐
                    │PrepareCheckState│ ← CLOB 重放操作
                    └───────┬───────┘
                            │
                          下一个区块
```

#### PreBlocker

**执行时机**: 在 BeginBlocker 之前,区块开始时第一个调用

**CLOB 模块操作**:
```go
func (k Keeper) PreBlocker(ctx sdk.Context) {
    // 1. 初始化 MemClob
    k.MemClob.CreateOrderbook(ctx, clobPair)

    // 2. 从 StateStore 加载 Long-Term 订单
    k.MemClob.RestoreLongTermOrders(ctx)

    // 3. 重建订单簿内存结构
    k.MemClob.RebuildOrderbook(ctx)
}
```

**为什么在这里初始化 MemClob?**

**原因分析**:
- MemClob 是纯内存结构,不持久化
- 每个新区块开始前需要重建订单簿状态
- PreBlocker 是最早的 ABCI 钩子,确保后续操作可用

**设计权衡**:
- ✅ 优势: 确保每个区块开始时订单簿状态一致
- ❌ 劣势: 每个区块都需要重建,增加开销
- 决策理由: 确保状态一致性优先于性能开销

#### BeginBlocker

**执行时机**: 在所有 DeliverTx 之前

**CLOB 模块操作**:
```go
func (k Keeper) BeginBlocker(ctx sdk.Context) {
    // 1. 重置匹配事件
    k.SetProcessProposerMatchesEvents(ctx, types.ProcessProposerMatchesEvents{})

    // 2. 清空已交付订单 ID 列表
    k.ClearDeliveredOrderIds(ctx)
}
```

**为什么重置某些状态?**

**原因分析**:
- 匹配事件是区块级状态,每个区块需要重新记录
- 已交付订单 ID 用于去重,新区块需要清空

**设计权衡**:
- ✅ 优势: 避免状态污染,确保区块间隔离
- ❌ 劣势: 每个区块都需要重置
- 决策理由: 区块间状态隔离是共识要求

#### DeliverTx (ProposedOperations)

**执行时机**: 在 DeliverTx 阶段,处理 MsgProposedOperations 消息

**CLOB 模块操作**:
```go
func (k msgServer) ProposedOperations(
    ctx sdk.Context,
    msg *types.MsgProposedOperations,
) (*types.MsgProposedOperationsResponse, error) {
    // 1. 验证提议者权限
    k.ValidateProposer(ctx, msg.ProposerAddress)

    // 2. 验证操作顺序(确定性要求)
    k.ValidateOperationOrdering(ctx, msg.Operations)

    // 3. 执行订单匹配
    matchedOrders := k.MemClob.MatchOrders(ctx, msg.Operations)

    // 4. 更新账户余额
    k.ProcessMatches(ctx, matchedOrders)

    // 5. 记录已成交订单
    k.UpdateOrderFillState(ctx, matchedOrders)

    // 6. 发出匹配事件
    k.EmitMatchEvents(ctx, matchedOrders)

    return &types.MsgProposedOperationsResponse{}, nil
}
```

**为什么订单匹配在这里执行?**

**原因分析**:
- DeliverTx 是共识保证的执行阶段
- ProposedOperations 确保提议者和验证者匹配结果一致
- 批量处理订单,减少 Gas 消耗

**设计权衡**:
- ✅ 优势: 共识保证、批量处理、确定性匹配
- ❌ 劣势: 单个区块匹配数量受 Gas 限制
- 决策理由: 共识和确定性是核心要求,性能其次

#### EndBlocker

**执行时机**: 在所有 DeliverTx 之后,区块结束前

**CLOB 模块操作**:
```go
func (k Keeper) EndBlocker(ctx sdk.Context) {
    // 1. 裁剪成交量记录 (OrderAmountFilled)
    k.PruneStateFillAmountsForShortTermOrders(ctx)

    // 2. 移除过期订单
    k.RemoveExpiredStatefulOrders(ctx)

    // 3. 生成 TWAP 子订单
    k.GenerateTWAPChildOrders(ctx)

    // 4. 触发条件订单
    k.TriggerConditionalOrders(ctx)

    // 5. 处理清算和去杠杆
    k.ProcessLiquidations(ctx)
}
```

**为什么这些操作在 EndBlocker?**

**原因分析**:
- 过期订单清理: 区块结束后判断订单是否过期
- TWAP 子订单生成: 根据区块时间判断是否生成新子订单
- 条件订单触发: 根据区块结束时的价格判断是否触发
- 清算处理: 区块结束后计算保证金率,判断是否清算

**设计权衡**:
- ✅ 优势: 在区块结束时统一处理,逻辑清晰
- ❌ 劣势: EndBlocker 执行时间长,可能影响区块确认速度
- 决策理由: 确保订单管理逻辑完整性

#### PrepareCheckState

**执行时机**: 在 Commit 之后,下一个区块开始前

**目的**: 为下一个区块的 CheckTx 阶段准备 MemClob 状态,确保本地状态与链上状态一致。

**CLOB 模块执行步骤**:

```go
func (k Keeper) PrepareCheckState(ctx sdk.Context) {
    // 1. 清理速率限制信息
    k.PruneRateLimits(ctx)

    // 2. 获取待重放操作队列
    operationsToReplay := k.MemClob.GetOperationsToReplay(ctx)

    // 3. 清除操作队列
    k.MemClob.RemoveAndClearOperationsQueue()

    // 4. 清理无效 MemClob 状态
    k.MemClob.PurgeInvalidMemclobState()

    // 5. 第一遍: 仅放置 POST_ONLY 订单
    k.PlaceStatefulOrdersFromLastBlock(ctx, operationsToReplay, postOnly=true)

    // 6. 第二遍: 放置所有其他 Long-Term 订单
    k.PlaceStatefulOrdersFromLastBlock(ctx, operationsToReplay, postOnly=false)

    // 7. 放置已触发的条件订单
    k.PlaceConditionalOrdersTriggeredInLastBlock(ctx)
}
```

---

### PrepareCheckState 详细流程说明

#### 步骤 1: 清理速率限制信息

```go
k.PruneRateLimits(ctx)
```

**作用**:
- 删除过时的速率限制数据,防止状态膨胀
- 确保速率限制仅跟踪最近的区块

---

#### 步骤 2-3: 获取并清除操作队列

```go
operationsToReplay := k.MemClob.GetOperationsToReplay(ctx)
k.MemClob.RemoveAndClearOperationsQueue()
```

**作用**:
- 获取本地验证器在上一个区块 CheckTx 期间收集的操作
- 这些操作需要在新区块的 MemClob 中重放,以保持状态一致性

**操作队列包含**:
- 订单放置操作
- 订单匹配操作
- 订单取消操作

---

#### 步骤 4: 清理无效 MemClob 状态

```go
k.MemClob.PurgeInvalidMemclobState()
```

**作用**:
- 清除已填充、已过期、已取消和已移除的订单
- 保持 MemClob 状态与 StateStore 一致
- 释放内存空间

---

#### 步骤 5-6: 两遍放置 Long-Term 订单 (关键设计!)

**为什么需要两遍放置?**

这是 PrepareCheckState 最关键的设计决策,确保 POST_ONLY 订单的语义正确性。

**第一遍: 仅放置 POST_ONLY 订单**
```go
k.PlaceStatefulOrdersFromLastBlock(ctx, operationsToReplay, postOnly=true)
```

**第二遍: 放置所有其他 Long-Term 订单**
```go
k.PlaceStatefulOrdersFromLastBlock(ctx, operationsToReplay, postOnly=false)
```

**设计理由**:

POST_ONLY 订单的语义要求它们**不能立即成交,必须挂单**。如果不分两遍放置:

**错误场景** (如果不分两遍):
```
假设有两个订单:
- 订单 A: POST_ONLY 买单,价格 50,000
- 订单 B: 普通卖单,价格 50,000

如果同时放置:
1. 订单 A 进入订单簿
2. 订单 B 进入订单簿,立即与订单 A 匹配
3. 订单 A 成交 ❌ 违反 POST_ONLY 语义!
```

**正确场景** (分两遍放置):
```
第一遍: 先放置 POST_ONLY 订单
1. 订单 A (POST_ONLY) 进入订单簿

第二遍: 放置其他订单
2. 订单 B (普通) 进入订单簿,与订单 A 匹配
3. 订单 A 作为 Maker 成交 ✅ 符合 POST_ONLY 语义!
```

**关键点**:
- **POST_ONLY 订单总是 Maker**: 先放置确保它们在订单簿中,后续订单作为 Taker 与之匹配
- **时间优先保证**: POST_ONLY 订单在相同价格档位拥有时间优先权
- **语义正确性**: POST_ONLY 订单绝不会作为 Taker 立即成交

**代码实现细节**:
```go
func (k Keeper) PlaceStatefulOrdersFromLastBlock(
    ctx sdk.Context,
    operations []Operation,
    postOnly bool,
) {
    for _, operation := range operations {
        if operation.Type != PlaceOrder {
            continue
        }

        order := operation.Order

        // 第一遍: 仅处理 POST_ONLY 订单
        if postOnly && order.TimeInForce != TIME_IN_FORCE_POST_ONLY {
            continue
        }

        // 第二遍: 跳过 POST_ONLY 订单 (已在第一遍处理)
        if !postOnly && order.TimeInForce == TIME_IN_FORCE_POST_ONLY {
            continue
        }

        // 放置订单到 MemClob
        k.MemClob.PlaceOrder(ctx, order)
    }
}
```

---

#### 步骤 7: 放置已触发的条件订单

```go
k.PlaceConditionalOrdersTriggeredInLastBlock(ctx)
```

**作用**:
- 放置在上一个区块触发的条件订单
- 这些订单现在转为普通限价单,参与匹配

---

### 为什么需要重放操作?

**问题**: 验证器在 CheckTx 阶段乐观处理订单,但这些操作可能与 FinalizeBlock 的最终结果不一致。

**场景示例**:
```
CheckTx 阶段 (本地验证器):
1. 收到订单 A,乐观匹配,放入 MemClob
2. 收到订单 B,乐观匹配,与 A 成交

FinalizeBlock 阶段 (共识后):
1. 区块中仅包含订单 A (订单 B 未被打包)
2. 订单 A 放入订单簿,未成交

PrepareCheckState 阶段:
1. 清空本地 MemClob
2. 重放 FinalizeBlock 的操作 (仅订单 A)
3. 本地状态与链上状态同步 ✅
```

**重放操作的内容**:
- 从上一个区块 `FinalizeBlock` 执行的操作中提取
- 包括所有已确认的订单放置、匹配、取消操作
- 确保本地 MemClob 与链上状态完全一致

---

### PrepareCheckState 完整流程图

```
Commit 之后
     │
     ▼
┌────────────────────────────────────────┐
│ 1. 清理速率限制信息                      │
└────────┬───────────────────────────────┘
         │
         ▼
┌────────────────────────────────────────┐
│ 2. 获取上一区块的操作队列                │
│    (订单放置、匹配、取消)                 │
└────────┬───────────────────────────────┘
         │
         ▼
┌────────────────────────────────────────┐
│ 3. 清空本地操作队列和无效状态             │
└────────┬───────────────────────────────┘
         │
         ▼
┌────────────────────────────────────────┐
│ 4. 第一遍: 放置 POST_ONLY 订单          │
│    (确保它们在订单簿中,作为 Maker)        │
└────────┬───────────────────────────────┘
         │
         ▼
┌────────────────────────────────────────┐
│ 5. 第二遍: 放置其他 Long-Term 订单       │
│    (可以与 POST_ONLY 订单匹配)           │
└────────┬───────────────────────────────┘
         │
         ▼
┌────────────────────────────────────────┐
│ 6. 放置已触发的条件订单                   │
└────────┬───────────────────────────────┘
         │
         ▼
    下一个区块 CheckTx 开始
```

---

### 设计权衡

**优势**:
- ✅ **状态一致性**: 确保本地 MemClob 与链上 StateStore 完全一致
- ✅ **POST_ONLY 语义正确性**: 两遍放置确保 POST_ONLY 订单总是 Maker
- ✅ **确定性保证**: 所有节点以相同方式重放操作,结果一致

**劣势**:
- ❌ **性能开销**: 需要两遍遍历订单,增加计算成本
- ❌ **复杂性**: 两遍放置逻辑增加代码复杂度

**决策理由**:
- **正确性优先**: POST_ONLY 语义是产品需求,必须保证
- **性能可接受**: 订单数量有限 (每个账户每侧最多 20 个),两遍遍历开销可控
- **确定性必需**: 共识要求所有节点状态一致,重放操作是必要的

---

## 3. 核心业务流程

### 3.1 订单生命周期

#### 3.1.1 Short-Term 订单流程

```
用户提交 Short-Term 订单
         │
         ▼
    CheckTx (所有节点)
         ├─ 验证订单格式
         ├─ 检查保证金充足
         ├─ 存储到 TransientStore
         ├─ 放入 MemClob
         └─ 乐观匹配
         │
         ▼
    Flood Gossip 传播
         │
         ▼
 PrepareProposal (仅 Proposer)
         ├─ 获取 operationsToPropose
         └─ 构建 MsgProposedOperations
         │
         ▼
    ProcessProposal (所有节点)
         └─ 验证操作有效性
         │
         ▼
FinalizeBlock - DeliverTx (所有节点)
         ├─ ProposedOperations 执行匹配
         ├─ 更新账户余额
         ├─ 记录成交事件
         └─ 更新订单簿状态
         │
         ▼
FinalizeBlock - EndBlocker
         └─ TransientStore 自动清空
                   │
                   ▼
            订单生命周期结束
```

**为什么 Short-Term 和 Long-Term 订单分开处理?**

**原因分析**:

**Short-Term 订单特点**:
- 有效期仅 1 个区块
- 主要用于做市和高频交易
- 数量大,频繁创建和销毁

**Long-Term 订单特点**:
- 有效期跨多个区块 (GTT)
- 主要用于限价单
- 数量相对较少,生命周期长

**分开处理的理由**:
1. **存储优化**: Short-Term 订单不需要持久化,节省链上存储
2. **性能优化**: TransientStore 自动清理,无需手动清理逻辑
3. **逻辑简化**: Short-Term 订单无需处理过期逻辑

**设计权衡**:
- ✅ 优势: 减少链上存储,提升性能
- ❌ 劣势: 需要维护两套订单处理逻辑
- 决策理由: 性能和存储优化优先

#### 3.1.2 Long-Term 订单流程

```
用户提交 Long-Term 订单
         │
         ▼
    CheckTx (所有节点)
         ├─ 验证订单格式
         ├─ 检查保证金充足
         ├─ 放入 MemClob
         └─ 乐观匹配
         │
         ▼
FinalizeBlock - DeliverTx (所有节点)
         ├─ ProposedOperations 执行匹配
         ├─ 更新账户余额
         ├─ 持久化到 StateStore
         └─ 记录成交事件
         │
         ├─ 未完全成交?
         │    ├─ 是 → 保留在 StateStore
         │    └─ 否 → 从 StateStore 删除
         │
         ▼
    后续区块继续匹配
         │
         ▼
FinalizeBlock - EndBlocker
         └─ 检查是否过期 (GoodTilBlockTime)
                   │
                   ├─ 是 → 删除订单
                   └─ 否 → 保留订单
                             │
                             ▼
                      订单生命周期结束
```

#### 3.1.3 Conditional 订单流程

```
用户提交 Conditional 订单
         │
         ▼
    CheckTx (所有节点)
         ├─ 验证订单格式
         ├─ 检查保证金充足
         └─ 存储到 UntriggeredConditionalOrderKeyPrefix
         │
         ▼
     每个区块 EndBlocker
         ├─ 检查触发条件 (价格条件)
         │    ├─ 满足? → 转移到 TriggeredConditionalOrderKeyPrefix
         │    └─ 不满足? → 保持在 Untriggered
         │
         ▼
    Triggered 订单
         ├─ 下一个区块放入 MemClob
         ├─ 参与匹配
         └─ 匹配成功后删除
                   │
                   ▼
            订单生命周期结束
```

**为什么 Conditional 订单需要两个存储位置(触发/未触发)?**

**原因分析**:

**业务需求**:
- 条件订单在触发前不应参与匹配
- 触发后需要立即参与匹配

**设计方案**:
1. **未触发状态** (UntriggeredConditionalOrderKeyPrefix):
   - 等待触发条件满足
   - 每个区块 EndBlocker 检查价格条件
   - 不参与订单匹配

2. **触发状态** (TriggeredConditionalOrderKeyPrefix):
   - 触发条件已满足
   - 下一个区块放入 MemClob
   - 参与正常匹配流程

**设计权衡**:
- ✅ 优势: 逻辑清晰,状态分离
- ❌ 劣势: 需要维护两个存储位置,状态转移逻辑复杂
- 决策理由: 业务逻辑需要明确区分触发前后状态

#### 3.1.4 TWAP 订单流程

```
用户提交 TWAP 订单
         │
         ▼
    CheckTx (所有节点)
         ├─ 验证订单格式
         ├─ 检查保证金充足
         └─ 持久化到 StateStore (TWAPOrderPlacement)
         │
         ▼
     每个区块 EndBlocker
         ├─ 检查时间窗口
         ├─ 判断是否生成子订单
         │    ├─ 是 → 生成 Short-Term 子订单
         │    │       └─ 放入 MemClob 参与匹配
         │    └─ 否 → 等待下一个区块
         │
         ▼
    子订单匹配成功
         ├─ 更新 TWAP 订单已成交数量
         ├─ 检查是否全部成交
         │    ├─ 是 → 删除 TWAP 订单
         │    └─ 否 → 继续等待生成子订单
         │
         ▼
    全部子订单成交
         └─ 订单生命周期结束
```

**TWAP 订单的自动分解机制设计理由**

**业务需求**:
- 大额订单直接成交可能导致价格剧烈波动 (市场冲击)
- 需要将大订单拆分为多个小订单,在时间窗口内逐步执行

**设计方案**:
1. **时间窗口划分**: 将总执行时间划分为多个时间窗口
2. **子订单生成**: 每个时间窗口生成一个子订单
3. **随机化起始时间**: 防止被预测和抢跑

**自动分解逻辑**:
```
TWAP 订单参数:
- TotalQuantity: 100 BTC
- TimeWindow: 60 分钟
- NumSubOrders: 10

每 6 分钟生成一个子订单:
- SubOrder1: 10 BTC (区块 X)
- SubOrder2: 10 BTC (区块 X+360)
- ...
- SubOrder10: 10 BTC (区块 X+3240)
```

**设计权衡**:
- ✅ 优势: 降低市场冲击,保护大额订单
- ❌ 劣势: 执行时间长,可能无法完全成交
- 决策理由: 大额订单场景需要保护,时间换价格

### 3.2 订单匹配引擎

#### 3.2.1 价格-时间优先算法

**算法描述**:

1. **价格优先**:
   - 买单: 价格高的优先匹配
   - 卖单: 价格低的优先匹配

2. **时间优先**:
   - 相同价格下,时间早的订单优先匹配

**实现细节**:

```
买单排序:
[100 USDC, 时间1] ← 最优先
[100 USDC, 时间2]
[99 USDC, 时间1]

卖单排序:
[99 USDC, 时间1] ← 最优先
[100 USDC, 时间1]
[100 USDC, 时间2]

匹配规则:
if 买单最高价 >= 卖单最低价:
    成交价格 = min(买单价格, 卖单价格)
    成交数量 = min(买单数量, 卖单数量)
```

#### 3.2.2 确定性保证机制

**为什么确定性如此重要?**

**共识要求**:
- 所有验证节点必须产生完全相同的状态转换
- 不同节点产生不同结果会导致共识失败
- 区块链网络会分叉

**确定性保证措施**:

1. **订单排序确定性**:
   ```go
   // 使用 OrderId 作为稳定排序的二级 key
   type OrderSorting struct {
       Price    uint64
       OrderId  OrderId  // 唯一且确定性
   }
   ```

2. **避免不确定性因素**:
   - ❌ 不使用 `rand.Random()` (随机数)
   - ❌ 不使用 `time.Now()` (当前时间戳)
   - ❌ 不使用 `map` 迭代 (Go map 迭代顺序不确定)
   - ✅ 使用 `ctx.BlockTime()` (共识保证的区块时间)
   - ✅ 使用 `BlockHeight()` (共识保证的区块高度)

3. **匹配顺序确定性**:
   ```
   匹配顺序严格按照 ProposedOperations 中的顺序执行
   - Operation 1: PlaceOrder
   - Operation 2: MatchOrders
   - Operation 3: CancelOrder
   顺序固定,所有节点一致
   ```

4. **测试确定性**:
   ```go
   // 运行多次,验证结果一致
   for i := 0; i < 100; i++ {
       result := keeper.MatchOrders(ctx, orders)
       assert.Equal(t, expectedResult, result)
   }
   ```

**设计权衡**:
- ✅ 优势: 共识保证,系统稳定
- ❌ 劣势: 限制实现灵活性 (不能使用某些语言特性)
- 决策理由: 共识是区块链核心要求,必须严格保证

#### 3.2.3 部分成交处理逻辑

**场景**:
```
买单: 100 USDC, 数量 10 BTC
卖单: 100 USDC, 数量 5 BTC

成交结果:
- 成交数量: 5 BTC
- 买单剩余: 5 BTC (未完全成交)
- 卖单完全成交,从订单簿移除
```

**处理逻辑**:

1. **记录成交量**:
   ```go
   keeper.SetOrderFillAmount(ctx, orderId, filledAmount)
   ```

2. **更新订单簿**:
   ```
   - 完全成交订单: 从订单簿移除
   - 部分成交订单: 更新剩余数量,保留在订单簿
   ```

3. **账户余额更新**:
   ```
   买方:
   - USDC 减少: 成交数量 × 成交价格
   - BTC 增加: 成交数量

   卖方:
   - BTC 减少: 成交数量
   - USDC 增加: 成交数量 × 成交价格
   ```

#### 3.2.4 匹配性能优化

**性能目标**:
- 每秒处理 1000+ 笔订单
- 匹配延迟 < 100ms

**优化措施**:

1. **价格层级聚合**:
   ```
   不存储: [100.01, 100.02, 100.03, ...]
   存储:   [100 USDC 档位, 总数量 1000 BTC]
   优势: 减少内存占用,加快匹配速度
   ```

2. **订单簿平衡树**:
   ```
   使用 Red-Black Tree 维护订单簿:
   - 插入: O(log n)
   - 删除: O(log n)
   - 查找最优价格: O(1)
   ```

3. **批量匹配**:
   ```
   单个 ProposedOperations 包含多个订单:
   - 减少交易数量
   - 降低 Gas 消耗
   - 提升吞吐量
   ```

4. **内存池复用**:
   ```go
   // 复用订单对象,减少 GC 压力
   var orderPool = sync.Pool{
       New: func() interface{} {
           return &Order{}
       },
   }
   ```

**设计权衡**:
- ✅ 优势: 显著提升性能
- ❌ 劣势: 增加代码复杂度
- 决策理由: 性能是交易所核心竞争力

### 3.3 清算流程

#### 3.3.1 触发条件判断

**清算触发条件**:
```
保证金率 = 净抵押品价值 / 仓位名义价值

如果 保证金率 < 维持保证金率 (MMR) → 触发清算
```

**监控流程**:
```
链下 Daemon (清算机器人)
    ├─ 订阅区块事件
    ├─ 监控所有子账户的保证金率
    ├─ 发现保证金率 < MMR 的账户
    ├─ 构建清算订单
    └─ 提交到链上 (ProposedOperations)
         │
         ▼
链上验证 (CLOB Keeper)
    ├─ 验证账户确实水下
    ├─ 验证清算订单有效性
    └─ 执行清算匹配
```

#### 3.3.2 清算执行流程

**清算订单特点**:
- **强制执行**: 不需要用户授权
- **市价成交**: 以市场最优价格成交
- **全额平仓**: 清空所有头寸

**执行步骤**:

1. **构建清算订单**:
   ```go
   liquidationOrder := Order{
       OrderId: GenerateLiquidationOrderId(subaccountId),
       Side: OppositeOf(position.Side),
       Quantums: position.Quantums,  // 全部平仓
       Subticks: 0,  // 市价单
       TimeInForce: IOC,  // 立即成交或取消
   }
   ```

2. **匹配清算订单**:
   ```
   MemClob.PlaceLiquidationOrder(liquidationOrder)
   ├─ 与市场最优价格的订单匹配
   ├─ 优先保证清算成功
   └─ 生成成交事件
   ```

3. **保险基金处理**:
   ```
   清算盈亏 = 成交价格 × 成交数量 - 仓位成本

   if 清算盈亏 > 0:
       盈利归保险基金
   else:
       亏损由保险基金弥补
   ```

4. **记录清算事件**:
   ```go
   k.EmitLiquidationEvent(ctx, LiquidationEvent{
       SubaccountId: subaccountId,
       LiquidatedQuantums: filledQuantums,
       InsuranceFundDelta: insuranceFundDelta,
   })
   ```

#### 3.3.3 Deleveraging (去杠杆) 机制

**触发条件**:
```
清算失败 OR 保险基金不足
    ↓
触发 Deleveraging
```

**去杠杆流程**:

1. **选择去杠杆对象**:
   ```
   选择标准:
   - 持有相反头寸的盈利账户
   - 按盈利率排序 (盈利率最高的优先)
   - 强制平仓盈利头寸,弥补清算亏损
   ```

2. **强制平仓**:
   ```go
   deleverageOrder := Order{
       SubaccountId: profitableSubaccount,
       Side: OppositeOf(liquidatedPosition.Side),
       Quantums: deleverageQuantums,
       Subticks: bankruptcyPrice,  // 破产价格成交
       TimeInForce: FOK,
   }
   ```

3. **损失分摊**:
   ```
   被去杠杆账户:
   - 强制平仓盈利头寸
   - 以破产价格成交 (可能低于市场价)
   - 损失 = (市场价 - 破产价) × 成交数量
   ```

**为什么需要保险基金?**

**原因分析**:

**市场风险**:
- 价格剧烈波动时,清算可能亏损
- 清算亏损需要有资金池弥补

**系统稳定性**:
- 保险基金吸收清算亏损
- 避免系统性风险传递给其他用户

**Deleveraging 作为最后手段**:
- 保险基金不足时才触发
- 强制平仓盈利账户,分摊损失

**设计权衡**:
- ✅ 优势: 系统稳定,风险隔离
- ❌ 劣势: 被去杠杆用户体验差
- 决策理由: 系统稳定性优先

### 3.4 MEV 保护机制

MEV (Maximal Extractable Value) 是指矿工/验证者通过重排序、插入、审查交易来提取的额外价值。在 DEX 中,MEV 主要表现为抢跑 (Front-Running) 和三明治攻击 (Sandwich Attack)。

#### 3.4.1 速率限制策略设计

**速率限制类型**:

1. **Short-Term 订单速率限制**:
   ```
   限制: 每个区块最多 N 个 Short-Term 订单
   目的: 防止垃圾订单攻击
   实现: TransientStore 计数器
   ```

2. **订单取消速率限制**:
   ```
   限制: 每 M 个区块最多 K 次订单取消
   目的: 防止恶意取消订单干扰市场
   实现: 滑动窗口计数
   ```

3. **杠杆更新速率限制**:
   ```
   限制: 每 N 个区块最多 1 次杠杆更新
   目的: 防止频繁调整杠杆规避风控
   实现: 区块高度记录
   ```

**实现机制**:

```go
func (k Keeper) RateLimitPlaceOrder(
    ctx sdk.Context,
    order *types.Order,
) error {
    // 获取当前区块已放置订单数
    count := k.GetShortTermOrderCount(ctx, order.OrderId.SubaccountId)

    // 检查是否超过限制
    if count >= k.GetMaxShortTermOrdersPerNBlocks(ctx) {
        return types.ErrRateLimitExceeded
    }

    // 增加计数器
    k.IncrementShortTermOrderCount(ctx, order.OrderId.SubaccountId)

    return nil
}
```

**为什么需要多种速率限制?**

**原因分析**:
- 不同操作有不同的攻击向量
- 单一速率限制无法覆盖所有场景
- 分层限制提供更细粒度的保护

**设计权衡**:
- ✅ 优势: 多层防护,灵活调整
- ❌ 劣势: 增加实现复杂度
- 决策理由: 安全性优先

#### 3.4.2 权益层级限制设计

**权益层级限制机制**:

```
根据子账户权益分配订单额度:

权益层级              订单额度限制
───────────────────────────────────
< 1,000 USDC        10 个订单/区块
1,000 - 10,000      50 个订单/区块
10,000 - 100,000    200 个订单/区块
> 100,000           1000 个订单/区块
```

**实现逻辑**:

```go
func (k Keeper) GetOrderLimitForSubaccount(
    ctx sdk.Context,
    subaccountId types.SubaccountId,
) uint32 {
    // 获取子账户权益
    equity := k.subaccountsKeeper.GetNetCollateral(ctx, subaccountId)

    // 根据权益层级返回订单限制
    for _, tier := range k.GetEquityTierLimitConfiguration(ctx).Tiers {
        if equity >= tier.MinEquity && equity < tier.MaxEquity {
            return tier.OrderLimit
        }
    }

    return 0 // 无权益账户禁止下单
}
```

**为什么需要权益层级限制?**

**原因分析**:

**Sybil 攻击防护**:
- 攻击者可以创建大量低权益账户
- 每个账户发送少量订单
- 总体形成垃圾订单攻击

**权益层级限制作用**:
- 限制低权益账户的订单额度
- 攻击者需要投入大量资金才能发动攻击
- 提高攻击成本

**设计权衡**:
- ✅ 优势: 提高攻击成本,保护系统
- ❌ 劣势: 限制小额用户体验
- 决策理由: 系统安全性优先

#### 3.4.3 确定性匹配的 MEV 防护作用

**MEV 攻击类型**:

1. **Front-Running (抢跑)**:
   ```
   用户订单: 买入 BTC, 价格 100 USDC
   攻击者: 看到用户订单后,先买入 BTC, 价格 99 USDC
   结果: 攻击者以低价买入,推高价格,然后卖给用户
   ```

2. **Sandwich Attack (三明治攻击)**:
   ```
   用户订单: 买入 BTC, 价格 100 USDC
   攻击者:
   - 前置订单: 买入 BTC, 价格 99 USDC
   - 后置订单: 卖出 BTC, 价格 101 USDC
   结果: 攻击者夹击用户订单,获取价差利润
   ```

**确定性匹配如何防护 MEV?**

**ProposedOperations 机制**:
```
传统 DEX (以太坊):
- 矿工/验证者可以任意重排序交易
- 攻击者贿赂矿工,插入攻击交易

Hermes DEX:
- 提议者构建 ProposedOperations 后无法修改
- 所有验证者验证 ProposedOperations 一致性
- 任何篡改都会导致共识失败
```

**确定性匹配保证**:
```
订单匹配顺序完全确定:
1. ProposedOperations 中的操作顺序固定
2. 所有节点产生相同的匹配结果
3. 无法通过重排序获得 MEV
```

**剩余 MEV 问题**:
```
提议者仍然可以:
- 选择包含哪些订单
- 决定订单在 ProposedOperations 中的顺序

缓解措施:
- 提议者轮换 (CometBFT 机制)
- 订单加密 (计划中)
- 批量拍卖 (可配置)
```

**设计权衡**:
- ✅ 优势: 显著降低 MEV 风险
- ❌ 劣势: 无法完全消除提议者 MEV
- 决策理由: 在现有技术下提供最佳保护

---

## 4. 模块依赖关系

### 4.1 上游依赖

CLOB 模块依赖以下模块的 Keeper:

#### 4.1.1 Subaccounts Keeper

**依赖原因**: 保证金计算和余额管理

**调用场景**:
- **订单验证**: 检查账户是否有足够保证金支持订单
- **订单成交**: 更新买卖双方的账户余额
- **清算判断**: 计算账户保证金率,判断是否触发清算

**关键接口**:
```go
// 获取账户净抵押品
GetNetCollateral(ctx, subaccountId) -> int64

// 检查保证金充足性
CanPlaceOrder(ctx, subaccountId, order) -> bool

// 更新账户余额 (成交后)
UpdateSubaccounts(ctx, fills) -> error

// 计算保证金要求
GetMarginRequirements(ctx, subaccountId) -> (initialMargin, maintenanceMargin)
```

#### 4.1.2 Perpetuals Keeper

**依赖原因**: 永续合约配置和资金费率

**调用场景**:
- **订单验证**: 获取永续合约的配置参数
- **保证金计算**: 获取合约的初始保证金率 (IMR) 和维持保证金率 (MMR)
- **清算判断**: 获取流动性层级配置

**关键接口**:
```go
// 获取永续合约配置
GetPerpetual(ctx, perpetualId) -> Perpetual

// 获取流动性层级
GetLiquidityTier(ctx, liquidityTierId) -> LiquidityTier

// 获取未平仓合约量
GetOpenInterest(ctx, perpetualId) -> int64
```

#### 4.1.3 Prices Keeper

**依赖原因**: 价格预言机数据

**调用场景**:
- **清算判断**: 获取市场价格,计算账户净值
- **条件订单触发**: 检查价格是否满足触发条件
- **保证金计算**: 使用市场价格计算仓位价值

**关键接口**:
```go
// 获取市场价格
GetMarketPrice(ctx, marketId) -> MarketPrice

// 获取所有市场价格
GetAllMarketPrices(ctx) -> []MarketPrice
```

#### 4.1.4 Stats Keeper

**依赖原因**: 交易统计和手续费分级

**调用场景**:
- **订单成交**: 记录成交量和手续费
- **手续费计算**: 根据用户交易量计算手续费折扣

**关键接口**:
```go
// 记录成交
RecordFill(ctx, fill) -> error

// 获取用户统计
GetUserStats(ctx, userAddress) -> UserStats
```

#### 4.1.5 其他依赖

**Assets Keeper**:
- 获取资产配置 (精度、最小交易量等)

**Bank Keeper**:
- 查询账户余额 (如果需要)

**FeeCollector**:
- 收取交易手续费

**IndexerEventManager**:
- 发送索引器事件 (成交记录、订单簿更新等)

### 4.2 下游被依赖

#### 4.2.1 Stats 模块

**依赖原因**: 接收成交记录

**调用场景**:
- CLOB 订单成交后,调用 `Stats.RecordFill()` 记录成交数据

#### 4.2.2 Vault 模块

**依赖原因**: Vault 自动做市

**调用场景**:
- Vault 调用 `CLOB.PlaceOrder()` 放置做市订单
- Vault 调用 `CLOB.CancelOrder()` 取消过期订单
- Vault 调用 `CLOB.ReplaceOrder()` 刷新订单

#### 4.2.3 Indexer

**依赖原因**: 链下数据索引

**调用场景**:
- CLOB 发出订单匹配事件,Indexer 监听并索引
- 用于前端查询订单历史、成交记录等

---

## 5. 数据流设计

### 5.1 订单数据流

```
                    订单完整数据流

┌─────────────────────────────────────────────────────────┐
│                    用户发起订单                          │
└───────────────────────┬─────────────────────────────────┘
                        │
            ┌───────────┴──────────┐
            │   CheckTx (所有节点)  │
            │  ├─ 验证订单格式      │
            │  ├─ 检查保证金       │
            │  └─ 初步验证         │
            └───────────┬──────────┘
                        │
        ┌───────────────┼───────────────┐
        │               │               │
        ▼               ▼               ▼
 [Short-Term]    [Long-Term]    [Conditional]
        │               │               │
TransientStore   StateStore      StateStore
        │               │               │
        │               │               │
        └───────────────┼───────────────┘
                        │
            ┌───────────┴──────────┐
            │      MemClob          │
            │  ├─ 放入订单簿       │
            │  ├─ 乐观匹配         │
            │  └─ 加入操作队列     │
            └───────────┬──────────┘
                        │
            ┌───────────┴──────────┐
            │ PrepareProposal       │
            │ (仅 Proposer)         │
            │ 构建 ProposedOperations│
            └───────────┬──────────┘
                        │
            ┌───────────┴──────────┐
            │ ProcessProposal       │
            │ (所有节点)            │
            │ 验证操作有效性        │
            └───────────┬──────────┘
                        │
            ┌───────────┴──────────┐
            │ FinalizeBlock         │
            │ DeliverTx (ProposedOps)│
            │  ├─ 确定性匹配       │
            │  ├─ 更新账户余额     │
            │  ├─ 记录成交量       │
            │  └─ 发出事件         │
            └───────────┬──────────┘
                        │
        ┌───────────────┼───────────────┐
        │               │               │
        ▼               ▼               ▼
   完全成交        部分成交         未成交
        │               │               │
  从订单簿移除     保留在订单簿      保留在订单簿
        │               │               │
        │               │               │
        └───────────────┼───────────────┘
                        │
            ┌───────────┴──────────┐
            │    EndBlocker         │
            │  ├─ 检查过期         │
            │  ├─ 清理状态         │
            │  └─ 更新订单簿       │
            └───────────┬──────────┘
                        │
                订单生命周期结束
```

### 5.2 匹配事件数据流

```
                匹配事件数据流

┌─────────────────────────────────────────────────────────┐
│           订单匹配 (DeliverTx - ProposedOps)             │
│        MatchOrders(buyOrder, sellOrder)                 │
└───────────────────────┬─────────────────────────────────┘
                        │
            ┌───────────┴──────────┐
            │  生成 Fill (成交记录)  │
            │  ├─ TakerOrder         │
            │  ├─ MakerOrder         │
            │  ├─ FillQuantums       │
            │  ├─ FillSubticks       │
            │  └─ Fee                │
            └───────────┬──────────┘
                        │
        ┌───────────────┼───────────────┐
        │               │               │
        ▼               ▼               ▼
ProcessProposerMatchesEvents  TransientStore  Stats.RecordFill()
        │               │               │
 (MemStore 存储)   (临时存储)      (记录统计)
        │               │               │
        └───────────────┼───────────────┘
                        │
            ┌───────────┴──────────┐
            │    Precommit          │
            │  ProcessStagedEvents  │
            └───────────┬──────────┘
                        │
        ┌───────────────┼───────────────┐
        │               │               │
        ▼               ▼               ▼
Indexer Event    gRPC Stream      日志输出
        │               │               │
 (链下索引)       (实时推送)       (调试)
        │               │               │
        └───────────────┼───────────────┘
                        │
                  事件消费完成
```

### 5.3 清算数据流

```
                    清算完整数据流

┌─────────────────────────────────────────────────────────┐
│              链下 Daemon (清算机器人)                     │
│  ├─ 订阅区块事件                                        │
│  ├─ 监控所有子账户保证金率                              │
│  └─ 计算: 净抵押品 / 仓位价值 < MMR?                    │
└───────────────────────┬─────────────────────────────────┘
                        │
                    是,触发清算
                        │
            ┌───────────┴──────────┐
            │  构建清算订单          │
            │  LiquidationOrder {   │
            │    Subaccount,        │
            │    OppositeDirection, │
            │    FullSize,          │
            │    MarketOrder        │
            │  }                    │
            └───────────┬──────────┘
                        │
            ┌───────────┴──────────┐
            │ 提交到链上             │
            │ MsgProposedOperations │
            │ (包含清算订单)         │
            └───────────┬──────────┘
                        │
            ┌───────────┴──────────┐
            │ FinalizeBlock         │
            │ DeliverTx (ProposedOps)│
            │  ├─ 验证账户确实水下  │
            │  ├─ 验证清算订单有效  │
            │  └─ 执行匹配          │
            └───────────┬──────────┘
                        │
        ┌───────────────┼───────────────┐
        │               │               │
        ▼               ▼               ▼
   清算成功         部分成交        清算失败
        │               │               │
 保险基金处理     继续清算         触发 Deleveraging
        │               │               │
        │               │               │
        └───────────────┼───────────────┘
                        │
            ┌───────────┴──────────┐
            │   记录清算事件        │
            │  LiquidationEvent {   │
            │    Subaccount,        │
            │    Quantums,          │
            │    InsuranceFundDelta │
            │  }                    │
            └───────────┬──────────┘
                        │
        ┌───────────────┼───────────────┐
        │               │               │
        ▼               ▼               ▼
 Indexer 索引     Stats 统计     Daemon 确认
        │               │               │
  (清算记录)        (清算量)        (停止监控)
        │               │               │
        └───────────────┼───────────────┘
                        │
                  清算流程结束
```

---

## 6. 性能与可扩展性

### 6.1 性能优化点

#### 6.1.1 MemClob 内存优化

**内存使用分析**:

```
单个订单内存占用:
- OrderId: 32 字节
- Price: 8 字节
- Quantity: 8 字节
- Metadata: ~100 字节
总计: ~150 字节/订单

假设订单簿容量:
- 10,000 个活跃订单
- 总内存: 150 * 10,000 = 1.5 MB
```

**优化措施**:

1. **对象池 (Object Pool)**:
   ```go
   var orderPool = sync.Pool{
       New: func() interface{} {
           return &Order{}
       },
   }

   // 复用订单对象,减少 GC 压力
   order := orderPool.Get().(*Order)
   defer orderPool.Put(order)
   ```

2. **内存预分配**:
   ```go
   // 预分配订单簿容量
   orderbook := make([]*Order, 0, 10000)
   ```

3. **紧凑数据结构**:
   ```go
   // 使用位域压缩数据
   type CompactOrder struct {
       OrderIdAndSide uint64  // 高 32 位: OrderId, 低 32 位: Side
       PriceAndQty    uint64  // 高 32 位: Price, 低 32 位: Quantity
   }
   ```

#### 6.1.2 批量操作优化

**单笔操作问题**:
- 每笔订单单独验证和执行
- Gas 消耗高
- 吞吐量受限

**批量操作优化**:

```go
// ProposedOperations 批量处理
func ProcessBatchOrders(ctx sdk.Context, orders []Order) error {
    // 批量验证
    for _, order := range orders {
        if err := ValidateOrder(order); err != nil {
            return err
        }
    }

    // 批量匹配
    matches := MemClob.MatchBatchOrders(ctx, orders)

    // 批量更新账户余额
    subaccountsKeeper.UpdateBatchSubaccounts(ctx, matches)

    return nil
}
```

**优化效果**:
- 单笔处理: 1000 Gas/订单
- 批量处理: 500 Gas/订单 (50% 节省)

#### 6.1.3 索引设计优化

**索引类型**:

1. **主键索引**:
   ```
   OrderId → Order
   O(1) 查找
   ```

2. **二级索引**:
   ```
   SubaccountId → []OrderId
   ClobPairId → []OrderId
   快速查找账户或交易对的所有订单
   ```

3. **价格索引** (MemClob):
   ```
   Red-Black Tree:
   Price → []Order (相同价格的订单列表)
   ```

**索引更新策略**:
- 订单放置: 更新所有索引
- 订单取消: 仅更新相关索引
- 订单匹配: 批量更新索引

### 6.2 扩展性设计

#### 6.2.1 如何支持更多订单类型?

**扩展步骤**:

1. **定义新订单类型**:
   ```protobuf
   // proto/hermes/clob/order.proto
   message IcebergOrder {
       Order visible_order = 1;    // 显示部分
       uint64 hidden_quantity = 2;  // 隐藏部分
       uint64 reveal_increment = 3; // 每次显示增量
   }
   ```

2. **实现验证逻辑**:
   ```go
   func ValidateIcebergOrder(order *IcebergOrder) error {
       // 验证显示部分 + 隐藏部分 = 总数量
       // 验证增量合理性
   }
   ```

3. **实现匹配逻辑**:
   ```go
   func MatchIcebergOrder(order *IcebergOrder) {
       // 成交显示部分后,自动显示下一批隐藏数量
   }
   ```

4. **添加测试**:
   ```go
   func TestIcebergOrderMatching(t *testing.T) {
       // 测试隐藏数量逐步显示
       // 测试完全成交逻辑
   }
   ```

#### 6.2.2 如何支持更高吞吐量?

**扩展方案**:

1. **垂直扩展** (单节点性能提升):
   - 优化匹配算法复杂度
   - 使用更快的数据结构
   - 增加节点内存和 CPU

2. **水平扩展** (增加节点数量):
   - CometBFT 支持增加验证者节点
   - 节点数量增加不会影响性能 (共识开销增加)

3. **分片** (未来可能):
   - 按交易对分片,不同交易对在不同分片
   - 分片间通过跨片通信

### 6.3 瓶颈分析

#### 6.3.1 已知瓶颈

**区块 Gas 限制**:
- 问题: 单个区块 Gas 有限,限制订单匹配数量
- 影响: 高峰期订单可能积压
- 缓解: 提高区块 Gas 限制 (治理参数)

**MemClob 内存消耗**:
- 问题: 订单数量增加,内存消耗增加
- 影响: 节点内存不足可能导致崩溃
- 缓解: 限制订单簿深度,自动修剪

**确定性验证开销**:
- 问题: 所有节点都需要验证匹配结果
- 影响: 验证节点 CPU 消耗高
- 缓解: 优化验证逻辑,使用缓存

#### 6.3.2 潜在瓶颈

**订单簿深度增加**:
- 问题: 订单簿过深,匹配遍历时间长
- 影响: 匹配延迟增加
- 缓解: 限制订单簿深度,价格聚合

**清算风暴**:
- 问题: 极端行情下,大量账户同时触发清算
- 影响: 区块 Gas 不足,清算积压
- 缓解: 清算优先级排序,分批处理

**状态膨胀**:
- 问题: 长期订单累积,StateStore 膨胀
- 影响: 节点存储和同步开销增加
- 缓解: 定期修剪过期订单,状态压缩

#### 6.3.3 缓解策略

**动态 Gas 限制调整**:
```
根据订单数量动态调整区块 Gas 限制
if 订单积压 > 阈值:
    提高区块 Gas 限制 (治理提案)
```

**订单簿深度限制**:
```
每个价格层级最多保留 N 个订单
超过限制的订单拒绝放置或自动取消
```

**状态修剪策略**:
```
EndBlocker 定期清理:
- 过期的 Long-Term 订单
- 完全成交的订单填充量记录
- 过期的清算信息
```

---

## 7. 安全与风险控制

### 7.1 MEV 保护机制

(详见 3.4 节)

**核心措施**:
- 确定性匹配
- 速率限制
- 权益层级限制
- 订单加密 (计划中)

### 7.2 速率限制策略

(详见 3.4.1 节)

**限制类型**:
- Short-Term 订单速率限制
- 订单取消速率限制
- 杠杆更新速率限制

### 7.3 确定性保证

(详见 3.2.2 节)

**保证措施**:
- 订单排序确定性
- 避免不确定性因素
- 匹配顺序确定性
- 测试确定性

### 7.4 其他安全措施

#### 7.4.1 保证金验证

```go
func (k Keeper) ValidateOrderCollateral(
    ctx sdk.Context,
    order *types.Order,
) error {
    // 计算订单所需保证金
    requiredMargin := k.CalculateRequiredMargin(ctx, order)

    // 获取账户可用保证金
    availableMargin := k.subaccountsKeeper.GetAvailableMargin(ctx, order.OrderId.SubaccountId)

    // 验证保证金充足
    if availableMargin < requiredMargin {
        return types.ErrInsufficientCollateral
    }

    return nil
}
```

#### 7.4.2 订单格式验证

```go
func ValidateOrder(order *Order) error {
    // 验证订单 ID
    if order.OrderId == nil {
        return ErrInvalidOrderId
    }

    // 验证数量 > 0
    if order.Quantums == 0 {
        return ErrInvalidQuantity
    }

    // 验证价格 >= 0 (市价单允许 0)
    if order.Subticks < 0 {
        return ErrInvalidPrice
    }

    // 验证数量和价格符合步长要求
    clobPair := k.GetClobPair(ctx, order.OrderId.ClobPairId)
    if order.Quantums % clobPair.StepBaseQuantums != 0 {
        return ErrQuantityNotMultipleOfStepSize
    }

    return nil
}
```

#### 7.4.3 权限验证

```go
func (k Keeper) ValidateOrderAuthority(
    ctx sdk.Context,
    order *types.Order,
    signer string,
) error {
    // 验证签名者是订单所有者
    subaccount := k.subaccountsKeeper.GetSubaccount(ctx, order.OrderId.SubaccountId)
    if subaccount.Owner != signer {
        return types.ErrUnauthorized
    }

    return nil
}
```

---

## 8. 技术决策记录 (ADR)

### 8.1 为什么使用 MemClob?

**问题**: 订单匹配引擎应该如何设计?

**备选方案**:

1. **方案 A: 纯链上订单簿** (直接使用 KVStore)
   - ✅ 优势: 实现简单,状态持久化
   - ❌ 劣势: 性能差 (磁盘 I/O 慢),Gas 消耗高

2. **方案 B: 混合模式** (链上+链下匹配)
   - ✅ 优势: 性能好,Gas 消耗低
   - ❌ 劣势: 复杂度高,确定性难保证,链下中心化

3. **方案 C: MemClob 纯内存订单簿** ✅ 最终选择
   - ✅ 优势: 极高性能,灵活数据结构,确定性保证
   - ❌ 劣势: 内存消耗大,状态重建复杂

**权衡分析**:

| 维度           | 方案 A | 方案 B | 方案 C |
|---------------|-------|-------|-------|
| 性能           | ⭐     | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| 确定性保证     | ⭐⭐⭐⭐⭐ | ⭐⭐   | ⭐⭐⭐⭐⭐ |
| 实现复杂度     | ⭐⭐   | ⭐⭐⭐⭐ | ⭐⭐⭐  |
| 内存消耗       | ⭐⭐⭐⭐⭐ | ⭐⭐⭐  | ⭐     |
| 去中心化程度   | ⭐⭐⭐⭐⭐ | ⭐⭐   | ⭐⭐⭐⭐⭐ |

**决策理由**:

**性能是核心要求**:
- 订单匹配是高频操作,必须极快
- 内存访问速度比磁盘快 1000 倍以上
- 交易所竞争力取决于匹配延迟

**确定性可以保证**:
- 通过 PreBlocker 重建机制
- 所有节点维护相同的 MemClob 状态
- 匹配算法确定性设计

**内存消耗可接受**:
- 订单数量有限 (速率限制、权益层级限制)
- 现代服务器内存充足 (GB 级别)
- 状态修剪机制防止膨胀

**最终决策**: 选择方案 C (MemClob 纯内存订单簿)

### 8.2 为什么区分 Short-Term 和 Long-Term 订单?

(详见 3.1.1 节)

**决策理由**: 性能和存储优化

### 8.3 为什么在 TransientStore 存储某些状态?

**问题**: Short-Term 订单应该存储在哪里?

**备选方案**:

1. **方案 A: StateStore** (持久化存储)
   - ✅ 优势: 节点重启后状态不丢失
   - ❌ 劣势: 写入开销大,状态膨胀

2. **方案 B: 仅 MemClob** (纯内存,不持久化)
   - ✅ 优势: 性能最优
   - ❌ 劣势: 无法验证,不可审计

3. **方案 C: TransientStore** (区块级瞬态存储) ✅ 最终选择
   - ✅ 优势: 区块内可验证,自动清理,性能好
   - ❌ 劣势: 需要额外的存储层

**权衡分析**:

| 维度           | 方案 A | 方案 B | 方案 C |
|---------------|-------|-------|-------|
| 性能           | ⭐⭐   | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| 可验证性       | ⭐⭐⭐⭐⭐ | ⭐     | ⭐⭐⭐⭐⭐ |
| 存储开销       | ⭐     | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| 自动清理       | ❌     | ✅     | ✅     |

**决策理由**:

**Short-Term 订单特点**:
- 单区块有效,无需跨区块持久化
- 数量大,持久化开销高
- 需要在区块内验证,不能完全不持久化

**TransientStore 优势**:
- 区块内可验证 (满足共识要求)
- 区块结束自动清理 (无需手动清理)
- 性能优于 StateStore (内存操作)

**最终决策**: 选择方案 C (TransientStore)

---

#### TransientStore 的典型使用场景

TransientStore 用于存储**仅在当前区块有效**的临时数据,主要包括以下场景:

**场景 1: 最小和最大成交价追踪**

**目的**: 跟踪当前区块内的最小和最大成交价格,用于改进条件订单触发

**为什么用 TransientStore**:
- **区块级作用域**: 只需要当前区块的成交价格,下个区块重新计算
- **自动清理**: 区块结束后自动清空,无需手动清理逻辑
- **性能优先**: 每次成交都要更新,内存操作速度快

**数据结构**:
```
MinTradePrice: PerpetualId -> uint64 (subticks)
MaxTradePrice: PerpetualId -> uint64 (subticks)
```

**使用流程**:
```
区块开始: MinTradePrice = MaxUint64, MaxTradePrice = 0

成交 1: 价格 49,800
    -> MinTradePrice = min(MaxUint64, 49,800) = 49,800
    -> MaxTradePrice = max(0, 49,800) = 49,800

成交 2: 价格 50,200
    -> MinTradePrice = min(49,800, 50,200) = 49,800
    -> MaxTradePrice = max(49,800, 50,200) = 50,200

条件订单触发检查:
    if oraclePrice <= stopLoss || MinTradePrice <= stopLoss {
        触发止损单
    }

区块结束: 自动清空,下个区块重新开始
```

**如果用 StateStore**: 需要手动清理,增加复杂度,且性能较差
**如果用 MemClob**: 无法在共识层验证,不满足确定性要求

---

**场景 2: 子账户清算信息**

**目的**: 跟踪当前区块内的清算事件,防止同一账户在同一区块内被多次清算

**为什么用 TransientStore**:
- **防止重复清算**: 区块内需要记录哪些账户已被清算
- **区块级限制**: 清算限制是按区块计算的 (如每个区块最多清算多少)
- **自动重置**: 下个区块限制重置,TransientStore 自动清理

**数据结构**:
```
SubaccountLiquidationInfo: SubaccountId -> LiquidationInfo {
    NotionalLiquidated: uint64        // 已清算的名义价值
    QuantumsInsuranceLost: uint64     // 保险基金损失
}
```

**使用流程**:
```
区块开始: SubaccountLiquidationInfo 为空

清算账户 A:
    1. 检查 A 是否已在 TransientStore 中
    2. 如果不存在,创建记录,记录清算量
    3. 如果存在,累加清算量,检查是否超出区块限制

清算账户 B:
    (同上)

区块结束: 所有清算记录自动清空
```

---

**场景 3: 未提交的订单放置和取消**

**目的**: 在 CheckTx 阶段跟踪待提交的订单操作,用于乐观验证

**为什么用 TransientStore**:
- **CheckTx 到 FinalizeBlock 的桥梁**: CheckTx 验证通过的操作暂存,等待 FinalizeBlock 确认
- **防止冲突**: 检测同一订单的多次提交或取消
- **自动清理**: FinalizeBlock 之后,未提交的操作自动清空

**数据结构**:
```
UncommittedStatefulOrderPlacement: OrderId -> OrderPlacement
UncommittedStatefulOrderCancellation: OrderId -> bool
```

**使用流程**:
```
CheckTx (用户提交订单 A):
    1. 验证订单有效性
    2. 写入 TransientStore: UncommittedStatefulOrderPlacement[A] = OrderPlacement
    3. 乐观地放入 MemClob

FinalizeBlock (区块确认):
    1. 从 TransientStore 读取 UncommittedStatefulOrderPlacement
    2. 正式写入 StateStore
    3. 从 TransientStore 删除

区块结束: TransientStore 自动清空 (未确认的操作丢弃)
```

---

**场景 4: 未提交的 Stateful 订单净计数**

**目的**: 跟踪每个子账户待提交的 Stateful 订单数量,提前验证订单限额

**为什么用 TransientStore**:
- **提前验证**: 在 CheckTx 阶段就检查订单限额,避免在 FinalizeBlock 时失败
- **暂时性**: 仅需要在区块内有效,提交后计数合并到 StateStore
- **支持批量操作**: 同一区块内多个订单操作的净变化

**数据结构**:
```
UncommittedStatefulOrderCount: SubaccountId -> int32 (可为负数)
```

**使用流程**:
```
CheckTx (用户放置 Long-Term 订单):
    1. UncommittedCount[Account] += 1
    2. 验证: StateStoreCount + UncommittedCount <= Limit

CheckTx (用户取消 Long-Term 订单):
    1. UncommittedCount[Account] -= 1

FinalizeBlock:
    1. 合并计数到 StateStore
    2. UncommittedCount 自动清空
```

---

**TransientStore 使用总结**:

| 使用场景 | 典型数据 | 生命周期 | 为什么不用 StateStore | 为什么不用 MemClob |
|---------|---------|---------|---------------------|-------------------|
| 成交价追踪 | Min/Max Trade Price | 区块级 | 无需跨区块,手动清理复杂 | 无法共识验证 |
| 清算信息 | SubaccountLiquidationInfo | 区块级 | 无需跨区块,自动重置 | 无法共识验证 |
| 未提交订单 | UncommittedOrders | 区块级 | 浪费存储,需手动清理 | 无法验证 |
| 订单计数 | UncommittedCount | 区块级 | 无需持久化,自动清理 | 无法验证 |

**核心设计原则**:
- **区块作用域**: 数据仅在单个区块内有效
- **自动清理**: 区块结束后自动清空,无需手动逻辑
- **共识验证**: 数据参与共识,所有节点一致
- **性能优化**: 内存操作,比 StateStore 快

---

### 8.4 为什么需要订单重新处理保护?

**问题**: 如何防止同一订单被重复处理,导致状态不一致?

**背景**:

在分布式共识环境中,订单可能在不同阶段被多次提交到 MemClob:
1. **CheckTx 阶段**: 验证器乐观地将订单放入 MemClob
2. **FinalizeBlock 阶段**: 订单通过共识后再次从 StateStore 读取并处理
3. **PrepareCheckState 阶段**: 重放操作时可能再次遇到同一订单

如果没有保护机制,同一订单可能被重复处理,导致:
- 订单数量计数错误
- 重复扣除保证金
- 订单簿状态混乱
- 节点间状态不一致

**备选方案**:

1. **方案 A: 不做保护**
   - ✅ 优势: 实现简单,无额外开销
   - ❌ 劣势: 可能重复处理,状态不一致,系统不稳定

2. **方案 B: 仅检查 OrderId**
   - ✅ 优势: 实现简单,开销小
   - ❌ 劣势: 无法防止相同 OrderId 的不同订单 (如替换订单)

3. **方案 C: 检查 OrderId + OrderHash** ✅ 最终选择
   - ✅ 优势: 安全,确定性,防止重复处理
   - ❌ 劣势: 略增加计算成本 (哈希计算)

**实现机制**:

```go
// 在 MemClob 中维护已处理订单的哈希集合
type MemClob struct {
    processedOrders map[OrderId]OrderHash  // OrderId -> OrderHash 映射
    // ... 其他字段
}

// 订单处理前检查
func (m *MemClob) PlaceOrder(ctx sdk.Context, order Order) error {
    orderId := order.OrderId
    orderHash := order.GetOrderHash()

    // 检查是否已处理
    if existingHash, found := m.processedOrders[orderId]; found {
        if existingHash == orderHash {
            // 完全相同的订单,拒绝重复处理
            return ErrOrderReprocessed
        }
        // OrderId 相同但 OrderHash 不同,这是订单替换,允许
    }

    // 记录订单哈希
    m.processedOrders[orderId] = orderHash

    // 继续处理订单...
}
```

**OrderHash 计算**:

OrderHash 包含订单的所有关键字段,确保唯一性:
```
OrderHash = Hash(
    OrderId +
    Side +
    Quantums +
    Subticks +
    GoodTilBlockTime +
    TimeInForce +
    ReduceOnly +
    ConditionType +
    ConditionalOrderTriggerSubticks
)
```

**典型场景分析**:

**场景 1: 正常订单放置**
```
1. CheckTx: 订单 A 放入 MemClob,记录 Hash_A
2. FinalizeBlock: 再次处理订单 A
3. MemClob 检查: OrderId 相同,Hash 相同
4. 拒绝重复处理 ✅
```

**场景 2: 订单替换**
```
1. CheckTx: 订单 A 放入 MemClob,记录 Hash_A
2. 用户取消订单 A,放置订单 A' (相同 OrderId,不同参数)
3. CheckTx: 订单 A' 放入 MemClob
4. MemClob 检查: OrderId 相同,但 Hash_A' ≠ Hash_A
5. 允许处理,更新为 Hash_A' ✅
```

**场景 3: PrepareCheckState 重放**
```
1. 上一区块: 订单 A 成交
2. PrepareCheckState: 重放上一区块操作
3. 尝试再次放置订单 A
4. MemClob 检查: OrderId 和 Hash 都匹配
5. 拒绝重复处理 ✅ (订单已成交,应被清理)
```

**权衡分析**:

| 维度           | 方案 A | 方案 B | 方案 C |
|---------------|-------|-------|-------|
| 安全性         | ⭐     | ⭐⭐⭐  | ⭐⭐⭐⭐⭐ |
| 确定性保证     | ⭐     | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| 性能开销       | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐  |
| 实现复杂度     | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐  |

**决策理由**:

**安全性是首要考虑**:
- 重复处理可能导致资金错误,必须绝对避免
- OrderHash 提供加密级别的唯一性保证
- 确定性要求所有节点行为一致

**性能开销可接受**:
- 哈希计算开销小 (微秒级)
- 相比订单匹配逻辑,开销可忽略
- 内存中的哈希表查找非常快 (O(1))

**支持订单替换场景**:
- 用户可以替换订单 (取消旧订单,放置新订单)
- OrderId 相同但内容不同时,需要区分
- OrderHash 能正确识别订单变化

**最终决策**: 选择方案 C (检查 OrderId + OrderHash)

**错误处理**:
```
错误码: ErrOrderReprocessed
错误消息: "Order reprocessed: OrderId <orderId> with hash <hash> was already processed"
影响: 订单被拒绝,不影响系统状态
日志级别: Info (正常保护机制,非异常)
```

**关键文件**:
- `protocol/x/clob/types/errors.go:ErrOrderReprocessed`
- `protocol/x/clob/memclob/memclob.go` (订单处理前检查)

---

### 8.5 为什么匹配在 EndBlocker 而非 DeliverTx?

**问题**: 订单匹配应该在哪个 ABCI 阶段执行?

**备选方案**:

1. **方案 A: CheckTx** (交易验证阶段)
   - ✅ 优势: 用户立即知道结果
   - ❌ 劣势: 不共识,节点间结果不一致

2. **方案 B: DeliverTx** (ProposedOperations) ✅ 最终选择
   - ✅ 优势: 共识保证,确定性匹配,批量处理
   - ❌ 劣势: 延迟高 (需要等待区块确认)

3. **方案 C: EndBlocker** (区块结束阶段)
   - ✅ 优势: 统一处理,逻辑清晰
   - ❌ 劣势: 延迟更高,无法与用户交易同步

**权衡分析**:

| 维度           | 方案 A | 方案 B | 方案 C |
|---------------|-------|-------|-------|
| 确定性保证     | ⭐     | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| 延迟           | ⭐⭐⭐⭐⭐ | ⭐⭐⭐  | ⭐⭐   |
| 批量处理       | ❌     | ✅     | ✅     |
| 提议者一致性   | ❌     | ✅     | ✅     |

**决策理由**:

**共识要求优先**:
- 订单匹配必须确定性,所有节点结果一致
- CheckTx 不共识,无法保证确定性

**ProposedOperations 机制**:
- 提议者构建匹配结果,所有验证者验证
- 确保提议者和验证者结果一致
- 批量处理,减少 Gas 消耗

**EndBlocker 不适合**:
- 无法与用户交易同步
- 用户订单在 DeliverTx 提交,EndBlocker 匹配会延迟一个区块

**最终决策**: 选择方案 B (DeliverTx - ProposedOperations)

### 8.5 为什么需要 ProposedOperations 机制?

**问题**: 如何保证提议者和验证者的订单匹配结果一致?

**挑战**:
- 提议者在 PrepareProposal 阶段构建匹配结果
- 验证者在 ProcessProposal 和 FinalizeBlock 阶段验证
- 如果提议者和验证者结果不一致,共识会失败

**解决方案**: ProposedOperations 机制

**工作流程**:

1. **提议者** (PrepareProposal):
   ```
   ├─ 从 MemClob 获取待匹配订单
   ├─ 执行匹配算法,生成匹配结果
   ├─ 构建 MsgProposedOperations (包含匹配结果)
   └─ 包含在区块提案中
   ```

2. **验证者** (ProcessProposal):
   ```
   ├─ 接收区块提案
   ├─ 解析 MsgProposedOperations
   ├─ 本地重新执行匹配算法
   ├─ 验证结果与提议者一致
   └─ 返回 ACCEPT 或 REJECT
   ```

3. **所有节点** (FinalizeBlock):
   ```
   ├─ 执行 MsgProposedOperations
   ├─ 更新账户余额
   ├─ 更新订单簿状态
   └─ 发出匹配事件
   ```

**优势**:
- ✅ 确保提议者和验证者结果一致
- ✅ 批量处理订单,减少交易数量
- ✅ 防止 MEV (匹配结果固定,无法篡改)

**最终决策**: 使用 ProposedOperations 机制

---

## 9. 集成点与扩展

### 9.1 与其他模块的关键集成

#### 9.1.1 与 Subaccounts 模块集成

**集成点**:
- 订单验证: 检查保证金充足性
- 订单成交: 更新账户余额
- 清算判断: 计算保证金率

**接口设计**:
```go
type SubaccountsKeeper interface {
    GetNetCollateral(ctx, subaccountId) -> int64
    CanPlaceOrder(ctx, subaccountId, order) -> bool
    UpdateSubaccounts(ctx, fills) -> error
    GetMarginRequirements(ctx, subaccountId) -> (IM, MM)
}
```

**设计考虑**:
- 接口最小化,减少耦合
- 批量操作支持,提升性能
- 错误处理清晰,便于调试

#### 9.1.2 与 Perpetuals 模块集成

**集成点**:
- 订单验证: 获取合约配置
- 保证金计算: 获取 IMR/MMR
- 清算判断: 获取流动性层级

**接口设计**:
```go
type PerpetualsKeeper interface {
    GetPerpetual(ctx, perpetualId) -> Perpetual
    GetLiquidityTier(ctx, liquidityTierId) -> LiquidityTier
    GetOpenInterest(ctx, perpetualId) -> int64
}
```

#### 9.1.3 与 Prices 模块集成

**集成点**:
- 清算判断: 获取市场价格
- 条件订单触发: 检查价格条件
- 保证金计算: 使用市场价格

**接口设计**:
```go
type PricesKeeper interface {
    GetMarketPrice(ctx, marketId) -> MarketPrice
    GetAllMarketPrices(ctx) -> []MarketPrice
}
```

### 9.2 现有扩展点

#### 9.2.1 可配置参数

**清算配置** (LiquidationsConfig):
```protobuf
message LiquidationsConfig {
    uint32 max_liquidation_fee_ppm = 1;           // 最大清算手续费
    uint32 position_block_limits_ppm = 2;          // 仓位区块限制
    uint32 subaccount_block_limits_ppm = 3;        // 子账户区块限制
    FillablePriceConfig fillable_price_config = 4; // 可成交价格配置
}
```

**权益层级限制配置** (EquityTierLimitConfiguration):
```protobuf
message EquityTierLimitConfiguration {
    repeated EquityTierLimit equity_tier_limit_list = 1;
}

message EquityTierLimit {
    uint64 usd_tnc_required = 1;  // 所需权益 (USDC)
    uint32 limit = 2;             // 订单限制
}
```

**区块速率限制配置** (BlockRateLimitConfiguration):
```protobuf
message BlockRateLimitConfiguration {
    uint32 max_short_term_orders_per_n_blocks = 1;    // Short-Term 订单限制
    uint32 max_stateful_orders_per_n_blocks = 2;      // Long-Term 订单限制
    uint32 max_short_term_order_cancellations_per_n_blocks = 3; // 取消限制
    uint32 max_leverage_updates_per_n_blocks = 4;     // 杠杆更新限制
}
```

#### 9.2.2 扩展机制

**订单类型扩展**:
- 在 `proto/hermes/clob/order.proto` 添加新订单类型
- 实现验证逻辑
- 实现匹配逻辑
- 添加测试

**匹配算法扩展**:
- 在 `memclob/memclob.go` 修改匹配算法
- 确保确定性
- 性能测试

**清算策略扩展**:
- 在 `keeper/liquidations_*.go` 修改清算逻辑
- 调整风险参数
- 测试极端场景

---

## 10. 总结

### 10.1 关键设计亮点

1. **4 层存储架构**: 针对不同数据生命周期优化
2. **MemClob 纯内存订单簿**: 极高性能匹配引擎
3. **确定性匹配**: 共识保证,MEV 防护
4. **ProposedOperations 机制**: 提议者和验证者一致性
5. **多层 MEV 保护**: 速率限制、权益层级限制、确定性匹配

### 10.2 设计权衡

| 设计选择              | 优势                 | 劣势                 | 决策理由             |
|---------------------|---------------------|---------------------|---------------------|
| MemClob 内存订单簿   | 极高性能             | 内存消耗大,状态重建复杂 | 性能是核心竞争力     |
| Short/Long 订单分离  | 存储优化,逻辑简化     | 维护两套逻辑         | 性能和存储优化优先   |
| TransientStore      | 自动清理,性能好       | 需要额外存储层       | 平衡性能和可验证性   |
| ProposedOperations  | 确定性匹配,批量处理   | 延迟高               | 共识和确定性优先     |

### 10.3 未来改进方向

1. **订单加密**: 防止提议者 MEV
2. **批量拍卖**: 完全消除抢跑
3. **分片**: 支持更高吞吐量
4. **状态压缩**: 减少链上存储
5. **高级订单类型**: Iceberg, Pegged, Bracket 等

---

**文档版本**: v1.0
**最后更新**: 2025-12-31
**文档作者**: Claude Sonnet 4.5 + Hermes DEX Team
**文档状态**: ✅ 完成

**参考资料**:
- 数据结构文档: `notes/data_structure/clob.md`
- 模块级文档: `protocol/x/clob/CLAUDE.md`
- 执行计划: `notes/plan/dex_prd_plan.md`