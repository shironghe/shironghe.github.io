---
title: "Skills: Giving Agents Reusable Operating Manuals"
description: "From runtime discovery and on-demand loading to evaluation-driven iteration, this article explains how Agent Skills work and how to write reusable, verifiable, and maintainable SKILL.md files."
date: 2026-08-28
draft: false
tags: ["Agent", "Skill", "Prompt Engineering"]
translationKey: "Skills"
---

When an Agent completes a task for the first time, we often provide extensive context in the conversation: project conventions, tool usage, checklists, error handling, and output formats. If that knowledge exists only in one conversation, the next task requires the same explanation again, and the Agent can easily miss a critical step.

A Skill packages reusable knowledge for a class of tasks. It usually contains an entry file named `SKILL.md`, along with reference materials, examples, templates, and scripts loaded on demand. A Skill is not a new model or a longer prompt. It is an organizational mechanism that lets an Agent obtain specialized operating instructions when they are needed.

Using Claude Platform's Agent Skills best practices as the primary reference and combining them with common semantics found in Agent platforms such as Codex, this article answers three questions: How are Skills discovered and loaded? How should `SKILL.md` be written? How can real tasks and evaluations make a Skill better over time?[1]

## 1. Understanding How Skills Work

### 1. A Skill Is Not a Tool, but Instructions for Using Tools

Skill, Prompt, Workflow, and Agent solve different problems:

| Concept | Primary responsibility | Typical contents |
| --- | --- | --- |
| Prompt | Guide one model response | Context, question, output requirements |
| Workflow | Orchestrate fixed steps | Functions, state machines, fixed branches |
| Skill | Package the method for a class of tasks | Rules, steps, resources, script instructions |
| Agent | Act dynamically based on the task and environment | Select Skills, call tools, interpret results |

An Agent is like an engineer who can make decisions in the moment, while a Skill is like a specialized manual in the engineer's toolbox. The manual explains how to perform a class of tasks, but it does not execute every step automatically. The Agent still makes decisions based on the task, current state, and tool results.

A Skill can instruct an Agent to call scripts or MCP tools, but it is not itself a tool server. For example, a link-checking Skill can specify when to run `scripts/check_links.py`, which arguments to pass, and how to interpret the output. The script makes the network requests; the Agent decides whether to call it.

### 2. Metadata Comes Before the Body

The key design behind Skills is progressive disclosure: first tell the model which capabilities are available, and read a capability's full instructions only when the task actually needs it.

In Claude's documented runtime model, every Skill's `name` and `description` are preloaded into the system prompt at startup. The model decides whether a Skill is relevant to the user's task. If it is, Claude uses file-reading capabilities to read `SKILL.md` and other resources on demand. Scripts can be executed directly without first loading their complete source into context; typically, only their output is returned to the model[1].

```text
Startup: load every Skill's name/description
    ↓
User task: the LLM decides whether a Skill is needed
    ↓
Read: load the Skill's SKILL.md on demand
    ↓
Navigate: use SKILL.md to decide which resources to read
    ↓
Execute: call scripts or other tools
    ↓
Verify: check the result, then continue, correct, or report failure
```

There is an easily overlooked detail here: the host cannot see the LLM's internal judgment. The host knows that a Skill is being used only after that judgment becomes an observable event. Common ways to connect the two include:

1. **File-reading protocol**: the LLM requests `read("translate/SKILL.md")`, and the host infers from the path which Skill to load.
2. **Dedicated loading protocol**: the LLM calls `load_skill("translate")`, and the host validates the name and permissions before loading the body.
3. **Host-side routing protocol**: the host uses rules or a classifier model to select a Skill first, then injects its body into the Agent's context.

Public documentation supports the semantic claim that names and descriptions are used for discovery while bodies and resources are loaded on demand. Whether a particular host uses `read`, bash, a dedicated tool, or internal injection is an implementation detail.

### Loading Pseudocode for Different Agent Hosts

The following code is not source code from any of the three projects. It abstracts their loading behavior from public documentation. All three follow the principle of discovering lightweight metadata at startup and loading the body when relevant, but their directories, explicit invocation commands, and tool interfaces differ.

