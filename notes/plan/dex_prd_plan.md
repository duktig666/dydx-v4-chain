# Hermes DEX 交易系统模块文档生成计划 (Updated)

## 需求概述

为交易系统负责人所负责的 7 个核心模块生成完整的技术文档:

1. **架构与系统设计文档** (使用 architect agent 生成)
2. **产品需求文档 PRD** (使用 analyst agent 生成)
3. **数据结构文档** (按模块组织,分存储类型整理)

**涉及模块**:
- CLOB (x/clob) - 中央限价订单簿
- Perpetuals (x/perpetuals) - 永续合约
- Listing (x/listing) - 市场上市
- Prices (x/prices) - 价格预言机
- Stats (x/stats) - 统计
- Vault (x/vault) - 金库/流动性池
- Leverage (x/leverage) - 杠杆管理

---

## 文档存放位置规划

基于现有 `notes/` 目录结构:

```
notes/
├── architecture/              # 新建: 架构与系统设计文档
│   ├── README.md             # 架构文档总览
│   ├── clob.md               # CLOB 模块架构设计
│   ├── perpetuals.md         # Perpetuals 模块架构设计
│   ├── listing.md            # Listing 模块架构设计
│   ├── prices.md             # Prices 模块架构设计
│   ├── stats.md              # Stats 模块架构设计
│   ├── vault.md              # Vault 模块架构设计
│   ├── leverage.md           # Leverage 模块架构设计 (轻量)
│   └── module_dependencies.md # 模块依赖关系图
│
├── business/                  # 现有: 业务分析文档
│   ├── clob_analyst.md       # ✅ 已存在
│   ├── perpetuals_analyst.md # ✅ 已存在
│   ├── slinky_oracle.md      # ✅ 已存在
│   └── prd/                  # 新建: PRD 子目录
│       ├── README.md         # PRD 文档总览
│       ├── clob_prd.md       # CLOB 模块 PRD
│       ├── perpetuals_prd.md # Perpetuals 模块 PRD
│       ├── listing_prd.md    # Listing 模块 PRD
│       ├── prices_prd.md     # Prices 模块 PRD
│       ├── stats_prd.md      # Stats 模块 PRD
│       ├── vault_prd.md      # Vault 模块 PRD
│       └── leverage_prd.md   # Leverage 模块 PRD
│
└── data_structure/            # 新建: 数据结构文档 (按模块组织)
    ├── README.md             # 数据结构总览
    ├── clob.md               # CLOB 模块数据结构
    ├── perpetuals.md         # Perpetuals 模块数据结构
    ├── listing.md            # Listing 模块数据结构
    ├── prices.md             # Prices 模块数据结构
    ├── stats.md              # Stats 模块数据结构
    ├── vault.md              # Vault 模块数据结构
    └── leverage.md           # Leverage 模块数据结构
```

**重要变更**: 数据结构文档按模块分文件,每个文件内部区分存储类型(StateStore, MemStore, TransientStore, MemClob)。

---

## 模块功能清单汇总

基于深度代码探索,以下是各模块的完整功能清单:

### 1. CLOB 模块 (x/clob) - 中央限价订单簿

#### 消息处理器 (12 个 Msg 类型)

**订单操作消息**:
- `MsgPlaceOrder` - 放置订单 (Short-Term/Long-Term)
- `MsgCancelOrder` - 取消单个订单
- `MsgBatchCancelOrders` - 批量取消订单
- `MsgProposedOperations` - 提议者提交操作集合 (包含订单匹配)

**配置管理消息**:
- `MsgCreateClobPair` - 创建新交易对
- `MsgUpdateClobPair` - 更新交易对配置
- `MsgUpdateLiquidationsConfig` - 更新清算配置
- `MsgUpdateEquityTierLimitConfig` - 更新权益层级限制配置
- `MsgUpdateBlockRateLimitConfig` - 更新区块速率限制配置

**杠杆管理消息**:
- `MsgUpdateLeverage` - 更新子账户杠杆倍数

**模块参数消息**:
- `MsgUpdateParams` - 更新模块参数

#### 核心业务流程

**订单生命周期**:
1. **Short-Term 订单流程** (单区块有效)
   - CheckTx: 验证订单格式和保证金
   - 存储在 TransientStore (区块结束自动过期)
   - EndBlock 时通过 ProposedOperations 匹配

2. **Long-Term 订单流程** (GTT 多区块有效)
   - 存储在 StateStore (持久化)
   - 有明确的 GoodTilBlockTime 过期时间
   - 系统在每个区块检查并匹配
   - 到期时自动清理 (EndBlocker)

3. **Conditional 订单流程** (止损/止盈)
   - 未触发: 存储在 UntriggeredConditionalOrderKeyPrefix
   - 触发条件满足: 转移到 TriggeredConditionalOrderKeyPrefix
   - 系统持续监控价格,满足条件时自动激活
   - 激活后按普通订单处理

4. **TWAP 订单流程** (时间加权平均价格)
   - 存储完整的 TWAP 订单参数
   - EndBlock 时生成子订单 (根据时间窗口切分)
   - 每个子订单独立匹配
   - 全部完成后清理父订单

**订单匹配引擎**:
- **价格-时间优先**: 最佳价格优先,相同价格下时间早的优先
- **确定性匹配**: 所有节点产生相同的匹配结果
- **部分成交支持**: 订单可分多次成交
- **成交记录**: 生成 Fill 事件供链下索引

**清算机制**:
- **触发条件**: 子账户保证金率低于维持保证金要求
- **清算流程**:
  1. Daemon 监控链上保证金率
  2. 发现低于阈值的子账户
  3. 提交 ProposedOperations 包含清算订单
  4. 清算订单与市场订单匹配
  5. 强制平仓,保护系统风险

- **Deleveraging (去杠杆)**:
  - **两种触发场景**:
    1. **负净抵押品 (Negative TNC)**: 账户资不抵债,清算无法覆盖损失
    2. **最终结算 (Final Settlement)**: 市场关闭,所有持仓必须平仓
  - **去杠杆流程**:
    1. 检查触发条件 (`CanDeleverageSubaccount`)
       - 负TNC: 使用破产价格 (Bankruptcy Price) 去杠杆
       - 最终结算: 使用预言机价格 (Oracle Price) 去杠杆
    2. 获取被去杠杆账户的完整持仓量
    3. 查找反向持仓账户,按盈利率排序
    4. 生成去杠杆成交记录 (Deleveraging Fills)
    5. 更新双方账户余额
    6. 记录到操作队列 (Operations Queue)
  - **价格计算**:
    - 破产价格: 使账户净抵押品恰好为 0 的价格
    - 预言机价格: 市场最终结算时使用的参考价格
  - **对手方选择策略**:
    - 选择持有反向仓位的账户
    - 按盈利率(从高到低)排序
    - 依次分配去杠杆数量直到全部平仓
  - **提现门控 (Withdrawal Gating)**:
    - 发现负TNC账户后,插入零成交去杠杆操作到队列
    - 提现逻辑检测到未解决的去杠杆操作时阻止提现
    - 确保负债账户先解决,防止资金逃逸

#### 订单类型完整清单 (8 种)

1. **Short-Term Order** (短期订单)
   - 有效期: 单区块
   - 存储: TransientStore
   - 用途: 高频交易,减少链上存储

2. **Long-Term Order** (长期订单)
   - 有效期: GoodTilBlockTime (可跨多区块)
   - 存储: StateStore
   - 用途: 限价单,挂单等待成交

3. **Conditional Order** (条件订单)
   - 类型: STOP_LOSS, TAKE_PROFIT
   - 触发: 价格满足条件时自动激活
   - 存储: Triggered/Untriggered 分开存储

4. **TWAP Order** (时间加权平均价格订单)
   - 特性: 分时段执行,降低市场冲击
   - 参数: TimeWindow, RandomizeStart
   - 实现: EndBlock 自动生成子订单

5. **Reduce-Only Order** (仅减仓订单)
   - 特性: 只能减少现有头寸,不能开新仓
   - 用途: 风险控制,避免误操作反向

