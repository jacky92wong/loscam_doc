# LOSCAM 项目资料总大纲

## 1. 使用说明
本目录不是按“已有文件名”来组织，而是按**理解业务 → 理解系统 → 理解数据 → 理解算法 → 理解实现 → 理解风险与边界**的逻辑来组织。

适合三类阅读方式：
- **客户沟通**：先看 Part 1、Part 2、Part 9
- **产品/方案梳理**：先看 Part 1 到 Part 6
- **研发/算法实施**：重点看 Part 3 到 Part 7

---

# Part 1. 项目背景与业务问题定义

## Chapter 1. 项目背景
### 1.1 LOSCAM 业务背景
### 1.2 托盘池化业务的基本模式
### 1.3 当前业务目标
### 1.4 为什么需要 AI 调度系统

## Chapter 2. 业务生态与参与方
### 2.1 客户
### 2.2 仓库 / depot
### 2.3 运营调度
### 2.4 运输承运商
### 2.5 IoT / 外部系统

## Chapter 3. 当前痛点
### 3.1 托盘利用率低
### 3.2 缺货与紧急补货
### 3.3 跨仓调拨成本高
### 3.4 多仓供需不平衡
### 3.5 状态数据不可信
### 3.6 运输执行与库存判断脱节

## Chapter 4. 现状业务流程
### 4.1 订单到需求形成
### 4.2 托盘投放与回收
### 4.3 库存状态变化
### 4.4 调拨审批与执行
### 4.5 异常处理

## Chapter 5. 业务术语表
### 5.1 托盘状态术语
### 5.2 供给与缺口术语
### 5.3 调拨与运输术语
### 5.4 预测与优化术语

## Chapter 6. KPI 与业务价值
### 6.1 利用率
### 6.2 服务水平
### 6.3 调拨成本
### 6.4 紧急补货率
### 6.5 空驶率
### 6.6 预测支持下的运营改进指标

---

# Part 2. 系统目标与总体技术思路

## Chapter 7. 系统目标与闭环
### 7.1 系统要回答的问题
### 7.2 输入
### 7.3 输出
### 7.4 核心闭环
### 7.5 MVP 原则

## Chapter 8. 总体架构
### 8.1 数据接入层
### 8.2 数据处理与特征层
### 8.3 预测服务层
### 8.4 优化求解层
### 8.5 调度应用层
### 8.6 监控与回写层

## Chapter 9. MVP 路线图
### 9.1 Phase 1 范围
### 9.2 Phase 2 范围
### 9.3 Phase 3 范围
### 9.4 明确不做的内容

---

# Part 3. 数据建模

## Chapter 10. 核心实体模型
### 10.1 Warehouse / Depot
### 10.2 Customer
### 10.3 Pallet
### 10.4 Inventory Snapshot
### 10.5 Order Demand
### 10.6 Vehicle
### 10.7 Route / Lane
### 10.8 Forecast Result
### 10.9 Dispatch Plan
### 10.10 Transfer Task

## Chapter 11. 状态模型与状态机
### 11.1 available
### 11.2 reserved
### 11.3 in_use
### 11.4 in_transit
### 11.5 available_soon
### 11.6 maintenance
### 11.7 lost / suspected_lost
### 11.8 状态流转规则

## Chapter 12. 时间与位置维度
### 12.1 事件时间
### 12.2 计划时间
### 12.3 可用时间
### 12.4 实际执行时间
### 12.5 区域 / 仓 / depot / 客户层级

## Chapter 13. 事件与快照模型
### 13.1 inventory_event
### 13.2 iot_tracking_event
### 13.3 transfer_execution_event
### 13.4 forecast_run_log
### 13.5 dispatch_run_log

## Chapter 14. 供给、需求、缺口口径
### 14.1 confirmed_demand
### 14.2 forecasted_demand
### 14.3 demand_requirement
### 14.4 expected_supply
### 14.5 forecast_gap
### 14.6 movable_qty
### 14.7 source_protection_threshold

---

# Part 4. 需求预测模型

## Chapter 15. 预测问题定义
### 15.1 预测目标
### 15.2 为什么先做聚合预测
### 15.3 预测粒度
### 15.4 预测标签定义

## Chapter 16. 预测特征工程
### 16.1 时间特征
### 16.2 历史需求特征
### 16.3 节点特征
### 16.4 库存辅助特征
### 16.5 外部特征

## Chapter 17. 预测模型 formal 定义
### 17.1 符号定义
### 17.2 预测函数
### 17.3 训练目标
### 17.4 分层建模
### 17.5 基线模型
### 17.6 LightGBM / XGBoost 方案

