# Hermes DEX 清算与去杠杆机制分析计划

## 任务概述

本计划旨在全面分析 Hermes DEX 的清算(Liquidation)和去杠杆(Deleveraging)机制,包括:

1. **需求1**: 根据代码实现、PRD和数据结构,梳理清算、去杠杆的业务机制和实现流程
2. **需求2**: 分析 OrderFill 订单履行值在哪个 ABCI 阶段更新
3. **需求3**: 梳理 Hyperliquid 清算和去杠杆机制作为对比

## 探索结果总结

通过3个并行的探索代理,已经完成了以下分析:

### 探索结果 1: 清算机制实现
- **核心文件**: `protocol/x/clob/keeper/liquidations.go` (1283行)
- **关键函数**:
  - `GetFillablePrice()` - 清算价格计算
  - `GetBankruptcyPriceInQuoteQuantums()` - 破产价格计算
  - `GetLiquidationInsuranceFundDelta()` - 保险基金变化计算
  - `PlacePerpetualLiquidation()` - 清算订单放置

### 探索结果 2: 去杠杆机制实现
- **核心文件**: `protocol/x/clob/keeper/deleveraging.go` (721行)
- **关键函数**:
  - `CanDeleverageSubaccount()` - 去杠杆触发条件判断
  - `OffsetSubaccountPerpetualPosition()` - 对手方匹配
  - `ProcessDeleveraging()` - 去杠杆执行

### 探索结果 3: OrderFill 更新机制
- **更新阶段**: **DeliverTx** (FinalizeBlock的子阶段)
- **核心文件**: `protocol/x/clob/keeper/process_single_match.go`
- **关键函数**: `setOrderFillAmountsAndPruning()` → `SetOrderFillAmount()`

## 最终输出计划

### 输出文档 1: 清算机制详细分析 (`notes/business/liquidations/liquidation_mechanism.md`)

#### 1.1 代码调用流程及详细分析

**完整调用链路** (含代码位置):
```
PrepareCheckState (protocol/x/clob/abci.go:262-279)
  ↓
LiquidateSubaccountsAgainstOrderbook() (liquidations.go:55-162)
  ├─ 遍历所有需要清算的子账户
  ├─ 调用 MaybeGetLiquidationOrder()
  └─ 调用 PlacePerpetualLiquidation()
  ↓
【步骤1: 生成清算订单】
MaybeGetLiquidationOrder(liquidations.go:164-197)
  ├─ Line 177: EnsureIsLiquidatable() → 检查 TNC < MMR
  ├─ Line 182: GetPerpetualPositionToLiquidate() → 选择要清算的持仓
  └─ Line 194: GetLiquidationOrderForPerpetual()
      ├─ Line 873: GetLiquidatablePositionSizeDelta() → 计算清算规模
      ├─ Line 896: GetFillablePrice() → 计算可成交价格 ⭐
      │   ├─ Line 535-537: 获取子账户和持仓信息
      │   ├─ Line 541-550: 验证 deltaQuantums 有效性
      │   ├─ Line 552-558: 获取永续合约、市场价格、流动性层级
      │   ├─ Line 560-566: 计算持仓的风险参数 (riskPos)
      │   ├─ Line 568-574: 获取账户总风险参数 (riskTotal)
      │   ├─ Line 604-619: 计算调整后的破产评级 (ABR)
      │   ├─ Line 622-625: 计算最大清算价差 (SMMR * PMMR)
      │   ├─ Line 627: 计算价格偏离 (ABR * maxSpread)
      │   ├─ Line 629-636: 计算可成交价格 (PNNV - 价格偏离) / PS
      │   └─ Line 641-649: 返回可成交价格
      └─ Line 902: ConvertFillablePriceToSubticks() → 转换为 subticks
  ↓
【步骤2: 放置清算订单】
PlacePerpetualLiquidation(liquidations.go:246-350)
  ├─ Line 276-283: validateLiquidationAgainstClobPairStatus() → 验证市场状态
  ├─ Line 288-305: MemClob.PlacePerpetualLiquidation() → 在内存订单簿匹配
  │   └─ 尝试与现有订单匹配,生成成交
  └─ Line 342: MustUpdateSubaccountPerpetualLiquidated() → 更新清算状态
  ↓
【步骤3: 匹配与结算】
ProcessSingleMatch(process_single_match.go:44-317)
  ├─ Line 91-97: 验证匹配的基本信息
  ├─ Line 139-180: validateMatchedLiquidation() → 清算特殊验证
  │   ├─ Line 1024: GetLiquidationInsuranceFundDelta() → 计算保险基金变化 ⭐
  │   │   ├─ Line 668-675: 验证 fillAmount 非零
  │   │   ├─ Line 678-689: 计算 deltaQuantums 和 deltaQuoteQuantums
  │   │   ├─ Line 692-700: 调用 GetBankruptcyPriceInQuoteQuantums() → 计算破产价格
  │   │   ├─ Line 704-707: 计算保险基金变化 = 实际成交 - 破产价格
  │   │   ├─ Line 712-713: 如果 <= 0,保险基金需要补偿
  │   │   └─ Line 722-732: 如果 > 0,收取清算费(不超过上限)
  │   ├─ Line 1032: IsValidInsuranceFundDelta() → 验证保险基金充足
  │   └─ Line 1050: validateLiquidationAgainstSubaccountBlockLimits() → 验证区块限制
  └─ Line 227-241: persistMatchedOrders() → 持久化成交
      ├─ Line 432-467: TransferInsuranceFundPayments() → 转账保险基金
      │   ├─ Line 395-424 (transfer.go): 确定转账方向
      │   ├─ 保险基金 → 抵押池 (insuranceFundDelta < 0)
      │   └─ 抵押池 → 保险基金 (insuranceFundDelta > 0)
      └─ Line 468-520: UpdateSubaccounts() → 更新账户余额和仓位
```

