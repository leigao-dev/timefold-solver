# AI-Native APS 高级计划与排程系统 PRD + DDD 技术设计文档 V6.0

> 文档定位：产品需求 PRD + DDD 领域设计 + 技术架构 + Timefold 求解器设计 + LLM/Agent 设计。  
> 当前最新主文档。以后以本文件为准。  
> 本版本重点合并：成品/半成品/物料 Item 模型、成品与半成品 BOM、多套 BOM 优先级选择、复杂工艺路线 DAG、设备/人员/模具/夹具/工装资源、生产节拍、换模换产时间、能力矩阵、Timefold 求解建模、AI Agent 解释能力。

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
能力矩阵：工序 × 产品/半成品 × 资源 × 节拍 × 成本 × 优先级
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

1. 成品和半成品都有 BOM。
2. 物料没有 BOM，物料是被消耗对象。
3. 一个成品或半成品可以有多套物料 BOM。
4. 多套 BOM 之间可以有限选择。
5. BOM 优先级应作为软约束，不应简单写死。
6. 一个成品或半成品可以有多条工艺路线。
7. 工艺路线优先级应作为软约束，不应简单写死。
8. 工艺路线不是简单线性工序列表，而是 ProcessGraph / DAG。
9. 生产资源不只有设备，还包括人员、模具、夹具、工装、工具。
10. APS 应支持只排设备、只排人员、设备和人员同时排、设备人员模具一起排。
11. 能力矩阵决定某个工序能由哪些资源加工、效率如何、成本如何、优先级如何。
12. 生产节拍、加工时间、换模时间、换产时间、准备时间都可能占用资源时间。
13. Timefold 负责严肃约束求解，LLM/Agent 负责解释、诊断、模拟和协同。

---

# 1. Item 主数据模型：成品、半成品、物料

## 1.1 统一 Item 模型

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
| 能力矩阵 | 说明哪些资源能做某个工序，以及效率、成本和优先级 |
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

## 2.3 Material BOM

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

## 2.5 多套 BOM 有限选择

一个成品或半成品可能有多套 BOM。

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

## 2.6 BOM 选择约束

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

> BOM 优先级不应该做成硬约束，而应该做成软约束。硬约束决定 BOM 能不能用，软约束决定优先用哪套 BOM。

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
根据能力矩阵筛选候选资源
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

## 5.2 OperationResourceRequirement 工序资源需求

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

# 6. 能力矩阵 Capability Matrix

## 6.1 能力矩阵是什么

能力矩阵是 APS 的核心基础数据，用于回答：

```text
这个产品或半成品的这个工序，哪些设备能做？
哪些人能做？
哪些模具/夹具/工装能用？
效率是多少？
节拍是多少？
成本是多少？
优先级是多少？
质量等级是否满足？
```

一句话定义：

> BOM 决定生产需要消耗什么，工艺路线决定有哪些工序，能力矩阵决定这些工序能由哪些资源来做，Timefold 决定最终排给谁、排在什么时候。

## 6.2 能力矩阵不是普通字典表

能力矩阵不是简单的资源标签，而是工序、产品/半成品、工艺路线、BOM、资源、节拍、质量、成本、优先级之间的关系模型。

最简单的能力矩阵：

| 工序 | 设备 | 是否可做 | 节拍 | 优先级 |
|---|---|---:|---:|---:|
| 冲压 | M01 | 是 | 60 件/小时 | 1 |
| 冲压 | M02 | 是 | 45 件/小时 | 2 |
| 冲压 | M03 | 否 | - | - |
| 焊接 | M04 | 是 | 30 件/小时 | 1 |

工业级能力矩阵应扩展为：

```text
工序 × Item × 工艺路线 × BOM × 资源类型 × 资源 × 节拍 × 效率 × 成本 × 优先级 × 质量等级
```

## 6.3 OperationResourceCapability 模型

```text
OperationResourceCapability
├── capabilityId
├── operationNodeId
├── itemId
├── itemType
├── routingId
├── bomId
├── resourceType
├── resourceId
├── requiredCapabilityCode
├── capable
├── priority
├── efficiencyFactor
├── durationPerUnit
├── capacityRate
├── setupTime
├── changeoverGroup
├── qualityLevel
├── costRate
├── minBatchSize
├── maxBatchSize
├── effectiveDate
├── expireDate
└── status
```

字段说明：

