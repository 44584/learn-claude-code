这一节主要讲Tool Use

感觉设置这一节的考量主要是工具的原子性。
看完后，觉得并非，只是用工具简化了常用功能，不然edit一个file，用shell挺复杂的。

只有bash工具时，LLM需要做两件事情：1.思考需要使用什么指令（cat，ls等等），2.编写shell（cat path/to/file）

有了更多工具时，模型只需要思考使用什么工具。

这一节做了工具分发：

1. 定义工具（Json Schema）；
2. 注册处理函数。
   然后通过模型的工具调用json获取函数名和参数，通过注册表获取函数然后执行。

多个工具调用：比如"读一下 a.py 和 b.py，然后列出所有 .py 文件"。
教学版按照 response.content 逐个执行

尝试运行，发现现在llm能力真是强，虽然agent loop很简洁且几乎没有错误处理，但是简单任务能一遍过。

```bash
(learn-claude-code) gkunix@laptopGK:~/workspace/learn-claude-code/s02_tool_use$ python ./code.py
s02: Tool Use — 在 s01 基础上加了 4 个工具
输入问题，回车发送。输入 q 退出。

[36ms02 >> [0mRead the file README.md and tell me what this project is about
> read_file
# s02: Tool Use — 多加一个工具，只加一行

[中文](README.md) · [English](README.en.md) · [日本語](README.ja.md)

s01 → `s02` → [s03](../s03_permission/) → s04 → ... → s20
> *"加一个工具, 只加一个 handler"* — 循环不用动, 新工具注册进 disp
## 项目简介

这个项目是 **s02: Tool Use**，属于一个教学系列（从 s01 到 s20），核心主题是 **让 AI Agent 学会使用多种工具**。

### 核心思路

从一个只有 `bash` 工具的简单 Agent（s01）扩展为拥有 **5 个专用工具** 的 Agent：

| 工具 | 用途 |
|------|------|
| `bash` | 执行 shell 命令 |
| `read_file` | 读取文件内容 |
| `write_file` | 写入文件内容 |
| `edit_file` | 替换文件中的文本 |
| `glob` | 按模式查找文件 |

### 架构亮点：工具分发（Tool Dispatch）

项目引入了一个 **`TOOL_HANDLERS` 字典**，实现"查表分发"——加一个工具只需要做两件事：

1. 在 `TOOLS` 数组里加一条工具定义（JSON schema）
2. 在 `TOOL_HANDLERS` 字典里加一行函数映射

Agent 主循环（`while True`）完全不用改，体现了 **"循环不变，工具可扩展"** 的设计理念。

### 适用场景

你可以用这个项目来体验：

- 让 Agent 同时调用多个工具（如同时读多个文件）
- 观察模型何时调用单个工具 vs 多个工具
- 理解工具分发机制如何让 Agent 能力平滑扩展

试一些 prompt，比如：_"Read the file README.md and tell me what this project is about"_ 😄

[36ms02 >> [0mRead the file README.md and tell me what this project is about, 然后在workdir创建一个md文件,写入你的回答.

> bash
> workdir does not exist
> bash
> (no output)
> write_file
> Wrote 1298 bytes to workdir/关于项目的介绍.md
> 全部完成！以下是操作总结：

### 📖 读取 README.md 后的回答

**项目 s02: Tool Use** 的核心是让 AI Agent 从只有 1 个 `bash` 工具，优雅地扩展到 5 个专用工具（`bash`、`read_file`、`write_file`、`edit_file`、`glob`）。关键设计是 **`TOOL_HANDLERS` 查表分发机制**——加一个工具只需在数组里加一条定义、在字典里加一行映射，主循环完全不用改。

### ✅ 已创建文件

`workdir/关于项目的介绍.md` — 已包含上述详细回答。

[36ms02 >> [0m

```

[代码](../s02_tool_use/code.py)
