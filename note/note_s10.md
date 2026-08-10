这一节讲系统提示词, 主要观点是, 系统提示词是运行时组装起来的.

目前从s01到s09, SYSTEM PROMPT 都是硬编码出来的. SYSTEM PROMPT要与Agent的能力对应, 造成了几个难点:
1. 每次请求需要携带全部内容;
2. 加描述可能和之前冲突
3. 换项目需要重写整个提示词

解决方法:

将 SYSTEM PROMPT 拆分成独立section, 运行时根据真实状态拼接, 比如

1. identity 	始终 	你是谁、怎么做事 
2. tools 		始终 	可用工具列表 
3. workspace 	始终 	工作目录
4. memory 		按需 	相关记忆内容

关键设计：section 是否加载取决于真实状态（工具是否存在、文件是否存在）。

缓存避免重复拼接, 当上下文不改变时, 使用缓存提示词内容


运行结果
```bash
(learn-claude-code) gkunix@laptopGK:~/workspace/learn-claude-code$ python ./s10_system_prompt/code.py
s10: system prompt — runtime assembly
Enter a question, press Enter to send. Type q to quit.

[36ms10 >> [0mRead the file README.md
  [assembled] sections: identity, tools, workspace, memory
> read_file
[English](./README.md) | [中文](./README-zh.md) | [日本語](./README-ja.md)

<a href="https://trendshift.io/repositories/19746" target="_blank"><img src="https://trendshift.io/api/badge/repositories/19746" 
  [cache hit] system prompt unchanged
I've read the README.md. It's the main documentation for the **learn-claude-code** repository — a 0-to-1 tutorial project on harness engineering for AI agents.

Key points:

- **Core thesis**: Agency (perceive/reason/act) comes from model training, not code. An agent product = Model + Harness. The harness is tools, knowledge, observation, action interfaces, and permissions.
- **Core pattern**: A single agent loop (`messages` → LLM → tool_use handling → loop), which never changes; each lesson layers one harness mechanism on top.
- **20 progressive lessons** (`s01_agent_loop` through `s20_comprehensive`), each adding one mechanism: tool use, permissions, hooks, TodoWrite, subagents, skill loading, context compaction, memory, system prompts, error recovery, task systems, background tasks, cron, agent teams, protocols, autonomous agents, worktree isolation, MCP, and a comprehensive endpoint.
- **Version status**: Root `s01-s20` is the current canonical track; legacy 12-lesson track lives in `docs/`, `agents/`, and `web/` (with a mapping table between the two).
- **Structure**: Each chapter folder has README (Chinese source + EN/JA translations), runnable `code.py`, and SVG diagrams.
- **Also covers**: Quick start (clone, pip install, run `s01`/`s08`/`s20`), project layout, related products (Kode Agent CLI/SDK), a sister tutorial (`claw0` for always-on assistants), and MIT license.

[36ms10 >> [0mCreate a file called .memory/MEMORY.md with content "- [test](test.md) — test memory"
  [cache hit] system prompt unchanged
> write_file
Wrote 32 bytes to .memory/MEMORY.md
  [assembled] sections: identity, tools, workspace, memory
Created `.memory/MEMORY.md` with the specified content:

```markdown
- [test](test.md) — test memory
```

[36ms10 >> [0mRead the file code.py
  [cache hit] system prompt unchanged
> bash
s01_agent_loop/code.py
s02_tool_use/code.py
s03_permission/code.py
s04_hooks/code.py
s05_todo_write/code.py
s06_subagent/code.py
s07_skill_loading/code.py
s08_context_compact/code.py
s09_memory/code.p
  [cache hit] system prompt unchanged
> read_file
#!/usr/bin/env python3
"""
s01_agent_loop.py - The Agent Loop

The entire secret of an AI coding agent in one pattern:

    while stop_reason == "tool_use":
        response = LLM(messages, tools)
   
  [cache hit] system prompt unchanged
I've read `s01_agent_loop/code.py`. It's the first lesson's standalone implementation of the core agent loop:

- **Tool definition**: A single `bash` tool, defined via an Anthropic protocol tool schema (name, description, input_schema).
- **`run_bash`**: Executes shell commands with `subprocess.run` (120s timeout, 50KB output cap, blocks dangerous commands like `rm -rf /` and `sudo`).
- **`agent_loop`**: The core pattern — repeatedly calls `client.messages.create` with `messages` + `tools`; appends the assistant turn; if `stop_reason != "tool_use"` it returns; otherwise executes each tool_use block, collects `tool_result` blocks, appends them as a user message, and loops.
- **Entry point**: An interactive REPL that accumulates `history` across turns, so the model retains multi-turn conversation context.
- **Docs**: File-ending comments document the `response` structure (`stop_reason` values, text vs. tool_use content blocks).

This is the canonical minimal agent: one loop + one tool (bash).
```