# 数据契约草案

## 1. 说明
本文件定义 MVP 闭环内的关键输入输出数据契约，供后续 API、批量导入、数据同步和页面展示统一口径。所有字段命名与状态枚举以 `implementation/01-logical-schema.md` 为准。

## 2. 通用约定
- 所有时间字段使用 ISO 8601 格式，例如 `2026-04-08T10:00:00+08:00`。
- 所有数量单位默认是“个托盘”。
- 所有批量写入请求建议携带 `batch_id`。
- 所有幂等写入请求建议携带 `idempotency_key`。
- 所有节点字段统一包含 `location_id` 和 `location_type`。
- 所有状态字段必须使用受控枚举，不接受自由文本。

## 3. 订单导入契约
用途：导入未来订单、出货计划或需求计划，形成 `order_demand` 输入。

### 3.1 请求结构
```json
{
  "batch_id": "ORD-20260408-001",
  "source_system": "OMS",
  "idempotency_key": "orders-import-20260408-001",
  "orders": [
    {
      "order_id": "SO-10001",
      "external_order_no": "EXT-9001",
      "customer_id": "CUST-001",
      "source_location_id": "WH-SH-01",
      "destination_location_id": "CUST-SITE-01",
      "destination_location_type": "customer_site",
      "order_date": "2026-04-08T08:30:00+08:00",
      "requested_delivery_time": "2026-04-09T12:00:00+08:00",
      "region_code": "EAST",
      "pallet_type": "STD",
      "demand_qty": 320,
      "priority": "high",
      "status": "confirmed"
    }
  ]
}
```

### 3.2 校验规则
- `orders` 不能为空。
- `order_id` 在同一来源系统内必须唯一。
- `demand_qty` 必须大于 0。
- `requested_delivery_time` 不得早于 `order_date`。
- `status` 只能取 `draft`、`confirmed`、`allocated`、`shipped`、`completed`、`cancelled`。

### 3.3 成功响应样例
```json
{
  "batch_id": "ORD-20260408-001",
  "accepted_count": 1,
  "rejected_count": 0,
  "errors": []
}
```

## 4. 库存快照导入契约
用途：导入仓库 / depot 当前库存聚合视图，形成 `inventory_snapshot`。

### 4.1 请求结构
```json
{
  "batch_id": "INV-20260408-0900",
  "source_system": "WMS",
  "snapshot_time": "2026-04-08T09:00:00+08:00",
  "idempotency_key": "inventory-20260408-0900",
  "snapshots": [
    {
      "location_id": "WH-SH-01",
      "location_type": "warehouse",
      "region_code": "EAST",
      "pallet_type": "STD",
      "available_qty": 800,
      "reserved_qty": 120,
      "in_use_qty": 300,
      "in_transit_qty": 90,
      "available_soon_qty": 60,
      "maintenance_qty": 20,
      "lost_qty": 3,
      "suspected_lost_qty": 8,
      "blocked_qty": 15
    }
  ]
}
```

### 4.2 校验规则
- `snapshot_time` 为整批统一时间。
- 所有数量字段必须大于等于 0。
- `location_type` 必须属于 `warehouse`、`depot`、`customer_site`、`service_point`、`transit_node`。
- 同一批次内 `location_id + pallet_type` 不可重复。

### 4.3 成功响应样例
```json
{
  "batch_id": "INV-20260408-0900",
  "accepted_count": 1,
  "rejected_count": 0,
  "errors": []
}
```

## 5. 托盘状态事件契约
用途：接收入池、回收、维修、IoT 到站、状态变更等事件，用于修正库存或单托盘状态。

### 5.1 请求结构
```json
{
  "batch_id": "EVT-20260408-01",
  "source_system": "IOT",
  "idempotency_key": "event-IOT-889900",
  "events": [
    {
      "event_id": "IOT-889900",
      "event_time": "2026-04-08T10:15:00+08:00",
      "pallet_id": "PLT-0001",
      "pallet_type": "STD",
      "from_location_id": "CUST-SITE-01",
      "from_location_type": "customer_site",
      "to_location_id": "DEPOT-SH-02",
      "to_location_type": "depot",
      "event_type": "returned",
      "new_status": "available_soon",
      "expected_available_time": "2026-04-08T18:00:00+08:00",
      "quantity": 1,
      "metadata": {
        "device_id": "TAG-001",
        "source_message_id": "mqtt-001"
      }
    }
  ]
}
```

### 5.2 事件类型建议
- `dispatched`
- `arrived`
- `returned`
- `inspected`
- `repaired`
- `cleaned`
- `status_corrected`
- `iot_position_updated`
- `lost_confirmed`
- `suspected_lost_flagged`

### 5.3 校验规则
- `event_id` 必须唯一。
- `event_time` 必填。
- `new_status` 必须属于统一托盘状态枚举。
- 若没有 `pallet_id`，则必须提供 `quantity` 和聚合级位置信息。

## 6. 预测结果输出契约
用途：输出预测服务的日级或周级需求预测结果，落入 `forecast_result`。