6. **Post-Only Order** (仅Maker订单)
   - 特性: 只能被动成交(Maker),不能立即成交(Taker)
   - 用途: 做市商获取 Maker 手续费优惠

7. **IOC (Immediate-Or-Cancel) Order**
   - 特性: 立即成交或取消,不挂单
   - 用途: 快速执行,避免挂单风险

8. **FOK (Fill-Or-Kill) Order**
   - 特性: 必须完全成交,否则取消
   - 用途: 确保大单全部执行

#### ABCI 生命周期集成

**PreBlocker**:
- 初始化 MemClob 订单簿
- 从 StateStore 加载长期订单
- 重建订单簿内存结构

**BeginBlocker**:
- 重置匹配事件
- 清空已交付订单列表

**EndBlocker**:
1. 裁剪成交量记录 (OrderAmountFilled)
2. 移除过期订单
3. 生成 TWAP 子订单
4. 触发条件订单
5. 处理清算和去杠杆

**PrepareCheckState**:
1. 重放操作 (Replay Operations)
2. 处理清算 (Liquidations)
3. 处理去杠杆 (Deleveraging)
4. 裁剪成交状态
5. 移除过期订单
6. 触发条件订单
7. 生成 TWAP 子订单
8. 移除已完成TWAP订单
9. 发送链外索引事件
10. 广播清算/去杠杆订单ID
11. 更新 MemClob 内部状态

#### 查询接口 (10 种查询)

1. `ClobPair` - 查询单个交易对
2. `ClobPairAll` - 查询所有交易对
3. `EquityTierLimitConfiguration` - 查询权益层级限制配置
4. `BlockRateLimitConfiguration` - 查询区块速率限制配置
5. `LiquidationsConfiguration` - 查询清算配置
6. `StatefulOrder` - 查询长期订单
7. `StreamOrderbookUpdates` - 订阅订单簿更新流 (gRPC streaming)
8. `NextClobPairId` - 查询下一个交易对ID
9. `Params` - 查询模块参数
10. `Leverage` - 查询子账户杠杆配置

#### MEV 保护机制

- **速率限制**:
  - 短期订单: 限制每个区块的订单数量
  - 取消订单: 限制每N区块的取消数量
  - 杠杆更新: 限制每N区块的更新数量

- **权益层级限制**:
  - 根据子账户权益分配订单额度
  - 防止低权益账户垃圾攻击

- **确定性匹配**:
  - 所有节点产生相同的匹配结果
  - 无法通过重排序获得MEV

#### 依赖模块

**上游依赖**:
- Subaccounts Keeper - 抵押品检查、保证金计算
- Perpetuals Keeper - 合约规范、资金费率
- Prices Keeper - 价格预言机、清算价格判断
- Stats Keeper - 交易统计、手续费分级
- Leverage Keeper - 杠杆倍数查询

**下游被依赖**:
- Stats 模块 - 接收成交记录
- Vault 模块 - 为 Vault 放置订单

#### 测试覆盖

- **测试文件**: 37+ 个测试文件
- **测试函数**: 150+ 个测试函数
- **核心测试**:
  - 订单匹配: 价格-时间优先、部分成交、完全成交
  - 订单取消: 单个取消、批量取消
  - 清算: 触发条件、清算执行、保险基金
  - TWAP: 子订单生成、时间窗口
  - 条件订单: 触发条件、自动激活
  - 速率限制: 各类型速率限制验证

---

### 2. Perpetuals 模块 (x/perpetuals) - 永续合约

#### 消息处理器 (5 个 Msg 类型)

**合约管理**:
- `MsgCreatePerpetual` - 创建新的永续合约
- `MsgUpdatePerpetualParams` - 更新永续合约参数

**流动性层级管理**:
- `MsgSetLiquidityTier` - 创建或更新流动性层级

**资金费率机制**:
- `MsgAddPremiumVotes` - 提议者提交溢价投票

**参数管理**:
- `MsgUpdateParams` - 更新模块参数

#### 核心业务流程

**资金费率计算流程 (两层时间框架)**:

1. **Funding Sample Epoch** (1分钟周期)
   - 触发: EndBlock 检查 NumBlocksSinceEpochStart == 0
   - 流程:
     1. 收集该 epoch 内所有 MsgAddPremiumVotes
     2. 对每个永续合约计算投票的**中位数**
     3. 将新 Sample 附加到 PremiumSamplesKey
     4. 清空 PremiumVotesKey
     5. 发出 FundingValues 事件

2. **Funding Tick Epoch** (1小时周期)
   - 触发: EndBlock 检查 NumBlocksSinceEpochStart == 0
   - 流程:
     1. 收集该 epoch 内所有 PremiumSamples (60个)
     2. 移除样本尾部 (top/bottom RemovedTailSampleRatioPpm%, 当前=0%)
     3. 计算剩余样本的**平均值** (AvgInt32)
     4. 添加 DefaultFundingPpm: `FundingRate = Premium + DefaultFunding`
     5. 应用 Clamp: `|R| <= FundingRateClampFactor * (IM - MM)`
     6. 验证 FundingRate 不超过 int32 范围
     7. 如果 FundingRate != 0:
        - 获取市场价格
        - 计算 FundingIndexDelta: `FundingIndexDelta = FundingRatePpm * (timeSinceLastFunding / 8小时) * quoteQuantumsPerBaseQuantum`
        - 更新 FundingIndex: `FundingIndex += FundingIndexDelta`
     8. 清空 PremiumSamplesKey
     9. 发出 FundingRatesAndIndices 事件

**未平仓合约量 (Open Interest, OI) 更新**:
- 触发: CLOB 模块订单执行时调用 `ModifyOpenInterest()`
- 流程:
  1. 验证 Delta != 0
  2. 计算 NewOI = OldOI + Delta
  3. 验证 NewOI >= 0
  4. 存储回 KVStore
- 用途: 计算 OIMF (开放利息保证金系数)

**OIMF 动态保证金调整机制**:
- 定义: 根据未平仓合约量动态调整保证金要求
- 公式:
  ```
  if OI <= open_interest_lower_cap:
      OIMF = 0% (不调整)
  elif open_interest_lower_cap < OI < open_interest_upper_cap:
      OIMF = (OI - lower_cap) / (upper_cap - lower_cap) * 100%
  else:
      OIMF = 100% (最大调整)

  初始保证金 = BaseIMF + OIMF * (BaseIMF - BaseMaintenance)
  ```
- 集成: `GetMarginRequirementsInQuoteQuantums()` 中调用

**流动性层级管理**:
- 创建/更新: 通过 MsgSetLiquidityTier
- 字段:
  - initial_margin_ppm - 初始保证金率
  - maintenance_fraction_ppm - 维持保证金率
  - impact_notional - 冲击名义金额
  - open_interest_lower_cap - OI 下限
  - open_interest_upper_cap - OI 上限
- 验证:
  - initial_margin_ppm > 0
  - maintenance_fraction_ppm <= initial_margin_ppm
  - open_interest_upper_cap >= open_interest_lower_cap

#### 数据结构

**StateStore 存储**:

1. **Perpetual** (`PerpetualKeyPrefix | PerpetualId`)
   ```
   Perpetual {
     Params: PerpetualParams {
       id, ticker, market_id, atomic_resolution,
       default_funding_ppm, liquidity_tier, market_type
     }
     FundingIndex: SerializableInt
     OpenInterest: SerializableInt
   }
   ```

2. **LiquidityTier** (`LiquidityTierKeyPrefix | LiquidityTierId`)
   ```
   LiquidityTier {
     id, name,
     initial_margin_ppm, maintenance_fraction_ppm,
     impact_notional,
     open_interest_lower_cap, open_interest_upper_cap
   }
   ```

3. **PremiumVotes** (`PremiumVotesKey`)
   ```
   PremiumStore {
     all_market_premiums: []MarketPremiums
     num_premiums: uint32
   }
   ```

4. **PremiumSamples** (`PremiumSamplesKey`)
   - 同 PremiumVotes 结构

