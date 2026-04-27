# AI-Native APS 高级计划与排程系统 PRD + DDD 技术设计文档 V3.0

> 文档定位：产品需求 PRD + DDD 领域设计 + 技术架构 + Timefold 求解器设计 + LLM/Agent 设计。  
> 本版本已合并：多工艺路线选择、工艺路线 DAG、并行/串行工序、多入口多出口、多产出、副产物/联产品建模。

---

# 0. 文档总览

本文件用于设计一套面向离散制造企业的 **AI-Native APS 高级计划与排程系统**。

系统核心思想：

```text
DDD 领域模型
    ↓
APS 核心业务能力
    ↓
Timefold 约束求解器
    ↓
甘特图调度工作台
    ↓
MES 执行反馈闭环
    ↓
Agent Tool 化
    ↓
LLM 智能调度助手
```

系统不是简单的“带聊天框的 APS”，而是一个：

> 所有核心排程能力都可以被人和智能体共同调用，并且具备权限、审批、版本、审计和解释能力的 AI 原生 APS 平台。

---

# 1. 产品定位

## 1.1 产品名称

AI-Native APS 智能计划与排程系统。

## 1.2 产品定义

本系统是面向离散制造企业的高级计划与排程系统，定位于 ERP 与 MES 之间的智能排程中枢。

系统通过 DDD 领域模型统一表达工厂、订单、产品、工艺路线、工序图、资源、物料、日历、约束、排程方案和执行反馈，通过 Timefold 求解器生成可执行排程方案，并通过 LLM 智能体提供自然语言交互、排程解释、异常诊断、方案对比、重排建议和人机协同调度能力。

## 1.3 一句话定义

> 这不是一个“带聊天框的 APS”，而是一个“所有核心排程能力都可以被人和智能体共同调用的 AI 原生 APS 平台”。

---

# 2. 建设目标

## 2.1 业务目标

| 目标 | 说明 |
|---|---|
| 降低人工排产成本 | 减少 Excel 手工排程、反复协调、人工核对 |
| 提高交期达成率 | 优先满足重要订单、紧急订单、客户承诺订单 |
| 提高设备利用率 | 减少设备空闲和不合理等待 |
| 降低换产损失 | 减少规格、颜色、材质、模具切换 |
| 快速响应异常 | 支持插单、缺料、设备故障后的快速重排 |
| 支持多工艺路线优化 | 一个产品可有多条候选工艺路线，系统可自动选择更优路线 |
| 支持复杂工艺图 | 一条工艺路线可包含串行、并行、分叉、汇合、多入口、多出口工序 |
| 支持多产出建模 | 工序可产生主产品、副产物、联产品、半成品 |
| 增强计划可解释性 | 解释为什么这样排、为什么等待、为什么选这条工艺路线、为什么产生副产物 |
| 构建智能调度助手 | 支持自然语言查询、诊断、模拟和调整建议 |

## 2.2 技术目标

| 目标 | 说明 |
|---|---|
| DDD 建模 | 用领域模型表达 APS 业务复杂性 |
| 求解器解耦 | Timefold 作为独立求解上下文 |
| 工艺图建模 | 用 ProcessGraph / TaskGraph 表达复杂工艺路线 |
| AI 原生 | 所有关键业务动作都可暴露为 Agent Tool |
| 可审计 | 智能体操作必须留痕、可回滚、可审批 |
| 可扩展 | 支持物料、模具、人员、外协、多工厂逐步扩展 |
| 可解释 | 约束冲突、评分变化、重排影响、路线选择原因、副产物流向必须可解释 |

---

# 3. 系统边界

## 3.1 APS 与其他系统关系

| 系统 | 职责 |
|---|---|
| ERP | 订单、BOM、库存、采购、财务 |
| PLM | 产品、工艺、工程变更 |
| WMS | 仓库、批次、库存、出入库 |
| MES | 现场执行、报工、设备状态、质检 |
| APS | 有限产能排程、方案优化、调度协同、计划发布 |
| LLM Agent | 解释、查询、诊断、模拟、建议、调用 APS 能力 |

## 3.2 APS 不做什么

1. 不替代 ERP 的订单管理。
2. 不替代 MES 的现场报工。
3. 不替代 WMS 的库存管理。
4. 不承诺数学意义上的全局最优。
5. 不允许 LLM 直接修改数据库。
6. 不允许智能体绕过权限、审批和版本机制。
7. 不允许已发布计划被直接覆盖修改。

---

# 4. 用户角色

| 角色 | 主要工作 |
|---|---|
| 计划员 | 创建排程方案、调整甘特图、发布计划 |
| 生产主管 | 审批方案、查看瓶颈、决策加班/外协 |
| 车间班组长 | 查看任务、反馈异常、确认执行 |
| 工艺工程师 | 维护产品、工艺路线、工序图、设备能力、工序参数 |
| 系统管理员 | 配置权限、基础数据、接口规则 |
| 智能调度助手 | 查询、解释、诊断、生成建议、调用工具 |
| 求解服务 | 根据约束生成排程结果 |
| 集成服务 | 对接 ERP、MES、WMS、PLM |

---

# 5. 核心业务流程

## 5.1 标准排程流程

```text
ERP/PLM/WMS/MES 数据同步
        ↓
APS 基础数据校验
        ↓
生产订单进入排程池
        ↓
根据产品筛选候选工艺路线
        ↓
选择或优化工艺路线
        ↓
根据选定工艺路线的 ProcessGraph 生成 TaskGraph
        ↓
识别串行、并行、汇合、分叉、多入口、多出口关系
        ↓
识别主产品、副产物、联产品、半成品输出
        ↓
选择排程范围
        ↓
调用 Timefold 求解
        ↓
生成排程方案
        ↓
甘特图展示与人工调整
        ↓
AI 助手解释风险、路线选择、等待原因和副产物流向
        ↓
方案审批
        ↓
发布到 MES
        ↓
MES 执行反馈
        ↓
APS 识别偏差
        ↓
局部重排 / 滚动重排
```

