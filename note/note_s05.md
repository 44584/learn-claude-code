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
