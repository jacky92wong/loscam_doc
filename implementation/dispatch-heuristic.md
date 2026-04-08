# 调拨启发式实施稿

## 1. 说明
本文件把 `../technical/optimization-model.md` 中的目标、约束与算法路径，收敛为 MVP 可执行的规则 + 启发式方案。当前明确不做复杂 LP/VRP 求解器，优先保证推荐可解释、可落地、可人工审批。

## 2. MVP 调拨目标
在给定预测需求、当前库存、路线与运力约束的前提下，输出一组跨仓调拨推荐任务，优先减少缺口和紧急补货，同时控制运输成本与对来源仓的副作用。

## 3. 输入
### 3.1 必要输入
- `inventory_snapshot`
- `forecast_result`
- `warehouse`
- `depot`
- `vehicle`
- `route_lane`
- `order_demand`（可用于识别已确认高优先级需求）

### 3.2 关键中间量
- `expected_supply`
- `forecast_gap`
- `movable_qty`
- `lane_cost_per_pallet`
- `arrival_feasibility`

## 4. 缺口与冗余定义
### 4.1 目标仓缺口
```text
forecast_gap = predicted_demand - (available_qty + releasable_available_soon_qty + confirmed_inbound_qty - reserved_qty)
```
当 `forecast_gap > 0` 时，视为候选缺口节点。

### 4.2 来源仓可调拨量
```text
movable_qty = available_qty + releasable_available_soon_qty - safety_buffer - reserved_qty_impact
```
若 `movable_qty <= 0`，则该节点不可作为来源仓。

说明：
- `safety_buffer` 用于保护来源仓自身未来需求。
- `reserved_qty_impact` 用于避免把已承诺订单库存错误拿去调拨。
- `releasable_available_soon_qty` 必须限定在规划窗口内可恢复可用的数量。

### 4.3 多时间窗下的补充口径
若后续扩展到多时间窗，则每个窗口都应单独计算：
- `expected_supply_(l,p,t)`
- `forecast_gap_(l,p,t)`
- `carry_over_gap_(l,p,t)`
- `movable_qty_(i,p,t)`

并且要求：
- 上一窗口未满足的缺口进入下一窗口
- 调拨量只有在 ETA 落入目标窗口后才能计入供给
- `available_soon` 必须按预计可用时间映射到对应窗口
- `confirmed_inbound` 必须按确认到达时间映射到对应窗口

### 4.4 多时间窗下 `movable_qty` 的 implementation 口径
实现时建议按窗口计算：
```text
movable_qty_(i,p,t) = max(
  0,
  on_hand_available_(i,p,t)
  + releasable_available_soon_qty_(i,p,t)
  + confirmed_inbound_qty_(i,p,t)
  - source_protection_threshold_(i,p,t)
  - reserved_qty_impact_(i,p,t)
)
```

不要直接复用单时间窗的静态 `movable_qty`。

### 4.5 多时间窗下 `expected_supply` 的 implementation 口径
```text
expected_supply_(l,p,t) =
  on_hand_available_(l,p,t)
  + releasable_available_soon_qty_(l,p,t)
  + confirmed_inbound_qty_(l,p,t)
  + inbound_transfer_(l,p,t)
  - reserved_qty_impact_(l,p,t)
  - blocked_qty_(l,p,t)
```

### 4.6 多时间窗下 `forecast_gap` 的 implementation 口径
```text
forecast_gap_(l,p,t) = max(
  0,
  demand_requirement_(l,p,t)
  + carry_over_gap_(l,p,t)
  - expected_supply_(l,p,t)
)
```

其中：
```text
demand_requirement_(l,p,t) = confirmed_demand_(l,p,t) + alpha * forecasted_demand_(l,p,t)
```

### 4.7 implementation 提醒
若 `alpha` 设置过高，而预测质量还不稳定，系统会更容易过度调拨；若 `alpha` 太低，则系统又会退回只对已确认订单被动响应。MVP 试点建议把 `alpha` 作为可调参数保留。

## 5. 主要约束过滤
在打分前，先做硬约束过滤：

