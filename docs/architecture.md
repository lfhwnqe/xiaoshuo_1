# 系统架构与工程约定

本项目以 Mastra 工作流为核心编排层，围绕“多智能体协作 + 记忆与一致性”的目标构建。下述为逻辑分层与关键组件设计。

## 分层架构

1. 表达与编排层（Workflows/Steps）
   - 使用 Mastra 的 `createWorkflow/createStep`（vNext）建模：
     - 初始化工作流：世界观/角色/风格/大纲。
     - 章节工作流：上下文提取 → 章节梗概 → 正文生成 → 一致性校验与记忆更新。
   - 控制流：`.then()` 串行、`.branch()` 条件、`.parallel()` 并行、`.while()` 或 `.until()` 循环迭代。

2. 智能体与工具层（Agents/Tools）
   - 角色：设定生成器、角色策划师、文风定制、连贯性审阅、上下文管理、章节策划师、章节写手、风格检查等。
   - 工具（Tool）：检索记忆、向量相似度搜索、章节索引、风格规则校验、术语表维护。
   - 工具作为 Step 注入：`createStep(myTool)` 可直接接至工作流。

3. 记忆与知识层（Memory/Knowledge）
   - 长程记忆：章节摘要、角色状态变更、设定补充；支持向量检索。
   - 知识库：世界观文档、风格指南、术语表；统一由 MemoryProvider 管理。
   - 可替换后端：本地 JSON/SQLite → 外部向量库（Pinecone/Weaviate/pgvector）。

4. 存储与状态层（Storage/State）
   - 运行状态：工作流 runId、步骤状态（CREATED/RUNNING/SUSPENDED/COMPLETED/FAILED）。
   - 产物存档：初始化蓝图、章节梗概、章节初稿/终稿、评审记录、回滚点。
   - 建议：以 `data/` 保存结构化 JSON；以 `outputs/` 保存 Markdown 正文与快照。

5. 接入与触发层（API/CLI/Jobs）
   - 本地 CLI：开发阶段直接启动/指定章节生成。
   - Web/API：Hono（`@mastra/deployer/server`）暴露工作流；支持外部事件触发。
   - 计划任务：定时生成/校验、批量重试。

## 目录结构（建议）

```
.
├─ src/
│  ├─ workflows/
│  │  ├─ init/                 # 初始化工作流与步骤
│  │  ├─ chapter/              # 章节工作流与子工作流
│  │  └─ common/               # 通用步骤（检索、校验、合成等）
│  ├─ agents/                  # Agent 定义与组合（如需要）
│  ├─ tools/                   # 工具：记忆检索、风格检查、术语表等
│  ├─ memory/                  # MemoryProvider 抽象与实现
│  ├─ storage/                 # 文件/DB 持久化适配层
│  ├─ server/                  # Hono/HTTP 接入与 Mastra 实例注册
│  ├─ config/                  # 配置与提示模板、风格指南模板
│  └─ index.ts                 # 入口与启动
├─ data/                       # 结构化中间数据（JSON）
├─ outputs/                    # 生成的正文/快照（Markdown）
├─ docs/                       # 文档（当前目录）
└─ .env                        # 环境变量（模型密钥等）
```

## Mastra 工作流建模要点

- 类型与校验：输入/输出使用 `zod` 定义，便于在步骤间映射与验证。
- 变量映射：使用 `variables` 将上游结果注入下游 Step 输入（含嵌套路径）。
- 嵌套工作流：章节工作流可复用“上下文提取子工作流”等，作为 Step 插入父工作流。
- 状态观察：`watch` 监听运行状态变更，输出进度与错误；`resume` 支持挂起步骤恢复。
- 控制与重试：`.branch()` 路由文本质量；`.while()` or `.until()` 配合“质量门”进行自动修订。

## 可观测性与回溯

- 日志与追踪：统一 Logger（如 Pino），记录 runId、stepId、输入输出摘要与耗时。
- 产物与快照：关键节点输出固化到 `outputs/`；记录 prompt 与模型版本，保证可重现。
- 回滚点：初始化蓝图、大纲、每章终稿均保存版本号与引用依赖，便于回退与重写。

