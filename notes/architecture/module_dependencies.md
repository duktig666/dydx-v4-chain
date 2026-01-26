# Hermes DEX 模块依赖关系图

## 1. 概述

本文档描述 Hermes DEX 交易系统 7 个核心模块之间的依赖关系、数据流向和协作模式。

**7 个核心模块**:
1. **CLOB** - 中央限价订单簿 (核心交易引擎)
2. **Perpetuals** - 永续合约配置与资金费率
3. **Listing** - 市场上市与创建
4. **Prices** - 价格预言机集成
5. **Stats** - 交易统计与手续费分级
6. **Vault** - 自动化做市与流动性管理
7. **Leverage** - 杠杆管理 (集成模块)

---

## 2. 模块依赖关系图

### 2.1 完整依赖关系图

```
                Hermes DEX 模块依赖关系

┌─────────────────────────────────────────────────────────┐
│                    Listing 模块                          │
│  (市场创建入口,创建 Perpetual + Market + CLOBPair)      │
└───────────────────┬─────────────────────────────────────┘
                    │ 创建市场
                    ▼
┌─────────────────────────────────────────────────────────┐
│                 Perpetuals 模块                          │
│  ├─ 永续合约定义                                        │
│  ├─ 资金费率计算 (Funding Rate)                         │
│  └─ 流动性层级 (LiquidityTier)                          │
└───────┬──────────────────────────────┬──────────────────┘
        │ 提供合约配置                  │ 提供 OIMF 参数
        ▼                              ▼
┌──────────────────┐          ┌──────────────────┐
│   CLOB 模块       │          │  Subaccounts 模块 │
│  ├─ 订单簿        │◀────────▶│  ├─ 保证金计算    │
│  ├─ 订单匹配      │  资金流   │  ├─ 杠杆管理      │
│  ├─ 清算          │          │  └─ 净值计算      │
│  └─ Leverage 数据 │          │                  │
└───────┬───────────┘          └────────┬─────────┘
        │ 记录成交                      │ 提供净值
        ▼                              ▼
┌──────────────────┐          ┌──────────────────┐
│   Stats 模块      │          │   Vault 模块      │
│  ├─ 成交统计      │          │  ├─ 份额管理      │
│  ├─ 手续费分级    │          │  ├─ 自动做市      │
│  └─ 联盟费用      │          │  └─ 存提款管理    │
└──────────────────┘          └────────┬─────────┘
                                       │ 获取价格
                                       ▼
                              ┌──────────────────┐
                              │   Prices 模块     │
                              │  ├─ Slinky Oracle │
                              │  ├─ 价格验证      │
                              │  └─ 价格存储      │
                              └──────────────────┘
```

### 2.2 模块分层架构

```
        Hermes DEX 分层架构

┌─────────────────────────────────────────────────┐
│               应用层 (Application Layer)         │
│  ├─ Listing (市场创建)                          │
│  └─ Vault (自动做市)                            │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────┐
│               核心层 (Core Layer)                │
│  ├─ CLOB (订单匹配引擎)                         │
│  ├─ Subaccounts (账户与保证金)                  │
│  └─ Stats (交易统计)                            │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────┐
│               基础层 (Foundation Layer)          │
│  ├─ Perpetuals (合约配置)                       │
│  ├─ Prices (价格预言机)                         │
│  └─ Leverage (杠杆管理 - 集成)                  │
└─────────────────────────────────────────────────┘
```

---

## 3. 模块依赖关系详解

### 3.1 CLOB 模块依赖

**上游依赖** (CLOB 依赖哪些模块):

| 依赖模块        | 用途                          | 调用接口示例                          |
|---------------|-------------------------------|--------------------------------------|
| Subaccounts   | 验证保证金,转账资金            | `CanPlaceOrder()`, `UpdateSubaccount()` |
| Perpetuals    | 获取合约配置                   | `GetPerpetual()`, `GetLiquidityTier()` |
| Prices        | 获取预言机价格 (清算判断)      | `GetMarketPrice()`                   |
| Stats         | 记录成交统计                   | `RecordFill()`                       |
| Epochs        | 获取当前 Epoch (资金费率)      | `GetCurrentEpoch()`                  |

**下游被依赖** (哪些模块依赖 CLOB):

