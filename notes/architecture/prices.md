# Prices 模块架构设计

## 1. 模块概述

### 1.1 职责定义

Prices (价格预言机) 模块是 Hermes DEX 的**价格数据源**,负责:

- **价格市场管理**: 创建和配置价格市场
- **Slinky Oracle 集成**: 通过 Vote Extensions 获取去中心化价格
- **价格验证**: 确保价格更新的有效性和准确性
- **价格查询服务**: 为其他模块提供实时价格数据

### 1.2 核心功能

**Slinky Oracle 集成**:
- ExtendVote: 验证器从 Slinky 获取价格
- VerifyVoteExtension: 验证价格数据有效性
- PrepareProposal: Proposer 聚合验证器价格
- ProcessProposal: 验证聚合价格
- FinalizeBlock: 更新链上价格

**价格验证机制**:
- Towards Condition: 价格朝向索引价格移动
- Crossing Condition: 价格跨越索引价格的缓冲约束
- 最小价格变化要求

### 1.3 关键约束

**确定性要求**:
- 价格更新必须确定性,所有节点一致
- 聚合算法 (中位数) 必须稳定

**精度要求**:
- 价格使用 uint64 + Exponent 表示
- 避免浮点数,确保精确计算

**安全性要求**:
- 防止价格操纵
- 验证价格合理性

---

## 2. 架构设计

### 2.1 组件结构

```
Prices 模块架构

┌─────────────────────────────────────────────────┐
│         gRPC Query Server (查询接口)             │
│  MarketPrice, MarketParam, AllMarkets           │
└─────────────────────────────────────────────────┘
                       ↕
┌─────────────────────────────────────────────────┐
│         Msg Server (消息处理器)                  │
│  CreateOracleMarket, UpdateMarketParam          │
│  UpdateMarketPrices (Slinky 集成)              │
└─────────────────────────────────────────────────┘
                       ↕
┌─────────────────────────────────────────────────┐
│              Keeper Layer (业务逻辑层)           │
│  ├─ Market Management                           │
│  ├─ Price Update Logic                          │
│  ├─ Price Validation (Towards/Crossing)         │
│  └─ Slinky Integration                          │
└─────────────────────────────────────────────────┘
                       ↕
┌─────────────────────────────────────────────────┐
│         Slinky Oracle (Vote Extensions)         │
│  ├─ ExtendVote (获取价格)                       │
│  ├─ VerifyVoteExtension (验证价格)              │
│  ├─ PrepareProposal (聚合价格)                  │
│  └─ ProcessProposal (验证聚合)                  │
└─────────────────────────────────────────────────┘
                       ↕
┌─────────────────────────────────────────────────┐
│              Storage Layer (存储层)              │
│  ├─ StateStore (市场配置、市场价格)              │
│  └─ MarketMap (货币对元数据)                    │
└─────────────────────────────────────────────────┘
```

### 2.2 存储架构

#### StateStore (链上持久化存储)

**存储内容** (4 个数据结构):

1. **MarketParam** (市场参数配置):
   - Id: 市场 ID
   - Pair: 交易对 "BTC/USD"
   - Exponent: 价格指数
   - MinExchanges: 最小交易所数量
   - MinPriceChangePpm: 最小价格变化
   - ExchangeConfigJson: Slinky 配置

2. **MarketPrice** (市场价格数据):
   - Id: 市场 ID
   - Exponent: 价格指数
   - Price: 当前价格 (链上表示)
   - BlockTimestamp: 更新时间戳

3. **CurrencyPairID** (货币对映射):
   - CurrencyPair → MarketId
   - 例如: "BTC/USD" → 0

4. **NextMarketID** (下一个市场ID):
   - 自增计数器

### 2.3 ABCI 生命周期集成 (Slinky Oracle)

