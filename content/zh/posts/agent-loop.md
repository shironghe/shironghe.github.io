---
title: "Agent Loop 架构解析：为什么主流 Agent 都选 ReAct？"
description: "Agent Loop 是 Agent 的核心循环。从 ReAct、Plan-and-Execute、ReWOO 到 Reflexion，看懂常见架构，也理解 ReAct 为什么成为主流默认。"
date: 2026-08-25
draft: false
tags: ["Agent", "ReAct", "Agent Loop", "LLM"]
translationKey: "Agent Loop"
---

## 先看结论

Agent 不是“模型 + 工具”这么简单，真正让它具备自主性的，是连接两者的 Agent Loop：一次任务由多轮“思考、行动、观察”组成，直到目标完成。常见的循环架构有 ReAct、Plan-and-Execute、ReWOO、Reflection/Reflexion，以及多智能体编排。

主流框架和产品不约而同地选择了 ReAct 风格的循环，不是因为 ReAct 是唯一正确的答案，而是因为它在简单性、通用性、可解释性和生态适配之间取得了最好的平衡。长任务、成本敏感、质量敏感等场景，反而会考虑其他架构。

## 什么是 Agent Loop

LangChain 对 Agent 的定义很直白：Agent 是“模型在循环中调用工具，直到完成给定任务”[2](https://docs.langchain.com/oss/python/langchain/agents)。LlamaIndex 的描述更进一步：Agent 是由 LLM 驱动的半自主程序，被赋予任务后执行一系列步骤，每一步完成后判断任务是否完成，未完成就回到循环开头[3](https://developers.llamaindex.ai/python/framework/understanding/agent/)。Anthropic 则把这类系统统称为 agentic systems，并强调 Agent 的核心特征是“由 LLM 动态决定自己的流程和工具使用”[4](https://www.anthropic.com/research/building-effective-agents)。AWS 的科普也把 Agent 描述为利用模型推理并调用工具完成任务的系统[20](https://aws.amazon.com/what-is/ai-agents/)。

把不同定义抽出来，Agent Loop 通常包含以下组件：

- 目标与指令：用户或系统给出的任务，以及约束条件。
- 状态与上下文：对话历史、工具结果、短期记忆、长期记忆。
- 推理单元：LLM，负责根据当前状态决定下一步。
- 行动空间：工具、API、代码执行器、子 Agent 等。
- 观察与反馈：行动执行后的结果，写回上下文。
- 终止条件：任务完成、达到最大轮数、请求人工介入等。

用伪代码表示就是：

```python
while not done:
    thought = llm(context)        # 推理：理解状态并决定下一步
    action = parse(thought)       # 解析出具体行动
    observation = execute(action) # 执行工具并拿到结果
    context += observation        # 把观察写回状态
    done = check(context)         # 判断是否完成
```

举一个更具体的例子：用户说“帮我订一张下周二从上海到北京的机票”。Agent 不会在一次模型调用里完成所有事，而是先调用搜索工具拿到航班列表，再根据价格和时间筛选，调用预订工具下单，最后确认订单并汇报。中间的每一步都可能产生新的问题，例如没有直达航班、价格超预算、需要登录，Agent Loop 就是支撑这些“发现问题、采取行动、再看结果”反复迭代的骨架。

如果没有这个循环，LLM 只是一次性生成器：无法查证事实、无法与环境交互、也无法根据错误修正自己。Agent Loop 的价值在于形成反馈闭环：推理产生行动，行动产生观察，观察再驱动下一轮推理[5](https://lilianweng.github.io/posts/2023-06-23-agent/)[12](https://arxiv.org/abs/2309.07864)。

值得注意的是，Agent Loop 不等于让模型无限自由发挥。生产系统通常还会叠加最大迭代次数、token 预算、权限边界、人工审批和审计日志；Anthropic 明确区分了 workflows 和 agents，并建议大多数团队从可预测的 workflow 开始，只在需要模型动态决策时升级为 agent[4](https://www.anthropic.com/research/building-effective-agents)。Loop 是骨架，约束才是让它在生产环境里安全运行的部分。

## 常见 Agent Loop 架构

在 AI Agent 出现前，我们使用 LLM 协助完成任务的行为是询问 LLM，然后按照 LLM 的回答进行操作，将操作结果复制给 LLM，再按照 LLM 的回答操作，反复直到任务完成。Agent Loop 就是模拟这个行为。但不同的 Agent Loop 在不同的场景下对这一行为做了简化，最常见的就是 ReAct。

### ReAct：思考与行动交织

ReAct（Reasoning + Acting）由 Yao 等人在 2023 年提出[[1](https://arxiv.org/abs/2210.03629)]。它的核心是让 LLM 交替生成 Thought（思考）、Action（行动）、Observation（观察），把推理轨迹和外部交互写进同一个上下文。Simon Willison 曾给出一个几十行 Python 的极简实现，验证了这个循环本身非常简单[21](https://til.simonwillison.net/llms/python-react-pattern)。一个典型的 ReAct 片段：

```text
Thought: 用户想了解北京的天气，我需要查询天气 API。
Action: search("北京 天气")
Observation: 北京，晴，26°C
Thought: 天气信息已经拿到，可以组织回答。
Action: finish("北京今天晴天，气温 26°C。")
```

ReAct 论文在问答、事实核查、交互式决策等任务上验证了这种模式：相比纯 CoT，ReAct 通过外部工具减少了幻觉和错误传播；相比只行动不推理，ReAct 拥有更好的规划和异常处理能力，同时推理过程更可解释[1](https://arxiv.org/abs/2210.03629)。

具体来说，除了 HotpotQA 和 FEVER 这类知识密集型任务，ReAct 还在 AlfWorld、WebShop 等交互式决策环境中验证了效果：推理轨迹帮助模型跟踪计划、处理异常，动作则让模型能主动获取外部信息[1](https://arxiv.org/abs/2210.03629)。

优点：实现简单、任务通用、每步都能根据最新观察调整，并且思考过程天然可审计。缺点：每轮都要调用 LLM，token 和延迟成本高；一旦某步观察被错误理解，错误会沿轨迹累积。

### Plan-and-Execute：先计划，再执行

Plan-and-Execute 把“计划”和“执行”拆成两个角色：规划器先生成完整计划，执行器再逐步执行；计划执行中发现问题时，再让规划器重新规划。Plan-and-Solve 论文是这类思想的早期代表[8](https://arxiv.org/abs/2305.04091)，LangChain 也提供了对应的 Plan-and-Execute agent 模式。

AutoGPT 和 BabyAGI 也属于这一类：它们把目标拆成任务列表，每轮选择一个任务执行，再根据结果更新任务列表[13](https://github.com/Significant-Gravitas/AutoGPT)[14](https://github.com/yoheinakajima/babyagi)。

一个典型的流程是：规划器把“调研三家云厂商并输出对比报告”拆成“查询每家产品文档”“整理价格与限制”“生成结论”等步骤，执行器依次完成；如果某一步发现文档不存在，执行器会把问题交回规划器重新拆解。

优点：长任务下更稳定，规划阶段集中消耗 token，执行阶段更省；缺点：计划一旦脱离现实就会过时，需要在执行中频繁重规划，动态性不如 ReAct。

### ReWOO：推理与观察解耦

ReWOO（Reasoning WithOut Observation）发现，ReAct 式“推理到一半去查工具，查完再接着推理”会产生大量重复 prompt。它把过程拆成 Planner、Worker、Solver：Planner 一次性生成带占位符的计划，Worker 批量执行工具调用，Solver 最后读取所有结果并产出答案[6](https://arxiv.org/abs/2305.18323)。

论文报告，在 HotpotQA 上 ReWOO 实现了约 5 倍 token 效率提升，同时准确率提高约 4%，并且在工具失败时更鲁棒[6](https://arxiv.org/abs/2305.18323)。代价是中间过程不能根据工具结果动态改计划，适合工具结果相对稳定、成本敏感的场景。

例如规划器先生成 `查询天气(city={city})` 这样的占位计划，Worker 批量把 `{city}` 替换成具体城市执行，Solver 再基于所有返回结果统一作答，而不是每查一个城市都中断一次推理。

### Reflection / Reflexion：自我纠错循环

Self-Refine 提出“生成 -> 自我反馈 -> 再生成”的迭代流程，让同一个模型反复评价并改进自己的输出[9](https://arxiv.org/abs/2303.17651)。Reflexion 更进一步，把反思文本写进 episodic memory，跨轮次积累经验，相当于用语言做“强化学习”[7](https://arxiv.org/abs/2303.11366)。论文报告，Reflexion 在 HumanEval 上达到 91% pass@1，超过当时 GPT-4 的 80%[7](https://arxiv.org/abs/2303.11366)。

这类循环适合代码生成、数学推理、写作等“质量比速度重要”的任务。缺点同样明显：额外调用次数多，模型可能过度修正，把原本正确的输出改坏。

一个直观的例子是代码生成：Agent 先生成代码并运行测试，把测试失败信息转化为“这里没有处理空输入”之类的反思，下一轮带着这条反思重写代码，而不是从零开始。

### 其他变体

- Tree of Thoughts（ToT）：不沿单一路径思考，而是像搜索树一样维护多条思路，用广度/深度优先搜索选择下一步，适合需要探索的问题[10](https://arxiv.org/abs/2305.10601)。
- 多智能体循环：AutoGen 让多个 Agent 以对话方式协作[18](https://microsoft.github.io/autogen/0.2/)，CrewAI 用角色与任务定义团队[19](https://docs.crewai.com/)，LangGraph 则用图结构显式编排 supervisor、子 Agent 和条件路由[17](https://docs.langchain.com/oss/python/langgraph/overview)。
- CoALA：把记忆、行动空间和决策周期统一成“认知架构”，给 Agent 提供了一个更系统的分析框架[11](https://arxiv.org/abs/2309.02427)。

### 架构对比

| 架构 | 核心思想 | 优点 | 缺点 | 典型代表 |
| --- | --- | --- | --- | --- |
| ReAct | Thought/Action/Observation 交织 | 简单、动态、可解释 | token 多、错误累积 | LangChain、LlamaIndex、OpenAI function calling |
| Plan-and-Execute | 先计划再执行 | 长任务稳定、省 token | 计划易过时 | Plan-and-Solve、BabyAGI |
| ReWOO | 推理与观察解耦 | 5x token 效率、抗工具失败 | 动态性弱 | ReWOO 论文 |
| Reflexion | 反思 + 记忆 | 自纠错、质量高 | 调用多、可能过度修正 | Self-Refine、Reflexion |
| 多智能体 | 多角色对话/编排 | 分工、可并行 | 复杂度与成本高 | AutoGen、CrewAI、LangGraph |

## 为什么主流 Agent 都选 ReAct 风格

先定义一个观察：现在主流 Agent 几乎都是“ReAct 风格”的循环。OpenAI 的 Responses API 和 Agents SDK 把工具调用放进循环，模型决定调用哪个工具、拿到结果后再继续[15](https://openai.com/index/new-tools-for-building-agents/)[16](https://openai.github.io/openai-agents-python/)；Anthropic 推荐的 agent 模式同样是“模型 + 工具 + 循环”[4](https://www.anthropic.com/research/building-effective-agents)；LangChain 的 create_agent 定义就是模型在循环里调用工具[2](https://docs.langchain.com/oss/python/langchain/agents)；LlamaIndex 的 FunctionAgent 也是函数调用循环[3](https://developers.llamaindex.ai/python/framework/understanding/agent/)；AutoGPT、BabyAGI 则把“每步看观察再决策”放进了更长期的任务循环[13](https://github.com/Significant-Gravitas/AutoGPT)[14](https://github.com/yoheinakajima/babyagi)。

为什么大家都选择它？我认为可以归结为五点：

1. 足够简单，且不需要额外训练。ReAct 是纯 prompt 层面的模式，任何能理解指令的模型都能跑，不需要训练奖励模型或改造模型结构[1](https://arxiv.org/abs/2210.03629)。
2. 任务通用。它不预设任务结构，所有决策都基于最新观察，适合无法提前穷举分支的开放任务。
3. 可解释、可观察。Thought 提供推理依据，Action 提供明确动作，日志、调试、审计都方便；论文也强调它比纯推理或纯行动更可解释[1](https://arxiv.org/abs/2210.03629)。
4. 与模型能力和生态高度契合。现在的模型普遍经过指令跟随和工具调用训练，function calling 本质上就是“结构化输出的 ReAct”：先决定调用哪个函数，拿到结果后再继续。框架层因此天然围绕这个循环构建[2](https://docs.langchain.com/oss/python/langchain/agents)[15](https://openai.com/index/new-tools-for-building-agents/)[16](https://openai.github.io/openai-agents-python/)。
5. 成本与效果的平衡最好。Anthropic 在工程实践中观察到，最成功的实现往往是“简单、可组合的模式，而不是复杂框架”[4](https://www.anthropic.com/research/building-effective-agents)；ReAct 恰好是最简单的可组合循环。

还有一个容易被忽略的原因：ReAct 论文出现的时间点恰好和工具调用能力普及重叠。模型本身开始原生支持 function calling 之后，框架只要把“Thought/Action/Observation”映射成“选择工具/调用工具/接收结果”，就能立刻获得一个可用的 Agent，几乎不需要额外工程。ReAct 因此不只是论文里的方法，更成了整个生态对 Agent 循环的默认心智模型[15](https://openai.com/index/new-tools-for-building-agents/)[16](https://openai.github.io/openai-agents-python/)。

但这里必须补充两个例外，否则会把结论讲得太绝对：

第一，很多生产系统并不是“严格 ReAct”。它们可能不显式输出 Thought，而是直接输出函数调用；或者把循环交给 API 内部实现，只暴露工具列表。严格 ReAct 更像这类工具调用循环的思想原型，而不是必须照抄的文本格式。

第二，ReAct 不是所有场景的最优解。任务复杂且稳定时，Plan-and-Execute 更省 token、更可控；工具调用密集且推理可一次性完成时，ReWOO 能显著降低成本；需要高最终质量时，Reflexion 的自我纠错更有价值；需要多人协作时，多智能体编排比单个 ReAct 循环更合适[6](https://arxiv.org/abs/2305.18323)[7](https://arxiv.org/abs/2303.11366)[8](https://arxiv.org/abs/2305.04091)[17](https://docs.langchain.com/oss/python/langgraph/overview)[18](https://microsoft.github.io/autogen/0.2/)。

所以更准确的说法是：主流 Agent 默认选择 ReAct 风格，因为它用最少的机制解决了“推理-行动-观察”闭环，而其他架构是在这个基础上针对特定约束做优化。

更进一步说，如果任务的目标和步骤从一开始就完全确定，那它往往更适合做成 workflow，而不是 agent；Agent 的价值来自不确定性，而 ReAct 恰恰是处理不确定性的最轻量方式。这也是为什么它在“需要动态决策”的通用 Agent 里成为默认，而不是在一切系统里被强行使用[4](https://www.anthropic.com/research/building-effective-agents)。

## 常见误区

1. Agent 等于“一次 LLM 调用 + 工具”。实际上没有循环就没有 Agent，单次调用只是工具增强的问答。
2. ReAct 必须显式输出 Thought。严格 ReAct 需要，但生产实现常把思考隐藏，直接输出函数调用，效果类似。
3. 架构越复杂越高级。Anthropic 的经验是先用简单可组合的模式，复杂框架反而增加风险[4](https://www.anthropic.com/research/building-effective-agents)；先选对循环，再谈复杂度。

## 选型建议

| 场景 | 推荐架构 | 原因 |
| --- | --- | --- |
| 通用助手、开放问答 | ReAct 风格 | 简单、动态、可解释 |
| 长流程任务、明确步骤 | Plan-and-Execute | 稳定、省 token |
| 工具密集、成本敏感 | ReWOO | token 效率高 |
| 代码/推理/写作质量要求高 | Reflexion | 自纠错 |
| 多角色复杂协作 | 多智能体 | 分工与并行 |

## 总结

Agent Loop 是 Agent 的真正核心：它把 LLM 从“生成器”变成“执行者”，把单次推理变成与环境不断交互的闭环。ReAct 之所以成为主流默认，是因为它在简单性、通用性、可解释性和生态适配上的综合表现最好，而不是因为它“必须如此”。理解不同架构的取舍，才能在具体场景里做出合理选择。

## 参考资料

1. Yao, S. et al. [ReAct: Synergizing Reasoning and Acting in Language Models](https://arxiv.org/abs/2210.03629). arXiv:2210.03629, 2023.
2. LangChain. [Agents](https://docs.langchain.com/oss/python/langchain/agents).
3. LlamaIndex. [Building an agent](https://developers.llamaindex.ai/python/framework/understanding/agent/).
4. Anthropic. [Building Effective AI Agents](https://www.anthropic.com/research/building-effective-agents).
5. Weng, L. [LLM Powered Autonomous Agents](https://lilianweng.github.io/posts/2023-06-23-agent/). Lil'Log, 2023.
6. Xu, B. et al. [ReWOO: Decoupling Reasoning from Observations for Efficient Augmented Language Models](https://arxiv.org/abs/2305.18323). arXiv:2305.18323, 2023.
7. Shinn, N. et al. [Reflexion: Language Agents with Verbal Reinforcement Learning](https://arxiv.org/abs/2303.11366). arXiv:2303.11366, 2023.
8. Wang, L. et al. [Plan-and-Solve Prompting](https://arxiv.org/abs/2305.04091). arXiv:2305.04091, 2023.
9. Madaan, A. et al. [Self-Refine: Iterative Refinement with Self-Feedback](https://arxiv.org/abs/2303.17651). arXiv:2303.17651, 2023.
10. Yao, S. et al. [Tree of Thoughts](https://arxiv.org/abs/2305.10601). arXiv:2305.10601, 2023.
11. Sumers, T. et al. [Cognitive Architectures for Language Agents](https://arxiv.org/abs/2309.02427). arXiv:2309.02427, 2023.
12. Xi, Z. et al. [The Rise and Potential of Large Language Model Based Agents: A Survey](https://arxiv.org/abs/2309.07864). arXiv:2309.07864, 2023.
13. Significant Gravitas. [AutoGPT](https://github.com/Significant-Gravitas/AutoGPT). GitHub.
14. Nakajima, Y. [BabyAGI](https://github.com/yoheinakajima/babyagi). GitHub.
15. OpenAI. [New tools for building agents](https://openai.com/index/new-tools-for-building-agents/).
16. OpenAI. [Agents SDK](https://openai.github.io/openai-agents-python/).
17. LangChain. [LangGraph overview](https://docs.langchain.com/oss/python/langgraph/overview).
18. Microsoft. [AutoGen 0.2](https://microsoft.github.io/autogen/0.2/).
19. CrewAI. [Documentation](https://docs.crewai.com/).
20. AWS. [What are AI Agents?](https://aws.amazon.com/what-is/ai-agents/).
21. Willison, S. [A simple Python implementation of the ReAct pattern for LLMs](https://til.simonwillison.net/llms/python-react-pattern). TIL.