| 字段 | 说明 |
|---|---|
| operationNodeId | 哪个工序 |
| itemId | 针对哪个成品/半成品 |
| routingId | 针对哪条工艺路线，可为空表示通用 |
| bomId | 针对哪套 BOM，可为空表示通用 |
| resourceType | 设备、人员、模具、夹具、工装等 |
| resourceId | 具体资源 |
| capable | 是否具备加工能力 |
| priority | 资源优先级，数字越小越优先 |
| efficiencyFactor | 效率系数 |
| durationPerUnit | 单件加工时间 |
| capacityRate | 单位时间产能 |
| setupTime | 准备时间 |
| changeoverGroup | 换产分组 |
| qualityLevel | 该资源能达到的质量等级 |
| costRate | 资源使用成本 |
| minBatchSize | 最小批量 |
| maxBatchSize | 最大批量 |
| effectiveDate | 生效时间 |
| expireDate | 失效时间 |

## 6.4 能力矩阵与资源需求的关系

`OperationResourceRequirement` 描述工序需要什么类型的资源。

`OperationResourceCapability` 描述哪些具体资源能满足这个需求，以及做得好不好。

示例：

```text
工序：冲压
目标 Item：半成品 H
资源需求：需要 1 台冲压设备 + 1 套模具 + 1 名操作员

能力矩阵：
- M01 可做，60 件/小时，优先级 1
- M02 可做，45 件/小时，优先级 2
- Mold-A 可用，优先级 1
- WorkerGroup-1 可做，优先级 1
```

Timefold 排程时必须同时找到可用设备、模具、人员，并且这些资源在同一时间窗口内都可用。

## 6.5 能力矩阵与 BOM、工艺路线的关系

```text
BOM 决定生产需要消耗什么物料。
工艺路线决定需要经过哪些工序。
能力矩阵决定每个工序可以由哪些资源完成。
Timefold 决定最终资源组合和开始结束时间。
```

能力矩阵可以细化到 BOM 和工艺路线级别：

```text
同一工序，在 Routing-A 下 M01 可做；在 Routing-B 下 M02 可做。
同一工序，使用 BOM-A 时需要 Mold-A；使用 BOM-B 时需要 Mold-B。
```

## 6.6 能力矩阵与排程模式

### 只排设备

```text
只使用 resourceType = MACHINE 的能力矩阵。
```

适合设备是瓶颈、人不是瓶颈的场景。

### 只排人员

```text
只使用 resourceType = WORKER / TEAM 的能力矩阵。
```

适合人工工序、装配工序、检验工序。

### 设备和人员同时排

```text
同时使用 MACHINE + WORKER 的能力矩阵。
```

任务必须同时找到可用设备和可用人员。

### 设备 + 人员 + 模具 + 夹具一起排

```text
MACHINE + WORKER + MOLD + FIXTURE + TOOLING
```

任务必须同时满足所有资源能力与时间可用性。

## 6.7 能力矩阵与节拍

同一个工序，不同资源效率不同。

因此任务时长不能简单写死在工序上，而应根据能力矩阵动态计算：

```text
任务时长 = 数量 / 资源产能 + 准备时间 + 换模时间 + 换产时间
```

或者：

```text
任务时长 = 数量 × 单件时间 / 资源效率系数
```

## 6.8 能力矩阵约束

### 硬约束

| 约束 | 说明 |
|---|---|
| Resource Must Be Capable | 没有能力矩阵记录或 capable=false 的资源不能分配 |
| Capability Effective | 能力必须在有效期内 |
| Resource Type Match | 资源类型必须匹配工序资源需求 |
| Quality Level Match | 资源能力质量等级必须满足工序要求 |
| Batch Size Match | 任务数量必须满足资源最小/最大批量限制 |
| Required Capability Match | 资源能力标签必须满足工序要求 |

### 软约束

| 约束 | 说明 |
|---|---|
| Prefer Higher Priority Resource | 优先选择能力矩阵中优先级高的资源 |
| Prefer Higher Efficiency Resource | 优先选择效率高的资源 |
| Prefer Lower Cost Resource | 优先选择成本低的资源 |
| Prefer Better Quality Resource | 重要订单优先使用质量等级高的资源 |
| Minimize Capability Switch | 减少资源能力切换导致的准备和换产损失 |

---

# 7. 生产节拍、加工时间与时间构成

## 7.1 BOM 不决定生产节拍

BOM 决定消耗什么。

生产节拍、加工时间、换模换产时间属于工艺路线、工序和能力矩阵。

```text
BOM：生产这个成品/半成品需要哪些物料和数量
Routing / Operation：怎么生产、有哪些工序
Capability Matrix：哪些资源能做、节拍多少、效率多少
```

## 7.2 工序时间构成

```text
总时长 = 准备时间 + 换模时间 + 换产时间 + 加工时间 + 检验时间 + 搬运时间 + 等待时间 + 拆卸/清理时间
```

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

---

# 8. 换模、换产、准备与清理时间

## 8.1 换模 Change Mold

```text
MoldChangeRule
├── fromMoldId
├── toMoldId
├── machineId
├── duration
├── requiredWorkerCapability
└── remark
```

## 8.2 换产 Changeover

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

