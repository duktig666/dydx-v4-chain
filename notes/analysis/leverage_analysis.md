# Leverage 模块深度分析报告

**生成时间**: 2025-12-31
**分析范围**: Leverage 模块完整功能探索
**代码版本**: 基于当前 clob 分支

---

## 一、执行摘要

本报告基于深度代码探索,识别出 Leverage 模块中 **8 个主要类别的未文档化功能**,其中包括:

- **3 个高优先级 (P0)** 遗漏内容:需要立即补充到 PRD
- **2 个中优先级 (P1)** 遗漏内容:建议补充
- **3 个低优先级 (P2)** 遗漏内容:可选补充

**关键发现**:
1. ✅ **已完成**: IMF 精度与 PPM 表示、多永续合约配置、流动性层级限制已补充到 PRD
2. ⏳ **待补充**: OI 缩放机制(最重要的遗漏内容)
3. ⏳ **待补充**: 完整的保证金计算流程
4. ⏳ **待补充**: 错误码和存储设计

---

## 二、未文档化功能详细分析

### 2.1 IMF (初始保证金率) 精度与表示 ⚠️ **高重要性** ✅ 已补充

**代码位置**: `/Users/renshiwei/code/company/DEX/hermes/protocol/x/clob/types/leverage.go:31-36`

**发现内容**: IMF 用 PPM (Parts Per Million) 表示,验证范围: (0, 1_000_000]

**代码引用**:
```go
if entry.CustomImfPpm == 0 || entry.CustomImfPpm > 1_000_000 {
    return errorsmod.Wrap(
        ErrInvalidLeverage,
        fmt.Sprintf("imf ppm for clob pair %d must be between (0, 1,000,000]", entry.ClobPairId),
    )
}
```

**PRD 状态**:
- 原 PRD: 仅提及"杠杆倍数",未说明内部表示
- 已补充: FR-1.2 详细说明 PPM 格式、转换公式、精度优势

**业务影响**:
- 用户视角: 设置"杠杆倍数"(如 20x)
- 系统内部: 转换为 `custom_imf_ppm` (如 50,000 ppm)
- 精度: 支持 6 位小数,避免浮点数误差

**补充内容**:
- 转换公式: `custom_imf_ppm = 1,000,000 / Leverage`
- 转换示例表格 (20x = 50,000 ppm, 10x = 100,000 ppm, 等)
- 精度优势说明
- API 层面对用户友好

---

### 2.2 OI (未平仓合约量) 缩放机制 ⚠️ **高重要性** ⏳ 待补充

**代码位置**: `/Users/renshiwei/code/company/DEX/hermes/protocol/x/perpetuals/types/liquidity_tier.go:77-147`

**发现内容**: 用户设置的杠杆只是**最小值**,实际保证金要求会根据市场未平仓合约量 (OI) 动态调整,**可能远高于用户设置**。

**核心算法**:
```go
func (liquidityTier LiquidityTier) GetInitialMarginQuoteQuantums(
    quoteQuantums *big.Int,        // 仓位名义价值
    oiQuoteQuantums *big.Int,      // 市场 OI (未平仓合约量)
    custom_imf_ppm *big.Int,       // 用户设置的 IMF (PPM)
) *big.Int {
    // 第 1 步: 根据 OI 计算调整后的 IMF
    totalImfPpm := liquidityTier.GetAdjustedInitialMarginPpm(oiQuoteQuantums)

    // 第 2 步: 取 OI 调整 IMF 和用户自定义 IMF 的最大值
    if custom_imf_ppm.Sign() > 0 {
        totalImfPpm = lib.BigMax(totalImfPpm, custom_imf_ppm)
    }

    // 第 3 步: 计算最终保证金要求
    return lib.BigMulPpm(quoteQuantums, totalImfPpm, true)
}
```

