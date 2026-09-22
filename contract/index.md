# 契约引擎

## 一、背景与目标

当前，软件工程”第二大脑“知识库的内容组织面临维度混杂的挑战：流程性内容（如 Code Agent 工作流、Code Document 循环）与分析性工具（如依赖图分析）混合存储，导致知识体系结构不够清晰。同时，架构约束散落在 CONTRIBUTING 散文、tests/ 硬编码、一次性分析脚本三处，各自维护，模块改名或移动时极易漏改，导致文档、测试、工具三者说的不是同一件事。

本方案旨在构建一套架构契约引擎，将架构约束从”人脑记忆+散文描述“升级为”可执行、可验证、防漂移的契约体系“，核心手段是结构与语义解耦、单一事实源派生。

## 二、核心设计思想

B 出结构，A 出语义，单一事实源防漂移。

- B（结构提取侧）：只负责一件事——扫描代码，产出 catalog.json。它不关心规则，只产出”代码现在长什么样“的事实。
- A（规则求值侧）：只负责一件事——读取 catalog.json，对照 contract.yaml 里的规则，判定是否有违规。它不关心代码怎么扫，只负责”架构应该长什么样“的语义求值。
- catalog.json：作为两者之间的唯一契约，图和检查都从同一份数据派生，物理上不可能漂移。

## 三、文件结构与命名

遵循量潮数据工程标准 v0.0.1 的命名规范，在 .quanttide/code/ 目录下建立软件工程的治理体系：

.quanttide/
├── data/
│   ├── pipeline/
│   ├── blueprint/
│   ├── contract/
│   └── catalog/
└── code/
    ├── catalog.json        # 代码结构事实目录
    └── contract.yaml       # 架构契约

- catalog.json：事实层，描述代码库”实际是什么样子“
- contract.yaml：契约层，描述架构”应该是什么样子“

## 四、流水线设计

src/**/*.rs
   │
   │ ① 结构提取（B：grep use crate::、行数、dependents）
   ▼
catalog.json          ← 代码结构事实目录
   │
   ├─ ② 规则求值（A：deny / allow / no-cycle / 纯度禁词）
   │       └─→ violations（CI 非零退出，附 evidence 行号）
   │
   └─ ③ 渲染
           └─→ dependency-graph.html

流水线分三段：
1. 结构提取：扫描 use crate::、统计行数与 dependents，产出 catalog.json
2. 规则求值：读取 catalog.json，对照 contract.yaml，产出 violations 供 CI 判定
3. 渲染：复用现有依赖图渲染逻辑，输出 dependency-graph.html

## 五、实现载体

使用 cargo-xtask，这是官方认可的为 cargo 添加自定义命令的模式。新建一个 xtask/ 子工程，提供两条命令：

- cargo xtask check-arch：执行完整流水线（提取→求值→输出 violations），CI 挂载这条
- cargo xtask graph：执行提取→渲染，本地开发时查看依赖图

## 六、职责划分
职责   归属   理由
:— :— :—
结构提取（use crate:: 扫描、dependents、行数）   B，xtask 子工程   已写过、已信任，meta 的方法论直接复用
规则求值语义   A，极薄自写求值器   不再手写判定逻辑，维护负担集中于此
规则声明   独立的 contract.yaml   单一事实源，CONTRIBUTING 散文降级为指向它
报告   共用 catalog.json   图与检查同源，物理上不可能漂移

## 七、契约文件格式

contract.yaml 只引用目录形状，不维护模块名单：

# contract.yaml —— 规则引用目录形状，不维护模块名单
layers:
  aggregate:
    - src/order
    - src/workflow
    - src/workspace
    - src/catalog
    - src/artifact
    - src/material
  service:
    - src/search
    - src/audit
  infra:
    - src/infra/*

rules:
  - deny:
      from: aggregate
      to: service

  - no-cycle:
      among: aggregate

  - deny:
      from:
        - src/events
      to: aggregate

  - forbid-tokens:
      in: aggregate
      tokens:
        - ”std::fs::“
        - ”std::env::“
        - ”serde_yaml::“

关键点：
- layers 定义目录形状，新增聚合模块只需在这里加一行
- deny 是层间引用约束
- no-cycle 是环检测
- forbid-tokens 是领域纯度禁词扫描，连模块名都不必出现

## 八、防漂移机制

融合之后只剩下两个”活的“东西：
- catalog.json 的提取器（代码）：随模块变化自动更新，提取器本身不需要知道模块叫什么
- contract.yaml（契约）：唯一需要人改的文件

图和检查都只是这两者的派生物。将 task 改名为 order 时，只需改一次 YAML，图的 meta、CI 的断言、CONTRIBUTING 的指向就会自动一致。这就是”B 出结构、A 出语义“融合之后买到的性质。

## 九、求值器选择

就当前规模（单 crate、20 个模块）而言，自写极薄求值器加 catalog.json 是更干净的融合：提取、求值、渲染三段都由自己的代码和自己的 schema 承载，没有外部黑盒。arch-test_core 留作备选，待规则词汇增长到十个以上再考虑替换。

## 十、演进路径
阶段   核心目标   关键产出
第一阶段   工具引入与手动干预   catalog.json 提取器完成，初步分析能力建立
第二阶段   规则引擎建设与沉淀   contract.yaml 完成，规则结构化表达，版本管理机制
第三阶段   智能体自动化实现   感知-决策-执行闭环、反馈机制、端到端自动化

这套方案的核心价值在于：catalog.json 是事实层，contract.yaml 是契约层。这一分工与软件工程建模约束的方向完全一致。


