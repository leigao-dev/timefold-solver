# AI-Native APS PRD 补充：客户化多套物料 BOM 有限选择

> 本文档用于补充主 PRD `AI_Native_APS_PRD.md` 中关于 BOM 的设计。  
> 补充重点：同一款成品或半成品，可以针对不同客户、客户等级、订单类型、质量要求配置不同物料 BOM；系统在满足硬约束的前提下优先选择优先级最高的 BOM。

---

# 1. 需求背景

在实际制造业务中，同一款成品或半成品不一定只有一套物料 BOM。

同一产品可能因为以下原因使用不同物料 BOM：

1. 不同客户指定不同材料。
2. 不同客户等级使用不同质量等级物料。
3. 不同销售订单要求不同包装、辅料或认证材料。
4. 国内订单和出口订单使用不同物料。
5. 标准 BOM 缺料时，允许使用替代 BOM。
6. 紧急订单可使用应急物料 BOM。
7. 不同工厂、车间或产线支持不同物料方案。

因此 APS 必须支持：

```text
同一成品 / 半成品
  ↓
多套物料 BOM
  ↓
按客户、订单、质量、工厂、数量、日期等条件筛选
  ↓
在满足硬约束的前提下优先选择高优先级 BOM
```

---

# 2. 业务定义

## 2.1 客户化多套 BOM

客户化多套 BOM 指的是：

> 同一个目标 Item，也就是同一成品或半成品，可以维护多套物料 BOM。每套 BOM 可以配置适用客户、客户等级、订单类型、质量要求、工厂范围、数量范围、生效日期和优先级。

示例：

```text
成品 Product-A
├── BOM-A：标准 BOM，适用于普通客户，优先级 1
├── BOM-B：客户甲专用 BOM，适用于 Customer-A，优先级 1
├── BOM-C：客户乙环保 BOM，适用于 Customer-B，优先级 1
├── BOM-D：通用替代 BOM，适用于所有客户，优先级 2
└── BOM-E：应急 BOM，适用于紧急订单，优先级 3
```

---

# 3. BOM 适用规则 BomApplicableRule

主 PRD 中的 `MaterialBom.applicableRules` 应明确展开为 `BomApplicableRule`。

```text
BomApplicableRule
├── ruleId
├── bomId
├── targetItemId
├── customerScope
├── customerGradeScope
├── salesOrderScope
├── orderTypeScope
├── marketRegionScope
├── factoryScope
├── workshopScope
├── productionLineScope
├── minQuantity
├── maxQuantity
├── qualityRequirement
├── certificationRequirement
├── packagingRequirement
├── materialCondition
├── effectiveDate
├── expireDate
└── requiredApproval
```

字段说明：

| 字段 | 说明 |
|---|---|
| customerScope | 适用客户范围，例如客户甲、客户乙、全部客户 |
| customerGradeScope | 适用客户等级，例如战略客户、普通客户 |
| salesOrderScope | 指定销售订单范围 |
| orderTypeScope | 订单类型，例如普通订单、紧急订单、出口订单 |
| marketRegionScope | 市场区域，例如国内、欧盟、北美 |
| factoryScope | 适用工厂 |
| workshopScope | 适用车间 |
| productionLineScope | 适用产线 |
| minQuantity / maxQuantity | 适用数量范围 |
| qualityRequirement | 质量等级要求 |
| certificationRequirement | 认证要求，例如环保、食品级、RoHS 等 |
| packagingRequirement | 包装要求 |
| materialCondition | 物料条件，例如是否齐套、是否允许替代料 |
| effectiveDate / expireDate | 生效与失效日期 |
| requiredApproval | 是否需要审批后才允许使用 |

---

# 4. BOM 候选筛选流程

生产订单进入排程时，系统应按以下流程筛选 BOM：

```text
生产订单
  ↓
读取目标 Item：成品或半成品
  ↓
读取订单客户、客户等级、订单类型、数量、质量要求、交付区域
  ↓
查询该目标 Item 的全部启用 BOM
  ↓
按 BomApplicableRule 过滤候选 BOM
  ↓
校验 BOM 生效日期、失效日期、审批状态
  ↓
校验 BOM 与工艺路线兼容性
  ↓
校验 BOM 物料可用性、质量等级、预计到料时间
  ↓
生成 BomCandidate 列表
  ↓
交给 APS / Timefold 在候选 BOM 中选择
```

---

# 5. BomCandidate 模型

建议新增 `BomCandidate` 概念，用于记录某个订单在当前排程上下文中可选择的 BOM。

```text
BomCandidate
├── candidateId
├── orderId
├── targetItemId
├── bomId
├── bomCode
├── priority
├── applicable
├── unavailableReason
├── customerMatched
├── orderTypeMatched
├── qualityMatched
├── materialAvailable
├── materialReadyTime
├── estimatedCost
├── estimatedRisk
└── scoreImpact
```

