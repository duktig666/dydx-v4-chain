# Leverage 模块架构设计

## 1. 模块概述

### 1.1 职责定义

Leverage (杠杆) 模块是 Hermes DEX 的一个**集成模块**,负责:

- **杠杆倍数设置**: 允许用户设置子账户的杠杆倍数
- **保证金计算集成**: 影响保证金要求计算
- **风险管理**: 限制用户的杠杆范围,控制系统风险

**重要特性**: Leverage 模块没有独立的 Keeper,数据存储在 CLOB 模块,逻辑分散在 CLOB 和 Subaccounts 模块中。

### 1.2 核心功能

**杠杆倍数管理**:
- 用户可以为子账户设置杠杆倍数 (例如 10x, 20x)
- 杠杆倍数影响初始保证金要求
- 杠杆越高,保证金要求越低,风险越大

**保证金计算集成**:
- 杠杆倍数用于计算初始保证金率 (IMR)
- IMR = 1 / 杠杆倍数
- 例如: 20x 杠杆 → IMR = 5%

**风险控制**:
- 限制最大杠杆倍数 (防止过度风险)
- 验证杠杆调整后账户仍满足保证金要求
- 防止用户在持有大量仓位时随意提高杠杆

### 1.3 关键约束

**集成设计**:
- 无独立 Keeper,逻辑分散到 CLOB 和 Subaccounts
- 数据存储在 CLOB 模块
- 减少模块间通信开销

**确定性要求**:
- 杠杆调整必须确定性
- 保证金计算必须确定性
- 避免浮点运算

**安全约束**:
- 杠杆调整需验证保证金充足性
- 防止用户通过调整杠杆绕过风控
- 限制最大杠杆倍数

---

## 2. 架构设计

### 2.1 组件结构

```
Leverage 功能架构 (集成设计)

┌─────────────────────────────────────────────────┐
│         CLOB 模块 (消息处理)                     │
│  ├─ MsgUpdateLeverage (更新杠杆消息)            │
│  └─ Leverage StateStore (存储杠杆数据)          │
└─────────────────────────────────────────────────┘
                       ↕
┌─────────────────────────────────────────────────┐
│         CLOB Keeper (验证与存储)                 │
│  ├─ UpdateLeverage() (更新杠杆)                 │
│  ├─ GetLeverage() (读取杠杆)                    │
│  └─ ValidateLeverageChange() (验证)             │
└─────────────────────────────────────────────────┘
                       ↕
┌─────────────────────────────────────────────────┐
│         Subaccounts Keeper (保证金计算)          │
│  ├─ GetInitialMarginRequirement()               │
│  │   └─ 使用 CLOB.GetLeverage() 计算 IMR       │
│  └─ CanUpdateLeverage() (验证保证金充足)        │
└─────────────────────────────────────────────────┘
                       ↕
┌─────────────────────────────────────────────────┐
│         CLOB StateStore (存储层)                 │
│  └─ LeverageKeyPrefix + SubaccountId            │
│      → Leverage { SubaccountId, Leverage }      │
└─────────────────────────────────────────────────┘
```

### 2.2 存储架构

#### StateStore (链上持久化存储,在 CLOB 模块)

**存储位置**: CLOB 模块 StateStore

**存储键**: `LeverageKeyPrefix` + `SubaccountId` = `"Leverage:" + <SubaccountId>`

**数据结构**: `Leverage`

```go
type Leverage struct {
    SubaccountId SubaccountId  // 子账户 ID
    Leverage uint32            // 杠杆倍数(例如 20 表示 20x)
}
```

**业务含义**:
- 存储子账户的杠杆倍数设置
- 影响保证金要求计算
- 杠杆越高,保证金要求越低,风险越大

**默认值**: 如果未设置,使用系统默认杠杆 (例如 20x)

### 2.3 ABCI 生命周期集成

