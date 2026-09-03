# 分布式平台与调度

## P01 · 从 workload 开始做平台设计

**主问题：** 在你选择 GPU、引擎和 Kubernetes 方案前，必须向业务收集哪些信息？

**强信号：** 模型/adapter/精度；input/output token 联合分布而非只有均值；到达过程、burst、并发与流式；prefix/多轮复用；TTFT/TPOT/E2E 分层 SLO；质量、可用性、数据驻留；租户优先级和预算。能解释每项如何改变容量或架构。

## P02 · 多租户推理网关

**主问题：** 设计一个同时服务在线聊天、离线批处理和内部 agent 的网关与准入控制。

**追问：** 如何避免一个 128K 请求拖垮所有租户？429 应该在何时返回？

**强信号：** 按 token cost 而非仅 RPS 限流；分 workload queue/pool；优先级、deadline、配额与加权公平；估算 prefill tokens、max output、KV blocks；有界队列、load shedding 和可解释的 retry-after；租户级指标及成本归属。

## P03 · Kubernetes GPU 扩缩容链路

**主问题：** 从请求峰值出现到新 replica 可服务，画出完整扩容链路。每一段的延迟和失败点是什么？

**强信号：** 应用信号触发 HPA/custom controller → pending/gang → scheduler → node provision → driver/runtime → image → model weights → engine warmup/readiness → traffic。知道 GPU utilization 对 LLM 常常滞后或误导，优先用 queue depth、waiting tokens、deadline miss/goodput。提及配额、容量不足、拓扑、镜像/权重加载和 readiness 抖动。

## P04 · Gang Scheduling 与拓扑

**主问题：** 一个 8-GPU TP replica 为什么需要 gang scheduling？如果只调度到 7 张卡会发生什么？

**追问：** 同节点 8 卡与跨节点 8 卡如何影响决策？

**强信号：** replica 必须原子获得整组资源，避免占住部分 GPU 死锁/浪费；调度应理解 NVLink/NVSwitch、PCIe、NIC/RDMA、rack 和故障域。能谈 bin packing 与 spreading 在性能、利用率和容灾上的冲突。

## P05 · 冷启动预算

**主问题：** 70B 模型从 0 到 ready 用时 8 分钟。请把时间拆开，并给出优化顺序。

**强信号：** 节点启动、镜像、远端权重下载、磁盘到 RAM/HBM、反序列化、engine compile/profile、collective 初始化、warmup；先测 critical path。手段包括预置镜像/权重、本地 NVMe/并行流式加载、warm pool、内存/GPU snapshot、共享缓存；会计算 warm pool 成本与峰值损失的平衡。

## P06 · Prefill/Decode 解耦

**主问题：** 为什么 prefill 和 decode 可以使用不同 GPU pool？什么时候解耦值得，什么时候不值得？

**追问：** KV 如何传输？pool ratio 如何动态调整？

**强信号：** compute-bound 与 bandwidth-bound 资源画像不同，可独立批处理和扩缩容；代价是 KV 传输、额外排队、路由与故障复杂度。短 prompt/短输出或低负载通常不值。会以到达率、prompt/output 分布、两阶段 service time 建队列模型，并观察 transfer latency 与 pool imbalance。

## P07 · KV Cache 感知路由

**主问题：** round-robin 为什么会破坏 prefix cache？设计一个同时考虑缓存、本机负载和租户隔离的路由评分。

**强信号：** `score = expected_saved_prefill - queue_delay - transfer_cost - policy_penalty` 一类思路；缓存目录需及时但不必强一致；考虑热点、cache stampede、失败转移、模型/adapter 版本和 tenant namespace。知道“命中最多”不一定让 P99 最小。

## P08 · 多区域与数据驻留

**主问题：** 为中国、欧洲和美国用户设计多区域推理。会话 KV 有 locality，但区域还可能故障，如何路由和恢复？

**强信号：** 先满足数据驻留/模型授权，再做 latency；sticky/cache-aware routing；区域内 replica 故障可重算或从分层缓存恢复，跨区复制 KV 的带宽与隐私成本通常很高；模型、tokenizer、配置、adapter 与安全策略版本要可恢复；明确 RTO/RPO 和降级模式。

## P09 · 多 LoRA/模型版本平台

**主问题：** 数百租户各有 LoRA，如何避免每个租户一套完整模型？

**强信号：** 共享 base weights，按需加载/缓存 adapter；请求按 base+adapter 路由/批处理；限制 adapter rank/大小和并发驻留；版本化、热加载、eviction、安全隔离和质量回滚。能指出某些量化/kernel 对 multi-LoRA 支持有限，需实测。

## P10 · 模型发布与回滚

**主问题：** 新模型或新 engine 版本如何从 shadow 走到 100%？哪些指标触发自动回滚？

**强信号：** offline quality/security → replay → shadow（不影响用户）→ 小比例 canary → 分阶段；按同一请求比较语义质量、结构化输出、错误、TTFT/TPOT、tokens/request 和成本；非确定性下用足够样本和置信区间。回滚包含模型、tokenizer、chat template、adapter、量化与 engine 配置的原子版本。

## P11 · Backpressure 的传播

**主问题：** GPU worker 已饱和，backpressure 应怎样从 worker 传播到 gateway 和 client？

**强信号：** worker 暴露有界 token/KV budget 与 queue estimate；router 停止准入/换池；gateway 限流、排队或快速失败；流式连接与取消信号能释放调度槽/KV；重试带 jitter 且有全局预算，避免 retry storm。

## P12 · Build vs Buy

**主问题：** 何时自建推理平台，何时使用托管服务？请给出可审计的决策模型。

**强信号：** 流量稳定性/规模、模型定制、数据合规、SLO、GPU 利用率、工程人力、机会成本和供应商锁定；比较 cost per successful/quality-adjusted request，而非标价/token；设计可逆的分阶段决策和退出方案。

## 素材来源

- [GPU Autoscaling on Kubernetes](https://github.com/rohitg00/ai-engineering-from-scratch/blob/a56b4b8a/phases/17-infrastructure-and-production/03-gpu-autoscaling-kubernetes/docs/en.md)
- [Multi-Region Serving and KV Locality](https://github.com/rohitg00/ai-engineering-from-scratch/blob/a56b4b8a/phases/17-infrastructure-and-production/11-multi-region-kv-locality/docs/en.md)
- [Disaggregated Prefill/Decode](https://github.com/rohitg00/ai-engineering-from-scratch/blob/a56b4b8a/phases/17-infrastructure-and-production/17-disaggregated-prefill-decode/docs/en.md)
- [Shadow, Canary and Progressive Deployment](https://github.com/rohitg00/ai-engineering-from-scratch/blob/a56b4b8a/phases/17-infrastructure-and-production/20-shadow-canary-progressive/docs/en.md)
- [Inference Platform Economics](https://github.com/rohitg00/ai-engineering-from-scratch/blob/a56b4b8a/phases/17-infrastructure-and-production/02-inference-platform-economics/docs/en.md)
