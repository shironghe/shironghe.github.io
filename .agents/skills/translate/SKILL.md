---
name: translate
description: Tanslate Chinese technical blogs and articles into English. Use when the user wants to translate blogs or submit a PR without corresponding English blogs been established.
license: MIT
metadata:
  author: shironghe
  version: "1.0"
  generatedBy: "1.10.0"
---

# Translate Chinese technical blogs and articles into English

This is a specialized skill designed to automatically translate Chinese technical blogs and articles into English, supporting common blog formats like Markdown and HTML, with intelligent handing of non-text elements such as images and links.

## PRINCIPLES
- Technical terms are maintained in [Technical Terminology](./technical-terminology.yaml)
- Translations should prioritize vocabulary from the technical terminology table. If a term has no corresponding entry, create one in the table before using it.
- A Chinese sentence may be split into multiple English sentences, as long as the meaning remains consistent.
- Prefer strong verbs over nominalization. For example, "测量厚度" should be translated as "Thickness is measured" rather than "Measurement of thickness is made."
- Use the passive voice where appropriate, but not as a strict requirement.
- Prefer short sentences over long, complex ones: avoid overly nested structures and clauses.
- Use declarative sentences as the primary form.

## Non-text elements
### Front Matter
- Translate only the title and description into English and keep all other fields unchanged
**Example:**
  ```yaml
  # Before
  title: "Agent Loop 架构解析：为什么主流 Agent 都选 ReAct？"
  description: "Agent Loop 是 Agent 的核心循环。从 ReAct、Plan-and-Execute、ReWOO 到 Reflexion，看懂常见架构，也理解 ReAct 为什么成为主流默认。"
  date: 2026-08-25
  draft: false
  tags: ["Agent", "ReAct", "Agent Loop", "LLM"]
  translationKey: "Agent Loop"

  # After
  title: "Agent Loop Architecture Explained: Why Do Mainstream Agents All Choose ReAct?"
  description: "Agent Loop is the core loop of an Agent. From ReAct, Plan-and-Execute, ReWOO to Reflexion, understand common architectures and why ReAct has become the mainstream default."
  date: 2026-08-25
  draft: false
  tags: ["Agent", "ReAct", "Agent Loop", "LLM"]
  translationKey: "Agent Loop"
  ```
### Images
- Translate `alt` text: describe image content
- Translate `title` attribute: image title
- Preserve image URL unchanged and image dimension parameters
**Example:**
  ```yaml
  # Before
  ![Docker 架构图](https://example.com/docker-arch.png "Docker 架构示意图")

  # After
  ![Docker architecture diagram](https://example.com/docker-arch.png "Docker architecture overview")
  ```

### Links
- Preserve complete URL and link attributes (target, rel, etc.) unchanged
- Translate link anchor text
**Example:**
  ```yaml
  # Before
  [点击查看官方文档](https://docs.docker.com "Docker 官方文档")

  # After
  [View the official documentation](https://docs.docker.com "Docker official documentation")
  ```
### Code
- Translate only comments
**Example:**
  ```yaml
  # Before
    ```python
    # 初始化数据库连接
    # 配置连接池大小
    db = connect(host='localhost', pool_size=10)
    ```
  # After
    ```python
    # Initialize database connection
    # Configure connection pool size
    db = connect(host='localhost', pool_size=10)
    ```
  ```
### HTML Tags
- Preserve tag structure, translate only inner text
**Example:**
  ```yaml
  # Before
  <p class="highlight">这是一个重要的技术概念，需要理解其核心原理。</p>

  # After
  <p class="highlight">This is an important technical concept; its core principles need to be understood.</p>
  ```
### Formulas
- Preserve Completely
**Example:**
  ```yaml
  # Before
  $$E = mc^2$$
  这个公式描述了质能等价关系。

  # After
  $$E = mc^2$$
  This formula describes the mass-energy equivalence relationship.
  ```

## Quality Assurance - After translation, it is necessary to check the translated blogs and articles
- [] Ensure all technical terms follow [Technical Terminology](./technical-terminology.yaml)
- [] Compare each sentence to the original—verify no loss, addition, or distortion. Split long sentences should retain logical flow. 
- [] Front Matter: translate only `title` and `description`.  
- [] Images: translate `alt` and `title` attributes.  
- [] Links: translate anchor text only.  
- [] Code: translate only comments.  
- [] HTML: translate only inner text.  
- [] Formulas: leave completely unchanged.  
- [] ]Preserve Markdown/HTML structure. Replace Chinese punctuation with English equivalents (e.g., quotes, enumeration commas).
