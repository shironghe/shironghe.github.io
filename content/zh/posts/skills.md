---
title: "Skills：给 Agent 装上一套可复用的操作说明书"
description: "从运行时发现、按需加载到评测迭代，系统解释 Agent Skill 的工作原理，并总结如何编写可复用、可验证、可维护的 SKILL.md。"
date: 2026-08-28
draft: false
tags: ["Agent", "Skill", "Prompt Engineering"]
translationKey: "Skills"
---

当 Agent 第一次完成一项任务时，我们往往会在对话中补充大量背景：项目约定、工具用法、检查清单、异常处理和输出格式。如果这些信息只存在于一次对话里，下一次任务仍然要重新解释，Agent 也很容易漏掉关键步骤。

Skill 把一类任务的可复用知识封装成一个能力包。它通常包含一个 `SKILL.md` 入口文件，以及按需读取的参考资料、示例、模板和脚本。Skill 不是一个新模型，也不是一句更长的提示词；它是一种让 Agent 在需要时取得专业操作说明的组织方式。

本文以 Claude Platform 的 Agent Skills 最佳实践为主要依据，结合 Codex 等 Agent 平台的共同语义，回答三个问题：Skill 如何被发现和加载？`SKILL.md` 应该怎样编写？怎样通过真实任务和评测让 Skill 持续变好？[1]

## 一、先理解 Skill 的工作原理

### 1. Skill 不是工具，而是工具的使用说明

Skill、Prompt、Workflow 和 Agent 解决的问题不同：

| 概念 | 主要职责 | 典型内容 |
| --- | --- | --- |
| Prompt（提示词） | 指导一次模型响应 | 背景、问题、输出要求 |
| Workflow（工作流） | 编排确定的步骤 | 函数、状态机、固定分支 |
| Skill | 封装一类任务的方法 | 规则、步骤、资源、脚本说明 |
| Agent | 根据任务和环境动态行动 | 选择 Skill、调用工具、解释结果 |

可以把 Agent 想成一名会临场判断的工程师，把 Skill 想成工具箱中的专业手册。手册告诉工程师如何完成某类工作，但不会替工程师自动执行每一步；Agent 仍然需要根据任务、当前状态和工具结果作出判断。

Skill 可以指导 Agent 调用脚本或 MCP 工具，但它本身不是工具服务器。比如，链接检查 Skill 可以规定何时运行 `scripts/check_links.py`、传入什么参数以及如何解释输出；真正发起网络请求的是脚本，真正决定是否调用脚本的是 Agent。

### 2. 元数据先于正文

Skill 的关键设计是渐进式披露（progressive disclosure）：先让模型知道“有哪些能力”，只有任务确实需要时，才读取某个能力的完整说明。

在 Claude 的公开运行时模型中，启动时所有 Skill 的 `name` 和 `description` 会预加载到 system prompt。模型根据用户任务判断某个 Skill 是否相关；如果需要，Claude 再使用文件读取能力按需读取 `SKILL.md` 和其他资源。脚本可以直接执行，不必先把完整源码读入上下文，通常只将执行结果返回给模型[1]。

```text
启动：加载所有 Skill 的 name/description
    ↓
用户任务：LLM 判断是否需要某个 Skill
    ↓
读取：按需读取该 Skill 的 SKILL.md
    ↓
导航：根据 SKILL.md 决定读取哪些资源
    ↓
执行：调用脚本或其他工具
    ↓
验证：检查结果并继续、修正或报告失败
```

这里有一个容易混淆的细节：LLM 的内部判断对宿主不可见，宿主只有在判断被转化为可观察事件后，才能知道 Skill 正在被使用。常见衔接方式有三种：

1. **文件读取协议**：LLM 请求 `read("translate/SKILL.md")`，宿主从路径知道要加载哪个 Skill。
2. **专用加载协议**：LLM 调用 `load_skill("translate")`，宿主校验名称和权限后加载正文。
3. **宿主路由协议**：宿主先使用规则或分类模型选择 Skill，再把正文注入 Agent 上下文。

