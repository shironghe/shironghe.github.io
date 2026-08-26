---
title: "Agent Loop Architecture Explained: Why Do Mainstream Agents All Choose ReAct?"
description: "Agent Loop is the core loop of an Agent. From ReAct, Plan-and-Execute, ReWOO to Reflexion, understand common architectures and why ReAct has become the mainstream default."
date: 2026-08-25
draft: false
tags: ["Agent", "ReAct", "Agent Loop", "LLM"]
categories: ["Agent"]
series: ["Agent"]
translationKey: "Agent Loop"
---

## Key Takeaways

An Agent is more than a model plus tools. What gives it autonomy is the Agent Loop that connects the two: a task is completed through multiple rounds of thought, action, and observation until the goal is reached. Common loop architectures include ReAct, Plan-and-Execute, ReWOO, Reflection/Reflexion, and multi-agent orchestration.

Mainstream frameworks and products have converged on ReAct-style loops, not because ReAct is the only correct answer, but because it offers the best balance of simplicity, generality, interpretability, and ecosystem fit. For long tasks, cost-sensitive scenarios, and quality-sensitive scenarios, other architectures are worth considering.

## What Is an Agent Loop

LangChain defines an Agent directly: an Agent is a model calling tools in a loop until a given task is complete [2](https://docs.langchain.com/oss/python/langchain/agents). LlamaIndex goes further: an Agent is a semi-autonomous program driven by an LLM that executes a series of steps toward a task and, after each step, decides whether the task is complete or loops back to the start [3](https://developers.llamaindex.ai/python/framework/understanding/agent/). Anthropic calls such systems agentic systems and emphasizes that the defining feature of an Agent is that the LLM dynamically directs its own process and tool usage [4](https://www.anthropic.com/research/building-effective-agents). AWS also describes an Agent as a system that uses model reasoning and tool calls to complete tasks [20](https://aws.amazon.com/what-is/ai-agents/).

Across these definitions, an Agent Loop usually contains the following components:

- Goal and instructions: the task and constraints given by the user or system.
- State and context: conversation history, tool results, short-term memory, and long-term memory.
- Reasoning unit: the LLM, which decides the next step based on the current state.
- Action space: tools, APIs, code executors, sub-agents, and more.
- Observation and feedback: the result of an action, written back into context.
- Termination condition: task completion, maximum rounds reached, or human intervention.

The loop can be expressed as pseudocode:

```python
while not done:
    thought = llm(context)        # Reason: understand the state and decide the next step
    action = parse(thought)       # Parse the concrete action
    observation = execute(action) # Execute the tool and get the result
    context += observation        # Write the observation back into the state
    done = check(context)         # Check whether the task is done
```

Consider a concrete example: a user says, "Book a flight from Shanghai to Beijing next Tuesday." The Agent does not do everything in a single model call. It first calls a search tool to get a list of flights, filters them by price and time, calls a booking tool to place the order, and finally confirms the order and reports back. Any step can surface a new problem, such as no direct flight, an over-budget price, or a login requirement. The Agent Loop is the skeleton that supports this repeated cycle of discovering a problem, taking an action, and examining the result.

Without this loop, an LLM is only a one-shot generator: it cannot verify facts, interact with the environment, or correct itself based on errors. The value of the Agent Loop lies in the feedback loop it creates: reasoning produces actions, actions produce observations, and observations drive the next round of reasoning [5](https://lilianweng.github.io/posts/2023-06-23-agent/)[12](https://arxiv.org/abs/2309.07864).

Note that an Agent Loop is not a license for the model to act freely. Production systems usually add maximum iteration counts, token budgets, permission boundaries, human approval, and audit logs. Anthropic explicitly distinguishes workflows from agents and recommends that most teams start with a predictable workflow and only move to an agent when model-driven decisions are needed [4](https://www.anthropic.com/research/building-effective-agents). The loop is the skeleton; constraints are what make it safe to run in production.

## Common Agent Loop Architectures

Before AI Agents existed, the way we used an LLM to complete a task was to ask it, follow its answer, copy the result back, ask again, and repeat until the task was done. An Agent Loop simulates this behavior. Different Agent Loops simplify this behavior differently for different scenarios, and the most common one is ReAct.

### ReAct: Interleaving Reasoning and Acting

ReAct (Reasoning + Acting) was proposed by Yao et al. in 2023 [[1](https://arxiv.org/abs/2210.03629)]. Its core idea is to let the LLM alternate between generating Thoughts, Actions, and Observations, writing the reasoning trace and external interactions into the same context. Simon Willison once demonstrated a minimal Python implementation of a few dozen lines, showing that the loop itself is very simple [21](https://til.simonwillison.net/llms/python-react-pattern). A typical ReAct trace:

```text
Thought: The user wants to know the weather in Beijing, so I need to query a weather API.
Action: search("Beijing weather")
Observation: Beijing, sunny, 26°C
Thought: I have the weather information and can compose an answer.
Action: finish("Beijing is sunny today, 26°C.")
```

The ReAct paper validated this pattern on question answering, fact verification, and interactive decision-making tasks. Compared with pure chain-of-thought (CoT), ReAct reduces hallucination and error propagation by interacting with external tools; compared with acting without reasoning, ReAct plans better and handles exceptions better, while its reasoning process is more interpretable [1](https://arxiv.org/abs/2210.03629).

Specifically, beyond knowledge-intensive tasks such as HotpotQA and FEVER, ReAct was also evaluated in interactive decision-making environments such as AlfWorld and WebShop: the reasoning trace helps the model track plans and handle exceptions, while actions let the model actively gather external information [1](https://arxiv.org/abs/2210.03629).

Advantages: it is simple to implement, works across tasks, adapts at every step to the latest observation, and the thought process is naturally auditable. Disadvantages: each round requires an LLM call, so token and latency costs are high; once an observation is misinterpreted, errors accumulate along the trace.

### Plan-and-Execute: Plan First, Then Execute

Plan-and-Execute separates planning and execution into two roles: the planner first generates a complete plan, the executor runs it step by step, and when a problem appears during execution, the planner replans. The Plan-and-Solve paper is an early representative of this idea [8](https://arxiv.org/abs/2305.04091), and LangChain also offers a corresponding Plan-and-Execute agent pattern.

AutoGPT and BabyAGI belong to this family as well: they decompose a goal into a task list, execute one task per round, and update the list based on results [13](https://github.com/Significant-Gravitas/AutoGPT)[14](https://github.com/yoheinakajima/babyagi).

A typical flow: the planner decomposes "research three cloud vendors and produce a comparison report" into steps such as "read each vendor's product documentation," "organize pricing and limitations," and "generate conclusions"; the executor completes them in order. If a step finds that a document does not exist, the executor sends the problem back to the planner for re-decomposition.

Advantages: it is more stable for long tasks and concentrates token usage in the planning phase, making execution cheaper. Disadvantages: a plan becomes stale once it diverges from reality, so it needs frequent replanning during execution and is less dynamic than ReAct.

### ReWOO: Decoupling Reasoning from Observations

ReWOO (Reasoning WithOut Observation) observes that a ReAct-style loop, which pauses reasoning to query a tool and then resumes, produces many repeated prompts. It splits the process into a Planner, Worker, and Solver: the Planner generates a plan with placeholders in one pass, the Worker executes tool calls in batch, and the Solver reads all results and produces the final answer [6](https://arxiv.org/abs/2305.18323).

The paper reports that on HotpotQA, ReWOO achieves roughly 5x token efficiency and about a 4% accuracy improvement, and is more robust to tool failures [6](https://arxiv.org/abs/2305.18323). The trade-off is that the intermediate process cannot dynamically revise the plan based on tool results, so ReWOO fits scenarios where tool results are relatively stable and cost matters.

For example, the Planner first generates a placeholder plan such as `query_weather(city={city})`; the Worker replaces `{city}` with concrete cities and executes them in batch; the Solver then answers based on all returned results instead of interrupting reasoning for every city.

### Reflection / Reflexion: Self-Correction Loops

Self-Refine proposes an iterative generate, self-feedback, regenerate flow that lets the same model repeatedly evaluate and improve its own output [9](https://arxiv.org/abs/2303.17651). Reflexion goes further: it writes reflective text into episodic memory and accumulates experience across rounds, which is equivalent to reinforcement learning through language [7](https://arxiv.org/abs/2303.11366). The paper reports that Reflexion reaches 91% pass@1 on HumanEval, beating GPT-4's 80% at the time [7](https://arxiv.org/abs/2303.11366).

This family of loops suits code generation, mathematical reasoning, writing, and other tasks where quality matters more than speed. The downside is also clear: extra calls add up, and the model may over-correct and break a previously correct output.

A concrete example is code generation: the Agent first generates code and runs tests, converts the failure information into a reflection such as "this does not handle empty input," and rewrites the code with that reflection in the next round, instead of starting from scratch.

### Other Variants

- Tree of Thoughts (ToT): instead of reasoning along a single path, it maintains multiple thoughts like a search tree and uses breadth-first or depth-first search to choose the next step; it suits problems that require exploration [10](https://arxiv.org/abs/2305.10601).
- Multi-agent loops: AutoGen lets multiple agents collaborate through conversation [18](https://microsoft.github.io/autogen/0.2/), CrewAI defines teams with roles and tasks [19](https://docs.crewai.com/), and LangGraph explicitly orchestrates supervisors, sub-agents, and conditional routing with a graph [17](https://docs.langchain.com/oss/python/langgraph/overview).
- CoALA: it unifies memory, action space, and decision cycles into a cognitive architecture, giving agents a more systematic analytical framework [11](https://arxiv.org/abs/2309.02427).

### Architecture Comparison

| Architecture | Core Idea | Advantages | Disadvantages | Typical Examples |
| --- | --- | --- | --- | --- |
| ReAct | Interleaved Thought/Action/Observation | Simple, dynamic, interpretable | High token use, error accumulation | LangChain, LlamaIndex, OpenAI function calling |
| Plan-and-Execute | Plan first, then execute | Stable for long tasks, saves tokens | Plans become stale | Plan-and-Solve, BabyAGI |
| ReWOO | Decouple reasoning from observation | 5x token efficiency, robust to tool failure | Less dynamic | ReWOO paper |
| Reflexion | Reflection plus memory | Self-correction, high quality | Many calls, possible over-correction | Self-Refine, Reflexion |
| Multi-agent | Multi-role conversation/orchestration | Division of labor, parallelism | High complexity and cost | AutoGen, CrewAI, LangGraph |

## Why Mainstream Agents Choose ReAct-Style Loops

First, define the observation: most mainstream Agents today use ReAct-style loops. OpenAI's Responses API and Agents SDK put tool calls into a loop: the model decides which tool to call, gets the result, and continues [15](https://openai.com/index/new-tools-for-building-agents/)[16](https://openai.github.io/openai-agents-python/). Anthropic's recommended agent pattern is likewise a model plus tools plus a loop [4](https://www.anthropic.com/research/building-effective-agents). LangChain's create_agent defines an Agent as a model calling tools in a loop [2](https://docs.langchain.com/oss/python/langchain/agents). LlamaIndex's FunctionAgent is also a function-calling loop [3](https://developers.llamaindex.ai/python/framework/understanding/agent/). AutoGPT and BabyAGI put the decide-after-every-observation pattern into longer task loops [13](https://github.com/Significant-Gravitas/AutoGPT)[14](https://github.com/yoheinakajima/babyagi).

Why does everyone choose it? I think the reasons can be grouped into five points:

1. It is simple and requires no additional training. ReAct is a prompt-level pattern: any model that can follow instructions can run it, with no reward model training or model architecture changes [1](https://arxiv.org/abs/2210.03629).
2. It is task-agnostic. It does not presuppose a task structure; every decision is based on the latest observation, so it suits open-ended tasks where branches cannot be enumerated in advance.
3. It is interpretable and observable. Thoughts provide the reasoning basis, Actions provide concrete moves, and logging, debugging, and auditing are easy; the paper also emphasizes that it is more interpretable than pure reasoning or pure acting [1](https://arxiv.org/abs/2210.03629).
4. It fits model capabilities and the ecosystem. Today's models are broadly trained for instruction following and tool calling. Function calling is essentially a structured ReAct: decide which function to call, get the result, and continue. Frameworks are therefore naturally built around this loop [2](https://docs.langchain.com/oss/python/langchain/agents)[15](https://openai.com/index/new-tools-for-building-agents/)[16](https://openai.github.io/openai-agents-python/).
5. It offers the best balance of cost and effect. Anthropic observed in engineering practice that the most successful implementations are often simple, composable patterns rather than complex frameworks [4](https://www.anthropic.com/research/building-effective-agents); ReAct is the simplest composable loop.

One reason that is easy to overlook: ReAct appeared around the same time that tool-calling capabilities became widespread. Once models natively supported function calling, frameworks only needed to map Thought/Action/Observation to choose a tool, call a tool, receive a result, and a usable Agent was almost ready with little extra engineering. ReAct is therefore not only a method from a paper; it has become the default mental model of the Agent loop across the ecosystem [15](https://openai.com/index/new-tools-for-building-agents/)[16](https://openai.github.io/openai-agents-python/).

Two caveats are needed, otherwise the conclusion would be too absolute.

First, many production systems are not strict ReAct. They may not explicitly output a Thought and instead directly output a function call, or they may hide the loop inside the API and only expose a tool list. Strict ReAct is better understood as the conceptual prototype of such tool-calling loops, not a text format that must be copied.

Second, ReAct is not optimal in every scenario. For complex but stable tasks, Plan-and-Execute saves tokens and is more controllable; when tool calls are dense and reasoning can be done in one pass, ReWOO significantly reduces cost; when final quality matters, Reflexion's self-correction is more valuable; when multiple roles must collaborate, multi-agent orchestration beats a single ReAct loop [6](https://arxiv.org/abs/2305.18323)[7](https://arxiv.org/abs/2303.11366)[8](https://arxiv.org/abs/2305.04091)[17](https://docs.langchain.com/oss/python/langgraph/overview)[18](https://microsoft.github.io/autogen/0.2/).

So a more accurate statement is: mainstream Agents choose ReAct-style loops by default because ReAct closes the reasoning-action-observation loop with minimal machinery, and other architectures optimize for specific constraints on top of it.

Going further: if the goal and steps are fully determined from the start, the task is often better built as a workflow than as an Agent. The value of an Agent comes from uncertainty, and ReAct is the lightest way to handle uncertainty. That is why it is the default for general Agents that need dynamic decisions, rather than something forced into every system [4](https://www.anthropic.com/research/building-effective-agents).

## Common Misconceptions

1. An Agent equals one LLM call plus tools. In fact, without a loop there is no Agent; a single call is only tool-augmented question answering.
2. ReAct must explicitly output a Thought. Strict ReAct requires it, but production implementations often hide the thought and directly output function calls, with similar results.
3. The more complex the architecture, the better. Anthropic's experience is to start with simple, composable patterns; complex frameworks add risk [4](https://www.anthropic.com/research/building-effective-agents). Choose the right loop first, then talk about complexity.

## Selection Recommendations

| Scenario | Recommended Architecture | Reason |
| --- | --- | --- |
| General assistant, open-ended Q&A | ReAct-style | Simple, dynamic, interpretable |
| Long workflows with clear steps | Plan-and-Execute | Stable, saves tokens |
| Tool-dense, cost-sensitive | ReWOO | High token efficiency |
| Code/reasoning/writing quality matters | Reflexion | Self-correction |
| Complex multi-role collaboration | Multi-agent | Division of labor and parallelism |

## Summary

The Agent Loop is the true core of an Agent: it turns an LLM from a generator into an executor and turns a single inference into a closed loop that continuously interacts with the environment. ReAct became the mainstream default because it offers the best overall mix of simplicity, generality, interpretability, and ecosystem fit, not because it is the only option. Understanding the trade-offs among different architectures is how you make the right choice for a specific scenario.

## References

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
