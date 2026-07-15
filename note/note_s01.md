这一章主要讲的是 Agent Loop

readme中给出的是 "while 循环中，执行 工具调用->执行->喂回->再问"。

感觉应该将描述直接改为ReAct范式，即 感知，思考，执行，观察的循环。

需要注意的是 stop_reason，分为 stop_reason=="tool_use"和stop_reason|="tool_use"两种。

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