## 5.2 AI 原生交互流程

用户可以这样问系统：

```text
帮我看看今天哪些订单有延期风险？
为什么订单 WO-10086 会延期？
为什么 WO-001 没有选择标准工艺路线？
为什么这个工序不能提前？
为什么这个订单开头有多个工序同时开始？
这个副产物什么时候可以作为饲料原料使用？
哪些订单消耗了这个副产物？
如果把 M01 设备停机 4 小时，会影响哪些订单？
如果不考虑副产物复用，主产品交期会不会更好？
比较方案 A 和方案 B，哪个更适合发布？
```

智能体不直接改数据，而是执行：

```text
理解用户意图
  ↓
选择可调用工具
  ↓
查询 APS 数据
  ↓
调用模拟 / 求解 / 分析服务
  ↓
生成解释和建议
  ↓
需要修改时请求用户确认
  ↓
通过正式 Command 修改系统
  ↓
记录审计日志
```

---

# 6. 产品模块设计

```text
AI-Native APS
├── 基础建模模块
│   ├── 工厂建模
│   ├── 资源建模
│   ├── 日历班次
│   ├── 产品建模
│   ├── 工艺路线
│   ├── 工艺路线 DAG 编辑器
│   ├── 多工艺路线候选规则
│   ├── 工序输入输出建模
│   ├── 副产物/联产品建模
│   └── 资源能力
│
├── 订单与任务模块
│   ├── 生产订单
│   ├── 排程池
│   ├── 候选工艺路线生成
│   ├── 工艺路线选择
│   ├── ProcessGraph → TaskGraph 转换
│   ├── 工序任务生成
│   ├── TaskDependency 生成
│   ├── PlannedOutput 生成
│   └── 任务状态管理
│
├── 排程求解模块
│   ├── 约束配置
│   ├── 全量求解
│   ├── 局部重排
│   ├── 方案评分
│   ├── 路线选择评分
│   ├── DAG 依赖约束
│   ├── 副产物可用性约束
│   └── 约束解释
│
├── 方案管理模块
│   ├── 排程方案
│   ├── 方案版本
│   ├── 方案对比
│   ├── 路线选择记录
│   ├── 任务图快照
│   ├── 副产物计划输出
│   ├── 审批发布
│   └── 回滚作废
│
├── 调度工作台
│   ├── 资源甘特图
│   ├── 订单甘特图
│   ├── 工艺任务图
│   ├── 路线选择解释
│   ├── 并行/汇合等待解释
│   ├── 副产物流向解释
│   ├── 拖拽调整
│   ├── 冲突检测
│   └── 局部重排
│
├── 执行反馈模块
│   ├── MES 报工
│   ├── 设备异常
│   ├── 缺料异常
│   ├── 偏差识别
│   └── 滚动重排
│
├── AI 智能体模块
│   ├── 调度助手
│   ├── 方案解释 Agent
│   ├── 路线选择解释 Agent
│   ├── 工艺图解释 Agent
│   ├── 副产物流向 Agent
│   ├── 异常诊断 Agent
│   ├── What-if 模拟 Agent
│   ├── 数据质量 Agent
│   └── 发布风控 Agent
│
└── 系统集成模块
    ├── ERP 集成
    ├── MES 集成
    ├── WMS 集成
    └── PLM 集成
```

---

# 7. DDD 领域划分

| 上下文 | 类型 | 说明 |
|---|---|---|
| Factory Modeling Context | 支撑域 | 工厂、车间、设备、工作中心 |
| Process Context | 核心支撑域 | 产品、工艺路线、工序图、输入输出、副产物 |
| Demand Context | 支撑域 | 生产订单、需求、交期 |
| Resource Capacity Context | 核心支撑域 | 资源能力、日历、班次、产能 |
| Routing Selection Context | 核心域 | 多工艺路线候选、选择、解释、版本记录 |
| Scheduling Context | 核心域 | 排程任务、方案、版本、计划 |
| Solver Context | 核心域 | Timefold 求解模型、约束、评分 |
| Execution Context | 支撑域 | MES 反馈、开工、完工、异常 |
| Integration Context | 通用域 | ERP/MES/WMS/PLM 集成 |
| Agent Context | 创新核心域 | LLM 智能体、工具调用、会话、任务 |
| Governance Context | 通用域 | 权限、审批、审计、版本、风控 |

最核心上下文：

```text
Process Context
Routing Selection Context
Scheduling Context
Solver Context
Agent Context
```

---

# 8. Process Context：复杂工艺路线建模

## 8.1 为什么不能只用线性工序列表

传统简单模型：

```text
工序 10 → 工序 20 → 工序 30
```

这种模型只能表达简单串行工艺，无法表达：

1. 开头多个工序并行。
2. 中间多个工序分叉。
3. 多个工序汇合后再进入下一工序。
4. 多个并行输出。
5. 一个工序产生主产品、副产物、联产品。
6. 副产物继续作为其他订单或其他产品的输入。

因此，本系统应将工艺路线建模为：

```text
工艺路线 = 工序节点 + 工序依赖边 + 输入物 + 输出物 + 资源能力 + 时间关系
```

也就是有向无环图 DAG。

## 8.2 Product 聚合

```text
Product
├── productId
├── productCode
├── productName
├── specification
├── status
└── routingIds
```

一个产品可以拥有多条工艺路线。

## 8.3 Routing 聚合

```text
Routing
├── routingId
├── productId
├── routingCode
├── routingName
├── version
├── status
├── priority
├── routingType
├── costFactor
├── efficiencyFactor
├── qualityLevel
├── effectiveDate
├── expireDate
└── processGraph
```