公开文档能够支撑“名称和描述用于发现、正文和资源按需加载”这一语义，但具体宿主使用 `read`、bash、专用工具还是内部注入，属于实现细节。

### 不同 Agent 宿主的加载伪代码

下面的代码不是三个项目的源码，而是依据它们公开文档抽象出的加载流程。三者共同遵循“启动时发现轻量元数据，任务相关时加载正文”的原则，但目录、显式调用命令和工具接口并不相同。

#### Claude Code：目录发现 + 自动触发或斜杠命令

Claude Code 从企业、个人、项目、插件和嵌套项目目录发现 Skill；项目 Skill 通常位于 `.claude/skills/<skill-name>/SKILL.md`，个人 Skill 位于 `~/.claude/skills/<skill-name>/SKILL.md`。它可以根据任务自动调用，也可以由用户输入 `/skill-name` 显式调用。支持文件在 Skill 正文需要时再读取；正文中的动态上下文表达式还可能在模型看到内容前由 Claude Code 执行并替换[3]。

```python
def claude_code_start(project, user_home):
    locations = discover_claude_code_locations(project, user_home)
    skills = scan_skill_directories(locations)  # 只解析 name/description
    system_prompt.add_skill_index(skills)


def claude_code_turn(task, system_prompt):
    requested = parse_slash_command(task)  # 例如 /code-review
    skill = requested or model_selects_from_metadata(task, system_prompt.skill_index)
    if skill is None:
        return run_without_skill(task)

    body = read_file(skill.path / "SKILL.md")
    body = resolve_dynamic_context(body)    # 仅在 Claude Code 支持该语法时
    context.add(body)
    return agent_loop(task, context, skill.supporting_files)
```

这里的 `model_selects_from_metadata` 表示模型根据 `description` 判断相关性，不表示 Claude Code 内部一定存在同名函数。`read_file`、动态上下文和实际工具调用由 Claude Code 运行时负责。

#### pi：扫描候选目录 + XML 元数据 + `read` 按需加载

pi 的公开文档描述得更直接：启动时扫描全局、项目、包、配置和命令行指定的 Skill 位置，提取名称与描述，以 XML 形式加入 system prompt；任务匹配后，Agent 使用 `read` 读取完整 `SKILL.md`，也可以使用 `/skill:name` 强制加载。pi 还明确提醒，模型并不总是会主动读取匹配的 Skill，显式命令或更强的提示可以强制它加载[4]。

```python
def pi_start(cwd, settings, cli_paths):
    locations = [
        "~/.pi/agent/skills",
        "~/.agents/skills",
        *ancestors(cwd, names=[".pi/skills", ".agents/skills"]),
        *settings.skill_paths,
        *cli_paths,
    ]
    skills = discover_and_validate(locations)
    system_prompt.add_xml("<available_skills>")
    for skill in skills:
        system_prompt.add_xml(render_xml_metadata(skill))
    system_prompt.add_xml("</available_skills>")


def pi_turn(task, tools):
    explicit = parse_command(task, prefix="/skill:")
    if explicit:
        skill = resolve_skill(explicit)
        content = tools.read(skill.path / "SKILL.md")
    else:
        skill = model_decides_relevant_skill(task)
        content = tools.read(skill.path / "SKILL.md") if skill else None

    if content:
        context.add(content)
    return agent_loop(task, tools)
```

pi 默认提供 `read`、`write`、`edit` 和 `bash` 等工具。Skill 只负责告诉 Agent 如何使用这些工具；Skill 的脚本仍然通过 `bash` 或其他工具执行，而不是变成新的 Skill 选择器。

#### Hermes：Skill 作为程序性记忆，并支持 Agent 管理

Hermes 将 Skill 定位为按需加载的程序性知识，并提供 `/skills` 浏览、`/<skill-name>` 调用以及 `skill_manage` 管理能力。官方文档还说明，Agent 可以在复杂任务、遇到错误或被用户纠正后创建或更新 Skill；Skill 与持久记忆共同构成其学习闭环[5]。