#### Claude Code: Directory Discovery + Automatic Invocation or Slash Commands

Claude Code discovers Skills from enterprise, personal, project, plugin, and nested-project directories. Project Skills typically live at `.claude/skills/<skill-name>/SKILL.md`, while personal Skills live at `~/.claude/skills/<skill-name>/SKILL.md`. Claude Code can invoke a Skill automatically based on the task, or the user can invoke it explicitly with `/skill-name`. Supporting files are read when the Skill needs them; dynamic context expressions in the body may also be executed and replaced by Claude Code before the model sees the content[3].

```python
def claude_code_start(project, user_home):
    locations = discover_claude_code_locations(project, user_home)
    skills = scan_skill_directories(locations)  # Parse only name/description
    system_prompt.add_skill_index(skills)


def claude_code_turn(task, system_prompt):
    requested = parse_slash_command(task)  # For example, /code-review
    skill = requested or model_selects_from_metadata(task, system_prompt.skill_index)
    if skill is None:
        return run_without_skill(task)

    body = read_file(skill.path / "SKILL.md")
    body = resolve_dynamic_context(body)    # Only when supported by Claude Code
    context.add(body)
    return agent_loop(task, context, skill.supporting_files)
```

Here, `model_selects_from_metadata` means that the model judges relevance from the `description`; it does not imply that Claude Code contains a function with this exact name. `read_file`, dynamic context, and the actual tool calls are handled by the Claude Code runtime.

#### pi: Candidate Directory Scan + XML Metadata + On-Demand `read`

pi's public documentation describes the process more directly: at startup, it scans global, project, package, settings, and command-line-specified Skill locations, extracts names and descriptions, and adds them to the system prompt in XML form. When a task matches, the Agent uses `read` to load the complete `SKILL.md`; it can also use `/skill:name` to force loading. pi explicitly warns that models do not always read a matching Skill automatically, so an explicit command or stronger prompting can force the load[4].

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

pi provides `read`, `write`, `edit`, and `bash` tools by default. A Skill only tells the Agent how to use these tools. Scripts bundled with a Skill are still executed through `bash` or another tool; they do not become a new Skill selector.

#### Hermes: Skills as Procedural Memory with Agent Management

Hermes treats Skills as on-demand procedural knowledge. It provides `/skills` for browsing, `/<skill-name>` for invocation, and `skill_manage` for management. Its official documentation also says that an Agent can create or update a Skill after a complex task, an error, or a user correction; Skills and persistent memory together form its learning loop[5].

```python
def hermes_start(config):
    skills = scan_hermes_skill_sources(config)
    system_prompt.add_skill_index([
        {"name": s.name, "description": s.description}
        for s in skills
    ])


def hermes_turn(task, tools):
    explicit = parse_skill_command(task)  # /skills or /<skill-name>
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

This Hermes pseudocode must be interpreted carefully. `skill_manage` represents Hermes's Agent-managed Skill capability; it does not mean that every Skill system allows an Agent to write Skills automatically. Production systems should still add content validation, version control, and human review. “Can create a Skill” and “is allowed to modify the online Skill” are separate permissions.

#### A Common Abstraction for the Three Implementations

| Stage | Claude Code | pi | Hermes |
| --- | --- | --- | --- |
| Discovery | Scan Claude Code Skill directories | Scan global, project, package, and settings directories | Scan Skill sources and build an index |
| Metadata | Provide metadata to Claude for automatic invocation | Add metadata to the system prompt as XML | Provide a Skill list to the Agent |
| Explicit entry | `/skill-name` | `/skill:name` | `/skills`, `/<skill-name>` |
| Body loading | Runtime loads `SKILL.md` | Agent reads `SKILL.md` on demand with `read` | Agent reads the Skill document on demand |
| Resource execution | Tools or scripts | Tools such as `bash` | Hermes tools and scripts |
| Self-modification | Update through file changes or a workflow | Skill files are maintained externally | Create or update through `skill_manage` |

This table is a common abstraction for understanding the systems; it does not mean that the projects share the same API or implementation. When writing a cross-host Skill, use the Agent Skills standard as the baseline, then add the directory, command, tool, and permission details required by the target host.

### 3. Selecting a Skill and Selecting a Tool Are Different

The two decisions occur at different stages:

| Object selected | When it happens | Decision basis |
| --- | --- | --- |
| Skill | Before reading the Skill body | Relevance between the task and `description` |
| Resource | After reading `SKILL.md` | Branch conditions and task context in the body |
| Tool | During task execution | Instructions, current state, and tool results |

For example, suppose the user says, “Extract the tables from this PDF and save them as an Excel file”:

```text
The LLM selects pdf-processing from the description
    ↓