```
        Leverage 功能 ABCI 集成

┌─────────────────────────────────────────────────┐
│    DeliverTx (交易执行)                          │
│    └─ MsgUpdateLeverage                         │
│        {                                        │
│          Owner: 子账户所有者,                   │
│          SubaccountId: 目标子账户,              │
│          NewLeverage: 新杠杆倍数                │
│        }                                        │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────┐
│    CLOB.MsgServer.UpdateLeverage()              │
│    ├─ 验证消息签名和权限                        │
│    │   (Owner 是否拥有 SubaccountId)            │
│    ├─ 验证杠杆倍数合法性                        │
│    │   (NewLeverage > 0 && <= MaxLeverage)      │
│    └─ 调用 Keeper.UpdateLeverage()              │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────┐
│    CLOB.Keeper.UpdateLeverage()                 │
│    ├─ 调用 Subaccounts.CanUpdateLeverage()      │
│    │   (验证保证金是否充足)                     │
│    └─ 如果通过,更新 StateStore                  │
│        SetLeverage(SubaccountId, NewLeverage)   │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────┐
│    Subaccounts.Keeper.CanUpdateLeverage()       │
│    ├─ 读取当前杠杆 (OldLeverage)                │
│    ├─ 计算新保证金要求                          │
│    │   NewIMR = 1 / NewLeverage                 │
│    │   NewMarginReq = PositionValue × NewIMR    │
│    ├─ 检查账户净资产 >= NewMarginReq            │
│    └─ 返回验证结果 (通过/失败)                  │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────┐
│    CLOB StateStore                              │
│    └─ 持久化新杠杆设置                          │
│        Leverage[SubaccountId] = NewLeverage     │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────┐
│    触发事件                                      │
│    └─ LeverageUpdatedEvent {                    │
│          SubaccountId,                          │
│          OldLeverage,                           │
│          NewLeverage                            │
│       }                                         │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
            杠杆更新完成
```

---

## 3. 核心业务流程

### 3.1 更新杠杆流程

```
        用户更新杠杆 → 验证 → 更新存储

┌─────────────────────────────────────────────────┐
│    用户提交 MsgUpdateLeverage                    │
│    {                                            │
│      Owner: 用户地址,                           │
│      SubaccountId: 子账户 ID,                   │
│      NewLeverage: 新杠杆倍数 (例如 30)          │
│    }                                            │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────┐
│    消息验证                                      │
│    ├─ 验证 Owner 拥有 SubaccountId               │
│    ├─ 验证 NewLeverage > 0                      │
│    └─ 验证 NewLeverage <= MaxLeverage (例如 100x)│
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────┐
│    读取当前杠杆                                  │
│    OldLeverage = GetLeverage(SubaccountId)      │
│    (如果未设置,使用默认值 20x)                  │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────┐
│    计算新保证金要求                              │
│    ├─ 获取子账户持仓                            │
│    │   Positions = GetPositions(SubaccountId)  │
│    ├─ 计算仓位总价值                            │
│    │   TotalPositionValue = Σ(Position × Price)│
│    ├─ 计算新初始保证金率                        │
│    │   NewIMR = 1 / NewLeverage                 │
│    └─ 计算新保证金要求                          │
│        NewMarginReq = TotalPositionValue × NewIMR│
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────┐
│    验证保证金充足性                              │
│    ├─ 获取账户净资产                            │
│    │   NetCollateral = GetNetCollateral(SubaccountId) │
│    ├─ 检查 NetCollateral >= NewMarginReq        │
│    └─ 如果不足,拒绝更新                         │
└───────────────────┬─────────────────────────────┘
                    │ (通过验证)
                    ▼
┌─────────────────────────────────────────────────┐
│    更新杠杆设置                                  │
│    ├─ SetLeverage(SubaccountId, NewLeverage)    │
│    └─ 持久化到 StateStore                       │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────┐
│    触发事件                                      │
│    └─ LeverageUpdatedEvent {                    │
│          SubaccountId,                          │
│          OldLeverage,                           │
│          NewLeverage                            │
│       }                                         │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
            杠杆更新完成
```

#### 保证金要求计算公式

**初始保证金率 (IMR) 计算**:

数学表达式:
```
IMR = 1 / Leverage
```

变量说明:
- **IMR**: 初始保证金率 (Initial Margin Ratio)
- **Leverage**: 杠杆倍数 (例如 20 表示 20x)