```python
def hermes_start(config):
    skills = scan_hermes_skill_sources(config)
    system_prompt.add_skill_index([
        {"name": s.name, "description": s.description}
        for s in skills
    ])


def hermes_turn(task, tools):
    explicit = parse_skill_command(task)  # /skills 或 /<skill-name>
    skill = explicit or model_selects_relevant_skill(task)
    if skill:
        skill_doc = tools.read_file(skill.skill_md)
        context.add(skill_doc)

    result = agent_loop(task, tools)

    if should_persist_procedural_knowledge(result):
        tools.skill_manage(
            action="create_or_patch",
            name=derive_skill_name(result),
            content=extract_reusable_workflow(result),
        )
    return result
```

这段 Hermes 伪代码需要特别谨慎地理解：`skill_manage` 表示 Hermes 的 Agent 管理 Skill 能力，不代表所有 Skill 系统都允许 Agent 自动写入 Skill。生产环境仍应增加内容校验、版本控制和人工审核；“能创建 Skill”和“允许修改线上 Skill”是两个不同的权限。

#### 三种实现的共同抽象

| 阶段 | Claude Code | pi | Hermes |
| --- | --- | --- | --- |
| 发现 | 扫描 Claude Code Skill 目录 | 扫描全局、项目、包和配置目录 | 扫描 Skill 来源并建立索引 |
| 元数据 | 提供给 Claude 做自动触发 | 以 XML 形式加入 system prompt | 提供 Skill 列表供 Agent 使用 |
| 显式入口 | `/skill-name` | `/skill:name` | `/skills`、`/<skill-name>` |
| 正文加载 | 运行时加载 `SKILL.md` | Agent 用 `read` 按需读取 | Agent 按需读取 Skill 文档 |
| 资源执行 | 工具或脚本 | `bash` 等工具 | Hermes 工具和脚本 |
| 自我修改 | 通过文件变更或工作流更新 | Skill 文件由外部维护 | 可通过 `skill_manage` 创建或修改 |

表中的“共同抽象”用于帮助理解，不意味着三个项目共享同一个 API 或实现。编写跨宿主 Skill 时，应以 Agent Skills 标准为底线，再为目标宿主补充目录、命令、工具和权限说明。

### 3. 选择 Skill 和选择工具是两件事

两种选择发生在不同阶段：

| 选择对象 | 发生时间 | 决策依据 |
| --- | --- | --- |
| 选择 Skill | 读取 Skill 正文之前 | 用户任务与 `description` 的相关性 |
| 选择资源 | 读取 `SKILL.md` 之后 | 正文中的分支条件和任务上下文 |
| 选择工具 | 执行任务过程中 | instructions、当前状态和工具结果 |

例如，用户说“提取这个 PDF 的表格并保存为 Excel”：

```text
LLM 根据 description 选择 pdf-processing
    ↓
read/bash 读取 pdf-processing/SKILL.md
    ↓
LLM 发现这是表格提取任务，继续读取 reference/table-extraction.md
    ↓
执行 scripts/extract_tables.py
    ↓
验证中间结果，再生成 Excel
```

`SKILL.md` 不是一个被动的知识库文件。它应当明确告诉 Agent 什么时候读取哪个文件、什么时候执行哪个脚本，以及每一步如何判断成功。

## 二、Skill 的目录和入口文件

### 1. 一个典型的目录

```text
pdf-processing/
├── SKILL.md                       # 入口：范围、步骤、资源索引
├── references/                    # 按需阅读的详细资料
│   ├── table-extraction.md
│   └── scanned-pdf.md
├── scripts/                       # 可直接执行的确定性工具
│   ├── extract_tables.py
│   └── validate_output.py
└── assets/                        # 模板、样例或测试输入
    └── output-template.xlsx
```

入口文件应像一份目录和操作指南，而不是把所有资料复制进去。Claude 的最佳实践建议将 `SKILL.md` 正文控制在 500 行以内；更长的内容应拆到参考文件中。引用关系尽量保持一层，文件名应表达内容，例如 `reference/table-extraction.md`，不要使用 `docs/file2.md` 这样的模糊命名[1]。

