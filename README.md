# LLM 推理平台专家面试题库

面向负责自研或搭建 LLM 推理平台的资深工程师/架构师。本题库重点考察候选人是否能把模型原理、Serving Engine、GPU 集群和生产 SLO 串成一个可运行、可扩展、可运营的系统，而不是记忆某个框架的参数。

## 使用方式

- 先读 [面试官手册](interviewer-guide.md)，按岗位级别选择题目。
- 电话面/技术一面使用 [核心原理与推理引擎](questions/01-inference-engine.md)。
- 架构面使用 [分布式平台与调度](questions/02-platform-and-scheduling.md)。
- 生产能力面使用 [性能、可靠性与成本](questions/03-performance-reliability-cost.md)。
- 终面选择一道 [系统设计与实战题](exercises/system-design.md)。
- 每轮结束填写 [候选人评分卡](scorecards/candidate-scorecard.md)。

题目不是逐题问答清单。推荐沿一条主线连续追问，让候选人画图、估算和做取舍。优秀回答必须包含假设、量化指标、失败模式和验证方法。

## 推荐面试闭环

| 轮次 | 时长 | 重点 | 建议题目 |
|---|---:|---|---|
| 技术一面 | 60 分钟 | 推理原理与引擎内部 | E01、E03、E05、E08，任选 2 道追问 |
| 平台架构面 | 75 分钟 | 多租户、调度、扩缩容、分布式 | P01、P03、P06、P08 |
| 性能/SRE 面 | 60 分钟 | SLO、压测、故障与成本 | O01、O03、O06、O09 |
| 系统设计终面 | 90 分钟 | 端到端判断与沟通 | S01 或 S02 |

## 题库原则

1. 先问 workload，再谈 engine 和硬件。
2. 同时考察 TTFT、TPOT、吞吐、goodput、质量和成本，不接受单指标优化。
3. 要求候选人做数量级估算；具体型号参数可由面试官提供。
4. 接受不同技术路线，只要假设清楚、论证自洽、能设计验证。
5. 框架版本和基准数字会变化；评价重点是方法，不是背版本号。

## 素材与版本

题库从 [`rohitg00/ai-engineering-from-scratch`](https://github.com/rohitg00/ai-engineering-from-scratch/tree/a56b4b8a) 的 Phase 10 与 Phase 17 提炼并二次设计，素材快照提交为 `a56b4b8a`。每个章节末尾列出对应来源。题目和评分标准为本仓库独立内容。

## License

建议在首次发布前由团队补充许可证；若公开发布，请同时确认素材仓库的许可证与引用要求。