计算示例:
- 假设 Leverage = 20x
- IMR = 1 / 20 = 0.05 = 5%

业务解释:
- 杠杆倍数越高,初始保证金率越低
- 20x 杠杆 → 5% 保证金率,可以用 5% 本金控制 100% 仓位
- 杠杆倍数与保证金率互为倒数关系

**保证金要求计算**:

数学表达式:
```
MarginRequirement = PositionValue × IMR
IMR = 1 / Leverage
```

变量说明:
- **MarginRequirement**: 保证金要求 (USDC)
- **PositionValue**: 仓位价值 (USDC)
  - PositionValue = PositionSize × OraclePrice
  - PositionSize: 持仓数量 (例如 BTC 数量)
  - OraclePrice: 预言机价格 (USDC/BTC)
- **IMR**: 初始保证金率
- **Leverage**: 杠杆倍数

计算示例:
- 假设用户持有 1 BTC 仓位
- 假设 BTC 价格 = 50,000 USDC
- 假设 Leverage = 20x (IMR = 5%)
- PositionValue = 1 × 50,000 = 50,000 USDC
- MarginRequirement = 50,000 × 0.05 = 2,500 USDC

业务解释:
- 用户持有 50,000 USDC 的仓位,只需要 2,500 USDC 保证金
- 杠杆倍数越高,保证金要求越低,风险越大
- 如果账户净资产低于 2,500 USDC,可能触发清算

**杠杆调整后保证金验证**:

数学表达式:
```
NetCollateral >= NewMarginRequirement
NewMarginRequirement = PositionValue × (1 / NewLeverage)
```

变量说明:
- **NetCollateral**: 账户净资产 (USDC)
- **NewMarginRequirement**: 新杠杆下的保证金要求 (USDC)
- **PositionValue**: 仓位价值 (USDC)
- **NewLeverage**: 新杠杆倍数

计算示例 (提高杠杆):
- 假设 PositionValue = 50,000 USDC
- 假设 OldLeverage = 20x (IMR = 5%)
- 假设 NewLeverage = 50x (IMR = 2%)
- OldMarginReq = 50,000 × 0.05 = 2,500 USDC
- NewMarginReq = 50,000 × 0.02 = 1,000 USDC
- 假设 NetCollateral = 3,000 USDC
- 验证: 3,000 >= 1,000 ✅ 通过

计算示例 (降低杠杆):
- 假设 PositionValue = 50,000 USDC
- 假设 OldLeverage = 50x (IMR = 2%)
- 假设 NewLeverage = 10x (IMR = 10%)
- OldMarginReq = 50,000 × 0.02 = 1,000 USDC
- NewMarginReq = 50,000 × 0.10 = 5,000 USDC
- 假设 NetCollateral = 3,000 USDC
- 验证: 3,000 >= 5,000 ❌ 失败

业务解释:
- 提高杠杆 (降低保证金要求): 通常可以通过验证
- 降低杠杆 (提高保证金要求): 需要账户有足够净资产,否则拒绝
- 防止用户在保证金不足时随意调整杠杆

### 3.2 保证金计算集成流程

```
        订单放置时的保证金计算

┌─────────────────────────────────────────────────┐
│    用户放置订单                                  │
│    PlaceOrder(Order)                            │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────┐
│    CLOB.Keeper.PlaceOrder()                     │
│    └─ 验证订单参数                              │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────┐
│    调用 Subaccounts 验证保证金                   │
│    Subaccounts.CanPlaceOrder(Order)             │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────┐
│    Subaccounts.Keeper                           │
│    ├─ 读取杠杆设置                              │
│    │   Leverage = CLOB.GetLeverage(SubaccountId)│
│    ├─ 计算初始保证金率                          │
│    │   IMR = 1 / Leverage                       │
│    ├─ 计算新订单保证金要求                      │
│    │   OrderMarginReq = OrderValue × IMR        │
│    ├─ 计算总保证金要求                          │
│    │   TotalMarginReq = ExistingMarginReq + OrderMarginReq │
│    ├─ 获取账户净资产                            │
│    │   NetCollateral = GetNetCollateral()       │
│    └─ 验证 NetCollateral >= TotalMarginReq      │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────┐
│    返回验证结果                                  │
│    ├─ 如果通过: 订单放置成功                    │
│    └─ 如果失败: 拒绝订单 (保证金不足)           │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
            订单验证完成
```