**关键函数详细分析**:

1. **GetFillablePrice()** (`liquidations.go:514-650`)
   - **输入**: 子账户ID、永续合约ID、清算数量
   - **核心逻辑**:
     - 获取账户总风险参数 (TNC, TMMR)
     - 计算持仓风险参数 (PNNV, PMMR)
     - 计算 ABR = BA × (1 - TNC/TMMR),并 Clamp 到 [0, 1]
     - 计算价格偏离 = ABR × SMMR × PMMR
     - 计算可成交价格 = (PNNV - 价格偏离) / PS
   - **输出**: 可成交价格 (big.Rat 类型)

2. **GetBankruptcyPriceInQuoteQuantums()** (`liquidations.go:404-509`)
   - **输入**: 子账户ID、永续合约ID、持仓变化量
   - **核心逻辑**:
     - 计算持仓变化前后的风险参数
     - 计算 DNNV = riskPosNew.NC - riskPosOld.NC
     - 计算 DMMR = riskPosNew.MMR - riskPosOld.MMR
     - 计算破产价格 = -DNNV - (TNC × abs(DMMR) / TMMR)
     - **向正无穷舍入**,确保保守估计
   - **输出**: 破产价格 (big.Int 类型)

3. **GetLiquidationInsuranceFundDelta()** (`liquidations.go:656-733`)
   - **输入**: 子账户ID、永续合约ID、买卖方向、成交量、成交价格
   - **核心逻辑**:
     - 计算实际成交的 deltaQuoteQuantums
     - 获取破产价格对应的 bankruptcyPriceQuoteQuantums
     - 计算 insuranceFundDelta = deltaQuoteQuantums - bankruptcyPrice
     - 如果 <= 0: 保险基金需要补偿
     - 如果 > 0: 收取清算费,但不超过 MaxLiquidationFeePpm
   - **输出**: 保险基金变化量 (big.Int 类型)

#### 1.2 清算价格计算机制

