# Vault 模块架构设计

## 1. 模块概述

### 1.1 职责定义

Vault (金库/流动性池) 模块负责 Hermes DEX 的**自动化做市和流动性管理**,职责包括:

- **Megavault 管理**: 统一的流动性池,聚合所有存款
- **份额机制**: 使用份额代表流动性提供者的权益
- **自动做市**: 自动在订单簿上放置和刷新订单
- **存提款管理**: 处理存款、提款和解锁逻辑

### 1.2 核心功能

**Megavault 系统**:
- 单一流动性池,支持所有交易对
- 份额制度,公平分配收益和损失
- 自动化做市,无需人工干预

**订单管理**:
- EndBlocker 自动刷新订单
- 根据 QuotingParams 生成报价
- 维持订单簿流动性

**风险控制**:
- 提款解锁期,防止闪电贷攻击
- 激活阈值,确保 Vault 有足够资金
- 杠杆限制,控制风险敞口

### 1.3 关键约束

**份额价值稳定**:
- 份额价值只能通过交易盈亏变化
- 存提款不改变现有持有者的份额价值
- 防止价值稀释

**确定性要求**:
- 订单生成必须确定性
- 份额计算必须精确
- 避免浮点运算

**流动性保证**:
- Vault 必须始终在订单簿上提供流动性
- 订单刷新必须及时
- 避免流动性断层

---

## 2. 架构设计

### 2.1 组件结构

```
Vault 模块架构

┌─────────────────────────────────────────────────┐
│         gRPC Query Server (查询接口)             │
│  TotalShares, OwnerShares, VaultParams,         │
│  MegavaultWithdrawalInfo                        │
└─────────────────────────────────────────────────┘
                       ↕
┌─────────────────────────────────────────────────┐
│         Msg Server (消息处理器)                  │
│  ├─ DepositToMegavault (存款)                   │
│  ├─ WithdrawFromMegavault (提款)                │
│  ├─ UpdateDefaultQuotingParams (更新报价参数)   │
│  └─ SetVaultParams (设置 Vault 参数)            │
└─────────────────────────────────────────────────┘
                       ↕
┌─────────────────────────────────────────────────┐
│              Keeper Layer (业务逻辑层)           │
│  ├─ ShareManagement (份额管理)                  │
│  │   ├─ MintShares() (铸造份额)                │
│  │   ├─ BurnShares() (销毁份额)                │
│  │   └─ GetShareValue() (计算份额价值)          │
│  ├─ OrderManagement (订单管理)                  │
│  │   ├─ RefreshAllVaultOrders() (刷新所有订单) │
│  │   ├─ GetVaultClobOrders() (生成订单)        │
│  │   └─ PlaceVaultOrders() (放置订单)          │
│  └─ WithdrawalManagement (提款管理)             │
│      ├─ LockSharesForWithdrawal() (锁定份额)   │
│      └─ ProcessMatureWithdrawals() (处理到期)  │
└─────────────────────────────────────────────────┘
                       ↕
┌─────────────────────────────────────────────────┐
│              Storage Layer (存储层)              │
│  StateStore:                                    │
│  ├─ TotalShares (总份额)                        │
│  ├─ OwnerShares (所有者份额)                    │
│  ├─ OwnerShareUnlocks (提款解锁计划)            │
│  ├─ VaultParams (Vault 参数)                    │
│  ├─ VaultAddress (Vault 地址映射)               │
│  └─ MostRecentClientIds (最近订单 ID)           │
└─────────────────────────────────────────────────┘
                       ↕
┌─────────────────────────────────────────────────┐
│              External Modules (外部模块依赖)     │
│  ├─ Subaccounts: 获取 Vault 净值               │
│  ├─ CLOB: 放置和取消订单                        │
│  ├─ Perpetuals: 获取市场配置                    │
│  └─ Prices: 获取当前价格                        │
└─────────────────────────────────────────────────┘
```

### 2.2 存储架构

#### StateStore (链上持久化存储)

**存储内容** (6 个数据结构):

1. **TotalShares** (总份额):
   - NumShares: Megavault 总份额数量
   - 份额价值 = Vault 总净值 / 总份额

