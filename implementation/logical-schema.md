# 逻辑表结构草案

## 1. 说明
本文件把 `../technical/data-model.md` 中的核心实体，收敛为可实施的逻辑表设计。当前以 MVP 为目标，优先保证“订单输入 → 库存修正 → 需求预测 → 调拨建议 → 审批反馈”的数据闭环，不追求一次性覆盖全部扩展字段。

## 2. 建模约定
- 主键统一使用业务主键或系统生成 UUID。
- 所有核心表保留 `created_at`、`updated_at`。
- 所有状态字段优先复用统一枚举，避免各表各自定义。
- 时间字段至少区分：事件时间、计划时间、可用时间、实际时间。
- 位置维度统一分为区域、节点、节点类型三层。

## 3. 枚举建议
### 3.1 location_type
- `warehouse`
- `depot`
- `customer_site`
- `service_point`
- `transit_node`

### 3.2 pallet_status
- `available`
- `reserved`
- `in_use`
- `in_transit`
- `available_soon`
- `maintenance`
- `lost`
- `suspected_lost`

### 3.3 transfer_task_status
- `draft`
- `recommended`
- `pending_approval`
- `approved`
- `rejected`
- `scheduled`
- `in_execution`
- `completed`
- `cancelled`
- `failed`

### 3.4 dispatch_plan_status
- `generated`
- `published`
- `partially_approved`
- `approved`
- `expired`
- `archived`

### 3.5 order_status
- `draft`
- `confirmed`
- `allocated`
- `shipped`
- `completed`
- `cancelled`

## 4. 核心主数据表

### 4.1 warehouse
用途：维护仓库节点主数据。

| 字段 | 类型建议 | 必填 | 说明 |
|---|---|---:|---|
| warehouse_id | string | 是 | 仓库主键 |
| warehouse_code | string | 是 | 对外编码 |
| warehouse_name | string | 是 | 仓库名称 |
| region_code | string | 是 | 所属区域 |
| city_code | string | 否 | 所属城市 |
| latitude | decimal(10,6) | 否 | 纬度 |
| longitude | decimal(10,6) | 否 | 经度 |
| storage_capacity | int | 否 | 托盘总容量 |
| operation_start_time | string | 否 | 作业开始时间 |
| operation_end_time | string | 否 | 作业结束时间 |
| pallet_type_support | json | 否 | 支持托盘类型列表 |
| is_active | boolean | 是 | 是否启用 |
| created_at | datetime | 是 | 创建时间 |
| updated_at | datetime | 是 | 更新时间 |

### 4.2 depot
用途：维护回收点、服务网点或第三方 depot 主数据。

| 字段 | 类型建议 | 必填 | 说明 |
|---|---|---:|---|
| depot_id | string | 是 | depot 主键 |
| depot_code | string | 是 | depot 编码 |
| depot_name | string | 是 | depot 名称 |
| region_code | string | 是 | 所属区域 |
| linked_warehouse_id | string | 否 | 归属仓库 |
| depot_type | string | 否 | 自营/第三方/回收点 |
| latitude | decimal(10,6) | 否 | 纬度 |
| longitude | decimal(10,6) | 否 | 经度 |
| operation_hours | string | 否 | 作业时间说明 |
| pallet_type_support | json | 否 | 支持托盘类型 |
| is_active | boolean | 是 | 是否启用 |
| created_at | datetime | 是 | 创建时间 |
| updated_at | datetime | 是 | 更新时间 |

### 4.3 customer
用途：维护客户与需求侧主数据。

| 字段 | 类型建议 | 必填 | 说明 |
|---|---|---:|---|
| customer_id | string | 是 | 客户主键 |
| customer_code | string | 是 | 客户编码 |
| customer_name | string | 是 | 客户名称 |
| customer_segment | string | 否 | 客户分层 |
| industry | string | 否 | 行业 |
| service_level | string | 否 | SLA 等级 |
| default_region_code | string | 否 | 默认服务区域 |
| historical_return_behavior | string | 否 | 回流行为标签 |
| is_active | boolean | 是 | 是否启用 |
| created_at | datetime | 是 | 创建时间 |
| updated_at | datetime | 是 | 更新时间 |

### 4.4 vehicle
用途：维护可用于调拨的车辆或运力类型。

| 字段 | 类型建议 | 必填 | 说明 |
|---|---|---:|---|
| vehicle_id | string | 是 | 车辆主键 |
| vehicle_code | string | 是 | 车辆编码 |
| vehicle_type | string | 是 | 车型 |
| pallet_capacity | int | 是 | 托盘容量 |
| region_scope | string | 否 | 覆盖区域 |
| available_start_time | datetime | 否 | 可用开始时间 |
| available_end_time | datetime | 否 | 可用结束时间 |
| cost_rule_code | string | 否 | 运价规则编码 |
| owner_type | string | 否 | 自有/外协 |
| is_active | boolean | 是 | 是否启用 |
| created_at | datetime | 是 | 创建时间 |
| updated_at | datetime | 是 | 更新时间 |

### 4.5 route_lane
用途：维护调拨路线与运输成本基线。