| 依赖模块        | 用途                          | 调用接口示例                          |
|---------------|-------------------------------|--------------------------------------|
| Vault         | 放置和取消订单 (自动做市)      | `PlaceShortTermOrder()`, `CancelOrder()` |
| Indexer       | 索引订单和成交事件             | 订阅 `OrderPlacedEvent`, `FillEvent` |
| 前端 UI       | 展示订单簿和成交记录           | `GetOrderbook()`, `GetFills()`       |

**依赖关系图**:

```
    Subaccounts ──> CLOB (验证保证金)
    Perpetuals  ──> CLOB (提供合约配置)
    Prices      ──> CLOB (提供价格)
    Stats       <── CLOB (记录成交)
    Vault       <── CLOB (放置订单)
```

### 3.2 Perpetuals 模块依赖

**上游依赖**:

| 依赖模块        | 用途                          | 调用接口示例                          |
|---------------|-------------------------------|--------------------------------------|
| Prices        | 获取指数价格 (计算溢价)        | `GetMarketPrice()`                   |
| Epochs        | 获取 Funding Epoch 信息        | `GetCurrentEpoch()`                  |

**下游被依赖**:

| 依赖模块        | 用途                          | 调用接口示例                          |
|---------------|-------------------------------|--------------------------------------|
| CLOB          | 获取合约配置                   | `GetPerpetual()`, `GetLiquidityTier()` |
| Subaccounts   | 获取 OIMF 参数 (保证金计算)    | `GetPerpetual()`, `GetLiquidityTier()` |
| Listing       | 创建新永续合约                 | `CreatePerpetual()`                  |

**依赖关系图**:

```
    Prices ──> Perpetuals (提供指数价格)
    Epochs ──> Perpetuals (提供 Epoch 信息)

    Perpetuals ──> CLOB (提供合约配置)
    Perpetuals ──> Subaccounts (提供 OIMF 参数)
    Perpetuals <── Listing (创建合约)
```

### 3.3 Listing 模块依赖

**上游依赖**:

| 依赖模块        | 用途                          | 调用接口示例                          |
|---------------|-------------------------------|--------------------------------------|
| Perpetuals    | 创建永续合约                   | `CreatePerpetual()`                  |
| Prices        | 创建价格市场                   | `CreateMarket()`                     |
| CLOB          | 创建交易对                     | `CreateCLOBPair()`                   |
| Vault         | 存入初始流动性                 | `DepositToMegavault()`               |

**下游被依赖**:

| 依赖模块        | 用途                          | 调用接口示例                          |
|---------------|-------------------------------|--------------------------------------|
| 前端 UI       | 触发市场创建                   | `MsgCreateMarket`                    |

**依赖关系图**:

```
    Listing ──> Perpetuals (创建合约)
    Listing ──> Prices (创建市场)
    Listing ──> CLOB (创建交易对)
    Listing ──> Vault (存入流动性)
```

### 3.4 Prices 模块依赖

**上游依赖**:

| 依赖模块        | 用途                          | 调用接口示例                          |
|---------------|-------------------------------|--------------------------------------|
| Slinky Oracle | 获取预言机价格 (Vote Extensions) | Vote Extensions 机制                  |

**下游被依赖**:

| 依赖模块        | 用途                          | 调用接口示例                          |
|---------------|-------------------------------|--------------------------------------|
| CLOB          | 获取价格 (清算判断)            | `GetMarketPrice()`                   |
| Perpetuals    | 获取指数价格 (计算溢价)        | `GetMarketPrice()`                   |
| Vault         | 获取价格 (订单报价)            | `GetMarketPrice()`                   |
| Subaccounts   | 获取价格 (保证金计算)          | `GetMarketPrice()`                   |

**依赖关系图**:

```
    Slinky Oracle ──> Prices (提供预言机价格)

    Prices ──> CLOB (提供价格)
    Prices ──> Perpetuals (提供指数价格)
    Prices ──> Vault (提供报价价格)
    Prices ──> Subaccounts (提供保证金价格)
```

### 3.5 Stats 模块依赖

**上游依赖**:

| 依赖模块        | 用途                          | 调用接口示例                          |
|---------------|-------------------------------|--------------------------------------|
| Epochs        | 获取当前 Epoch (统计聚合)      | `GetCurrentEpoch()`                  |

**下游被依赖**:

| 依赖模块        | 用途                          | 调用接口示例                          |
|---------------|-------------------------------|--------------------------------------|
| CLOB          | 记录成交统计                   | `RecordFill()`                       |
| Fees          | 计算手续费折扣                 | `GetUserStats()`                     |
| 前端 UI       | 展示交易统计                   | `GetUserStats()`, `GetGlobalStats()` |