**OI 调整 IMF 的计算逻辑**:
```go
func (liquidityTier LiquidityTier) GetAdjustedInitialMarginPpm(
    oiQuoteQuantums *big.Int,
) *big.Int {
    baseImfPpm := big.NewInt(int64(liquidityTier.InitialMarginPpm))

    // 如果 OI <= LowerCap,返回基础 IMF
    if oiQuoteQuantums.Cmp(liquidityTier.OpenInterestLowerCap) <= 0 {
        return baseImfPpm
    }

    // 如果 OI >= UpperCap,返回最大 IMF (100% = 1,000,000 ppm)
    if oiQuoteQuantums.Cmp(liquidityTier.OpenInterestUpperCap) >= 0 {
        return big.NewInt(lib.OneMillion)
    }

    // 线性插值计算
    oiDelta := new(big.Int).Sub(oiQuoteQuantums, liquidityTier.OpenInterestLowerCap)
    capDelta := new(big.Int).Sub(liquidityTier.OpenInterestUpperCap, liquidityTier.OpenInterestLowerCap)
    imfDelta := new(big.Int).Sub(big.NewInt(lib.OneMillion), baseImfPpm)

    // adjustedIMF = baseIMF + (oiDelta / capDelta) * (1,000,000 - baseIMF)
    adjustment := new(big.Int).Mul(oiDelta, imfDelta)
    adjustment.Quo(adjustment, capDelta)
    return new(big.Int).Add(baseImfPpm, adjustment)
}
```

**完整计算公式**:
```
# 第 1 步: 计算 OI 调整后的 IMF
if OI <= LowerCap:
    OI_Adjusted_IMF = Base_IMF
elif OI >= UpperCap:
    OI_Adjusted_IMF = 1,000,000 ppm (100%)
else:
    # 线性插值
    OI_Adjusted_IMF = Base_IMF + ((OI - LowerCap) / (UpperCap - LowerCap)) × (1,000,000 - Base_IMF)

# 第 2 步: 取最大值
Effective_IMF = max(OI_Adjusted_IMF, Custom_IMF)

# 第 3 步: 计算保证金要求
Margin_Requirement = Position_Value × (Effective_IMF / 1,000,000)
```

**实际案例分析**:

**场景 1: 低 OI,用户设置生效**
- 流动性层级配置:
  - Base_IMF = 20,000 ppm (50x 杠杆)
  - LowerCap = 100,000,000 USDC
  - UpperCap = 1,000,000,000 USDC
- 市场状态: OI = 50,000,000 USDC (低于 LowerCap)
- 用户设置: 10x 杠杆 (custom_imf_ppm = 100,000)
- 计算:
  - OI_Adjusted_IMF = 20,000 (OI 低于 LowerCap,使用 Base_IMF)
  - Effective_IMF = max(20,000, 100,000) = 100,000
  - 实际杠杆 = 1,000,000 / 100,000 = **10x** ✅
- 结果: **用户获得预期的 10x 杠杆**

**场景 2: 中等 OI,部分缩放**
- 流动性层级配置: 同上
- 市场状态: OI = 550,000,000 USDC (处于 LowerCap 和 UpperCap 之间)
- 用户设置: 10x 杠杆 (custom_imf_ppm = 100,000)
- 计算:
  - OI_Adjusted_IMF = 20,000 + ((550M - 100M) / (1000M - 100M)) × (1,000,000 - 20,000)
  - OI_Adjusted_IMF = 20,000 + (450M / 900M) × 980,000
  - OI_Adjusted_IMF = 20,000 + 0.5 × 980,000
  - OI_Adjusted_IMF = 20,000 + 490,000 = **510,000 ppm**
  - Effective_IMF = max(510,000, 100,000) = **510,000**
  - 实际杠杆 = 1,000,000 / 510,000 ≈ **1.96x** ❌
- 结果: **用户设置 10x,实际只有 1.96x 杠杆** (OI 缩放导致)

**场景 3: 高 OI,强制最大保证金**
- 流动性层级配置: 同上
- 市场状态: OI = 1,200,000,000 USDC (高于 UpperCap)
- 用户设置: 10x 杠杆 (custom_imf_ppm = 100,000)
- 计算:
  - OI_Adjusted_IMF = 1,000,000 (OI 超过 UpperCap,100% 保证金)
  - Effective_IMF = max(1,000,000, 100,000) = **1,000,000**
  - 实际杠杆 = 1,000,000 / 1,000,000 = **1x** ❌
- 结果: **用户设置 10x,实际只有 1x 杠杆 (无杠杆)** (OI 过高,系统强制降杠杆)

**PRD 状态**:
- 原 PRD: **完全未提及 OI 缩放机制**,仅说明 `IMR = 1 / Leverage`
- 严重问题: 用户期望 10x 杠杆,实际可能只有 1.96x 甚至 1x
- 用户体验影响: **极大**,用户会认为系统"欺骗"或"故障"