| 字段 | 类型建议 | 必填 | 说明 |
|---|---|---:|---|
| lane_id | string | 是 | 路线主键 |
| origin_location_id | string | 是 | 起点节点 |
| origin_location_type | string | 是 | 起点类型 |
| destination_location_id | string | 是 | 终点节点 |
| destination_location_type | string | 是 | 终点类型 |
| distance_km | decimal(10,2) | 否 | 距离 |
| duration_hours | decimal(10,2) | 否 | 时长 |
| base_cost | decimal(12,2) | 否 | 基础成本 |
| toll_cost | decimal(12,2) | 否 | 过路费 |
| service_frequency | string | 否 | 班次频率 |
| lane_status | string | 否 | 是否可用 |
| created_at | datetime | 是 | 创建时间 |
| updated_at | datetime | 是 | 更新时间 |

## 5. 状态与交易表

### 5.1 pallet
用途：单托盘级主档。若客户只有聚合库存，可先不启用本表的全量能力。

| 字段 | 类型建议 | 必填 | 说明 |
|---|---|---:|---|
| pallet_id | string | 是 | 托盘唯一标识 |
| pallet_type | string | 是 | 托盘类型 |
| ownership_pool | string | 否 | 所属池 |
| current_location_id | string | 否 | 当前节点 |
| current_location_type | string | 否 | 当前节点类型 |
| current_status | string | 是 | 当前状态，复用 `pallet_status` |
| quality_grade | string | 否 | 质量等级 |
| last_event_time | datetime | 否 | 最近事件时间 |
| expected_available_time | datetime | 否 | 预计可用时间 |
| iot_device_id | string | 否 | IoT 设备号 |
| created_at | datetime | 是 | 创建时间 |
| updated_at | datetime | 是 | 更新时间 |

### 5.2 inventory_snapshot
用途：保存节点维度的库存快照，是 MVP 最关键状态表之一。

| 字段 | 类型建议 | 必填 | 说明 |
|---|---|---:|---|
| snapshot_id | string | 是 | 快照主键 |
| snapshot_time | datetime | 是 | 快照时间 |
| location_id | string | 是 | 节点 ID |
| location_type | string | 是 | 节点类型 |
| region_code | string | 是 | 区域编码 |
| pallet_type | string | 是 | 托盘类型 |
| available_qty | int | 是 | 可用数量 |
| reserved_qty | int | 是 | 已预留数量 |
| in_use_qty | int | 是 | 使用中数量 |
| in_transit_qty | int | 是 | 在途数量 |
| available_soon_qty | int | 是 | 即将可用数量 |
| maintenance_qty | int | 是 | 维修中数量 |
| lost_qty | int | 否 | 已确认丢失数量 |
| suspected_lost_qty | int | 否 | 疑似丢失数量 |
| blocked_qty | int | 否 | 账上存在但当前不可作为可信供给的数量 |
| data_source | string | 否 | 来源系统 |
| source_batch_id | string | 否 | 批次号 |
| created_at | datetime | 是 | 创建时间 |
| updated_at | datetime | 是 | 更新时间 |

### 5.3 order_demand
用途：保存订单需求与预测目标的输入事实。

| 字段 | 类型建议 | 必填 | 说明 |
|---|---|---:|---|
| order_id | string | 是 | 订单主键 |
| external_order_no | string | 否 | 外部订单号 |
| customer_id | string | 是 | 客户 ID |
| source_location_id | string | 否 | 发货节点 |
| destination_location_id | string | 是 | 收货节点 |
| destination_location_type | string | 是 | 收货节点类型 |
| requested_delivery_time | datetime | 是 | 要求到达时间 |
| order_date | datetime | 是 | 订单日期 |
| pallet_type | string | 是 | 托盘类型 |
| demand_qty | int | 是 | 需求量 |
| priority | string | 否 | 优先级 |
| status | string | 是 | 订单状态，复用 `order_status` |
| region_code | string | 否 | 所属区域 |
| import_batch_id | string | 否 | 导入批次号 |
| created_at | datetime | 是 | 创建时间 |
| updated_at | datetime | 是 | 更新时间 |

### 5.4 transfer_task
用途：保存推荐、审批和执行中的调拨任务。

