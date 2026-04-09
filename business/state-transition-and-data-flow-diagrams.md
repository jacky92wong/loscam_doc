# 状态流转图与数据流图

## 1. 说明
本文件用于补充 LC 当前方案中最关键的两类图：

1. 状态流转图
2. 数据流图

它们的作用分别是：
- **状态流转图**：检查状态建模是否闭环、状态语义是否一致、事件触发是否合理
- **数据流图**：检查数据从源系统进入算法、再进入结果输出和页面展示的路径是否闭环

如果这两张图讲不清楚，后续开发很容易出现：
- 状态定义和业务动作脱节
- 数据表、接口和算法对象之间对不上
- 页面很难解释系统为什么这样判断

---

# Part 1. 托盘状态流转图

## 2. 核心状态流转总图

```text
                         ┌────────────────────┐
                         │      available     │
                         │ 当前可直接使用      │
                         └─────────┬──────────┘
                                   │
                  订单锁定 / 计划分配 │
                                   ▼
                         ┌────────────────────┐
                         │      reserved      │
                         │ 已被未来需求占用    │
                         └─────────┬──────────┘
                                   │
                         发运 / 出库 / 调拨执行
                                   ▼
                         ┌────────────────────┐
                         │     in_transit     │
                         │      运输途中       │
                         └─────────┬──────────┘
                                   │
                              到达 / 签收
                                   ▼
                         ┌────────────────────┐
                         │       in_use        │
                         │ 客户 / 节点使用中    │
                         └─────────┬──────────┘
                                   │
                        回收 / 归还 / 状态修正事件
                                   ▼
                         ┌────────────────────┐
                         │   available_soon    │
                         │ 预计短期内恢复可用   │
                         └─────────┬──────────┘
                                   │
                  检验通过 / 清洗完成 / 维修完成 / 可用确认
                                   ▼
                         ┌────────────────────┐
                         │      available     │
                         │ 回到可直接使用状态   │
                         └────────────────────┘
```

---

## 3. 异常状态分支图

```text
                  ┌────────────────────┐
                  │       in_use        │
                  └─────────┬──────────┘
                            │
                   长时间未回流 / 位置异常
                            ▼
                  ┌────────────────────┐
                  │   suspected_lost    │
                  │  疑似丢失 / 高风险   │
                  └───────┬───────┬─────┘
                          │       │
                人工确认恢复 │       │ 人工确认丢失
                          │       ▼
                          │  ┌────────────────┐
                          │  │      lost      │
                          │  │   已确认丢失    │
                          │  └────────────────┘
                          │
                          ▼
                  ┌────────────────────┐
                  │   available_soon    │
                  │ 或直接 available     │
                  └────────────────────┘
```

---

## 4. 维修 / 清洗 / 待检分支图

```text
           ┌────────────────────┐
           │   available_soon    │
           │ 回收后待恢复可用     │
           └─────────┬──────────┘
                     │
            需维修 / 待检 / 清洗
                     ▼
           ┌────────────────────┐
           │    maintenance     │
           │   维修/清洗/待检中  │
           └─────────┬──────────┘
                     │
              维修完成 / 检验通过
                     ▼
           ┌────────────────────┐
           │      available     │
           └────────────────────┘
```

---

## 5. 状态流转背后的事件触发建议

### 5.1 `available -> reserved`
触发来源：
- 订单确认
- 调拨预留
- 人工锁定库存

### 5.2 `reserved -> in_transit`
触发来源：
- 发运执行
- 调拨任务出发

### 5.3 `in_transit -> in_use`
触发来源：
- 到货签收
- 节点接收确认

### 5.4 `in_use -> available_soon`
触发来源：
- 回收事件
- 归还事件
- IoT 到站 / 位置修正
- 人工确认已回流

### 5.5 `available_soon -> available`
触发来源：
- 检验通过
- 清洗完成
- 维修完成
- 业务确认可重新投入使用

### 5.6 `in_use -> suspected_lost`
触发来源：
- 长时间未回流
- 长时间无事件更新
- IoT / 业务侧出现高风险异常

### 5.7 `suspected_lost -> lost`
触发来源：
- 人工确认丢失
- 业务闭环确认无法追回

---

## 6. 状态建模检查点
你可以用这张图检查当前模型有没有缺：

### 6.1 是否每个状态都有清晰定义？
### 6.2 是否每个关键状态都有触发事件？
### 6.3 是否区分了“预计可用”和“立即可用”？
### 6.4 是否区分了“疑似丢失”和“确认丢失”？
### 6.5 是否明确哪些状态不能进入 expected_supply？

如果这 5 个问题答不清楚，状态模型就还不算真正闭环。

---

# Part 2. 数据流图

## 7. 总体数据流图

