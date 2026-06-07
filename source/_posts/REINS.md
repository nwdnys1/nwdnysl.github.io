---
title: SJTU-REINS
date: 2024-10-05 21:19:00
categories: REINS
tags:
  - 读研
index_img:
banner_img:
excerpt: " "
---

# PHAROS

灯塔项目基于流控分离思想的无人机管理系统 希望通过云边协同计算框架对于无人机的整体态势进行计算和分析 让无人机自己通过安全空间等态势数据控制规划路径

## CEDS

CEDS 是黄子昂学长的硕士毕业论文中设计的云边融合存储系统 大概包含以下几个部分：

### 边缘数据就近存储机制

大部分云边融合系统会将数据存储于资源富足可扩展的云端数据库中 但这会带来很多不必要的传输开销 利用边缘节点本身的存储能力 可以将数据就近存储 大大减少传输延迟

然而数据的就近存储会带来其他问题 比如原本全都位于云端中心的数据 由于就近存储就会分布在不同的边缘节点上 针对此问题 CEDS 提出一种基于 Rosetta 过滤器的全局索引机制 简单来讲 这个过滤器能够实现一个数据表内任一字段的范围过滤 即给定一个字段的某一范围 在常数时间内返回这个表是否存在这个范围内的数据 基于此种过滤器 加上基于时间划分的数据分表 可以做到在很短的时间内定位到某一个时间范围内的数据分表 并快速判断哪些分表内存在本次范围查询相关的数据 这就是 CEDS 的全局索引机制 基于此种索引 CEDS 在云端仅需存储数据表的元信息摘要（包括过滤器的 bitarray） 同时这也将为后文的查询下推提供便利

### 查询任务拆分下推

大部分云边存储系统基于边缘节点的存储数据 每次查询会将相关的数据表全部聚合到云端后进行筛选 产生了很多不必要的传输开销 CEDS 基于任务划分 将查询和聚合任务下推给各个相关的边缘子节点 减少带宽开销

具体而言 一次查询会先向云端中心索要本次查询可能涉及到的所有数据分表 以及这些分表存储在哪些边缘节点上 其方法可以简单的由之前提到的过滤器实现

获取到节点列表后 只需要把子查询任务分发给这些节点即可 每个子节点会进行相关的范围查询（节点内存在常规的索引来加快查询）并把查询结果返回 最终在查询节点进行数据聚合任务 把结果返回给用户

### 负载感知数据迁移

数据就近存储定会存在不均衡现象 由于查询任务划分给子节点并发执行 查询的延迟显然由查询时间最大的子查询决定 如果某些节点上的数据查询次数特别多 导致超出边缘节点的承受范围 导致子查询阻塞 就会导致所有涉及的查询延迟显著变高 为了防止此种情况出现 CEDS 会监测并记录每段时间内边缘节点的查询资源开销 比如记录前 3 分钟内某一节点的内存占用 一旦超过 1G 就判定存在热点数据 并将热点数据迁移到云端来减缓此节点的压力

此种策略是折中的策略 理论上可以达到最均衡的方案肯定是为每一个边缘节点根据查询开销动态分配性能资源 但这在现实中肯定不可行 因此使用较为折中的策略来解决数据不均衡分布的问题

## 技术栈

### Redisearch

- 用 redisearch 建立索引 实现实时数据查询的加速
- 文档参考：https://redis.io/docs/latest/develop/interact/search-and-query/indexing/（redis不同类型的索引如何使用）https://redis.readthedocs.io/en/stable/examples/search_json_examples.html#Projecting-using-JSON-Path-expressions（将 JSON 数据添加到索引的示例代码）

### Apache Avro

- 用 apache avro 进行数据的序列化
- 文档参考：https://avro.apache.org/docs/

# TODO

## REVIEW

- 评估体系：固定接口、数据格式等其它影响 只对发送规则做区分 探究其收益 论文声称这样的场景更符合实际
- 问题假设：J是全体回报 z是每一个agent的历史状态编码 P是一个信道模型 用逻辑斯蒂来模拟通信概率 VPB是每比特消息可以带来的团队收益
- COUNTERCOMM：一个创新的发送规则 用一个GNN来预测团队奖励 节点是每个agent的状态 边则代表了两个agent是否可以通信 每一步决策会先计算每个agent发送b比特消息或不发送的团队收益 然后计算VPB 最后用softmax选取一个VPB较高的b 发送消息
- 实验：设计了三个场景 计算归一化回报（除以全发机制的回报）并进行消融实验 验证反事实评分与推演步数H的有效性
- STRENGTHS
  - 创新的评估体系
  - 创新的机制带来的效率提升
  - 充足完整的实验