**内容要点**:
- **可成交价格 (Fillable Price) 公式**:
  ```
  fillablePrice = (PNNV - ABR * SMMR * PMMR) / PS

  其中:
  - PNNV: 持仓净名义价值
  - ABR: 调整后的破产评级 = BA * (1 - TNC/TMMR)
  - SMMR: 价差与维持保证金比率
  - PMMR: 持仓维持保证金要求
  - PS: 持仓规模
  ```
- **代码实现**: `liquidations.go:514-650`
- **业务含义**:
  - 多头清算: fillable price < oracle price (打折卖出)
  - 空头清算: fillable price > oracle price (溢价买入)

- **破产价格 (Bankruptcy Price) 公式**:
  ```
  破产价格 = -DNNV - (TNC * abs(DMMR) / TMMR)

  其中:
  - DNNV: 持仓净名义价值变化
  - DMMR: 维持保证金要求变化
  - TNC: 总净抵押品
  - TMMR: 总维持保证金要求
  ```
- **代码实现**: `liquidations.go:404-509`
- **舍入规则**: 向正无穷舍入,确保不需要额外保险基金支付

#### 1.2 保险基金机制详解

**内容要点**:
- **保险基金变化计算**:
  ```
  insuranceFundDelta = 实际成交价值 - 破产价格价值
  ```
- **代码实现**: `liquidations.go:656-733` - `GetLiquidationInsuranceFundDelta()`
- **三种场景**:
  - delta > 0: 被清算账户支付费用给保险基金
  - delta = 0: 无资金转移
  - delta < 0: **保险基金补偿损失**(账户已破产)

- **保险基金架构**:
  - 每个永续合约可以有独立的保险基金(隔离市场)
  - Cross市场共享一个保险基金(`perptypes.InsuranceFundModuleAddress`)
  - 保险基金余额存储在 `x/bank` 模块

- **转账实现**: `subaccounts/keeper/transfer.go:390-434`

#### 1.3 强制平仓补偿与社会化损失

**内容要点**:
- **强制平仓补偿**:
  - 场景: 清算成交价 < 破产价格(对空头清算)
  - 保险基金补偿差额(负 insuranceFundDelta)
  - 代码: `transfer.go:395-424` 从保险基金转账到抵押池

- **社会化损失 (Socialized Loss)**:
  - **定义**: 当保险基金不足时,对手方强制吸收破产账户损失
  - **实现机制**: 去杠杆流程
  - **代码路径**: `deleveraging.go:289-466` - `OffsetSubaccountPerpetualPosition()`
  - **损失分配**:
    - 对手方以**破产价格**(而非市场价格)成交
    - 差价 = 破产价格 - 市场价格
    - 对手方吸收损失
  - **体现位置**: `ProcessDeleveraging()` 中双方账户更新

#### 1.4 清算执行流程详解

**完整调用链** (代码路径):
```
PrepareCheckState (abci.go:262-279)
  ↓
LiquidateSubaccountsAgainstOrderbook()
  ↓
【生成清算订单】
MaybeGetLiquidationOrder(liquidations.go:164-197)
  ├─ EnsureIsLiquidatable() → TNC < MMR ?
  ├─ GetPerpetualPositionToLiquidate() → 选择持仓
  └─ GetLiquidationOrderForPerpetual()
      ├─ GetLiquidatablePositionSizeDelta() → 计算规模
      ├─ GetFillablePrice() → 计算可成交价格 ⭐
      └─ ConvertFillablePriceToSubticks()
  ↓
【放置清算订单】
PlacePerpetualLiquidation(liquidations.go:246-350)
  ├─ validateLiquidationAgainstClobPairStatus()
  ├─ MemClob.PlacePerpetualLiquidation()
  └─ MustUpdateSubaccountPerpetualLiquidated()
  ↓
【匹配与结算】
ProcessSingleMatch(process_single_match.go:44-317)
  ├─ validateMatchedLiquidation()
  │   ├─ GetLiquidationInsuranceFundDelta() ⭐
  │   ├─ IsValidInsuranceFundDelta() → 保险基金充足?
  │   └─ validateLiquidationAgainstSubaccountBlockLimits()
  └─ persistMatchedOrders()
      ├─ TransferInsuranceFundPayments() → 转账保险基金
      └─ UpdateSubaccounts() → 更新账户状态
  ↓
【去杠杆(如清算未完全成交)】
MaybeDeleverageSubaccount()
```

