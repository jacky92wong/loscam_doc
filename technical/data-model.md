# 数据模型

## 1. 建模原则
数据模型应服务于三个核心任务：
1. 看清当前库存与状态
2. 预测未来需求
3. 生成可执行的调拨方案

同时需要兼顾两个目标：
- **客户可读**：能让业务方理解“系统为什么这样判断”
- **开发可用**：能直接指导后续 DDL、API 和脚本原型

## 2. 核心实体
### 2.1 Warehouse / Depot
字段示例：
- id
- name
- region
- geo_location
- storage_capacity
- operation_hours
- pallet_type_support

### 2.2 Pallet
字段示例：
- pallet_id
- pallet_type
- ownership_pool
- current_location_id
- current_status
- last_event_time
- quality_grade

### 2.3 Inventory Snapshot
用于按时间窗聚合库存：
- location_id
- pallet_type
- available_qty
- reserved_qty
- in_use_qty
- in_transit_qty
- available_soon_qty
- maintenance_qty

### 2.4 Order / Demand
- order_id
- customer_id
- source_location
- destination_location
- requested_delivery_time
- pallet_type
- demand_qty
- priority
- status

### 2.5 Customer
- customer_id
- customer_segment
- industry
- service_level
- historical_return_behavior

### 2.6 Vehicle
- vehicle_id
- vehicle_type
- pallet_capacity
- region_scope
- available_time_window
- cost_rule

### 2.7 Route / Lane
- origin
- destination
- distance
- duration
- base_cost
- toll_cost
- service_frequency

### 2.8 Transfer Task
- task_id
- from_location
- to_location
- planned_qty
- planned_departure_time
- assigned_vehicle
- status

### 2.9 Forecast Result
- forecast_id
- target_location
- target_time_window
- predicted_demand
- lower_bound
- upper_bound
- model_version

### 2.10 Dispatch Plan
- plan_id
- planning_window
- objective_score
- shortage_risk
- total_transfer_cost
- recommended_actions

## 3. 实体关系说明
从 MVP 角度，关键关系如下：
- `warehouse` / `depot` / `customer` / `vehicle` / `route_lane` 属于主数据层
- `order_demand` 记录未来或历史需求事实，是预测输入
- `inventory_snapshot` 记录聚合库存状态，是当前供给基础
- `pallet` 若存在，则补充单托盘级状态与事件追踪能力
- `forecast_result` 输出未来需求估计
- `dispatch_plan` 汇总一次调度运行结果
- `transfer_task` 是计划下的执行明细

可以理解为：
**主数据定义网络 → 状态数据描述当前供给 → 需求数据描述未来消耗 → 算法数据输出推荐动作。**

### 3.1 建议补充的事件实体
若客户后续能提供更细粒度过程数据，建议补充：
- `inventory_event`：状态变化事件
- `iot_tracking_event`：位置与轨迹事件
- `transfer_execution_event`：调拨执行过程事件
- `forecast_run_log`：预测任务运行记录
- `dispatch_run_log`：调拨任务运行记录

### 3.2 为什么运行日志也是模型的一部分
因为算法系统不只是表结构，还需要回答：
- 这次预测是何时跑的
- 用的是哪个版本
- 输入数据批次是什么
- 为什么本次方案和上次不同

没有运行日志，后续很难复盘模型偏差和客户投诉。

## 4. 为什么采用“事件 + 快照”并存
### 4.1 只看快照的问题
快照能回答“某时刻库存是多少”，但无法解释：
- 为什么库存变了
- 变化发生在何时
- 某批托盘何时可能恢复可用

### 4.2 只看事件的问题
事件能表达过程，但对 MVP 来说，若没有稳定聚合视图，业务方很难快速得到“当前可调拨库存”。

### 4.3 并存的好处
因此建议：
- 用 **事件** 表达状态变化、回流、维修、IoT 修正等过程
- 用 **快照** 提供算法和页面消费的稳定视图

这样既支持可审计，也支持高效推荐。

## 5. 状态模型
建议统一托盘状态：
- `available`
- `reserved`
- `in_use`
- `in_transit`
- `available_soon`
- `maintenance`
- `lost`
- `suspected_lost`