| 字段 | 类型建议 | 必填 | 说明 |
|---|---|---:|---|
| task_id | string | 是 | 调拨任务主键 |
| plan_id | string | 否 | 所属调度方案 |
| from_location_id | string | 是 | 来源节点 |
| from_location_type | string | 是 | 来源类型 |
| to_location_id | string | 是 | 目标节点 |
| to_location_type | string | 是 | 目标类型 |
| pallet_type | string | 是 | 托盘类型 |
| planned_qty | int | 是 | 建议数量 |
| approved_qty | int | 否 | 审批数量 |
| planned_departure_time | datetime | 否 | 计划发车时间 |
| planned_arrival_time | datetime | 否 | 计划到达时间 |
| source_window_start | datetime | 否 | 来源窗口开始 |
| source_window_end | datetime | 否 | 来源窗口结束 |
| target_arrival_window_start | datetime | 否 | 目标生效窗口开始 |
| target_arrival_window_end | datetime | 否 | 目标生效窗口结束 |
| assigned_vehicle_id | string | 否 | 分配车辆 |
| lane_id | string | 否 | 路线 ID |
| reason_code | string | 否 | 主推荐原因码 |
| reason_codes | json | 否 | 推荐原因码列表 |
| risk_flags | json | 否 | 风险标签列表 |
| eta_confidence | string | 否 | ETA 可信度等级 high/medium/low |
| priority_score | decimal(10,4) | 否 | 优先级分数 |
| status | string | 是 | 任务状态，复用 `transfer_task_status` |
| approved_by | string | 否 | 审批人 |
| approved_at | datetime | 否 | 审批时间 |
| actual_departure_time | datetime | 否 | 实际发车时间 |
| actual_arrival_time | datetime | 否 | 实际到达时间 |
| actual_qty | int | 否 | 实际执行数量 |
| feedback_note | string | 否 | 执行反馈 |
| created_at | datetime | 是 | 创建时间 |
| updated_at | datetime | 是 | 更新时间 |

## 6. 算法结果表

### 6.1 forecast_result
用途：保存需求预测结果。

| 字段 | 类型建议 | 必填 | 说明 |
|---|---|---:|---|
| forecast_id | string | 是 | 预测结果主键 |
| target_location_id | string | 是 | 预测目标节点 |
| target_location_type | string | 是 | 节点类型 |
| region_code | string | 是 | 区域编码 |
| pallet_type | string | 是 | 托盘类型 |
| time_granularity | string | 是 | 日级/周级 |
| window_start_time | datetime | 是 | 窗口开始 |
| window_end_time | datetime | 是 | 窗口结束 |
| predicted_demand | int | 是 | 预测需求 |
| lower_bound | int | 否 | 下界 |
| upper_bound | int | 否 | 上界 |
| baseline_value | int | 否 | 基线值 |
| reliability_level | string | 否 | 预测可信度等级 high/medium/low |
| model_version | string | 是 | 模型版本 |
| run_id | string | 否 | 本次运行 ID |
| generated_at | datetime | 是 | 生成时间 |
| created_at | datetime | 是 | 创建时间 |
| updated_at | datetime | 是 | 更新时间 |

### 6.2 dispatch_plan
用途：保存每次调度引擎输出的整体方案头信息。

| 字段 | 类型建议 | 必填 | 说明 |
|---|---|---:|---|
| plan_id | string | 是 | 方案主键 |
| planning_window_start | datetime | 是 | 规划开始时间 |
| planning_window_end | datetime | 是 | 规划结束时间 |
| plan_scope_region | string | 否 | 方案区域范围 |
| pallet_type | string | 否 | 覆盖托盘类型 |
| objective_score | decimal(12,4) | 否 | 综合评分 |
| shortage_risk_score | decimal(12,4) | 否 | 缺口风险评分 |
| total_transfer_cost | decimal(12,2) | 否 | 预计调拨成本 |
| service_level_score | decimal(12,4) | 否 | 服务水平评分 |
| plan_status | string | 是 | 方案状态，复用 `dispatch_plan_status` |
| heuristic_version | string | 否 | 启发式版本 |
| generated_by | string | 否 | 生成方式/任务 |
| generated_at | datetime | 是 | 生成时间 |
| created_at | datetime | 是 | 创建时间 |
| updated_at | datetime | 是 | 更新时间 |

## 7. 关系说明
- `warehouse` / `depot` / `customer` / `vehicle` / `route_lane` 组成主数据层。
- `inventory_snapshot`、`pallet` 组成状态层。
- `order_demand` 作为需求事实输入，也是预测训练与评估的重要来源。
- `forecast_result` 作为缺口识别和调拨推荐输入。
- `dispatch_plan` 是调度方案头；`transfer_task` 是方案明细与审批执行主体。

## 8. MVP 最小必备表
若客户数据条件有限，MVP 至少落地以下表：
1. `warehouse`
2. `depot`
3. `customer`
4. `vehicle`
5. `route_lane`
6. `inventory_snapshot`
7. `order_demand`
8. `forecast_result`
9. `dispatch_plan`
10. `transfer_task`

`pallet` 表可作为增强项，在后续接入单托盘级事件后再逐步完善。

## 9. 一致性约束建议
- 所有状态字段统一使用本文定义的枚举，不允许接口层自行扩展自由文本。
- 所有时间窗统一使用 `[window_start_time, window_end_time)` 口径。
- 所有数量字段默认单位为“个托盘”。
- 所有位置字段必须同时带 `location_id` 与 `location_type`，避免仓库与 depot 编号冲突。
- `inventory_snapshot.snapshot_time`、`forecast_result.generated_at`、`dispatch_plan.generated_at` 应作为审计关键时间。

## 10. 待客户确认
- 是否可以拿到单托盘级 `pallet_id` 与事件流；若不能，MVP 先以聚合库存实现。
- `warehouse` 与 `depot` 是否已有统一编码体系。
- 订单是否一定存在来源仓，还是仅有区域/客户侧需求。
- 车辆与路线成本是否已有结构化表，还是需要先人工维护。
- 调拨执行是否需要对接外部 TMS 任务号。