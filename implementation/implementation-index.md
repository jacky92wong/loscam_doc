# 实施文档总索引

## 1. 文档目标
本目录用于把已有的业务资料与技术资料，收敛成可供客户确认和后续开发使用的实施稿。当前阶段不进入代码开发，重点输出：数据结构、数据契约、API 草案、算法实施说明，以及 MVP 主闭环的实现参考。

## 2. 与现有资料的关系
实施稿直接承接以下已发布文档：
- `../technical/data-model.md`：定义实体、状态、时间与位置维度
- `../technical/optimization-model.md`：定义优化目标、约束与分阶段算法路线
- `api-draft.md`：定义 MVP 主闭环内的接口草案
- `logical-schema.md`：定义逻辑数据底座
- `data-contracts.md`：定义输入输出契约

## 3. 阅读顺序
建议客户与实施团队按以下顺序阅读：
1. `logical-schema.md`
2. `data-contracts.md`
3. `api-draft.md`
4. `forecasting-pseudocode.md`
5. `dispatch-heuristic.md`

## 4. 文档依赖关系
- `logical-schema.md` 是数据实施底座。
- `data-contracts.md` 基于逻辑表结构定义输入输出口径。
- `api-draft.md` 基于数据契约定义接口草案。
- `forecasting-pseudocode.md` 与 `dispatch-heuristic.md` 复用 schema、状态定义和 MVP 路线。

## 5. 本轮实施稿覆盖范围
本轮只覆盖 MVP 最小闭环：
- 订单输入
- 库存输入
- 库存状态修正
- 需求预测输出
- 调拨推荐输出
- 调拨审批反馈
- 执行结果回写

## 6. 本轮明确不做
为控制范围，本轮实施稿不展开以下内容：
- 全网实时自动调度
- 复杂 VRP 深度优化
- 多区域一次性上线方案
- 完整 OpenAPI 文件与数据库 DDL
- 前端高保真原型

## 7. 建议使用方式
### 7.1 客户确认阶段
客户优先确认：
- 数据可得性与字段口径
- 外部系统接口方式
- 人工审批机制
- MVP 试点范围
- 优先 KPI 与约束权重

### 7.2 后续开发阶段
若客户确认通过，可继续补充：
- 数据库 DDL
- OpenAPI 定义
- 页面线框图
- 预测原型脚本
- 调拨启发式原型脚本

## 8. 验证清单
完成实施稿后应检查：
1. 是否形成 `business/`、`technical/`、`implementation/` 三层文档结构。
2. 字段名、状态枚举、时间窗定义是否与 `../technical/data-model.md` 一致。
3. 是否完整覆盖订单输入、库存输入、预测输出、调拨推荐、审批反馈闭环。
4. 当前发布版是否仍保持 MVP 范围，不混入未发布草稿。