字段解释：

| 字段 | 说明 |
|---|---|
| priority | 路线优先级，数字越小优先级越高 |
| routingType | 标准路线、替代路线、应急路线、外协路线 |
| costFactor | 成本系数 |
| efficiencyFactor | 效率系数 |
| qualityLevel | 质量等级 |
| processGraph | 工艺路线图，不是简单工序列表 |

## 8.4 ProcessGraph 聚合

```text
ProcessGraph
├── routingId
├── nodes: List<OperationNode>
├── edges: List<OperationEdge>
├── inputItems: List<ProcessInput>
├── outputItems: List<ProcessOutput>
└── validationRules
```

领域规则：

1. ProcessGraph 必须是有向无环图。
2. 不允许出现循环依赖。
3. 不允许出现孤立工序节点。
4. 允许多个入口节点。
5. 允许多个出口节点。
6. 允许多个输出物。
7. 每个出口节点必须定义输出结果。
8. 每条边必须定义依赖关系类型。

## 8.5 OperationNode 工序节点

```text
OperationNode
├── operationNodeId
├── routingId
├── operationCode
├── operationName
├── operationType
├── requiredCapabilities
├── processingTimeRule
├── setupTimeRule
├── inputItems
├── outputItems
└── resourceRequirements
```

operationType 示例：

```text
PROCESSING
INSPECTION
PACKAGING
MIXING
SPLITTING
SEPARATION
ASSEMBLY
OUTSOURCING
STORAGE_WAIT
```

## 8.6 OperationEdge 工序依赖边

```text
OperationEdge
├── edgeId
├── routingId
├── predecessorOperationNodeId
├── successorOperationNodeId
├── relationType
├── minLag
├── maxLag
├── transferTime
└── dependencyCondition
```

relationType 建议支持：

| 类型 | 含义 |
|---|---|
| FS | 前工序完成后，后工序才能开始 |
| SS | 前工序开始后，后工序才能开始 |
| FF | 前工序完成后，后工序才能完成 |
| SF | 前工序开始后，后工序才能完成，较少使用 |
| JOIN_ALL | 多个前置工序全部完成后才能开始 |
| JOIN_ANY | 多个前置工序任一完成后即可开始 |
| SPLIT | 一个工序完成后分叉到多个后续工序 |

## 8.7 工艺 DAG 示例

### 简单串行

```text
A → B → C → D
```

### 开头多个并行工序

```text
A1 ┐
   ├→ C → D
A2 ┘
```

含义：A1 和 A2 可以并行开始，C 必须等 A1 和 A2 都完成后才能开始。

### 中间分叉再汇合

```text
A → B1 ┐
       ├→ D
A → B2 ┘
```

含义：A 完成后，B1 和 B2 可以并行，D 需要等 B1 和 B2 都完成。

### 多出口工艺路线

```text
A → B → C1 → 主产品
      → C2 → 副产物
      → C3 → 联产品
```

含义：同一工艺路线可能最终产生多个输出物，这些输出物可能有不同用途、库存位置、质量等级和后续需求。

---

# 9. 多输入、多输出、副产物与联产品建模

## 9.1 OperationInput 工序输入

```text
OperationInput
├── inputId
├── operationNodeId
├── itemId
├── itemType
├── quantity
├── unit
├── consumeTiming
└── requiredQualityLevel
```

itemType 可包括：

```text
RAW_MATERIAL
SEMI_FINISHED
MAIN_PRODUCT
BY_PRODUCT
CO_PRODUCT
AUXILIARY_MATERIAL
PACKAGING_MATERIAL
```

consumeTiming 可包括：

```text
AT_START
DURING_PROCESS
AT_END
```

## 9.2 OperationOutput 工序输出

```text
OperationOutput
├── outputId
├── operationNodeId
├── itemId
├── itemType
├── quantityRule
├── unit
├── outputTiming
├── qualityLevel
├── storageLocationRule
└── availableForReuse
```

outputTiming 可包括：

```text
AT_END
BATCH_COMPLETE
CONTINUOUS
```

availableForReuse 表示该输出是否可以被其他订单、其他工艺路线、后续工序使用。

## 9.3 主产品 Main Product

主产品是生产订单的主要交付对象。

```text
ProductionOrder.productId = MainProduct
```

## 9.4 副产物 By-product

副产物是生产过程中附带产生的物料，不一定是废料，可能具备后续经济价值。

例如：食品、生物加工、视频内容加工、化工等行业中，主产品生产过程中产生的附加物，可以作为饲料、二次加工原料、素材包、边角料再利用资源。

副产物需要建模：

```text
ByProduct
├── itemId
├── sourceOperationNodeId
├── quantityRule
├── qualityLevel
├── availableTime
├── storageLocation
├── reusable
└── targetUsage
```

## 9.5 联产品 Co-product

联产品是与主产品同时产生、同样具有计划价值的产出。

与副产物不同，联产品通常需要被计划和交付，不应简单作为附带收益处理。

## 9.6 排程影响

如果某个工序产生副产物，并且该副产物可用于其他订单，那么 APS 需要知道：

1. 副产物什么时候产生。
2. 副产物数量是多少。
3. 副产物质量是否满足要求。
4. 副产物是否需要检验后才能使用。
5. 副产物是否需要进入库存。
6. 后续订单是否可以消耗该副产物。

---

# 10. Routing Selection Context：多工艺路线有限选择

## 10.1 业务定义

同一个产品允许存在多条可选工艺路线。系统在排程时不能固定使用某一条路线，而应在满足硬约束的前提下，优先选择优先级更高、成本更低、效率更好、交期结果更优的工艺路线。

例如：