### 6.1 响应结构
```json
{
  "run_id": "FC-20260408-01",
  "model_version": "baseline_lgbm_v1",
  "generated_at": "2026-04-08T11:00:00+08:00",
  "time_granularity": "day",
  "results": [
    {
      "forecast_id": "FC-ROW-001",
      "target_location_id": "WH-HZ-01",
      "target_location_type": "warehouse",
      "region_code": "EAST",
      "pallet_type": "STD",
      "window_start_time": "2026-04-09T00:00:00+08:00",
      "window_end_time": "2026-04-10T00:00:00+08:00",
      "predicted_demand": 420,
      "lower_bound": 360,
      "upper_bound": 490,
      "baseline_value": 398,
      "reliability_level": "medium"
    }
  ]
}
```

### 6.2 使用说明
- `predicted_demand` 用于缺口识别。
- `lower_bound`、`upper_bound` 用于风险提示和安全库存边界。
- `baseline_value` 用于页面对比与模型评估展示。
- `reliability_level` 用于判断预测结果是否可作为强输入，还是只应作为弱参考。

## 7. 调拨建议输出契约
用途：输出调度引擎生成的推荐方案与任务明细。

### 7.1 响应结构
```json
{
  "plan_id": "DP-20260408-01",
  "planning_window_start": "2026-04-08T12:00:00+08:00",
  "planning_window_end": "2026-04-10T00:00:00+08:00",
  "plan_scope_region": "EAST",
  "objective_score": 82.5,
  "shortage_risk_score": 0.21,
  "total_transfer_cost": 12800,
  "heuristic_version": "rule_heuristic_v1",
  "tasks": [
    {
      "task_id": "TASK-001",
      "from_location_id": "WH-SH-01",
      "from_location_type": "warehouse",
      "to_location_id": "WH-HZ-01",
      "to_location_type": "warehouse",
      "pallet_type": "STD",
      "planned_qty": 200,
      "planned_departure_time": "2026-04-08T15:00:00+08:00",
      "planned_arrival_time": "2026-04-08T21:00:00+08:00",
      "source_window_start": "2026-04-08T12:00:00+08:00",
      "source_window_end": "2026-04-09T00:00:00+08:00",
      "target_arrival_window_start": "2026-04-08T18:00:00+08:00",
      "target_arrival_window_end": "2026-04-09T00:00:00+08:00",
      "assigned_vehicle_id": "VEH-01",
      "lane_id": "LANE-SH-HZ",
      "priority_score": 91.2,
      "reason_code": "forecast_shortage_high",
      "reason_codes": ["forecast_shortage_high", "source_risk_controlled", "eta_within_window"],
      "risk_flags": ["tight_time_window"],
      "eta_confidence": "medium",
      "status": "pending_approval",
      "explanations": [
        "目标仓未来24小时预测缺口高",
        "来源仓当前available_qty充足",
        "路线成本在可接受范围内"
      ]
    }
  ]
}
```

## 8. 调拨审批反馈契约
用途：运营调度人员对推荐任务进行通过、拒绝或改量反馈。

### 8.1 请求结构
```json
{
  "idempotency_key": "approve-20260408-01",
  "plan_id": "DP-20260408-01",
  "approver_id": "ops_user_01",
  "approved_at": "2026-04-08T12:30:00+08:00",
  "decisions": [
    {
      "task_id": "TASK-001",
      "decision": "approved",
      "approved_qty": 180,
      "comment": "调减20个，避免影响来源仓晚班出货"
    }
  ]
}
```

### 8.2 decision 枚举
- `approved`
- `rejected`
- `adjusted`

### 8.3 规则
- `approved_qty` 不能大于 `planned_qty`。
- `decision = rejected` 时建议填写 `comment`。
- 系统应保留原始推荐值，不能被审批值直接覆盖。

## 9. 执行反馈契约
用途：回写调拨执行结果，形成任务闭环与复盘依据。

### 9.1 请求结构
```json
{
  "idempotency_key": "feedback-20260408-01",
  "task_feedbacks": [
    {
      "task_id": "TASK-001",
      "status": "completed",
      "actual_departure_time": "2026-04-08T15:20:00+08:00",
      "actual_arrival_time": "2026-04-08T21:40:00+08:00",
      "actual_qty": 180,
      "feedback_note": "车辆晚到20分钟，已完成签收"
    }
  ]
}
```

### 9.2 规则
- `status` 必须属于 `scheduled`、`in_execution`、`completed`、`cancelled`、`failed`。
- 已完成任务必须填写 `actual_qty`。
- 若 `status = failed`，建议补充失败原因。

## 10. 通用错误返回结构
```json
{
  "code": "VALIDATION_ERROR",
  "message": "requested_delivery_time must be later than order_date",
  "details": [
    {
      "field": "orders[0].requested_delivery_time",
      "reason": "invalid_time_sequence"
    }
  ]
}
```

## 11. 版本与审计建议
- 所有契约建议保留 `source_system`、`batch_id`、`run_id`、`idempotency_key`。
- 预测与调度输出应保留算法版本号。
- 所有审批和执行反馈应保留操作人、操作时间和原始输入，便于审计。

## 12. 待客户确认
- 订单导入是按单据粒度还是按日计划粒度提供。
- 库存快照是实时推送、定时导入，还是数据库只读同步。
- 状态事件是否能提供单托盘 `pallet_id`。
- 审批反馈是否需要双人复核或区域经理加签。
- 执行反馈由 TMS 回写还是人工录入。