## 8.3 TaskSegment

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

---

# 9. Timefold 求解器建模

## 9.1 Planning Solution

```java
ScheduleSolution
├── List<Item>
├── List<MaterialBom>
├── List<OrderBomAssignment>
├── List<OrderRoutingAssignment>
├── List<Resource>
├── List<OperationResourceCapability>
├── List<OperationTaskAssignment>
├── List<TaskDependency>
├── List<TaskSegment>
├── List<PlannedOutput>
└── HardSoftScore
```

## 9.2 Planning Entity

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
├── candidateMachines
├── candidateWorkers
├── candidateMolds
├── candidateFixtures
├── candidateToolings
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

## 9.3 Value Range 来源

Timefold 的候选值域来自能力矩阵：

```text
candidateMachines  = capabilityMatrix.filter(resourceType=MACHINE)
candidateWorkers   = capabilityMatrix.filter(resourceType=WORKER or TEAM)
candidateMolds     = capabilityMatrix.filter(resourceType=MOLD)
candidateFixtures  = capabilityMatrix.filter(resourceType=FIXTURE)
candidateToolings  = capabilityMatrix.filter(resourceType=TOOLING)
```

## 9.4 硬约束

| 约束 | 说明 |
|---|---|
| BOM Belongs To Target Item | BOM 必须属于目标成品或半成品 |
| BOM Applicable | BOM 必须满足适用条件 |
| Material Availability | BOM 所需物料必须可用 |
| Routing Belongs To Target Item | 工艺路线必须属于目标成品或半成品 |
| Routing Applicable | 工艺路线必须满足适用条件 |
| Bom Routing Compatible | 所选 BOM 与所选工艺路线必须兼容 |
| Resource Capability | 资源必须出现在能力矩阵中且 capable=true |
| Resource Conflict | 同一设备/人员/模具/夹具同一时间不能超量占用 |
| Resource Calendar | 任务不能安排在资源不可用时间 |
| Task Graph Dependency | 满足工艺 DAG 依赖 |
| Semi-finished Availability | 半成品未产出或库存不足前不能消耗 |
| Tooling Availability | 模具、夹具、工装库存或容量不能超限 |
| Batch Size Match | 任务批量必须满足资源能力矩阵的最小/最大批量 |
| Pinned Task | 锁定任务不能被自动修改 |

## 9.5 软约束

| 约束 | 说明 |
|---|---|
| Prefer Higher Priority BOM | 优先选择高优先级 BOM |
| Prefer Higher Priority Routing | 优先选择高优先级工艺路线 |
| Prefer Higher Priority Resource | 优先选择能力矩阵中优先级高的资源 |
| Prefer Higher Efficiency Resource | 优先选择效率高的资源 |
| Prefer Lower Cost Resource | 优先选择成本低的资源 |
| Minimize Tardiness | 减少订单延期 |
| Minimize Setup | 减少准备时间 |
| Minimize Mold Change | 减少换模时间 |
| Minimize Changeover | 减少换产时间 |
| Balance Machine Load | 均衡设备负荷 |
| Balance Worker Load | 均衡人员负荷 |
| Plan Stability | 重排时尽量少改动原计划 |
| BOM Stability | 重排时尽量不改变已选 BOM |
| Routing Stability | 重排时尽量不改变已选工艺路线 |

---

# 10. 数据库表补充

## 10.1 material_bom

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

## 10.2 bom_line

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

## 10.3 resource

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

## 10.4 operation_resource_requirement

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

## 10.5 operation_resource_capability

```text
operation_resource_capability
├── id
├── operation_node_id
├── item_id
├── item_type
├── routing_id
├── bom_id
├── resource_type
├── resource_id
├── required_capability_code
├── capable
├── priority
├── efficiency_factor
├── duration_per_unit
├── capacity_rate
├── setup_time
├── changeover_group
├── quality_level
├── cost_rate
├── min_batch_size
├── max_batch_size
├── effective_date
├── expire_date
└── status
```

## 10.6 operation_time_model

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

## 10.7 task_segment

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

# 11. API 补充

## 11.1 BOM 接口

```http
POST /api/boms
GET  /api/boms/{bomId}
GET  /api/items/{itemId}/boms
POST /api/orders/{orderId}/bom-candidates
POST /api/orders/{orderId}/select-bom
POST /api/orders/{orderId}/simulate-bom-change
GET  /api/orders/{orderId}/bom-selection-explanation
```

## 11.2 能力矩阵接口

```http
POST /api/capability-matrix
GET  /api/capability-matrix
GET  /api/operation-nodes/{nodeId}/capabilities
GET  /api/resources/{resourceId}/capabilities
POST /api/capability-matrix/batch-import
POST /api/capability-matrix/validate
```

## 11.3 资源需求接口