**依赖关系图**:

```
    Epochs ──> Stats (提供 Epoch 信息)

    CLOB ──> Stats (记录成交)
    Stats ──> Fees (提供统计数据)
```

### 3.6 Vault 模块依赖

**上游依赖**:

| 依赖模块        | 用途                          | 调用接口示例                          |
|---------------|-------------------------------|--------------------------------------|
| Subaccounts   | 获取 Vault 净值                | `GetNetCollateral()`                 |
| CLOB          | 放置和取消订单                 | `PlaceShortTermOrder()`, `CancelOrder()` |
| Perpetuals    | 获取交易对列表                 | `GetAllPerpetuals()`                 |
| Prices        | 获取预言机价格 (订单报价)      | `GetMarketPrice()`                   |
| Bank          | 转账 USDC (存提款)             | `SendCoins()`                        |

**下游被依赖**:

| 依赖模块        | 用途                          | 调用接口示例                          |
|---------------|-------------------------------|--------------------------------------|
| Listing       | 存入初始流动性                 | `DepositToMegavault()`               |
| 前端 UI       | 展示 Vault 状态                | `GetTotalShares()`, `GetOwnerShares()` |

**依赖关系图**:

```
    Subaccounts ──> Vault (提供净值)
    CLOB        <── Vault (放置订单)
    Perpetuals  ──> Vault (提供市场列表)
    Prices      ──> Vault (提供价格)
    Bank        <── Vault (转账)

    Vault <── Listing (初始流动性)
```

### 3.7 Leverage 模块依赖 (集成模块)

**上游依赖**:

| 依赖模块        | 用途                          | 调用接口示例                          |
|---------------|-------------------------------|--------------------------------------|
| Subaccounts   | 验证保证金充足性               | `CanUpdateLeverage()`                |
| Perpetuals    | 获取最大杠杆限制               | `GetLiquidityTier()`                 |

**下游被依赖**:

| 依赖模块        | 用途                          | 调用接口示例                          |
|---------------|-------------------------------|--------------------------------------|
| Subaccounts   | 读取杠杆计算保证金             | `CLOB.GetLeverage()`                 |
| 前端 UI       | 展示杠杆设置                   | `GetLeverage()`                      |

**依赖关系图**:

```
    Subaccounts ──> Leverage (验证保证金)
    Perpetuals  ──> Leverage (提供限制)

    Leverage (数据存储在 CLOB)
    Leverage ──> Subaccounts (提供杠杆数据)
```

---

## 4. 数据流分析

### 4.1 订单放置与匹配数据流

```
        完整订单生命周期数据流

用户
  │ MsgPlaceOrder
  ▼
CLOB.MsgServer
  │ 验证订单参数
  ▼
CLOB.Keeper
  │ 读取合约配置
  ├──> Perpetuals.GetPerpetual(perpetualId)
  │    └─ 返回 PerpetualParams (Ticker, LiquidityTier, ...)
  │
  │ 验证保证金
  ├──> Subaccounts.CanPlaceOrder(order)
  │    ├─ 读取杠杆 CLOB.GetLeverage(subaccountId)
  │    ├─ 计算保证金要求 (IMR = 1 / Leverage)
  │    └─ 返回验证结果
  │
  │ 放置订单到 MemClob
  ▼
MemClob
  │ 订单进入订单簿
  │ 尝试匹配
  ▼
匹配引擎
  │ 价格-时间优先匹配
  │ 生成 Fill
  ▼
CLOB.Keeper
  │ 处理成交
  ├──> Subaccounts.UpdateSubaccount(taker, maker, fill)
  │    └─ 更新余额和持仓
  │
  ├──> Perpetuals.UpdateOpenInterest(perpetualId, delta)
  │    └─ 更新未平仓合约量
  │
  └──> Stats.RecordFill(fill)
       └─ 记录成交统计
```

### 4.2 清算数据流

```
        清算触发 → 执行完整流程

EndBlocker (CLOB)
  │ ProcessDeleveraging()
  ▼
CLOB.Keeper
  │ 获取可能被清算的账户
  ├──> Subaccounts.GetAccountsForLiquidation()
  │    ├─ 读取所有持仓账户
  │    ├─ 计算 TotalCollateral
  │    ├─ 读取价格 Prices.GetMarketPrice()
  │    ├─ 计算 MaintenanceMarginReq
  │    └─ 返回 TotalCollateral < MaintenanceMarginReq 的账户
  │
  │ 生成清算订单
  ├─ 按 NetCollateral 排序 (确定性)
  ├─ 创建 Liquidation Orders
  │
  │ 匹配清算订单
  └──> MemClob.PlaceLiquidationOrder()
       └─ 匹配引擎执行清算成交
```