## Chapter 18. 预测训练与推理
### 18.1 样本构造
### 18.2 补零逻辑
### 18.3 异常样本标记
### 18.4 训练流程
### 18.5 推理流程
### 18.6 reliability_level

## Chapter 19. 预测评估
### 19.1 MAE / RMSE / WAPE / Bias
### 19.2 shortage recall / precision / F1
### 19.3 滚动回测
### 19.4 降级策略

---

# Part 5. 优化与调拨模型

## Chapter 20. 优化问题定义
### 20.1 目标
### 20.2 为什么是多目标
### 20.3 供需匹配层
### 20.4 执行约束层

## Chapter 21. 优化模型 formal 定义
### 21.1 集合与索引
### 21.2 参数
### 21.3 决策变量
### 21.4 slack 变量
### 21.5 目标函数
### 21.6 惩罚系数

## Chapter 22. 约束系统
### 22.1 来源供给约束
### 22.2 目标容量约束
### 22.3 lane 可用性约束
### 22.4 车辆容量约束
### 22.5 最小经济发运量
### 22.6 来源保护约束
### 22.7 拆单约束
### 22.8 延期约束

## Chapter 23. 多时间窗优化
### 23.1 库存递推
### 23.2 carry-over shortage
### 23.3 延期满足
### 23.4 ETA 映射到窗口
### 23.5 rolling horizon

## Chapter 24. 关键量总公式
### 24.1 expected_supply
### 24.2 forecast_gap
### 24.3 movable_qty
### 24.4 releasable_available_soon_qty
### 24.5 confirmed_inbound_qty
### 24.6 source_protection_threshold

## Chapter 25. 启发式与演进路径
### 25.1 候选任务生成
### 25.2 hard filter
### 25.3 score 机制
### 25.4 贪心选择
### 25.5 heuristic → min-cost flow
### 25.6 heuristic → MILP

---

# Part 6. 路由与调度执行

## Chapter 26. 运输问题定义
### 26.1 为什么库存推荐必须考虑运输
### 26.2 lane / vehicle 建模
### 26.3 ETA 估算
### 26.4 trip_cost / cost_per_pallet

## Chapter 27. 运输可行性与风险
### 27.1 时间窗可行性
### 27.2 容量可行性
### 27.3 班次可行性
### 27.4 ETA 可信度
### 27.5 风险标签规则

---

# Part 7. 系统实现与接口

## Chapter 28. 逻辑 Schema
### 28.1 主数据层
### 28.2 交易层
### 28.3 状态层
### 28.4 算法层

## Chapter 29. 数据契约
### 29.1 订单输入契约
### 29.2 库存输入契约
### 29.3 预测输出契约
### 29.4 调拨输出契约
### 29.5 执行回写契约

## Chapter 30. API 草案
### 30.1 Forecast API
### 30.2 Dispatch Recommendation API
### 30.3 Approval API
### 30.4 Execution Feedback API

## Chapter 31. 伪代码与实施稿
### 31.1 forecasting pseudocode
### 31.2 dispatch heuristic
### 31.3 数据校验逻辑
### 31.4 降级策略

---

# Part 8. 页面、流程与角色

## Chapter 32. MVP 页面
### 32.1 缺口看板
### 32.2 调拨推荐页
### 32.3 审批页
### 32.4 执行反馈页
### 32.5 KPI 页

## Chapter 33. 流程与角色
### 33.1 调度员
### 33.2 区域经理
### 33.3 仓库 / depot
### 33.4 数据 / 算法运维

---

# Part 9. 风险、边界与待确认事项

## Chapter 34. MVP 边界
### 34.1 当前要做
### 34.2 当前不做
### 34.3 为什么不能一步到位

## Chapter 35. 风险与降级
### 35.1 数据不完整
### 35.2 预测不稳定
### 35.3 运输数据不可信
### 35.4 状态口径不一致
### 35.5 无解与降级策略

## Chapter 36. 待客户确认清单
### 36.1 数据可得性
### 36.2 业务规则
### 36.3 权重优先级
### 36.4 MVP 试点范围
### 36.5 验收标准

---

## 2. 建议阅读顺序
### 2.1 如果你是第一次接触这个行业
建议顺序：
1. Part 1 项目背景与业务问题定义
2. Part 2 系统目标与总体技术思路
3. Part 3 数据建模
4. Part 5 优化与调拨模型
5. Part 6 路由与调度执行
6. Part 7 系统实现与接口

### 2.2 如果你要和客户沟通方案
建议顺序：
1. Chapter 1~6
2. Chapter 7~9
3. Chapter 20~25
4. Chapter 34~36

### 2.3 如果你要推进研发实施
建议顺序：
1. Chapter 10~14
2. Chapter 15~19
3. Chapter 20~31
4. Chapter 32~36
