# 系统设计与实战题

## 通用评价方式

不要给候选人所有数据。先给场景，观察其主动澄清什么；随后按需提供 workload card。评分顺序：需求澄清 15%、容量与瓶颈 20%、架构 25%、可靠性与发布 15%、观测和验证 15%、成本与取舍 10%。

## S01 · 多租户企业推理平台

### 给候选人的题面

设计一个兼容 OpenAI API 的内部 LLM 推理平台，承载 20 个业务团队。主模型为 70B，另有 8B 和 embedding 模型；业务包含聊天、RAG 和 agent。目标是三个月上线第一版，一年内扩展到峰值 100 RPS。

请覆盖 API/鉴权、模型与 engine、GPU 拓扑、路由与调度、KV 管理、扩缩容、可观测性、发布回滚、隔离和成本归属。画出关键数据流，并给出第一版与演进版边界。

### 按需提供的数据

- 70B：输入 P50/P95/P99 为 1K/8K/32K；输出为 200/800/2K。
- 峰值到达具有 5 倍突发；80% 流式；20% 请求会在生成途中取消。
- 聊天共享 1K system prompt；RAG 内容按租户隔离。
- SLO：P95 TTFT < 800 ms，P95 TPOT < 40 ms，可用性 99.9%。
- 单区域两组 GPU 机型，跨机互联明显慢于节点内 NVLink。

### 专家信号

- 明确 SLO 是否在峰值、哪些长度 bucket 生效，不假装所有请求都能满足同一阈值。
- 用 token/KV 容量建模，分离在线与 batch，设计 admission/backpressure/cancel。
- 将 TP replica 放进拓扑与 gang scheduling；讨论 warm pool 和模型加载。
- prefix cache 有租户边界；cache-aware routing 不牺牲队列公平。
- MVP 不过度设计，但接口、版本和 telemetry 能支撑后续 disaggregation。

### 注入故障追问

上线后 P99 TTFT 在每日整点升到 12 秒，但 TPOT 正常。要求候选人提出前三个假设、查询/图表和缓解动作。强回答会优先怀疑到达突发、长 prefill、冷实例/队列，而不是泛泛“加 GPU”。

## S02 · 长上下文推理平台改造

### 给候选人的题面

现有 vLLM 集群服务 8K 对话稳定。产品要上线 128K 文档分析，直接把 `max-model-len` 改为 128K 后并发骤降、OOM 与 P99 超时频发。请设计改造和验证方案。

### 期待覆盖

- 用 KV 公式估算单请求 HBM，并解释预留长上下文为何损害并发。
- chunked prefill、admission、长度分池、KV 量化/offload、prefix reuse 的适用边界。
- 是否使用 prefill/decode disaggregation，以及 KV transfer 是否吞掉收益。
- realistic distribution 压测；按长度 bucket 定 SLO/goodput；OOM/抢占/cancel 指标。
- 产品层降级：检索/分块、最大上下文、异步 batch，不把基础设施当作唯一答案。

## S03 · 现场性能诊断

### 题面

给候选人以下发布前后数据，请其在 30 分钟内写出诊断树和最小验证实验：

| 指标 | 发布前 | 发布后 |
|---|---:|---:|
| P95 TTFT | 620 ms | 650 ms |
| P95 TPOT | 28 ms | 47 ms |
| 请求吞吐 | 18 RPS | 18 RPS |
| 平均 running sequences | 42 | 19 |
| KV cache occupancy | 72% | 46% |
| HBM bandwidth utilization | 81% | 49% |
| GPU compute utilization | 74% | 51% |
| 错误率 | 0.2% | 0.3% |

发布包含 engine 升级和量化格式变化。候选人应先提出回滚/分离变量的 A/B，检查 kernel 是否 fallback、batch/admission 配置是否漂移、CUDA graph 和量化 kernel 是否命中，而不是把轻微错误率变化当根因。

## S04 · Take-home：容量模型与调度器

建议用时 3 小时，不要求接入真实 GPU。

实现一个离散事件模拟器：请求包含 arrival time、input tokens、max output、tenant 和 priority；系统有 prefill/decode token budget 与有限 KV blocks。实现准入、continuous batching、取消和一种公平策略，输出 TTFT、TPOT 近似值、P95/P99、goodput、KV occupancy 和 tenant fairness。

**评审重点：** 模型假设写清楚、核心逻辑可测试、极端情况、指标口径、权衡说明。不要按代码行数评分；允许 Python/Go/Rust 等任意语言，不要求使用特定框架。

**面谈追问：** 模拟器与真实 engine 最大的偏差是什么？如果加入 chunked prefill、prefix cache 或 TP collective，状态和事件应如何变化？