- WEAKNESSES
  - 延迟较高 影响实际应用（高回报率场景、低计算资源平台）
  - 仿真环境距离实际场景较远 比如信道模型可能过于理想 存在较大的gap
  - 比特成本的线性假设是否合理？实际场景中 消息通信的开销并不是线性的 存在固定的协议开销 这一事实可能会影响模型的最优性能

---

**Review for Paper: "Fixed-Interface Swarm Communication Scheduling: Evaluating Paired Send-Versus-Silent Pricing"**

**Overall Assessment:** This paper presents a well-motivated, controlled, and rigorous study on a specific yet practical problem in multi-agent communication: optimizing the send-rule (scheduler) under a completely fixed communication interface. The core innovation lies in proposing and thoroughly evaluating the "paired send-versus-silent counterfactual pricing" mechanism. The experimental design is exemplary, and the results are compelling within the defined scope. The paper is recommended for publication, as it makes a clear, focused contribution to the field of communication-efficient multi-agent systems. However, its practical deployment readiness is limited by the computational overhead and the simulation-based evaluation.

**Strengths:**

1.  **Innovative Evaluation Framework and Clear Problem Scoping (Novelty of the "Fixed-Interface" Regime):**
    The paper's greatest strength is its methodological rigor. By _freezing_ the reactive controller shell, the 8-bit VQ payload alphabet, the fitted channel model, and the pooled load target, the authors successfully isolate the variable of interest: **send-rule quality**. This "fixed-interface" setup transforms a typically ambiguous claim about scheduler performance into a clean, falsifiable scientific question. The paper does not aim to beat all communication architectures but to determine if a better scheduling rule can extract more value from an _existing, committed_ workflow. This framing is both novel and highly relevant for real-world systems where low-level stacks are often fixed after deployment.

2.  **Innovative Mechanism and Demonstrated Efficiency Gains (Novelty of the "Paired Pricing" Core):**
    The proposed **COUNTERCOMM** mechanism is elegant and effective. The core idea of performing short-horizon, paired counterfactual rollouts ("send b bits" vs. "stay silent") to compute a **Value-Per-Delivered-Bit (VPB)** score is a significant conceptual contribution. It moves beyond simple binary triggering or scalar penalty methods. The VPB metric naturally balances the expected marginal team value of a message against its resource cost (bits), internalizing the congestion externality in a decentralized manner. The ablation studies (e.g., Table 2) strongly support the claim that this multi-step, paired pricing is the active ingredient, not just the use of a world model or encoder scaffold.

3.  **Extensive, Rigorous, and Targeted Experimental Suite:**
    The experimental section is a masterclass in thorough evaluation. The authors go far beyond a simple performance comparison table. Key evidence includes:
    - **Matched-load fairness:** Strictly comparing methods at the same pooled transmitted bit budget (48.2% of dense broadcast).
    - **Novelty-critical foils:** Comparing against strong _same-interface_ baselines (Lyapunov-Deadband, Learned Trigger, Adaptive Self-Triggered) rather than richer, incomparable architectures.
    - **Comprehensive falsification checks:** The paper systematically addresses potential counter-arguments through latency-bounded analysis, information-pattern ablations, channel-model calibration/mismatch tests, heterogeneous-agent holdouts, packet-trace replays, and a second-stack (ROS 2) benchmark. This "evidence package" greatly strengthens the validity of the core claim.

**Weaknesses and Limitations:**

1.  **Non-Trivial Computational Overhead, Impacting Practical Applicability:**
    The paper transparently reports a median per-agent decision latency of **12.6 ms** for the full COUNTERCOMM (H=5). While this is framed as a "compute-cost boundary," it consumes a substantial portion (~25%) of a 20 Hz (50 ms) control interval _before_ accounting for sensing, actuation, or network transport. This overhead is the direct cost of the multi-step rollouts. Although a cheaper H=1 variant is explored, the highest gains are tied to higher latency. This presents a significant barrier for deployment on real robotic platforms with tight control loops or limited onboard compute.

2.  **Simulation-to-Reality Gap: Channel and Environment Modeling:**
    The evaluation, while extensive, remains within a **custom simulation framework**. The channel model, though more advanced than a fixed drop rate, is still a fitted logistic function of load and size, not accounting for explicit RF propagation, interference, queueing dynamics, or precise timing jitter. The paper's claims about robustness are therefore bounded by its simulation assumptions.