2. **OwnerShares** (所有者份额):
   - Key: `OwnerSharesKeyPrefix` + `OwnerAddress`
   - Value: `NumShares` (该所有者持有的份额)
   - 提款金额 = 份额数 × 份额价值

3. **OwnerShareUnlocks** (份额解锁):
   - Key: `OwnerShareUnlocksKeyPrefix` + `OwnerAddress` + `UnlockIndex`
   - Value: `ShareUnlock` (Shares, UnlockBlockHeight)
   - 提款需要等待解锁期

4. **VaultParams** (Vault 参数):
   - Key: `VaultParamsKeyPrefix` + `VaultType` + `VaultNumber`
   - Value: `VaultParams` (Status, QuotingParams, ...)
   - 配置 Vault 的做市参数

5. **VaultAddress** (Vault 地址映射):
   - Key: `VaultAddressKeyPrefix` + `VaultType` + `VaultNumber`
   - Value: `VaultId`

6. **MostRecentClientIds** (最近订单客户端 ID):
   - Key: `MostRecentClientIdsKeyPrefix` + `VaultId`
   - Value: `MostRecentClientIds`
   - 跟踪 Vault 最近放置的订单 ID

### 2.3 ABCI 生命周期集成

```
        Vault 模块 ABCI 集成

┌─────────────────────────────────────────────────┐
│    BeginBlocker (可选)                           │
│    - 当前未使用                                  │
└─────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────┐
│    DeliverTx (交易执行)                          │
│    ├─ MsgDepositToMegavault                     │
│    │   ├─ 转账 USDC 到 Vault 模块账户           │
│    │   ├─ 计算份额数 = 存款金额 / 份额价值      │
│    │   ├─ 增加 TotalShares                      │
│    │   └─ 增加 OwnerShares[用户地址]            │
│    │                                             │
│    └─ MsgWithdrawFromMegavault                  │
│        ├─ 创建 ShareUnlock (锁定期)             │
│        ├─ 减少 OwnerShares (立即)               │
│        └─ 记录解锁计划                          │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────┐
│    EndBlocker (核心逻辑)                         │
│    ├─ RefreshAllVaultOrders()                   │
│    │   ├─ 遍历所有激活的 Vaults                 │
│    │   ├─ 取消旧订单 (基于 MostRecentClientIds) │
│    │   ├─ 根据 QuotingParams 生成新订单         │
│    │   │   ├─ 获取当前价格                      │
│    │   │   ├─ 计算价差 (Spread)                 │
│    │   │   ├─ 计算订单大小                      │
│    │   │   └─ 生成买卖订单对                    │
│    │   ├─ 放置新订单到 CLOB                     │
│    │   └─ 更新 MostRecentClientIds              │
│    │                                             │
│    └─ ProcessMatureWithdrawals()                │
│        ├─ 检查 ShareUnlocks 是否到期            │
│        ├─ 计算提款金额 = 份额数 × 份额价值      │
│        ├─ 转账 USDC 给用户                      │
│        ├─ 减少 TotalShares                      │
│        └─ 删除 ShareUnlock 记录                 │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
            下一个区块
```

---

## 3. 核心业务流程

### 3.1 存款流程

```
        用户存款 → Megavault 完整流程

┌─────────────────────────────────────────────────┐
│    用户提交 MsgDepositToMegavault                │
│    {                                            │
│      SubaccountId: 用户子账户,                  │
│      QuoteQuantums: 存款金额 (USDC)             │
│    }                                            │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────┐
│    消息验证                                      │
│    ├─ 检查 SubaccountId 是否有效                │
│    ├─ 检查 QuoteQuantums > 0                    │
│    └─ 检查用户账户余额充足                      │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────┐
│    计算份额数                                    │
│    ├─ VaultEquity = GetVaultEquity()            │
│    │   (Vault 总净值 = 资产 - 负债)             │
│    ├─ TotalShares = GetTotalShares()            │
│    ├─ ShareValue = VaultEquity / TotalShares    │
│    │   (份额价值)                                │
│    └─ SharesForDeposit = QuoteQuantums / ShareValue │
│        (铸造份额数)                              │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────┐
│    转账 USDC                                     │
│    ├─ 从用户 Subaccount 转账                    │
│    └─ 到 Vault 模块账户                         │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────┐
│    更新份额记录                                  │
│    ├─ TotalShares += SharesForDeposit           │
│    ├─ OwnerShares[用户地址] += SharesForDeposit │
│    └─ 持久化到 StateStore                       │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────┐
│    触发事件                                      │
│    └─ DepositToMegavaultEvent {                 │
│          SubaccountId,                          │
│          QuoteQuantums,                         │
│          MintedShares                           │
│       }                                         │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
            存款完成
```

