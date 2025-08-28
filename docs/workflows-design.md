# 工作流设计（Mastra）

本文将 README 的两条主线流程转化为 Mastra 的工作流与步骤设计，并给出关键输入/输出、变量映射、控制流与错误恢复策略。

> 术语：Workflow（工作流）、Step（步骤）、vNext API（`createWorkflow/createStep`）。文中 TypeScript 片段为示意，便于落地实现。

## 工作流一：小说初始化（InitializationWorkflow）

目标：在正式写作前生成统一蓝图（世界观、角色档案、风格指南、剧情大纲）。

步骤拆解：

1) 核心概念与世界观设定（`worldbuildingStep`）
- 输入：用户初始题材/主题、任意参考资料。
- 输出：`{ logline, worldDoc }`（故事前提 + 世界观文档）。
- Agent 职责：设定生成器；必要时调用“头脑风暴”子流程产出候选并筛选。

2) 角色档案生成（`characterProfilesStep`）
- 输入：`worldDoc, logline`。
- 输出：`{ characters: Character[] , relations: Graph }`。
- Agent 职责：角色策划师 + 一致性检查者（校验冲突/重复）。

3) 文风与风格指南（`styleGuideStep`）
- 输入：`logline, worldDoc, characters, userStylePrefs`。
- 输出：`{ styleGuide, glossary? }`（视角/时态/语域/修辞偏好 + 示例）。
- Agent 职责：文风定制 + 风格检查者（后续复用）。

4) 故事大纲规划（`globalOutlineStep`）
- 输入：`logline, worldDoc, characters, styleGuide, chapterCount?`。
- 输出：`{ outline: OutlineByChapters }`（分章事件图）。
- Agent 职责：大纲策划师 + 连贯性审阅。

控制流示例：

```ts
import { createWorkflow, createStep } from "@mastra/core/workflows";
import { z } from "zod";

export const initializationWorkflow = createWorkflow({
  id: "initialization-workflow",
  inputSchema: z.object({
    seed: z.string().describe("初始创意/题材"),
    userStylePrefs: z.string().optional(),
    chapterCount: z.number().optional(),
  }),
  outputSchema: z.object({
    logline: z.string(),
    worldDoc: z.string(),
    characters: z.array(z.any()),
    styleGuide: z.string(),
    outline: z.any(),
  }),
})
  .then(worldbuildingStep)
  .then(characterProfilesStep)
  .then(styleGuideStep)
  .then(globalOutlineStep)
  .commit();
```

质量门与重试：

- 在 `characterProfilesStep`/`globalOutlineStep` 后设置 `.branch()`：若一致性检查失败，进入“修订子流程”，否则继续。
- 对 `styleGuideStep` 可采用 `.while()`：直至风格规则覆盖率/示例充分性达标。

## 工作流二：章节写作（ChapterWorkflow）

目标：基于初始化蓝图，循环生成章节内容，保持连续性与风格一致。

子工作流与步骤：

1) 上下文提取与理解（`contextExtractionSubflow`）
- 输入：`global blueprint (worldDoc, characters, styleGuide, outline) + memory (previous chapters summaries)`；章节编号。
- 输出：`{ contextForChapter }`（与本章强相关的前情、角色最新状态、伏笔）。
- Agent 职责：上下文管理 + 记忆检索（向量召回 + 章节索引）。

2) 章节梗概生成（`chapterPlanStep`）
- 输入：`outline[chapterN] + contextForChapter`。
- 输出：`{ scenePlan }`（场景级要点与事件顺序）。
- Agent 职责：章节策划师 + 一致性审核。

3) 章节正文写作（`chapterDraftStep`）
- 输入：`scenePlan + styleGuide + contextForChapter`。
- 输出：`{ draftText }`（章节初稿，长度约束可控）。
- Agent 职责：章节写手（可调用对话/描写子代理）。

4) 连续性校验与记忆更新（`consistencyAndMemoryStep`）
- 输入：`draftText + styleGuide + memory`。
- 输出：`{ finalText, chapterSummary, memoryDelta }`；将 `chapterSummary` & `memoryDelta` 写回记忆库。
- Agent 职责：连续性监察 + 风格检查 + 记忆管理。

控制流与修订回路：

```ts
export const chapterWorkflow = createWorkflow({
  id: "chapter-workflow",
  inputSchema: z.object({
    chapterIndex: z.number(),
  }),
  outputSchema: z.object({
    finalText: z.string(),
    chapterSummary: z.string(),
  }),
})
  .then(contextExtractionSubflow)
  .then(chapterPlanStep)
  .then(chapterDraftStep)
  .then(consistencyAndMemoryStep)
  .commit();

// 质量门路由（示意）：
// chapterDraftStep 之后：若风格/一致性评分 < 阈值，则回到 chapterPlanStep 或触发修订 step
```

分支与并行：

- `.branch()`：
  - 若上下文召回不足（召回分 < 阈值），进入“增强检索”分支（扩大窗口/改写查询）。
  - 若降雨（比喻自 Mastra 示例）→ 采用“室内/户外”不同创作策略（对应不同风格模板）。
- `.parallel()`：
  - 可并行生成不同视角片段，随后合成（synthesize step），但需严格一致性校验。

挂起与恢复：

- 审阅节点（HITL）：在 `chapterPlanStep` 或 `consistencyAndMemoryStep` 触发 `SUSPENDED`，待人工审核后 `resume`。

## 步骤 I/O 与变量映射范式

示例：将初始化大纲映射到章节策划步骤输入。

```ts
parentWorkflow
  .then(chapterPlanStep, {
    variables: {
      outlineNode: { step: "trigger", path: "outline[chapterIndex]" },
      contextForChapter: { step: contextExtractionSubflow, path: "context" },
    },
  })
```

## 错误处理与重试策略

- 输入校验：`zod` 的 `inputSchema/outputSchema` 保障契约。
- 业务错误：
  - 召回为空 → 退避重试（修改查询/放宽过滤），上限 N 次；
  - 风格偏差 → 调整提示模板或权重，进入 `.while()` 修订，限制迭代次数。
- 非幂等步骤：将上游输入指纹化（hash）并缓存结果，避免因网络/配额导致重复开销。

## 运行与编排（注册与启动）

```ts
import { Mastra } from "@mastra/core/mastra";
import { initializationWorkflow } from "./workflows/init";
import { chapterWorkflow } from "./workflows/chapter";

export const mastra = new Mastra({
  workflows: {
    initializationWorkflow,
    chapterWorkflow,
  },
});

// 运行（命令行/脚本）
const run = await mastra.getWorkflow("chapter-workflow").createRunAsync();
const result = await run.start({ inputData: { chapterIndex: 5 } });
```