5. **Params** (`ParamsKey`)
   ```
   Params {
     funding_rate_clamp_factor_ppm,
     premium_vote_clamp_factor_ppm,
     min_num_votes_per_sample
   }
   ```

**TransientStore**: 初始化但未使用

#### 查询接口 (7 种查询)

1. `Perpetual` - 查询单个永续合约
2. `AllPerpetuals` - 查询所有永续合约 (分页)
3. `AllLiquidityTiers` - 查询所有流动性层级
4. `PremiumVotes` - 查询当前 funding-sample epoch 的投票
5. `PremiumSamples` - 查询当前 funding-tick epoch 的样本
6. `Params` - 查询模块参数
7. `NextPerpetualId` - 查询下一个永续合约ID

#### 依赖模块

**上游依赖**:
- Prices Keeper - 市场价格查询
- Epochs Keeper - Epoch 信息查询
- CLOB Keeper - 订单簿交互

**下游被依赖**:
- Subaccounts 模块 - 保证金计算
- CLOB 模块 - 订单执行

#### 测试覆盖

- **测试文件**: 13 个测试文件
- **测试行数**: ~96,000+ 行
- **核心测试**:
  - Perpetual 管理: 创建、修改、查询
  - 资金费率: 投票→样本→费率→Index更新
  - 流动性层级: 创建、更新、OIMF 线性插值
  - 保证金: GetMarginRequirementsInQuoteQuantums

---

### 3. Listing 模块 (x/listing) - 市场上市

#### 消息处理器 (4 个 Msg 类型)

**市场上市**:
- `MsgCreateMarketPermissionless` - 无权限上市新市场

**硬上限管理**:
- `MsgSetMarketsHardCap` - 设置市场最大数量限制

**Vault 配置**:
- `MsgSetListingVaultDepositParams` - 配置 Vault 存款参数

**永续合约升级**:
- `MsgUpgradeIsolatedPerpetualToCross` - 升级永续合约从隔离到交叉模式

#### 核心业务流程

**无权限上市流程 (CreateMarketPermissionless)**:
1. 检查当前市场数量是否超过硬上限
2. **创建价格市场**:
   - 从 PricesKeeper 获取下一个 MarketID
   - 从 MarketMap 读取市场元数据 (Reference Price, Decimals, CrossLaunch)
   - 创建 MarketParam (MinPriceChangePpm = 800 ppm 长尾市场)
3. **创建永续合约**:
   - 从 PerpetualsKeeper 获取下一个 PerpetualID
   - 根据 Reference Price 计算 Atomic Resolution
   - 根据 CrossLaunch 标志设置 Market Type (ISOLATED/CROSS)
   - 设置默认参数 (LiquidityTier=7 (IML 5x), DefaultFundingPpm=100)
4. **创建 CLOB Pair**:
   - 从 ClobKeeper 获取下一个 ClobPairID
   - 创建活跃的 CLOB 对 (Status=ACTIVE)
5. **Vault 存款流程**:
   - 计算总存款 = NewVaultDepositAmount + MainVaultDepositAmount
   - 向 Megavault 存款并获得 Shares
   - 分配新 Vault 的存款额
   - 锁定 Shares (锁定时间 = 当前块高 + NumBlocksToLockShares)
   - 激活新市场 Vault (QUOTING 状态)

**默认参数**:
- NewVaultDepositAmount: 10,000,000,000 (10,000 USDC)
- MainVaultDepositAmount: 0
- NumBlocksToLockShares: 2,592,000 (30 天)
- Step Base Quantums: 1,000,000
- Subticks Per Tick: 1,000,000
- Quantum Conversion Exponent: -9

**硬上限管理**:
- 默认值: 500
- 存储: `HardCapForMarketsKey` (uint32)
- 验证: CreateMarketPermissionless 时检查

**隔离到交叉升级流程**:
1. 验证权限 (Authority)
2. 获取目标永续合约
3. 验证当前为隔离模式
4. 转移隔离保险基金到交叉保险基金
5. 转移隔离抵押品到交叉抵押品池
6. 更新永续合约 MarketType 为 CROSS
7. 发出 UpdatePerpetualEvent

#### 数据结构

**StateStore 存储**:

1. **HardCapForMarkets** (`HardCapForMarketsKey`)
   - 类型: uint32
   - 说明: 上市市场的硬上限

2. **ListingVaultDepositParams** (`ListingVaultDepositParamsKey`)
   ```
   ListingVaultDepositParams {
     NewVaultDepositAmount: SerializableInt
     MainVaultDepositAmount: SerializableInt
     NumBlocksToLockShares: uint32
   }
   ```

#### 查询接口 (2 种查询)

1. `MarketsHardCap` - 查询市场硬上限
2. `ListingVaultDepositParams` - 查询 Vault 存款参数

#### 依赖模块

**上游依赖**:
- Prices Keeper - 创建价格市场
- CLOB Keeper - 创建订单簿对
- Perpetuals Keeper - 创建和管理永续合约
- MarketMap Keeper - 获取市场元数据
- Subaccounts Keeper - 转移隔离基金
- Vault Keeper - 管理 Vault 存款

#### 测试覆盖

- **测试文件**: 9 个测试文件
- **测试函数**: 33 个测试用例
- **核心测试**:
  - CreateMarketPermissionless: 成功、硬上限、市场验证
  - SetMarketsHardCap: 权限、参数验证
  - SetListingVaultDepositParams: 权限、金额验证
  - UpgradeIsolatedPerpetualToCross: 权限、状态转换

---

### 4. Prices 模块 (x/prices) - 价格预言机

#### 消息处理器 (3 个 Msg 类型)

**市场管理**:
- `MsgCreateOracleMarket` - 创建新的预言机市场
- `MsgUpdateMarketParam` - 修改市场参数

**价格更新**:
- `MsgUpdateMarketPrices` - 更新市场价格 (从Slinky Oracle集成)

#### 核心业务流程

**市场创建流程**:
1. 验证权限 (Authority检查)
2. 从 MarketMap 获取货币对指数 (Exponent)
3. 创建初始零价格的 MarketPrice
4. 调用 `CreateMarket()`:
   - 检查市场ID唯一性
   - 验证 MarketParam 格式
   - 验证 MarketPrice 格式
   - 检查 Pair 唯一性
   - 转换 Pair 到 CurrencyPair
   - 从 MarketMap 验证货币对存在
   - 验证 Exponent = -Decimals
   - KVStore 持久化 MarketParam 和 MarketPrice
   - 添加 CurrencyPairID 映射
   - 生成 Indexer 事件
   - 在 MarketMap 启用该市场

**市场参数修改流程**:
1. 验证输入的 MarketParam
2. 从存储获取现有的 MarketParam
3. 检查新 Pair 是否与其他市场重复
4. 如果 Pair 已改变:
   - 删除旧的 CurrencyPairID 映射
   - 在 MarketMap 禁用旧市场
   - 添加新的 CurrencyPairID 映射
   - 在 MarketMap 启用新市场
5. KVStore 持久化修改后的 MarketParam
6. 生成 Indexer 事件

**价格更新流程**:
1. 对每个价格更新:
   - 从存储获取当前 MarketPrice
   - 计算价格变化率 (DiffRate)
   - 记录遥测数据
   - 更新 MarketPrice.Price
   - 记录价格更新块高度
2. 原子性地写入所有更新
3. 生成 Indexer 事件
4. 如果启用 GRPC 流,发送 PriceUpdate

**Slinky Oracle 集成流程**:

**有效价格更新定义**:
- 索引价格存在且非零
- 平滑价格存在且非零
- 平滑价格和索引价格在 Oracle 价格同侧
- 提议价格更接近 Oracle 价格
- 提议价格满足最小价格变化要求

**GetValidMarketPriceUpdates** 流程:
1. 获取所有市场参数和价格
2. 从 indexPriceCache 获取所有索引价格 (中位数)
   - 过滤过期价格
   - 过滤不足最小交易所数
3. 遍历每个市场:
   - 检查索引价格是否存在和非零
   - 调用 `shouldProposePrice()` 判断
   - 如果应提议,添加到更新列表