read/bash loads pdf-processing/SKILL.md
    ↓
The LLM recognizes a table-extraction task and reads reference/table-extraction.md
    ↓
Execute scripts/extract_tables.py
    ↓
Verify the intermediate result, then generate the Excel file
```

`SKILL.md` is not a passive knowledge-base file. It should explicitly tell the Agent when to read a file, when to execute a script, and how to determine whether each step succeeded.

## 2. Skill Directories and Entry Files

### 1. A Typical Directory

```text
pdf-processing/
├── SKILL.md                       # Entry: scope, steps, resource index
├── references/                    # Detailed material read on demand
│   ├── table-extraction.md
│   └── scanned-pdf.md
├── scripts/                       # Deterministic tools that can be executed
│   ├── extract_tables.py
│   └── validate_output.py
└── assets/                        # Templates, examples, or test inputs
    └── output-template.xlsx
```

The entry file should function as a table of contents and operating guide, not as a copy of every resource. Claude's best practices recommend keeping the `SKILL.md` body under 500 lines; longer content should be split into reference files. Keep references as shallow as possible, and use descriptive names such as `reference/table-extraction.md` instead of vague paths such as `docs/file2.md`[1].

### 2. Let Front Matter Handle Discovery

A minimal entry file looks like this:

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

`name` and `description` are routing information for the discovery stage. A `description` should answer two questions: What does this Skill do? What tasks should use it? It does not need to contain every step, tool parameter, or piece of background knowledge. Otherwise, the metadata for every Skill becomes bloated and less discriminating.

In Claude's implementation, front matter also has explicit constraints: `name` uses lowercase letters, numbers, and hyphens and is at most 64 characters; `description` is non-empty, at most 1,024 characters, and cannot contain XML tags[1]. Other platforms may impose different limits, so follow the specification of the target host.

### 3. Let the Body Define Execution

The body should contain:

- The task objective and applicable boundaries.
- Preconditions and input requirements.
- Ordered work steps.
- Branch conditions and tool usage.
- Failure handling and human handoff conditions.
- Output format and completion criteria.
- Conditions for loading or executing references, scripts, and templates.

Every step in the entry file should use an observable action. Compared with “carefully inspect the PDF,” “count the pages, extract text page by page, and run the output validation script” is a better Agent instruction.

## 3. How to Write a Reliable Skill

### 1. Start with Real Tasks and Failures

Do not begin by writing a long Skill based on imagined requirements. Claude's official guidance recommends an evaluation-driven workflow: first run representative tasks without a Skill, record concrete failures or missing context, build evaluations around those gaps, and then write only the minimum instructions needed to address them[1].

The recommended process is:

```text
Run real tasks without a Skill
    ↓
Record concrete failures and missing information
    ↓
Create at least three evaluation scenarios
    ↓
Measure the no-Skill baseline
    ↓
Write the minimum Skill
    ↓
Compare results and iterate
```

The three scenarios should not all be successful examples. Cover at least one primary use case, one boundary input, and one failure-recovery path. For a PDF Skill, for example, test a normal text PDF, a scanned PDF, and a form PDF with missing fields.

### 2. Define the Boundary Before the Steps

A Skill should make clear what belongs to it and what does not:

- A link-checking Skill can verify whether a URL is reachable, but it does not determine whether the page content is correct.
- A code-review Skill can report issues, but it should not modify code without a request.
- A PDF-extraction Skill can read and transform files, but it should not publish extracted sensitive data without authorization.

Boundaries reduce false activation and prevent the Agent from exceeding its scope during execution. For neighboring capabilities, use discriminating keywords and usage scenarios in `description` rather than a catch-all description such as “process any document.”

### 3. Organize Steps as a Workflow

Complex tasks can use an “analyze—plan—execute—verify” structure:

```text
Analyze input → choose strategy → create plan → validate plan
                                               ↓ failure: correct plan
                                               ↓ pass
                                        execute changes → verify result