1. 来源仓 `movable_qty > 0`
2. 目标仓 `forecast_gap > 0`
3. 来源与目标不能相同
4. 目标仓接收后不能超过容量上限
5. 路线必须存在且可用
6. 到达时间必须落在可接受服务窗内
7. 车辆容量必须满足最小推荐量
8. `maintenance`、`lost`、`suspected_lost` 库存不能进入可调拨集合

## 6. 候选任务生成逻辑
```text
for each target_location with forecast_gap > 0:
    for each source_location with movable_qty > 0:
        if pass hard constraints:
            candidate_qty = min(forecast_gap, movable_qty, vehicle_capacity_limit)
            create candidate task
```

候选任务的最小字段：
- 来源节点
- 目标节点
- 托盘类型
- 建议数量
- 建议发车时间
- 预计到达时间
- 候选车辆/路线

### 6.1 多时间窗下的候选生成补充
若进入多时间窗模式，则候选任务还应额外带上：
- `source_window`
- `target_arrival_window`
- `eta_arrival`
- `carry_over_gap_reference`

否则后续无法判断：
- 这条任务是在解决哪个窗口的缺口
- 它的供给何时真正生效

## 7. 打分项设计
MVP 不求全局最优，先采用加权打分：

```text
score = shortage_priority_score
      + service_urgency_score
      + source_surplus_score
      - transfer_cost_score
      - distance_penalty_score
      - source_risk_penalty
      - delay_penalty_score
```

### 7.1 shortage_priority_score
目标仓缺口越大，分越高。

### 7.2 service_urgency_score
需求窗口越近、客户 SLA 越高，分越高。

### 7.3 source_surplus_score
来源仓剩余越充足，分越高。

### 7.4 transfer_cost_score
单位成本越高，扣分越多。

### 7.5 distance_penalty_score
距离越远、运输时间越长，扣分越多。

### 7.6 source_risk_penalty
调出后若来源仓自身也会产生缺口，则重罚。

### 7.7 delay_penalty_score
预计无法按时到达或接近时间窗边界时扣分。

### 7.8 推荐初始权重口径
首版可由业务人工设权，例如：
- `w1 shortage_priority = 0.30`
- `w2 service_urgency = 0.20`
- `w3 source_surplus = 0.15`
- `w4 transfer_cost = 0.15`
- `w5 distance = 0.08`
- `w6 source_risk = 0.07`
- `w7 delay = 0.05`

该权重不是定值，而是首轮客户讨论口径，用于把“服务优先还是成本优先”显式化。

## 8. 打分伪代码
```text
for each candidate_task:
    shortage_priority_score = normalize(target forecast_gap)
    service_urgency_score = normalize(urgency by requested delivery window and SLA)
    source_surplus_score = normalize(source movable_qty)
    transfer_cost_score = normalize(lane cost per pallet)
    distance_penalty_score = normalize(distance_km or duration_hours)
    source_risk_penalty = high if source post-transfer risk > threshold else low
    delay_penalty_score = high if estimated arrival near or after deadline else low

    final_score = w1*shortage_priority_score
                + w2*service_urgency_score
                + w3*source_surplus_score
                - w4*transfer_cost_score
                - w5*distance_penalty_score
                - w6*source_risk_penalty
                - w7*delay_penalty_score
```

MVP 建议先人工设权重，不先做自动学习权重。

### 8.1 normalize 的推荐口径
为了避免不同量纲直接相加，建议统一把连续指标压到 `[0,1]`：

```text
normalize(value) = (value - lower_bound) / (upper_bound - lower_bound)
clip to [0,1]
```

示例：
- `forecast_gap` 可按当前规划批次内最小/最大缺口归一化
- `lane cost per pallet` 可按当前区域成本分布归一化
- `distance` 可按候选 lane 距离分布归一化

### 8.2 二值风险项的推荐口径
对不适合连续归一化的风险项，可直接设：
- `low = 0`
- `medium = 0.5`
- `high = 1`

这样更适合 MVP 快速落地。

## 9. 任务选择策略
### 9.1 单目标仓排序
每个缺口仓先按 `final_score` 从高到低排序候选任务。