#### 份额价值计算公式

**份额价值计算**:

数学表达式:
```
ShareValue = VaultEquity / TotalShares
```

变量说明:
- **ShareValue**: 单个份额的价值 (USDC)
- **VaultEquity**: Vault 总净值 (USDC)
  - VaultEquity = TotalAssets - TotalLiabilities
  - TotalAssets: Vault 持有的 USDC + 未平仓合约的未实现盈亏
  - TotalLiabilities: Vault 的负债
- **TotalShares**: Megavault 总份额数

计算示例:
- 假设 Vault 总净值 = 1,000,000 USDC
- 假设 TotalShares = 500,000
- ShareValue = 1,000,000 / 500,000 = 2 USDC/份额

业务解释:
- 份额价值随 Vault 盈亏动态变化
- 盈利时份额价值上升,亏损时下降
- 新存款不影响现有持有者的份额价值

**存款份额计算**:

数学表达式:
```
SharesForDeposit = DepositAmount / ShareValue
```

变量说明:
- **SharesForDeposit**: 存款铸造的份额数
- **DepositAmount**: 存款金额 (USDC)
- **ShareValue**: 当前份额价值 (USDC/份额)

计算示例:
- 假设用户存款 10,000 USDC
- 假设当前 ShareValue = 2 USDC/份额
- SharesForDeposit = 10,000 / 2 = 5,000 份额

业务解释:
- 份额价值高时,相同金额获得更少份额
- 份额价值低时,相同金额获得更多份额
- 确保公平性,避免价值稀释

### 3.2 提款流程

```
        用户提款 → Megavault 完整流程

┌─────────────────────────────────────────────────┐
│    用户提交 MsgWithdrawFromMegavault             │
│    {                                            │
│      SubaccountId: 用户子账户,                  │
│      Shares: 提款份额数                         │
│    }                                            │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────┐
│    消息验证                                      │
│    ├─ 检查 SubaccountId 是否有效                │
│    ├─ 检查 Shares > 0                           │
│    └─ 检查用户持有的份额充足                    │
│        (OwnerShares[用户地址] >= Shares)        │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────┐
│    锁定份额 (立即减少 OwnerShares)               │
│    ├─ OwnerShares[用户地址] -= Shares           │
│    └─ 份额进入解锁期,不能再交易                 │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────┐
│    创建解锁记录                                  │
│    ├─ UnlockBlockHeight = CurrentHeight + LockupPeriod │
│    ├─ ShareUnlock {                             │
│    │     Shares: 提款份额数,                    │
│    │     UnlockBlockHeight: 解锁高度            │
│    │   }                                        │
│    └─ 存储到 OwnerShareUnlocks                  │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────┐
│    触发事件                                      │
│    └─ WithdrawFromMegavaultEvent {              │
│          SubaccountId,                          │
│          LockedShares,                          │
│          UnlockBlockHeight                      │
│       }                                         │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
        等待解锁期 (LockupPeriod 个区块)
                    │
                    ▼
┌─────────────────────────────────────────────────┐
│    EndBlocker: ProcessMatureWithdrawals()       │
│    ├─ 遍历所有 ShareUnlocks                     │
│    └─ 检查 CurrentHeight >= UnlockBlockHeight   │
└───────────────────┬─────────────────────────────┘
                    │ (到期)
                    ▼
┌─────────────────────────────────────────────────┐
│    计算提款金额                                  │
│    ├─ ShareValue = VaultEquity / TotalShares    │
│    └─ WithdrawAmount = Shares × ShareValue      │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────┐
│    执行提款                                      │
│    ├─ 转账 USDC 从 Vault 模块账户到用户         │
│    ├─ TotalShares -= Shares                     │
│    ├─ 删除 ShareUnlock 记录                     │
│    └─ 触发 WithdrawalCompletedEvent             │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
            提款完成
```