```

For batch operations, destructive writes, and high-stakes tasks, save the plan as a structured intermediate file such as `changes.json`. A validation script should first check that objects exist, fields are complete, and values do not conflict. Only then should the actual change be allowed. This catches errors early, preserves audit evidence, and lets the Agent revise the plan without modifying the original data[1].

### 4. Give Deterministic Work to Scripts

Scripts should solve problems rather than hand them back to the Agent. File parsing, numerical computation, batch renaming, format validation, and result verification are good candidates for scripts.

The Skill should distinguish the two intentions clearly:

```markdown
## Utility scripts

Run `scripts/validate_output.py result.json` to validate the generated result.
Read `references/algorithm.md` only when you need to explain the extraction algorithm.
```

Deterministic tools should usually be executed directly instead of having the LLM read their source and rewrite the logic. Script output should include specific errors, such as “field `signature_date` does not exist; available fields are ...,” rather than returning only `failed`. Timeouts, retry counts, and other parameters should also have explicit reasons, avoiding unexplained “magic numbers”[1].

If a Skill uses MCP (Model Context Protocol) tools, use the fully qualified server and tool name, such as `GitHub:create_issue`, rather than only `create_issue`. A full name reduces ambiguity when multiple servers expose tools with the same name[1].

### 5. Make Resources Easy to Discover

The filesystem is itself an Agent navigation interface. Use forward slashes in paths, such as `scripts/validate_output.py`, rather than Windows backslashes. Organize directories by domain or feature, and use filenames that directly describe their contents.

The entry file should explicitly tell the Agent:

- Read `references/table-extraction.md` for table tasks.
- Read `references/scanned-pdf.md` when the input is a scan.
- Run `scripts/validate_output.py` after generating the result.

If a resource is always read, it may belong in `SKILL.md`; if a resource is never read, its link may be unclear or the resource may not be needed. Do not hide important rules behind multiple layers of indexes.

### 6. State Visual and Environment Conditions Explicitly

If an input can be rendered as an image, such as a form, a layout-heavy PDF, or a diagram, the Skill should specify when to convert pages to images and perform visual analysis. Do not assume that every runtime has the required packages or network access. List dependencies, installation instructions, and runtime limitations in `SKILL.md`, and distinguish local, cloud API, and sandbox environments[1].

## 4. Skill Failure Handling and Safety Boundaries

A reliable Skill cannot describe only the success path. It should cover at least the following cases:

| Situation | What the Agent should do |
| --- | --- |
| Incomplete input | List the missing information and ask the user to provide it |
| Invalid parameters | Correct them using the error message, then retry within a limit |
| Temporary network error | Retry with backoff and record the final state |
| Insufficient permissions | Stop and ask the user to resolve the permission issue |
| Failed validation | Return to the plan or generation step and correct it |
| High-risk write operation | Ask for confirmation before execution |
| Budget or step limit reached | Stop and report completed and incomplete work |

Write failure handling as conditional branches rather than “handle problems appropriately.” The Agent needs to know where to return after an error, how many times to retry, and when it must stop.

For scripts, handle errors explicitly:

```python
def read_input(path):
    try:
        return path.read_text(encoding="utf-8")
    except FileNotFoundError:
        raise ValueError(f"input file does not exist: {path}")
    except PermissionError:
        raise ValueError(f"input file is not readable: {path}")
```

For high-risk tasks, combine an intermediate artifact, a validation script, and a final check:

```text
Read source data → generate changes.json → validate changes.json
                                             ↓
                                      failure: do not execute
                                             ↓
                                      execute changes
                                             ↓
                                      verify final result
