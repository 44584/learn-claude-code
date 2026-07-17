这一节讲TodoWrite，_看起来像是Plan-and-Execute_。

提出的问题是：提示词影响力被工具结果稀释。

提出的解决方案是： TodoWrite工具 + reminder机制。

这里的TodoWrite工具，操作的是 进程内存中的一个列表；_感觉原理和Plan-and-Execute一致，但是这里是通过工具操作_。

Nag reminder，模型连续 3 轮没调 todo_write 时，自动注入一条提醒

看完代码，发现这个TodoWrite工具有些不同。
TodoWrite工具在定义 schema json 时，要求模型给出的input参数就是 todo list 本身；
然后对于模型输出的工具调用，核心只是用工具调用的input覆盖内存中的CURRENT_TODOS。

代码中并没有显式地将CURRENT_TODOS追加到messages中，但是提出CURRENT_TODOS的assistant的content中，已经包含信息了，这很巧妙。

运行了代码，对于同一个任务，会分成几个步骤，任务有时一次性被完成，有时分步完成。可能是因为任务过于简单。

```shell
(learn-claude-code) gkunix@laptopGK:~/workspace/learn-claude-code$ python ./s05_todo_write/code.py
s05: TodoWrite — plan before execute, nag if you forget
Type a question, press Enter. Type q to quit.

[36ms05 >> [0mRefactor s05_todo_write/example/hello.py: add type hints, docstrings, and a main guard
[HOOK] UserPromptSubmit: working in /home/gkunix/workspace/learn-claude-code
TOOL: read_file({'path': '/home/gkunix/workspace/learn-claude-code/s05_todo_write/example/hello.py'})
[HOOK] read_file
TOOL: todo_write({'todos': [{'content': 'Add type hints to greet function', 'status': 'in_progress'}, {'content': 'Add docstring to greet function', 'status': 'pending'}, {'content': 'Add main guard', 'status': 'pending'}]})
[HOOK] todo_write

## Current Tasks
  [▸] Add type hints to greet function
  [ ] Add docstring to greet function
  [ ] Add main guard
TOOL: edit_file({'path': '/home/gkunix/workspace/learn-claude-code/s05_todo_write/example/hello.py', 'old_text': 'def greet(name):\n    message = "Hello, " + name\n    print(message)\n\n\ngreet("Claude")', 'new_text': 'def greet(name: str) -> None:\n    """Print a greeting message for the given name.\n\n    Args:\n        name: The person to greet.\n    """\n    message = "Hello, " + name\n    print(message)\n\n\nif __name__ == "__main__":\n    greet("Claude")'})
[HOOK] edit_file
TOOL: read_file({'path': '/home/gkunix/workspace/learn-claude-code/s05_todo_write/example/hello.py'})
[HOOK] read_file
TOOL: todo_write({'todos': [{'content': 'Add type hints to greet function', 'status': 'completed'}, {'content': 'Add docstring to greet function', 'status': 'completed'}, {'content': 'Add main guard', 'status': 'completed'}]})
[HOOK] todo_write

## Current Tasks
  [✓] Add type hints to greet function
  [✓] Add docstring to greet function
  [✓] Add main guard
[HOOK] Stop: session used 5 tool calls
All three refactoring tasks are complete. Here's what changed:

| Change | Before | After |
|---|---|---|
| **Type hints** | `def greet(name)` | `def greet(name: str) -> None` |
| **Docstring** | _(none)_ | Google-style docstring with `Args:` section |
| **Main guard** | `greet("Claude")` (unconditional) | Wrapped in `if __name__ == "__main__":` |

The script still runs identically when executed directly, but the main guard prevents it from auto-running when imported as a module by other code.

[36ms05 >> [0m
```

## 分析 TodoWrite 与 Plan-and-Execute 范式

为什么选择轻量的 TodoWrite 工具，而不是经典的 Plan-and-Execute 模式？

---

### 一、两种模式的对比

| 维度                   | Plan-and-Execute                                                              | learn-claude-code (TodoWrite)                                                     |
| ---------------------- | ----------------------------------------------------------------------------- | --------------------------------------------------------------------------------- |
| **规划生成时机**       | 任务开始时由 LLM 或外部模块生成一份静态列表                                   | Agent 在执行过程中**按需**调用 `todo_write` 创建/更新列表                         |
| **状态如何传递给 LLM** | 每次 LLM 调用前，由外部系统将**当前步骤 + 剩余任务**注入到 user/system 消息中 | 状态**隐含在 assistant 的历史中**（todo_write的调用json中，input就是一个todo列表），LLM 通过回顾自己之前的输出得知进度 |
| **状态维护者**         | 外部系统（如状态机、数据库）                                                  | LLM 自身（借助对话历史，调用todo_write工具得到新的计划状态）                      |
| **纠偏机制**           | 外部系统强制执行下一步                                                        | 通过 nag reminder（user 角色提醒）软性引导                                        |
| **Token 开销**         | 每次注入都会增加 token（即使任务列表很长）                                    | 仅在 LLM 主动更新时产生一次工具调用参数，后续不再重复                             |

---

### 二、learn-claude-code 不采用 Plan-and-Execute 的原因

#### 1. 保持 Agent 的自主性与灵活性

Plan-and-Execute 本质上是**外部强控**：这虽然减少了偏离风险，但也剥夺了 Agent 根据中间结果调整计划的能力。  
learn-claude-code 的设计者希望 Agent **自己决定何时规划、何时更新**。TodoWrite 只是一个可选的工具，Agent 可以在任务开始前调用，也可以在发现需要调整时再次调用。

#### 2. 避免上下文污染与 token 浪费

在 Plan-and-Execute 中，每轮对话都需要注入当前任务列表（例如 `Current step: 3/10 - Rename file foo.py`）。随着任务增多，这部分内容会反复占据上下文窗口，挤占真正有用的工具结果和推理过程。  
而 TodoWrite 的方案中，任务列表**只出现在 LLM 自己发出的 tool_use block 里**，后续对话不会重复出现。LLM 需要时可以通过回顾历史（或再次调用 todo_write 刷新）来获取状态。这种方式更节省 token，也更符合“让上下文自然流动”的理念。