| 工艺路线 | 优先级 | 说明 |
|---|---:|---|
| Routing-A | 1 | 标准路线，成本最低，质量最稳定 |
| Routing-B | 2 | 替代路线，效率稍低 |
| Routing-C | 3 | 应急路线，可用设备多，但成本高 |

核心原则：

```text
优先路线 ≠ 强制路线
```

正确表达：

```text
在满足硬约束和整体优化目标的前提下，尽量选择优先级更高的路线。
```

## 10.2 RoutingApplicableRule

```text
RoutingApplicableRule
├── routingId
├── minQuantity
├── maxQuantity
├── customerScope
├── orderTypeScope
├── materialCondition
├── allowedFactoryIds
├── allowedWorkshopIds
├── effectiveDate
├── expireDate
└── requiredApproval
```

## 10.3 RoutingCandidate

```text
RoutingCandidate
├── candidateId
├── orderId
├── routingId
├── priority
├── applicable
├── unavailableReason
├── costFactor
├── efficiencyFactor
├── qualityLevel
├── estimatedDuration
├── estimatedResourceLoad
└── estimatedRisk
```

## 10.4 OrderRoutingAssignment

```text
OrderRoutingAssignment
├── assignmentId
├── orderId
├── selectedRoutingId
├── selectionMode
├── selectionReason
├── scoreImpact
├── priorityPenalty
├── planVersionId
├── createdBy
└── createdAt
```

路线选择模式：

```text
AUTO_SELECTED
MANUALLY_SPECIFIED
EXTERNALLY_SPECIFIED
SIMULATION_SELECTED
```

## 10.5 硬约束

| 约束 | 说明 |
|---|---|
| 路线必须属于产品 | 订单只能选择该产品允许的工艺路线 |
| 路线必须启用 | 停用路线不能选择 |
| 路线必须在有效期内 | 未生效或已失效路线不能选择 |
| 路线必须满足适用条件 | 数量、客户、订单类型、工厂范围必须匹配 |
| 路线工序必须可执行 | 所有工序都要能匹配资源 |
| 已发布路线不可随意变更 | 已发布计划中的路线变更需要新版本 |
| 人工指定路线必须校验 | 用户指定路线也不能违反硬约束 |

## 10.6 软约束

| 约束 | 说明 |
|---|---|
| 优先选择高优先级路线 | priority 越高，扣分越少 |
| 减少路线切换成本 | 尽量少用应急路线 |
| 优先选择低成本路线 | 成本系数越低越好 |
| 优先选择高效率路线 | 效率系数越高越好 |
| 优先选择质量等级高的路线 | 重要客户优先高质量路线 |
| 减少延期 | 如果高优先级路线导致延期，可以选择低优先级路线 |
| 保持计划稳定 | 重排时尽量不改变已选路线 |

## 10.7 评分公式建议

```text
routePriorityPenalty = (priority - 1) × routingPriorityWeight
```

示例：

| 路线 | priority | 扣分 |
|---|---:|---:|
| Routing-A | 1 | 0 |
| Routing-B | 2 | -100 |
| Routing-C | 3 | -200 |

如果 Routing-A 导致延期 1000 分，而 Routing-B 只扣 100 分，则系统可选择 Routing-B。

关键结论：

> 不要把“优先级高的工艺路线”做成硬约束，而应该做成软约束；否则系统会为了死守高优先级路线，排不出更好的整体计划。

---

# 11. Scheduling Context：任务图与排程方案

## 11.1 ProductionOrder

```text
ProductionOrder
├── orderId
├── orderCode
├── productId
├── quantity
├── dueDate
├── priority
├── customer
├── earliestStartDate
├── materialReadyDate
├── status
├── sourceSystem
├── selectedRoutingId
├── routingSelectionMode
└── allowedRoutingIds
```

## 11.2 TaskGraph 聚合

当生产订单选择某条工艺路线后，系统根据 ProcessGraph 生成 TaskGraph。

```text
TaskGraph
├── taskGraphId
├── orderId
├── routingId
├── taskNodes: List<OperationTask>
├── taskEdges: List<TaskDependency>
├── plannedOutputs: List<PlannedOutput>
└── status
```

## 11.3 OperationTask

OperationTask 必须记录来源工艺路线和来源工序节点。

```text
OperationTask
├── taskId
├── orderId
├── routingId
├── operationNodeId
├── taskCode
├── quantity
├── status
├── assignedMachineId
├── plannedStartTime
├── plannedEndTime
├── actualStartTime
├── actualEndTime
├── isPinned
├── pinType
├── predecessorTaskIds
├── successorTaskIds
├── inputItems
├── outputItems
└── version
```

## 11.4 TaskDependency

```text
TaskDependency
├── dependencyId
├── predecessorTaskId
├── successorTaskId
├── relationType
├── minLag
├── maxLag
├── transferTime
└── condition
```

## 11.5 PlannedOutput

```text
PlannedOutput
├── plannedOutputId
├── taskId
├── orderId
├── itemId
├── itemType
├── quantity
├── unit
├── availableTime
├── qualityLevel
├── storageLocation
├── reusable
└── consumedByTaskId
```

## 11.6 SchedulePlan

```text
SchedulePlan
├── planId
├── planCode
├── planName
├── planScope
├── status
├── score
├── createdBy
├── createdAt
├── currentVersionId
└── versions
```

## 11.7 ScheduleVersion

```text
ScheduleVersion
├── versionId
├── planId
├── versionNo
├── status
├── score
├── hardScore
├── softScore
├── generatedBy
├── generatedType
├── createdAt
├── taskAssignments
├── taskDependencies
├── plannedOutputs
└── routingAssignments
```

---

# 12. Timefold 求解器建模

## 12.1 求解器定位

Timefold 是本系统的严肃约束求解核心。LLM 不能替代 Timefold。

Timefold 负责：

```text
硬约束
软约束
评分
搜索
优化
重排
```