```

## 5. Evaluating and Iterating on a Skill

### 1. Build Repeatable Evaluations

A result that “looks good” once does not prove that a Skill works. An evaluation record should contain at least:

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

Measure separately whether:

- The Skill activates when it should.
- Unrelated tasks avoid false activation.
- Critical steps are executed.
- Tool calls are correct.
- The Agent can recover from failures.
- The output passes automated validation and human acceptance.
- Token usage, latency, and tool-call count are acceptable.

The source of truth for an evaluation should be task behavior and acceptance criteria, not the length of the Skill file or how detailed it appears. Claude's documentation also notes that there is currently no built-in general-purpose evaluation runner; teams need to build evaluation systems for their own tasks[1].

### 2. Use an Author Agent and a Test Agent

An effective iteration process gives different instances different roles:

1. **Author Agent**: Designs or modifies the Skill based on real tasks and failure records.
2. **Test Agent**: Loads the candidate Skill and runs it on new tasks.
3. **Observer**: Records what the Test Agent read, what it called, and where it failed.
4. **Maintainer**: Updates the Skill based on evidence and reruns the same regression tests.

Using a fresh test instance matters. The Author Agent remembers the intent behind the rules and can overestimate how clear the document is. The Test Agent's behavior is more likely to reveal omissions. Every change should be compared against the baseline, the old version, and the new version—not only against newly successful examples[1].

### 3. Observe How the Agent Navigates Files

The directory design can be evaluated through the Agent's exploration path:

- Repeatedly reading the same file may mean that its content belongs in the entry file.
- Failing to find a critical reference may mean that the path or trigger condition is unclear.
- Reading files in an unexpected order may indicate that the directory structure or filenames are unintuitive.
- Never accessing a resource may mean that it is unused or lacks an explicit link.
- Always reading a large amount of material may mean progressive disclosure is not working.

These observations are more valuable than an author's subjective reading of the document. A Skill should evolve through real use, but every change should correspond to a reproducible failure or a clear evaluation improvement.

## 6. Skill Maintenance and Controlled Self-Evolution

Skill self-evolution does not mean letting the task-executing Agent freely rewrite its own instructions in production. It means connecting task feedback to a reviewable development process. Warp implemented an especially clear architecture on Claude: an inner Base Skill executes the real task, while an outer Improver Skill periodically observes feedback and proposes small changes to the Base Skill[2].

### 1. The Two-Skill Architecture

The **Base Skill** stores stable domain knowledge and execution methods. In a code-review task, for example, it specifies how to inspect changes, which issues are worth commenting on, and how to follow repository conventions before producing a review.

The **Improver Skill** does not handle every business task. It runs as an observer Agent on a schedule. It collects accumulated human feedback, compares what the Agent proposed with how people evaluated or corrected it, and proposes a small, focused change to the Base Skill.

```text
Business task
    ↓
Base Skill + task context → Agent output
                                ↓
                         Human feedback at the original work location
                                ↓
                  Improver Skill periodically reads the feedback
                                ↓
                   Update the Base Skill and create a PR
                                ↓
                    Human review, merge, or rejection
                                ↓
                    The next task uses the new version
