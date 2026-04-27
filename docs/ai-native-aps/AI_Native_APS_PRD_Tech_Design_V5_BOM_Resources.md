# AI-Native APS 高级计划与排程系统 PRD + DDD 技术设计文档 V5.0

> 文档定位：产品需求 PRD + DDD 领域设计 + 技术架构 + Timefold 求解器设计 + LLM/Agent 设计。  
> 本版本重点补充：成品/半成品/物料统一 Item 模型、成品与半成品 BOM、多套物料 BOM 有限选择、BOM 优先级、设备/人员/模具夹具资源、生产节拍、换模换产时间、只排设备/只排人/设备人员同时排等能力。

---

# 0. 核心结论

APS 不能只围绕“订单、工艺路线、设备”建模。工业级 APS 至少要同时表达：

```text
成品 / 半成品 / 物料
  ↓
物料 BOM / 多套 BOM / BOM 优先级
  ↓
工艺路线 / 多工艺路线 / 工艺 DAG
  ↓
工序 / 串行 / 并行 / 汇合 / 分叉
  ↓
设备 / 人员 / 模具 / 夹具 / 工装
  ↓
生产节拍 / 加工时间 / 换模 / 换产 / 准备时间
  ↓
Timefold 有限产能排程
  ↓
甘特图调度 / AI 解释 / MES 执行反馈
```

关键原则：

1. **成品和半成品都有 BOM**。
2. **物料没有 BOM**，物料是被消耗对象。
3. 一个成品或半成品可以有多套物料 BOM。
4. 多套 BOM 之间可以有限选择。
5. BOM 优先级应作为软约束，不应简单写死。
6. 工艺路线优先级、BOM 优先级都应在满足硬约束的前提下优化选择。
7. 生产资源不只有设备，还包括人员、模具、夹具、工装、工具。
8. APS 应支持只排设备、只排人员、设备和人员同时排、设备人员模具一起排。
9. 生产节拍、加工时间、换模时间、换产时间、准备时间都可能占用资源时间。
10. Timefold 负责严肃约束求解，LLM/Agent 负责解释、诊断、模拟和协同。

---

# 1. Item 主数据模型：成品、半成品、物料

## 1.1 为什么需要统一 Item 模型

系统中不能只用“产品”一个概念表达所有对象。真实生产中至少存在：

| 类型 | 说明 |
|---|---|
| 成品 | 最终对外售卖和交付的产品 |
| 半成品 | 内部生产出来，被后续工序或其他订单消耗的中间产物 |
| 物料 | 外购或库存消耗对象，例如原材料、辅料、包材 |
| 副产物 | 生产过程中附带产生，可复用或销售的附加产出 |
| 联产品 | 与主产品共同产生，也具备计划价值的产出 |

建议统一建模为 `Item`，通过 `itemType` 区分业务语义。

```text
Item
├── itemId
├── itemCode
├── itemName
├── itemType
├── specification
├── unit
├── status
├── shelfLife
├── qualityGrade
├── inventoryManaged
├── purchasable
├── producible
├── sellable
└── remark
```

## 1.2 itemType

```text
FINISHED_GOOD      成品
SEMI_FINISHED      半成品
RAW_MATERIAL       原材料
AUXILIARY_MATERIAL 辅料
PACKAGING_MATERIAL 包材
BY_PRODUCT         副产物
CO_PRODUCT         联产品
WASTE              废料
```

## 1.3 成品 Finished Good

成品是最终对外售卖和交付的产品。

```text
FinishedGood
├── itemType = FINISHED_GOOD
├── sellable = true
├── producible = true
├── inventoryManaged = true
└── canBeProductionOrderTarget = true
```

成品通常可以：

1. 被销售订单引用。
2. 被生产订单作为目标产出。
3. 拥有一套或多套物料 BOM。
4. 拥有一条或多条工艺路线。
5. 入成品库存。

## 1.4 半成品 Semi-finished Good

半成品是内部生产出来的中间产物，不一定直接销售，通常用于后续工序或其他生产订单。

