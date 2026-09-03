# 核心原理与推理引擎

每题均给出主问题、有效追问和评分锚点。“强信号”不是唯一正确答案，而是专家回答通常覆盖的证据。

## E01 · Prefill 与 Decode 为什么要区别对待？

**主问题：** 请从计算形态和硬件瓶颈解释 prefill 与 decode。一个服务 TTFT 达标但 TPOT 很差，你如何定位？

**追问：** batch size 增大为何通常更利于 decode？长 prompt 如何影响其他正在 decode 的请求？

**强信号：** prefill 是大矩阵乘、通常 compute-bound；decode 每步 token 少且反复读取权重，通常 memory-bandwidth-bound。能用 arithmetic intensity/roofline 推导，而不是背结论。定位会拆分排队、tokenization、prefill、每步 decode、网络流式耗时，并检查批大小与 HBM 带宽。

**风险信号：** 把“GPU 利用率高”直接等同于吞吐好；混淆 TTFT 与生成速度。

## E02 · 做一次推理下界估算

**主问题：** 某 70B dense 模型以 BF16 部署，单次 decode 近似要读取全部权重。给定 GPU 聚合 HBM 带宽为 6 TB/s，忽略通信和计算，估算 batch=1 的理论 token latency 下界。为什么实测只会更差？

**评分锚点：** 权重约 140 GB，下界约 `140/6000 ≈ 23 ms/token`；实际受带宽利用率、kernel launch、KV 读写、采样、跨卡通信和同步影响。优秀候选人会指出 tensor parallel 下聚合带宽不可直接无损相加，并明确估算假设。

## E03 · KV Cache 容量与并发

**主问题：** 推导单请求 KV Cache 大小公式。模型为 80 层、8 个 KV heads、head_dim=128、BF16，4K context 大约需要多少 KV Cache？

**追问：** MHA、GQA、MQA 怎样影响容量？128K context、多 beam、prefix sharing、KV 量化又如何影响？

**评分锚点：** `2 × layers × kv_heads × head_dim × seq_len × bytes`；本例每 token 320 KiB，4K 约 1.25 GiB。能区分权重内存、KV、activation 与 allocator 预留，并从剩余 HBM 推导最大并发。

## E04 · KV Cache 用完时该怎么办？

**主问题：** 线上出现 KV Cache exhaustion 和请求抢占，请列出至少四种处理手段，并说明各自牺牲什么。

**强信号：** admission control/排队；降低 max context 或并发；分页分配；prefix reuse；KV FP8/INT8；CPU/NVMe offload；重算/抢占策略；增加实例或切分 prefill/decode。能讨论 PCIe/RDMA 传输、命中率、P99 和质量风险。

## E05 · PagedAttention 解决的到底是什么？

**主问题：** 把 PagedAttention 类比为虚拟内存，并说明 block table、内部碎片、外部碎片、copy-on-write 各自的作用。

**追问：** block 越小越好吗？它和 FlashAttention 是一回事吗？

**强信号：** 明确它主要是 KV 内存管理与 attention 访问布局；小 block 降低尾部浪费但增加元数据/间接寻址开销。FlashAttention 主要减少 attention kernel 的 HBM IO，两者正交且可组合。

## E06 · Continuous Batching 的调度器

**主问题：** 请画出 iteration-level continuous batching 的一次调度循环，说明 waiting/running/swapped 队列何时变化。

**追问：** 如何避免长 prefill 饿死 decode？如何避免短请求或低优先级租户饥饿？

**强信号：** 每个 decode iteration 释放完成序列并准入新工作；以 token/KV block budget 而非请求数调度；chunked prefill 与 decode 交错；考虑优先级、age、deadline、公平配额和抢占恢复成本。

## E07 · Chunked Prefill 的双刃剑

**主问题：** 为什么把 32K prompt 切成 chunk 可以保护 decode 的尾延迟？哪些配置会使 TTFT 或吞吐反而恶化？