```
         Slinky Oracle Vote Extensions 流程

┌─────────────────────────────────────────────────┐
│    ExtendVote (每个验证器)                       │
│    ├─ 从 Slinky 获取最新价格                    │
│    ├─ 验证价格有效性                            │
│    └─ 返回 VoteExtension (包含价格数据)         │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────┐
│    VerifyVoteExtension (验证器验证其他验证器)   │
│    ├─ 检查价格数据格式                          │
│    ├─ 验证价格合理性                            │
│    └─ 返回 ACCEPT/REJECT                        │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────┐
│    PrepareProposal (Proposer)                   │
│    ├─ 收集所有验证器的 VoteExtensions           │
│    ├─ 聚合价格 (计算中位数)                     │
│    ├─ 过滤无效价格                              │
│    ├─ 构建 MsgUpdateMarketPrices                │
│    └─ 包含在区块提案中                          │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────┐
│    ProcessProposal (所有验证器)                 │
│    ├─ 验证 MsgUpdateMarketPrices                │
│    ├─ 检查价格精度 (Towards/Crossing)           │
│    ├─ 验证最小价格变化                          │
│    └─ 返回 ACCEPT/REJECT                        │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────┐
│    FinalizeBlock - DeliverTx                    │
│    ├─ 执行 MsgUpdateMarketPrices                │
│    ├─ 更新 MarketPrice                          │
│    ├─ 记录价格更新时间戳                        │
│    └─ 发出 PriceUpdate 事件                     │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
            价格更新完成
```

#### 为什么使用 Vote Extensions?

**问题**: 如何获取去中心化价格?

**备选方案**:

1. **方案 A: 链下预言机 + 链上提交**
   - ❌ 中心化风险
   - ❌ 需要额外的价格提交交易

2. **方案 B: Vote Extensions** ✅ 最终选择
   - ✅ 去中心化 (每个验证器提交价格)
   - ✅ 无需额外交易 (集成在共识流程)
   - ✅ 聚合机制 (中位数抗操纵)

**Vote Extensions 优势**:
- 验证器在投票时附带价格数据
- Proposer 聚合所有验证器的价格
- 自动包含在区块中,无需额外交易

**决策理由**: 去中心化和效率优先

---

## 3. 核心业务流程

### 3.1 价格更新流程

#### 3.1.1 有效价格更新定义

**必须同时满足以下条件**:

1. **索引价格存在且非零**:
   - Slinky 聚合的索引价格可用

2. **平滑价格存在且非零**:
   - 链上当前价格可用

3. **平滑价格和索引价格在 Oracle 价格同侧**:
   - 防止价格跳跃

4. **提议价格更接近 Oracle 价格**:
   - 价格朝向真实市场价格移动

5. **提议价格满足最小价格变化要求**:
   - 过滤微小波动

#### 3.1.2 价格精度验证

**两阶段验证**:

**阶段 1: 非确定性验证 (ProcessProposal)**:
- 索引价格可用性检查
- 价格精度验证 (Towards 或 Crossing 条件)
- 跨界缓冲约束

**阶段 2: 确定性验证 (DeliverTx)**:
- 市场存在性检查
- 最小价格变化要求检查

#### 3.1.3 Towards Condition (朝向条件)

**定义**:

```
newPrice 在 [min(oldPrice, indexPrice), max(oldPrice, indexPrice)] 范围内
```

**示例**:

```
oldPrice = 100
indexPrice = 105

有效的 newPrice:
- 100 (不变)
- 101, 102, 103, 104, 105 (朝向 indexPrice)

无效的 newPrice:
- 99 (远离 indexPrice)
- 106 (跨越 indexPrice,需要 Crossing Condition)
```

**业务解释**:

价格更新应该朝向真实市场价格 (indexPrice) 移动,而不是远离它。这防止了价格被恶意拉向错误方向。

#### 3.1.4 Crossing Condition (跨越条件)

**当 newPrice 跨越 indexPrice 时,必须满足缓冲约束**:

**公式**:

```
情况 A: oldDelta <= 1 tick
  newDelta <= oldDelta

情况 B: oldDelta > 1 tick
  newDelta² × 1,000,000 <= oldDelta × tickSizePpm
  即: newDelta <= sqrt(oldDelta × tickSize)

其中:
- tickSize = oldPrice × MinPriceChangePpm / 1,000,000
- oldDelta = |oldPrice - indexPrice|
- newDelta = |indexPrice - newPrice|
```

**示例**:

```
oldPrice = 100
indexPrice = 105
MinPriceChangePpm = 1,000 (0.1%)

tickSize = 100 × 1,000 / 1,000,000 = 0.1
oldDelta = |100 - 105| = 5

情况 B 约束:
newDelta² × 1,000,000 <= 5 × 1,000
newDelta² <= 0.005
newDelta <= 0.071

允许的 newPrice:
- [104.929, 105.071] (indexPrice ± 0.071)
```

**业务解释**:

当价格跨越索引价格时,不能跨越太多,必须在一个缓冲范围内。这防止了价格剧烈跳跃。

缓冲范围随着 oldDelta 增加而增加,但增速放缓 (平方根关系),确保价格逐步向索引价格收敛。

