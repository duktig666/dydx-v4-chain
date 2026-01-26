# Perpetuals 模块架构设计

## 1. 模块概述

### 1.1 职责定义

Perpetuals (永续合约) 模块是 Hermes DEX 的**核心风险管理和定价机制**,负责:

- **永续合约管理**: 创建、配置和管理永续合约
- **资金费率计算**: 两层时间框架的资金费率机制
- **流动性层级管理**: 动态保证金要求配置
- **未平仓合约量跟踪**: OIMF (开放利息调整保证金系数) 机制

### 1.2 核心功能

**资金费率两层机制**:
1. **Funding Sample Epoch** (1分钟): 收集溢价投票,计算中位数
2. **Funding Tick Epoch** (1小时): 聚合溢价样本,计算平均值,更新资金费率指数

**OIMF 动态保证金调整**:
- 根据未平仓合约量动态调整保证金要求
- 防止市场过度杠杆化
- 保护系统稳定性

**流动性层级系统**:
- 多层级保证金配置
- 支持不同风险等级的合约
- Cross vs Isolated 保证金模式

### 1.3 关键约束

**确定性要求**:
- 资金费率计算必须确定性,所有节点结果一致
- 聚合算法 (中位数、平均值) 必须稳定

**精度要求**:
- 资金费率使用 PPM (百万分之一) 精度
- FundingIndex 使用高精度整数避免浮点误差

**性能要求**:
- 每个 Epoch 处理大量溢价投票
- 聚合算法时间复杂度必须可控

---

## 2. 架构设计

### 2.1 组件结构

```
Perpetuals 模块架构

┌─────────────────────────────────────────────────┐
│         gRPC Query Server (查询接口)             │
│  Perpetual, AllPerpetuals, LiquidityTiers       │
│  PremiumVotes, PremiumSamples, Params          │
└─────────────────────────────────────────────────┘
                       ↕
┌─────────────────────────────────────────────────┐
│         Msg Server (消息处理器)                  │
│  CreatePerpetual, UpdatePerpetualParams         │
│  SetLiquidityTier, AddPremiumVotes              │
└─────────────────────────────────────────────────┘
                       ↕
┌─────────────────────────────────────────────────┐
│              Keeper Layer (业务逻辑层)           │
│  ├─ Perpetual Management                        │
│  ├─ Funding Rate Calculation (两层机制)         │
│  ├─ Liquidity Tier Management                   │
│  ├─ Open Interest Tracking                      │
│  └─ OIMF Calculation                            │
└─────────────────────────────────────────────────┘
                       ↕
┌─────────────────────────────────────────────────┐
│              Storage Layer (存储层)              │
│  ├─ StateStore (永续合约、流动性层级、溢价数据) │
│  └─ TransientStore (未平仓合约量增量)           │
└─────────────────────────────────────────────────┘
```

### 2.2 存储架构

#### StateStore (链上持久化存储)

**存储内容** (6 个核心结构):

1. **Perpetual** (永续合约):
   - 合约参数 (Ticker, MarketId, AtomicResolution)
   - FundingIndex (资金费率指数,累计值)
   - OpenInterest (未平仓合约量)

2. **LiquidityTier** (流动性层级):
   - 保证金率配置 (InitialMarginPpm, MaintenanceFractionPpm)
   - OIMF 参数 (OpenInterestLowerCap, OpenInterestUpperCap)

3. **PremiumVotes** (溢价投票):
   - 当前 1 分钟周期的验证者溢价投票
   - 用于计算溢价样本

4. **PremiumSamples** (溢价样本):
   - 过去 1 小时的溢价样本 (60个)
   - 用于计算资金费率

5. **Params** (模块参数):
   - FundingRateClampFactorPpm
   - PremiumVoteClampFactorPpm
   - MinNumVotesPerSample

6. **NextPerpetualID** (下一个永续合约ID):
   - 自增计数器

**为什么使用 StateStore?**

**持久化需求**:
- 永续合约配置需要跨区块持久化
- FundingIndex 是累计值,必须持久化
- 流动性层级配置需要长期保存

**共识保证**:
- 所有节点必须有相同的合约配置
- FundingIndex 必须一致,影响账户盈亏计算

#### TransientStore (区块级瞬态存储)

**存储内容** (1 个结构):

**UpdatedPerpetualOpenInterest** (已更新的未平仓合约量):
- 存储当前区块更新的未平仓合约量增量
- 区块结束后清空