#### 1.5 实例解释

**实例 1: 清算成功,保险基金收费**
```
用户 Alice:
- 持仓: 10 BTC 多头,开仓价 50,000 USDC
- 余额: 3,000 USDC
- 当前价格: 49,000 USDC

清算触发:
1. TNC = 3,000 + (49,000 - 50,000) × 10 = 3,000 - 10,000 = -7,000 USDC
2. MMR = 10 × 49,000 × 2.5% = 12,250 USDC
3. TNC < MMR → 触发清算

清算执行:
1. 可成交价格 = 48,800 USDC (略低于市场价)
2. 破产价格 = 50,000 - (3,000 / 10) = 49,700 USDC
3. 清算成交 @ 48,800 USDC
4. 成交价值 = 10 × 48,800 = 488,000 USDC
5. 破产价格价值 = 10 × 49,700 = 497,000 USDC
6. insuranceFundDelta = 488,000 - 497,000 = -9,000 USDC
7. 保险基金支付 9,000 USDC 补偿损失
```

**实例 2: 清算失败,触发去杠杆**
```
用户 Bob:
- 持仓: -5 BTC 空头,开仓价 50,000 USDC
- 余额: 1,000 USDC
- 当前价格: 55,000 USDC (暴涨)

清算失败场景:
1. TNC = 1,000 + (50,000 - 55,000) × (-5) = 1,000 - 25,000 = -24,000 USDC (严重破产)
2. 清算订单无法在订单簿匹配(流动性不足)
3. 进入去杠杆流程

去杠杆执行:
1. 破产价格 = 50,000 + (1,000 / 5) = 50,200 USDC
2. 系统查找多头对手方(假设 Carol: +5 BTC,盈利中)
3. 强制匹配: Bob 买入 5 BTC @ 50,200,Carol 卖出 5 BTC @ 50,200
4. Bob 账户归零(TNC = 0)
5. Carol 损失 = (55,000 - 50,200) × 5 = 24,000 USDC (社会化损失)
```

### 输出文档 2: 去杠杆机制详细分析 (`notes/business/liquidations/deleveraging_mechanism.md`)

#### 2.1 代码调用流程及详细分析

