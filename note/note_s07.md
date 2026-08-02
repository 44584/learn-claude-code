这一节讲的是Skill的按需加载

首先抛出了一个问题：在项目开发中，为了要求遵守多个规范，直接将规范全部加入到system prompt中，每次调用都使用全部，造成了token浪费。

---

通过启动时注入目录，运行时按需加载内容解决。

- 启动时(系统提示词构建时)，harness扫描skills/，构建system prompt
  在启动时 \_scan_skills() 一次性读入内存的shills注册表，只使用name和description构建catalog作为system prompt的一部分
- 运行时(PreToolUse hook 中)，通过指定name读取完整的SKILL.md 【这里理解有误，load_skill是作为tool而非hook】
  - 运行时 load_skill 只是按 name 在skills的注册表中查找加载
  - 防路径遍历的设计

---

skill的结构：
统一存储在skills/目录下，每个skill对应一个子目录，必须包含一个SKILL.md文件。

SKILL.md是YAML结构，包括name，description，content等。目录就使用name和description构建。【这里理解有误差，content改为body】

虽然启动时只使用name和description构建目录，但是整个SKILL.md的内容也作为content也被注册到字典中，方便使用时按照name调取。然后file和bash工具可以按照指导，进一步访问对应skill的reference/，script/，assets/等目录。

技能内容只是作为工具结果，进入当前messages _会随历史一直保留，直到压缩/截断/会话结束。_，而不是system prompt。

看完代码实现后，发现了几处误解

1. SKILL_REGISTRY中的content是SKILL.md的全部内容，而非仅仅body。
2. 运行时的load_skill是作为tool，而非注入到hook中。这也可以理解，因为作为工具就可以统一管理（和其他工具共用调用方式）。

运行：

两个观察重点都有了：

- 从SYSTEM知道了有哪些技能
- 需要完整规范时出现[HOOK] load_skill

````bash
(learn-claude-code) gkunix@laptopGK:~/workspace/learn-claude-code$ python ./s07_skill_loading/code.py
s07: Skill Loading — catalog in SYSTEM, content on demand
Type a question, press Enter. Type q to quit.

[36ms07 >> [0mtell me what skills are available
[HOOK] UserPromptSubmit: working in /home/gkunix/workspace/learn-claude-code
[HOOK] Stop: session used 0 tool calls
Here are the skills available in this workspace:

1. **agent-builder** — Design and build AI agents for any domain. Use when users ask to "create an agent," "build an assistant," or "design an AI system"; want to understand agent architecture, agentic patterns, or autonomous AI; need help with capabilities, subagents, planning, or skill mechanisms; or ask about Claude Code, Cursor, or similar agent internals.

2. **code-review** — Perform thorough code reviews with security, performance, and maintainability analysis. Use when the user asks to review code, check for bugs, or audit a codebase.

3. **mcp-builder** — Build MCP (Model Context Protocol) servers that give Claude new capabilities. Use when the user wants to create an MCP server, add tools to Claude, or integrate external services.

4. **pdf** — Process PDF files: extract text, create PDFs, merge documents. Use when the user asks to read a PDF, create a PDF, or work with PDF files.

To use one, just ask — e.g., "review my code," "create an agent," "build an MCP server," or "extract text from this PDF" — and I'll load the full skill details to help. Is there anything specific you'd like to do?

[36ms07 >> [0mtell me the content of pdf skill
[HOOK] UserPromptSubmit: working in /home/gkunix/workspace/learn-claude-code
[HOOK] load_skill
[HOOK] Stop: session used 1 tool calls
Here's the full content of the **pdf** skill:

---

# PDF Processing Skill

You now have expertise in PDF manipulation. Follow these workflows:

## Reading PDFs

**Option 1: Quick text extraction (preferred)**
```bash
# Using pdftotext (poppler-utils)
pdftotext input.pdf -  # Output to stdout
pdftotext input.pdf output.txt  # Output to file

# If pdftotext not available, try:
python3 -c "
import fitz  # PyMuPDF
doc = fitz.open('input.pdf')
for page in doc:
    print(page.get_text())
"

```
````

# tool 和 hook 辨析（因为一开始将load_skill误解为hook）

工具让模型成为知识获取的决策者，hook 让 harness 成为规则的执行者。 load_skill 的本质是"模型按需获取知识"，天然属于前者；权限拦截、日志这类"不管模型怎么想都必须执行"的才属于后者。

| 维度         | tool                                         | hook                                              |
| :----------- | :------------------------------------------- | :------------------------------------------------ |
| 谁发起       | 模型主动调用                                 | harness 固定事件触发                              |
| 对模型可见   | 可见（schema 进 API）                        | 不可见                                            |
| 结果去向     | 作为 tool_result 进入 messages，成为对话历史 | 通常只打日志；除非返回阻塞/强制消息，否则不进历史 |
| 能否推理判断 | 能——模型根据当前任务决定调不调               | 不能——按写死的代码条件执行                        |
| 失败处理     | 返回字符串结果，模型可见并可自我纠正         | 只能静默、打日志或强行阻塞                        |
| 本质角色     | 模型的能力（capability）                     | 系统的策略（policy）                              |

## load_skill 作为 tool 的好处

1. 按需决策权交给了模型——这是核心。
   两级加载的成立前提是"模型自己判断此刻需不需要这份规范"。它可以在改 SQL 时调 load_skill("sql-style")，改 CSS 时不调。hook 做不到这一点。

2. 复用现有调度机制，零特判。
   加一个工具只需在 TOOLS 加 schema、在 TOOL_HANDLERS 加映射，之后的 PreToolUse/PostToolUse 钩子、tool_result 注入、Stop 统计全部自动生效，不需要为技能加载单独写一条代码路径。

3. 内容自然进入对话历史。
   工具的返回值按 API 语义作为 tool_result 落进 messages，随后续轮次一直保留，模型可以边看规范边继续调用 file/bash 访问 references/ 等资源。如果做成 hook，要手动往 messages 里塞消息，还绕过了 tool_result 协议，格式和子 Agent 的隔离都会出问题。

## 如果反过来做成 hook 会怎样

设想要在 UserPromptSubmit hook 里扫描用户问题、发现提到 "pdf" 就自动注入 PDF 技能内容：每次提问都会跑这个检测；匹配靠关键词而不是推理，容易误触或漏判；注入的内容要么塞进 system prompt（违背按需原
则、每轮都付 token），要么手工伪造消息（破坏 API 的消息格式）；而且"先加载技能、再按技能指引读文件"这种顺序依赖用 hook 根本无法表达——它们没有返回值进入对话的能力。