4. 按 marketId 排序更新
5. 返回 MsgUpdateMarketPrices

**价格精度验证** (两阶段):

1. **非确定性验证 (ProcessProposal)**:
   - 索引价格可用性检查
   - 价格精度验证 (Towards 或 Crossing 条件)
   - 跨界缓冲约束 (sqrt 条件)

2. **确定性验证 (DeliverTx)**:
   - 市场存在性检查
   - 最小价格变化要求检查

**价格精度算法**:

**Towards Condition**:
```
newPrice 在 [min(oldPrice, indexPrice), max(oldPrice, indexPrice)] 范围内
```

**Crossing Condition** (当 newPrice 跨越 indexPrice 时):
```
情况A: oldDelta <= 1 tick
  newDelta <= oldDelta

情况B: oldDelta > 1 tick
  newDelta^2 * 1_000_000 <= oldDelta * tickSizePpm
  (即: newDelta <= sqrt(oldDelta * tickSize))

其中:
- tickSize = oldPrice * MinPriceChangePpm / 1_000_000
- oldDelta = |oldPrice - indexPrice|
- newDelta = |indexPrice - newPrice|
```

#### 数据结构

**StateStore 存储**:

1. **MarketParam** (`MarketParamKeyPrefix | MarketId`)
   ```
   MarketParam {
     id, pair, exponent (deprecated),
     min_exchanges, min_price_change_ppm,
     exchange_config_json
   }
   ```

2. **MarketPrice** (`MarketPriceKeyPrefix | MarketId`)
   ```
   MarketPrice {
     id, exponent, price, block_timestamp
   }
   ```

3. **CurrencyPairID** (`CurrencyPairIDPrefix | CurrencyPair`)
   - 类型: uint64 (marketId)

4. **NextMarketID** (`NextMarketIDKey`)
   - 类型: uint32

#### 查询接口 (5 种查询)

1. `AllMarketPrices` - 分页查询所有市场价格
2. `MarketPrice` - 查询单个市场价格
3. `AllMarketParams` - 分页查询所有市场参数
4. `MarketParam` - 查询单个市场参数
5. `NextMarketId` - 查询下一个市场ID

#### 依赖模块

**上游依赖**:
- MarketMap Keeper - 货币对验证、市场启用/禁用
- RevShare Keeper - 市场创建时创建 RevShare 条目
- Indexer - 生成市场创建/修改/价格更新事件
- Pricefeed - 提供交易所价格缓存

#### 测试覆盖

- **测试文件**: 13 个测试文件
- **测试行数**: ~122,000+ 行
- **核心测试**:
  - 市场管理: 创建、修改、参数验证
  - 价格更新: 空更新、多市场、大幅波动
  - 价格精度验证: Towards, Crossing, sqrt 条件
  - 价格计算工具: min price change, 价格转换

---

### 5. Stats 模块 (x/stats) - 统计

#### 消息处理器 (1 个 Msg 类型)

**参数管理**:
- `MsgUpdateParams` - 更新模块参数 (窗口时长)

#### 核心业务流程

**Epoch 统计数据收集流程** (EndBlock 阶段):

1. **收集 Block 内数据**:
   - 源: BlockStats (TransientStore)
   - 数据: 本区块所有成交记录 (Fills)
   - 每个 Fill: Taker, Maker, 名义本金, 附属费用, 附属属性

2. **聚合到 Epoch 统计**:
   - 获取当前 Epoch 信息
   - 检索该 Epoch 的 EpochStats
   - 为每个 Taker/Maker 创建或更新 UserStats

3. **更新用户统计**:
   ```
   TakerNotional += Fill.Notional
   MakerNotional += Fill.Notional
   Affiliate_30DRevenueGeneratedQuantums += Fill.AffiliateFeeGenerated
   ```

4. **处理附属属性** (Affiliate Attributions):
   - 跟踪 Referrer (推荐者)
   - 跟踪 Referee (被推荐者)
   - 更新:
     - Referrer: `Affiliate_30DReferredVolumeQuoteQuantums`
     - Referee: `Affiliate_30DAttributedVolumeQuoteQuantums`

5. **排序保证确定性**:
   - 按用户地址字母序排序 Epoch 统计

6. **全局统计更新**:
   - `GlobalStats.NotionalTraded += Fill.Notional` (仅统计 Taker 成交量)

**过期统计清理流程** (ExpireOldStats):

1. 获取当前 Epoch
2. 获取 TrailingEpoch (待删除的最旧 Epoch)
3. 检查是否超出窗口期:
   - 如果 `EpochEndTime + WindowDuration < 当前时间` → 删除
4. 从 GlobalStats 和 UserStats 递减已删除数据:
   - `GlobalStats.NotionalTraded -= removedStats.TakerNotional`
   - `UserStats.TakerNotional -= ...`
   - 处理 Affiliate 字段 (防止 uint64 下溢)
5. 更新 TrailingEpoch 指针

#### 数据结构

**BlockStats** (TransientStore):
```
BlockStats {
  Fills: []BlockStats_Fill {
    taker, maker, notional,
    affiliate_fee_generated_quantums,
    affiliate_attributions
  }
}
```

**UserStats** (StateStore):
```
UserStats {
  TakerNotional: uint64
  MakerNotional: uint64
  Affiliate_30DRevenueGeneratedQuantums: uint64
  Affiliate_30DReferredVolumeQuoteQuantums: uint64
  Affiliate_30DAttributedVolumeQuoteQuantums: uint64
}
```

**EpochStats** (StateStore):
```
EpochStats {
  EpochEndTime: time.Time
  Stats: []EpochStats_UserWithStats {
    User: string
    Stats: *UserStats
  }
}
```

**GlobalStats** (StateStore):
```
GlobalStats {
  NotionalTraded: uint64  // 30 日滚动窗口总成交量
}
```

**StatsMetadata** (StateStore):
```
StatsMetadata {
  TrailingEpoch: uint32  // 滑动窗口尾部 Epoch
}
```

**Params** (StateStore):
```
Params {
  WindowDuration: time.Duration  // 滚动窗口时长 (默认 30 天)
}
```

#### 查询接口 (5 种查询)

1. `Params` - 获取模块参数
2. `StatsMetadata` - 获取模块元数据
3. `GlobalStats` - 获取全局统计
4. `UserStats` - 查询用户 30 日统计
5. `EpochStats` - 查询 Epoch 维度统计

#### 依赖模块

**上游依赖**:
- Epochs Keeper - 获取当前 Epoch 信息
- Staking Keeper - 查询抵押代币余额

**入站依赖**:
- CLOB 模块 - 调用 `RecordFill()` 记录成交

#### 测试覆盖

- **测试文件**: 9 个测试文件
- **测试函数**: 20+ 个测试函数
- **核心测试**:
  - RecordFill: 无成交、单个、多个
  - ProcessBlockStats: 基础流程、附属属性
  - ExpireOldStats: 窗口清理
  - 查询接口: 所有查询方法

---

### 6. Vault 模块 (x/vault) - 金库/流动性池

#### 消息处理器 (8 个 Msg 类型)

**存款/提款消息**:
- `MsgDepositToMegavault` - 从用户 subaccount 存入资金
- `MsgWithdrawFromMegavault` - 从 Megavault 提取资金
- `MsgAllocateToVault` - 从主 Vault 分配资金到子 Vault
- `MsgRetrieveFromVault` - 从子 Vault 回收资金到主 Vault

**股份管理消息**:
- `MsgUnlockShares` - 解锁已达到 unlock block height 的股份

**参数配置消息**:
- `MsgUpdateDefaultQuotingParams` - 更新默认报价参数
- `MsgSetVaultParams` - 设置特定 Vault 的参数
- `MsgUpdateOperatorParams` - 更新 Megavault 操作员地址

#### 核心业务流程

**Megavault 存款流程**:
1. **MintShares** (铸造股份):
   - 验证 deposit 金额 > 0
   - 获取当前总股份 totalShares
   - 如果 totalShares == 0:
     - 股份数 = quoteQuantums (1:1初始比例)
   - 否则:
     - equity = GetMegavaultEquity()
     - 股份数 = quoteQuantums * totalShares / equity
   - 增加总股份和 owner 股份