**完整调用链路** (含代码位置):
```
PrepareCheckState (protocol/x/clob/abci.go:262-279)
  ↓
【步骤1: 收集需要去杠杆的账户】
DeleverageSubaccounts() (deleveraging.go:691-720)
  ├─ Line 701-706: 收集清算失败的账户 (subaccountsToDeleverage)
  ├─ Line 710-718: 收集最终结算市场的持仓账户
  │   └─ GetSubaccountsWithPositionsInFinalSettlementMarkets() (654-687)
  │       ├─ 遍历所有 CLOB Pairs
  │       ├─ 检查市场状态 == STATUS_FINAL_SETTLEMENT
  │       └─ 收集该市场的所有持仓账户
  └─ Line 720: 遍历所有需要去杠杆的账户,调用 MaybeDeleverageSubaccount()
  ↓
【步骤2: 判断是否需要去杠杆】
MaybeDeleverageSubaccount() (deleveraging.go:35-140)
  ├─ Line 50-54: 获取子账户和持仓信息
  ├─ Line 71-82: CanDeleverageSubaccount() → 判断触发条件 ⭐
  │   ├─ Line 166-174: 检查 TNC < 0 (负净抵押品)
  │   │   └─ 返回 (true, false) → 破产价格去杠杆
  │   └─ Line 177-194: 检查市场是否最终结算
  │       └─ 返回 (false, true) → 预言机价格去杠杆
  ├─ Line 90-97: 计算去杠杆数量 (deltaQuantums = -positionSize)
  └─ Line 106-134: MemClob.DeleverageSubaccount() → 执行去杠杆
  ↓
【步骤3: 查找对手方并匹配】
OffsetSubaccountPerpetualPosition() (deleveraging.go:295-466)
  ├─ Line 319-322: 获取对手方持仓列表
  │   └─ DaemonLiquidationInfo.GetSubaccountsWithOpenPositionsOnSide()
  │       └─ 返回持有相反方向持仓的所有账户
  ├─ Line 337-341: 随机起点遍历 (防止MEV)
  │   ├─ pseudoRand := GetPseudoRand(ctx)
  │   └─ indexOffset := pseudoRand.Intn(numSubaccounts)
  ├─ Line 343-443: 遍历对手方账户,逐个匹配
  │   ├─ Line 356-361: 计算本次匹配的数量
  │   ├─ Line 368-384: getDeleveragingQuoteQuantumsDelta() → 计算去杠杆价格 ⭐
  │   │   ├─ Line 478-482: 如果是最终结算
  │   │   │   └─ 使用预言机价格: GetNetNotional()
  │   │   └─ Line 483-487: 如果是负TNC
  │   │       └─ 使用破产价格: GetBankruptcyPriceInQuoteQuantums()
  │   └─ Line 387-394: ProcessDeleveraging() → 执行单次去杠杆
  └─ Line 449-459: 如果完全成交,返回成交列表;否则返回部分成交
  ↓
【步骤4: 执行去杠杆并更新状态】
ProcessDeleveraging() (deleveraging.go:502-642)
  ├─ Line 526-538: 验证 deltaQuantums 有效性
  │   ├─ 与被清算仓位方向相反
  │   └─ 不超过双方持仓大小
  ├─ Line 547-580: 构建双方账户更新
  │   ├─ 被去杠杆账户: PerpetualUpdate + AssetUpdate
  │   └─ 抵消账户: 相反的 PerpetualUpdate + AssetUpdate
  ├─ Line 583-591: 应用更新到状态
  │   └─ subaccountsKeeper.UpdateSubaccounts(updates, satypes.Match)
  │       ├─ 更新被去杠杆账户的余额和仓位
  │       ├─ 更新抵消账户的余额和仓位
  │       └─ 验证更新后状态有效
  └─ Line 614-639: 发出去杠杆事件
      └─ NewCreateMatchEvent(IsDeleverage: true, IsLiquidation: false)
  ↓
【步骤5: 处理零成交场景】
GateWithdrawalsIfNegativeTncSubaccountSeen() (deleveraging.go:197-260)
  ├─ Line 213-216: 检查是否有负TNC账户
  ├─ Line 250: 插入零成交去杠杆操作
  │   └─ InsertZeroFillDeleveragingIntoOperationsQueue(subaccountId, perpetualId)
  └─ Line 255-257: 设置提现门控
      └─ SetNegativeTncSubaccountSeenAtBlock(ctx, blockHeight)
```

**关键函数详细分析**:

1. **CanDeleverageSubaccount()** (`deleveraging.go:151-195`)
   - **输入**: 子账户ID、永续合约ID
   - **核心逻辑**:
     - 获取账户净资产和保证金要求 (TNC, MMR)
     - 如果 TNC < 0: 返回 (true, false) → 破产价格去杠杆
     - 否则检查市场状态: 如果 STATUS_FINAL_SETTLEMENT,返回 (false, true)
   - **输出**: (shouldDeleverageAtBankruptcyPrice, shouldDeleverageAtOraclePrice, error)

2. **getDeleveragingQuoteQuantumsDelta()** (`deleveraging.go:471-490`)
   - **输入**: 永续合约ID、子账户ID、持仓变化量、是否最终结算
   - **核心逻辑**:
     - 如果 isFinalSettlement: 调用 GetNetNotional() 获取预言机价格价值
     - 否则: 调用 GetBankruptcyPriceInQuoteQuantums() 获取破产价格价值
   - **输出**: 去杠杆应支付/收取的报价余额 (big.Int)

