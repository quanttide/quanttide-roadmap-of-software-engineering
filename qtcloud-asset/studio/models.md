# qtcloud-asset Studio 模型与工程标准对照

## 一句话概括

Studio 是展示层，它的模型是 Provider 的镜像加上交互模型。对照数字资产治理标准（quanttide-asset `docs/specification`），主要缺口是分类逻辑重复实现、契约页面未真正由契约驱动。

## 概念

### Bucket 与 OssObject

与 Provider 一致，都是资产——容器层（`type: OSS`）与内容层，客户端字段即 Provider 字段的镜像，对应标准 `catalog/asset.md` 资产模型。缺口同 Provider：无 id/category/tags/title。

### category 派生（Bucket.category）

`lib/main.dart` 里按桶名后缀推断分类（`-studio`/`-private`/`-site`/`-provider`）并派生图标颜色，对应标准的 `category`——「用于结构化分类和权限控制」。这是 Provider `isPrivateBucket` 同一逻辑的**重复实现**：分类在两端各硬编码一份。标准建模是资产属性驱动展示，应改为消费 Provider 返回的 category/tags 字段，删除客户端推断逻辑。

### AssetContractScreen

`lib/screens/asset_contract_screen.dart` 的资产注册表页面，对应标准契约的 `schemas`——「结构即界面」，契约驱动的目录镜像。当前是静态硬编码的资产卡片格子，未读取 `.quanttide/asset/contract.yaml`。演化方向：按契约 `schemas`/`discovery` 结果渲染，成为契约驱动界面的落地页。

### ProviderUser / FolderShare

标准无对应：身份与分享是资产治理之外的能力层，与 Provider 侧结论一致，保持独立建模。

### 交互模型（BucketSortMode、筛选、分页、isDir）

纯展示层模型，标准不定义，不冲突。

### 平台隔离

`download_url`/`login_redirect`/`provider_http_client` 的 io/web 双实现对应标准的「连接器/适配器」思想——展示端把平台差异隔离在接口后。实现方式已符合标准分层，无需改动。

## 结论

对得上的：镜像模型、平台隔离双实现。

对不上的：category 分类逻辑在客户端重复硬编码（应改为消费资产属性）、契约页面静态未由契约驱动。

升级动作见 index.md 演化路线图（阶段四资产目录化后，Studio 由目录数据驱动展示）。