**为什么使用平方根缓冲?**

**原因分析**:

**线性缓冲问题**:
```
如果 newDelta <= oldDelta:
- oldDelta = 100 → newDelta <= 100 (允许跨越 100)
- 价格可以一次跨越很远
```

**平方根缓冲优势**:
```
newDelta <= sqrt(oldDelta × tickSize):
- oldDelta = 100, tickSize = 0.1 → newDelta <= 3.16
- 价格只能逐步收敛,不能一次跨越太远
```

**决策理由**: 防止价格剧烈波动,保护用户

### 3.2 市场创建流程

```
        CreateOracleMarket 流程

┌─────────────────────────────────────────────────┐
│    Authority 提交 MsgCreateOracleMarket         │
│    ├─ Ticker: "BTC/USD"                         │
│    └─ Authority 验证                            │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────┐
│    从 MarketMap 获取货币对指数 (Exponent)        │
│    MarketMapKeeper.GetMarket(ticker)            │
│    └─ Decimals → Exponent = -Decimals           │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────┐
│    创建初始零价格的 MarketPrice                  │
│    Price = 0 (等待 Slinky 更新)                 │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────┐
│    调用 CreateMarket()                          │
│    ├─ 检查市场 ID 唯一性                        │
│    ├─ 验证 MarketParam 格式                     │
│    ├─ 验证 MarketPrice 格式                     │
│    ├─ 检查 Pair 唯一性                          │
│    ├─ 转换 Pair 到 CurrencyPair                 │
│    ├─ 从 MarketMap 验证货币对存在               │
│    ├─ 验证 Exponent = -Decimals                 │
│    ├─ 持久化 MarketParam 和 MarketPrice         │
│    ├─ 添加 CurrencyPairID 映射                  │
│    ├─ 生成 Indexer 事件                         │
│    └─ 在 MarketMap 启用该市场                   │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
            市场创建完成
```

---

## 4. 模块依赖关系

### 4.1 上游依赖

#### 4.1.1 MarketMap Keeper

**依赖原因**: 货币对元数据

**调用场景**:
- 市场创建时获取 Decimals
- 验证货币对存在
- 启用/禁用市场

#### 4.1.2 RevShare Keeper

**依赖原因**: 市场创建时创建 RevShare 条目

#### 4.1.3 Pricefeed

**依赖原因**: Slinky 价格数据源

### 4.2 下游被依赖

#### 4.2.1 Perpetuals 模块

**依赖原因**: 资金费率计算需要市场价格

#### 4.2.2 CLOB 模块

**依赖原因**: 清算判断需要市场价格

#### 4.2.3 Subaccounts 模块

**依赖原因**: 保证金计算需要市场价格

---

## 5. 技术决策记录 (ADR)

### 5.1 为什么使用 Exponent 机制?

**问题**: 如何表示价格精度?

**备选方案**:

1. **方案 A: 浮点数**
   - ❌ 精度损失
   - ❌ 不确定性 (浮点运算)

2. **方案 B: uint64 + Exponent** ✅ 最终选择
   - ✅ 精确计算
   - ✅ 确定性
   - ✅ 灵活精度

**Exponent 机制**:

```
链上价格 = Price × 10^Exponent

示例:
Price = 50,000,000,000
Exponent = -5
实际价格 = 50,000,000,000 × 10^(-5) = 50,000 USDC
```

**决策理由**: 精确性和确定性优先

### 5.2 为什么需要 Towards 和 Crossing 条件?

(详见 3.1.3 和 3.1.4 节)

**决策理由**: 防止价格操纵和剧烈波动

---

## 6. 总结

### 6.1 关键设计亮点

1. **Slinky Oracle 集成**: 去中心化价格源
2. **Vote Extensions**: 高效的价格聚合机制
3. **价格验证**: Towards/Crossing 条件防止操纵
4. **Exponent 机制**: 精确的价格表示

### 6.2 设计权衡

| 设计选择              | 优势                 | 劣势                 | 决策理由             |
|---------------------|---------------------|---------------------|---------------------|
| Vote Extensions     | 去中心化,高效         | 复杂度高             | 去中心化优先         |
| 价格验证            | 防止操纵             | 可能延迟价格更新      | 安全性优先           |
| Exponent 机制       | 精确,确定性           | 需要转换计算         | 精确性优先           |

---

**文档版本**: v1.0
**最后更新**: 2025-12-31
**文档作者**: Claude Sonnet 4.5
**文档状态**: ✅ 完成