3. **OffsetSubaccountPerpetualPosition()** (`deleveraging.go:295-466`)
   - **输入**: 被去杠杆账户ID、永续合约ID、持仓变化量、是否最终结算
   - **核心逻辑**:
     - 从 DaemonLiquidationInfo 获取对手方列表
     - 使用伪随机起点遍历 (确定性但防止MEV)
     - 对每个对手方: 计算匹配数量 → 计算去杠杆价格 → 执行去杠杆
     - 累积成交,直到完全平仓或遍历完所有对手方
   - **输出**: 成交列表 (MatchPerpetualDeleveraging[])

4. **ProcessDeleveraging()** (`deleveraging.go:502-642`)
   - **输入**: 被去杠杆账户ID、抵消账户ID、永续合约ID、数量变化、报价变化
   - **核心逻辑**:
     - 验证数量有效性 (方向相反、不超过持仓)
     - 构建双方更新: PerpetualUpdate + AssetUpdate
     - 调用 UpdateSubaccounts() 原子性更新状态
     - 发出去杠杆事件到索引器
   - **输出**: error (成功返回 nil)

#### 2.2 触发条件详解

**条件 1: 负净抵押品 (TNC < 0)**
- **判断逻辑**: `deleveraging.go:166-174`
- **计算**: TNC = Account Balance + Unrealized PnL
- **触发时机**: 账户已破产,清算无法覆盖损失
- **去杠杆价格**: 破产价格

**条件 2: 最终结算 (Final Settlement)**
- **判断逻辑**: `deleveraging.go:177-194`
- **触发时机**: 市场状态 = `ClobPair_STATUS_FINAL_SETTLEMENT`
- **特点**: 即使账户健康也必须去杠杆
- **去杠杆价格**: 预言机价格

#### 2.2 社会化损失详细分析

**实现机制**:
- **对手方选择**: 按盈利率排序(PnL_Ratio = UnrealizedPnL / PositionValue)
- **损失分配**: 盈利最多的账户优先承担
- **代码实现**: `deleveraging.go:317-443`

**损失计算**:
```
假设被去杠杆账户 Eve:
- 持仓: 10 BTC 多头,开仓价 50,000 USDC
- 余额: 2,000 USDC
- 破产价格: 49,800 USDC
- TNC = -1,000 USDC (破产)

对手方 Frank (空头):
- 持仓: -10 BTC,开仓价 52,000 USDC
- 当前市场价: 49,700 USDC
- 本应盈利: (52,000 - 49,700) × 10 = 23,000 USDC

去杠杆执行:
1. Eve 平仓 @ 49,800 (破产价格)
2. Frank 被迫 @ 49,800 平仓
3. Frank 实际盈利: (52,000 - 49,800) × 10 = 22,000 USDC
4. Frank 损失 = 23,000 - 22,000 = 1,000 USDC (吸收Eve的负TNC)
```

#### 2.3 零成交去杠杆与提现门控

**场景**: 找不到足够对手方
- **代码实现**: `deleveraging.go:197-260`
- **操作**: 插入零成交去杠杆操作 (`InsertZeroFillDeleveragingIntoOperationsQueue`)
- **门控机制**: 阻止所有提现(`SetNegativeTncSubaccountSeenAtBlock`)
- **解除条件**: 外部资金注入或问题解决

#### 2.4 去杠杆与清算的区别

| 维度 | 清算 | 去杠杆 |
|------|------|-------|
| **触发条件** | MMR < TNC < IMR | TNC < 0 或 最终结算 |
| **执行方式** | 在订单簿下单 (IOC) | 强制匹配对手方 |
| **价格** | 可成交价格 | 破产价格/预言机价格 |
| **对手方** | 市场参与者(自愿) | 持仓账户(**强制**) |
| **费用** | 保险基金收费 | 对手方吸收损失 |
| **代码位置** | `liquidations.go` | `deleveraging.go` |

### 输出文档 3: OrderFill 更新机制分析 (`notes/business/liquidations/orderfill_update.md`)

#### 3.1 更新阶段

**ABCI 阶段**: **DeliverTx** (具体是处理 MsgProposedOperations 时)