**为什么使用 TransientStore?**

**性能优化**:
- 未平仓合约量频繁更新
- 不需要持久化增量,只需要最终值

**区块级聚合**:
- 当前区块的所有增量聚合后更新 StateStore
- 减少 StateStore 写入次数

### 2.3 ABCI 生命周期集成

```
ABCI 生命周期中的 Perpetuals 模块

┌──────────────────────────────────────────────┐
│            PreBlocker                        │
│            (无操作)                          │
└──────────────────────────────────────────────┘
                     │
┌──────────────────────────────────────────────┐
│            BeginBlocker                      │
│            (无操作)                          │
└──────────────────────────────────────────────┘
                     │
┌──────────────────────────────────────────────┐
│            DeliverTx                         │
│  ├─ MsgAddPremiumVotes (每个区块一次)       │
│  │   └─ 收集验证者的溢价投票                │
│  │                                           │
│  └─ 订单成交时调用 ModifyOpenInterest()     │
│      └─ 更新 TransientStore 中的 OI 增量    │
└──────────────────────────────────────────────┘
                     │
┌──────────────────────────────────────────────┐
│            EndBlocker ⭐ 核心                │
│  ├─ 检查 Funding Sample Epoch (1分钟)       │
│  │   └─ 聚合溢价投票 → 生成溢价样本         │
│  │                                           │
│  ├─ 检查 Funding Tick Epoch (1小时)         │
│  │   └─ 聚合溢价样本 → 更新资金费率指数     │
│  │                                           │
│  └─ 更新未平仓合约量 (从 TransientStore)    │
│      └─ OI = OI + Delta                     │
└──────────────────────────────────────────────┘
```

#### EndBlocker - 为什么资金费率在这里计算?

**原因分析**:

**Epoch 周期性**:
- Funding Sample Epoch: 每 60 秒 (约 600 个区块)
- Funding Tick Epoch: 每 3600 秒 (约 3600 个区块)
- EndBlocker 是检查 Epoch 结束的理想位置

**聚合操作**:
- 需要收集当前 Epoch 的所有数据
- 区块结束时数据已全部收集完成
- 可以统一聚合和更新

**状态更新原子性**:
- 资金费率更新影响所有账户盈亏
- 必须在区块边界原子性更新
- EndBlocker 保证原子性

**设计权衡**:
- ✅ 优势: 逻辑清晰,原子性保证
- ❌ 劣势: EndBlocker 执行时间长
- 决策理由: 正确性和一致性优先

---

## 3. 核心业务流程

### 3.1 资金费率计算流程 (两层时间框架)

#### 为什么使用两层周期机制?

**问题**: 如何设计资金费率计算机制?

**备选方案**:

1. **方案 A: 单层周期** (直接每小时计算资金费率)
   - ✅ 优势: 简单,实现容易
   - ❌ 劣势: 数据波动大,容易被操纵

2. **方案 B: 两层周期** ✅ 最终选择
   - ✅ 优势: 平滑数据,抗操纵
   - ❌ 劣势: 复杂度高,存储开销大

**权衡分析**:

**单层周期问题**:
```
假设每小时直接收集溢价投票:
- 恶意验证者可以在某个时刻提交极端值
- 单个极端值直接影响资金费率
- 资金费率波动大,不稳定
```

**两层周期优势**:
```
第一层 (1分钟):
- 收集溢价投票
- 计算中位数 → 过滤极端值

第二层 (1小时):
- 收集 60 个溢价样本 (每分钟一个)
- 计算平均值 → 平滑波动
```

**决策理由**:
- 中位数过滤极端值 (抗操纵)
- 平均值平滑波动 (稳定性)
- 两层机制提供更可靠的资金费率

#### 3.1.1 Funding Sample Epoch (1分钟周期)

```
                Funding Sample Epoch 流程

┌─────────────────────────────────────────────────┐
│    每个区块: 验证者提交溢价投票                  │
│    MsgAddPremiumVotes                           │
│    ├─ Proposer 计算订单簿溢价                   │
│    │   溢价 = (订单簿中间价 - 预言机价格) / 预言机价格 │
│    └─ 存储到 PremiumVotesKey                   │
└───────────────────┬─────────────────────────────┘
                    │
                    │ (每 60 秒,约 600 个区块)
                    ▼
┌─────────────────────────────────────────────────┐
│    EndBlocker: Funding Sample Epoch 结束        │
│    ├─ 收集所有 PremiumVotes                     │
│    ├─ 对每个永续合约计算中位数                  │
│    │   Median(vote1, vote2, ..., voteN)        │
│    ├─ 生成 PremiumSample                        │
│    ├─ 附加到 PremiumSamplesKey                  │
│    └─ 清空 PremiumVotesKey                      │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
            溢价样本已生成
        (等待 Funding Tick Epoch 聚合)
```