**必须补充的内容**:
1. **FR-3.2: OI 缩放机制**
   - OI 是什么 (未平仓合约量)
   - 为什么需要 OI 缩放 (风险管理)
   - OI 缩放如何影响保证金 (完整公式)
   - 实际案例说明 (低/中/高 OI)
2. **用户场景: OI 缩放导致实际杠杆降低**
   - 用户设置 10x,但市场 OI 高,实际只有 2x
   - 用户需要理解这是正常的风险管理机制
3. **查询接口: 查询实际有效杠杆**
   - 用户设置的杠杆 vs 当前有效杠杆
   - 当前市场 OI 和调整系数

**业务价值**:
- **透明化**: 用户理解为什么实际杠杆低于设置值
- **风险管理**: 系统根据市场风险动态调整杠杆
- **用户信任**: 明确说明机制,避免误解

---

### 2.3 流动性层级最大杠杆限制 ⚠️ **高重要性** ✅ 已补充

**代码位置**: `/Users/renshiwei/code/company/DEX/hermes/protocol/x/subaccounts/keeper/leverage.go:155-169`

**发现内容**: 每个永续合约通过流动性层级定义最大杠杆,用户设置不能超过此限制。

**代码引用**:
```go
func (k Keeper) GetMinImfForPerpetual(ctx sdk.Context, perpetualId uint32) (uint32, error) {
    _, _, liquidityTier, err := k.perpetualsKeeper.GetPerpetualAndMarketPriceAndLiquidityTier(ctx, perpetualId)
    if err != nil {
        return 0, err
    }

    if liquidityTier.InitialMarginPpm == 0 {
        return 0, types.ErrInitialMarginPpmIsZero
    }

    return liquidityTier.InitialMarginPpm, nil
}
```

**PRD 状态**:
- 原 PRD: 提及"最大杠杆限制 (通过流动性层级配置,例如 100x)",但未详细说明
- 已补充: FR-2.2 详细说明流动性层级配置、验证逻辑、不同合约示例

**补充内容**:
- 流动性层级配置示例表格
- 验证逻辑 (custom_imf_ppm >= InitialMarginPpm)
- 不同合约的最大杠杆 (BTC-USD 50x, ETH-USD 20x, 小币种 10x)
- 用户场景: 杠杆超过限制被拒绝
- 错误码说明

---

### 2.4 多永续合约杠杆配置 ⚠️ **中重要性** ✅ 已补充

**代码位置**: `/Users/renshiwei/code/company/DEX/hermes/protocol/x/clob/types/leverage.go:58-75`

**发现内容**: 一个子账户可以为不同永续合约设置不同杠杆,支持增量更新。

**代码引用**:
```go
func ValidateAndConstructPerpetualLeverageMap(
    ctx sdk.Context,
    msg *MsgUpdateLeverage,
    clobKeeper ClobKeeper,
) (map[uint32]uint32, error) {
    perpetualLeverageMap := make(map[uint32]uint32)
    for _, entry := range msg.ClobPairLeverage {
        clob, _ := clobKeeper.GetClobPair(ctx, ClobPairId(entry.ClobPairId))
        perpetualId := clob.MustGetPerpetualId()
        perpetualLeverageMap[perpetualId] = entry.CustomImfPpm
    }
    return perpetualLeverageMap, nil
}
```

**PRD 状态**:
- 原 PRD: 未提及多合约配置
- 已补充: FR-1.3 详细说明 API 格式、内部存储、增量更新逻辑

**补充内容**:
- API 请求格式示例
- 内部存储结构
- 增量更新逻辑 (3 个场景)
- 用户场景: 差异化杠杆策略

---

### 2.5 错误类型与消息 ⚠️ **中重要性** ⏳ 待补充

**代码位置**: `/Users/renshiwei/code/company/DEX/hermes/protocol/x/clob/types/errors.go`

**发现内容**: 定义了多种杠杆相关错误,但 PRD 未说明。

**主要错误码**:

| 错误码 | 定义 | 触发场景 |
|-------|------|---------|
| `ErrInvalidLeverage` | 杠杆设置无效 | 杠杆超出流动性层级限制,或 custom_imf_ppm 不在 (0, 1,000,000] |
| `ErrInitialMarginPpmIsZero` | InitialMarginPpm 为 0 | 流动性层级配置错误 |