2. **ProcessTransfer**: 从 subaccount 转账到 megavault
3. **EmitEvent**: deposit_to_megavault 事件

**Megavault 提取流程**:
1. 验证 sharesToWithdraw > 0
2. 检查 owner 拥有足够 unlocked 股份
3. **RedeemFromMainAndSubVaults**:
   - 从主 Vault 赎回: `redeemedQuantums = equity * shares / totalShares`
   - 从每个子 Vault 赎回:
     - 计算 withdrawal slippage (考虑 leverage)
     - 赎回金额 = equity * (shares/totalShares) * (1 - slippage)
     - 从子 Vault 转账到主 Vault
4. **ProcessTransfer**: 从 megavault 转账到 subaccount
5. **销毁股份**: 减少 owner 股份和总股份
6. **EmitEvent**: withdraw_from_megavault 事件

**提取滑点计算**:
```
slippage = min(simple_slippage, estimated_slippage)

simple_slippage = leverage * initial_margin

estimated_slippage = spread * (l + integral * (n-m)/m)
  其中:
  - l = leverage
  - n = total_shares
  - m = shares_to_withdraw
  - integral = SkewAntiderivative(skew_factor, posterior_leverage)
              - SkewAntiderivative(skew_factor, leverage)
  - posterior_leverage = leverage * n / (n-m)
```

**股份锁定与解锁流程**:

**LockShares**:
1. 验证参数有效性
2. 获取 owner 现有 unlock 记录
3. 添加新的 ShareUnlock 条目 (shares, unlockBlockHeight)
4. 验证总锁定股份 <= owner 总股份
5. DelayMessage: 在 tilBlock 高度自动调用 MsgUnlockShares
6. SetOwnerShareUnlocks()

**UnlockShares** (DelayMsg 自动触发):
1. 获取 owner 的 ShareUnlocks
2. 过滤出 unlockBlockHeight <= currentBlockHeight 的 unlock
3. 累加待解锁股份
4. 如无 remaining unlocks: 删除 owner 记录
5. 否则: 更新 remaining unlocks

**Vault 订单放置流程**:

**RefreshAllVaultOrders** (每个 EndBlock):
```
遍历所有 Vault:
  ├─ 检查 Vault 状态: QUOTING 或 CLOSE_ONLY
  ├─ 跳过无仓位且 USDC 余额 < activation_threshold 的 Vault
  └─ RefreshVaultClobOrders(vaultId)
```

**RefreshVaultClobOrders**:
1. 获取最近的 client IDs (mostRecentClientIds)
2. 计算要放置的新订单 (GetVaultClobOrders)
3. 对于每层 (layer):
   - 若 mostRecentClientIds[i] 不存在: PlaceVaultClobOrder(newOrder)
   - 若旧 order 已过期/全部成交: 翻转 client ID 最后一位,PlaceVaultClobOrder
   - 若订单参数变化: ReplaceVaultClobOrder(oldOrderId, newOrder)
   - 否则: 保持现有 order
4. 取消多余层的旧 order
5. SetMostRecentClientIds(newClientIds)

**GetVaultClobOrders** (订单计算):
```
对每层 i (0 to layers-1):
  ├─ leverage_i = leverage ± i * orderSizePctPpm (ask-, bid+)
  ├─ 计算 skew_i (基于 leverage_i 的非线性函数)
  ├─ spread_i = max(spreadMin, spreadBuffer + minPriceChange)
  ├─ ask_spread_i = (1 + skew_i) * spread
  ├─ bid_spread_i = (1 - skew_i) * spread
  ├─ ask_price_i = oraclePrice * (1 + ask_spread_i)
  ├─ bid_price_i = oraclePrice * (1 - bid_spread_i)
  └─ 创建 ask 和 bid 订单
```

**订单客户端 ID 编码**:
```
clientId = (side-1 << 31) | (layer << 23)
  bit 31: side-1 (0=BUY, 1=SELL)
  bits 23-30: layer (0-255)
  bits 0-22: 预留
```

#### 数据结构

**StateStore 存储**:

1. **TotalShares** (`TotalSharesKey`)
   - 类型: NumShares
   - 说明: Megavault 总股份

2. **OwnerShares** (`OwnerSharesKeyPrefix:{owner}`)
   - 类型: NumShares
   - 说明: 每个 owner 的股份数

3. **OwnerShareUnlocks** (`OwnerShareUnlocksKeyPrefix:{owner}`)
   ```
   OwnerShareUnlocks {
     OwnerAddress: string
     ShareUnlocks: []ShareUnlock {
       Shares: NumShares
       UnlockBlockHeight: uint32
     }
   }
   ```

4. **DefaultQuotingParams** (`DefaultQuotingParamsKey`)
   ```
   QuotingParams {
     layers, spread_min_ppm, spread_buffer_ppm,
     skew_factor_ppm, order_size_pct_ppm,
     order_expiration_seconds,
     activation_threshold_quote_quantums
   }
   ```

5. **VaultParams** (`VaultParamsKeyPrefix:{vaultId}`)
   ```
   VaultParams {
     Status: VaultStatus
     QuotingParams: *QuotingParams (nil 则使用默认)
   }
   ```

6. **VaultAddress** (`VaultAddressKeyPrefix:{address}`)
   - 仅用于存在性检查

7. **MostRecentClientIds** (`MostRecentClientIdsKeyPrefix:{vaultId}`)
   - 类型: []uint32
   - 说明: 每个 Vault 最近的订单 client IDs

8. **OperatorParams** (`OperatorParamsKey`)
   ```
   OperatorParams {
     Operator: string
   }
   ```

**VaultStatus 枚举**:
- VAULT_STATUS_STAND_BY - 待命 (无订单)
- VAULT_STATUS_QUOTING - 正在报价 (active)
- VAULT_STATUS_CLOSE_ONLY - 仅平仓
- VAULT_STATUS_DEACTIVATED - 已关闭

#### 查询接口 (7 种查询)

1. `Vault` - 查询 Vault 信息 (equity, inventory, params)
2. `AllVaults` - 查询所有 Vaults
3. `VaultParams` - 查询 Vault 参数
4. `MegavaultTotalShares` - 查询总股份
5. `MegavaultOwnerShares` - 查询 owner 股份和锁定信息
6. `MegavaultAllOwnerShares` - 查询所有 owner 股份
7. `MegavaultWithdrawalInfo` - 查询提取信息 (预期金额)
8. `Params` - 查询默认报价参数

#### 依赖模块

**上游依赖**:
- Assets Keeper - 获取资产配置
- Bank Keeper - 查询余额
- CLOB Keeper - 放置/取消/替换订单
- DelayMsg Keeper - 延迟执行 UnlockShares
- Perpetuals Keeper - 获取永续合约配置
- Prices Keeper - 获取市场价格
- Sending Keeper - 处理资金转账
- Subaccounts Keeper - 计算权益和保证金

#### 测试覆盖

- **测试文件**: 22 个测试文件
- **Keeper 代码**: ~2,876 行
- **核心测试**:
  - 存取款: MintShares, RedeemFromMainAndSubVaults
  - 订单管理: RefreshVaultClobOrders, GetVaultClobOrders
  - 股份锁定: LockShares, UnlockShares
  - Vault 生命周期: 创建、激活、关闭

---

### 7. Leverage 模块 (x/leverage) - 杠杆管理

**特殊说明**: Leverage 模块采取极简化设计,**不存在独立的 `x/leverage` 模块实现**。所有杠杆功能都集成在 **CLOB 模块** 中作为订单管理的一部分,而状态存储和验证由 **Subaccounts 模块** 负责。

#### 消息处理器 (1 个 Msg 类型,在 CLOB 模块中)

**杠杆管理**:
- `MsgUpdateLeverage` - 更新子账户杠杆倍数

