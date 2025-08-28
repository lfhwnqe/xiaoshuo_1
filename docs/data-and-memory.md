# 数据与记忆（Data & Memory）

目标：确保长文本创作的链路中，关键信息可被准确提取、压缩、检索与更新，以支持跨章节的连续性与一致性。

## 数据对象

- `Blueprint` 初始化蓝图：`{ logline, worldDoc, characters[], styleGuide, outline }`
- `Chapter` 章节实体：`{ index, scenePlan, draftText, finalText, summary, createdAt, version }`
- `MemoryRecord` 记忆条目：`{ id, type, content, embedding?, refs, createdAt }`
  - type：`world|character|plot|chapter_summary|glossary|style_rule` 等
  - refs：引用路径（如章节索引/角色 ID），便于定向召回

## 存储布局（建议）

```
data/
  blueprint.json                 # 初始化蓝图（最新）
  blueprint.history.jsonl        # 历史版本（逐条追加）
  chapters/
    001.json                     # 第1章结构化数据
    002.json
  memory/
    records.jsonl                # 记忆条目（JSON Lines）
    embeddings.idx?              # 向量索引（可选/占位）
outputs/
  chapters/
    001.md                       # 第1章终稿
    002.md
```

## 记忆与检索流程

1) 写入（从步骤输出到 Memory）
- 在 `consistencyAndMemoryStep` 提取：
  - `chapterSummary`（事件摘要）
  - `memoryDelta`（角色状态变化、伏笔兑现、设定补充）
- 统一格式化为 `MemoryRecord` 追加写入，并（可选）计算嵌入存入索引。

2) 读取（上下文提取）
- 依据 `chapterIndex` / 目标场景，检索：
  - 最近 N 章摘要
  - 涉及角色的最新状态
  - 与当前大纲节点相关的设定片段
- 采用“层级压缩”策略（先大纲层再细节层），形成 `contextForChapter`。

## MemoryProvider 抽象（示意）

```ts
export interface MemoryProvider {
  add(records: MemoryRecord[]): Promise<void>;
  search(query: string, options?: { topK?: number; filter?: any }): Promise<MemoryRecord[]>;
  byRefs(refs: string[]): Promise<MemoryRecord[]>;
}
``;

默认实现：

- `JsonlMemoryProvider`：基于 `records.jsonl` 的简单检索；向量相似度可暂用 BM25/关键词匹配占位。
- 可替换为：Pinecone/Weaviate/pgvector 实现，保持接口兼容。

## 一致性校验要点

- 角色一致性：称谓、口吻、动机与人设不冲突；状态变化（受伤/关系变更）需在后续章节生效。
- 设定一致性：世界规则（魔法、科技、政治结构）不自相矛盾；术语与时间线统一。
- 风格一致性：视角/时态/语域与 `styleGuide` 对齐；句式/标点/专有名词风格统一。

## 版本与回滚

- 每次蓝图/章节变更追加到 `.history.jsonl`；
- 输出终稿（Markdown）保存版本头（runId、stepId、模型/提示摘要）。