---

## 4. 模块依赖关系

### 4.1 上游依赖 (Leverage 功能依赖哪些模块)

**Subaccounts Keeper**:
- 用途: 验证保证金充足性
- 调用: `CanUpdateLeverage()`, `GetNetCollateral()`
- 作用: 确保杠杆调整不违反风控要求

**Perpetuals Keeper**:
- 用途: 获取 LiquidityTier 配置
- 调用: `GetLiquidityTier()`
- 作用: 获取最大杠杆限制

**Prices Keeper**:
- 用途: 获取预言机价格
- 调用: `GetMarketPrice()`
- 作用: 计算仓位价值,用于保证金验证

### 4.2 下游被依赖 (哪些模块依赖 Leverage 功能)

**Subaccounts 模块**:
- 用途: 计算保证金要求
- 调用: `CLOB.GetLeverage(SubaccountId)`
- 作用: 获取杠杆倍数,计算 IMR

**前端 UI**:
- 用途: 展示用户杠杆设置
- 查询: `GetLeverage(SubaccountId)`
- 作用: 显示当前杠杆倍数

### 4.3 依赖关系图

```
        Leverage 功能依赖关系

    ┌──────────────┐
    │ Subaccounts  │ ──> CanUpdateLeverage (验证保证金)
    └──────────────┘
           ↓
    ┌──────────────┐
    │  CLOB 模块    │
    │  (Leverage    │ <─> GetLeverage (读取杠杆)
    │   StateStore) │     SetLeverage (更新杠杆)
    └──────────────┘
           ↓
    ┌──────────────┐
    │  Perpetuals  │ ──> GetLiquidityTier (获取最大杠杆)
    └──────────────┘
           ↓
    ┌──────────────┐
    │    Prices    │ ──> GetMarketPrice (计算仓位价值)
    └──────────────┘
```

---

## 5. 数据流设计

### 5.1 杠杆更新数据流

```
用户请求 → CLOB 验证 → Subaccounts 验证 → 更新 StateStore

用户
  │ MsgUpdateLeverage
  ▼
CLOB.MsgServer
  │ 验证消息签名和权限
  ▼
CLOB.Keeper
  │ 验证杠杆倍数合法性
  ▼
Subaccounts.Keeper
  │ CanUpdateLeverage(SubaccountId, NewLeverage)
  │ ├─ 读取当前杠杆
  │ ├─ 计算新保证金要求
  │ ├─ 获取账户净资产
  │ └─ 验证 NetCollateral >= NewMarginReq
  ▼
CLOB.Keeper
  │ SetLeverage(SubaccountId, NewLeverage)
  ▼
CLOB StateStore
  │ 持久化新杠杆设置
  ▼
Event Bus
  │ LeverageUpdatedEvent
  ▼
Indexer
  │ 索引杠杆更新事件
  ▼
前端 UI
  (展示更新后的杠杆)
```

### 5.2 保证金计算数据流

```
订单放置 → 读取杠杆 → 计算保证金 → 验证通过/失败

订单放置请求
  │ PlaceOrder(Order)
  ▼
CLOB.Keeper
  │ 调用 Subaccounts 验证
  ▼
Subaccounts.Keeper
  │ CanPlaceOrder(Order)
  │ ├─ 读取杠杆
  │ │   Leverage = CLOB.GetLeverage(SubaccountId)
  │ ├─ 计算 IMR
  │ │   IMR = 1 / Leverage
  │ ├─ 计算订单保证金
  │ │   OrderMarginReq = OrderValue × IMR
  │ ├─ 获取账户净资产
  │ │   NetCollateral = GetNetCollateral()
  │ └─ 验证
  │     NetCollateral >= TotalMarginReq
  ▼
CLOB.Keeper
  │ 根据验证结果决定是否放置订单
  ▼
MemClob
  (订单进入订单簿)
```

---

## 6. 性能与可扩展性