### 2. front matter 负责发现

一个最小入口文件如下：

```markdown
---
name: pdf-processing
description: Extract text, tables, and fields from PDF files. Use when the user asks to process, inspect, or transform a PDF.
---

# PDF processing

## Workflow

1. Identify whether the PDF contains text, tables, or scanned pages.
2. Read `references/table-extraction.md` for table tasks.
3. Run `scripts/validate_output.py` before returning a generated file.

## Completion

Report the number of processed pages and verify that the output file exists and can be opened.
```

`name` 和 `description` 是发现阶段的路由信息。`description` 至少应回答两个问题：这个 Skill 做什么？什么任务应该使用它？它不需要包含全部步骤、工具参数和背景知识，否则会让所有 Skill 的元数据都变得臃肿，降低区分度。

在 Claude 的实现中，front matter 还有明确约束：`name` 使用小写字母、数字和连字符，最长 64 个字符；`description` 非空，最长 1024 个字符，且不能包含 XML 标签[1]。其他平台的字段限制可能不同，实际项目应以目标宿主的规范为准。

### 3. 正文负责执行

正文应该包含：

- 任务目标和适用边界。
- 前置条件和输入要求。
- 按顺序排列的工作步骤。
- 分支条件和工具调用方式。
- 失败处理与人工接管条件。
- 输出格式和完成标准。
- 参考资料、脚本和模板的加载或执行条件。

入口文件中所有步骤都应使用能观察到的动作。与“认真检查 PDF”相比，“统计总页数，逐页提取文本，并对输出运行校验脚本”更适合作为 Agent 指令。

## 三、如何编写可靠的 Skill

### 1. 从真实任务和失败开始

不要一开始就凭想象写一份很长的 Skill。Claude 官方推荐评测驱动的开发流程：先在没有 Skill 的情况下运行代表性任务，记录具体失败或缺失的上下文，再建立评测，最后只写足以解决这些问题的最小 instructions[1]。

推荐流程如下：

```text
没有 Skill 执行真实任务
    ↓
记录具体失败和缺失信息
    ↓
至少建立三个评测场景
    ↓
测量无 Skill 基线
    ↓
编写最小 Skill
    ↓
比较结果并迭代
```

三个场景不应只是三个成功样例。至少应覆盖一个主要用例、一个边界输入和一个失败恢复路径。例如 PDF Skill 可以测试普通文本 PDF、扫描 PDF 和存在缺失字段的表单 PDF。

### 2. 先定义边界，再写步骤

一个 Skill 应明确什么属于它、什么不属于它：

- 链接检查 Skill 可以验证 URL 是否可访问，但不负责判断网页内容是否正确。
- 代码审查 Skill 可以报告问题，但不应未经请求修改代码。
- PDF 提取 Skill 可以读取和转换文件，但不应擅自发布提取出的敏感数据。

边界不仅减少误触发，也能防止 Agent 在任务过程中越权。对于相邻能力，应该在 `description` 中使用有区分度的关键词和使用场景，而不是写成“处理所有文档”这样的捕获式描述。

### 3. 用工作流写法组织步骤

复杂任务可以用“分析—计划—执行—验证”的结构：

```text
分析输入 → 选择策略 → 生成计划 → 验证计划
                                  ↓ 失败：修正计划
                                  ↓ 通过
                           执行变更 → 验证结果
```

对于批量操作、破坏性写入和高风险任务，计划应保存为结构化中间文件，例如 `changes.json`。验证脚本先检查对象存在、字段完整、值不冲突，验证通过后再允许真正执行。这样可以提前发现错误，保留审计证据，并让 Agent 在不修改原始数据的情况下反复修正计划[1]。

### 4. 把确定性工作交给脚本

脚本应当解决问题，而不是把问题原样推回给 Agent。解析文件、计算数值、批量重命名、格式校验和结果验证，都适合交给脚本。

在 Skill 中要明确区分两种意图：

```markdown
## Utility scripts

Run `scripts/validate_output.py result.json` to validate the generated result.
Read `references/algorithm.md` only when you need to explain the extraction algorithm.
```