3.  **Questionable Modeling Assumption: Linear Bit Cost:**
    A fundamental modeling choice is the **linear relationship between bit count `b` and resource cost** in the VPB denominator (`b + ε`). The authors justify this via their fitted channel model where bit count linearly reduces log-odds of delivery. However, in real-world networks, transmission cost is rarely linear in bits due to protocol overheads (preambles, headers, ACKs), modulation schemes, and the fact that small messages often occupy fixed-duration time slots. This assumption, while simplifying the optimization, may not hold in practice and could affect the optimality of the scheduler's bitrate choices on real hardware.

**Summary and Recommendation:**
This is a **theoretically strong and methodologically exemplary paper** with clear, impactful contributions to the study of communication scheduling in multi-agent systems. Its "fixed-interface" evaluation paradigm and the "paired counterfactual pricing" mechanism are important conceptual advances supported by exceptionally rigorous experimentation. The primary weaknesses are practical: the computational latency and the simulation-based validation limit immediate deployment readiness. The authors are commendably transparent about these limitations. The work is best viewed as a **foundational study that provides a powerful new tool and evaluation standard for the community**, paving the way for future research into more computationally efficient approximations and physical validation.

**Decision:** **Accept.**

## AAAI

- 用transformer做自回归的优势到底是？
- 条件生成（对于自回归采样而言很简单 只需要基于给定数据继续采样即可）
- 拟合任意形状的连续分布 比如diffusion来建模连续特征 或是参数化一个mlp 或是采用特殊编码
- 更实践应用的DP会计（比如模型并不是每一代都发布、用公开数据集预训练）从而确保模型的训练更有效稳定 以及一些经验性攻击指标来验证dp
- 改良损失函数 比如引入下游分类器 或是统计指标 从而提升模型训练的稳定性
- 类似LLM一样 进行预训练+微调
- 插值场景

## PHAROS

- 湘家荡：
  - 把4D和CEDS结合起来
  - 接入原CEDS的datareceiver和index部分（主要是数据聚合和查询下推）
- java 这边的 avro 由于我使用了 fastjson 导致需要先解析为 jsonobject 再进行使用 需要手动分类序列化 未来可以考虑直接使用 avro 的 record 类
- avro 的压缩 如何自动生成 schema 文件？如何定义统一的 realtime_map 的 schema？（环境变量）
- 索引层
  - 需要结合之前的查询下推实现中心的查询接口 目前用 http 简单实现
  - 目前 GM 的 schema 是固定为 taxi 的 后续加入转换逻辑适用于不同设备 转换逻辑可以是启动时发送请求获取 schema 然后存在内存里待用
  - GM 的时间索引可能不如遍历筛选 未测试
- 边缘节点
  - 向邻居节点发送请求可以并发 不过由于 http 有长连接机制 请求应该不会重新建立连接 且一个设备的计算最多也就发送 3 个请求（大多数情况是 1 个） 所以并发处理的意义不大
  - realtime_map 中接收消息时的 time 和消息中的 time 有延迟 差不多是 2s 不知道为什么
  - 节点刚好处于区域边界时可能会有问题 暂时没做测试
  - 断开连接时如何告知节点 目前是心跳计时实现的 考虑是否要更加实时（反正会去重 其实无所谓）
  - 数据格式变了导致 datahandler 需要重新写
- 中心
  - 现在并发处理消息的逻辑比较简单 就是一个 List<ExecutorService> 里面存了 n 个单线程池 每次请求根据时间戳取模分配一个线程池处理 不知道是否均匀
  - 中心下发态势数据是 for 循环+publish 不知道 mqtt 的 publish 是不是异步的 可能需要多线程或者至少保证异步发送
- 安全空间
  - jedis 中取出的 Document 类中的 properties 不知道为什么是$={}的格式 导致必须要解析 string

## Other

- 多智能体相关
- 查论文 谷歌学术 图书馆网站的电子数据库 VLDB ICDE SIGMOD ICDM EDBT 等会议 一是空间索引相关 二是查询优化 queryplan 清华李国良 ai 优化查询计划
- 后续可能要参与的工作
  - SQL 生成 类似的 灯塔的索引层也可以通过自然语言生成查询 最终返回结果 其中转换过程自己定 比如 NL 转换为时空具体范围 然后进行查询 或者转化为具体的查询窗口 另外比较重要的是 要根据数据的存储进行优化 这个是比较抽象的 具体方式可以用本地的 LLM
  - 发票的图神经网络 对于边敏感的神经网络可以用于识别行贿违法的 pattern 然后用模型学习并推理出哪些是异常的 另外还有个知识图谱
  - kfm 在做查询下推相关 我可以看看相关论文 结合到索引层