**PRD 状态**:
- 原 PRD: 未提及错误码
- 已部分补充: FR-2.2 添加了错误码表格

**建议补充**:
- 在"非功能需求"或附录中添加完整错误码参考
- 每个错误码的触发场景、用户消息、恢复建议

---

### 2.6 存储设计 ⚠️ **低重要性** ⏳ 待补充

**代码位置**: `/Users/renshiwei/code/company/DEX/hermes/protocol/x/subaccounts/types/keys.go:30`

**发现内容**: Leverage 数据存储在 Subaccounts 模块,而非 CLOB 模块。

**存储键设计**:
```go
const LeverageKeyPrefix = "Lev:"

// 存储键: "Lev:" + SubaccountId.ToStateKey()
// 存储值: Protobuf 编码的 LeverageData
```

**数据结构**:
```protobuf
message LeverageData {
    map<uint32, uint32> perpetual_leverage = 1;  // perpetual_id -> custom_imf_ppm
}
```

**PRD 状态**:
- 原 PRD: 未提及存储设计
- 这是技术细节,PRD 不需要详细说明,但可以在"参考资料"中链接数据结构文档

**建议**:
- 在 PRD 中保持简洁,仅说明"数据存储在链上,持久化"
- 详细的存储设计放在数据结构文档中

---

### 2.7 CLI 命令 ⚠️ **低重要性** ⏳ 待补充

**代码位置**: `/Users/renshiwei/code/company/DEX/hermes/protocol/x/clob/client/cli/tx.go`

**发现内容**: 提供了 CLI 命令用于设置杠杆。

**命令**:
```bash
hermesd tx clob update-leverage [subaccount_id] [leverage_entries_json] [flags]
```

**示例**:
```bash
hermesd tx clob update-leverage \
  "alice/0" \
  '[{"clob_pair_id": 0, "custom_imf_ppm": 50000}]' \
  --from alice
```

**PRD 状态**:
- 原 PRD: 未提及 CLI
- PRD 主要面向产品和业务,CLI 是开发工具,可选补充

**建议**:
- 在附录中添加 CLI 使用示例,便于开发者和高级用户

---

### 2.8 测试覆盖分析 ⚠️ **低重要性**

**主要测试文件**:
1. `/Users/renshiwei/code/company/DEX/hermes/protocol/x/clob/types/leverage_test.go`
   - 测试 `ValidateAndConstructPerpetualLeverageMap` 函数
   - 边界值测试 (custom_imf_ppm = 0, > 1,000,000)
   - clob_pair_id 与 perpetual_id 转换测试

2. `/Users/renshiwei/code/company/DEX/hermes/protocol/x/perpetuals/types/liquidity_tier_test.go`
   - 测试 OI 缩放机制 (`GetAdjustedInitialMarginPpm`)
   - 测试线性插值计算
   - 测试边界条件 (OI <= LowerCap, OI >= UpperCap)

3. `/Users/renshiwei/code/company/DEX/hermes/protocol/x/subaccounts/keeper/leverage_test.go`
   - 测试杠杆存储和查询
   - 测试流动性层级限制验证

**关键测试场景**:
- ✅ IMF PPM 范围验证
- ✅ 流动性层级最大杠杆限制
- ✅ OI 缩放线性插值
- ✅ 多永续合约配置
- ❌ **缺失**: OI 缩放对实际用户体验的影响测试 (用户设置 10x,实际只有 2x)

**PRD 状态**:
- PRD 不需要包含测试细节
- 测试用例用于验证 PRD 需求的正确实现

---

## 三、PRD 更新摘要

### 已完成更新 ✅

1. **FR-1.2: IMF 精度与表示**
   - 位置: FR-1 之后
   - 内容: PPM 格式说明、转换公式、转换示例表格、精度优势、业务规则、用户影响

2. **FR-1.3: 多永续合约杠杆配置**
   - 位置: FR-1.2 之后
   - 内容: API 格式、内部存储、增量更新逻辑、用户场景

3. **FR-2.2: 流动性层级最大杠杆限制**
   - 位置: FR-2.1 之后
   - 内容: 流动性层级配置示例、验证逻辑、不同合约最大杠杆、用户场景、错误码

### 待完成更新 ⏳

**P0 (高优先级)**:

1. **FR-3.2: OI 缩放机制** (最重要!)
   - 位置: FR-3.1 之后
   - 建议章节结构:
     ```markdown
     #### FR-3.2: OI 缩放机制 (Open Interest Scaling)

     **需求描述**: 系统根据市场未平仓合约量 (OI) 动态调整保证金要求,实际杠杆可能低于用户设置。

     **什么是 OI (未平仓合约量)**:
     - OI = 市场上所有多头或空头仓位的总和
     - OI 越高,市场风险越大
     - 高 OI 时系统自动提高保证金要求,降低杠杆

     **OI 缩放的原因**:
     - 保护系统免受市场单边持仓风险
     - 防止高 OI 导致系统性清算
     - 动态风险管理,适应市场状态

     **OI 缩放计算公式**:

     [完整公式和计算步骤]

     **实际案例**:

     [场景 1: 低 OI,用户获得预期杠杆]
     [场景 2: 中等 OI,杠杆部分降低]
     [场景 3: 高 OI,杠杆大幅降低]

     **用户影响**:
     - 用户设置的杠杆是最小值,不是最终值
     - 实际杠杆 = max(用户设置, OI 调整后要求)
     - 查询接口可以查看当前实际杠杆

     **验收标准**:
     - OI 缩放计算正确
     - 用户理解实际杠杆可能低于设置
     - 查询接口返回实际有效杠杆
     ```

2. **更新 FR-3.1: 订单保证金验证**
   - 补充: 保证金计算需要考虑 OI 缩放,而非简单的 `1 / Leverage`
   - 更新公式:
     ```
     Effective_IMF = max(OI_Adjusted_IMF, Custom_IMF)
     OrderMarginRequirement = OrderValue × (Effective_IMF / 1,000,000)
     ```

3. **添加用户场景: OI 缩放影响杠杆**
   - 场景: 用户 Henry 设置 10x 杠杆,但市场 OI 高,实际只有 2x
   - 展示完整计算过程
   - 说明这是正常的风险管理机制

**P1 (中优先级)**:

4. **附录: 错误码参考**
   - 完整的错误码列表
   - 每个错误的触发场景和恢复建议

5. **附录: CLI 使用示例**
   - `update-leverage` 命令示例
   - 参数说明

**P2 (低优先级)**:

6. **参考资料: 数据结构文档链接**
   - 链接到详细的存储设计文档 (待创建)

---

## 四、关键业务逻辑澄清

### 保证金计算的完整流程

**原 PRD 描述** (简化版,不完整):
```
IMR = 1 / Leverage
MarginRequirement = PositionValue × IMR
```

**实际实现** (完整版):
```
# 第 1 步: 获取流动性层级最小 IMF
Base_IMF = LiquidityTier.InitialMarginPpm

# 第 2 步: 根据市场 OI 计算调整后 IMF
if OI <= LowerCap:
    OI_Adjusted_IMF = Base_IMF
elif OI >= UpperCap:
    OI_Adjusted_IMF = 1,000,000 ppm (100%)
else:
    OI_Adjusted_IMF = Base_IMF + ((OI - LowerCap) / (UpperCap - LowerCap)) × (1,000,000 - Base_IMF)

# 第 3 步: 获取用户设置的自定义 IMF
Custom_IMF = user_leverage_settings.custom_imf_ppm

# 第 4 步: 取三者的最大值
Effective_IMF = max(Base_IMF, OI_Adjusted_IMF, Custom_IMF)

# 第 5 步: 计算最终保证金要求
MarginRequirement = PositionValue × (Effective_IMF / 1,000,000)
```

**关键点**:
1. 用户设置的杠杆**不是最终杠杆**,只是一个输入参数
2. 实际保证金要求取决于**三个因素**的最大值:
   - 流动性层级基础 IMF (合约自身风险)
   - OI 调整后 IMF (市场风险)
   - 用户自定义 IMF (用户风险偏好)
3. 系统总是选择**最保守的保证金要求**,确保安全

**用户理解误区**:
- ❌ 误区: "我设置 10x 杠杆,就能用 10x 杠杆交易"
- ✅ 正确: "我设置 10x 杠杆,系统会根据市场情况,可能给我 10x,也可能只有 2x"

---

## 五、数据结构补充建议

### Leverage 数据存储

**存储位置**: Subaccounts 模块 (非 CLOB 模块)

**存储键**:
```
"Lev:" + SubaccountId.ToStateKey()
```