**完整时序**:
```
FinalizeBlock
  ├─ PreBlock
  ├─ BeginBlock
  ├─ DeliverTx[3]: ProposedOperations ⭐ OrderFill 更新发生在这里
  │   ├─ ProcessProposerOperations()
  │   │   ├─ ProcessInternalOperations()
  │   │   │   └─ ProcessSingleMatch()
  │   │   │       ├─ GetOrderFillAmount() → 读取旧填充量
  │   │   │       ├─ 计算新填充量 = 旧填充量 + fillAmount
  │   │   │       └─ setOrderFillAmountsAndPruning()
  │   │   │           └─ SetOrderFillAmount() ⭐⭐⭐
  │   │   │               └─ KVStore.Set(orderId, OrderFillState)
  └─ EndBlock
      └─ PruneStateFillAmountsForShortTermOrders() → 清理
```

#### 3.2 数据结构

**Protobuf 定义** (`proto/hermesprotocol/clob/order.proto:64`):
```protobuf
message OrderFillState {
  uint64 fill_amount = 1;                // 累计成交量
  uint32 prunable_block_height = 2;      // 可清理区块高度
}
```

#### 3.3 存储机制

- **Key**: `OrderAmountFilledKeyPrefix` + `orderId.ToStateKey()`
- **Value**: Protobuf 序列化的 `OrderFillState`
- **代码**: `keeper/order_state.go:69-94`

#### 3.4 清理机制

**触发时机**: EndBlock
- **短期订单**: `GoodTilBlock + ShortBlockWindow` 后清理
- **状态化订单**: `pruneableBlockHeight = math.MaxUint32` (永久保存)
- **代码**: `abci.go:78-88`, `order_state.go:287-236`

### 输出文档 4: Hyperliquid 对比分析 (`notes/business/liquidations/hyperliquid_comparison.md`)

#### 4.1 Hyperliquid 清算机制

**清算触发**:
- 维持保证金 = 初始保证金的 50%
- 使用 Mark Price (结合 CEX 价格和订单簿状态)
- 当账户权益 < 维持保证金时触发

**清算执行**:
1. 市价单提交到订单簿(全仓位规模)
2. 如果满足维持保证金,剩余抵押品归交易者
3. 如果账户权益 < 2/3 维持保证金且订单簿清算失败:
   - 启动 Backstop Liquidation
   - 由 Liquidator Vault (HLP) 接管

#### 4.2 Hyperliquid Auto-Deleveraging (ADL)

**触发条件**:
- 用户账户价值或独立持仓价值为负
- HLP (Hyperliquid Liquidity Provider Vault) 无法覆盖剩余损失

**执行逻辑**:
1. 对手方按**未实现盈亏**和**杠杆**排序
2. 盈利最多且杠杆最高的优先被去杠杆
3. 强制平仓价格: **当前 Mark Price**

**2025年10月事件**:
- Hyperliquid 首次触发跨保证金 ADL
- 12分钟内触发超过40次 ADL 事件
- 总清算金额超过 $17M

#### 4.3 Hermes vs Hyperliquid 对比

| 维度 | Hermes DEX | Hyperliquid |
|------|------------|-------------|
| **清算价格** | Fillable Price (基于公式) | Mark Price (CEX + 订单簿) |
| **保险基金** | 每个永续合约可独立 | 统一 HLP Vault |
| **去杠杆触发** | TNC < 0 或 最终结算 | 账户价值 < 0 且 HLP 不足 |
| **去杠杆价格** | 破产价格/预言机价格 | Mark Price |
| **对手方选择** | 伪随机遍历 | 按 PnL + 杠杆排序 |
| **社会化损失** | 对手方吸收(破产价差) | 对手方吸收(Mark Price) |
| **提现门控** | 负TNC时自动门控 | 未明确说明 |