通常，确定性工具应该直接执行，而不是先读取源码再让 LLM 重写一遍。脚本输出应包含具体错误，例如“字段 `signature_date` 不存在，可用字段为……”，而不是只返回 `failed`。超时、重试次数和其他参数也应有明确理由，避免出现无法解释的“魔法数字”[1]。

如果 Skill 使用 MCP（Model Context Protocol）工具，应使用完整的服务器名和工具名，例如 `GitHub:create_issue`，不要只写 `create_issue`。多个服务器存在同名工具时，完整名称可以减少解析歧义[1]。

### 5. 让资源容易被发现

文件系统本身就是 Agent 的导航界面。路径使用正斜杠，例如 `scripts/validate_output.py`，不要使用 Windows 反斜杠。目录按领域或功能组织，文件名直接表达内容。

入口文件应显式告诉 Agent：

- 处理表格时读取 `references/table-extraction.md`。
- 输入是扫描件时读取 `references/scanned-pdf.md`。
- 生成结果后执行 `scripts/validate_output.py`。

如果某个资源始终被读取，它可能应该上移到 `SKILL.md`；如果某个资源从未被读取，可能是链接不明显，也可能根本不需要保留。不要用一层又一层的索引隐藏真正重要的规则。

### 6. 为视觉输入和环境依赖写清楚条件

如果输入可以渲染成图像，例如表单、版式复杂的 PDF 或流程图，Skill 应明确何时把页面转换为图片并进行视觉分析。不要假设所有运行环境都预装了包或具备网络访问能力：在 `SKILL.md` 中列出依赖、安装方式和运行限制，并区分本地环境、云端 API 或沙箱环境的差异[1]。

## 四、Skill 的失败处理与安全边界

可靠的 Skill 不能只描述成功路径。至少要覆盖以下情况：

| 情况 | Agent 应该做什么 |
| --- | --- |
| 输入不完整 | 列出缺少的信息，请用户补充 |
| 参数错误 | 根据错误信息修正后限次重试 |
| 临时网络错误 | 退避重试，记录最终状态 |
| 权限不足 | 停止操作，请求用户处理权限 |
| 验证不通过 | 返回计划或生成步骤修正 |
| 高风险写操作 | 在执行前请求确认 |
| 达到预算或步数限制 | 停止并报告已完成和未完成部分 |

失败处理应写成条件分支，而不是“遇到问题时妥善处理”。Agent 需要知道错误发生后回到哪里、最多重试几次、什么情况下必须停止。

对于脚本，优先显式处理错误：

```python
def read_input(path):
    try:
        return path.read_text(encoding="utf-8")
    except FileNotFoundError:
        raise ValueError(f"input file does not exist: {path}")
    except PermissionError:
        raise ValueError(f"input file is not readable: {path}")
```

对于高风险任务，使用中间产物、验证脚本和最终检查组成闭环：

```text
读取原始数据 → 生成 changes.json → 校验 changes.json
                                      ↓
                               失败：不执行
                                      ↓
                                 执行变更
                                      ↓
                                验证最终结果
```

## 五、如何评测和迭代 Skill

### 1. 建立可重复的评测

一次“看起来不错”的结果不能证明 Skill 有效。评测记录至少应包含：

```json
{
  "skills": ["pdf-processing"],
  "query": "提取这个 PDF 的全部文本并保存到 output.txt",
  "files": ["test-files/document.pdf"],
  "expected_behavior": [
    "使用适合的工具读取 PDF",
    "提取所有页面的文本且不遗漏页面",
    "将结果保存为可读的 output.txt"
  ]
}
```

需要分别测量：

- Skill 是否在应该使用时触发。
- 不相关任务是否避免误触发。
- 关键步骤是否执行。
- 工具调用是否正确。
- 失败后是否能够恢复。
- 输出是否通过自动验证和人工验收。
- token、耗时、工具调用次数是否可接受。

评测的“源事实”应该是任务行为和验收标准，而不是 Skill 文件的长度或看起来是否详细。Claude 文档也特别提醒，目前并没有内置的通用评测运行器，团队需要根据自己的任务建立评测系统[1]。