LLM 负责：

```text
理解
解释
诊断
建议
调用工具
生成报告
人机协同
```

## 12.2 V1 求解范围

第一版只做：

```text
设备有限产能排程
工序 DAG 依赖
设备能力匹配
设备日历
订单交期
任务锁定
基础换产
基础多工艺路线选择
基础副产物可用性
```

暂缓：

```text
复杂人员排班
复杂模具共享
多工厂协同
复杂批处理
拆单拆批
完全动态实体数量的路线求解
```

## 12.3 Planning Solution

```java
ScheduleSolution
├── List<Machine>
├── List<OrderRoutingAssignment>
├── List<OperationTaskAssignment>
├── List<TaskDependency>
├── List<PlannedOutput>
├── List<TimeSlot> 或时间变量
└── HardSoftScore
```

## 12.4 Planning Entity

### OperationTaskAssignment

```java
OperationTaskAssignment
├── taskId
├── orderId
├── routingId
├── operationNodeId
├── requiredCapability
├── quantity
├── dueDate
├── priority
├── pinned
├── assignedMachine
├── startTime
├── endTime
└── sequence
```

### OrderRoutingAssignment

```java
OrderRoutingAssignment
├── orderId
├── candidateRoutings
├── selectedRouting
├── routingSelectionMode
├── routingPriority
├── routingCostFactor
└── routingPenalty
```

## 12.5 DAG 依赖硬约束

传统约束：

```text
后工序开始时间 >= 前工序结束时间
```

DAG 约束：

```text
对于每一条 TaskDependency：
successor.startTime >= predecessor.endTime + minLag + transferTime
```

如果是 JOIN_ALL：

```text
successor.startTime >= max(all predecessor.endTime + lag)
```

如果是 SPLIT：

```text
多个后续工序可以在前置工序完成后并行开始
```

## 12.6 副产物可用性约束

如果任务 B 消耗任务 A 产生的副产物，则：

```text
taskB.startTime >= plannedOutputA.availableTime
```

同时：

```text
sum(consumedQuantity) <= producedQuantity
```

## 12.7 硬约束汇总

| 约束 | 说明 |
|---|---|
| Resource Conflict | 同一设备同一时间不能加工多个任务 |
| Task Graph Dependency | 按 TaskDependency 满足 DAG 依赖 |
| Join Dependency | 汇合任务必须等待所有必需前置完成 |
| Machine Capability | 工序必须分配给具备能力的设备 |
| Calendar Availability | 任务不能排在非工作时间 |
| Pinned Task | 锁定任务不能被改变 |
| Material Ready Time | 物料未齐套前不能开工 |
| By-product Availability | 副产物未产生前不能被消耗 |
| By-product Quantity | 副产物消耗量不能超过产生量 |
| Machine Downtime | 停机时间不能安排任务 |
| Routing Belongs To Product | 工艺路线必须属于该产品 |
| Routing Applicable | 工艺路线必须满足适用条件 |
| Routing Enabled | 工艺路线必须启用并在有效期内 |

## 12.8 软约束汇总

| 约束 | 说明 |
|---|---|
| Minimize Tardiness | 减少订单延期 |
| Priority Satisfaction | 优先满足高优先级订单 |
| Prefer Higher Priority Routing | 优先选择高优先级路线 |
| Minimize Routing Cost | 优先低成本路线 |
| Prefer Higher Efficiency Routing | 优先高效率路线 |
| Minimize Setup | 减少换产 |
| Balance Load | 均衡设备负荷 |
| Minimize Waiting | 减少工序间等待 |
| Plan Stability | 重排时尽量少改动原计划 |
| Routing Stability | 重排时尽量不改变已选路线 |
| Minimize By-product Inventory | 减少副产物长期积压 |
| Prefer Internal By-product Reuse | 优先复用内部副产物，减少外购物料 |
| Avoid By-product Over-optimization | 副产物收益不能反向破坏主产品交期 |

---

# 13. AI Agent / LLM 设计

## 13.1 AI 原生原则

AI 原生 APS 不是在系统上加一个聊天框，而是把 APS 的核心能力全部工具化，让 LLM/Agent 能在权限、审计、审批保护下调用这些能力。

核心原则：

1. Agent 不直接操作数据库。
2. Agent 只能调用已注册 Tool。
3. 高风险动作必须人工确认。
4. 所有 Tool 调用必须审计。
5. Agent 不能绕过权限。
6. Agent 输出建议，不替代主管最终决策。
7. 涉及发布、作废、回滚、批量调整必须审批。

## 13.2 Agent 类型

| Agent | 职责 |
|---|---|
| Scheduling Copilot | 调度助手，自然语言查询和操作入口 |
| Plan Explanation Agent | 解释排程结果和延期原因 |
| Routing Explanation Agent | 解释为什么选择某条工艺路线 |
| Process Graph Explanation Agent | 解释工艺 DAG、并行、汇合、等待原因 |
| By-product Flow Agent | 解释副产物产生时间、数量、流向和消耗关系 |
| What-if Simulation Agent | 做假设模拟，例如设备停机、插单、路线切换 |
| Constraint Diagnosis Agent | 诊断约束冲突和不可排原因 |
| Data Quality Agent | 检查基础数据缺失、工艺图异常 |
| Replan Agent | 生成局部重排建议 |
| Publish Risk Agent | 发布前检查风险 |

## 13.3 Tool 分类

### 查询类工具

```text
get_order_status
get_schedule_plan
get_gantt_tasks
get_machine_load
get_delayed_orders
get_bottleneck_resources
get_task_conflicts
get_plan_score
get_execution_deviation
get_routing_candidates
get_selected_routing
get_task_graph
get_task_dependencies
get_planned_outputs
get_byproduct_flow
```

### 分析类工具

