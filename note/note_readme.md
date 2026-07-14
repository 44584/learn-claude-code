这个教程着重将Agent差分成为Agency和Harness两个部分，认为Agency来自LLM的模型训练。Harness是教程的重点。
对Agent给出的定义是“一个通过训练学会了行动的模型，加上让它能在特定环境中工作的基础设施。”。

Agency是感知、推理、行动的能力，来自神经网络的参数。

几个里程碑：
|年代| Agent| Backbone|
|---|---|-----|
|2013| DQN Atari| CNN​|
|2019| OpenAI Five| LSTM (RNN)|​
|2019| AlphaStar |CNN + Transformer + LSTM|​
|2019/21| 绝悟 |LSTM → +Attention​|
|2024-| LLM Agent |Transformer​|

### Harness

```
Harness = Tools + Knowledge + Observation + Action Interfaces + Permissions

    Tools:          文件读写、Shell、网络、数据库、浏览器
    Knowledge:      产品文档、领域资料、API 规范、风格指南
    Observation:    git diff、错误日志、浏览器状态、传感器数据
    Action:         CLI 命令、API 调用、UI 交互
    Permissions:    沙箱隔离、审批流程、信任边界
```

#### Harness工程的任务

- 实现工具：给Agent“双手”
- 策划知识：给Agent领域专长
- 管理上下文：记忆功能
- 控制权限：给Agent设立边界
- 收集任务过程数据

### 以Claude Code为例讲解Harness

```
Claude Code = 一个 agent loop
            + 工具 (bash, read, write, edit, glob, grep, browser...)
            + 按需 skill 加载
            + 上下文压缩
            + 子 agent 派生
            + 带依赖图的任务系统
            + 异步邮箱的团队协调
            + worktree 隔离的并行执行
            + 权限治理
```

### 教程目录

```
s01   "One loop & Bash is all you need" — 一个工具 + 一个循环 = 一个 Agent
s02   "加一个工具, 只加一个 handler" — 循环不用动, 新工具注册进 dispatch map 就行
s03   "先划边界, 再给自由" — 先判断操作能不能做，要不要问用户
s04   "挂在循环上, 不写进循环里" — 在工具前后留插口，不改主循环也能扩展
s05   "没有计划的 agent 走哪算哪" — 先列步骤再动手, 完成率翻倍
s06   "大任务拆小, 每个小任务干净的上下文" — 子 Agent 自己干活，只把结果带回来
s07   "用到时再加载, 别全塞 prompt 里" — 技能先列目录，用到时再展开
s08   "上下文总会满, 要有办法腾地方" — 四层压缩策略, 便宜的先跑贵的后跑
s09   "记住该记的, 忘掉该忘的" — 三个子系统: 筛选、提取、整理
s10   "prompt 是组装出来的, 不是写死的" — 分段 + 按需拼接
s11   "错误不是终点, 是重试的起点" — 出错时会重试、腾空间、换路子
s12   "大目标拆成小任务, 排好序, 持久化" — 文件持久化的任务图, 多 agent 协作的基础
s13   "慢操作丢后台, agent 继续思考" — 后台线程跑命令, 完成后注入通知
s14   "定时触发, 不需要人推" — 按时间自动触发任务
s15   "一个搞不定, 组队来" — 持久化队友 + 异步邮箱
s16   "队友之间要有约定" — 用固定的请求-回复格式沟通
s17   "队友自己看板, 有活就认领" — 不需要领导逐个分配, 自组织
s18   "各干各的目录, 互不干扰" — 任务管目标, worktree 管目录, 按 ID 绑定
s19   "能力不够? 插上 MCP" — 把外部工具接进同一个工具池
s20   "机制很多，循环一个" — 前面所有机制回到一个完整 harness
```