```

The important idea is separation of responsibilities: the Base Skill aims to complete tasks consistently, while the Improver Skill looks for reusable improvements in feedback. The improver should not update the Base Skill synchronously during every task, because one accidental comment could change the behavior of all subsequent tasks.

### 2. Feedback Must Return to Where the Work Happens

Warp's case shows why feedback matters: it prevents context from disappearing when an Agent session ends. In code review, feedback can be as simple as a thumbs-up or thumbs-down, but “this comment was useful” is less valuable than a specific explanation—for example, that a rename suggestion violates the repository's naming convention for global variables, together with an explanation of the correct convention[2].

Useful feedback should include at least:

- Which output or action was unsatisfactory.
- What exactly was wrong or insufficient.
- What result was expected.
- Why the expected approach is correct.

The feedback entry point should be close to the original workflow, such as a direct comment on a Pull Request or GitHub Issue, and those comments should be collected automatically. Do not require users to open another form, copy the result, or re-enter the context. Lower feedback friction makes a continuous stream of useful signals more likely.

### 3. How an Improver Skill Works

An Improver Skill can run the following workflow:

1. Read recent feedback from the system where it was given.
2. Associate each item with the Agent output, task context, and Base Skill version.
3. Distinguish explicit domain feedback from vague preference statements.
4. Determine whether the problem is a missing Base Skill rule, an inconspicuous rule, a tool failure, or a task-specific exception.
5. Propose the smallest focused change to the Base Skill.
6. Generate a change description, evidence, and expected impact.
7. Submit the result to a human through a PR or equivalent review workflow.

In Warp's Issue triage example, a scheduled Improver Skill reads GitHub Issues with feedback, uses a Python script bundled with the Skill to fetch and summarize the data as JSON, and then modifies the triage Skill based on maintainer comments. It creates a PR, and the maintainer reviews and merges it before the next triage run inherits the new rule[2].

This example illustrates a practical division of labor: package deterministic feedback retrieval and data preparation as Skill scripts; let the Agent decide which signals are worth preserving and how to make the smallest change; let the normal code-review workflow decide whether the change takes effect.

### 4. Write Principles, Not Piles of Rules

Self-improving Skill instructions should not attempt to enumerate every input. Warp's experience is that principles often generalize better than a large collection of rigid exceptions. For example:

```text
Prioritize recurring code patterns and explain why they increase maintenance cost.
```

is usually more adaptable than a rule set like:

```text
Apply rule A when the variable name starts with x;
apply rule B when the variable name starts with y and is on line 3;
apply rule C when the variable name starts with z but the filename is not test.
```

Explain why a principle exists. The rationale helps an Agent reason by analogy in a new situation instead of mechanically applying conditions that were valid only for old examples. An Improver Skill should first add generalizable decision criteria, and only then consider whether a specific example is necessary.

### 5. Feedback Quality Matters More Than Feedback Volume

Many simple thumbs-up and thumbs-down signals can show direction, but they do not explain what should change. A small amount of detailed feedback from domain experts is often more valuable than a large amount of binary feedback without explanations. At the same time, as the system covers more tasks, a larger corpus of consistently high-quality signals helps the Improver Skill distinguish accidents from reusable patterns[2].

Feedback should therefore not be written into a Skill by simply voting on every user opinion. Preserve the source and context of the feedback, analyze conflicting opinions, and decide whether domain-expert review is required based on task risk. Incorrect feedback is inevitable; an Improver Skill must be able to question feedback rather than accept it blindly.

### 6. Use Version Control and Human Review as Gates

A Skill is an ordinary file, so its changes can go through the normal version-control workflow. Every candidate change should at least describe:

```text
Target: triage/SKILL.md
Reason: A real issue was not labeled ready-to-spec
Evidence: A maintainer explicitly identified the omission and its decision criterion in the issue
Minimal change: Add the conditions for ready-to-spec
Scope: Change only triage-label decisions
Validation: Replay historical issues and verify that existing label rules do not regress
```

Human review is not a ceremonial final step; it is the safety boundary of self-evolution. Reviewers should check whether the feedback is reliable, whether the change is small enough, whether it introduces conflicts, and whether it changes permissions or high-risk behavior. They can merge, request changes, or reject the proposal. A candidate patch that has not passed review must not affect the production Base Skill.

### 7. From One Agent to a Set of Agents

With one Agent, maintain one Base Skill and one Improver Skill. As the number of Agents grows, balance two strategies: Agents with substantially different domains can have separate improvers while sharing the mechanism and general workflow; Agents with similar behavior can share a templated Improver Skill with domain-specific configuration layered on top.

Warp applies the same pattern to triage, specification writing, and code-review Agents in its repository. What is reusable is not a particular domain rule, but the improvement mechanism of “read feedback—understand the reason—propose a small change—enter review”[2].

### 8. How to Tell Whether the System Is Actually Better

Do not look only at local Base Skill success cases. Track the business metrics that people already care about, such as:

- The proportion of code-review comments that are accepted or closed.
- The time from issue creation to entering the specification stage.
- The time humans spend editing Agent output.
- Task retry rate, human handoff rate, and cost.
- Feedback volume and the proportion of useful feedback.

Deploy with a crawl-walk-run progression: first replay historical data offline, then run on a small set of low-risk tasks, and only then expand the scope. Bind every execution to a Base Skill version and retain the old version and PR records. If global metrics deteriorate or a new security issue appears, roll back the version rather than letting the Improver Skill keep stacking automatic changes in production.

In short, the minimum controlled self-evolution loop is not “the Agent modifies itself,” but:

```text
Base Skill executes
    → Human provides low-friction feedback
    → Improver Skill observes periodically
    → Proposes a small, explainable change
    → PR/evaluation/human review
    → The merged version inherits the improvement