**参考来源**:
- [Hyperliquid Liquidations Documentation](https://hyperliquid.gitbook.io/hyperliquid-docs/trading/liquidations)
- [Hyperliquid Auto-Deleveraging Documentation](https://hyperliquid.gitbook.io/hyperliquid-docs/trading/auto-deleveraging)
- [Hyperliquid ADL Activation News (2025)](https://wublockchain.medium.com/hyperliquid-activates-cross-margin-auto-deleveraging-for-the-first-time-what-are-hlp-and-adl-9eb811418e9b)

## 关键文件清单

### 清算相关
- `protocol/x/clob/keeper/liquidations.go` (1283行)
- `protocol/x/clob/keeper/liquidations_config.go` (70行)
- `protocol/x/clob/types/liquidation_order.go` (150行)
- `protocol/x/subaccounts/keeper/transfer.go` (600行)

### 去杠杆相关
- `protocol/x/clob/keeper/deleveraging.go` (721行)
- `protocol/x/clob/keeper/process_operations.go` (包含持久化逻辑)
- `protocol/x/clob/memclob/memclob.go` (包含 DeleverageSubaccount)

### OrderFill 相关
- `protocol/x/clob/keeper/process_single_match.go` (核心更新)
- `protocol/x/clob/keeper/order_state.go` (存储操作)
- `proto/hermesprotocol/clob/order.proto` (数据结构定义)

### PRD 与数据结构
- `notes/business/prd/clob_prd.md`
- `notes/business/prd/perpetuals_prd.md`
- `notes/data_structure/clob.md`

## 执行步骤

### 步骤 1: 创建输出目录
```bash
mkdir -p notes/business/liquidations
```

### 步骤 2: 编写清算机制文档
基于探索结果,编写 `notes/business/liquidations/liquidation_mechanism.md`,包含:
- 代码调用流程及详细分析
- 清算价格计算详解(公式 + 代码 + 示例)
- 保险基金机制(计算 + 转账 + 场景)
- 强制平仓补偿说明
- 社会化损失体现位置
- 完整调用链路图
- 实例解释

### 步骤 3: 编写去杠杆机制文档
基于探索结果,编写 `notes/business/liquidations/deleveraging_mechanism.md`,包含:
- 代码调用流程及详细分析
- 触发条件详解(负TNC + 最终结算)
- 社会化损失计算公式
- 对手方选择逻辑
- 零成交去杠杆处理
- 与清算的对比
- 实例解释

### 步骤 4: 编写 OrderFill 分析文档
基于探索结果,编写 `notes/business/liquidations/orderfill_update.md`,包含:
- ABCI 阶段定位
- 数据结构说明
- 存储机制分析
- 清理机制说明
- 完整时序图

### 步骤 5: 编写 Hyperliquid 对比文档
基于 Web 搜索结果,编写 `notes/business/liquidations/hyperliquid_comparison.md`,包含:
- Hyperliquid 清算机制
- Hyperliquid ADL 机制
- 与 Hermes 的对比表
- 2025年真实案例分析

### 步骤 6: 创建总结文档
编写 `notes/business/liquidations/README.md`,汇总所有分析结果,提供快速导航。

## 预期输出

完成后将生成以下文档:

```
notes/business/liquidations/
├── README.md                          # 总览与导航
├── liquidation_mechanism.md           # 清算机制详细分析
├── deleveraging_mechanism.md          # 去杠杆机制详细分析
├── orderfill_update.md                # OrderFill 更新机制
└── hyperliquid_comparison.md          # Hyperliquid 对比分析
```

每个文档将包含:
- ✅ 详细的代码路径和行号引用
- ✅ 完整的计算公式和变量说明
- ✅ 具体的实例解释和计算过程
- ✅ 清晰的调用链路图
- ✅ 中文注释和业务解释

## 注意事项

1. **代码引用准确性**: 所有代码路径和行号基于当前最新代码
2. **公式完整性**: 所有计算公式包含完整的变量说明
3. **实例真实性**: 所有实例基于真实代码逻辑推导
4. **对比客观性**: Hyperliquid 对比基于公开文档和新闻

## 下一步行动

等待用户审核此计划后,将执行以上步骤,生成完整的分析文档。