**消息结构**:
```protobuf
message MsgUpdateLeverage {
  SubaccountId subaccount_id = 1;
  repeated LeverageEntry clob_pair_leverage = 2;
}

message LeverageEntry {
  uint32 clob_pair_id = 1;
  uint32 custom_imf_ppm = 2;  // 初始保证金系数 (PPM)
}
```

#### 核心业务流程

**杠杆更新流程**:

1. **CheckTx Phase** (Ante Handler):
   - ClobDecorator.AnteHandle() 拦截
   - 验证消息格式 (ValidateAndConstructPerpetualLeverageMap):
     - 检查子账户 ID 非空且有效
     - 检查杠杆条目列表非空
     - 检查每个 CustomImfPpm 在 (0, 1,000,000] 范围内
     - 检查交易对 ID 唯一性
     - 验证交易对是否存在
   - 构建映射: perpetualId → CustomImfPpm
   - 速率限制检查 (RateLimitUpdateLeverage)
   - 委托给 Subaccounts Keeper.UpdateLeverage():
     - 获取最小 IMF (最大杠杆限制)
     - 验证新杠杆不超过每个永续合约的最大值
     - 获取现有杠杆配置
     - 合并新旧配置
     - 验证新杠杆是否违反保证金要求
     - 存储到 KVStore

**杠杆倍数与 IMF 的关系**:
```
Leverage = 1 / IMF
IMF = CustomImfPpm / 1,000,000

示例:
- CustomImfPpm = 1,000,000 → IMF = 100% → 1x 杠杆 (无杠杆)
- CustomImfPpm = 500,000 → IMF = 50% → 2x 杠杆
- CustomImfPpm = 100,000 → IMF = 10% → 10x 杠杆
- CustomImfPpm = 50,000 → IMF = 5% → 20x 杠杆
```

**保证金要求验证**:
1. 对每个受影响的永续合约
2. 构建虚拟的头寸更新 (数量=0)
3. 使用新杠杆配置重新计算 IMR
4. 检查净抵押品是否 >= 新的 IMR
5. 如果不满足,拒绝杠杆更新

#### 数据结构

**StateStore 存储** (在 Subaccounts 模块):

**LeverageData** (`LeverageKeyPrefix`):
```protobuf
message LeverageData {
  repeated PerpetualLeverageEntry entries = 1;
}

message PerpetualLeverageEntry {
  uint32 perpetual_id = 1;
  uint32 custom_imf_ppm = 2;
}
```

**存储特性**:
- 持久化存储在区块链状态
- 按永续合约 ID 排序存储 (确定性)
- 支持增量更新 (合并新旧配置)

#### 查询接口 (1 种查询,在 CLOB 模块中)

**查询方法**:
- `Leverage` - 查询子账户的杠杆配置

**查询结构**:
```protobuf
message QueryLeverageRequest {
  string owner = 1;
  uint32 number = 2;
}

message QueryLeverageResponse {
  repeated ClobPairLeverageInfo clob_pair_leverage = 1;
}

message ClobPairLeverageInfo {
  uint32 clob_pair_id = 1;
  uint32 custom_imf_ppm = 2;
}
```

**查询端点**:
- `/hermesprotocol/clob/leverage/{owner}/{number}`

#### 模块依赖关系

```
Leverage 功能分布:

├─ CLOB 模块
│  ├─ msg_server_update_leverage.go - 消息处理入口
│  ├─ types/leverage.go - 消息验证逻辑
│  ├─ keeper/leverage.go - 委托给 Subaccounts Keeper
│  ├─ keeper/rate_limit.go - 速率限制
│  ├─ keeper/grpc_query_leverage.go - 查询处理
│  └─ ante/clob.go - Ante 处理器集成
│
├─ Subaccounts 模块
│  ├─ keeper/leverage.go - 核心实现
│  │  ├─ SetLeverage - 存储
│  │  ├─ GetLeverage - 读取
│  │  └─ UpdateLeverage - 业务逻辑
│  └─ types/leverage.proto - 数据结构
│
└─ Perpetuals 模块
   └─ keeper - GetMinImfForPerpetual 获取最大杠杆限制
```

#### 速率限制

**RateLimitUpdateLeverage** 实现:
- 仅在 CheckTx 阶段应用
- 使用子账户所有者地址作为限流键
- 同一地址的多个子账户共享限额
- 配置: BlockRateLimitConfiguration.max_leverage_updates_per_n_blocks

#### 测试覆盖

- **主要测试文件**: leverage_e2e_test.go (557 行)
- **测试函数**: 6 个主要测试函数
- **核心测试**:
  - 基础订单放置: 杠杆配置生效
  - 杠杆限制订单: 低杠杆拒绝订单
  - 杠杆与提现: 杠杆影响可用抵押品
  - 现有头寸更新: 增加杠杆可能失败
  - 速率限制: 按地址限流

---

## 执行计划

### 阶段 1: 数据结构文档生成 (优先级最高)

#### 1.1 生成各模块数据结构文档

**目标**: 按模块组织数据结构文档,每个文件内部区分存储类型。

**文档结构模板**:
```markdown
# [模块名] 数据结构文档

## 概述
[模块的数据结构设计概述]

## StateStore 数据结构 (链上持久化)

### 1. [数据结构名称]
**存储键**: `前缀 | key`
**数据类型**: `protobuf 类型`

**数据结构定义**:
\`\`\`
[字段定义]
\`\`\`

**业务含义**:
- [说明业务用途]

**使用场景**:
- [具体使用场景]

### 2. [另一个数据结构]
...

## MemStore 数据结构 (内存存储)

### 1. [数据结构名称]
...

## TransientStore 数据结构 (瞬态存储)

### 1. [数据结构名称]
...

## MemClob 数据结构 (纯内存订单簿,仅 CLOB 模块)

### 1. [数据结构名称]
...

## 数据访问模式

### 读取模式
\`\`\`
Keeper.Get[DataStructure](ctx, key) -> [Type]
\`\`\`

### 写入模式
\`\`\`
Keeper.Set[DataStructure](ctx, value)
\`\`\`

### 迭代模式
\`\`\`
Keeper.GetAll[DataStructures](ctx) -> [][Type]
\`\`\`

## 存储特点

### 持久化特性
- [持久化行为说明]

### 性能考虑
- [性能优化点]

### 数据完整性
- [一致性保证]
```

**待生成文档列表**:
1. `notes/data_structure/clob.md` - CLOB 模块数据结构
2. `notes/data_structure/perpetuals.md` - Perpetuals 模块数据结构
3. `notes/data_structure/listing.md` - Listing 模块数据结构
4. `notes/data_structure/prices.md` - Prices 模块数据结构
5. `notes/data_structure/stats.md` - Stats 模块数据结构
6. `notes/data_structure/vault.md` - Vault 模块数据结构
7. `notes/data_structure/leverage.md` - Leverage 模块数据结构 (轻量)
8. `notes/data_structure/README.md` - 数据结构总览

#### 1.2 数据结构文档重点内容

**CLOB 模块** (重点中的重点):
- StateStore: CLOBPair, LongTermOrderPlacement, ConditionalOrderPlacement, OrderAmountFilled, LiquidationsConfig, EquityTierLimitConfig, BlockRateLimitConfig, Leverage
- MemStore: 从 StateStore 同步的配置
- TransientStore: ShortTermOrderPlacement, ProcessProposerMatchesEvents, DeliveredOrders
- MemClob: 订单簿数据结构 (Orderbook, OrderIdSet, OrderMap)

**Perpetuals 模块**:
- StateStore: Perpetual, LiquidityTier, PremiumVotes, PremiumSamples, Params, NextPerpetualID

**Listing 模块**:
- StateStore: HardCapForMarkets, ListingVaultDepositParams

**Prices 模块**:
- StateStore: MarketParam, MarketPrice, CurrencyPairID, NextMarketID

**Stats 模块**:
- StateStore: UserStats, EpochStats, GlobalStats, StatsMetadata, Params
- TransientStore: BlockStats

**Vault 模块**:
- StateStore: TotalShares, OwnerShares, OwnerShareUnlocks, DefaultQuotingParams, VaultParams, VaultAddress, MostRecentClientIds, OperatorParams