```text
SemiFinishedGood
├── itemType = SEMI_FINISHED
├── sellable = false，特殊场景可配置 true
├── producible = true
├── inventoryManaged = true 或 false
└── canBeIntermediateOutput = true
```

半成品通常可以：

1. 由某个工序产出。
2. 被后续工序消耗。
3. 临时入库或直接流转。
4. 跨订单、跨工艺路线复用。
5. 拥有自己的物料 BOM。
6. 拥有自己的工艺路线。

## 1.5 物料 Material

物料是生产过程中被消耗的对象，通常包括原材料、辅料、包材等。

```text
Material
├── itemType = RAW_MATERIAL / AUXILIARY_MATERIAL / PACKAGING_MATERIAL
├── sellable = false
├── producible = false，通常不由本厂生产
├── purchasable = true
└── inventoryManaged = true
```

物料通常：

1. 没有 BOM。
2. 不作为 APS 的生产目标。
3. 来自采购、库存、外部供应。
4. 约束工序是否可以开工。
5. 可配置替代料规则、批次、质量等级和预计到货时间。

---

# 2. BOM 模型：成品和半成品都有 BOM，物料没有 BOM

## 2.1 BOM 是什么

BOM，Bill of Materials，物料清单。

在 APS 中，BOM 描述的是：

> 生产一个成品或半成品，需要消耗哪些物料、半成品、副产物或替代料，以及各自的数量、损耗率和适用条件。

BOM 不是工艺路线。

| 对象 | 说明 |
|---|---|
| BOM | 说明生产目标需要消耗什么 |
| 工艺路线 | 说明怎么生产、经过哪些工序 |
| 资源模型 | 说明用什么设备、人、模具、夹具生产 |
| 排程模型 | 说明什么时候生产、由谁生产、用什么资源生产 |

## 2.2 谁有 BOM

| Item 类型 | 是否有 BOM | 说明 |
|---|---|---|
| 成品 | 有 | 生产成品需要物料和半成品 |
| 半成品 | 有 | 生产半成品也需要物料或更低层半成品 |
| 原材料 | 无 | 原材料通常来自采购，不再分解 |
| 辅料 | 无 | 通常为消耗物料 |
| 包材 | 无 | 通常为采购或库存物料 |
| 副产物 | 一般无 | 通常由工序附带产生，不作为常规生产目标；特殊业务可配置 |
| 联产品 | 可有 | 如果联产品也可独立生产，则可有 BOM |

关键结论：

```text
成品和半成品都可以有 BOM。
物料本身没有 BOM。
```

## 2.3 Material BOM 物料 BOM

物料 BOM 表示生产某个成品或半成品需要消耗的物料结构。

```text
MaterialBom
├── bomId
├── targetItemId
├── targetItemType
├── bomCode
├── bomName
├── version
├── status
├── priority
├── effectiveDate
├── expireDate
├── applicableRules
└── bomLines
```

说明：

1. `targetItemId` 可以是成品，也可以是半成品。
2. `targetItemType` 通常为 `FINISHED_GOOD` 或 `SEMI_FINISHED`。
3. 物料本身不作为 `targetItemId` 建 BOM。

## 2.4 BOMLine

```text
BomLine
├── bomLineId
├── bomId
├── componentItemId
├── componentItemType
├── quantity
├── unit
├── lossRate
├── substituteGroupId
├── requiredQualityLevel
├── issueTiming
└── remark
```

componentItemType 可以是：

```text
RAW_MATERIAL
AUXILIARY_MATERIAL
PACKAGING_MATERIAL
SEMI_FINISHED
BY_PRODUCT
CO_PRODUCT
```

issueTiming 表示投料时机：

```text
AT_ORDER_START
AT_OPERATION_START
DURING_OPERATION
AT_OPERATION_END
```

## 2.5 多套 BOM 有限选择

一个成品或半成品可能有多套 BOM。

例如：

```text
产品 A：
- BOM-A：标准物料方案，优先级 1，成本最低
- BOM-B：替代物料方案，优先级 2，使用替代料
- BOM-C：应急物料方案，优先级 3，成本高但库存充足
```

系统应支持：