### 2. 使用作者 Agent 和测试 Agent

一个有效的迭代方式是让不同实例承担不同角色：

1. **作者 Agent**：根据真实任务和失败记录设计或修改 Skill。
2. **测试 Agent**：加载候选 Skill，在新的任务中执行它。
3. **观察者**：记录测试 Agent 读取了什么、调用了什么、哪里失败。
4. **维护者**：根据证据修改 Skill，再运行同一批回归测试。

使用新的测试实例很重要。作者 Agent 记得自己写规则时的意图，容易高估文档的清晰度；测试 Agent 的真实行为更能暴露遗漏。每次修改都应比较基线、旧版本和新版本，而不是只看新增成功案例[1]。

### 3. 观察 Agent 如何导航文件

Skill 的目录设计可以通过 Agent 的探索路径来验证：

- 反复读取同一文件：该内容可能应该上移到入口文件。
- 找不到关键 reference：入口中的路径或触发条件不够明确。
- 按意外顺序读取文件：目录结构或文件名不直观。
- 从不访问某个资源：资源可能无用，或缺少显式链接。
- 总是读取大量资料：渐进式披露没有真正发挥作用。

这些观察比作者主观阅读文档更有价值。Skill 需要随着真实使用不断修正，但每个修正都应对应一个可复现的失败或明确的评测收益。

## 六、Skill 的维护和受控自进化

Skill 的“自进化”不是让执行任务的 Agent 在生产环境里随意改写自己的 instructions，而是把任务反馈接入一个可审核的开发流程。Warp 在 Claude 上实践了一种简单而重要的架构：用一个内层 Base Skill 执行实际任务，用一个外层 Improver Skill 定期观察反馈并提出对 Base Skill 的小幅修改[2]。

### 1. 双 Skill 架构

Base Skill（基础 Skill）保存稳定的领域知识和执行方法。例如，在代码审查任务中，它规定如何阅读变更、哪些问题值得评论、如何遵循代码库约定，并据此生成审查结果。

Improver Skill（改进 Skill）不负责每一次业务任务，而是作为观察者 Agent 按计划运行。它收集已经积累的人类反馈，比较“Agent 提出了什么”和“人类如何评价或修正”，然后提出一份针对 Base Skill 的小而明确的修改。

```text
业务任务
    ↓
Base Skill + 任务上下文 → Agent 输出
                                ↓
                         人类在原工作位置反馈
                                ↓
                  Improver Skill 定期读取反馈
                                ↓
                   修改 Base Skill 并创建 PR
                                ↓
                    人类审核、合并或拒绝
                                ↓
                    下一次任务使用新版本
```

这里的关键是职责分离：Base Skill 追求稳定地完成任务，Improver Skill 追求从反馈中找到可复用的改进。改进 Agent 不应该在每次任务中同步修改 Base Skill，否则一次偶然反馈就可能改变后续所有任务的行为。

### 2. 反馈必须回到工作发生的地方

Warp 的案例强调，反馈之所以有价值，是因为它解决了 Agent 会话结束后上下文消失的问题。代码审查中的反馈可以是简单的赞踩，但“这个评论有帮助”不如具体说明更有用：例如指出某个重命名建议违反了代码库的全局变量命名约定，并解释正确约定是什么[2]。

一个好的反馈至少包含：

- 对哪个输出或行动不满意。
- 具体哪里错了或不够好。
- 期望的结果是什么。
- 为什么应该这样做。

反馈入口应尽量贴近原工作流，例如直接评论 Pull Request 或 GitHub Issue，并自动收集这些评论。不要要求用户额外打开表单、复制结果、重新填写上下文；反馈成本越低，持续产生有效信号的可能性越高。

### 3. Improver Skill 如何工作

Improver Skill 可以按以下步骤运行：