```text
analyze_order_delay_reason
analyze_machine_bottleneck
analyze_plan_risk
analyze_data_quality
compare_schedule_plans
explain_score
explain_constraint_violations
explain_routing_selection
compare_routing_candidates
explain_process_graph_waiting
explain_byproduct_availability
```

### 模拟类工具

```text
simulate_machine_downtime
simulate_urgent_order_insertion
simulate_due_date_change
simulate_overtime
simulate_resource_removal
simulate_material_delay
simulate_routing_change
simulate_ignore_byproduct_reuse
```

### 操作类工具

这些必须有权限和审批：

```text
create_schedule_plan
start_full_solve
start_partial_replan
pin_tasks
unpin_tasks
move_task
change_task_machine
select_order_routing
change_order_routing
freeze_time_window
create_plan_version
submit_plan_for_approval
publish_schedule_plan
rollback_schedule_plan
```

## 13.4 示例：工艺图等待解释

用户问：

```text
为什么这个工序不能提前？
```

AI 应回答：

```text
该工序属于 JOIN_ALL 汇合节点，需要等待前置工序 A1 和 A2 都完成后才能开始。
当前 A1 预计 10:30 完成，A2 预计 11:20 完成，因此该工序最早只能在 11:20 加上搬运时间后开始。
```

## 13.5 示例：副产物解释

用户问：

```text
这个副产物什么时候可以作为饲料原料使用？
```

AI 应回答：

```text
该副产物由任务 TASK-120 的分离工序产生，预计在 2026-05-01 14:30 完成。
根据工艺规则，该副产物需要经过 2 小时检验等待，因此最早可用时间为 2026-05-01 16:30。
当前预计产出 320kg，质量等级为 B，可用于饲料原料订单 FEED-009。
```

---

# 14. 技术架构

## 14.1 推荐技术路线

第一阶段建议采用模块化单体架构，而不是一开始拆微服务。

推荐技术栈：

| 层 | 技术 |
|---|---|
| 后端 | Java 17+ / Spring Boot |
| 架构 | DDD Modular Monolith |
| 数据库 | PostgreSQL |
| 缓存 | Redis |
| 消息 | RabbitMQ / Kafka |
| 求解器 | Timefold |
| 前端 | Vue / React |
| 甘特图 | 专业甘特图组件或自研 Canvas/SVG |
| 工艺图编辑器 | SVG / Canvas / React Flow / AntV X6 等 |
| AI Gateway | LLM Client + Tool Registry |
| 权限 | RBAC + 数据权限 |
| 审计 | 操作日志 + Agent 工具调用日志 |

## 14.2 模块结构

```text
aps-server
├── aps-factory
├── aps-process
├── aps-demand
├── aps-routing-selection
├── aps-scheduling
├── aps-solver
├── aps-execution
├── aps-agent
├── aps-integration
├── aps-governance
└── aps-common
```

## 14.3 应用服务设计

### ProcessApplicationService

```java
createRouting(command)
updateProcessGraph(command)
validateProcessGraph(command)
createOperationNode(command)
createOperationEdge(command)
defineOperationInput(command)
defineOperationOutput(command)
```

### RoutingSelectionApplicationService

```java
generateRoutingCandidates(command)
selectRouting(command)
changeRouting(command)
simulateRoutingChange(command)
explainRoutingSelection(query)
compareRoutingCandidates(query)
```

### SchedulingApplicationService

```java
createSchedulePlan(command)
generateTaskGraph(command)
generateOperationTasks(command)
startFullSolve(command)
startPartialReplan(command)
adjustTask(command)
pinTask(command)
unpinTask(command)
submitForApproval(command)
publishPlan(command)
rollbackPlan(command)
```

### SolverApplicationService

```java
createSolverJob(command)
startSolver(jobId)
cancelSolver(jobId)
getSolverStatus(jobId)
getScoreExplanation(versionId)
validateMove(command)
```

### AgentApplicationService

```java
createAgentSession(command)
handleUserMessage(command)
invokeTool(command)
requestApproval(command)
confirmAgentAction(command)
rejectAgentAction(command)
```

---

# 15. 数据库设计草案

## 15.1 基础建模表

```text
factory
workshop
work_center
machine
resource_capability
calendar
shift
calendar_exception
```

## 15.2 工艺与订单表

```text
product
routing
operation_node
operation_edge
operation_input
operation_output
routing_applicable_rule
production_order
bom
material
material_availability
```

### operation_node

```text
operation_node
├── id
├── routing_id
├── operation_code
├── operation_name
├── operation_type
├── processing_time_rule
├── setup_time_rule
├── status
└── remark
```

### operation_edge

```text
operation_edge
├── id
├── routing_id
├── predecessor_operation_node_id
├── successor_operation_node_id
├── relation_type
├── min_lag
├── max_lag
├── transfer_time
└── dependency_condition
```

### operation_input

```text
operation_input
├── id
├── operation_node_id
├── item_id
├── item_type
├── quantity
├── unit
├── consume_timing
└── required_quality_level
```

### operation_output

```text
operation_output
├── id
├── operation_node_id
├── item_id
├── item_type
├── quantity_rule
├── unit
├── output_timing
├── quality_level
├── storage_location_rule
└── available_for_reuse
```

## 15.3 多工艺路线选择表

```text
routing_candidate_snapshot
order_routing_assignment
```

### routing_candidate_snapshot

```text
routing_candidate_snapshot
├── id
├── plan_version_id
├── order_id
├── routing_id
├── priority
├── applicable
├── unavailable_reason
├── cost_factor
├── efficiency_factor
├── quality_level
├── estimated_duration
├── estimated_resource_load
├── estimated_risk
└── created_at
```

### order_routing_assignment

