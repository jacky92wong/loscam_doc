# LOSCAM 图示化总览（ASCII 版）

## 1. 托盘池化业务总图

```text
       ┌──────────────┐
       │    客户需求    │
       │ order / plan │
       └──────┬───────┘
              │
              ▼
      ┌───────────────┐
      │ 目标节点需要托盘 │
      │ warehouse/site │
      └──────┬────────┘
             │
             │ 若本地不足
             ▼
   ┌────────────────────────┐
   │ 调度系统判断哪里有可支援库存 │
   └──────┬─────────────────┘
          │
          ├──────────────┐
          │              │
          ▼              ▼
┌────────────────┐  ┌────────────────┐
│ 来源仓 / depot   │  │ 在途 / soon available │
│ on-hand supply │  │ future supply   │
└──────┬─────────┘  └────────┬───────┘
       │                     │
       └──────────┬──────────┘
                  ▼
        ┌──────────────────┐
        │ 生成调拨推荐任务   │
        │ from / to / qty  │
        └────────┬─────────┘
                 │
                 ▼
        ┌──────────────────┐
        │ 人工审批与执行     │
        └────────┬─────────┘
                 │
                 ▼
        ┌──────────────────┐
        │ 到货 / 回写 / 复盘 │
        └──────────────────┘
```

---

## 2. 系统闭环总图

```text
┌────────────┐
│  数据接入层  │
│ orders      │
│ inventory   │
│ lane/vehicle│
│ IoT events  │
└─────┬──────┘
      ▼
┌────────────┐
│ 状态处理层   │
│ clean       │
│ standardize │
│ snapshot    │
│ event merge │
└─────┬──────┘
      ▼
┌────────────┐
│ 预测层       │
│ demand      │
│ risk band   │
└─────┬──────┘
      ▼
┌────────────┐
│ 优化层       │
│ expected    │
│ gap         │
│ dispatch    │
└─────┬──────┘
      ▼
┌────────────┐
│ 应用与审批层  │
│ recommend   │
│ approve     │
│ adjust      │
└─────┬──────┘
      ▼
┌────────────┐
│ 执行反馈层   │
│ actual run  │
│ deviation   │
│ KPI         │
└────────────┘
```

---

## 3. 供给、需求、缺口关系图

```text
需求侧
confirmed_demand
      +
alpha * forecasted_demand
      +
carry_over_gap
      │
      ▼
 demand_requirement

供给侧
on_hand_available
      +
releasable_available_soon
      +
confirmed_inbound
      +
inbound_transfer
      -
reserved_qty
      -
blocked_qty
      │
      ▼
 expected_supply

最终
forecast_gap = max(0, demand_requirement - expected_supply)
```

---

## 4. 多时间窗缺口滚动图

```text
window t
┌──────────────────────────┐
│ demand_requirement_(t)   │
│ expected_supply_(t)      │
│ if not enough -> gap_t   │
└──────────┬───────────────┘
           │ carry over
           ▼
window t+1
┌──────────────────────────┐
│ demand_requirement_(t+1) │
│ + carry_over_gap_(t+1)   │
│ expected_supply_(t+1)    │
└──────────┬───────────────┘
           │ carry over
           ▼
window t+2 ...
```

---

## 5. 调拨推荐算法流程图

```text
开始
  │
  ▼
读取库存、预测、lane、vehicle
  │
  ▼
计算 expected_supply / forecast_gap / movable_qty
  │
  ▼
识别缺口节点
  │
  ▼
生成候选调拨任务
  │
  ▼
做硬约束过滤
  │
  ├─ 不可行 -> 丢弃 / 标记原因
  │
  ▼
对可行任务打分
  │
  ▼
按分数排序 + 贪心选择
  │
  ▼
输出 dispatch plan
  │
  ▼
人工审批
  │
  ▼
执行反馈回写
```

---

## 6. 从 MVP 到后续演进图

```text
Phase 1
规则 + 启发式
(single window / explainable)
      │
      ▼
Phase 2
min-cost flow / LP
(multi-source matching)
      │
      ▼
Phase 3
MILP / rolling horizon
(multi-window / defer / dynamic supply)
      │
      ▼
Phase 4
VRP + execution optimization
(route + vehicle + time windows)
```

---

## 7. 建议优先阅读的图示
建议按以下顺序理解：
1. 托盘池化业务总图
2. 系统闭环总图
3. 供给、需求、缺口关系图
4. 多时间窗缺口滚动图
5. 调拨推荐算法流程图
