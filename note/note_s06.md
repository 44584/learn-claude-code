这一节讲的是 subagent, 主旨是 大任务拆小，让小任务有干净的上下文。

开一个独立的子进程，给它一个独立的消息列表，让它专心做一件事，然后只把最终答案返回给Parent（但文件系统上的改动保留）。subagent的工具受限，比如不能再spawn子进程了。

本章将spawn一个subagent的能力封装为一个工具。

subagent设计注意事项：

- 上下文隔离：subagent的中间过程不干扰agent
- 只传回结论
- 禁止递归
- 遵守安全策略

```bash

(learn-claude-code) gkunix@laptopGK:~/workspace/learn-claude-code$ python ./s06_subagent/code.py
s06: Subagent — spawn sub-agents with fresh context, summary only
Type a question, press Enter. Type q to quit.

[36ms06 >> [0mUse a subtask to find what testing framework this project uses
[HOOK] UserPromptSubmit: working in /home/gkunix/workspace/learn-claude-code
[HOOK] task

[Subagent spawned]
        Task: Investigate the project at /home/gkunix/workspace/learn-claude-code to determine what testing framework(s) it uses. Look for:
1. Configuration files (package.json, pyproject.toml, pom.xml, build.gradle, Cargo.toml, go.mod, etc.)
2. Test configuration files (jest.config, vitest.config, karma.conf, .mocharc, pytest.ini, tox.ini, etc.)
3. Any test files and their extensions/imports
4. Any CI/CD config files that reference test runners
5. Lock files or dependency manifests

Report back with the testing framework name(s) found and the evidence for each.
[HOOK] glob
  [sub] glob: /home/gkunix/workspace/learn-claude-code/s15_agent_teams
/home/gkunix/workspace/learn-claude-code/re
[HOOK] glob
  [sub] glob: /home/gkunix/workspace/learn-claude-code/.gitignore
/home/gkunix/workspace/learn-claude-code/.venv
/
[HOOK] read_file
  [sub] read_file: anthropic>=0.25.0
python-dotenv>=1.0.0
pyyaml>=6.0
[HOOK] read_file
  [sub] read_file: # Byte-compiled / optimized / DLL files
__pycache__/
*.py[codz]
*$py.class

# C extensions
*.so

# D
[HOOK] read_file
  [sub] read_file: # API Key (required)
# Get yours at: https://console.anthropic.com/
ANTHROPIC_API_KEY=sk-ant-xxx

#
[HOOK] glob
  [sub] glob: (no matches)
[HOOK] glob
  [sub] glob: /home/gkunix/workspace/learn-claude-code/.github/workflows/ci.yml
/home/gkunix/workspace/learn-claud
[HOOK] glob
  [sub] glob: /home/gkunix/workspace/learn-claude-code/skills/code-review/SKILL.md
/home/gkunix/workspace/learn-cl
[HOOK] read_file
  [sub] read_file: name: Test

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  python-sm
[HOOK] read_file
  [sub] read_file: name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build:

[HOOK] bash
  [sub] bash: total 36
drwxr-xr-x  2 gkunix gkunix 4096 Jul 14 16:28 .
drwxr-xr-x 32 gkunix gkunix 4096 Jul 16 16:
[HOOK] glob
  [sub] glob: (no matches)
[HOOK] glob
  [sub] glob: (no matches)
[HOOK] glob
  [sub] glob: (no matches)
[HOOK] glob
  [sub] glob: (no matches)
[HOOK] glob
  [sub] glob: (no matches)
[HOOK] glob
  [sub] glob: (no matches)
[HOOK] glob
  [sub] glob: /home/gkunix/workspace/learn-claude-code/tests/test_agents_smoke.py
/home/gkunix/workspace/learn-cla
[HOOK] glob
  [sub] glob: (no matches)
[HOOK] read_file
  [sub] read_file: from __future__ import annotations

from pathlib import Path
import py_compile

import pytest


ROOT
[HOOK] read_file
  [sub] read_file: import importlib.util
import os
import sys
import tempfile
import types
import unittest
from pathlib
[HOOK] read_file
  [sub] read_file: import importlib.util
import os
import sys
import tempfile
import types
import unittest
from pathlib
[HOOK] read_file
  [sub] read_file: import importlib.util
import os
import sys
import tempfile
import types
import unittest
from pathlib
[HOOK] glob
  [sub] glob: /home/gkunix/workspace/learn-claude-code/web/package.json
[HOOK] glob
  [sub] glob: /home/gkunix/workspace/learn-claude-code/web/package-lock.json
[HOOK] read_file
  [sub] read_file: {
  "name": "web",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "extract": "tsx scrip
[HOOK] glob
  [sub] glob: (no matches)
[HOOK] glob
  [sub] glob: /home/gkunix/workspace/learn-claude-code/.vscode/settings.json
[HOOK] glob
  [sub] glob: /home/gkunix/workspace/learn-claude-code/CONTRIBUTING.md
[HOOK] read_file
  [sub] read_file: # Contributing

This is a curated teaching repository. Each chapter (`sNN_topic/`) builds one idea o
[HOOK] read_file
  [sub] read_file: {
        "python-envs.defaultEnvManager": "ms-python.python:venv",
        "python-envs.defaultPackageManager": "
[Subagent done]
[HOOK] Stop: session used 1 tool calls
## Testing Framework Analysis

The project uses **two Python testing frameworks**:

### 1. **pytest** (primary runner)
- CI/CD config (`.github/workflows/test.yml`) runs tests via `python -m pytest tests -q`
- The file `tests/test_agents_smoke.py` directly imports `pytest` and uses `@pytest.mark.parametrize`

### 2. **unittest** (Python standard library)
- Three test files use `unittest` directly:
  - `tests/test_todo_write_string_input.py`
  - `tests/test_compaction_tool_pairs.py`
  - `tests/test_s_full_background.py`
- They import `unittest`, inherit from `unittest.TestCase`, and use `unittest.main()` guards
- These run seamlessly under `pytest` since pytest is compatible with `unittest.TestCase` classes

### Web subproject
- **No testing framework** — `web/package.json` has no test-related dependencies or scripts, and CI only runstype checking and build.

[36ms06 >> [0mq

```

[代码](../s06_subagent/code.py)
