这节讲的是 任务系统：

- 持久化到 `.task/`，可以CURD管理；
- 有状态；
- 有检查依赖（一个任务只有在它的依赖全部完成后，才可以被agent认领开始）；

每个任务被设计成Task数据结构，持久化为一个json文件。

```python
@data
class Task:
    id: str                 # timestamp + random hex 生成
    subject: str
    description, str
    status: str             # pending | in_progress | completed
    owner: str | None       # Agent 名（多 Agent 场景）
    blockedBy: list[str]    # 依赖的任务 ID 列表
```

- 任务管理：
  - `create_task`
    - 动作：创建任务对应的json文件
  - `can_start`
    - 条件：检查`blockedBy`列表成员是否都存在，且依赖项都`completed`
  - `claim_task`
    - 条件：`pending` 且 `can_start`
    - 动作：设置`owner`，且状态更新为`in_progress`
  - `complete_task`
    - 动作：设置`status`为`completed`，再找出现在`can_start`且是`pending`的任务
  - `get_task`

任务管理封装为tool，任务交给LLM管理维护。