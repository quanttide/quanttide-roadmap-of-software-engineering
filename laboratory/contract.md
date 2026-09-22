# 架构检查融合方案

融合方案要让结构提取与规则语义各归其位：B 出结构，A 出语义，两者之间用一份 `structure.json` 衔接。下文所称 A 指规则求值一侧，B 指结构提取一侧，二者是此前分别评估过的两条路线。

这样分工的效果可以一句话说明：分析脚本 B 只做它最擅长的一件事，把代码库变成结构化数据；规则语义 A 交给现成词汇表达；两者共享同一份提取结果。图与检查因此永远不会漂移，因为它们本来就是同一份数据的两种渲染。

## 流水线

```text
src/**/*.rs
   │
   │ ① 结构提取（B：grep use crate::、行数、dependents）
   ▼
structure.json          ← 既有的 meta 分析 JSON 即其 schema
   │
   ├─ ② 规则求值（A：deny / allow / no-cycle / 纯度禁词）
   │       └─→ violations（CI 非零退出，附 evidence 行号）
   │
   └─ ③ 渲染
           └─→ dependency-graph.html
```

流水线分三段：结构提取沿用 B 的方法论，扫描 `use crate::`、统计行数与 dependents，产出 `structure.json`；规则求值取 A 的语义，判定 deny、allow、no-cycle 与纯度禁词，产出 `violations` 供 CI 判定；渲染产出 `dependency-graph.html`，即现有的那张图。

实现载体用 `cargo-xtask`，这是官方认可的为 cargo 添加自定义命令的模式。新建一个 `xtask/` 子工程，提供 `cargo xtask check-arch` 与 `cargo xtask graph` 两条命令，CI 挂前者。

## 职责划分

| **职责** | **归属** | **理由** |
|:--|:--|:--|
| 结构提取（`use crate::` 扫描、dependents、行数） | B，`xtask` 子工程 | 已写过、已信任，meta 的方法论直接复用 |
| 规则求值语义 | A，借库词汇或极薄的自写求值器 | 不再手写判定逻辑，维护负担集中于此 |
| 规则声明 | 独立的 `architecture-rules.toml` | 单一事实源，CONTRIBUTING 散文降级为指向它 |
| 报告 | 共用 `structure.json` | 图与检查同源，物理上不可能漂移 |

## 规则文件

规则集中声明在独立的 `architecture-rules.toml` 中，只引用目录形状，不维护模块名单。

```toml
# architecture-rules.toml —— 规则引用目录形状，不维护模块名单
[layers]
aggregate = ["src/order", "src/workflow", "src/workspace", "src/catalog", "src/artifact", "src/material"]
service   = ["src/search", "src/audit"]
infra     = ["src/infra/*"]

[[rule]]  # 聚合不得依赖服务
deny = { from = "aggregate", to = "service" }

[[rule]]  # 聚合之间不得成环
no-cycle = { among = "aggregate" }

[[rule]]  # events 不得反向引用聚合
deny = { from = ["src/events"], to = "aggregate" }

[[rule]]  # 领域纯度：禁词清单
forbid-tokens = { in = "aggregate", tokens = ["std::fs::", "std::env::", "serde_yaml::"] }
```

注意第一条与最后一条的区别：`deny` 是引用层定义，新增聚合时只需改动 `layers` 中的一行；`forbid-tokens` 按禁词扫描，连模块名字都不必出现。此前讨论过的枚举负担因此被压到最小，而且即便暂时仍用名单，它也只有一个副本，不再出现 CONTRIBUTING 一份、测试一份、分析脚本一份的情况。

## 求值器的选择

规则求值有两个取舍，都不差。

借库方案是直接用 `arch-test_core`。它本身就做静态分析并提取 use 关系，可以直接使用它的 `MayNotAccess` 与环检测词汇，提取步骤一并交给它，脚本只剩渲染图。代价是提取口径与既有方法论不完全一致，需要先对一次账。

自写极薄求值器则以 `structure.json` 的 schema 为准。该 schema 简单且已经定型，在 xtask 中写一个百来行的求值器逐条规则查图即可，遍历与环检测（DFS 染色）都是教材级代码，规则词汇全部由 TOML 定义。

就当前规模（单 crate、20 个模块）而言，自写求值器加 `structure.json` 是更干净的融合：提取、求值、渲染三段都由自己的代码和自己的 schema 承载，没有外部黑盒。`arch-test_core` 留作备选，待规则词汇增长到十个以上再考虑替换。

## 防漂移

防漂移的性质是这套方案最关键的部分。此前 CONTRIBUTING 的散文、`tests/` 中的硬编码、一次性的分析脚本三处各自维护，是全部问题的共性。

融合之后只剩下两个「活的」东西：`structure.json` 的提取器，即代码，随模块变化，但提取器本身并不需要知道模块叫什么；以及 `architecture-rules.toml`，唯一需要人改的文件。图与检查都只是这两者的派生物。

将 `task` 改名为 `order` 时，只需改一次 TOML，图的 meta、CI 的断言、CONTRIBUTING 的指向就会自动一致。这就是「B 出结构、A 出语义」融合之后买到的性质，也是单用 A 或单用 B 都拿不到的。

## 结论

`structure.json` 是事实层，`architecture-rules.toml` 是契约层。这一分工与软件工程建模约束的方向完全一致。