**为什么溢价用中位数?**

**原因分析**:

**中位数特性**:
- 对极端值不敏感
- 50% 的数据在中位数之下,50% 在中位数之上
- 难以通过少数节点操纵

**平均值问题**:
- 极端值会拉高或拉低平均值
- 少数恶意节点可以显著影响结果

**示例对比**:
```
溢价投票: [100, 105, 110, 115, 1000]  (1个极端值)

平均值: (100+105+110+115+1000) / 5 = 286
中位数: 110

中位数更能代表真实市场溢价
```

**决策理由**: 中位数抗操纵,更可靠

#### 3.1.2 Funding Tick Epoch (1小时周期)

```
                Funding Tick Epoch 流程

┌─────────────────────────────────────────────────┐
│    累积溢价样本 (60 个,每分钟一个)              │
│    PremiumSamplesKey 存储                       │
└───────────────────┬─────────────────────────────┘
                    │
                    │ (每 3600 秒,约 3600 个区块)
                    ▼
┌─────────────────────────────────────────────────┐
│    EndBlocker: Funding Tick Epoch 结束          │
│    ├─ 收集 60 个 PremiumSamples                 │
│    ├─ (可选) 移除样本尾部 (top/bottom X%)       │
│    │   当前: RemovedTailSampleRatioPpm = 0      │
│    ├─ 计算平均值                                │
│    │   AvgPremium = Sum(samples) / Count        │
│    ├─ 添加默认资金费率                          │
│    │   FundingRate = AvgPremium + DefaultFundingPpm │
│    ├─ 应用 Clamp (上下限)                       │
│    │   |FundingRate| <= ClampFactor * (IM - MM) │
│    ├─ 计算 FundingIndexDelta                    │
│    │   Delta = FundingRate * (时间/8小时) * 价格 │
│    ├─ 更新 FundingIndex                         │
│    │   FundingIndex += FundingIndexDelta        │
│    ├─ 清空 PremiumSamplesKey                    │
│    └─ 发出 FundingRatesAndIndices 事件         │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
            资金费率已更新
        (影响所有持仓账户盈亏)
```

**为什么资金费率用平均值?**

**原因分析**:

**平均值特性**:
- 平滑多个样本的波动
- 反映一段时间内的平均水平
- 适合计算费率

**中位数不适合**:
- 损失信息 (只保留中间值)
- 不能准确反映平均溢价水平

**示例对比**:
```
溢价样本 (60个): [90, 95, 100, 105, 110, ..., 150]

平均值: 125 (反映整体溢价水平)
中位数: 110 (丢失了高溢价信息)

平均值更适合计算费率
```

**为什么第一层用中位数,第二层用平均值?**

**设计理由**:
- **第一层 (中位数)**: 过滤单个区块的极端投票,抗操纵
- **第二层 (平均值)**: 平滑多个样本,反映真实溢价趋势

### 3.2 OIMF 动态保证金调整机制

#### 为什么需要 OIMF 机制?

**问题**: 固定保证金率的风险

**风险场景**:
```
假设永续合约固定初始保证金率 10%:

市场平静期:
- 未平仓合约量: 1,000 BTC
- 风险: 低

市场狂热期:
- 未平仓合约量: 100,000 BTC (增加 100 倍)
- 风险: 极高 (单边风险,清算风暴)

固定保证金率无法应对风险变化!
```

**OIMF 解决方案**:
- 根据未平仓合约量动态调整保证金要求
- 未平仓合约量越高,保证金要求越高
- 防止市场过度杠杆化

#### OIMF 计算公式

**公式定义**:

```
OIMF (Open Interest-adjusted Initial Margin Fraction):

if OI <= OpenInterestLowerCap:
    OIMF = 0% (无调整)

elif OpenInterestLowerCap < OI < OpenInterestUpperCap:
    OIMF = (OI - LowerCap) / (UpperCap - LowerCap) * 100%
    (线性插值)

else:  # OI >= UpperCap
    OIMF = 100% (最大调整)

调整后初始保证金率:
AdjustedIMR = BaseIMR + OIMF * (BaseIMR - MaintenanceMarginRate)
```

**变量说明**:
- **OI**: 当前未平仓合约量 (Open Interest)
- **LowerCap**: OI 下限,低于此值不调整
- **UpperCap**: OI 上限,高于此值最大调整
- **BaseIMR**: 基础初始保证金率 (来自 LiquidityTier.InitialMarginPpm)
- **MaintenanceMarginRate**: 维持保证金率

**计算示例**:

```
假设:
- BaseIMR = 10% (10x 杠杆)
- MaintenanceMarginRate = 5%
- LowerCap = 1,000 BTC
- UpperCap = 10,000 BTC

场景 1: OI = 500 BTC (< LowerCap)
- OIMF = 0%
- AdjustedIMR = 10% + 0% * (10% - 5%) = 10%
- 杠杆: 10x (无调整)

场景 2: OI = 5,500 BTC (中间)
- OIMF = (5,500 - 1,000) / (10,000 - 1,000) = 50%
- AdjustedIMR = 10% + 50% * (10% - 5%) = 12.5%
- 杠杆: 8x (降低杠杆)

场景 3: OI = 15,000 BTC (> UpperCap)
- OIMF = 100%
- AdjustedIMR = 10% + 100% * (10% - 5%) = 15%
- 杠杆: 6.67x (显著降低杠杆)
```

**业务解释**:

当市场未平仓合约量过高时,系统会自动提高保证金要求,降低允许的最大杠杆倍数,从而:
1. 减少新增杠杆仓位
2. 降低市场单边风险
3. 防止清算风暴

**设计权衡**:
- ✅ 优势: 动态风险管理,防止系统性风险
- ❌ 劣势: 高 OI 时用户杠杆受限
- 决策理由: 系统稳定性优先

### 3.3 未平仓合约量更新流程

```
            未平仓合约量更新流程

┌─────────────────────────────────────────────────┐
│    订单成交 (CLOB 模块)                          │
│    ├─ Fill: BuyOrder 成交 1 BTC                 │
│    │   → Long 仓位增加 1 BTC                    │
│    └─ Fill: SellOrder 成交 1 BTC                │
│        → Short 仓位增加 1 BTC                   │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────┐
│    CLOB 调用 Perpetuals.ModifyOpenInterest()   │
│    ├─ Delta = +1 BTC (Long 增加)               │
│    └─ 存储到 TransientStore                     │
│        UpdatedPerpetualOpenInterest             │
└───────────────────┬─────────────────────────────┘
                    │
                    │ (当前区块其他成交也更新)
                    │
                    ▼
┌─────────────────────────────────────────────────┐
│    EndBlocker                                   │
│    ├─ 读取 TransientStore 中的 Delta           │
│    ├─ 聚合当前区块所有 Delta                   │
│    │   TotalDelta = Sum(Delta1, Delta2, ...)   │
│    ├─ 更新 StateStore 中的 OI                   │
│    │   NewOI = OldOI + TotalDelta              │
│    ├─ 验证 NewOI >= 0                           │
│    ├─ 清空 TransientStore                       │
│    └─ 发出 OpenInterestUpdate 事件             │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
            OI 更新完成
        (影响 OIMF 计算)
```

**为什么 OI 更新分两步?**

**原因分析**:

**频繁更新问题**:
- 每笔成交都更新 StateStore → 写入开销大
- 同一个区块可能有数百笔成交

**两步优化**:
1. **TransientStore 暂存增量**: 每笔成交快速写入 TransientStore
2. **EndBlocker 聚合更新**: 区块结束统一更新 StateStore

**设计权衡**:
- ✅ 优势: 减少 StateStore 写入次数,提升性能
- ❌ 劣势: 增加一层存储,逻辑复杂
- 决策理由: 性能优化优先

---

## 4. 模块依赖关系

### 4.1 上游依赖

#### 4.1.1 Prices Keeper

**依赖原因**: 获取市场价格

**调用场景**:
- **溢价计算**: `溢价 = (订单簿中间价 - 预言机价格) / 预言机价格`
- **FundingIndexDelta 计算**: 需要市场价格转换为 quote quantums