### 4.3 资金费率数据流

```
        资金费率计算与结算流程

EndBlocker (Perpetuals)
  │ ProcessFundingTick()
  ▼
Perpetuals.Keeper
  │ 获取溢价样本
  ├─ 读取 PremiumSamples (过去 1 小时 60 个样本)
  │
  │ 计算资金费率
  ├─ 计算平均溢价 AvgPremium = Mean(PremiumSamples)
  ├─ 计算资金费率 FundingRate = Clamp(8 * AvgPremium, ...)
  │
  │ 更新 FundingIndex
  ├─ FundingIndex += FundingRate
  │
  │ 结算资金费用
  └──> Subaccounts.SettleFunding(perpetualId, fundingRate)
       ├─ 遍历所有持仓账户
       ├─ 计算资金费用 = Position × FundingRate
       ├─ 多头支付,空头收取 (或相反)
       └─ 更新 Subaccount 余额
```

### 4.4 自动做市数据流

```
        Vault 自动做市流程

EndBlocker (Vault)
  │ RefreshAllVaultOrders()
  ▼
Vault.Keeper
  │ 获取 Vault 净值
  ├──> Subaccounts.GetNetCollateral(VaultSubaccountId)
  │    └─ 返回 VaultEquity
  │
  │ 检查激活条件
  ├─ 如果 VaultEquity < ActivationThreshold,跳过
  │
  │ 取消旧订单
  ├──> CLOB.CancelShortTermOrder(oldOrderIds)
  │
  │ 获取价格
  ├──> Prices.GetMarketPrice(marketId)
  │    └─ 返回 OraclePrice
  │
  │ 计算订单报价
  ├─ BidPrice = OraclePrice × (1 - Spread/2)
  ├─ AskPrice = OraclePrice × (1 + Spread/2)
  ├─ OrderSize = VaultEquity × PositionSizePct / OraclePrice
  │
  │ 放置新订单
  └──> CLOB.PlaceShortTermOrder(buyOrder, sellOrder)
       └─ 订单进入订单簿,提供流动性
```

---

## 5. 模块初始化顺序

### 5.1 Genesis 初始化顺序

**为什么初始化顺序重要?**
- 某些模块依赖其他模块的状态
- 错误的顺序可能导致初始化失败
- 确保依赖关系正确解析

**推荐初始化顺序**:

```
1. Epochs         (基础时间模块,被 Perpetuals, Stats 依赖)
2. Prices         (价格模块,被 CLOB, Perpetuals, Vault 依赖)
3. Perpetuals     (合约配置,被 CLOB, Subaccounts 依赖)
4. Subaccounts    (账户模块,被 CLOB 依赖)
5. CLOB           (核心交易引擎,被 Vault, Stats 依赖)
6. Stats          (统计模块,被 CLOB 调用)
7. Vault          (自动做市,依赖所有核心模块)
8. Listing        (市场创建,依赖所有模块)
```

**初始化依赖图**:

```
Epochs ──> Perpetuals ──> CLOB ──> Stats
              │           ↑
Prices ───────┴──> Subaccounts ──> Vault ──> Listing
```

---

## 6. 循环依赖分析

### 6.1 潜在循环依赖

**CLOB ↔ Subaccounts**:
- CLOB 调用 Subaccounts 验证保证金
- Subaccounts 调用 CLOB 读取杠杆数据 (Leverage)

**解决方案**:
- Leverage 数据存储在 CLOB,但逻辑分散
- Subaccounts 通过 CLOB Keeper 读取 Leverage,不产生循环调用
- 验证保证金在 Subaccounts,不依赖 CLOB 其他功能

**CLOB ↔ Stats**:
- CLOB 调用 Stats 记录成交
- Stats 不调用 CLOB (单向依赖,无循环)

**结论**: 当前设计没有真正的循环依赖,所有依赖都是单向或通过接口隔离的。

---

## 7. 模块协作模式

### 7.1 调用模式分类

**同步调用** (Synchronous):
- CLOB → Subaccounts.CanPlaceOrder()
- CLOB → Perpetuals.GetPerpetual()
- Vault → Prices.GetMarketPrice()
- 特点: 立即返回结果,阻塞等待