```text
┌──────────────────────────────────────────────┐
│                 外部 / 源系统层               │
├──────────────────────────────────────────────┤
│ OMS / Order System   -> 订单 / 需求数据        │
│ WMS / Inventory      -> 库存快照 / 状态数据     │
│ TMS / Transport      -> lane / ETA / 车辆数据  │
│ IoT / Tracking       -> 位置 / 回流 / 异常事件  │
│ Excel / Manual       -> 补充计划 / 人工确认数据 │
└──────────────────────────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────┐
│                 数据接入与标准化层             │
├──────────────────────────────────────────────┤
│ orders import                                 │
│ inventory snapshot import                     │
│ pallet status events import                   │
│ 主键统一 / 时间格式统一 / 状态枚举统一           │
└──────────────────────────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────┐
│                 核心业务数据层                 │
├──────────────────────────────────────────────┤
│ order_demand                                  │
│ inventory_snapshot                            │
│ pallet / inventory_event                      │
│ warehouse / depot / customer                  │
│ vehicle / route_lane                          │
└──────────────────────────────────────────────┘
                         │
                         ├──────────────┐
                         │              │
                         ▼              ▼
┌──────────────────────┐   ┌────────────────────────┐
│   预测计算层          │   │   供给 / 缺口分析层      │
├──────────────────────┤   ├────────────────────────┤
│ 特征生成              │   │ expected_supply         │
│ 训练 / 推理            │   │ forecast_gap            │
│ forecast_result       │   │ movable_qty             │
│ reliability_level     │   │ carry_over_gap          │
└──────────┬───────────┘   └──────────┬─────────────┘
           │                          │
           └──────────────┬───────────┘
                          ▼
┌──────────────────────────────────────────────┐
│                 Shortage Alerts 层            │
├──────────────────────────────────────────────┤
│ shortage_analysis_result                      │
│ shortage_alert                                │
│ alert_level / warning_flags                   │
└──────────────────────┬───────────────────────┘
                       ▼
┌──────────────────────────────────────────────┐
│                 调拨推荐与优化层              │
├──────────────────────────────────────────────┤
│ source_capacity_result                        │
│ candidate generation                          │
│ hard constraint filtering                     │
│ heuristic scoring / optimization              │
│ dispatch_plan / transfer_task                 │
└──────────────────────┬───────────────────────┘
                       ▼
┌──────────────────────────────────────────────┐
│                 应用与审批层                  │
├──────────────────────────────────────────────┤
│ shortage dashboard                            │
│ dispatch recommendations                      │
│ approval actions                              │
│ alert_only / degraded mode                    │
└──────────────────────┬───────────────────────┘
                       ▼
┌──────────────────────────────────────────────┐
│                 执行反馈与复盘层              │
├──────────────────────────────────────────────┤
│ execution feedback                            │
│ actual departure / arrival / qty              │
│ KPI / deviation / replay                      │
└──────────────────────────────────────────────┘
```

---

## 8. 预测数据流图

```text
order_demand
   +
calendar / holiday / external features
   +
warehouse / customer static features
   +
inventory related auxiliary features
   │
   ▼
forecast_training_dataset
   │
   ▼
forecast model / baseline
   │
   ▼
forecast_result
   │
   ├── predicted_demand
   ├── lower_bound / upper_bound
   ├── baseline_value
   └── reliability_level
```

---

## 9. 供给与缺口分析数据流图

```text
inventory_snapshot
   +
state events / IoT corrections
   +
soon available mapping
   +
confirmed inbound mapping
   │
   ▼
supply_analysis_result
   │
   ├── on_hand_available_qty
   ├── releasable_available_soon_qty
   ├── confirmed_inbound_qty
   ├── inbound_transfer_qty
   ├── reserved_qty_impact
   ├── blocked_qty
   └── expected_supply

forecast_result
   +
carry_over_gap
   +
confirmed demand
   │
   ▼
shortage_analysis_result
   │
   ├── demand_requirement
   ├── expected_supply
   └── forecast_gap
```

---

## 10. 调拨推荐数据流图

```text
shortage_analysis_result
   +
source_capacity_result
   +
route_lane
   +
vehicle
   │
   ▼
candidate tasks
   │
   ├── hard constraint filtering
   ├── ETA feasibility
   ├── cost estimation
   ├── source protection check
   │
   ▼
scored tasks
   │
   ▼
dispatch_plan + transfer_task
   │
   ├── reason_codes
   ├── risk_flags
   ├── eta_confidence
   └── explanations
```

---

## 11. 审批与反馈数据流图

```text
dispatch_plan + transfer_task
          │
          ▼
人工审批
(approved / adjusted / rejected)
          │
          ▼
approved transfer_task
          │
          ▼
执行反馈
(actual departure / arrival / qty)
          │
          ▼
execution history / KPI / deviation analysis
```

---

## 12. 数据流检查点
你可以用这组图来检查当前方案是否还有空档：

### 12.1 源系统输入是否都能落到统一对象？
### 12.2 中间量是否已经对象化？
### 12.3 shortage alert 是否和 recommendation 分层清楚？
### 12.4 审批和反馈是否形成闭环？
### 12.5 降级模式是否在数据流中有位置？

如果这 5 个问题都能答清楚，说明当前数据流已经基本闭环。

---

## 13. 最后一句总结
这两组图的核心作用是：

> **把“状态怎么变”和“数据怎么流”明确下来，这样你才能判断 LC 当前方案到底是停留在概念层，还是已经具备实施闭环。**