说明：

1. `customerMatched` 表示是否命中客户适用规则。
2. `materialAvailable` 表示该 BOM 的物料是否满足当前排程需求。
3. `materialReadyTime` 表示该 BOM 最早齐套时间。
4. `estimatedRisk` 表示缺料、质量、到货延迟、审批等风险。

---

# 6. OrderBomAssignment 模型增强

主 PRD 中的 `OrderBomAssignment` 应增强如下：

```text
OrderBomAssignment
├── assignmentId
├── orderId
├── targetItemId
├── selectedBomId
├── selectionMode
├── selectionReason
├── customerCode
├── customerName
├── matchedRuleId
├── priorityPenalty
├── materialAvailabilityImpact
├── estimatedCostImpact
├── scoreImpact
├── planVersionId
├── createdBy
└── createdAt
```

selectionMode：

```text
AUTO_SELECTED         系统自动选择
MANUALLY_SPECIFIED    人工指定
EXTERNALLY_SPECIFIED  ERP/PLM/MES 指定
SIMULATION_SELECTED   模拟方案选择
```

---

# 7. BOM 选择硬约束

硬约束决定 BOM 能不能选。

| 约束 | 说明 |
|---|---|
| BOM Belongs To Target Item | BOM 必须属于当前成品或半成品 |
| BOM Enabled | BOM 必须启用 |
| BOM Effective | BOM 必须在有效期内 |
| Customer Scope Match | BOM 必须满足客户适用范围 |
| Customer Grade Match | BOM 必须满足客户等级 |
| Order Type Match | BOM 必须满足订单类型 |
| Quantity Range Match | 订单数量必须在 BOM 适用范围内 |
| Quality Requirement Match | BOM 必须满足订单质量要求 |
| Certification Match | BOM 必须满足认证要求 |
| Bom Routing Compatible | BOM 必须与工艺路线兼容 |
| Material Available | BOM 所需关键物料必须可用或可在可接受时间内到货 |
| Approval Required | 需要审批的 BOM 未审批时不能直接用于正式发布计划 |

示例：

```text
客户甲订单不能使用客户乙专用 BOM。
出口订单不能使用不满足出口认证的 BOM。
高质量等级订单不能使用低质量物料 BOM。
```

---

# 8. BOM 选择软约束

软约束决定优先选哪套 BOM。

| 约束 | 说明 |
|---|---|
| Prefer Customer Specific BOM | 优先选择客户专用 BOM |
| Prefer Higher Priority BOM | 优先选择优先级高的 BOM |
| Prefer Material Ready BOM | 优先选择物料齐套时间更早的 BOM |
| Prefer Lower Cost BOM | 优先选择成本更低的 BOM |
| Minimize Substitute Material | 尽量减少替代料使用 |
| Prefer Stable BOM | 重排时尽量保持原 BOM 不变 |
| Prefer Lower Risk BOM | 优先选择缺料、质量、审批风险更低的 BOM |

评分建议：

```text
bomPriorityPenalty = (bomPriority - 1) × bomPriorityWeight
customerBomBonus = customerSpecificMatched ? customerBomBonusWeight : 0
materialReadyPenalty = delayMinutes × materialReadyWeight
substituteMaterialPenalty = substituteMaterialCount × substitutePenaltyWeight
```

关键原则：

```text
客户适用范围是硬约束。
BOM 优先级是软约束。
物料齐套时间可以作为软约束或硬约束，取决于业务要求。
```

---

# 9. 客户化 BOM 示例

## 9.1 业务数据

```text
成品：Product-A
客户：Customer-A
订单数量：1000
交期：2026-05-10
```

候选 BOM：

| BOM | 适用客户 | 优先级 | 物料状态 | 预计齐套时间 |
|---|---|---:|---|---|
| BOM-A | 所有客户 | 2 | 齐套 | 2026-05-01 |
| BOM-B | Customer-A | 1 | 缺关键物料 | 2026-05-08 |
| BOM-C | Customer-A | 2 | 齐套 | 2026-05-01 |
| BOM-D | Customer-B | 1 | 齐套 | 2026-05-01 |

系统筛选结果：

```text
BOM-D 不可选，因为客户不匹配。
BOM-B 可选，但物料齐套较晚。
BOM-C 可选，客户匹配且齐套。
BOM-A 可选，通用 BOM。
```

选择逻辑：

```text
如果 BOM-B 齐套时间不影响交期，则优先选择 BOM-B。
如果 BOM-B 导致延期，系统可以选择 BOM-C 或 BOM-A。
选择低优先级 BOM 时，必须记录原因。
```