#### 提款金额计算公式

**提款金额计算**:

数学表达式:
```
WithdrawAmount = WithdrawShares × ShareValue
ShareValue = VaultEquity / TotalShares
```

变量说明:
- **WithdrawAmount**: 提款金额 (USDC)
- **WithdrawShares**: 提款份额数
- **ShareValue**: 解锁时的份额价值 (USDC/份额)
- **VaultEquity**: 解锁时的 Vault 总净值 (USDC)
- **TotalShares**: 解锁时的 Megavault 总份额数

计算示例:
- 假设用户提款 5,000 份额
- 假设解锁时 Vault 总净值 = 1,100,000 USDC
- 假设解锁时 TotalShares = 500,000
- ShareValue = 1,100,000 / 500,000 = 2.2 USDC/份额
- WithdrawAmount = 5,000 × 2.2 = 11,000 USDC

业务解释:
- 提款金额取决于解锁时的份额价值
- 如果 Vault 在解锁期内盈利,用户获得更多
- 如果 Vault 在解锁期内亏损,用户获得更少
- 这是份额机制的核心:公平分配盈亏

### 3.3 订单自动刷新流程

```
        EndBlocker 订单刷新流程

┌─────────────────────────────────────────────────┐
│    EndBlocker: RefreshAllVaultOrders()          │
│    每个区块执行一次                              │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────┐
│    遍历所有 Vaults                               │
│    ├─ 获取所有 VaultParams                      │
│    └─ 过滤 Status = ACTIVE 的 Vaults            │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────┐
│    检查激活条件                                  │
│    ├─ VaultEquity >= ActivationThreshold        │
│    └─ 如果未激活,跳过订单刷新                   │
└───────────────────┬─────────────────────────────┘
                    │ (激活)
                    ▼
┌─────────────────────────────────────────────────┐
│    取消旧订单                                    │
│    ├─ 读取 MostRecentClientIds[VaultId]         │
│    ├─ 遍历所有旧订单 ID                         │
│    └─ 调用 CLOB.CancelOrder() 取消              │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────┐
│    生成新订单 (核心算法)                         │
│    ├─ GetVaultClobOrders(VaultId, QuotingParams)│
│    │   ├─ 遍历所有交易对                        │
│    │   ├─ 获取当前价格 (OraclePrice)            │
│    │   ├─ 计算价差                              │
│    │   │   BidPrice = OraclePrice × (1 - Spread/2) │
│    │   │   AskPrice = OraclePrice × (1 + Spread/2) │
│    │   ├─ 计算订单大小                          │
│    │   │   OrderSize = VaultEquity × PositionSizePct │
│    │   └─ 生成订单对                            │
│    │       BuyOrder {Price: BidPrice, Size}     │
│    │       SellOrder {Price: AskPrice, Size}    │
│    └─ 返回所有订单列表                          │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────┐
│    放置新订单                                    │
│    ├─ 遍历所有新订单                            │
│    ├─ 调用 CLOB.PlaceOrder() 放置               │
│    └─ 收集订单客户端 ID                         │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────┐
│    更新订单跟踪                                  │
│    ├─ MostRecentClientIds[VaultId] = 新订单 IDs │
│    └─ 持久化到 StateStore                       │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
            订单刷新完成
        (Vault 继续提供流动性)
```

#### 订单报价计算公式

**买卖价格计算**:

数学表达式:
```
BidPrice = OraclePrice × (1 - Spread / 2)
AskPrice = OraclePrice × (1 + Spread / 2)
```

