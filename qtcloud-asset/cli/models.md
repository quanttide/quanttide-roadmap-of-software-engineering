# qtcloud-asset CLI 模型与工程标准对照

## 一句话概括

从 CLI 实际代码出发，把每个模型概念对照数字资产治理标准（quanttide-asset `docs/specification`）：部分概念已对齐，多数需要改名、补字段或重构。

## 概念

### ContractSchema

`src/cli/src/contract.rs` 的契约模型（assets/skills/validation）对应标准 `contract/index.md` 的契约文件。契约 YAML 与标准同为 `.quanttide/asset/contract.yaml`，位置一致。

| 代码字段 | 标准字段 | 关系 |
|:--|:--|:--|
| assets | schemas | 改名：语义一致，字段名不符 |
| skills | skills | 一致 |
| validation | validation | 一致，但策略未下沉到契约文件 |
| — | spec_version | 缺口：标准必选，代码无 |
| — | version | 缺口：标准推荐，代码无 |
| — | workflows | 缺口：标准推荐，代码无 |
| — | discovery | 缺口：标准推荐，代码无（桶清单硬编码） |
| — | registry | 缺口：标准推荐，代码无 |

### AssetConfig

`assets` 条目（type/provider/metadata）对应标准 `contract/asset.md` 的资产字段（name/title/description/type/category/path）。type 一致；缺 title/category/path；provider/metadata 是自用扩展字段。

### SkillConfig

（version/entrypoint/params）对应标准 `contract/skill.md`（name/title/description/type/domain/commands）。语义相关但字段风格不同：标准的 `type`（rule/bridge/agent）与 `domain` 代码没有，代码的 version/entrypoint/params 标准没有。

### Policy

`contract.rs` 的 `Policy{selector, mode, required_categories}` 对应标准 `contract/validation.md` 的策略集。selector 与 mode（ATOMIC/SCOPED）完全一致——模型层已对齐；required_categories 是自用扩展；缺口是策略写在代码里，未随契约文件下沉。

### ValidationResult

对应标准 `lifecycle.md` 的验证结果。代码是 passed/failed 二元；标准是 ALIGNED（物理与契约一致）/ DISCOVERED（物理存在而契约未定义）/ DRIFTED（契约定义而物理不存在）三态。DISCOVERED 语义代码无法表达，是主要缺口。

### scan 与 guess_asset_type

对应标准发现行为（`government/discover.md` 五步：加载→剪枝→采样→映射→发射）与 `contract/discovery.md` 配置。scan 是发现行为的实现，但 `guess_asset_type` 是按目录名硬编码猜测的 Map 步骤，标准要求 `maps` 规则配置驱动，且缺 `excludes` 剪枝配置。

### file_op / archive

对应标准生命周期「下线」的归档步骤（Deprecate/Archive/Purge/Sever）与 `contract/workflow.md` 工作流。归档是 Retire.Archive 的实现，未建模为标准 workflow（stages/tasks/actions）。

## 结论

对得上的：skills、Policy（selector/mode）、契约文件位置。

对不上的：`assets`→`schemas` 改名、缺 spec_version/version/workflows/discovery/registry、验证二元→三态、guess_asset_type→maps 规则、技能缺 type/domain、归档未建模为 workflow。

升级动作的优先级即按此对照展开（见 index.md 演化路线图）。