```http
POST /api/operation-nodes/{nodeId}/resource-requirements
GET  /api/operation-nodes/{nodeId}/resource-requirements
POST /api/operation-nodes/{nodeId}/time-model
GET  /api/operation-nodes/{nodeId}/time-model
```

## 11.4 换模换产接口

```http
POST /api/changeover-rules
GET  /api/changeover-rules
POST /api/mold-change-rules
GET  /api/mold-change-rules
```

---

# 12. AI Agent 补充能力

用户可以问：

```text
这个成品用了哪套 BOM？
为什么没有选择优先级最高的 BOM？
这个半成品需要哪些物料？
这个工序哪些设备能做？
为什么这个任务排给 M02 而不是 M01？
这道工序到底需要设备还是人员？
如果只排设备不排人，结果会有什么风险？
为什么这两个任务之间有 40 分钟换模时间？
如果换成另一个模具，交期会不会更好？
这次延期是因为设备不足、人员不足、模具不足、能力矩阵不匹配，还是物料不足？
```

新增工具：

```text
get_bom_candidates
get_selected_bom
explain_bom_selection
simulate_bom_change
get_operation_resource_requirements
get_operation_capability_matrix
get_candidate_resources
explain_resource_assignment
explain_resource_shortage
explain_capability_mismatch
explain_changeover_time
simulate_resource_mode
simulate_mold_change
```

---

# 13. MVP 一期范围更新

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
资源主数据：设备、人员、模具、夹具、工装
工序资源需求
能力矩阵基础模型
设备能力矩阵
人员能力矩阵
模具/夹具/工装基础能力矩阵
生产节拍模型
换模/换产基础时间
只排设备模式
只排人员模式
设备+人员同时排模式
Timefold 基础求解
甘特图展示
AI 解释 BOM、路线、资源、能力矩阵、换产原因
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
复杂副产物库存优化
复杂能力继承与能力衰减模型
```

---

# 14. 验收标准补充

## 14.1 能力矩阵验收

1. 系统可以维护工序与设备的能力矩阵。
2. 系统可以维护工序与人员的能力矩阵。
3. 系统可以维护工序与模具、夹具、工装的能力矩阵。
4. 能力矩阵可以维护资源优先级。
5. 能力矩阵可以维护节拍、效率系数或单位加工时间。
6. 能力矩阵可以维护质量等级。
7. 能力矩阵可以维护最小批量和最大批量。
8. 没有能力矩阵匹配的资源不能被 Timefold 分配。
9. 停用或失效能力不能参与排程。
10. 用户可以查看某个任务的候选资源列表。
11. 用户可以解释为什么任务选择了某个资源。
12. AI 可以解释资源不匹配原因。

## 14.2 BOM 验收

1. 成品可以维护 BOM。
2. 半成品可以维护 BOM。
3. 物料不能作为常规 BOM 目标。
4. 一个成品或半成品可以维护多套 BOM。
5. 每套 BOM 可以配置优先级。
6. 系统可以根据适用条件筛选 BOM。
7. 系统可以在满足硬约束的前提下优先选择高优先级 BOM。
8. BOM 选择结果必须写入排程版本。
9. 用户可以查看为什么选择某套 BOM。

## 14.3 资源验收

1. 工序可以配置设备需求。
2. 工序可以配置人员需求。
3. 工序可以配置模具需求。
4. 工序可以配置夹具和工装需求。
5. 系统支持只排设备。
6. 系统支持只排人员。
7. 系统支持设备和人员同时排。
8. 同一资源同一时间不能被超量占用。
9. 资源必须满足能力矩阵要求。

---

# 15. 关键架构结论

## 15.1 BOM、工艺路线、能力矩阵必须分开

```text
BOM 解决“生产什么需要消耗什么”。
工艺路线解决“怎么生产”。
能力矩阵解决“哪些资源能做，做得多快，成本多高，优先级如何”。
排程模型解决“什么时候生产、由谁生产、用什么资源生产”。
```

## 15.2 能力矩阵是 Timefold 候选资源的来源

```text
Timefold 不能从所有设备里乱选，只能从能力矩阵筛选出来的候选资源里选择。
```

## 15.3 资源不只是设备

设备、人员、模具、夹具、工装都可能是有限资源，都可能造成冲突，都可能影响交期。

## 15.4 时间不只是加工时间

准备、换模、换产、检验、搬运、等待、清理都可能占用计划时间和资源。

## 15.5 LLM 的价值是解释复杂选择

LLM/Agent 应解释：

1. 为什么选这套 BOM。
2. 为什么选这条工艺路线。
3. 为什么这个工序只能由这些资源做。
4. 为什么任务分配给这个设备/人员/模具。
5. 为什么有换模时间。
6. 为什么产生延期。
7. 如果换 BOM、换路线、换资源，结果会怎样。