**关键接口**:
```go
GetMarketPrice(ctx, marketId) -> MarketPrice
```

#### 4.1.2 Epochs Keeper

**依赖原因**: 获取 Epoch 信息

**调用场景**:
- 判断 Funding Sample Epoch 是否结束
- 判断 Funding Tick Epoch 是否结束

**关键接口**:
```go
GetEpochInfo(ctx, epochIdentifier) -> EpochInfo
```

#### 4.1.3 CLOB Keeper

**依赖原因**: 订单簿交互

**调用场景**:
- 订单成交时调用 `ModifyOpenInterest()` 更新 OI

**关键接口**:
```go
ModifyOpenInterest(ctx, perpetualId, delta) -> error
```

### 4.2 下游被依赖

#### 4.2.1 Subaccounts 模块

**依赖原因**: 保证金计算

**调用场景**:
- Subaccounts 调用 `GetPerpetual()` 获取合约配置
- Subaccounts 调用 `GetLiquidityTier()` 获取保证金率
- Subaccounts 使用 OIMF 机制计算调整后的保证金要求

#### 4.2.2 CLOB 模块

**依赖原因**: 订单验证

**调用场景**:
- CLOB 调用 `GetPerpetual()` 验证订单关联的永续合约存在
- CLOB 调用 `ModifyOpenInterest()` 更新 OI

---

## 5. 数据流设计

### 5.1 资金费率完整数据流

```
                    资金费率完整数据流

┌─────────────────────────────────────────────────┐
│         每个区块: Proposer 提交溢价投票          │
│         MsgAddPremiumVotes                      │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
         存储到 PremiumVotesKey
                    │
                    │ (累积到 Funding Sample Epoch 结束)
                    ▼
┌─────────────────────────────────────────────────┐
│    Funding Sample Epoch 结束 (每 60 秒)         │
│    ├─ 聚合所有溢价投票                          │
│    ├─ 计算中位数                                │
│    └─ 生成溢价样本                              │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
        存储到 PremiumSamplesKey
                    │
                    │ (累积到 Funding Tick Epoch 结束)
                    ▼
┌─────────────────────────────────────────────────┐
│    Funding Tick Epoch 结束 (每 3600 秒)         │
│    ├─ 聚合 60 个溢价样本                        │
│    ├─ 计算平均值                                │
│    ├─ 计算资金费率                              │
│    ├─ 更新 FundingIndex                         │
│    └─ 清空溢价样本                              │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
            FundingIndex 已更新
                    │
        ┌───────────┼───────────┐
        │           │           │
        ▼           ▼           ▼
 Subaccounts    Indexer      前端显示
  (盈亏计算)     (历史数据)    (费率展示)
```

### 5.2 未平仓合约量数据流

```
            未平仓合约量数据流

┌─────────────────────────────────────────────────┐
│         订单成交 (CLOB 模块)                     │
│         每笔成交更新 OI                          │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
        ModifyOpenInterest(perpetualId, delta)
                    │
                    ▼
         存储到 TransientStore
    (UpdatedPerpetualOpenInterest)
                    │
                    │ (区块内累积)
                    ▼
┌─────────────────────────────────────────────────┐
│         EndBlocker                              │
│         ├─ 聚合当前区块所有 Delta               │
│         ├─ 更新 StateStore 中的 OI              │
│         └─ 清空 TransientStore                  │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
            OI 已更新
                    │
        ┌───────────┼───────────┐
        │           │           │
        ▼           ▼           ▼
OIMF 计算     Indexer      前端显示
(保证金调整)  (历史数据)    (OI 展示)
```

---

## 6. 性能与可扩展性

### 6.1 性能优化点

#### 6.1.1 溢价数据压缩

**问题**: 溢价投票和样本数据量大

**优化方案**:

```go
// 压缩存储
type CompactPremiumVote struct {
    PerpetualId uint32    // 4 字节
    PremiumPpm  int32     // 4 字节 (PPM 精度)
}

// 而非
type PremiumVote struct {
    PerpetualId uint32
    Premium     sdk.Dec   // 16+ 字节 (高精度 Decimal)
}
```

**压缩效果**:
- 原始: 20 字节/投票
- 压缩: 8 字节/投票
- 节省: 60% 存储

#### 6.1.2 聚合算法优化