1. 一个目标 Item 配置多套 BOM。
2. 每套 BOM 配置优先级。
3. 每套 BOM 配置适用条件。
4. APS 在满足硬约束的前提下优先选择优先级最高的 BOM。
5. 当高优先级 BOM 物料不足、质量不匹配、预计到料晚时，可以选择低优先级 BOM。
6. BOM 选择结果必须记录到排程方案版本中。

## 2.6 BOMApplicableRule

```text
BomApplicableRule
├── bomId
├── minQuantity
├── maxQuantity
├── customerScope
├── orderTypeScope
├── factoryScope
├── workshopScope
├── materialCondition
├── qualityRequirement
├── effectiveDate
├── expireDate
└── requiredApproval
```

## 2.7 OrderBomAssignment

```text
OrderBomAssignment
├── assignmentId
├── orderId
├── targetItemId
├── selectedBomId
├── selectionMode
├── selectionReason
├── priorityPenalty
├── materialAvailabilityImpact
├── scoreImpact
├── planVersionId
└── createdAt
```

selectionMode：

```text
AUTO_SELECTED
MANUALLY_SPECIFIED
EXTERNALLY_SPECIFIED
SIMULATION_SELECTED
```

## 2.8 BOM 选择约束

### 硬约束

| 约束 | 说明 |
|---|---|
| BOM 必须属于目标 Item | 生产订单只能选择目标成品或半成品允许的 BOM |
| BOM 必须启用 | 停用 BOM 不能选择 |
| BOM 必须在有效期内 | 未生效或已失效 BOM 不能选择 |
| BOM 必须满足适用条件 | 数量、客户、订单类型、工厂范围必须匹配 |
| BOM 物料必须可用 | 关键物料不足或不可用时不能开工 |
| BOM 质量必须满足 | 物料质量等级必须满足要求 |
| 已发布 BOM 不可直接修改 | 已发布计划中的 BOM 变更必须生成新版本 |

### 软约束

| 约束 | 说明 |
|---|---|
| 优先选择高优先级 BOM | priority 越高，扣分越少 |
| 优先选择低成本 BOM | 成本更低优先 |
| 优先选择物料齐套 BOM | 齐套时间更早优先 |
| 优先减少替代料使用 | 标准料优先于替代料 |
| 保持 BOM 稳定 | 重排时尽量不改变已选 BOM |

评分建议：

```text
bomPriorityPenalty = (bomPriority - 1) × bomPriorityWeight
```

关键结论：

> BOM 优先级也不应该做成硬约束，而应该做成软约束。硬约束决定 BOM 能不能用，软约束决定优先用哪套 BOM。

---

# 3. 工艺路线与 BOM 的关系

BOM 和工艺路线是两个不同维度，但在排程时必须组合使用。

```text
生产订单
  ↓
选择目标 Item
  ↓
选择 BOM：决定消耗什么
  ↓
选择工艺路线：决定怎么生产
  ↓
生成 TaskGraph：决定有哪些工序任务
  ↓
绑定资源需求：决定需要设备、人、模具、夹具
  ↓
进入 Timefold 排程
```

一个订单最终应记录：

```text
OrderPlanSelection
├── orderId
├── targetItemId
├── selectedBomId
├── selectedRoutingId
├── selectedResourceMode
├── planVersionId
└── selectionReason
```

## 3.1 BOM 与工艺路线组合选择

同一个产品可能存在：

```text
BOM-A + Routing-A
BOM-A + Routing-B
BOM-B + Routing-A
BOM-B + Routing-C
```

需要支持组合可用性规则。

```text
BomRoutingCompatibilityRule
├── targetItemId
├── bomId
├── routingId
├── compatible
├── reason
└── priorityAdjustment
```

示例：

```text
BOM-B 使用替代材料，只能走 Routing-C。
BOM-A 使用标准材料，可以走 Routing-A 或 Routing-B。
```

---

# 4. 工艺路线 DAG 与工序建模

一条工艺路线不是简单线性列表，而是 ProcessGraph。

```text
Routing
├── routingId
├── targetItemId
├── priority
└── processGraph
```