变量说明:
- **BidPrice**: 买单价格 (Vault 愿意买入的价格)
- **AskPrice**: 卖单价格 (Vault 愿意卖出的价格)
- **OraclePrice**: 预言机价格 (市场中间价)
- **Spread**: 价差百分比 (例如 0.01 表示 1% 价差)

计算示例:
- 假设 BTC-USD OraclePrice = 50,000 USDC
- 假设 Spread = 0.002 (0.2% 价差)
- BidPrice = 50,000 × (1 - 0.002/2) = 50,000 × 0.999 = 49,950 USDC
- AskPrice = 50,000 × (1 + 0.002/2) = 50,000 × 1.001 = 50,050 USDC

业务解释:
- Vault 在中间价上下对称报价
- 价差越大,做市利润越高,但竞争力越低
- 价差越小,吸引更多交易,但利润越薄

**订单大小计算**:

数学表达式:
```
OrderSize = VaultEquity × PositionSizePct / OraclePrice
```

变量说明:
- **OrderSize**: 订单数量 (BTC 数量)
- **VaultEquity**: Vault 总净值 (USDC)
- **PositionSizePct**: 仓位大小百分比 (例如 0.05 表示 5%)
- **OraclePrice**: 预言机价格 (USDC/BTC)

计算示例:
- 假设 VaultEquity = 1,000,000 USDC
- 假设 PositionSizePct = 0.05 (5%)
- 假设 OraclePrice = 50,000 USDC/BTC
- OrderSize = 1,000,000 × 0.05 / 50,000 = 1 BTC

业务解释:
- 订单大小与 Vault 净值成正比
- PositionSizePct 控制单笔订单风险敞口
- 价格越高,相同资金购买的数量越少

---

## 4. 模块依赖关系

### 4.1 上游依赖 (Vault 依赖哪些模块)

**Subaccounts Keeper**:
- 用途: 获取 Vault 净值 (VaultEquity)
- 调用: `GetNetCollateral(VaultSubaccountId)`
- 作用: 计算份额价值,判断激活条件

**CLOB Keeper**:
- 用途: 放置和取消 Vault 订单
- 调用:
  - `PlaceShortTermOrder()` - 放置 Vault 订单
  - `CancelShortTermOrder()` - 取消旧订单
- 作用: 自动做市,维持流动性

**Perpetuals Keeper**:
- 用途: 获取交易对配置
- 调用: `GetAllPerpetuals()`
- 作用: 确定在哪些市场做市

**Prices Keeper**:
- 用途: 获取预言机价格
- 调用: `GetMarketPrice(marketId)`
- 作用: 计算订单报价

**Bank Keeper**:
- 用途: USDC 转账
- 调用:
  - `SendCoinsFromAccountToModule()` - 存款转账
  - `SendCoinsFromModuleToAccount()` - 提款转账
- 作用: 处理存提款资金流

### 4.2 下游被依赖 (哪些模块依赖 Vault)

**Indexer**:
- 用途: 索引 Vault 事件
- 订阅: `DepositToMegavaultEvent`, `WithdrawFromMegavaultEvent`
- 作用: 提供查询接口

**前端 UI**:
- 用途: 展示 Vault 状态
- 查询:
  - `TotalShares` - 总份额
  - `OwnerShares` - 用户份额
  - `VaultEquity` - Vault 净值
- 作用: 用户界面

### 4.3 依赖关系图

```
        Vault 模块依赖关系

    ┌──────────────┐
    │ Subaccounts  │ ──> GetNetCollateral (计算 VaultEquity)
    └──────────────┘
           ↓
    ┌──────────────┐
    │    Vault     │
    └──────────────┘
           ↓
    ┌──────────────┐
    │     CLOB     │ <─> PlaceOrder/CancelOrder (自动做市)
    └──────────────┘
           ↓
    ┌──────────────┐
    │  Perpetuals  │ ──> GetAllPerpetuals (获取市场列表)
    └──────────────┘
           ↓
    ┌──────────────┐
    │    Prices    │ ──> GetMarketPrice (获取报价)
    └──────────────┘
           ↓
    ┌──────────────┐
    │     Bank     │ ──> SendCoins (存提款转账)
    └──────────────┘
```

---

