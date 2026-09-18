这一节主要讲的是 Agent Loop

readme中给出的是 "while 循环中，模型调用工具->执行->喂回->再问"。

<!-- 20260914补充 -->

- 模型调用工具是一条 `stop_reason==tool_use`的消息，`content`中包含了**tool call**（工具以及工具的参数）。
  - tool specification 已经以Json Schema格式加入了 prompt
- 执行则是 agent runtime 根据 `content` 执行工具。
  > 没有把工具调用作为一条消息加入消息列表，因为 tool call 已经以block形式存在于assistant消息中了
- 喂回：向维护的消息列表中，添加`role`为`user`，`content`为工具执行结果的消息。

<!-- /20260914补充 -->

感觉应该将描述直接改为ReAct范式，即 感知，思考，执行，观察的循环。

需要注意的是 stop_reason，分为 stop_reason=="tool_use"和stop_reason!="tool_use"两种。

```bash
(learn-claude-code) gkunix@laptopGK:~/workspace/learn-claude-code$ python ./s01_agent_loop/code.py
s01: Agent Loop
输入问题，回车发送。输入 q 退出。

[36ms01 >> [0mcount the number of files under this dir.
$ find . -type f | wc -l
3139
There are **3,139** files under the current directory (including all subdirectories).

[36ms01 >> [0m
```

突然注意到，这里的消息列表设计和我设想的不一样。

这里的设计中，role 只有 assistant 和 user 两种（而没有tool），tool_result 作为 content block 放在 user 的 content 中。
保持对话历史中每条消息的 role 清晰：用户提供输入（包括工具结果），助手给出回复（包括工具调用）。

[代码](../s01_agent_loop/code.py)

# 看了s08后回顾

## tool specification的格式

```json
TOOLS = [
	{
        "name": "bash",
        "description": "Run a shell command.",
        "input_schema": {
            "type": "object",
            "properties": {"command": {"type": "string"}},
            "required": ["command"],
        },
    },
]
```

## response的格式

response的结构

```json
{
"id": "msg_xxx",
"type": "message",
"role": "assistant",
"content": [ /* 这里是 block 数组 */ ],
"model": "...",
"stop_reason": "end_turn" | "tool_use" | "max_tokens" | "stop_sequence",
"usage": { "input_tokens": N, "output_tokens": M }
}
```

纯文本

```json
{
  "content": [{ "type": "text", "text": "你好！" }],
  "stop_reason": "end_turn"
}
```

tool call

```json
{
  "content": [
    { "type": "text", "text": "让我查一下" },
    {
      "type": "tool_use",
      "id": "toolu_xxx",
      "name": "bash",
      "input": { "command": "ls" }
    }
  ],
  "stop_reason": "tool_use"
}
```

## message的格式

需要注意的是message如何收集response，形成自己的格式.

- user 文本消息的 content 是 str
- tool_result 消息的 content 是 dict 列表，
- assistant 消息的 content 才是 response.content

对于assistant消息：

```json
{
	"role":"assistant",
	"content": response.content
}
```

对于user消息，分为user输入的消息和工具结果消息这两种。

user输入：

```json
{
	"role":"user",
	"content": input
}
```

工具结果：

```json
{
	"role":"user",
	"content": [
		{
			"type":"tool_result",
			"tool_use_id":id1,
			"content":output1
		},
		{
			"type":"tool_result",
			"tool_use_id":id2,
			"content":output2
		},
	]
}
```

# 关于 主循环 与 智能体循环

在本项目中，外层 `turn` 是一次用户输入到 `agent_loop(history)` 返回的完整处理；其内部可以有多个 `step`。一个 `step` 是一次 LLM 调用，以及其后可能发生的工具执行和 `tool_result` 回填。

| 术语 | English | 含义 | 在代码中的对应 |
| --- | --- | --- | --- |
| 会话 | session | 一段持续的交互状态；在这个 REPL 中，约等于一次程序启动到退出 | `__main__` 中 `history` 从创建到进程退出。s01 本身不跨进程保留历史；s09 才会把部分长期信息写入 `.memory/` |
| 一轮 | turn | 一次用户输入到智能体完整处理完，可包含多次模型调用和工具调用 | 外层 `input()` 循环的一次迭代：追加用户消息 → `agent_loop(history)` 返回 |
| 消息 | message | `messages` / `history` 列表中的一个消息对象 | `{"role": "user" \| "assistant", "content": ...}` |
| 块 | block | 一条 list 形式 `content` 中的一个元素 | `text`、`tool_use`、`tool_result` |

单条 `message` 的形态如下：

```python
# 1. 用户的原始文本输入：content 是字符串
{"role": "user", "content": "统计当前目录下的文件数"}

# 2. assistant 的普通最终回复：content 是 text block 列表
{"role": "assistant", "content": [
    {"type": "text", "text": "当前目录共有 42 个文件。"}
]}

# 3. assistant 请求工具：同一条消息可含文字和多个 tool_use block
{"role": "assistant", "content": [
    {"type": "text", "text": "我来检查一下。"},
    {"type": "tool_use", "id": "toolu_1", "name": "bash",
     "input": {"command": "find . -type f | wc -l"}},
    {"type": "tool_use", "id": "toolu_2", "name": "bash",
     "input": {"command": "pwd"}},
]}

# 4. 工具执行反馈：role 仍是 user；一条消息可对应多个 tool_result block
{"role": "user", "content": [
    {"type": "tool_result", "tool_use_id": "toolu_1", "content": "42"},
    {"type": "tool_result", "tool_use_id": "toolu_2", "content": "/workspace"},
]}
```

`turn` 中的消息序列则有多种情况。`A(...)` 表示 assistant message，`U(...)` 表示 user message：

```text
# 没有工具调用：一个用户输入 + 一个最终回答
U(原始文本) → A(最终 text)

# 一次工具调用：一个 tool_use 对应一个 tool_result
U(原始文本) → A(tool_use 1) → U(tool_result 1) → A(最终 text)

# 并行工具调用：同一条 assistant message 有多个 tool_use，
# 因而紧随的一条 user message 有多个对应的 tool_result
U(原始文本) → A(tool_use 1, tool_use 2) → U(tool_result 1, tool_result 2) → A(最终 text)

# 多次循环：模型收到第一批结果后再次调用工具，产生多条 tool-result user message
U(原始文本) → A(tool_use 1) → U(tool_result 1)
             → A(tool_use 2) → U(tool_result 2) → A(最终 text)

# 两种情况可组合：每一次工具调用步骤都可以并行调用多个工具，且整个 turn 可以有多步
U(原始文本) → A(tool_use 1, tool_use 2) → U(tool_result 1, tool_result 2)
             → A(tool_use 3, tool_use 4) → U(tool_result 3, tool_result 4)
             → A(最终 text)
```

所以，多个 `tool_result` 可能来自同一条 user message 中的并行工具调用，也可能分布在同一 `turn` 内多次循环产生的多条 user message 中，或同时具备两种情况。工具调用链中的 `user(tool_result)` 是协议中的 user 侧反馈，不代表人类用户又发了一条文本消息。
