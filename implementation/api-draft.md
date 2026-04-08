# API 草案

## 1. 说明
本文件基于当前发布版的实施层数据契约与 MVP 主闭环范围，细化接口草案。当前目标不是形成最终 OpenAPI 文件，而是统一接口用途、字段、幂等、筛选、错误码和回写逻辑，供客户确认与后续开发使用。

## 2. API 设计原则
- 优先覆盖最小闭环：订单输入、库存输入、预测查询、调拨建议查询、调拨审批、执行反馈。
- 写接口优先支持批量导入与幂等。
- 查询接口优先支持区域、节点、时间窗与托盘类型筛选。
- 推荐结果必须可解释、可审计、可回写。
- MVP 保留人工审批节点，不直接自动下发执行。

## 3. 通用规范
### 3.1 认证
本稿暂不限定认证实现，后续可选：
- 内网 SSO
- API Key
- JWT

### 3.2 通用请求头建议
- `Content-Type: application/json`
- `X-Request-Id`: 请求追踪 ID
- `Idempotency-Key`: 幂等键，写接口必填

### 3.3 通用分页参数
查询类接口统一支持：
- `page_no`
- `page_size`
- `sort_by`
- `sort_order`

### 3.4 通用错误码
- `VALIDATION_ERROR`：字段校验失败
- `DUPLICATE_REQUEST`：重复提交
- `NOT_FOUND`：对象不存在
- `STATE_CONFLICT`：状态冲突
- `UPSTREAM_ERROR`：外部系统失败
- `INTERNAL_ERROR`：内部异常

## 4. 订单输入接口

### 4.1 POST /orders/import
用途：导入订单或需求计划，写入 `order_demand`。

#### 请求体
复用 `implementation/02-data-contracts.md` 的订单导入契约。

#### 响应体
```json
{
  "batch_id": "ORD-20260408-001",
  "accepted_count": 120,
  "rejected_count": 3,
  "errors": [
    {
      "row_no": 17,
      "field": "requested_delivery_time",
      "reason": "invalid_time_sequence"
    }
  ]
}
```

#### 幂等规则
- 使用 `Idempotency-Key` 或请求体内 `idempotency_key`。
- 相同幂等键重复提交时，不重复写入，只返回首次处理结果。

#### 回写逻辑
- 成功导入后生成 `import_batch_id`。
- 可选回写至来源系统：导入结果、失败行、失败原因。

## 5. 库存输入接口

### 5.1 POST /inventory/snapshots/import
用途：导入库存快照，写入 `inventory_snapshot`。

#### 请求体
复用库存快照契约。

#### 响应体
```json
{
  "batch_id": "INV-20260408-0900",
  "accepted_count": 48,
  "rejected_count": 2,
  "errors": []
}
```

#### 幂等规则
- 以 `Idempotency-Key + snapshot_time + source_system` 作为去重依据。

#### 回写逻辑
- 成功写入后更新“最新可信库存视图”。
- 若后续接入状态修正任务，可触发库存一致性检查。

## 6. 托盘状态事件接口

### 6.1 POST /events/pallet-status
用途：导入托盘状态事件、回收事件、IoT 位置事件。

#### 请求体
复用托盘状态事件契约。

#### 响应体
```json
{
  "batch_id": "EVT-20260408-01",
  "accepted_count": 560,
  "rejected_count": 4,
  "errors": []
}
```

#### 幂等规则
- `event_id` 全局唯一。
- 相同 `event_id` 重复提交直接返回已处理结果。

#### 回写逻辑
- 若存在单托盘 `pallet_id`，更新 `pallet` 状态。
- 若只有聚合事件，则进入库存修正任务。
- 可触发异常规则，如疑似丢失、长时间未回流。

## 7. 预测查询接口

### 7.1 GET /forecast/demand
用途：查询预测服务输出的需求预测结果。

#### 查询参数
- `region_code`：区域
- `location_id`：节点 ID
- `location_type`：节点类型
- `pallet_type`：托盘类型
- `window_start_time`
- `window_end_time`
- `time_granularity`：`day` / `week`
- `model_version`：可选
- `page_no`
- `page_size`

#### 响应体
```json
{
  "page_no": 1,
  "page_size": 20,
  "total": 2,
  "items": [
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
      "reliability_level": "medium",
      "model_version": "baseline_lgbm_v1",
      "generated_at": "2026-04-08T11:00:00+08:00"
    }
  ]
}
```

#### 说明
- 页面侧可基于 `predicted_demand` 和库存视图计算缺口。
- `lower_bound`、`upper_bound` 用于缺口风险展示。

## 8. 调拨建议查询接口

### 8.1 GET /dispatch/recommendations
用途：查询调度引擎输出的调拨方案与任务。

#### 查询参数
- `plan_id`
- `region_code`
- `status`
- `from_location_id`
- `to_location_id`
- `pallet_type`
- `planning_window_start`
- `planning_window_end`
- `page_no`
- `page_size`

