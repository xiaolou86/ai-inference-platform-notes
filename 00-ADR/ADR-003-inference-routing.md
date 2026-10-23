# 

# ADR-003: 大模型推理智能路由

## Status

Accepted

## Context

一期需要建设完整的大模型推理平台，而不仅仅是 GPU 调度平台。

平台预计部署约 48 个 GPU 节点，并可能部署多个推理实例 / 推理组。

因此请求不能简单地通过 Round Robin 等传统负载均衡方式转发。

尤其在大模型推理场景中，请求路由需要考虑：

- Model；
- Instance Health；
- Instance Load；
- KV Cache；
- Prefill / Decode 拓扑；
- 多个推理实例之间的资源状态。

平台对外提供 OpenAI-compatible API，因此需要在 API 请求进入推理实例之前完成模型和实例选择。

## Decision

平台采用独立的 Intelligent Router 负责推理请求路由。

整体数据路径：

```
OpenAI Client
 │
 ▼
API Gateway / Platform Entry
 │
 ▼
Intelligent Router
 │
 ├── Model Routing
 ├── Health Awareness
 ├── Load Awareness
 ├── KV Cache Awareness
 └── P/D Routing
 │
 ▼
Inference Instance / Inference Group
 │
 ▼
vLLM / MindIE
```

Router 不负责 GPU 资源调度。

GPU 的申请、释放、部署和故障恢复由计算调度 / 推理部署层负责。

Router 的职责是：

> **根据请求和推理实例当前状态，选择最合适的推理实例或推理组。**

## Routing Strategy

一期至少需要支持以下路由能力：

### 1. Model Routing

根据请求中的：

```
model
```

选择对应模型的推理实例。

### 2. Health-aware Routing

异常实例需要从可用路由集合中移除。

### 3. Load-aware Routing

在多个可用实例之间，根据当前负载进行选择。

### 4. KV Cache-aware Routing

对于存在 Prefix Cache / KV Cache 的推理实例，尽可能将相关请求路由到具有缓存收益的实例。

如果不存在有效 Cache，则允许回退到普通实例选择。

### 5. P/D-aware Routing

当推理采用 Prefill / Decode 分离架构时，Router 需要理解推理组的 P/D 拓扑，而不能将 Prefill 和 Decode 实例简单视为完全独立的普通后端。

## Why

传统负载均衡主要解决：

```
Request → Backend
```

而大模型推理需要解决：

```
Request   ↓Model   ↓Inference Group   ↓P/D topology   ↓Instance   ↓KV Cache
```

请求选择结果会直接影响 TTFT、吞吐以及 KV Cache 利用率。

因此一期的平台路由能力不能仅依赖简单的 Round Robin。

## Alternatives

### Alternative 1：传统 Load Balancer

例如：

```
Round RobinLeast ConnectionRandom
```

优点：

- 简单；
- 稳定；
- 实现成本低。

缺点：

- 不理解模型；
- 不理解 KV Cache；
- 不理解 P/D；
- 无法针对大模型推理进行优化。

仅作为基础兜底策略。

### Alternative 2：完全自研 Router

优点：

- 可以完全控制路由策略；
- 可以针对公司实际业务进行定制。

缺点：

- 开发成本高；
- 需要自行解决实例发现、健康检查、缓存状态、扩缩容等问题；
- 一期交付风险较高。

### Alternative 3：基于现有开源推理路由能力进行集成

例如基于 llm-d 等组件构建智能路由能力。

优点：

- 可以复用已有的模型路由、实例选择和 KV Cache-aware 能力；
- 降低一期实现成本；
- 后续仍可以在平台层增加自己的策略。

缺点：

- 对底层组件存在依赖；
- 不同硬件和推理 Runtime 的适配需要验证；
- 需要明确平台与 Router 的职责边界。

一期优先采用该思路进行验证和集成。

## Trade-offs

平台不重复实现所有底层路由算法，而是建立：

```
平台 API / 推理管理    
    │        
    ▼
Intelligent Router    
    │        
    ▼
Inference Runtime
```

平台负责业务和生命周期管理，Router 负责请求级实例选择。

这样可以降低一期实现风险，同时保留后续自研路由策略的空间。

## Consequences

后续平台需要能够向 Router 提供或同步必要的：

- Model 信息；
- Instance 信息；
- Health 状态；
- Load 状态；
- KV Cache 状态；
- P/D 拓扑信息。

Router 与推理部署平台之间需要建立明确的状态同步机制。