**中位数计算**:
```go
// O(n log n) 排序算法
sort.Slice(votes, func(i, j int) bool {
    return votes[i] < votes[j]
})
median := votes[len(votes)/2]
```

**平均值计算**:
```go
// O(n) 线性算法
sum := int64(0)
for _, sample := range samples {
    sum += sample
}
average := sum / int64(len(samples))
```

### 6.2 扩展性设计

#### 6.2.1 支持更多流动性层级

**扩展步骤**:
1. 创建新的 LiquidityTier
2. 配置保证金率和 OIMF 参数
3. 关联到永续合约

**无需代码修改**: 配置驱动,灵活扩展

#### 6.2.2 调整资金费率计算参数

**可配置参数**:
- FundingRateClampFactorPpm (资金费率上下限系数)
- PremiumVoteClampFactorPpm (溢价投票上下限系数)
- MinNumVotesPerSample (最少投票数要求)
- RemovedTailSampleRatioPpm (移除样本尾部比例)

**通过治理调整**: 无需升级合约

---

## 7. 技术决策记录 (ADR)

### 7.1 为什么使用两层周期机制?

(详见 3.1 节)

**决策理由**: 抗操纵 + 平滑波动

### 7.2 为什么溢价用中位数,资金费率用平均值?

(详见 3.1.1 和 3.1.2 节)

**决策理由**:
- 中位数: 过滤极端值,抗操纵
- 平均值: 平滑波动,反映趋势

### 7.3 为什么需要 OIMF 机制?

(详见 3.2 节)

**决策理由**: 动态风险管理,防止系统性风险

### 7.4 为什么支持 Cross 和 Isolated 保证金模式?

**问题**: 保证金模式应该如何设计?

**备选方案**:

1. **方案 A: 仅 Cross 模式** (所有合约共享保证金)
   - ✅ 优势: 资金利用率高
   - ❌ 劣势: 单个合约爆仓影响全部

2. **方案 B: 仅 Isolated 模式** (每个合约独立保证金)
   - ✅ 优势: 风险隔离
   - ❌ 劣势: 资金利用率低

3. **方案 C: 同时支持** ✅ 最终选择
   - ✅ 优势: 灵活性高,满足不同用户需求
   - ❌ 劣势: 实现复杂度高

**Cross 模式特点**:
```
用户持有多个永续合约:
- BTC-USD: Long 1 BTC
- ETH-USD: Short 10 ETH

保证金共享:
- BTC 盈利可以弥补 ETH 亏损
- 总保证金要求 = Sum(各合约保证金要求)
```

**Isolated 模式特点**:
```
每个合约独立保证金:
- BTC-USD: 独立保证金池 A
- ETH-USD: 独立保证金池 B

风险隔离:
- BTC 爆仓不影响 ETH
- 每个合约单独计算保证金率
```

**决策理由**:
- 专业交易者偏好 Cross (资金利用率)
- 保守交易者偏好 Isolated (风险隔离)
- 同时支持满足不同需求

---

## 8. 总结

### 8.1 关键设计亮点

1. **两层资金费率机制**: 中位数过滤 + 平均值平滑
2. **OIMF 动态保证金**: 根据 OI 自动调整风险
3. **流动性层级系统**: 灵活的保证金配置
4. **Cross/Isolated 模式**: 满足不同风险偏好

### 8.2 设计权衡

| 设计选择              | 优势                 | 劣势                 | 决策理由             |
|---------------------|---------------------|---------------------|---------------------|
| 两层周期机制         | 抗操纵,平滑波动       | 复杂度高,延迟增加     | 可靠性优先           |
| OIMF 机制           | 动态风险管理         | 高 OI 时杠杆受限     | 系统稳定性优先       |
| TransientStore OI   | 性能优化             | 增加存储层次         | 性能优先             |
| Cross/Isolated 支持  | 灵活性高             | 实现复杂             | 用户需求优先         |

### 8.3 与 CLOB 模块的协作

**紧密集成**:
- CLOB 订单成交 → 更新 OI
- CLOB 保证金验证 → 查询 LiquidityTier
- CLOB 清算判断 → 使用 OIMF 调整保证金

**数据流向**:
- Perpetuals → CLOB: 提供合约配置、保证金率
- CLOB → Perpetuals: 更新 OI、触发资金费率计算

---

**文档版本**: v1.0
**最后更新**: 2025-12-31
**文档作者**: Claude Sonnet 4.5
**文档状态**: ✅ 完成