## 5. 数据流设计

### 5.1 存款数据流

```
用户 USDC → Vault 模块账户 → 铸造份额 → OwnerShares 增加

用户
  │ MsgDepositToMegavault
  ▼
Vault.MsgServer
  │ 验证存款金额
  ▼
Vault.Keeper
  │ 计算份额数
  ▼
Bank.Keeper
  │ 转账 USDC
  ▼
Vault.Keeper
  │ 更新 TotalShares
  │ 更新 OwnerShares
  ▼
StateStore
  │ 持久化份额数据
  ▼
Event Bus
  │ DepositToMegavaultEvent
  ▼
Indexer
  │ 索引存款事件
  ▼
前端 UI
  (展示更新后的份额)
```

### 5.2 提款数据流

```
提款请求 → 锁定份额 → 等待解锁期 → 转账 USDC → 销毁份额

用户
  │ MsgWithdrawFromMegavault
  ▼
Vault.MsgServer
  │ 验证提款份额
  ▼
Vault.Keeper
  │ 减少 OwnerShares (立即)
  │ 创建 ShareUnlock
  ▼
StateStore
  │ 持久化解锁记录
  ▼
EndBlocker (每个区块)
  │ ProcessMatureWithdrawals
  │ 检查解锁是否到期
  ▼
Vault.Keeper (到期时)
  │ 计算提款金额
  ▼
Bank.Keeper
  │ 转账 USDC 给用户
  ▼
Vault.Keeper
  │ 减少 TotalShares
  │ 删除 ShareUnlock
  ▼
Event Bus
  │ WithdrawalCompletedEvent
  ▼
前端 UI
  (展示提款完成)
```

### 5.3 订单刷新数据流

```
EndBlocker 触发 → 取消旧订单 → 生成新订单 → 放置到订单簿

EndBlocker
  │ RefreshAllVaultOrders
  ▼
Vault.Keeper
  │ 遍历所有激活的 Vaults
  ▼
CLOB.Keeper
  │ 取消旧订单 (MostRecentClientIds)
  ▼
Prices.Keeper
  │ 获取预言机价格
  ▼
Vault.Keeper
  │ 计算买卖价格 (BidPrice, AskPrice)
  │ 计算订单大小 (OrderSize)
  ▼
CLOB.Keeper
  │ 放置新订单 (PlaceShortTermOrder)
  ▼
MemClob
  │ 订单进入订单簿
  ▼
Vault.Keeper
  │ 更新 MostRecentClientIds
  ▼
StateStore
  │ 持久化订单跟踪数据
  ▼
订单簿
  (Vault 流动性可用于匹配)
```

---

## 6. 性能与可扩展性

### 6.1 性能优化点

**订单刷新优化**:
- 批量取消旧订单,减少 CLOB 调用次数
- 订单生成算法简单高效,避免复杂计算
- 使用短期订单,自动过期,无需手动清理

**份额计算优化**:
- 份额价值计算复杂度 O(1)
- 使用整数运算,避免浮点数
- 预先计算 VaultEquity,避免重复查询

**存储优化**:
- 份额数据结构简洁,最小化存储空间
- 解锁记录按时间排序,快速查询到期项
- 使用索引优化 OwnerShares 查询

### 6.2 扩展性设计

**多 Vault 支持**:
- VaultParams 按 VaultType 和 VaultNumber 索引
- 支持创建多个独立的 Vaults
- 每个 Vault 有独立的配置和份额

**灵活的报价参数**:
- QuotingParams 可动态调整
- 支持不同市场使用不同策略
- 通过治理或管理员更新

**模块化设计**:
- 份额管理、订单管理、提款管理职责清晰分离
- 便于未来扩展新功能 (如自动复投、动态策略)

### 6.3 瓶颈分析

**潜在瓶颈**:

1. **订单刷新频率**:
   - 每个区块刷新所有 Vault 订单
   - 如果 Vault 数量多,可能消耗大量 Gas
   - 解决方案: 限制 Vault 数量,或分批刷新