**Leverage 模块**:
- StateStore (在 Subaccounts 模块): LeverageData, PerpetualLeverageEntry

---

### 阶段 2: 架构与系统设计文档生成

#### 2.1 生成各模块架构文档 (使用 architect agent)

**文档结构模板** (基于 architect.md 规范):

```markdown
# [模块名] 模块架构设计

## 1. 模块概述
- 职责定义
- 核心功能
- 关键约束

## 2. 架构设计
### 2.1 组件结构
[描述模块的主要组件和层次]

### 2.2 存储架构
- StateStore 设计
- MemStore 设计 (如适用)
- TransientStore 设计 (如适用)
- MemClob 纯内存结构 (仅 CLOB)

### 2.3 ABCI 生命周期集成
- PreBlocker 流程 (如适用)
- BeginBlocker 流程 (如适用)
- EndBlocker 流程 (如适用)
- Precommit 流程 (如适用)

## 3. 核心业务流程
### 3.1 [主要业务流程1]
[流程图 + 详细说明]

### 3.2 [主要业务流程2]
...

## 4. 模块依赖关系
### 4.1 上游依赖
[列出依赖的其他模块 Keeper]

### 4.2 下游被依赖
[列出哪些模块依赖本模块]

## 5. 数据流设计
### 5.1 [数据流1]
[数据流图 + 说明]

### 5.2 [数据流2]
...

## 6. 性能与可扩展性
### 6.1 性能优化点
[关键性能优化措施]

### 6.2 扩展性设计
[如何支持业务扩展]

### 6.3 瓶颈分析
[已知或潜在瓶颈]

## 7. 安全与风险控制
### 7.1 [安全机制1]
### 7.2 [安全机制2]
...

## 8. 技术决策记录 (ADR)
### 8.1 为什么[关键设计决策1]?
[权衡分析 + 理由]

### 8.2 为什么[关键设计决策2]?
...

## 9. 集成点与扩展
### 9.1 与其他模块的关键集成
[详细说明集成方式]

### 9.2 现有扩展点
[可配置参数和扩展机制]
```

**待生成文档列表**:
1. `notes/architecture/clob.md` - CLOB 模块架构设计
2. `notes/architecture/perpetuals.md` - Perpetuals 模块架构设计
3. `notes/architecture/listing.md` - Listing 模块架构设计
4. `notes/architecture/prices.md` - Prices 模块架构设计
5. `notes/architecture/stats.md` - Stats 模块架构设计
6. `notes/architecture/vault.md` - Vault 模块架构设计
7. `notes/architecture/leverage.md` - Leverage 模块架构设计 (轻量,说明集成设计)
8. `notes/architecture/module_dependencies.md` - 模块依赖关系图
9. `notes/architecture/README.md` - 架构文档总览

#### 2.2 架构文档重点关注

**关键点** (来自 architect.md):
- ✅ 重点描述**为什么**这样设计,而非**是什么**
- ✅ 技术决策要有权衡分析
- ✅ 数据流和组件交互要清晰
- ✅ 不包含代码细节,专注架构层面

**CLOB 模块架构重点**:
- 技术决策: 为什么使用 MemClob? 为什么区分 Long-Term/Short-Term 订单? 为什么在 TransientStore 存储某些状态?
- 性能优化: 订单匹配算法复杂度、MemClob 内存管理、确定性保证
- MEV 保护: 速率限制策略、权益层级限制、确定性匹配

**Perpetuals 模块架构重点**:
- 技术决策: 为什么使用两层周期机制? 为什么溢价用中位数,资金费率用平均值? 为什么需要 OIMF 机制?
- 关键算法: 中位数聚合、平均值聚合、OIMF 线性插值

**Leverage 模块架构重点**:
- 技术决策: 为什么不实现独立模块? 为什么存储在 Subaccounts? 为什么在 CLOB 处理消息?
- 模块化设计: 关注点分离、高内聚低耦合

---

### 阶段 3: 产品需求文档 (PRD) 生成

#### 3.1 生成各模块 PRD (使用 analyst agent)

**重要约束** (来自需求):
- ❌ **绝对不能包含代码**
- ✅ 以产品经理视角描述功能
- ✅ 基于测试用例推断需求
- ✅ 简洁明了,直切重点
- ❌ **不包含版本规划部分** (只文档当前实现)

**文档结构模板** (基于 analyst.md 规范):

```markdown
# [模块名] 产品需求文档 (PRD)

## 1. 产品概述
### 1.1 产品定位
[模块在交易系统中的定位]

### 1.2 核心价值
[为用户提供的核心价值]

### 1.3 目标用户
[主要用户群体]

## 2. 功能需求

### FR-1: [功能名称]
**需求描述**: [功能的核心目标]

**功能点**:
- [功能点1]
- [功能点2]
- ...

**业务规则**:
- [规则1]
- [规则2]
- ...

**计算公式** (如涉及):
- **公式名称**: [公式的业务含义]

  数学表达式:
  ```
  结果 = 变量A × 变量B / 变量C
  ```

  变量说明:
  - 变量A: [定义和单位]
  - 变量B: [定义和单位]
  - 变量C: [定义和单位]

  计算示例:
  - 假设 变量A = 100, 变量B = 50, 变量C = 10
  - 结果 = 100 × 50 / 10 = 500

  业务解释:
  [用通俗语言解释公式的含义和用途]

**验收标准**:
- [标准1]
- [标准2]
- ...

**参考测试用例**:
- `TestXxx` - [测试说明]
- `TestYyy` - [测试说明]

### FR-2: [另一个功能]
...

## 3. 非功能需求

### NFR-1: 性能要求
[具体性能指标]

### NFR-2: 可靠性要求
[可用性、确定性要求]

### NFR-3: 安全性要求
[安全机制要求]

## 4. 用户场景

### 场景 1: [场景名称]
**参与者**: [用户角色]
**前置条件**: [必要条件]
**流程**:
1. [步骤1]
2. [步骤2]
...
**后置条件**: [结果状态]

### 场景 2: [另一个场景]
...

## 5. 业务指标

### 关键指标 (KPI)
- [指标1]
- [指标2]
- ...

### 监控指标
- [指标1]
- [指标2]
- ...

## 6. 附录

### 术语表
- **术语1**: [解释]
- **术语2**: [解释]
- ...

### 参考资料
- 架构文档: `notes/architecture/[module].md`
- 技术分析: `notes/business/[module]_analyst.md` (如存在)
```

**待生成文档列表**:
1. `notes/business/prd/clob_prd.md` - CLOB 模块 PRD
2. `notes/business/prd/perpetuals_prd.md` - Perpetuals 模块 PRD
3. `notes/business/prd/listing_prd.md` - Listing 模块 PRD
4. `notes/business/prd/prices_prd.md` - Prices 模块 PRD
5. `notes/business/prd/stats_prd.md` - Stats 模块 PRD
6. `notes/business/prd/vault_prd.md` - Vault 模块 PRD
7. `notes/business/prd/leverage_prd.md` - Leverage 模块 PRD
8. `notes/business/prd/README.md` - PRD 文档总览

#### 3.2 PRD 文档重点关注

**CLOB 模块 PRD 示例功能需求**:

**FR-1: 订单放置**
- 功能点: 支持 8 种订单类型 (Short-Term, Long-Term, Conditional, TWAP, Reduce-Only, Post-Only, IOC, FOK)
- 业务规则: 订单必须有足够保证金、短期订单单区块有效、长期订单到期自动清理
- 验收标准: 用户可成功放置有效订单、保证金不足被拒绝、过期订单自动清理
- 参考测试: TestPlaceOrder, TestPlaceShortTermOrder, TestPlaceLongTermOrder

**FR-2: 订单匹配**
- 功能点: 价格-时间优先、部分成交支持、确定性匹配
- 业务规则: 匹配必须确定性、匹配后立即更新账户余额
- 验收标准: 买卖订单正确匹配、价格-时间优先严格执行
- 参考测试: TestMatchOrders, TestPriceTimePriority, TestPartialFill