```text
ProcessGraph
├── nodes: List<OperationNode>
├── edges: List<OperationEdge>
├── inputs: List<OperationInput>
└── outputs: List<OperationOutput>
```

工序关系支持：

| 类型 | 含义 |
|---|---|
| FS | 前工序完成后，后工序才能开始 |
| SS | 前工序开始后，后工序才能开始 |
| FF | 前工序完成后，后工序才能完成 |
| JOIN_ALL | 多个前置工序全部完成后才能开始 |
| JOIN_ANY | 多个前置工序任一完成后即可开始 |
| SPLIT | 一个工序完成后分叉到多个后续工序 |

示例：

```text
A1 ┐
   ├→ C → D
A2 ┘
```

含义：A1 和 A2 可以并行开始，C 必须等 A1 和 A2 都完成后才能开始。

---

# 5. 工序资源需求：设备、人、模具、夹具、工装

## 5.1 资源类型

APS 不能只排设备。生产一个工序可能同时需要：

| 资源 | 说明 |
|---|---|
| 设备 Machine | 机床、产线、工位、加工设备 |
| 人员 Worker | 操作工、检验员、技术员 |
| 班组 Team | 多人协作资源 |
| 模具 Mold | 模具、型腔、模架 |
| 夹具 Fixture | 装夹工具、定位夹具 |
| 工装 Tooling | 专用工装、刀具、治具 |
| 辅助资源 Auxiliary Resource | 行车、叉车、检测设备等 |

统一建模：

```text
Resource
├── resourceId
├── resourceCode
├── resourceName
├── resourceType
├── status
├── calendarId
├── capacity
├── capabilityTags
└── costRate
```

resourceType：

```text
MACHINE
WORKER
TEAM
MOLD
FIXTURE
TOOLING
AUXILIARY
```

## 5.2 工序资源需求 OperationResourceRequirement

```text
OperationResourceRequirement
├── requirementId
├── operationNodeId
├── resourceType
├── requiredCapability
├── quantity
├── selectionMode
├── required
├── occupationMode
├── occupationStart
├── occupationEnd
└── priority
```

selectionMode：

```text
AUTO_SELECT       系统自动选择
FIXED_RESOURCE    固定资源
CANDIDATE_SET     候选资源集合
```

occupationMode：

```text
FULL_PROCESS      占用整个加工过程
SETUP_ONLY        只在准备/换模阶段占用
PROCESS_ONLY      只在生产阶段占用
TEARDOWN_ONLY     只在拆卸阶段占用
PARTIAL           部分时间占用
```

## 5.3 排程模式

系统应支持不同排程模式：

| 模式 | 说明 |
|---|---|
| MACHINE_ONLY | 只排设备，不排人员 |
| WORKER_ONLY | 只排人员，不排设备 |
| MACHINE_AND_WORKER | 同时排设备和人员 |
| MACHINE_WORKER_TOOLING | 同时排设备、人员、模具、夹具、工装 |
| CAPACITY_BUCKET | 只按产能池粗排，不分具体资源 |

每个工厂、车间、工作中心、工艺路线或工序可以配置排程模式。

---

# 6. 生产节拍、加工时间与时间构成

## 6.1 BOM 不决定生产节拍

BOM 决定消耗什么。

生产节拍、加工时间、换模换产时间属于工艺路线和工序资源能力模型。

```text
BOM：生产这个成品/半成品需要哪些物料和数量
Routing / Operation：怎么生产、用什么资源、花多少时间
Resource Capability：某资源做这个工序的效率和节拍
```

## 6.2 工序时间构成

一个工序的计划时间可由多段组成：

```text
总时长 = 准备时间 + 换模时间 + 换产时间 + 加工时间 + 检验时间 + 搬运时间 + 等待时间 + 拆卸/清理时间
```

建模：

```text
OperationTimeModel
├── operationNodeId
├── fixedDuration
├── durationPerUnit
├── taktTime
├── setupTime
├── moldChangeTime
├── changeoverTime
├── inspectionTime
├── transferTime
├── waitTime
├── teardownTime
└── timeUnit
```

## 6.3 生产节拍 Takt / Cycle Time

