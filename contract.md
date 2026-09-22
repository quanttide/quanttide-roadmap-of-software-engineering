融合方案的核心：B 出结构，A 出语义，中间一份 structure.json
一句话讲透这个融合：你们的分析脚本（B）只做它最擅长的一件事——把代码库变成结构化数据；规则语义（A）交给现成词汇表达；两者共享同一份提取结果。这样图和检查永远不会漂移，因为它们本来就是同一份数据的两种渲染。
流水线长这样
src/**/*.rs
   │
   │ ① 结构提取（B 的方法论：grep use crate::、行数、dependents）
   ▼
structure.json          ← 你 meta 里那份分析 JSON 就是它的 schema，现成的
   │
   ├─② 规则求值（A 的语义：deny / allow / no-cycle / 纯度禁词）
   │     └─→ violations（CI 非零退出，带 evidence 行号）
   │
   └─③ 渲染
         └─→ dependency-graph.html（你已经有的那张图）
载体用 cargo-xtask——官方认可的给 cargo 加自定义命令的 polyfill 模式，写一个 xtask/ 子工程，cargo xtask check-arch 和 cargo xtask graph 两条命令，CI 挂前者。
职责划分
职责	归属	理由
结构提取（use crate:: 扫描、dependents、行数）	B（xtask）	已经写过、已经信任、meta 里的方法论直接复用
规则求值语义	A（借库的词汇或极薄的自写求值器）	不再手写判定逻辑，这是你之前担心的维护负担所在
规则声明	独立的 architecture-rules.toml	单一事实源，CONTRIBUTING 散文降级为指向它
报告	共用 structure.json	图与检查同源，物理上不可能漂移
规则文件长这样
# architecture-rules.toml —— 规则引用目录形状，不维护模块名单
[layers]
aggregate = [”src/order“, ”src/workflow“, ”src/workspace“, ”src/catalog“, ”src/artifact“, ”src/material“]
service   = [”src/search“, ”src/audit“]
infra     = [”src/infra/*“]
[[rule]]  # 聚合不得依赖服务
deny = { from = ”aggregate“, to = ”service“ }
[[rule]]  # 聚合之间不得成环
no-cycle = { among = ”aggregate“ }
[[rule]]  # events 不得反向引用聚合
deny = { from = [”src/events“], to = ”aggregate“ }
[[rule]]  # 领域纯度：禁词清单
forbid-tokens = { in = ”aggregate“, tokens = [”std::fs::“, ”std::env::“, ”serde_yaml::“] }
注意第 1 条和最后一条的区别：deny from aggregate to service 是引用层定义，加新聚合时只动 layers 里的一行；forbid-tokens 是按禁词扫，连名字都不用提。之前讨论过的「枚举负担」，在这个文件里被压到最小——而且就算暂时还是名单，它也只有一个副本（不是 CONTRIBUTING 一个、测试一个、分析脚本一个）。
求值器怎么选
两个都不错的取舍：
• 借库：arch-test_core 自己就做静态分析提取 use 关系，可以直接用它的 MayNotAccess / 环检测词汇，提取步骤交给它，你的脚本只剩渲染图；缺点是提取口径和你们的方法论不完全一致，得对一次账。
• 自写极薄求值器：structure.json 的 schema 简单且已定型（你自己刚写的），在 xtask 里写个百来行的求值器，逐条规则查图——遍历、环检测（DFS 染色）都是教材级代码，规则词汇全由你们的 toml 定义。
你们这个规模（单 crate、20 模块），自写求值器 + structure.json 是更干净的融合：提取、求值、渲染三段全是自己的代码和自己的 schema，没有外部依赖的黑盒；arch-test_core 留作备选，等规则词汇涨到十个以上再考虑换。
防漂移的性质是怎么成立的
这条最关键：CONTRIBUTING 的散文、tests/ 的硬编码、一次性的分析脚本——三处各自维护，是之前所有问题的共性。融合后只剩两个「活的」东西：
• structure.json 的提取器（代码，跟着模块改，提取器本身不需要知道模块叫什么）；
• architecture-rules.toml（唯一需要人改的文件）。
图和检查都只是这两者的派生物。改名 task→order 的那天，你改一次 toml，图的 meta、CI 的断言、CONTRIBUTING 的指向全部自动一致——这就是「B 出结构、A 出语义」融合之后买到的性质，也是单用 A 或单用 B 都拿不到的。

---

structure.json 是事实层，.toml 是契约层。和我们想要做的软件工程建模约束方向完全一致。


