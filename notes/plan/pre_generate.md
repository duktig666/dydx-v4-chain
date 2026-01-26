### **工程师A - 交易系统负责人**

**职责范围**：负责整个交易执行系统，从订单管理到撮合成交的完整流程

### **主要负责模块**

**1. CLOB（中央限价订单簿）**
路径：`/v4-chain/protocol/x/clob/`

* 订单匹配引擎
* 订单生命周期管理
* 内存订单簿（memclob）
* MEV保护机制
* 订单速率限制

**2. Perpetuals（永续合约）**
路径：`/v4-chain/protocol/x/perpetuals/`

* 永续合约定义和管理
* 资金费率计算
* 市场参数管理
* 流动性层级

**3. Listing（市场上市）**
路径：`/v4-chain/protocol/x/listing/`

* 新交易对上市流程
* 市场参数初始化

**4. Prices（价格预言机）**
路径：`/v4-chain/protocol/x/prices/`

* 价格数据接入
* 价格验证和聚合
* 与Slinky预言机集成

**5. Stats（统计）**
路径：`/v4-chain/protocol/x/stats/`

* 交易量统计
* 用户交易数据追踪

**6. Leverage（杠杆）**
路径：`/v4-chain/protocol/x/leverage/`

* 杠杆倍数管理
* 风险参数设置
* 与永续合约的杠杆集成

**7. Vault（金库）**
路径：`/v4-chain/protocol/x/vault/`

* 流动性池管理
* 自动做市功能
* PnL分配机制

**工作重点**：构建高性能、公平、可靠的交易系统

```rust
上述是我要负责的模块，以下是我的需求：
1. 使用architect 先定位以上模块，写出架构和系统设计，并在notes下合适位置生成文档
2. 使用analyst 根据架构和系统设计文档，以及备注文件并结合生成项目代码，按照上述分模块生成产品需求文档，并在notes下合适位置生成文档。要求：
① 每个模块下根据代码和测试用例，生成对应的需求，写入文档
② 需求尽可能简介明了，直切重点
③ 需求文档中不要包含任何代码，设想你只是一个产品经理，不懂代码
④ 需求文档中的需求牵扯到计算公式时，需要进行罗列，并解释如何计算
3. 按模块分类，根据不同的存储类型生成数据结构文档。要求：
① 将上述模块中的数据结构定义，分模块分存储类型 梳理到一个文档中
② 重要的数据结构做简介明了的介绍
③ 不同的存储类型有：StateStore链上持久化存储、Mem Store内存存储、Transient Store瞬态存储（当前区块有效）、MemClob (纯内存 - 不持久化)
④ 重点中的重点，最重要的是关注StateStore链上持久化存储

参考文件：
以下文件之前已经做了大部分的详细设计分析，可以作为参考：
1. notes/business/clob_analyst.md clob模块详细分析
2. notes/business/perpetuals_analyst.md perpetuals模块详细分析
3. notes/business/slinky_oracle.md 价格预言机分析

先将要实现的需求进行plan，并将plan写入到notes下合适的位置，后续等我review后再根据plan进行分析
```
