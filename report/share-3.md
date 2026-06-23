# 对抗协作闭环：多Agent生成评估与大数据智能交互





![202606162154214](https://aea62e6.webp.li/2026/06/202606162154214.png)





## 闭环 [Loop Engineering](https://www.langchain.com/blog/the-art-of-loop-engineering)

输入一段提示词，等结果，手动改一轮，再问一次。整个过程本质上还是人在一下一下的拧螺丝。即便是在用Claude、GPT 这些最强模型，实际还是停留在高级聊天。

这有点像请了一个很能干的实习生，本来可以让他自己创意、自己文案、自己检查、自己改完再交付。结果还是要求他每写一小段就停下来等你确认。人还是那个人，能力也没变，慢就慢在交互方式没升级。

随模型能力越来越强，真正能把效率拉开差距的，不是还在手动一轮轮来回对话，是有没有把AI从问答工具用成可循环执行的工作流。



## 对抗协作、大数据、多Agent

> 本次分享聚焦于**生成对抗与评估**（Generative Adversarial & Evaluation）、**多Agent协作系统**（Multi-Agent Collaboration）以及**大数据场景下的智能数据交互**（Big Data Intelligence）。这三个领域的交叉正在催生一种全新的系统架构范式：由多个异构AI Agent通过对抗性协作，在生成、批判、审计、验证的闭环中处理大规模结构化数据。

第一眼印象我理解的是 LLM 后训练，可以完美覆盖上述3个关键词的领域。

LLM 一个吸收了全人类文明上下几千年的知识结晶。预训练用的数据是**静态的、历史的**（互联网快照）；而在后训练中用的是多Agent在生成式对抗与协作（RLHF、RLVR、Agent Swarm等）中产生海量的大数据。例如，两个Agent互相辩论数学题，它们生成的每一步推理链条、每一次反驳、每一次妥协，都是全新的、具有推理深度的**合成数据（Synthetic Data）**；这些数据反过来再喂给模型，形成**“自我对弈（Self-play）”**的数据飞轮。



![image-20260623104329583](./assets/image-20260623104329583.png)



## LLM 与 Agent 界线？

> LLM 侧做还是 Agent 侧做？答案是都可以，像tools、memory、multi-agent 等。
>
> - memory: Agent 侧做应该都了解（Cline SR 的memory bank），LLM 侧（deepseek Engram、evermind MSA）； 
> - “Best of N”：deepseek r1 后训练中的 GRPO （让模型生成多个候选答案，内部互相比较、挑刺、最终优胜者获得奖励），同样 agent 侧，[XiaomiMiMo/MiMo-Code: MiMo Code: Where Models and Agents Co-Evolve](https://github.com/XiaomiMiMo/MiMo-Code) 的Max Mode并行采样（Max Mode 在每一轮并行生成 N 个候选方案（默认 N=5），每个候选独立完成推理和工具调用规划，然后由同一个模型作为 judge，对比所有候选的推理过程和行动计划，选出最优的一个执行）。



- *大数据*：后训练使用大量数据（偏好数据、强化学习数据），但比预训练小得多且**质量高**得多（arXiv:2510.25817/CPQS-Tuning等），数据量大小参考Deepseek V3/R1 Paper。另外，多智能体和环境交互时，会生成海量的轨迹数据。==> **如何获取高质量数据？**
- *多Agent*：后训练越来越多地涉及多智能体系统（如生成式对抗网络在LLM中的应用、多智能体RLHF、RLVR、AI互评、MAGIC 框架、AegisLLM、Multi-Agent Debate）。==>  **如何设计多Agent 编排/架构？**
- *对抗协作*：典型的 LLM as a judge，例如，生成器与判别器（GANs）、红队与蓝队攻击、辩论智能体、自我对弈（AlphaGo式）。==> **如何实现自我迭代？**



### **如何获取高质量数据？**

- 爬：互联网爬取、过滤、清洗等；
- 买：购买标注过的优质数据 ；
- 造：LLM 合成数据；





### **如何设计多Agent 编排？**

Text-to-SQL领域的多Agent架构在过去五年经历了三次范式跃迁：从静态手工设计的流水线，到可训练的Agent工作流，再到强化学习驱动的自主进化系统。



![image-20260623131512669](./assets/image-20260623131512669.png)

2023年末，[MAC-SQL](https://arxiv.org/abs/2312.11242) 作为首个系统性的多Agent协作框架，奠定了三Agent架构的基本范式——由核心Decomposer Agent负责few-shot Chain-of-Thought SQL生成，Selector Agent获取子数据库，Refiner Agent纠错——在BIRD基准上达到59.59%的执行准确率。这一架构的核心在于：**将单一LLM的单体推理拆解为多个专门化角色的协作**，能够在不增加模型参数的前提下提升系统级表现。





![image-20260623132925372](./assets/image-20260623132925372.png)

2024年CHESS [(stanford.edu)](https://scalingintelligence.stanford.edu/pubs/CHESSpaper/) 提出**四Agent层次化架构（Information Retriever → Schema Selector → Candidate Generator → Unit Tester**），通过自适应Schema剪枝将工业级数据库（超过4,000列）的token消耗降低83%，同时将每查询API调用成本从\$0.0605压缩至\$0.0265。



![image-20260623133813807](./assets/image-20260623133813807.png)

[CHASE-SQL](https://arxiv.org/html/2510.17586v3) 则引入了三路候选生成策略（ ICL-based SQL Generation、 Divide-and-Conquer-based SQL Generation、 Skeleton-based SQL Generation）配合二元选择Agent，以75.07%的BIRD准确率超越同期方法。



![image-20260623140313442](./assets/image-20260623140313442.png)

同一时期， [XGenerationLab/XiYan-SQL: A MULTI-GENERATOR ENSEMBLE FRAMEWORK FOR NATURAL LANGUAGE TO SQL](https://github.com/XGenerationLab/XiYan-SQL) 通过**多生成器集成**和**M-Schema半结构化模式**表示，将准确率推升至75.63%。(Fine-tuned)
体验产品请访问：https://bailian.console.aliyun.com/xiyan

![image-20260623134948114](./assets/image-20260623134948114.png)



![image-20260623135840214](./assets/image-20260623135840214.png)

2025年强化学习与多Agent架构的深度融合 [MARS-SQL](https://arxiv.org/pdf/2511.01008)

- Grounding Agent负责推理驱动的Schema识别与干扰项过滤；
- Generation Agent通过多轮ReAct风格的Think-Act-Observe循环进行交互式查询构建；
- Validation Agent将解决方案选择重新框架化为生成建模任务；

三者通过强化学习策略而非手工Prompt进行联合训练，准确率在BIRD-DEV 测试集上达到77.84%、Spider测试集上达到89.75%。MARS-SQL的核心方法论突破在于"使Agent工作流可训练"：此前所有多Agent系统的协作逻辑均由人工设计，而MARS-SQL通过RL策略让Agent自主发现最优交互模式。







**标准化管线模式：Schema Pruning → Planning → Generation → Validation → Selection**

[BAPPA: Benchmarking Agents, Plans, and Pipelines for Automated Text-to-SQL Generation](https://arxiv.org/pdf/2511.04153)

尽管具体实现各异，Text-to-SQL系统已收敛于一条标准化的五阶段管线。

- 第一阶段Schema Pruning解决大规模数据库的模式过载问题——CHESS的Schema Selector可处理超过4,000列的工业级模式，将token数减少5倍 [(stanford.edu)](https://scalingintelligence.stanford.edu/pubs/CHESSpaper/) ；MARS-SQL的Grounding Agent通过推理驱动的干扰项过滤提供更精确的Schema上下文 [MARS-SQL](https://arxiv.org/pdf/2511.01008) 。对于小型模式场景，研究表明引入Schema Selector可能引入precision-recall权衡，此时省略该环节反而更优。

- 第二阶段Planning将自然语言问题转化为结构化的查询计划。CHASE-SQL的Query Plan CoT策略和Swiggy Hermes V3的意图解析模块均属于此类，后者通过ReAct风格的推理循环将模糊问题分解为离散任务，在模糊提示上实现20-25%的准确率提升 [(InfoQ)](https://www.infoq.com/news/2026/01/swiggy-hermes-conversational-ai/) 。
- 第三阶段Generation是系统的核心，当前最佳实践倾向于**多候选生成而非单输出**：CHASE-SQL的三路生成策略 [(arXiv.org)](https://arxiv.org/html/2510.17586v3) 、XiYan-SQL的ICL/SFT双模式生成 [XGenerationLab/XiYan-SQL: A MULTI-GENERATOR ENSEMBLE FRAMEWORK FOR NATURAL LANGUAGE TO SQL](https://github.com/XGenerationLab/XiYan-SQL)、以及SIRIUS-SQL的difficulty-smoothing RL训练 [(arXiv.org)](https://arxiv.org/html/2606.01246v1) ，均旨在最大化候选空间的多样性与可执行性。
- 第四阶段Validation通过执行反馈锚定质量。SIRIUS-SQL提出执行反馈生命周期的三层次架构：预执行筛查→执行锚定诊断→失败感知修复，所有候选必须重新执行后才能重新进入候选池 [SIRIUS-SQL](https://arxiv.org/html/2606.01246v1) 。DeepEye-SQL则从软件工程实践借鉴确定性单元测试（Syntax/Logic/Quality Checker），以确定性验证替代概率性LLM自评估 [DeepEye-SQL](https://arxiv.org/html/2510.17586v3) 。
- 第五阶段Selection决定最终输出：CHASE-SQL的消融实验表明，二元选择Agent（pairwise comparison）比self-consistency提升4.17%，比ranker agent提升7.5%  [DeepEye-SQL](https://arxiv.org/html/2510.17586v3) 。



在大数据场景维度，一个被反复验证的核心认知是**数据质量优于数据量**。[ReViSQL](https://arxiv.org/pdf/2603.20004) 证明：经人工验证的2,500条训练样本带来的模型表现比9,000条未验证样本高出8.2-13.9个百分点。BIRD训练集中超过50%的标注错误被[SIRIUS-SQL](https://arxiv.org/html/2606.01246v1) 的重新标注工作揭示。与此同时，公共基准与企业部署之间存在巨大的鸿沟——Spider 1.0上90%+的准确率在Spider 2.0真实企业工作流中骤降至6-31%，Uber内部Text-to-SQL应用与黄金标准表的重叠率仅约50%  。这一鸿沟使得多Agent审计管线从"可选优化"转变为"生产必需"。

[(arXiv.org)](https://arxiv.org/html/2606.02240v2)



![img](https://www.pythian.com/hs-fs/hubfs/undefined-Jan-12-2026-06-07-55-1151-PM.png?width=1536&height=1024&name=undefined-Jan-12-2026-06-07-55-1151-PM.png)

在多Agent协作维度，MCP（Model Context Protocol）与A2A（Agent-to-Agent）协议栈形成了互补的垂直-水平集成架构：MCP处理Agent与工具之间的标准化访问，A2A支撑Agent间的语义化协作。然而，MAST（Multi-Agent System Testing）研究揭示了令人警醒的现实：当前最先进的多Agent系统失败率高达41%-86.7%，其中79%的失败根源于规范与协调问题而非模型能力不足 [(Pythian)](https://www.pythian.com/blog/from-prompts-to-processes-building-reliable-nlp-to-sql-with-multi-agent-reasoning) 。多Agent编排的复杂性催生了动态拓扑优化方法——DyLAN框架通过运行时团队调整将MMLU任务准确率提升25% [(arXiv.org)](https://arxiv.org/html/2606.01246v1) ，AFlow框架借助蒙特卡洛树搜索（MCTS）自动发现优于人工设计5.7%的工作流拓扑 [(databricks.com)](https://community.databricks.com/t5/generative-ai/using-genie-in-multi-agent-systems-sql-query-missing-from-output/td-p/120534) 。





多Agent管线中的"单文化陷阱"与异构对抗编排的必要性。若五阶段管线的所有Agent节点使用相同基础模型（如均为GPT-4或Claude），整个验证层实际上共享同一套推理盲点——UC Berkeley MAST研究识别出这种"单文化问题"导致错误率41%-86.7%的系统中79%的失败源于协调失效 [(Pythian)](https://www.pythian.com/blog/from-prompts-to-processes-building-reliable-nlp-to-sql-with-multi-agent-reasoning) 。Zhang等人（2025）的综合基准进一步证明：同构多Agent辩论在大多数设置中很少超过简单基线，模型异质性才是有效协作推理的关键 [(arXiv.org)](https://arxiv.org/html/2606.01317v1) 。真正的对抗性验证要求生成器、批判器、审计器分别来自不同模型族甚至不同架构（LLM+规则引擎+程序分析），否则多Agent编排退化为同一模型的自我对话。



从"固定管线"到"可进化拓扑"——RL驱动的动态Agent编排。** 固定五阶段管线对不同复杂度查询采用"一刀切"处理：简单SELECT查询承受不必要的延迟成本，复杂多表JOIN可能因管线固定而无法获得足够验证。RoboPhD通过ELO进化机制在18次迭代中将代码量从70行扩展至1500行，BIRD准确率达到73.67% [(arXiv.org)](https://arxiv.org/html/2605.10267v1) 。DyLAN证明动态拓扑可提升25%准确率且对模拟Agent故障仅下降0.3点（基线下降13点） [(arXiv.org)](https://arxiv.org/html/2606.01246v1) 。AFlow的MCTS搜索发现的工作流比人工设计高5.7%、比现有自动方法高19.5% [(databricks.com)](https://community.databricks.com/t5/generative-ai/using-genie-in-multi-agent-systems-sql-query-missing-from-output/td-p/120534) 。这些证据指向一个结论：最优管线应根据查询复杂度自动调整Agent参与数量和顺序，根据历史表现持续优化编排策略。



多Agent管线的经济可行性与最优规模。** 多Agent系统的token成本是单Agent的2-5倍，通信开销随Agent数量超线性增长（O(N²)），安全措施的部署又进一步降低协作效率（"安全税"） [(arXiv.org)](https://arxiv.org/html/2605.28122v1) 。但两个对冲机制同时存在：TokenDance通过KV Cache共享实现11-17倍压缩率，在相同延迟SLO下支持2.7倍更多并发Agent [(arXiv.org)](https://arxiv.org/html/2604.20911v1) ；异构Agent团队中较小模型承担大部分工作、仅在必要时调用大模型，混合方法可在30美元成本下实现85%准确率（纯Claude方案需42.58美元） [(arXiv.org)](https://arxiv.org/html/2604.04989v1) 。RoboPhD的"逆向缩放效应"进一步证明进化在较弱模型上提供更大收益（Haiku +8.9分 vs Opus +2.3分） [(arXiv.org)](https://arxiv.org/html/2605.10267v1) 。三个优化叠加后，五阶段管线的增量成本可能控制在单模型的1.5-2倍以内，存在一个可通过MCTS动态搜索逼近的"最优Agent规模"。





Uber & Databricks 











Code Review

> 任务驱动：issue(task panel) -> multi-agent(security/grammar/arch/performance等)/multi-modes -> self-test -> create PR(repo 原有test) -> Merge





### **如何实现自我迭代？**

> Insight：模型自我迭代，agent 同样

- 任务面板
- 多个触发
- memory
    - tape
    - msa
    - nowledge