**存储值** (Protobuf):
```protobuf
message LeverageData {
    map<uint32, uint32> perpetual_leverage = 1;  // perpetual_id -> custom_imf_ppm
}
```

**示例**:
```
Key: "Lev:alice/0"
Value: {
  perpetual_leverage: {
    0: 50000,    // BTC-USD: 20x 杠杆
    1: 100000,   // ETH-USD: 10x 杠杆
  }
}
```

**访问方法**:
```go
// 读取
leverageData := keeper.GetLeverage(ctx, subaccountId)

// 写入
keeper.SetLeverage(ctx, subaccountId, leverageData)
```

---

## 六、建议的文档改进优先级

### Phase 1: 必须补充 (P0)
1. ✅ IMF 精度与表示 (已完成)
2. ✅ 多永续合约配置 (已完成)
3. ✅ 流动性层级限制 (已完成)
4. ⏳ **OI 缩放机制** (最重要,严重影响用户体验)
5. ⏳ 更新 FR-3.1 保证金计算公式

### Phase 2: 建议补充 (P1)
6. ⏳ 错误码参考
7. ⏳ CLI 使用示例
8. ⏳ 用户场景: OI 缩放影响

### Phase 3: 可选补充 (P2)
9. ⏳ 数据结构文档链接
10. ⏳ 测试覆盖说明

---

## 七、下一步行动

### 立即行动 (暂停)
- ✅ 已将当前分析写入文件: `/Users/renshiwei/code/company/DEX/hermes/notes/analysis/leverage_analysis.md`

### 待用户确认后继续
1. 补充 **FR-3.2: OI 缩放机制** (最重要的遗漏内容)
2. 更新 **FR-3.1** 的保证金计算公式
3. 添加**用户场景: OI 缩放影响实际杠杆**
4. 补充错误码参考和 CLI 示例
5. Review 完整 PRD,确保一致性

---

## 八、总结

Leverage 模块的核心复杂性在于**保证金计算的多层逻辑**:

1. **用户层**: 用户设置"杠杆倍数"(如 10x),简单直观
2. **API 层**: 转换为 `custom_imf_ppm` (如 100,000 ppm),保证精度
3. **流动性层级层**: 验证不超过合约最大杠杆 (如 BTC-USD 最大 50x)
4. **OI 缩放层**: 根据市场 OI 动态调整保证金 (如高 OI 时强制降杠杆)
5. **最终计算层**: 取所有约束的最大值,确保系统安全

**关键建议**:
- 原 PRD 将复杂性简化为 `IMR = 1 / Leverage`,这**严重低估**了系统的复杂性
- 必须补充 **OI 缩放机制**,否则用户会对"设置 10x 却只有 2x"感到困惑和不满
- 透明化机制,提高用户信任度

---

**文档状态**: ✅ 全部完成
**完成时间**: 2025-12-31

## 九、最终更新摘要

### 已完成的 PRD 更新 ✅

**P0 (高优先级) - 全部完成**:
1. ✅ FR-1.2: IMF 精度与 PPM 表示
2. ✅ FR-1.3: 多永续合约杠杆配置
3. ✅ FR-2.2: 流动性层级最大杠杆限制
4. ✅ FR-3.2: OI 缩放机制 (最重要补充)
5. ✅ 更新 FR-3.1: 完整保证金计算公式

**P1 (中优先级) - 全部完成**:
6. ✅ 附录 7.1: 错误码参考
7. ✅ 附录 7.2: CLI 使用示例
8. ✅ 扩展术语表 (新增 IMF, PPM, OI 等 8 个术语)
9. ✅ 用户场景: OI 缩放影响实际杠杆

**文档质量**:
- 从 v1.0 (479 行) 升级到 v2.0 (1086 行)
- 新增内容 600+ 行
- 完整覆盖 OI 缩放机制
- 提供 3 个详细的 OI 缩放案例
- 补充完整的公式和计算步骤
- 添加错误码和 CLI 使用指南

**关键改进**:
1. **透明化 OI 缩放**: 用户现在理解为什么设置 10x 只有 2x
2. **精度说明**: PPM 格式保证计算精度
3. **多合约支持**: 差异化杠杆策略
4. **流动性层级限制**: 不同合约的最大杠杆
5. **完整公式**: 三因素取最大值的保证金计算

**下一步**: 继续审查其他模块 (Perpetuals, Listing, Prices, Stats, Vault)