### 9.2 全局选择
```text
sort all candidate tasks by final_score desc
for task in sorted candidates:
    if source movable_qty still enough
       and target gap still not fully covered
       and vehicle/lane capacity still available:
        accept task
        deduct source movable_qty
        deduct target forecast_gap
        update resource availability
```

### 9.3 形式化理解
上述贪心流程可以理解为：
- 在候选边集合上按局部收益排序
- 依次选择当前最优可行边
- 每接受一条边，就更新剩余供给、剩余缺口和资源容量

它是对 formal 优化模型的一种工程化近似，而不是与 formal 模型无关的另一套逻辑。

### 9.4 输出停止条件
- 所有缺口已被覆盖
- 没有可行候选任务
- 已达到本次规划窗口最大任务数

## 10. 输出结构
每次运行输出：
- 一个 `dispatch_plan`
- 多个 `transfer_task`
- 未满足缺口清单
- 关键约束说明

`transfer_task` 建议包含：
- `task_id`
- `plan_id`
- `from_location_id`
- `to_location_id`
- `pallet_type`
- `planned_qty`
- `planned_departure_time`
- `planned_arrival_time`
- `assigned_vehicle_id`
- `lane_id`
- `priority_score`
- `reason_code`
- `status = pending_approval`

## 11. 推荐解释规则
MVP 每条推荐至少给出 2~3 条解释：
- 为什么目标仓需要补货
- 为什么选这个来源仓
- 为什么选这条路线或车辆

推荐原因码建议：
- `forecast_shortage_high`
- `available_supply_sufficient`
- `nearby_low_cost_lane`
- `sla_urgent`
- `source_risk_controlled`

## 12. 例外处理
### 12.1 无法满足缺口
输出未满足原因：
- 无可调库存
- 路线不可达
- 时间窗不满足
- 车辆容量不足

### 12.2 数据异常
若库存快照过旧、预测结果缺失或路线成本缺失：
- 标记方案可靠性低
- 可降级只输出风险预警，不输出正式推荐

### 12.3 对应的 hard / soft constraint 理解
在 implementation 层，建议也按以下思路处理：
- hard constraints：直接过滤候选任务
- soft constraints：进入打分扣罚，不直接过滤

示例：
- lane 不存在 → hard filter
- ETA 明显晚于硬性截止 → hard filter
- 来源仓保护线接近阈值 → soft penalty
- 单位成本偏高 → soft penalty

这样 implementation 才能和 formal 模型保持一致。

## 13. MVP 边界控制
当前明确不做：
- 全局最优求解器
- 多车型联合装载最优解
- 复杂多站点路径规划
- 实时秒级重算

当前要做：
- 规则过滤
- 启发式打分
- 可解释推荐
- 人工审批闭环

### 13.1 与后续 min-cost flow / MILP 的关系
当前 implementation 并不是和 formal 模型割裂的临时方案，而是其近似版本：
- 候选任务 = 图上的候选边
- `movable_qty` / `forecast_gap` = 供需容量
- 打分 = 边费用与收益的近似表达
- 贪心选择 = 简化求解策略

后续若客户确认数据质量和范围，可以逐步升级到：
1. min-cost flow
2. LP / MILP
3. 多时间窗滚动优化

### 13.2 多时间窗演进的 implementation 含义
当后续引入多时间窗时，implementation 层至少需要补三件事：
1. 按窗口重建 `expected_supply`
2. 记录未满足缺口如何滚入下一窗口
3. 按 ETA 把调拨量归入正确到达窗口

否则“多时间窗优化”会退化成只是在多个单窗上重复跑同一套逻辑。

## 14. 与页面和审批的衔接
调拨推荐页读取 `dispatch_plan` 与 `transfer_task`。
审批页允许：
- 全量通过
- 改量通过
- 拒绝
- 填写备注

执行反馈页回写：
- 实际出发/到达时间
- 实际数量
- 异常说明

## 15. 待客户确认
- `safety_buffer` 的业务口径如何定义。
- 缺口优先级与运输成本哪个权重更高。
- 是否允许拆单、多来源补同一目标仓。
- 是否存在固定合同路线和锁定运力。
- MVP 是否只支持仓到仓调拨，还是也包含 depot 到仓。