2. **VaultEquity 计算**:
   - 需要查询 Subaccounts 获取净值
   - 如果 Vault 持仓复杂,计算开销大
   - 解决方案: 缓存 VaultEquity,定期更新

3. **提款解锁检查**:
   - 每个区块遍历所有 ShareUnlocks
   - 如果解锁记录多,遍历开销大
   - 解决方案: 按 UnlockBlockHeight 索引,只查询到期项

---

## 7. 安全与风险控制

### 7.1 闪电贷攻击防护

**问题**: 攻击者可能通过闪电贷操纵份额价值,低价买入高价卖出。

**防护机制**:

**提款解锁期**:
- 提款需要等待 LockupPeriod 个区块
- 期间份额价值可能变化
- 防止攻击者快速套利

**份额价值锁定**:
- 存款时的份额价值 = 存款时 VaultEquity / TotalShares
- 提款时的份额价值 = 提款时 VaultEquity / TotalShares
- 攻击者无法通过瞬时操纵获利

### 7.2 流动性风险控制

**激活阈值**:
- Vault 只有在 VaultEquity >= ActivationThreshold 时才激活
- 防止小额 Vault 占用系统资源
- 确保 Vault 有足够资金做市

**仓位限制**:
- PositionSizePct 限制单笔订单大小
- 防止过度集中风险
- 分散风险到多个市场

**订单刷新策略**:
- 每个区块取消旧订单,放置新订单
- 避免过期订单堆积
- 确保报价始终贴近市场价

### 7.3 确定性保证

**确定性要求**:

**订单生成确定性**:
- 订单价格和数量计算基于确定性输入 (OraclePrice, VaultEquity)
- 避免使用随机数或时间戳
- 所有节点生成相同订单

**份额计算确定性**:
- 份额数使用整数运算,避免浮点数
- 除法使用向下取整 (Floor Division)
- 所有节点计算相同份额数

**订单放置顺序确定性**:
- 订单按交易对 ID 排序
- 确保所有节点以相同顺序放置订单
- 保证状态一致性

---

## 8. 技术决策记录 (ADR)

### 8.1 为什么使用份额机制?

**问题**: 如何公平分配 Vault 盈亏?

**备选方案**:

1. **方案 A: 直接记录存款金额**
   - 优势: 简单直观
   - 劣势: 无法公平分配盈亏
   - 示例: 用户 A 存 100,用户 B 存 100,Vault 盈利 20。如果直接返还本金,无法分配盈利。

2. **方案 B: 份额制度** ✅ 最终选择
   - 优势: 自动公平分配盈亏
   - 劣势: 计算复杂度稍高
   - 示例:
     - 用户 A 存 100 → 获得 50 份额 (假设初始 ShareValue = 2)
     - 用户 B 存 100 → 获得 50 份额
     - Vault 盈利 20 → VaultEquity = 220, TotalShares = 100, ShareValue = 2.2
     - 用户 A 提款 50 份额 → 获得 50 × 2.2 = 110 (盈利 10)
     - 用户 B 提款 50 份额 → 获得 50 × 2.2 = 110 (盈利 10)

**决策理由**: 份额机制自动公平分配盈亏,无需复杂的会计逻辑。

### 8.2 为什么需要提款解锁期?

**问题**: 提款是否应该立即执行?

**备选方案**:

1. **方案 A: 立即提款**
   - 优势: 用户体验好,资金流动性高
   - 劣势: 容易被闪电贷攻击
   - 攻击场景:
     1. 攻击者借入大量 USDC
     2. 存入 Vault,获得份额
     3. 操纵市场价格,使 Vault 盈利
     4. 立即提款,获得高价值份额
     5. 归还闪电贷,获利

2. **方案 B: 延迟提款 (解锁期)** ✅ 最终选择
   - 优势: 防止闪电贷攻击
   - 劣势: 用户体验稍差,资金锁定
   - 防护机制:
     - 攻击者无法在单个区块内完成存款→操纵→提款
     - 解锁期内份额价值可能变化,无法保证盈利

**决策理由**: 安全性优先,防止闪电贷攻击比用户体验更重要。

### 8.3 为什么每个区块刷新订单?

**问题**: 订单刷新频率应该多高?

