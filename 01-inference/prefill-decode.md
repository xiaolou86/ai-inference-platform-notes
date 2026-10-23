|      | Prefill             | Decode                      |
| ---- | ------------------- | --------------------------- |
| 输入   | 大量 Token            | 通常 1 Token                  |
| 特征   | 计算密集                | 访存密集                        |
| 并行度  | 高                   | 相对低                         |
| 主要瓶颈 | GPU Compute         | Memory Bandwidth / KV Cache |
| 关注指标 | TTFT                | TPOT                        |
| 适合优化 | Batch / GPU Compute | KV Cache / Batching         |



### Prefill 和 Decode 为什么计算特征不同？