**强信号：** 长 prefill 不再独占一个大 iteration，decode 可穿插；chunk 太小会增加调度/kernel 开销并拉长该请求 TTFT，太大仍造成 head-of-line blocking。要基于 prompt 分布和 TTFT/TPOT SLO 调 token budget。

## E08 · Prefix Cache 与 RadixAttention

**主问题：** prefix cache 的 key 应建立在原始文本还是 token IDs 上？Radix tree 相比精确哈希命中带来什么？

**追问：** 多租户环境中如何防止缓存泄漏？为什么请求编排顺序会改变命中率？

**强信号：** 以模型、tokenizer、模板/adapter 版本和 token 序列形成安全 key；radix tree 支持最长公共前缀与部分复用；cache-aware routing/scheduling 将相近请求聚集。需做 tenant namespace、权限、加密/清零、容量与 eviction 策略，不能只追全局命中率。

## E09 · 投机解码何时真的加速？

**主问题：** draft-target speculative decoding 为什么能保持目标模型分布不变？速度由哪些量决定？

**追问：** 为什么高 acceptance rate 仍可能没有收益？代码生成、低延迟小 batch 和高吞吐大 batch 的结论是否相同？

**强信号：** 理解 rejection sampling/残差分布与 KV rollback；速度取决于 acceptance、draft 成本、验证并行度、候选长度、batch 和目标模型单步成本。高并发下目标 GPU 已充分批处理时，额外 draft/验证可能降低总吞吐。

## E10 · 量化不能只看“能不能装下”

**主问题：** 为生产推理选择 BF16、FP8、INT8 或 INT4 时，你会如何决策？

**追问：** weight-only 与 weight+activation 量化的收益差别？为什么 KV Cache 精度是另一条轴？

**强信号：** 分清 HBM 容量、带宽、tensor core 支持、校准数据、per-channel/group、outlier、kernel 可用性和模型质量。用目标任务/长上下文/推理能力评测，不只看 perplexity；量化格式必须匹配硬件和引擎 kernel。

## E11 · Tensor Parallel 不是免费午餐

**主问题：** 一个模型单卡放不下，你如何在 TP、PP、EP 和 replica/data parallel 之间选择？

**强信号：** TP 每层有 collective，低延迟依赖 NVLink/NVSwitch；PP 有 bubble 且单请求流水难填；MoE 的 EP 引入 all-to-all；replica 扩吞吐但不解决模型容量。候选人会先看节点拓扑，再决定是否跨节点 TP，并把并行策略纳入容量和故障域。

## E12 · Engine 选型实验

**主问题：** vLLM、SGLang、TensorRT-LLM/其他引擎之间如何做选择？请设计一个两周内可完成的 bake-off。

**强信号：** 不从榜单直接选；固定模型/精度/硬件，用生产输入输出长度、prefix reuse、并发和结构化输出测试；比较 goodput、P99、稳定性、模型覆盖、可观测性、升级成本与团队能力；保留质量校验和失败请求。

## 素材来源

- [Inference Optimization](https://github.com/rohitg00/ai-engineering-from-scratch/blob/a56b4b8a/phases/10-llms-from-scratch/12-inference-optimization/docs/en.md)
- [Speculative Decoding and EAGLE-3](https://github.com/rohitg00/ai-engineering-from-scratch/blob/a56b4b8a/phases/10-llms-from-scratch/15-speculative-decoding-eagle3/docs/en.md)
- [vLLM Serving Internals](https://github.com/rohitg00/ai-engineering-from-scratch/blob/a56b4b8a/phases/17-infrastructure-and-production/04-vllm-serving-internals/docs/en.md)
- [Production Quantization](https://github.com/rohitg00/ai-engineering-from-scratch/blob/a56b4b8a/phases/17-infrastructure-and-production/09-production-quantization/docs/en.md)