**备选方案**:

1. **方案 A: 固定订单,不刷新**
   - 优势: 节省 Gas,减少计算
   - 劣势: 价格漂移后订单失效,流动性断层

2. **方案 B: 每个区块刷新** ✅ 最终选择
   - 优势: 订单始终贴近市场价,流动性稳定
   - 劣势: 消耗更多 Gas
   - 权衡: Vault 是系统核心流动性来源,优先保证流动性

3. **方案 C: 价格变化超过阈值时刷新**
   - 优势: 平衡 Gas 和流动性
   - 劣势: 实现复杂,需要监控价格变化

**决策理由**: 流动性是 DEX 的核心,每个区块刷新确保 Vault 始终提供可用流动性。

### 8.4 为什么使用 Megavault 单一池?

**问题**: 是否应该为每个交易对创建独立 Vault?

**备选方案**:

1. **方案 A: 每个交易对独立 Vault**
   - 优势: 风险隔离,用户可选择特定市场
   - 劣势: 流动性分散,管理复杂

2. **方案 B: Megavault 单一池** ✅ 最终选择
   - 优势:
     - 聚合流动性,提高资金利用率
     - 简化用户体验,无需选择市场
     - 风险分散到多个市场
   - 劣势: 单个市场风险可能影响整体

**决策理由**: 聚合流动性提高资金效率,风险分散到多个市场更安全。

---

## 9. 集成与扩展点

### 9.1 集成点

**与 CLOB 模块集成**:
- Vault 使用 CLOB 接口放置和取消订单
- Vault 订单与用户订单竞争匹配
- Vault 作为被动做市商,提供基础流动性

**与 Subaccounts 模块集成**:
- Vault 有独立的 Subaccount,存储资金和持仓
- GetNetCollateral 计算 Vault 净值
- Vault 盈亏反映在 Subaccount 余额中

**与 Prices 模块集成**:
- Vault 依赖预言机价格计算订单报价
- 价格更新触发订单刷新
- 确保 Vault 报价贴近市场

### 9.2 扩展点

**自定义做市策略**:
- 当前: 固定价差 (Spread)
- 扩展: 动态价差 (根据波动率调整)
- 实现: 修改 GetVaultClobOrders 逻辑

**多策略 Vault**:
- 当前: 所有 Vault 使用相同策略
- 扩展: 不同 Vault 使用不同策略 (保守型、激进型)
- 实现: 增加策略参数到 VaultParams

**自动复投**:
- 当前: 盈利保留在 Vault,增加份额价值
- 扩展: 定期分配盈利给持有者
- 实现: 增加分红逻辑到 EndBlocker

---

## 10. 总结

### 10.1 关键设计亮点

1. **份额机制**: 公平分配盈亏,自动计算价值
2. **提款解锁期**: 防止闪电贷攻击,保证安全性
3. **自动做市**: EndBlocker 每个区块刷新订单,维持流动性
4. **Megavault 聚合**: 单一流动性池,提高资金利用率

### 10.2 设计权衡

| 设计选择              | 优势                 | 劣势                 | 决策理由             |
|---------------------|---------------------|---------------------|---------------------|
| 份额机制             | 公平分配盈亏         | 计算复杂度高         | 公平性优先           |
| 提款解锁期           | 防止闪电贷攻击       | 用户体验稍差         | 安全性优先           |
| 每区块刷新订单       | 流动性稳定           | Gas 消耗高           | 流动性是核心         |
| Megavault 单一池    | 聚合流动性,高效      | 风险不隔离           | 资金效率优先         |

### 10.3 核心流程总结

**存款**: 转账 USDC → 计算份额 → 铸造份额 → 更新 OwnerShares

**提款**: 锁定份额 → 创建解锁记录 → 等待解锁期 → 转账 USDC → 销毁份额

**订单刷新**: 取消旧订单 → 获取价格 → 计算报价 → 放置新订单 → 更新跟踪

---

**文档版本**: v1.0
**最后更新**: 2025-12-31
**文档作者**: Claude Sonnet 4.5
**文档状态**: ✅ 完成
