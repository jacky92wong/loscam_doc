# LC 发布版阅读地图

## 1. 页面用途
本页是当前 GitBook 发布版本的阅读地图。

它的设计目标是与当前发布仓库中的实际结构保持一致，确保在客户演示、管理层评审与技术讨论时，页面标题、导航结构与阅读路径始终对应。

## 2. 当前发布范围
当前 GitBook 发布版本仅包含成熟文档集。

当前发布分区包括：
- 业务文档
- 技术文档
- 实施文档
- 评审与就绪度

当前版本不包含：
- 内部工作笔记
- 草稿材料
- 日志与脚本
- 当前成熟文档集之外、尚未发布的支撑页面

---

## 3. 分区结构

### 3.1 业务文档
本分区主要用于干系人沟通、范围对齐与管理层评审。

包含页面：
- [LC 行业背景与系统设计说明](../business/industry-background-and-system-design-overview.md)
- [LC 图示化总览（ASCII 版）](../business/visual-overview-ascii-diagrams.md)
- [LC 行业调研与方案前置信息清单](../business/industry-research-and-pre-solution-checklist.md)
- [LC 客户访谈提纲](../business/client-interview-questionnaire.md)
- [LC 系统边界与非目标说明](../business/system-boundary-and-non-goals.md)
- [LC 推荐解释规则说明](../business/recommendation-explainability-guide.md)
- [LC 数据质量假设与降级规则说明](../business/data-quality-assumptions-and-degradation-rules.md)
- [LC 试点实施与验收方案](../business/pilot-implementation-and-acceptance-plan.md)
- [LC 待客户确认问题总表](../business/client-confirmation-master-checklist.md)
- [LC 汇报版总览文档](../business/executive-summary-and-meeting-version.md)
- [状态流转图与数据流图](../business/state-transition-and-data-flow-diagrams.md)

### 3.2 技术文档
本分区用于说明方案背后的核心数据、预测、优化与调拨逻辑。

包含页面：
- [数据模型](../technical/data-model.md)
- [需求预测模型](../technical/forecasting-model.md)
- [优化模型](../technical/optimization-model.md)
- [路由与调度](../technical/routing-and-dispatch.md)

### 3.3 实施文档
本分区将业务与技术材料进一步转换为面向 MVP 的实施输出。

包含页面：
- [实施文档总索引](../implementation/implementation-index.md)
- [逻辑表结构草案](../implementation/logical-schema.md)
- [数据契约草案](../implementation/data-contracts.md)
- [API 草案](../implementation/api-draft.md)
- [需求预测实施伪代码](../implementation/forecasting-pseudocode.md)
- [调拨启发式实施稿](../implementation/dispatch-heuristic.md)

### 3.4 评审与就绪度
本分区用于总结当前整套文档包的就绪程度。

包含页面：
- [LC 当前文档包完整性评审结论](../review-and-readiness/document-package-review-readiness.md)

---

## 4. 推荐阅读路径

### 4.1 管理层 / 汇报路径
推荐顺序：
1. [LC 汇报版总览文档](../business/executive-summary-and-meeting-version.md)
2. [LC 系统边界与非目标说明](../business/system-boundary-and-non-goals.md)
3. [LC 试点实施与验收方案](../business/pilot-implementation-and-acceptance-plan.md)
4. [LC 待客户确认问题总表](../business/client-confirmation-master-checklist.md)
5. [LC 当前文档包完整性评审结论](../review-and-readiness/document-package-review-readiness.md)

### 4.2 业务与方案评审路径
推荐顺序：
1. [LC 行业背景与系统设计说明](../business/industry-background-and-system-design-overview.md)
2. [LC 图示化总览（ASCII 版）](../business/visual-overview-ascii-diagrams.md)
3. [LC 系统边界与非目标说明](../business/system-boundary-and-non-goals.md)
4. [LC 推荐解释规则说明](../business/recommendation-explainability-guide.md)
5. [LC 数据质量假设与降级规则说明](../business/data-quality-assumptions-and-degradation-rules.md)
6. [状态流转图与数据流图](../business/state-transition-and-data-flow-diagrams.md)

### 4.3 技术评审路径
推荐顺序：
1. [数据模型](../technical/data-model.md)
2. [需求预测模型](../technical/forecasting-model.md)
3. [优化模型](../technical/optimization-model.md)
4. [路由与调度](../technical/routing-and-dispatch.md)
5. [实施文档总索引](../implementation/implementation-index.md)
6. [逻辑表结构草案](../implementation/logical-schema.md)
7. [数据契约草案](../implementation/data-contracts.md)
8. [API 草案](../implementation/api-draft.md)

### 4.4 MVP 实施路径
推荐顺序：
1. [实施文档总索引](../implementation/implementation-index.md)
2. [逻辑表结构草案](../implementation/logical-schema.md)
3. [数据契约草案](../implementation/data-contracts.md)
4. [API 草案](../implementation/api-draft.md)
5. [需求预测实施伪代码](../implementation/forecasting-pseudocode.md)
6. [调拨启发式实施稿](../implementation/dispatch-heuristic.md)

---

## 5. 如何理解本次发布
本次 GitBook 发布版本应被理解为更大文档体系中的一个经过整理、适合演示与评审的子集，而不是全部内部材料。

它主要用于支持：
- 管理层沟通
- 客户评审
- 技术对齐
- MVP 实施规划

它并不用于承载所有内部草稿，也不代表未来所有支撑文档都已经纳入当前发布版本。
