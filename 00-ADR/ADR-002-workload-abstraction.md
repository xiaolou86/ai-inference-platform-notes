# ADR-002: Workload 抽象与具体工作负载类型

## Status

Accepted

## Context

平台需要同时支持大模型推理、训练、Notebook、算力租赁等不同类型的工作负载。

如果平台直接为每种业务定义完全独立的资源 API，会导致资源生命周期、状态管理、调度接口等逻辑重复。

另一方面，不同工作负载的运行时需求存在明显差异。

例如：

- InferenceWorkload 需要模型、模型版本、Runtime、Replica、GPU 等信息；
- TrainingWorkload 需要训练任务、数据集、分布式拓扑等信息；
- NotebookWorkload 主要关注交互式开发环境；
- 算力租赁 Workload 则主要关注资源规格和租期。

因此需要决定是否建立一个统一的 Workload 抽象。

## Decision

保留 `Workload` 作为平台层面的统一概念，但不强制所有工作负载共享一套完整的 Spec。

具体工作负载采用独立类型表达，例如：

- `InferenceWorkload`
- `TrainingWorkload`
- `NotebookWorkload`

对外 API 可以使用 Workload 作为统一的资源概念，并通过 `type` 或具体资源类型进行路由。

内部 Kubernetes 控制面则优先使用具体的 CRD 表达实际工作负载。

例如：

```text
API
 │
 ├── InferenceWorkload
 ├── TrainingWorkload
 └── NotebookWorkload
          │
          ▼
     Kubernetes CRD
          │
          ▼
      Controller
```

一期重点实现 `InferenceWorkload`，暂不为了抽象而提前实现训练和 Notebook 的完整能力。

## Why

统一 Workload 概念可以保证平台整体资源模型的一致性，同时避免建立一个包含大量无关字段的“万能 Workload”。

推理、训练、Notebook 在调度、生命周期和运行时方面存在明显差异，强行共用完整 Spec 会造成：

1. 大量 Optional 字段；
2. 不同类型之间语义混淆；
3. Controller 逻辑复杂；
4. 后续扩展困难。

因此采用：

> **概念统一，类型具体，Spec 独立。**

## Alternatives

### Alternative 1：所有工作负载使用统一 Workload CRD

例如：

```
kind: Workloadspec:  type: inference  ...
```

优点：

- API 数量少；
- 统一管理。

缺点：

- Spec 容易变成“大杂烩”；
- 不同类型的生命周期和字段差异难以表达；
- Controller 复杂度增加。

不采用。

### Alternative 2：完全取消 Workload 抽象

直接使用：

```
InferenceWorkloadTrainingWorkloadNotebookWorkload
```

优点：

- 类型清晰；
- CRD 结构简单。

缺点：

- 平台层缺少统一的工作负载概念；
- 上层 API 和资源管理容易出现多套模型。

不采用。

### Alternative 3：统一 Workload 概念 + 具体 Workload 类型

采用该方案。

## Trade-offs

优点：

- 保留平台统一资源模型；
- 各类工作负载可以独立演进；
- Controller 职责清晰；
- 一期可以只实现推理能力；
- 为后续训练、Notebook 等能力预留扩展空间。

代价：

- API 和 CRD 类型会增加；
- 需要维护统一 Workload 与具体类型之间的映射关系；
- 后续需要定义统一的状态、生命周期和错误模型。

## Consequences

一期重点围绕：

```
InferenceWorkload
    ↓
Inference Deployment
    ↓
Inference Instance / Pod
    ↓
GPU Resource
```

进行设计。

不因为未来可能支持训练、Notebook，而提前引入过度复杂的统一抽象。