### 5.1 状态含义
- `available`：当前可立即用于订单或调拨
- `reserved`：已被未来需求占用，不应再次推荐
- `in_use`：在客户或业务节点中使用中
- `in_transit`：运输途中
- `available_soon`：当前不可立即使用，但预计在短时间内恢复可用
- `maintenance`：维修、清洗、待检，不可直接使用
- `lost`：已确认丢失
- `suspected_lost`：长时间失联或高风险异常，默认不进入供给集合

### 5.2 最小状态机思路
典型流转路径可概括为：
`available -> reserved -> in_transit -> in_use -> available_soon -> available`

异常路径：
- `in_use -> suspected_lost`
- `available_soon -> maintenance`
- `maintenance -> available`
- 任意状态在确认损失后可转 `lost`

MVP 不必把所有分支全部自动化，但必须保证状态语义一致。

## 6. 时间维度
必须保留：
- 事件时间
- 计划时间
- 可用时间
- 实际执行时间

否则难以支持预测与滚动优化。

### 6.1 为什么需要区分不同时间
- **事件时间**：真实发生时刻，用于审计和状态重建
- **计划时间**：系统建议或业务计划时刻，用于推荐与审批
- **可用时间**：托盘何时能重新进入供给池
- **实际执行时间**：用于复盘推荐与执行偏差

### 6.2 时间窗口约定
建议统一采用半开区间：
`[window_start_time, window_end_time)`

这样可避免相邻窗口重复计算。

## 7. 位置维度
至少支持三层：
- 区域
- 仓库 / depot
- 客户 / 业务节点

### 7.1 为什么需要层级位置
因为项目既要回答“华东是否缺托盘”，也要回答“具体哪个仓缺、哪个 depot 可补”。

### 7.2 MVP 建议
首期至少把 `region_code + location_id + location_type` 建清楚，避免：
- depot 与 warehouse 编号冲突
- 客户点位与仓库点位混淆
- 区域级汇总无法做

## 8. 供给、需求与缺口定义
### 8.1 需求
需求一般来自已确认订单、补货计划或预测结果。

建议统一区分三类口径：
- `confirmed_demand`：已确认订单或明确计划
- `forecasted_demand`：模型预测需求
- `protected_demand`：用于来源仓保护的近窗需求

### 8.2 供给
对调拨模块来说，供给不等于账面总库存，而应更接近：
- 当前 `available`
- 规划窗内可释放的 `available_soon`
- 已确认在途入库量

### 8.3 缺口
缺口不是简单的“需求 - available_qty”，而应在统一时间窗内计算：
`forecast_gap = predicted_demand - expected_supply`

其中 `expected_supply` 应根据状态、到达时间和可信度过滤得到。

### 8.4 推荐的 expected_supply 计算口径
建议：
`expected_supply = available_qty + releasable_available_soon_qty + confirmed_inbound_qty - reserved_qty`

其中：
- `releasable_available_soon_qty`：在规划窗口内预计可恢复可用的数量
- `confirmed_inbound_qty`：到达时间落入窗口且可信度达标的在途量
- `reserved_qty`：已被高优先级需求锁定的数量

### 8.5 为什么不能直接拿账面总库存做供给
因为账面库存常常包含：
- 已经被占用的库存
- 维修中的库存
- 长时间未确认位置的库存
- 到不了当前窗口的在途库存

如果不做过滤，调拨推荐会系统性高估供给。

## 9. 建议的数据分层
- 主数据层：仓库、客户、车辆、路线、托盘类型
- 交易层：订单、账户转移、回收、调拨任务
- 状态层：库存快照、托盘状态、IoT 事件
- 算法层：预测结果、优化结果、执行反馈

这种分层有助于后续分清：哪些字段是业务源头，哪些字段是系统推导结果。

## 10. MVP 建模边界
当前阶段优先支持：
- 聚合库存建模
- 统一状态枚举
- 日级 / 周级需求预测
- 仓 / depot 级调拨推荐

当前不要求：
- 全量单托盘追踪必备
- 全自动实时状态机
- 复杂多级责任结算模型

## 11. 待客户确认
- 是否能拿到单托盘级数据，还是仅能拿聚合库存
- 车辆与路线是否有结构化数据
- 库存快照更新频率
- 是否已有唯一主键体系贯通多个系统
- `available_soon` 的业务定义与可用时间口径如何确定