1. 从反馈所在系统读取最近一段时间的反馈。
2. 将反馈与对应的 Agent 输出、任务上下文和 Base Skill 版本关联起来。
3. 区分明确的领域反馈与模糊的偏好表达。
4. 判断问题是 Base Skill 缺少规则、规则位置不明显、工具失败，还是任务本身特殊。
5. 提出最小、聚焦的 Base Skill 修改。
6. 生成修改说明、证据和预期影响。
7. 通过 PR 或等价的审核流程交给人类决定。

Warp 的 Issue triage 案例中，Improver Skill 通过定时 Agent 读取带有反馈的 GitHub Issue，使用 Skill 附带的 Python 脚本拉取并汇总数据为 JSON，再根据维护者评论修改 triage Skill，最后创建 PR。维护者审核并合并后，下一次 triage 才会获得新的规则[2]。

这个例子展示了一个实用原则：把拉取反馈、整理数据等确定性工作封装为 Skill 自带脚本；把“哪些反馈值得沉淀、应该如何最小修改”留给 Agent；把最终是否生效留给正常的代码评审流程。

### 4. 写原则，而不是堆规则

自进化 Skill 的 instructions 不应试图穷举所有输入。Warp 的经验是，用原则指导智能 Agent，通常比编写大量僵硬的例外规则更容易泛化。例如：

```text
优先关注重复出现的代码模式，并说明它为什么会增加维护成本。
```

通常比下面的规则集合更有适应性：

```text
变量名以 x 开头时执行规则 A；
变量名以 y 开头且位于第 3 行时执行规则 B；
变量名以 z 开头但文件名不是 test 时执行规则 C
```

原则也要解释“为什么”。原因能帮助 Agent 在新情境中进行类比和判断，而不是机械套用只在旧样例上成立的条件。改进 Skill 应优先增加能泛化的判断依据，再考虑是否需要加入具体示例。

### 5. 反馈质量比反馈数量更重要

大量简单的赞踩可以提供方向，但无法说明怎样改。少量来自领域专家、包含具体原因的反馈，往往比大量没有解释的二元反馈更有价值。另一方面，随着系统覆盖更多任务，质量稳定的反馈样本越多，Improver Skill 越能区分偶然事件和可复用模式[2]。

因此，反馈处理不应简单地把所有用户意见投票后写入 Skill。应保留反馈来源和上下文，对冲突意见进行归因，并根据任务风险决定是否需要领域专家复核。错误反馈是必然存在的，Improver Skill 需要有能力质疑反馈，而不是盲目接受。

### 6. 用版本控制和人工审核形成门禁

Skill 是普通文件，因此它的修改可以进入正常的版本控制流程。每个候选修改至少应说明：

```text
修改对象：triage/SKILL.md
修改原因：真实问题 Issue 没有被标记为 ready-to-spec
反馈证据：维护者在 Issue 中明确指出遗漏及其判断标准
最小修改：补充 ready-to-spec 的适用条件
影响范围：只改变 triage 标签判断
验证方式：回放历史 Issue，并检查原有标签规则未退化
```

人类审核不是形式上的最后一步，而是自进化环节的安全边界。审核者应检查反馈是否可靠、修改是否足够小、是否引入冲突、是否改变了权限或高风险行为，并决定合并、要求修改或拒绝。没有通过审核的候选补丁不能影响线上 Base Skill。

### 7. 从一个 Agent 到一组 Agent

当系统中只有一个 Agent 时，可以为它维护一套 Base Skill 和一套 Improver Skill；当 Agent 数量增加时，可以在两种策略之间取平衡：领域差异明显的 Agent 分别维护自己的改进器，共享机制和通用流程；行为相近的 Agent 则共享一个模板化 Improver Skill，再叠加领域特定的配置。

Warp 将同一模式用于代码库中的 triage、spec 编写和代码审查 Agent。可复用的不是某条领域规则，而是“读取反馈—理解原因—提出小修改—进入审核”的改进机制[2]。

### 8. 如何判断系统真的变好了

不要只观察 Base Skill 的局部成功案例，还要跟踪人们原本就关心的业务指标，例如：

- 代码审查评论被采纳或关闭的比例。
- Issue 从提交到进入 spec 阶段的时间。
- 人工修改 Agent 输出所花的时间。
- 任务重试率、人工接管率和成本。
- 用户反馈量与有效反馈比例。