---

# 10. Timefold 建模补充

## 10.1 Planning Entity

```java
OrderBomAssignment
├── orderId
├── targetItemId
├── customerId
├── orderType
├── candidateBoms
├── selectedBom
├── bomPriority
├── customerMatched
├── materialReadyTime
└── bomPenalty
```

## 10.2 Value Range

`selectedBom` 的候选范围来自：

```text
MaterialBom
  .filter(targetItemId == order.targetItemId)
  .filter(status == ENABLED)
  .filter(effectiveDate <= order.planDate <= expireDate)
  .filter(BomApplicableRule matches order.customer/orderType/quantity/quality)
```

## 10.3 硬约束

```text
selectedBom 必须属于订单目标 Item
selectedBom 必须匹配客户范围
selectedBom 必须匹配订单类型
selectedBom 必须满足质量和认证要求
selectedBom 必须与 selectedRouting 兼容
```

## 10.4 软约束

```text
优先客户专用 BOM
优先高优先级 BOM
优先物料齐套更早的 BOM
优先成本更低的 BOM
减少替代料使用
重排时保持 BOM 稳定
```

---

# 11. API 补充

## 11.1 BOM 候选查询

```http
POST /api/orders/{orderId}/bom-candidates
```

响应示例：

```json
{
  "orderId": "WO-001",
  "targetItemId": "Product-A",
  "customerCode": "Customer-A",
  "candidates": [
    {
      "bomId": "BOM-B",
      "priority": 1,
      "customerMatched": true,
      "applicable": true,
      "materialAvailable": false,
      "materialReadyTime": "2026-05-08T08:00:00",
      "risk": "关键物料预计到货较晚"
    },
    {
      "bomId": "BOM-C",
      "priority": 2,
      "customerMatched": true,
      "applicable": true,
      "materialAvailable": true,
      "materialReadyTime": "2026-05-01T08:00:00"
    }
  ]
}
```

## 11.2 选择 BOM

```http
POST /api/orders/{orderId}/select-bom
```

请求：

```json
{
  "planVersionId": "V001",
  "bomId": "BOM-C",
  "selectionMode": "MANUALLY_SPECIFIED",
  "reason": "客户专用 BOM-B 关键物料到货晚，选择客户兼容 BOM-C 保证交期"
}
```

## 11.3 解释 BOM 选择

```http
GET /api/orders/{orderId}/bom-selection-explanation
```

---

# 12. AI Agent 补充能力

用户可以问：

```text
这个订单为什么选了这套 BOM？
这个客户可以用哪些 BOM？
为什么没有选客户专用 BOM？
客户甲和客户乙同一产品的 BOM 有什么不同？
如果强制使用客户专用 BOM，会不会延期？
如果改用通用 BOM，会影响哪些物料和工艺路线？
这套 BOM 是客户指定的，还是系统自动选择的？
```

新增工具：

```text
get_customer_bom_candidates
get_selected_bom
explain_bom_selection
compare_bom_candidates
simulate_bom_change
explain_customer_bom_mismatch
explain_material_ready_risk
```

---

# 13. 验收标准补充

1. 同一成品可以维护多套 BOM。
2. 同一半成品可以维护多套 BOM。
3. 每套 BOM 可以配置适用客户。
4. 每套 BOM 可以配置适用客户等级。
5. 每套 BOM 可以配置适用订单类型。
6. 每套 BOM 可以配置适用数量范围。
7. 每套 BOM 可以配置优先级。
8. 客户甲订单不能选择客户乙专用 BOM。
9. 默认 BOM 可以作为通用候选 BOM。
10. 系统可以根据订单客户筛选候选 BOM。
11. 系统可以在满足硬约束的前提下优先选择高优先级 BOM。
12. 当高优先级客户 BOM 缺料或导致严重延期时，系统可以选择低优先级兼容 BOM。
13. BOM 选择结果必须写入排程方案版本。
14. 用户可以查看 BOM 选择原因。
15. AI 可以解释为什么没有选择客户专用 BOM。

---

# 14. 合并到主 PRD 的位置建议

建议将本文内容合并到主 PRD 的以下章节：

1. `# 2. BOM 模型：成品和半成品都有 BOM，物料没有 BOM`
2. `# 9. Timefold 求解器建模`
3. `# 11. API 补充`
4. `# 12. AI Agent 补充能力`
5. `# 14. 验收标准补充`

合并后，主 PRD 应明确表达：

> 同一款成品或半成品可以按客户、订单类型、质量要求、工厂、数量范围维护多套物料 BOM；APS 在满足客户适用、物料可用、质量认证、BOM 与工艺路线兼容等硬约束后，优先选择优先级最高、物料齐套更早、风险更低的 BOM。
