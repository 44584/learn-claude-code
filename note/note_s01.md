这一节主要讲的是 Agent Loop

readme中给出的是 "while 循环中，执行 工具调用->执行->喂回->再问"。

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

tool_use

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

session 是一次运行，turn 是用户消息到完整响应，内部每调一次模型算一个 step。

| 术语     | English          | 含义                                                           | 在代码中的对应                                                                                                                     |
| -------- | ---------------- | -------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| 一次对话 | conversation     | 从第一条消息到会话结束的完整过程                               | history 列表的整个生命周期                                                                                                         |
| 一个会话 | session          | 程序的一次运行（打开终端到退出）                               | `__main__` 里 history 从空到积累的全部；跨会话 = 两次运行之间靠磁盘 `.memory/` 保留                                                |
| 一轮对话 | turn / round     | 用户发一条消息 + 智能体完整处理完（可能内部调多次 LLM 和工具） | 主循环里一次 `input()` → 一次 `agent_loop(history)` 调用。`extract_memories` 正是在"本轮结束"（`stop_reason != "tool_use"`）时触发 |
| 一个回合 | exchange         | 一条 user 消息 + 一条 assistant 消息（最简对话单元）           | 可以粗略等同于"一轮"，但更常用于指单个问答对                                                                                       |
| 一步     | step / iteration | `agent_loop` 内部的一次 LLM API 调用（含其后可能执行的工具）   | `agent_loop` 里 `while True` 的一次迭代：`client.messages.create` → 执行工具 → 继续                                                |
| 一条消息 | message          | `messages` 列表里的一个 dict                                   | `{"role": "user", "content": ...}`                                                                                                 |
| 一个块   | block            | 消息 content 里的一个元素                                      | `text` / `tool_use` / `tool_result` 块                                                                                             |