部署可以采用 crawl-walk-run 的节奏：先离线回放历史数据，再在低风险任务上小范围运行，最后扩大范围。每次执行绑定 Base Skill 版本，保留旧版本和 PR 记录；一旦全局指标恶化或出现新的安全问题，回滚版本，而不是让 Improver Skill 在生产环境继续自动叠加修改。

总的来说，受控自进化的最小闭环不是“Agent 修改自己”，而是：

```text
Base Skill 执行
    → 人类低摩擦反馈
    → Improver Skill 定期观察
    → 提出最小、可解释的修改
    → PR/评测/人工审核
    → 合并后由新版本继承经验
```

Skill 因此成为连接运行时经验和工程化知识的文件接口：Agent 可以帮助发现和提出改进，人类和版本控制系统负责决定哪些改进真正进入生产。

## 七、常见反模式

### 反模式一：把所有知识塞进 `SKILL.md`

结果是入口文件越来越长，Agent 每次都要面对大量无关内容。应将所有分支都需要的内容留在入口，将特定分支的资料拆到 reference。

### 反模式二：用模糊描述捕获所有任务

“处理任何文档”“帮助完成各种工作”会导致误触发。description 应具体说明能力和使用时机，并写出与相邻 Skill 的区别。

### 反模式三：让脚本只抛出异常

脚本应该返回能帮助 Agent 修正问题的错误信息，而不是把诊断工作完全推回模型。配置参数也要有理由，避免无法解释的超时和重试常数。

### 反模式四：给 Agent 太多等价选项

如果多个工具都能完成同一个动作，应指定默认方案，并只在明确条件下提供备用方案。过多选择会增加决策成本，也让评测难以稳定。

### 反模式五：直接执行未经验证的批量变更

复杂操作先生成结构化计划，验证计划后再执行，最后检查最终结果。不要用“模型看起来理解了”代替机器可验证的中间结果。

## 八、检查清单

发布 Skill 前可以逐项检查：

- `description` 是否同时说明能力和使用场景？
- 是否定义了不适用的边界？
- `SKILL.md` 是否保持短小，详细内容是否合理拆分？
- 文件名和路径是否清晰、可移植？
- reference 是否从入口文件显式指向？
- 脚本是应该执行还是阅读，是否写清楚？
- 脚本是否显式处理常见错误？
- 关键操作是否有验证器或完成条件？
- 高风险操作是否有确认和回滚路径？
- 是否有至少三个覆盖正常、边界和失败的评测场景？
- 是否测量了无 Skill 基线？
- 是否用新实例测试过真实任务？
- 是否记录了版本、轨迹和失败案例？

## 总结

Skill 的核心不是一份更长的提示词，而是一套可发现、可加载、可执行、可验证的 Agent 工作流说明。

它的运行链路可以概括为：

```text
元数据发现 → LLM 选择 → 按需读取 SKILL.md
→ 读取必要资源 → 执行脚本和工具 → 验证结果
→ 记录失败 → 评测迭代 → 受控发布
```

编写 Skill 时，先从真实失败和评测开始，再写最小 instructions；组织 Skill 时，把稳定规则放在入口，把分支细节放在资源，把确定性工作交给脚本；维护 Skill 时，用版本、门禁、灰度和回滚控制自进化。这样，Skill 才能从一次性的提示词，成长为真正可复用的 Agent 能力。

## 参考文献
[1]: [Claude Platform：Skill authoring best practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)\
[2]: [Anthropic：How Warp builds self-improving agents on Claude](https://claude.com/blog/how-warp-builds-self-improving-agents-on-claude)\
[3]: [Claude Code：Extend Claude with skills](https://code.claude.com/docs/en/skills)\
[4]: [pi：Skills 文档](https://github.com/badlogic/pi-mono/blob/main/packages/coding-agent/docs/skills.md)\
[5]: [Hermes Agent：Skills System](https://hermes-agent.nousresearch.com/docs/user-guide/features/skills)
