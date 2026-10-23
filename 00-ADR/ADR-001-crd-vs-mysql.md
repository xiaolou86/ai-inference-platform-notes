# ADR-001：资源状态采用 Kubernetes CRD 作为事实来源

## 背景：

平台需要管理 ResourcePool、ComputeSpec、InferenceWorkload 等资源。
考虑到平台运行在 Kubernetes 上，需要决定资源状态由谁作为 Source of Truth。

## 决策：

Kubernetes CRD 作为资源状态的唯一事实来源。
MySQL 不完整复制 CRD 状态，仅保存查询投影、操作记录等平台数据。

## 原因：

1. Kubernetes 本身具备 Watch / Reconcile / Status / ResourceVersion 等机制
2. 避免 MySQL 与 CRD 双写导致数据不一致
3. Controller 可以直接基于 CRD 进行状态收敛

## 备选方案：

A. CRD + MySQL 双写
B. MySQL 作为事实来源，CRD 作为运行时对象
C. CRD 作为事实来源，MySQL 作为查询投影

## 最终选择：

C

代价：
复杂查询不能完全依赖 CRD，需要维护 MySQL Projection。