生产节拍表示单位产出所需时间，或者单位时间产出能力。

两种表达方式：

```text
每件耗时：durationPerUnit = 30 秒/件
单位产能：capacityRate = 120 件/小时
```

系统可换算：

```text
processingDuration = quantity × durationPerUnit / resourceEfficiency
```

或：

```text
processingDuration = quantity / capacityRate × 60 分钟
```

## 6.4 资源相关节拍

同一工序在不同设备、人员、模具组合下节拍可能不同。

```text
ResourceProcessCapability
├── resourceId
├── operationNodeId
├── itemId
├── durationPerUnit
├── capacityRate
├── efficiencyFactor
├── qualityLevel
└── priority
```

示例：

| 工序 | 设备 | 节拍 |
|---|---|---|
| 冲压 | M01 | 60 件/小时 |
| 冲压 | M02 | 45 件/小时 |
| 冲压 | M03 | 80 件/小时 |

Timefold 应根据资源选择动态计算任务时长。

---

# 7. 换模、换产、准备与清理时间

## 7.1 换模 Change Mold

换模是更换模具、夹具、工装导致的时间消耗。

```text
MoldChangeRule
├── fromMoldId
├── toMoldId
├── machineId
├── duration
├── requiredWorkerCapability
└── remark
```

## 7.2 换产 Changeover

换产是从一种产品、规格、颜色、材质、BOM、工艺路线切换到另一种时产生的时间和成本。

```text
ChangeoverRule
├── resourceId
├── fromItemId
├── toItemId
├── fromAttribute
├── toAttribute
├── duration
├── cost
└── priority
```

换产维度可包括：

```text
产品
规格
颜色
材质
客户
BOM
工艺路线
模具
清洁等级
质量等级
```

## 7.3 准备时间 Setup Time

准备时间可能包括：

1. 领料。
2. 上料。
3. 调机。
4. 首件检验。
5. 工装安装。
6. 程序切换。
7. 清洁准备。

准备时间可能占用：

1. 设备。
2. 人员。
3. 模具。
4. 夹具。
5. 检验资源。

## 7.4 时间段拆分

一个任务可拆成多个内部时间段：

```text
TaskSegment
├── segmentId
├── taskId
├── segmentType
├── startTime
├── endTime
├── occupiedResources
└── duration
```

segmentType：

```text
SETUP
MOLD_CHANGE
CHANGEOVER
PROCESSING
INSPECTION
TRANSFER
WAITING
TEARDOWN
CLEANING
```

这样可以支持：

1. 换模阶段只占用人员和模具。
2. 加工阶段占用设备和人员。
3. 检验阶段占用检验员和检测设备。
4. 清理阶段占用设备但不占用生产人员。

---

# 8. Timefold 求解器建模补充

## 8.1 Planning Solution

```java
ScheduleSolution
├── List<Item>
├── List<MaterialBom>
├── List<OrderBomAssignment>
├── List<OrderRoutingAssignment>
├── List<Resource>
├── List<OperationTaskAssignment>
├── List<TaskDependency>
├── List<TaskSegment>
├── List<PlannedOutput>
└── HardSoftScore
```

## 8.2 Planning Entity

### OrderBomAssignment

```java
OrderBomAssignment
├── orderId
├── targetItemId
├── candidateBoms
├── selectedBom
├── bomPriority
└── bomPenalty
```

### OrderRoutingAssignment

```java
OrderRoutingAssignment
├── orderId
├── candidateRoutings
├── selectedRouting
├── routingPriority
└── routingPenalty
```

### OperationTaskAssignment

```java
OperationTaskAssignment
├── taskId
├── orderId
├── selectedBomId
├── selectedRoutingId
├── operationNodeId
├── quantity
├── assignedMachine
├── assignedWorkers
├── assignedMold
├── assignedFixtures
├── assignedToolings
├── startTime
├── endTime
├── taskSegments
└── pinned
```

## 8.3 硬约束

