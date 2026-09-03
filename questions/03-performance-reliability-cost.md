# 性能、可靠性与成本

## O01 · 定义一个可执行的 SLO

**主问题：** 为流式聊天接口定义 SLI/SLO。为什么只有“P99 latency < 5s”不够？

**强信号：** 分开 queue、TTFT、TPOT/ITL、E2E；用 input/output bucket 和流量类别分层；定义成功/取消/限流口径。goodput 是同时满足多个阈值的有效请求率。SLO 还要包含可用性、质量/安全底线和 error budget。

## O02 · 平均值掩盖了什么？

**主问题：** 平均 TTFT 300 ms，P99 为 8 s。列出可能原因以及你需要的维度。

**强信号：** 长 prompt head-of-line blocking、冷启动、queue burst、GC/CPU tokenizer、cache miss、某 GPU/节点降频或错误、collective straggler、adapter load。按 prompt/output bucket、model、tenant、instance、cache hit、batch size、queue time 和版本切分；用 trace 串 gateway→scheduler→worker。

## O03 · 设计不会“撒谎”的压测

**主问题：** 如何复现线上 LLM 流量？为什么 500 个相同 prompt 的并发测试可能非常乐观？

**强信号：** 使用 input/output 的联合分布、到达间隔、内容多样性、prefix 命中、流式和取消率；相同 prompt 会造成不真实的 prefix cache 命中。客户端本身需做 CPU/网络容量验证。覆盖 steady/ramp/spike/soak，使用 open-loop 检测过载而不是只用 closed-loop 自节流。

## O04 · 找到容量拐点

**主问题：** 随 RPS 增加，吞吐仍上升但 P99 突然恶化。如何定义安全容量？

**强信号：** 这是 queueing knee；以 goodput 和 SLO miss 而不是最大吞吐定容量；留 burst/故障余量，按 token workload 分桶；画 offered load、achieved throughput、queue、TTFT、TPOT、batch、KV occupancy。能讨论 utilization 与排队延迟的非线性。

## O05 · 一次 TPOT 回归排障

**主问题：** 发布后 TTFT 不变，TPOT 恶化 35%，GPU compute utilization 反而下降。你如何逐层排查？

**强信号：** 先回滚/对照；核对模型精度、kernel 路径、batch、CUDA graph、quant kernel、TP collective、时钟/功耗、HBM 带宽、KV layout/offload、采样 CPU、流式统计口径。用 profiler/trace/engine metrics 缩小范围，不先归因网络。

## O06 · 可观测性数据模型

**主问题：** 推理平台的一条 trace、metrics 和 logs 应分别记录什么？哪些字段不能直接记录？

**强信号：** trace 关联 gateway、route、queue、prefill、decode、tool/downstream；metrics 记录 token bucket、TTFT/TPOT、goodput、KV occupancy、batch、cache、抢占、GPU/互联；logs 留版本和错误。prompt/completion 可能含 PII/机密，需默认不采集或脱敏、加密、RBAC、保留期与租户策略；控制高基数与 sampling bias。

## O07 · 取消请求为什么重要？

**主问题：** 用户关闭流式连接后，系统继续生成会造成什么？如何保证取消传播？

**强信号：** 浪费 decode 带宽、KV 和成本并挤压 goodput；disconnect/cancel 从 gateway 经 router 到 scheduler，安全点移除 sequence 并回收 blocks；处理竞态、幂等和客户端重试。监控 abandoned tokens/cancel latency。

## O08 · Chaos 实验

**主问题：** 给出三个 LLM 推理特有的 chaos 实验及其稳态假设和停止条件。

**强信号：** 例如杀死 TP rank/worker、注入 KV transfer 延迟、损坏/减慢权重下载、制造 cache miss storm、GPU ECC/Xid、长 prompt 洪峰。每项先定义稳态 SLO、blast radius、自动 abort、回滚与证据；不会直接在全量生产“看看会怎样”。

## O09 · 单位经济性

**主问题：** 你会用什么单位成本衡量平台？为什么 $/GPU-hour 和 $/million tokens 都可能误导？

**强信号：** cost per successful SLO-meeting request/session/task，必要时 quality-adjusted；计入 prompt/cache read/output、空闲、失败/重试、网络/存储、平台人力和折旧。token 单价忽略不同 output 长度、质量和缓存，GPU-hour 忽略利用率与用户价值。

## O10 · 容量估算题

**主问题：** 峰值 20 RPS，平均输入 2K、输出 300 token；单 replica 在目标 SLO 下可承载 3,000 prefill tok/s 和 600 decode tok/s。忽略方差，最低需要多少 replica？生产应部署多少？

**评分锚点：** prefill 需求 40K tok/s → 14；decode 需求 6K tok/s → 10，最低由 prefill 决定为 14。生产还需考虑 P95/P99 分布、burst、N+1/区域故障、效率下降、headroom，不能直接部署 14；应说明选取 headroom 的依据。

## O11 · 降本栈的相互作用

**主问题：** 量化、continuous batching、prefix cache、speculative decoding、路由小模型都能降本。为什么不能把各自百分比简单相加？

**强信号：** 收益有重叠且改变瓶颈；量化可能让 decode 更快从而改变最佳 batch；prefix hit 消除部分 prefill；speculation 占用额外算力且影响 batching；路由改变长度/质量分布。需要逐步实验、交互矩阵和端到端 quality/goodput/cost 指标。

## O12 · 故障复盘

**主问题：** “GPU utilization 100%，扩容后仍大量超时”这份复盘缺了什么？

**强信号：** utilization 不是根因；需要时间线、变更、workload/token 分布、队列、KV、batch、TTFT/TPOT、goodput、冷启动链路、取消/重试、容量假设和控制面行为。改进项必须有 owner、期限、验证门禁和防复发指标。

## 素材来源

- [Inference Metrics and Goodput](https://github.com/rohitg00/ai-engineering-from-scratch/blob/a56b4b8a/phases/17-infrastructure-and-production/08-inference-metrics-goodput/docs/en.md)
- [LLM Observability](https://github.com/rohitg00/ai-engineering-from-scratch/blob/a56b4b8a/phases/17-infrastructure-and-production/13-llm-observability/docs/en.md)
- [Load Testing LLM APIs](https://github.com/rohitg00/ai-engineering-from-scratch/blob/a56b4b8a/phases/17-infrastructure-and-production/22-load-testing-llm-apis/docs/en.md)
- [Chaos Engineering for LLM Production](https://github.com/rohitg00/ai-engineering-from-scratch/blob/a56b4b8a/phases/17-infrastructure-and-production/24-chaos-engineering-llm/docs/en.md)
- [FinOps for LLMs](https://github.com/rohitg00/ai-engineering-from-scratch/blob/a56b4b8a/phases/17-infrastructure-and-production/27-finops-llms/docs/en.md)