```text
order_routing_assignment
├── id
├── plan_version_id
├── order_id
├── selected_routing_id
├── selection_mode
├── selection_reason
├── priority_penalty
├── score_impact
├── created_at
└── created_by
```

## 15.4 排程核心表

```text
schedule_plan
schedule_version
operation_task
task_dependency
task_assignment
planned_output
schedule_publish_record
schedule_approval_record
schedule_change_log
```

### task_dependency

```text
task_dependency
├── id
├── plan_version_id
├── predecessor_task_id
├── successor_task_id
├── relation_type
├── min_lag
├── max_lag
├── transfer_time
└── condition
```

### planned_output

```text
planned_output
├── id
├── plan_version_id
├── task_id
├── order_id
├── item_id
├── item_type
├── quantity
├── unit
├── available_time
├── quality_level
├── storage_location
├── reusable
└── consumed_by_task_id
```

## 15.5 求解器表

```text
solver_job
solver_config
constraint_config
score_detail
constraint_violation
```

## 15.6 执行反馈表

```text
execution_feedback
machine_event
task_actual_record
deviation_record
replan_trigger_record
```

## 15.7 Agent 表

```text
agent_session
agent_message
agent_task
tool_invocation
agent_approval_request
agent_audit_log
```

---

# 16. API 接口设计

## 16.1 工艺路线与工艺图接口

```http
POST /api/routings
GET  /api/routings/{routingId}
POST /api/routings/{routingId}/process-graph
POST /api/routings/{routingId}/operation-nodes
POST /api/routings/{routingId}/operation-edges
POST /api/routings/{routingId}/validate-graph
POST /api/operation-nodes/{nodeId}/inputs
POST /api/operation-nodes/{nodeId}/outputs
```

## 16.2 多工艺路线接口

```http
POST /api/orders/{orderId}/routing-candidates
POST /api/orders/{orderId}/select-routing
POST /api/orders/{orderId}/simulate-routing-change
GET  /api/orders/{orderId}/routing-selection-explanation
```

## 16.3 TaskGraph 接口

```http
POST /api/orders/{orderId}/task-graph
GET  /api/orders/{orderId}/task-graph
GET  /api/task-graphs/{taskGraphId}/dependencies
GET  /api/task-graphs/{taskGraphId}/planned-outputs
```

## 16.4 排程方案接口

```http
POST /api/schedule-plans
GET  /api/schedule-plans/{planId}
POST /api/schedule-plans/{planId}/generate-tasks
POST /api/schedule-plans/{planId}/solve
POST /api/schedule-plans/{planId}/partial-replan
POST /api/schedule-plans/{planId}/submit-approval
POST /api/schedule-plans/{planId}/publish
POST /api/schedule-plans/{planId}/rollback
```

## 16.5 甘特图接口

```http
GET  /api/gantt/resource-view
GET  /api/gantt/order-view
POST /api/gantt/tasks/{taskId}/move
POST /api/gantt/tasks/{taskId}/change-machine
POST /api/gantt/tasks/{taskId}/pin
POST /api/gantt/tasks/{taskId}/unpin
POST /api/gantt/validate-move
```

## 16.6 Agent 接口

```http
POST /api/agent/sessions
POST /api/agent/sessions/{sessionId}/messages
POST /api/agent/tasks/{taskId}/confirm
POST /api/agent/tasks/{taskId}/reject
GET  /api/agent/tools
GET  /api/agent/audit-logs
```

---

# 17. 页面设计

## 17.1 工艺路线编辑器

系统应提供图形化工艺路线编辑能力，支持：

1. 添加工序节点。
2. 拖拽连接工序依赖边。
3. 设置 FS、SS、FF 等关系。
4. 设置并行分叉。
5. 设置多工序汇合。
6. 设置工序输入物。
7. 设置工序输出物。
8. 标记主产品、副产物、联产品。
9. 校验工艺路线是否为有向无环图。
10. 校验是否存在孤立节点、循环依赖、无入口、无出口等问题。

## 17.2 排程工作台

核心页面布局：

```text
左侧：资源树 / 订单树
中间：甘特图
右侧：任务详情 / 冲突解释 / 路线选择解释 / 工艺图解释 / AI 建议
顶部：时间范围、方案版本、求解按钮、发布按钮
底部：风险列表、延期列表、评分解释、副产物流向
```

## 17.3 订单路线选择面板

展示：

```text
当前选择工艺路线：Routing-B
路线优先级：2
选择方式：系统自动选择
选择原因：Routing-A 对应瓶颈设备 M01 已满载，使用 Routing-B 可减少延期 240 分钟

替代路线：
- Routing-A：优先级 1，预计延期 360 分钟
- Routing-B：优先级 2，预计延期 120 分钟
- Routing-C：优先级 3，预计延期 0 分钟，但成本较高
```

## 17.4 工艺任务图视图

展示：

1. 订单的 TaskGraph。
2. 并行工序。
3. 汇合等待关系。
4. 分叉输出关系。
5. 主产品、副产物、联产品。
6. 副产物被哪个后续任务消耗。

---

# 18. MVP 一期范围

## 18.1 一期必须做

```text
工厂 / 车间 / 工作中心 / 设备
设备日历 / 班次
产品 / 工艺路线 / 工序节点 / 工序边
基础 ProcessGraph 建模
多工艺路线候选规则
生产订单
候选工艺路线生成
基础路线选择
TaskGraph 生成
DAG 依赖约束
基础 OperationOutput / PlannedOutput
设备能力匹配
自动排程
资源甘特图
订单甘特图
任务拖拽
冲突检测
任务锁定
排程方案版本
计划发布
MES 基础反馈
AI 查询助手
AI 方案解释
AI 路线选择解释
AI 工艺图等待解释
AI 副产物基础解释
```

## 18.2 一期暂缓