#### 响应体
```json
{
  "page_no": 1,
  "page_size": 20,
  "total": 1,
  "plans": [
    {
      "plan_id": "DP-20260408-01",
      "planning_window_start": "2026-04-08T12:00:00+08:00",
      "planning_window_end": "2026-04-10T00:00:00+08:00",
      "plan_scope_region": "EAST",
      "objective_score": 82.5,
      "shortage_risk_score": 0.21,
      "total_transfer_cost": 12800,
      "plan_status": "generated",
      "heuristic_version": "rule_heuristic_v1",
      "tasks": [
        {
          "task_id": "TASK-001",
          "from_location_id": "WH-SH-01",
          "to_location_id": "WH-HZ-01",
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
            "未来24小时缺口高",
            "来源仓可用库存充足",
            "路线成本可接受"
          ]
        }
      ]
    }
  ]
}
```

#### 说明
- MVP 默认返回方案头 + 任务明细。
- 后续若数据量大，可拆为 `/dispatch/plans` 与 `/dispatch/tasks`。
- 若关键输入数据不足，接口可返回 `recommendation_mode = alert_only`，仅输出风险预警和数据阻塞原因，不输出正式任务列表。

#### 降级返回样例
```json
{
  "plan_id": "DP-20260408-02",
  "recommendation_mode": "alert_only",
  "plan_reliability_level": "low",
  "warning_summary": [
    "库存快照超过时效阈值",
    "ETA 数据可信度不足"
  ],
  "data_quality_blockers": [
    "stale_inventory_snapshot",
    "eta_confidence_low"
  ],
  "plans": []
}
```

## 9. 调拨审批接口

### 9.1 POST /dispatch/approve
用途：对推荐任务执行审批、拒绝或改量。

#### 请求体
复用调拨审批反馈契约。

#### 响应体
```json
{
  "plan_id": "DP-20260408-01",
  "processed_count": 1,
  "success_count": 1,
  "failed_count": 0,
  "results": [
    {
      "task_id": "TASK-001",
      "status": "approved",
      "approved_qty": 180
    }
  ]
}
```

#### 幂等规则
- 同一 `Idempotency-Key` 不能重复应用审批动作。
- 已进入 `scheduled`、`in_execution`、`completed` 的任务不能再次审批。

#### 状态流转
- `pending_approval` → `approved`
- `pending_approval` → `rejected`
- `pending_approval` → `approved`（但 `approved_qty < planned_qty`，视为改量通过）

#### 回写逻辑
- 审批通过后更新 `transfer_task.approved_qty`、`approved_by`、`approved_at`。
- 若后续对接 TMS，可在通过后触发任务下发。
- 页面需保留原始建议值与审批值对比。

## 10. 执行反馈接口

### 10.1 POST /dispatch/feedback
用途：回写调拨任务执行结果。

#### 请求体
复用执行反馈契约。

#### 响应体
```json
{
  "processed_count": 1,
  "success_count": 1,
  "failed_count": 0,
  "results": [
    {
      "task_id": "TASK-001",
      "status": "completed",
      "actual_qty": 180
    }
  ]
}
```

#### 幂等规则
- 相同 `Idempotency-Key` 重放时返回历史结果。
- 若同一任务已有终态 `completed` / `cancelled` / `failed`，重复写入应返回 `STATE_CONFLICT`。

#### 回写逻辑
- 更新 `transfer_task` 的实际执行字段。
- 触发计划 vs 实际偏差记录。
- 后续可进入 KPI 与推荐采纳率统计。

## 11. 内部任务触发接口

### 11.1 POST /forecast/run
用途：触发预测任务运行。

#### 请求体
```json
{
  "region_code": "EAST",
  "pallet_type": "STD",
  "time_granularity": "day",
  "window_days": 7,
  "trigger_mode": "manual"
}
```

#### 响应体
```json
{
  "run_id": "FC-20260408-01",
  "status": "accepted"
}
```

### 11.2 POST /optimization/run
用途：触发调拨推荐任务。

#### 请求体
```json
{
  "region_code": "EAST",
  "planning_window_start": "2026-04-08T12:00:00+08:00",
  "planning_window_end": "2026-04-10T00:00:00+08:00",
  "pallet_type": "STD",
  "trigger_mode": "manual"
}
```

#### 响应体
```json
{
  "plan_id": "DP-20260408-01",
  "status": "accepted"
}
```

## 12. 推荐错误返回样例
```json
{
  "code": "STATE_CONFLICT",
  "message": "task TASK-001 is already completed",
  "details": [
    {
      "task_id": "TASK-001",
      "current_status": "completed"
    }
  ]
}
```

## 13. MVP 最小闭环映射
- 订单输入：`POST /orders/import`
- 库存输入：`POST /inventory/snapshots/import`
- 状态修正输入：`POST /events/pallet-status`
- 预测输出：`GET /forecast/demand`
- 调拨推荐输出：`GET /dispatch/recommendations`
- 审批反馈：`POST /dispatch/approve`
- 执行回写：`POST /dispatch/feedback`

## 14. 待客户确认
- 订单与库存输入优先采用文件导入还是系统 API 对接。
- 审批通过后是否立即对接 TMS 下发执行。
- 查询接口是否需要按区域经理、调度员做数据权限隔离。
- 是否需要保留人工新建调拨任务接口。
- 是否要求实时事件接口，还是 MVP 先接受批量导入。