### 6.1 性能优化点

**杠杆读取优化**:
- 杠杆数据存储在 CLOB StateStore,读取性能 O(1)
- 使用缓存减少重复读取
- 默认杠杆值避免未设置时的查询

**保证金计算优化**:
- IMR 计算简单 (IMR = 1 / Leverage)
- 避免复杂的公式和浮点运算
- 整数运算确保确定性和性能

**验证逻辑优化**:
- 杠杆调整验证只在更新时执行
- 不在查询时验证,减少计算开销

### 6.2 扩展性设计

**灵活的杠杆限制**:
- 最大杠杆可以通过 LiquidityTier 配置
- 不同市场可以有不同的杠杆限制
- 支持动态调整风险参数

**多子账户支持**:
- 每个子账户可以有独立的杠杆设置
- 支持用户为不同策略使用不同杠杆

### 6.3 瓶颈分析

**潜在瓶颈**:

1. **杠杆调整频繁**:
   - 如果用户频繁调整杠杆,验证开销大
   - 解决方案: 限制杠杆调整频率 (例如每天一次)

2. **保证金计算复杂**:
   - 如果用户持仓复杂,保证金计算开销大
   - 解决方案: 缓存保证金计算结果,仅在仓位变化时重新计算

---

## 7. 安全与风险控制

### 7.1 过度杠杆风险控制

**问题**: 用户设置过高杠杆,导致系统风险增加。

**防护机制**:

**最大杠杆限制**:
- LiquidityTier 配置最大杠杆 (例如 100x)
- 用户无法设置超过最大杠杆
- 不同市场可以有不同的限制

**保证金充足性验证**:
- 杠杆调整时验证账户净资产 >= 新保证金要求
- 防止用户在保证金不足时提高杠杆
- 确保系统风控要求

### 7.2 杠杆操纵风险控制

**问题**: 用户可能通过调整杠杆绕过风控。

**防护机制**:

**实时验证**:
- 杠杆调整时立即验证保证金要求
- 不允许杠杆调整导致保证金不足

**清算机制**:
- 即使用户调整杠杆,清算机制仍然有效
- 如果账户净资产低于维持保证金,触发清算

### 7.3 确定性保证

**确定性要求**:

**杠杆存储确定性**:
- 杠杆倍数使用 uint32 整数
- 避免浮点数,确保确定性

**保证金计算确定性**:
- IMR 计算使用整数除法
- 所有节点计算结果一致

---

## 8. 技术决策记录 (ADR)

### 8.1 为什么没有独立的 Leverage Keeper?

**问题**: Leverage 功能是否需要独立模块?

**备选方案**:

1. **方案 A: 独立 Leverage 模块**
   - 优势: 模块职责清晰,便于维护
   - 劣势:
     - 增加模块间通信开销
     - Leverage 功能简单,不需要复杂逻辑
     - 与 CLOB 和 Subaccounts 紧密耦合

2. **方案 B: 集成设计 (数据在 CLOB,逻辑分散)** ✅ 最终选择
   - 优势:
     - 减少模块间通信开销
     - 杠杆数据与订单数据同在 CLOB,便于读取
     - 保证金计算在 Subaccounts,职责清晰
   - 劣势:
     - 功能分散,不够集中

**决策理由**: 杠杆功能简单,与订单和保证金紧密耦合,集成设计更高效。

### 8.2 为什么杠杆数据存储在 CLOB 模块?

**问题**: 杠杆数据应该存储在哪个模块?

**备选方案**:

1. **方案 A: 存储在 Subaccounts 模块**
   - 优势: 杠杆影响保证金,逻辑上属于 Subaccounts
   - 劣势: CLOB 需要频繁跨模块读取杠杆数据

2. **方案 B: 存储在 CLOB 模块** ✅ 最终选择
   - 优势:
     - CLOB 是杠杆更新的入口 (MsgUpdateLeverage)
     - CLOB 验证订单时需要读取杠杆,本地读取更快
     - 减少跨模块通信
   - 劣势: 杠杆数据与 CLOB 核心功能关联不强

**决策理由**: 性能优先,CLOB 频繁读取杠杆数据,本地存储更高效。

