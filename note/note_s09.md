这一节讲的是记忆功能。 文件仓库 + 索引 + 按需加载，跨压缩、跨会话

- 每个记忆对应一个.md文件，yaml formatter(name, description, type)
- 索引文件MEMORY.md，一行一个链接指向记忆文件
- 索引常驻SYSTEM PROMPT，记忆按需注入
  - 每次用户请求开始时，把最近对话和记忆catalog（name和description）发给LLM做轻量的side-query，选出相关name（name.md也就是文件名），再读取文件中的内容，临时注入到当前的 user turn

- 每轮对话结束后，写入记忆
  - 使用压缩前的快照自动提取记忆（因为压缩先于记忆提取）
  - 要求 LLM 返回 {name, type, description, body} 的 JSON 数组，
  - 在提示词中给出现有的记忆的catalog，要求只给出确实不存在的记忆

- 整理记忆：低频合并去重
  - 每轮对话结束后检查，文件数目超过阈值（10）时触发；
  - 让LLM完成去重、合并矛盾、淘汰过时

- 四类记忆（对应记忆文件中的 type）
  - user 你是谁
  - feedback 怎么做事
  - project 正在发生什么
  - reference 东西在哪里找

Memory适合保存什么？\
保存跨对话依然有效的信息。比如用户偏好，反复出现的反馈、项目背景、常用入口、排查线索。\
关注以后还会用到什么并通过 索引+按需加载，把信息带回当前对话。

# 运行代码检验

确实出现了新建记忆文件和索引，以及按照记忆工作。

````python
(learn-claude-code) gkunix@laptopGK:~/workspace/learn-claude-code$ python ./s09_memory/code.py
s09: Memory — persistent cross-session knowledge
输入问题，回车发送。输入 q 退出。

[36ms09 >> [0mI prefer using tabs for indentation, not spaces. Remember that.
> bash
total 256
drwxr-xr-x 34 gkunix gkunix  4096 Aug  8 13:23 .
drwxr-xr-x  7 gkunix gkunix  4096 Jul 22 19:38 ..
-rw-r--r--  1 gkunix gkunix  2547 Jul 15 17:15 .env
-rw-r--r--  1 gkunix gkunix  2524 Jul 1
> bash
total 8
drwxr-xr-x  2 gkunix gkunix 4096 Aug  8 13:23 .
drwxr-xr-x 34 gkunix gkunix 4096 Aug  8 13:23 ..
=== /home/gkunix/workspace/learn-claude-code/.memory/* ===
cat: '/home/gkunix/workspace/learn-c
> bash
/home/gkunix/workspace/learn-claude-code/s09_memory/code.py
/home/gkunix/workspace/learn-claude-code/s09_memory/images/memory-subsystems.en.svg
/home/gkunix/workspace/learn-claude-code/s09_memory/imag
> bash
# s09: Memory — 压缩会丢细节，要有一层不丢的

[中文](README.md) · [English](README.en.md) · [日本語](README.ja.md)

s01 → ... → s07 → s08 → `s09` → [s10](../s10_system_prompt/) → s11 → ... → s20
> *"压缩会丢细节, 要有一层不丢的"* —
> write_file
Wrote 243 bytes to /home/gkunix/workspace/learn-claude-code/.memory/user-preference-tabs.md
> write_file
Wrote 86 bytes to /home/gkunix/workspace/learn-claude-code/.memory/MEMORY.md
Remembered! ✅

I've stored your preference as a `user` memory:

- **`.memory/user-preference-tabs.md`** — the full memory file with metadata (`name`, `description`, `type`)
- **`.memory/MEMORY.md`** — the index, which gets injected into the system prompt so the memory can be matched and loaded on demand

**Rule going forward:** I'll always use tabs for indentation, never spaces, when writing or editing files.

Note this only takes effect for future turns in this workspace once the memory system loads the index — let me know if you'd like me to apply it to any existing files right now.

[36ms09 >> [0mCreate a Python file called test.py under ./tests/
> bash
total 36
drwxr-xr-x  2 gkunix gkunix 4096 Jul 14 16:28 .
drwxr-xr-x 34 gkunix gkunix 4096 Aug  8 13:23 ..
-rw-r--r--  1 gkunix gkunix  603 Jul 14 16:28 test_agents_smoke.py
-rw-r--r--  1 gkunix gkunix
> write_file
Wrote 123 bytes to /home/gkunix/workspace/learn-claude-code/tests/test.py
> bash
"""Example test file. Indentation uses tabs."""$
$
$
def test_example():$
^I"""A simple placeholder test."""$
^Iassert 1 + 1 == 2$
---
compiles OK
Done! Created `tests/test.py`:

```python
"""Example test file. Indentation uses tabs."""


def test_example():
        """A simple placeholder test."""
        assert 1 + 1 == 2
````

- ✅ Indentation uses **tabs** (`^I` confirmed via `cat -A`)
- ✅ File compiles cleanly (`py_compile` passed)

Want me to replace the placeholder test with something useful, or add tests for a specific module?

[36ms09 >> [0mWhat did I tell you about my preferences?
You told me you **prefer using tabs for indentation, not spaces**. I saved that as a memory (`user-preference-tabs`) and have been applying it — e.g., `tests/test.py` was written with tab indentation.

[36ms09 >> [0mI also prefer single quotes over double quotes for strings.

> bash

- [user-preference-tabs](user-preference-tabs.md) — User prefers tabs for indentation
  MEMORY.md
  user-preference-tabs.md

  > write_file
  > Wrote 319 bytes to /home/gkunix/workspace/learn-claude-code/.memory/user-preference-single-quotes.md
  > write_file
  > Wrote 214 bytes to /home/gkunix/workspace/learn-claude-code/.memory/MEMORY.md
  > Done! ✅ New memory saved:

- **`.memory/user-preference-single-quotes.md`** — `User prefers single quotes over double quotes for strings`
- **`.memory/MEMORY.md`** — index updated

So now your preferences on record are:

1. **Tabs** for indentation, not spaces
2. **Single quotes** for strings, not double quotes

I'll apply both going forward. One note: `tests/test.py` currently uses a double-quoted docstring — want me to switch it to single quotes?

[36ms09 >> [0m

```

```

# codex指正与补充

使用时，用户主动要求记住，会发生两层保障：

1. 模型主动调用write_file，按照SYSTEM PROMPT，完成记忆文件与索引文件的构建；
2. harness事后，根据对话内容完成提取和构建。

_如果模型不守规矩，同一偏好可能被写成两个文件。CC 有显式的跳过逻辑（"如果主 Agent已写入记忆文件则跳过提取"）_

关于session memory与user memory

README 专门有一节讲：user memory 管跨会话，session memory 管"compact 之后当前会话还要保留什么"

在教学版代码中，session memory就是messages列表；user memory则是当前的实现