```text
复杂物料齐套
复杂人员排班
复杂模具共享
外协排程
多工厂协同
复杂批处理
自动采购建议
成本优化
完全动态实体数量的工艺路线求解
复杂副产物库存优化
```

---

# 19. 验收标准

## 19.1 基础排程验收

1. 系统可以导入生产订单。
2. 系统可以根据产品生成候选工艺路线。
3. 系统可以根据选定工艺路线生成 TaskGraph。
4. 系统可以自动分配设备和时间。
5. 同一设备任务不重叠。
6. 后道工序不早于前道工序。
7. 汇合工序必须等待所有必要前置工序完成。
8. 任务不能安排到停机时间。
9. 排程结果可以在甘特图展示。

## 19.2 工艺图验收

1. 工艺路线可以包含多个入口工序。
2. 工艺路线可以包含多个出口工序。
3. 工艺路线可以包含串行关系。
4. 工艺路线可以包含并行关系。
5. 工艺路线可以包含分叉关系。
6. 工艺路线可以包含汇合关系。
7. 系统能校验循环依赖并阻止保存。
8. 系统能根据 ProcessGraph 生成 TaskGraph。

## 19.3 多产出验收

1. 工序可以定义主产品输出。
2. 工序可以定义副产物输出。
3. 工序可以定义联产品输出。
4. 副产物可设置是否可复用。
5. 副产物可生成 planned_output。
6. 后续任务消耗副产物时必须等待副产物可用。
7. 副产物消耗数量不能超过产生数量。

## 19.4 多工艺路线验收

1. 一个产品可以配置多条工艺路线。
2. 每条工艺路线可以配置优先级。
3. 系统可以根据订单条件筛选候选路线。
4. 不满足适用条件的路线不能被选择。
5. 停用路线不能被选择。
6. 自动选择模式下优先选择高优先级路线。
7. 当高优先级路线导致严重延期时，系统可选择替代路线。
8. 路线选择结果必须写入排程版本。
9. 用户可以查看路线选择原因。
10. 已发布计划中的路线不能直接覆盖修改。

## 19.5 AI 能力验收

1. 用户可以询问订单延期原因。
2. 用户可以询问为什么选择某条工艺路线。
3. 用户可以询问为什么某个工序不能提前。
4. 用户可以询问哪个前置工序导致汇合等待。
5. 用户可以询问某个副产物什么时候可用。
6. 用户可以询问副产物被哪些订单消耗。
7. AI 调用工具必须有审计日志。
8. AI 执行高风险动作前必须请求确认。

---

# 20. 推荐实施路线

## 阶段一：MVP 主链路

目标：

```text
订单 → 候选工艺路线 → ProcessGraph → TaskGraph → 自动排程 → 甘特图 → 发布 → 反馈
```

重点：

1. 设备有限产能排程。
2. 多工艺路线基础选择。
3. 基础 DAG 工艺图。
4. 甘特图。
5. Timefold 基础求解。
6. AI 查询与解释。

## 阶段二：闭环与局部重排

重点：

1. MES 反馈。
2. 偏差分析。
3. 局部重排。
4. What-if 模拟。
5. 路线切换模拟。
6. 方案对比。
7. 副产物可用性分析。

## 阶段三：高级约束

重点：

1. 物料齐套。
2. 模具共享。
3. 人员技能。
4. 换产矩阵。
5. 多目标评分。
6. 副产物库存优化。

## 阶段四：AI 原生增强

重点：

1. 自动发现风险。
2. 自动生成重排建议。
3. 自动解释方案差异。
4. 自动解释路线选择差异。
5. 自动解释工艺图等待原因。
6. 自动解释副产物流向。
7. 多智能体协作。

---

# 21. 关键架构结论

## 21.1 DDD 是底座

APS 不是 CRUD 系统。

如果不用 DDD，很容易变成：

```text
一堆订单表
一堆任务表
一堆状态字段
一堆 if else
最后谁也不敢改
```

## 21.2 工艺路线必须从线性列表升级为 ProcessGraph

核心结论：

```text
工艺路线不能只用 sequenceNo 表示，必须支持 ProcessGraph。
工序之间不能只支持简单前后顺序，必须支持 DAG 依赖。
一个工艺路线可以有多个入口工序，也可以有多个出口工序。
工序输出不一定只是主产品，也可能是副产物、联产品、半成品。
```

## 21.3 Timefold 是求解核心

Timefold 负责：

```text
硬约束
软约束
评分
搜索
优化
重排
DAG 依赖约束
多路线选择约束
副产物可用性约束
```

## 21.4 LLM 是协同入口

LLM 负责：

```text
理解
解释
诊断
建议
调用工具
生成报告
人机协同
```

## 21.5 多工艺路线的核心原则

```text
硬约束决定能不能选
软约束决定优先选哪条
Timefold 负责计算整体最优
LLM 负责解释为什么这样选
DDD 负责记录选择过程、版本和业务一致性
```

最重要的一句话：

> 不要把“优先级高的工艺路线”做成硬约束，而应该做成软约束；否则系统会为了死守高优先级路线，排不出更好的整体计划。

## 21.6 副产物建模原则

```text
主产品交付是核心目标
副产物复用是优化目标
副产物可用性是物料约束
副产物收益不能反向破坏主产品交期
```

---

# 22. 结论

本系统应按照以下路径建设：

```text
先做强 APS 领域模型
再做 ProcessGraph / TaskGraph
再做稳定 Timefold 求解器
再做甘特图调度工作台
再做执行反馈闭环
最后把核心能力开放给 Agent
```

AI-Native APS 的本质不是聊天，而是：

```text
所有业务能力都工具化
所有工具都有权限
所有操作都有审计
所有高风险动作都要确认
所有结果都能解释
```

这样做出来的系统，才不是一个“会聊天的排程系统”，而是一个真正可以长期演进的智能排程平台。
