这一节将hook，挂在循环上，不写在循环里。hook在工具执行前后注入拓展逻辑。

定义了四种event：

| 事件             | 触发时机                    | 典型用途                            |
| ---------------- | --------------------------- | ----------------------------------- |
| UserPromptSubmit | 用户输入提交后、进入 LLM 前 | 输入验证、注入上下文                |
| PreToolUse       | 工具执行前                  | 权限检查、日志记录                  |
| PostToolUse      | 工具执行后                  | 副作用（自动 git add 等）、输出检查 |
| Stop             | 循环即将退出时              | 收尾清理（CC 还支持强制续跑）       |

```python
HOOKS = {
    "UserPromptSubmit": [],
    "PreToolUse": [],
    "PostToolUse": [],
    "Stop": [],
}

def register_hook(event: str, callback):
    """将一个函数注册到指定的event中"""
	HOOKS[event].append(callback)

def trigger_hooks(event: str, *args):
	"""使用参数，执行指定事件列表的所有函数"""
    for callback in HOOKS[event]:
        result = callback(*args)
        if result is not None:   # 返回值 ≠ None → hook 说"停"
            return result
    return None
```

感觉这是Agent的一种设计模式，保证可拓展性。