### 8.3 为什么使用杠杆倍数而非保证金率?

**问题**: 用户界面应该显示杠杆倍数还是保证金率?

**备选方案**:

1. **方案 A: 保证金率 (例如 5%)**
   - 优势: 更直观地表达风险
   - 劣势: 交易所行业惯例使用杠杆倍数

2. **方案 B: 杠杆倍数 (例如 20x)** ✅ 最终选择
   - 优势:
     - 行业标准,用户熟悉
     - 数值更大,视觉冲击力强
     - 便于市场营销 (100x 杠杆)
   - 劣势: 需要转换为保证金率用于计算

**决策理由**: 符合行业惯例,用户体验更好。

### 8.4 为什么需要验证杠杆调整?

**问题**: 是否允许用户随意调整杠杆?

**备选方案**:

1. **方案 A: 允许随意调整**
   - 优势: 用户体验好,自由度高
   - 劣势: 用户可能通过调整杠杆绕过风控

2. **方案 B: 验证保证金充足性后才允许调整** ✅ 最终选择
   - 优势:
     - 防止用户在保证金不足时提高杠杆
     - 确保系统风控要求
     - 降低系统风险
   - 劣势: 用户体验稍差,可能拒绝调整

**决策理由**: 安全性优先,防止用户通过杠杆调整绕过风控。

---

## 9. 集成与扩展点

### 9.1 集成点

**与 CLOB 模块集成**:
- 杠杆数据存储在 CLOB StateStore
- MsgUpdateLeverage 在 CLOB 模块处理
- CLOB 验证订单时读取杠杆数据

**与 Subaccounts 模块集成**:
- Subaccounts 计算保证金时读取杠杆
- Subaccounts 验证杠杆调整的合法性
- 保证金计算逻辑在 Subaccounts

**与 Perpetuals 模块集成**:
- Perpetuals 的 LiquidityTier 配置最大杠杆
- 杠杆限制与流动性层级绑定

### 9.2 扩展点

**动态杠杆限制**:
- 当前: 最大杠杆固定 (LiquidityTier)
- 扩展: 根据市场波动率动态调整最大杠杆
- 实现: 增加动态杠杆计算逻辑

**杠杆分级**:
- 当前: 所有用户相同杠杆限制
- 扩展: VIP 用户可以使用更高杠杆
- 实现: 增加用户等级验证

**自动去杠杆**:
- 当前: 用户手动调整杠杆
- 扩展: 保证金不足时自动降低杠杆
- 实现: 增加自动去杠杆逻辑到清算流程

---

## 10. 总结

### 10.1 关键设计亮点

1. **集成设计**: 无独立 Keeper,逻辑分散到 CLOB 和 Subaccounts,减少通信开销
2. **数据存储优化**: 杠杆数据存储在 CLOB,便于本地读取,提高性能
3. **保证金验证**: 杠杆调整时验证保证金充足性,确保系统安全
4. **行业标准**: 使用杠杆倍数而非保证金率,符合用户习惯

### 10.2 设计权衡

| 设计选择              | 优势                 | 劣势                 | 决策理由             |
|---------------------|---------------------|---------------------|---------------------|
| 集成设计 (无独立 Keeper) | 减少通信开销,高效    | 功能分散,不够集中    | 性能优先             |
| 数据在 CLOB          | 本地读取,性能高      | 逻辑关联不强         | 性能优先             |
| 杠杆倍数表示         | 符合行业惯例         | 需要转换为保证金率   | 用户体验优先         |
| 验证保证金充足性     | 安全性高             | 用户体验稍差         | 安全性优先           |

### 10.3 核心流程总结

**更新杠杆**: 验证权限 → 验证杠杆合法性 → 验证保证金充足 → 更新 StateStore

**保证金计算**: 读取杠杆 → 计算 IMR → 计算保证金要求 → 验证账户净资产

**订单验证**: 读取杠杆 → 计算订单保证金 → 验证总保证金 → 决定是否放置订单

---

**文档版本**: v1.0
**最后更新**: 2025-12-31
**文档作者**: Claude Sonnet 4.5
**文档状态**: ✅ 完成