| 约束 | 说明 |
|---|---|
| BOM Belongs To Target Item | BOM 必须属于目标成品或半成品 |
| BOM Applicable | BOM 必须满足适用条件 |
| Material Availability | BOM 所需物料必须可用 |
| Routing Belongs To Target Item | 工艺路线必须属于目标成品或半成品 |
| Routing Applicable | 工艺路线必须满足适用条件 |
| Bom Routing Compatible | 所选 BOM 与所选工艺路线必须兼容 |
| Resource Conflict | 同一设备/人员/模具/夹具同一时间不能超量占用 |
| Resource Capability | 资源必须具备工序所需能力 |
| Resource Calendar | 任务不能安排在资源不可用时间 |
| Task Graph Dependency | 满足工艺 DAG 依赖 |
| Semi-finished Availability | 半成品未产出或库存不足前不能消耗 |
| Tooling Availability | 模具、夹具、工装库存或容量不能超限 |
| Pinned Task | 锁定任务不能被自动修改 |

## 8.4 软约束

| 约束 | 说明 |
|---|---|
| Prefer Higher Priority BOM | 优先选择高优先级 BOM |
| Prefer Higher Priority Routing | 优先选择高优先级工艺路线 |
| Minimize Tardiness | 减少订单延期 |
| Minimize Setup | 减少准备时间 |
| Minimize Mold Change | 减少换模时间 |
| Minimize Changeover | 减少换产时间 |
| Balance Machine Load | 均衡设备负荷 |
| Balance Worker Load | 均衡人员负荷 |
| Prefer Higher Efficiency Resource | 优先选择高效率资源 |
| Plan Stability | 重排时尽量少改动原计划 |
| BOM Stability | 重排时尽量不改变已选 BOM |
| Routing Stability | 重排时尽量不改变已选工艺路线 |

---

# 9. 数据库表补充

## 9.1 material_bom

```text
material_bom
├── id
├── target_item_id
├── target_item_type
├── bom_code
├── bom_name
├── version
├── status
├── priority
├── effective_date
├── expire_date
└── remark
```

## 9.2 bom_line

```text
bom_line
├── id
├── bom_id
├── component_item_id
├── component_item_type
├── quantity
├── unit
├── loss_rate
├── substitute_group_id
├── required_quality_level
├── issue_timing
└── remark
```

## 9.3 order_bom_assignment

```text
order_bom_assignment
├── id
├── plan_version_id
├── order_id
├── target_item_id
├── selected_bom_id
├── selection_mode
├── selection_reason
├── priority_penalty
├── material_availability_impact
├── score_impact
├── created_at
└── created_by
```

## 9.4 resource

```text
resource
├── id
├── resource_code
├── resource_name
├── resource_type
├── status
├── calendar_id
├── capacity
├── capability_tags
└── cost_rate
```

## 9.5 operation_resource_requirement

```text
operation_resource_requirement
├── id
├── operation_node_id
├── resource_type
├── required_capability
├── quantity
├── selection_mode
├── required
├── occupation_mode
├── occupation_start
├── occupation_end
└── priority
```

## 9.6 operation_time_model

```text
operation_time_model
├── id
├── operation_node_id
├── fixed_duration
├── duration_per_unit
├── takt_time
├── setup_time
├── mold_change_time
├── changeover_time
├── inspection_time
├── transfer_time
├── wait_time
├── teardown_time
└── time_unit
```

## 9.7 task_segment

```text
task_segment
├── id
├── task_id
├── segment_type
├── start_time
├── end_time
├── duration
└── occupied_resources
```

---

# 10. API 补充

## 10.1 BOM 接口

```http
POST /api/boms
GET  /api/boms/{bomId}
GET  /api/items/{itemId}/boms
POST /api/orders/{orderId}/bom-candidates
POST /api/orders/{orderId}/select-bom
POST /api/orders/{orderId}/simulate-bom-change
GET  /api/orders/{orderId}/bom-selection-explanation
```

## 10.2 资源需求接口

```http
POST /api/operation-nodes/{nodeId}/resource-requirements
GET  /api/operation-nodes/{nodeId}/resource-requirements
POST /api/operation-nodes/{nodeId}/time-model
GET  /api/operation-nodes/{nodeId}/time-model
```

