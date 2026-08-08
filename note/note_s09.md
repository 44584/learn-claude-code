这一节讲的是记忆功能。 文件仓库 + 索引 + 按需加载，跨压缩、跨会话

- 每个记忆对应一个.md文件，yaml formatter(name, description, type)
- 索引文件MEMORY.md，一行一个链接指向记忆文件
- 索引常驻SYSTEM PROMPT，记忆按需注入
  - 每次用户请求开始时，把最近对话和记忆catalog（name和description）发给LLM做轻量的side-query，选出相关name（name.md也就是文件名），再读取文件中的内容，临时注入到当前的 user turn

- 每轮对话结束后，写入记忆
  - 使用压缩前的快照自动提取记忆（因为压缩先于记忆提取）
  - 要求 LLM 返回 {name, type, description, body} 的 JSON 数组，
  - 在提示词中给出现有的记忆的catalog，要求只给出确实不存在的记忆

- 整理记忆：低频合并去重
  - 每轮对话结束后检查，文件数目超过阈值（10）时触发；
  - 让LLM完成去重、合并矛盾、淘汰过时

- 四类记忆（对应记忆文件中的 type）
  - user 你是谁
  - feedback 怎么做事
  - project 正在发生什么
  - reference 东西在哪里找


Memory适合保存什么？\
保存跨对话依然有效的信息。比如用户偏好，反复出现的反馈、项目背景、常用入口、排查线索。\
关注以后还会用到什么并通过 索引+按需加载，把信息带回当前对话。
