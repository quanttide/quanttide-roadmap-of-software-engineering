# qtcloud-asset 演化路线图

让 qtcloud-asset 的契约与实现对齐数字资产治理建模方向（见 quanttide-asset `docs/specification`）：契约驱动、发现规则化、验证三态、资产目录化。

## 阶段一：契约升级

对齐契约字段，纯配置变更。

- [ ] `contract.yaml` 增加 `spec_version: 0.1.0` 与 `version`
- [ ] `assets` 段改名为 `schemas`，字段对齐标准（title/type/category/path）
- [ ] 40 个 OSS 桶从硬编码清单改为 `discovery.maps` 规则（`*-studio` → studio、`*-private` → private、`*-site` → site、`*-provider` → provider）
- [ ] 补充 `validation.policies`，把现有验证逻辑搬进契约
- [ ] skills 补充 `type`（rule / bridge / agent）
- [ ] 归档流程建模为 `workflows`

## 阶段二：发现引擎对齐

把发现活动对齐五步流程（加载→剪枝→采样→映射→发射）。

- [ ] CLI scanner 升级为 Load（scope）→ Prune（excludes）→ Map（maps 规则）→ Emit
- [ ] `guess_asset_type` 硬编码改为 `maps` 规则驱动
- [ ] Provider OSS 发现从硬编码清单改为 discovery 规则驱动

## 阶段三：验证对齐

- [ ] 验证结果对齐三态：ALIGNED / DISCOVERED / DRIFTED
- [ ] policies 从契约文件加载，删除代码内硬编码

## 阶段四：资产目录与注册

- [ ] Provider 引入资产目录，OSS 桶注册为资产条目（id / type / category / tags）
- [ ] 注册、维护、下线成为显式生命周期行为
