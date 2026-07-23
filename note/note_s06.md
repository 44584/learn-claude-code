这一节讲的是 subagent, 主旨是 大任务拆小，让小任务有干净的上下文。

开一个独立的子进程，给它一个独立的消息列表，让它专心做一件事，然后只把最终答案返回给Parent（但文件系统上的改动保留）。subagent的工具受限，比如不能再spawn子进程了。

本章将spawn一个subagent的能力封装为一个工具。

subagent设计注意事项：

- 上下文隔离：subagent的中间过程不干扰agent
- 只传回结论
- 禁止递归
- 遵守安全策略

[代码](../s06_subagent/code.py)
