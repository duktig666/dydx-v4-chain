## DEX 执行层落地（单节点优先）文档索引

### 目标与边界

- **目标**：用 Rust 落地一个**高性能、低延迟**的 DEX 执行层（一期先做单节点），能力对标 HyperLiquid 的“用户体验层指标”（低延迟确认、稳定撮合、完备风控与清算/ADL），并为二期引入共识（Sui DAG / ZK）预留清晰接口。
- **边界**：本目录仅做**调研、架构设计、方案设计**，不涉及代码开发。

### 输出文件

- **调研**：`mynotes/dex/exec_layer/01_research.md`
- **架构设计**：`mynotes/dex/exec_layer/02_architecture.md`
- **方案设计**：`mynotes/dex/exec_layer/03_solution_design.md`

### 复用/引用的既有材料（本仓库）

- `mynotes/dex/prd/DEX完整业务需求.md`：业务需求基线（账户/资产/风控/撮合/资金费率/清算/费用/Vault 等）
- `mynotes/dex/arch/sui_dex_arch.md`：偏“基于 Sui Fork + Sequencer/Validator”的全链架构草案（可作为二期参考）
- `mynotes/dex/tech/sui_dex_tech.md`：偏实现细节的技术方案草案（可作为二期参考）
- `mynotes/dex/result/use_sui_result.md`：复用 Sui 的结论与风险点（其中“Precompile 不易被识别”的问题需要在本次方案中重新分类与处理）
- `mynotes/dex/analyst/liquidations/hyperliquid_comparison.md`：HyperLiquid 清算/ADL 风险瀑布机制对标
- `mynotes/dex/data_structure/clob.md`、`mynotes/dex/data_structure/perpetuals.md`：CLOB / Perp 数据模型与参数体系参考