**事件发布订阅** (Event-Driven):
- CLOB 发布 FillEvent → Stats 订阅记录成交
- CLOB 发布 OrderPlacedEvent → Indexer 订阅索引
- 特点: 解耦,异步处理

**EndBlocker 协作** (Batch Processing):
- Perpetuals.ProcessFundingTick() → Subaccounts.SettleFunding()
- Vault.RefreshAllVaultOrders() → CLOB.PlaceShortTermOrder()
- 特点: 批量处理,每个区块统一执行

### 7.2 数据共享模式

**共享状态存储**:
- Leverage 数据存储在 CLOB,Subaccounts 读取
- 优势: 减少重复存储,提高读取性能
- 劣势: 逻辑耦合,需要接口约定

**独立状态存储**:
- 每个模块有独立的 StateStore
- 通过 Keeper 接口访问其他模块数据
- 优势: 模块独立,便于维护
- 劣势: 需要跨模块调用

---

## 8. 性能与扩展性分析

### 8.1 关键路径分析

**热路径** (Hot Path):
- 订单放置与匹配 (CLOB → Subaccounts → Perpetuals)
- 价格查询 (Prices → CLOB/Vault/Subaccounts)
- 优化方向: 缓存价格,减少跨模块调用

**冷路径** (Cold Path):
- 市场创建 (Listing → Perpetuals/Prices/CLOB)
- 杠杆调整 (CLOB → Subaccounts)
- 优化方向: 异步处理,后台任务

### 8.2 扩展性考虑

**水平扩展**:
- 增加更多市场 (Listing 模块)
- 增加更多 Vaults (Vault 模块)
- 瓶颈: CLOB 单点匹配引擎

**垂直扩展**:
- 提高单个区块处理能力
- 优化匹配算法 (MemClob)
- 瓶颈: 区块 Gas 限制

---

## 9. 故障隔离与容错

### 9.1 模块故障影响分析

| 模块故障        | 影响范围                      | 缓解措施                          |
|---------------|-------------------------------|-----------------------------------|
| Prices 故障   | 无法获取价格,清算和做市暂停    | 使用上一个有效价格,报警           |
| CLOB 故障     | 交易完全中断                   | 核心模块,优先级最高,快速修复       |
| Perpetuals 故障 | 资金费率停止,新市场无法创建   | 使用默认参数,延迟资金费率结算     |
| Vault 故障    | 自动做市停止,但用户交易正常    | 降级到仅用户流动性,暂停 Vault     |
| Stats 故障    | 统计丢失,但交易正常            | 后台补录统计数据                  |

### 9.2 容错设计

**价格容错**:
- Prices 模块缓存上一个有效价格
- 如果预言机故障,使用缓存价格 + 报警

**保证金容错**:
- Subaccounts 保证金计算失败时,拒绝订单 (安全优先)
- 不允许保证金不足的订单通过

**清算容错**:
- 清算失败时,记录日志,下一个区块重试
- 防止清算失败导致系统风险

---

## 10. 总结

### 10.1 依赖关系特点

1. **分层清晰**: 基础层 (Prices, Perpetuals) → 核心层 (CLOB, Subaccounts) → 应用层 (Vault, Listing)
2. **单向依赖**: 大部分依赖是单向的,避免循环依赖
3. **接口隔离**: 模块间通过 Keeper 接口通信,不直接访问状态
4. **事件解耦**: 使用事件机制解耦模块,提高可维护性

### 10.2 关键协作模式

1. **订单放置**: CLOB → Subaccounts (验证) → Perpetuals (配置) → MemClob (匹配)
2. **清算**: CLOB → Subaccounts (查找) → Prices (价格) → MemClob (执行)
3. **资金费率**: Perpetuals → Prices (溢价) → Subaccounts (结算)
4. **自动做市**: Vault → Prices (报价) → CLOB (放置) → MemClob (流动性)

### 10.3 优化建议

1. **缓存优化**: 缓存频繁读取的数据 (价格、合约配置)
2. **批量处理**: EndBlocker 中批量处理,减少单次调用开销
3. **异步解耦**: 使用事件机制异步处理非关键路径
4. **故障隔离**: 模块故障不应影响其他模块,降级处理

---

**文档版本**: v1.0
**最后更新**: 2025-12-31
**文档作者**: Claude Sonnet 4.5
**文档状态**: ✅ 完成
