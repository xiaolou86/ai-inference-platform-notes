# ADR-004: Prefill / Decode 分离推理架构

## Status

Accepted

## Context

一期大模型推理平台面向约 48 个 GPU 节点。

在实际 GLM5.x 等大模型部署中，需要考虑 Prefill 和 Decode 阶段的资源特征差异。

Prefill 和 Decode 的计算特征不同：

- Prefill 更偏计算密集；
- Decode 更偏逐 Token 生成；
- 两个阶段对 GPU 资源和调度策略的需求不同。

因此平台需要支持 Prefill / Decode 分离部署（P/D Disaggregation）。

一期预期存在类似：

```
2 × Prefill
+
1 × Decode
```

组成一个推理组（Inference Group），并且整个集群可能存在多个这样的推理组。

## Decision

平台将 P/D 分离作为推理部署和智能路由需要理解的一种拓扑，而不是将 Prefill 和 Decode 简单视为两个完全独立的服务。

一期采用：

```
Inference Group
│
├── Prefill Instance × 2
│
└── Decode Instance × 1
```

多个 Inference Group 可以并存：

```
Inference Group A
├── P × 2
└── D × 1
Inference Group B
├── P × 2
└── D × 1
Inference Group C
├── P × 2
└── D × 1
```

请求进入平台后，由 Intelligent Router 根据模型、实例健康状态、负载以及 KV Cache 等信息选择合适的推理组。

## Architecture

```
                    OpenAI API
                        │                        
                        ▼                 
                    Intelligent Router
                        │             
             ┌──────────┴──────────┐             
             ▼                     ▼      
     Inference Group A      Inference Group B             
             │                     │
        ┌────┴────┐           ┌────┴────┐
        ▼         ▼           ▼         ▼
       P1        P2          P1        P2
        │         │           │         │
        └────┬────┘           └────┬────┘
             ▼                     ▼ 
             D1                    D1
```

具体的 P/D 通信机制和 KV Cache 传输机制，需要结合实际 Runtime（如 vLLM / MindIE）进一步验证。

## Why

P/D 分离可以：

1. 分离 Prefill 和 Decode 的资源需求；
2. 分别优化两类实例的资源配置；
3. 根据实际业务负载调整 P/D 比例；
4. 为后续更细粒度的推理调度和路由提供基础。

同时，将 P/D 组织成 Inference Group，可以避免上层平台直接暴露大量底层 Pod / Instance 细节。

平台关注：

```
Inference Group
```

而不是直接管理：

```
Pod APod BPod C
```

## Alternatives

### Alternative 1：Prefill / Decode 不分离

```
Request  ↓Unified Inference Instance
```

优点：

- 架构简单；
- 部署简单；
- 运维成本低。

缺点：

- 无法针对 P/D 阶段进行独立优化；
- 资源利用率和扩展策略受限。

可以作为非 P/D 部署模式存在，但不是一期目标架构。

### Alternative 2：P/D 完全独立管理

例如平台只管理：

```
Prefill ServiceDecode Service
```

由其他组件负责两者之间的关系。

优点：

- 服务模型简单；
- 底层实现自由度高。

缺点：

- 平台无法直接表达一个完整推理实例组；
- Router 需要额外维护 P/D 关系；
- 生命周期管理复杂。

不作为平台层的主要抽象。

### Alternative 3：Inference Group

```
Inference Group
├── Prefill
└── Decode
```

采用该方案。

## Trade-offs

优点：

- 能表达实际 P/D 部署拓扑；
- Router 可以以 Inference Group 为基本路由单元；
- 支持多个推理组；
- 后续可以调整 P/D 比例；
- 可以进一步结合 KV Cache 状态进行路由。

代价：

- 部署模型更加复杂；
- P/D 之间需要额外通信；
- Router 需要理解 P/D 拓扑；
- KV Cache 传输 / 外部存储机制需要进一步验证；
- Runtime 对 P/D 分离能力存在依赖。

## Consequences

一期平台的推理资源模型需要能够表达：

```
Model
  │  
  ▼
Inference Group
  │
  ├── Prefill Instances
  │  
  └── Decode Instances
```

并且推理部署、健康检查、扩缩容、故障恢复和智能路由都需要考虑 Inference Group 的完整性。

具体的：

- P/D 通信协议；
- KV Cache 共享 / 传输；
- P/D 扩缩容策略；
- Prefill / Decode 最优比例；

不在本 ADR 中固定，由后续 Runtime 性能测试和一期验证结果决定。