```

Skill is therefore a file interface connecting runtime experience with engineering knowledge: Agents can help discover and propose improvements, while people and the version-control system decide which improvements actually enter production.

## 7. Common Anti-Patterns

### Anti-Pattern 1: Put All Knowledge in `SKILL.md`

The entry file grows longer and the Agent must face a large amount of irrelevant content on every use. Keep information needed by every branch in the entry file and split branch-specific material into references.

### Anti-Pattern 2: Use a Vague Description to Capture Every Task

“Process any document” and “help with all kinds of work” cause false activation. A `description` should state the capability and usage context precisely, and distinguish the Skill from neighboring Skills.

### Anti-Pattern 3: Make Scripts Only Throw Exceptions

Scripts should return errors that help the Agent correct the problem, rather than pushing all diagnosis back to the model. Configuration parameters also need reasons, so that timeouts and retry constants are explainable.

### Anti-Pattern 4: Give the Agent Too Many Equivalent Options

When multiple tools can perform the same action, specify a default and offer alternatives only under explicit conditions. Too many choices increase decision cost and make evaluations less stable.

### Anti-Pattern 5: Execute Unvalidated Batch Changes Directly

For complex operations, generate a structured plan first, validate it, execute it, and check the final result. Do not replace machine-verifiable intermediate results with “the model seems to understand.”

## 8. Release Checklist

Before releasing a Skill, check each item:

- Does `description` state both the capability and the usage context?
- Is the out-of-scope boundary defined?
- Is `SKILL.md` short, with detailed content split appropriately?
- Are filenames and paths clear and portable?
- Are references explicitly linked from the entry file?
- Is it clear whether each script should be executed or read?
- Do scripts handle common errors explicitly?
- Do critical operations have validators or completion criteria?
- Do high-risk operations have confirmation and rollback paths?
- Are there at least three evaluation scenarios covering normal, boundary, and failure cases?
- Was a no-Skill baseline measured?
- Was the Skill tested on real tasks with a fresh instance?
- Are versions, trajectories, and failure cases recorded?

## Conclusion

The core of a Skill is not a longer prompt, but an Agent workflow description that can be discovered, loaded, executed, and verified.

Its execution path can be summarized as:

```text
Metadata discovery → LLM selection → on-demand SKILL.md loading
→ read necessary resources → execute scripts and tools → verify results
→ record failures → evaluate and iterate → controlled release
```

When writing a Skill, start with real failures and evaluations, then write the minimum instructions. When organizing it, keep stable rules in the entry file, branch-specific details in resources, and deterministic work in scripts. When maintaining it, use versions, gates, gradual rollout, and rollback to control self-evolution. This is how a Skill grows from a one-off prompt into a genuinely reusable Agent capability.

## References

[1]: [Claude Platform: Skill authoring best practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)\
[2]: [Anthropic: How Warp builds self-improving agents on Claude](https://claude.com/blog/how-warp-builds-self-improving-agents-on-claude)\
[3]: [Claude Code: Extend Claude with skills](https://code.claude.com/docs/en/skills)\
[4]: [pi: Skills documentation](https://github.com/badlogic/pi-mono/blob/main/packages/coding-agent/docs/skills.md)\
[5]: [Hermes Agent: Skills System](https://hermes-agent.nousresearch.com/docs/user-guide/features/skills)