## 10.3 换模换产接口

```http
POST /api/changeover-rules
GET  /api/changeover-rules
POST /api/mold-change-rules
GET  /api/mold-change-rules
```

---

# 11. AI Agent 补充能力

用户可以问：

```text
这个成品用了哪套 BOM？
为什么没有选择优先级最高的 BOM？
这个半成品需要哪些物料？
这个物料缺少会影响哪些订单？
这道工序到底需要设备还是人员？
如果只排设备不排人，结果会有什么风险？
为什么这两个任务之间有 40 分钟换模时间？
如果换成另一个模具，交期会不会更好？
这次延期是因为设备不足、人员不足、模具不足还是物料不足？
```

新增工具：

```text
get_bom_candidates
get_selected_bom
explain_bom_selection
simulate_bom_change
get_operation_resource_requirements
explain_resource_shortage
explain_changeover_time
simulate_resource_mode
simulate_mold_change
```

---

# 12. MVP 一期范围更新

一期必须支持：

```text
Item 主数据：成品、半成品、物料
成品 BOM
半成品 BOM
多套 BOM 候选
BOM 优先级选择
BOM 物料可用性校验
工艺路线 DAG
多工艺路线候选
设备资源排程
人员资源可选排程
模具/夹具/工装基础约束
生产节拍模型
换模/换产基础时间
只排设备模式
只排人员模式
设备+人员同时排模式
Timefold 基础求解
甘特图展示
AI 解释 BOM、路线、资源、换产原因
```

一期暂缓：

```text
复杂替代料优化
复杂人员班组技能优化
复杂模具寿命管理
复杂工装维修管理
多工厂协同 BOM 替代
成本精细核算
动态实体数量求解
```

---

# 13. 验收标准补充

## 13.1 BOM 验收

1. 成品可以维护 BOM。
2. 半成品可以维护 BOM。
3. 物料不能作为常规 BOM 目标。
4. 一个成品或半成品可以维护多套 BOM。
5. 每套 BOM 可以配置优先级。
6. 系统可以根据适用条件筛选 BOM。
7. 系统可以在满足硬约束的前提下优先选择高优先级 BOM。
8. BOM 选择结果必须写入排程版本。
9. 用户可以查看为什么选择某套 BOM。

## 13.2 资源验收

1. 工序可以配置设备需求。
2. 工序可以配置人员需求。
3. 工序可以配置模具需求。
4. 工序可以配置夹具和工装需求。
5. 系统支持只排设备。
6. 系统支持只排人员。
7. 系统支持设备和人员同时排。
8. 同一资源同一时间不能被超量占用。
9. 资源必须满足能力要求。

## 13.3 时间模型验收

1. 工序支持固定时长。
2. 工序支持按数量计算加工时长。
3. 工序支持节拍模型。
4. 工序支持准备时间。
5. 工序支持换模时间。
6. 工序支持换产时间。
7. 任务总时长可以由多个 TaskSegment 组成。
8. 换模和换产时间可以显示在甘特图中。

---

# 14. 关键架构结论

## 14.1 BOM 和工艺路线必须分开

```text
BOM 解决“生产什么需要消耗什么”。
工艺路线解决“怎么生产”。
资源模型解决“用什么设备、人、模具生产”。
排程模型解决“什么时候生产”。
```

## 14.2 BOM 选择和工艺路线选择都应是优化问题

```text
硬约束决定能不能选。
软约束决定优先选哪一个。
```

## 14.3 资源不只是设备

设备、人员、模具、夹具、工装都可能是有限资源，都可能造成冲突，都可能影响交期。

## 14.4 时间不只是加工时间

准备、换模、换产、检验、搬运、等待、清理都可能占用计划时间和资源。

## 14.5 LLM 的价值是解释复杂选择

LLM/Agent 应解释：

1. 为什么选这套 BOM。
2. 为什么选这条工艺路线。
3. 为什么需要这台设备。
4. 为什么需要这些人员。
5. 为什么有换模时间。
6. 为什么产生延期。
7. 如果换 BOM、换路线、换资源，结果会怎样。