**FR-3: 清算机制**
- 功能点: 自动清算保证金不足的账户
- 业务规则: 当账户抵押品 < 维持保证金要求时触发清算
- 计算公式:
  - **保证金率计算**:

    数学表达式:
    ```
    保证金率 = 净抵押品价值 / 仓位名义价值
    ```

    变量说明:
    - 净抵押品价值: 账户中的 USDC 余额,单位为 quote quantums
    - 仓位名义价值: 所有永续合约仓位的总价值,单位为 quote quantums

    清算条件:
    ```
    如果 保证金率 < 维持保证金率 → 触发清算
    ```

    计算示例:
    - 假设用户账户有 10,000 USDC (净抵押品价值)
    - 持有 BTC 永续合约仓位,名义价值 50,000 USDC
    - 保证金率 = 10,000 / 50,000 = 20%
    - 如果维持保证金率要求为 5%,则该账户安全
    - 如果维持保证金率要求为 25%,则触发清算

    业务解释:
    保证金率反映了账户的安全程度。当价格波动导致仓位价值增加或抵押品价值减少时,保证金率会下降。一旦低于系统设定的维持保证金率,系统会强制平仓以保护系统和其他用户。

**FR-4: 速率限制**
**FR-5: MEV 保护**
...

**用户场景示例**:

**场景 1: 限价单交易**
- 参与者: 普通交易者
- 前置条件: 账户有足够保证金
- 流程:
  1. 用户创建限价买单 (BTC-USD, 价格=50000, 数量=1)
  2. 系统验证保证金充足
  3. 订单进入订单簿
  4. 当卖方出现匹配价格时,自动成交
  5. 用户收到成交通知,余额更新
- 后置条件: 订单完全成交或部分成交

---

### 阶段 4: 总览文档生成

#### 4.1 生成各类文档总览

**数据结构总览** (`notes/data_structure/README.md`):
- 4 种存储类型对比
- 各模块数据分布
- 数据访问性能分析
- 链接到各模块详细文档

**架构总览** (`notes/architecture/README.md`):
- 交易系统整体架构概览
- 7 个模块的定位和职责
- 模块间协作流程
- 链接到各模块详细架构文档

**模块依赖关系图** (`notes/architecture/module_dependencies.md`):
- 7 个模块的依赖关系可视化图 (Mermaid)
- 依赖说明和数据流向
- 循环依赖分析 (如果存在)
- 模块初始化顺序

**PRD 总览** (`notes/business/prd/README.md`):
- 产品需求总览
- 7 个模块的产品定位
- 模块间的产品协作
- 链接到各模块详细 PRD

---

## 执行顺序建议

### 优先级排序

1. **高优先级**: 数据结构文档 (最重要,其他文档依赖)
2. **中优先级**: 架构与系统设计文档
3. **低优先级**: PRD 文档 (依赖前两者)
4. **补充**: 总览文档

### 建议执行顺序

**第一批: 核心模块数据结构 (并行)**
1. 生成 CLOB 数据结构文档 (重点中的重点)
2. 生成 Perpetuals 数据结构文档
3. 生成其他模块数据结构文档

**第二批: 架构文档 (并行)**
4. 使用 architect 生成 CLOB 架构文档
5. 使用 architect 生成 Perpetuals 架构文档
6. 使用 architect 生成其他模块架构文档
7. 生成模块依赖关系图和架构总览

**第三批: PRD 文档 (依赖架构文档)**
8. 使用 analyst 生成 CLOB PRD
9. 使用 analyst 生成 Perpetuals PRD
10. 使用 analyst 生成其他模块 PRD
11. 生成 PRD 总览

**第四批: 总览文档**
12. 生成数据结构总览文档

---

## 关键注意事项

### 文档质量要求

1. **数据结构文档**:
   - 重点关注 StateStore (链上持久化)
   - 每个数据结构都要说明业务含义
   - 使用场景要具体
   - 数据访问模式要清晰
   - 按模块组织,每个文件内部区分存储类型

2. **架构文档**:
   - 重点描述**为什么**这样设计,而非**是什么**
   - 技术决策要有权衡分析
   - 数据流和组件交互要清晰
   - 不包含代码细节,专注架构层面

3. **PRD 文档**:
   - ❌ **绝对不能包含代码**
   - 以产品经理视角描述功能
   - 每个需求都映射到测试用例
   - 使用用户可理解的语言,避免技术术语
   - ❌ **不包含版本规划部分** (只文档当前实现)
   - ✅ **公式文档化要求** (NEW):
     - 当需求涉及计算公式时,必须在需求描述中明确列出公式
     - 对每个公式进行详细解释,说明:
       - 公式的业务含义
       - 各变量的定义和单位
       - 计算步骤(如果公式复杂)
       - 具体计算示例(带入实际数值演示)
     - 公式使用数学符号表示,并配以文字说明
     - 避免使用代码表达公式,改用数学表达式

### 参考文件使用

- `notes/business/clob_analyst.md`: 文件过大 (492.6KB),需要分段阅读,重点提取架构和数据结构信息
- `notes/business/perpetuals_analyst.md`: 包含详细的业务流程和数据模型,直接参考
- `notes/business/slinky_oracle.md`: 价格预言机集成逻辑,用于 Prices 模块文档

### 文档一致性

- 所有文档使用统一的术语
- 模块间的依赖关系描述要一致
- 数据流在不同文档中要对应
- 交叉引用确保准确性

---

## 成功标准

文档生成完成后,应满足以下标准:

1. ✅ 所有 7 个模块都有完整的数据结构文档 (按模块组织)
2. ✅ 所有 7 个模块都有完整的架构文档
3. ✅ 所有 7 个模块都有完整的 PRD 文档
4. ✅ 架构文档重点描述设计理由和权衡
5. ✅ PRD 文档完全不包含代码,面向产品视角
6. ✅ PRD 文档不包含版本规划,只文档当前实现
7. ✅ 数据结构文档详细说明业务含义和使用场景
8. ✅ 所有文档使用统一术语,交叉引用准确
9. ✅ 文档存放位置符合规划,便于查找
10. ✅ **PRD 文档中涉及计算的需求,必须包含完整的公式说明** (NEW):
    - 公式使用数学表达式而非代码
    - 包含变量定义和单位
    - 提供具体计算示例
    - 配有业务含义解释

---

## 下一步行动

等待用户 Review 本计划后,按以下顺序执行:

### Phase 1: 数据结构文档 (优先)

- [ ] 生成 CLOB 数据结构文档 (重点中的重点)
- [ ] 生成 Perpetuals 数据结构文档
- [ ] 生成 Listing 数据结构文档
- [ ] 生成 Prices 数据结构文档
- [ ] 生成 Stats 数据结构文档
- [ ] 生成 Vault 数据结构文档
- [ ] 生成 Leverage 数据结构文档
- [ ] 生成数据结构总览文档

### Phase 2: 架构文档

- [ ] 使用 architect 生成 CLOB 架构文档
- [ ] 使用 architect 生成 Perpetuals 架构文档
- [ ] 使用 architect 生成 Listing 架构文档
- [ ] 使用 architect 生成 Prices 架构文档
- [ ] 使用 architect 生成 Stats 架构文档
- [ ] 使用 architect 生成 Vault 架构文档
- [ ] 使用 architect 生成 Leverage 架构文档
- [ ] 生成模块依赖关系图
- [ ] 生成架构总览文档

### Phase 3: PRD 文档

- [ ] 使用 analyst 生成 CLOB PRD
- [ ] 使用 analyst 生成 Perpetuals PRD
- [ ] 使用 analyst 生成 Listing PRD
- [ ] 使用 analyst 生成 Prices PRD
- [ ] 使用 analyst 生成 Stats PRD
- [ ] 使用 analyst 生成 Vault PRD
- [ ] 使用 analyst 生成 Leverage PRD
- [ ] 生成 PRD 总览文档

---

请 Review 此计划,确认是否符合预期。如有调整,请告知,我会据此修改计划。
