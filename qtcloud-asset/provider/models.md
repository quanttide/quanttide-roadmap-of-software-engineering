# qtcloud-asset Provider 模型与工程标准对照

## 一句话概括

从 Provider 实际代码出发，把每个模型概念对照数字资产治理标准（quanttide-asset `docs/specification`）：大部分概念能找到标准对应，少数需要补字段或重构。

## 概念

### Bucket 与 Object

`internal/schema/schema.go` 的 `Bucket`（name/region/storage_class/created_at）与 `Object`（key/size/type/storage_class/last_modified）都是资产，对应标准 `catalog/asset.md` 的资产模型，只是不同层级、不同类型：Bucket 是容器层资产（`type: OSS`），Object 是内容层资产（key 即其唯一标识）。

| 代码字段 | 标准字段 | 关系 |
|:--|:--|:--|
| Bucket.Name / Object.Key | name | 一致：key 即资产标识 |
| Bucket.CreatedAt / Object.LastModified | created_at / updated_at | 一致 |
| Bucket.Region / StorageClass、Object.Size / Type / StorageClass | — | 标准未定义，保留为连接器私有属性 |
| — | id | 缺口：代码无唯一标识 |
| — | title / description | 缺口：代码无资产标题与描述 |
| — | category | 缺口：代码的用途分类（studio/private/site）是枚举逻辑，应落为 category |
| — | tags | 缺口：无标签 |

层级关系（桶包含对象）标准资产模型未定义 parent 字段，可用 `tags`（如 `parent: <桶名>`）或注册结构表达。

### SourceAdapter 与 OssAdapter

对应标准的「连接器/适配器」概念——`catalog/asset.md` 明确 `type` 对应不同连接器/适配器（S3、Git 等）。`OssAdapter` 就是 OSS 连接器的实现，GitHub、飞书适配器即标准多源的扩展。五个窄接口（BucketLister/ObjectLister 等）是连接器的实现细节，标准不定义到接口级，不冲突。

### 隐私边界（isPrivateBucket）

硬编码在 `internal/repository/oss.go` 的私密桶判定（`-private` 后缀、`quanttide-terraform-state`），对应标准的 `category` 用途——「用于结构化分类和权限控制」。标准建模是资产属性驱动权限，现状是代码函数驱动，是重构点。

### User / Role / Session

标准无对应。资产治理标准不定义身份模型，身份是正交的能力层，保持独立建模即可。

### AuditLog

对应标准生命周期「维护（Maintain）——通过自动化审计与人工干预确保元数据与物理实体一致」。现状的 AuditAction 枚举覆盖登录、列桶、分享等操作级审计；标准行为级动作（注册、验证、下线）尚未进入枚举，是扩展点。

### Share（Record / Store）

标准无对应。分享是资产治理之上的应用能力（把资产打包给人看），超出标准范围。

## 结论

对得上的：SourceAdapter/连接器、审计（维护）。

对不上的：Bucket/Object 缺 id/category/tags、隐私边界硬编码→资产属性。

升级动作的优先级即按此对照展开（见 index.md 演化路线图）。
