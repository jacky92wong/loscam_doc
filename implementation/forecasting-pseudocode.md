# 需求预测实施伪代码

## 1. 说明
本文件把当前发布版中的需求预测方向，收敛成 MVP 可执行实施稿。当前强调可解释、可落地、可评估，不直接进入复杂深度学习。

## 2. MVP 预测目标
在日级或周级时间窗内，预测各区域 / 仓库 / 关键节点未来托盘需求，为缺口预警和调拨推荐提供输入。

优先回答：
- 某仓未来 1~7 天是否会缺托盘。
- 缺口大概有多大。
- 预测不确定性区间有多宽。

## 3. 预测对象
### 3.1 粒度建议
MVP 优先使用：
- `region_code + location_id + pallet_type + day`

若数据量不足，可先退化为：
- `region_code + pallet_type + day`

### 3.2 预测标签
标签定义：
- `predicted_demand` = 未来某个时间窗内的托盘需求量

训练标签来源：
- `order_demand.demand_qty`
- 按窗口聚合后的订单需求事实

## 4. 数据输入
### 4.1 核心输入表
- `order_demand`
- `inventory_snapshot`
- `warehouse`
- `customer`
- `forecast_result`（用于回测对比）

### 4.2 扩展输入
- 节假日表
- 促销事件表
- 天气表
- 大客户专项计划表

MVP 若拿不到扩展输入，可先只用历史订单与时间特征。

## 5. 特征设计
### 5.1 时间特征
- 星期几
- 是否月初 / 月末
- 是否节假日前后
- 周序号
- 是否工作日

### 5.2 历史需求特征
- 近 1/3/7/14 天移动平均
- 近 1/3/7 天需求和
- 同比上周同日需求
- 最近一次非零需求距离今天多少天
- 历史峰值 / 波动率

### 5.3 节点特征
- 区域
- 仓库类型
- 客户结构标签
- 服务等级

### 5.4 库存相关辅助特征
- 当前 `available_qty`
- 当前 `available_soon_qty`
- 当前 `in_transit_qty`

说明：库存相关特征不是预测需求的因变量，但可帮助识别历史数据中的异常波动和调度干预影响。

## 6. Baseline 选择
MVP 至少同时保留两类 baseline：

### 6.1 规则基线
- 上周同日值
- 近 7 天均值

### 6.2 机器学习基线
- LightGBM 或 XGBoost 回归

选择理由：
- 可解释性较强
- 对表格数据友好
- 训练和部署成本低
- 便于做特征重要性分析

## 7. 训练流程

### 7.1 数据准备伪代码
```text
load order_demand
filter valid status in [confirmed, allocated, shipped, completed]
aggregate demand by region_code, location_id, pallet_type, day
join calendar features
join warehouse/customer static features
join latest available inventory features if needed
fill missing days with zero demand
flag abnormal samples
output forecast_training_dataset
```

### 7.2 训练伪代码
```text
input: forecast_training_dataset
split by time into train/validation/test
build baseline_1 using last_week_same_day
build baseline_2 using rolling_mean_7d
train model_lgbm on engineered features
predict on validation/test
calculate metrics: MAE, MAPE, WAPE, shortage_recall
compare model_lgbm with baselines
select best model version
save model version and evaluation summary
```

### 7.3 推荐增加的数据校验步骤
```text
check duplicate keys by location_id, pallet_type, day
check missing dates ratio
check abnormal zero streaks
check invalid future leakage fields
check sample count by location and pallet_type
if data quality below threshold:
    downgrade training scope or stop model release
```

## 8. 推理流程
```text
input: latest order history, latest inventory_snapshot, calendar features
for each target location and pallet_type:
    build future time windows
    generate time features
    derive rolling history features from latest known demand
    call selected model to predict demand
    compute lower_bound and upper_bound using residual distribution or quantile rule
    derive reliability_level based on sample sufficiency, residual stability, feature completeness
    write results to forecast_result
return forecast_result set
```

### 8.1 reliability_level 建议规则
```text
if sample_count >= threshold_high
   and feature_missing_ratio <= low
   and residual_wape <= target:
    reliability_level = high
elif sample_count >= threshold_low:
    reliability_level = medium
else:
    reliability_level = low
```

## 9. 缺口预警生成
预测本身不直接等于预警，需要结合库存状态：

```text
for each location and pallet_type:
    expected_supply = available_qty + releasable_available_soon_qty + confirmed_inbound_qty - reserved_qty
    forecast_gap = predicted_demand - expected_supply
    if forecast_gap > threshold:
        mark shortage alert
```

预警分级建议：
- `low`：预测缺口 > 0
- `medium`：预测缺口超过安全阈值
- `high`：预测缺口超过可接受服务水平边界

## 10. 评估方式
### 10.1 模型指标
- MAE
- WAPE
- MAPE（仅在需求非零场景下参考）
- Bias（系统性高估/低估）

### 10.2 业务指标
- 高缺口仓识别召回率
- 紧急调拨触发率是否下降
- 预测支持下的订单满足率是否提升

### 10.3 回测建议
- 采用滚动时间窗回测，而不是随机切分。
- 至少回测最近 8~12 周。
- 单独观察峰值周、节假日前后和大客户异常波动。

### 10.4 推荐增加的 formal 指标口径
```text
MAE  = mean(abs(y_true - y_pred))
WAPE = sum(abs(y_true - y_pred)) / sum(abs(y_true))
Bias = sum(y_pred - y_true) / sum(abs(y_true))
```

缺口识别建议补充：
```text
gap_true = true_demand - true_expected_supply
gap_pred = predicted_demand - predicted_expected_supply
```

然后评估：
- shortage_recall
- shortage_precision
- high_risk_shortage_f1

## 11. MVP 实施边界
当前不做：
- 深度学习时序模型
- 复杂多任务联合预测
- 全实时在线学习
- 自动闭环调参

当前要做：
- 日级 / 周级需求预测
- 与规则基线并行对比
- 输出可解释的结果与区间
- 支持人工复盘预测偏差

## 12. 输出结构
预测任务每次运行至少输出：
- `run_id`
- `model_version`
- `time_granularity`
- `window_start_time`
- `window_end_time`
- `predicted_demand`
- `lower_bound`
- `upper_bound`
- `baseline_value`
- `reliability_level`
- `generated_at`

## 13. 失败与降级策略
- 若模型训练失败，降级使用“近 7 天均值”或“上周同日”。
- 若某节点历史数据不足，降级到区域级预测再按占比分摊。
- 若数据缺失严重，直接标记“不可靠预测”，避免误导调度。

## 14. 与调拨模块的接口
调拨模块读取：
- `forecast_result.predicted_demand`
- `forecast_result.lower_bound`
- `forecast_result.upper_bound`
- `forecast_result.model_version`

调拨模块不应直接依赖训练细节，只依赖标准预测输出表。

## 15. 待客户确认
- 预测粒度是按天、按周，还是二者都要。
- 是否存在显著节假日 / 促销 / 大客户计划影响，需要作为特征接入。
- MVP 试点区域的历史订单数据是否至少覆盖 6~12 个月。
- 预测结果是否需要人工修订入口。
- 验收时优先看技术指标还是业务缺口识别效果。