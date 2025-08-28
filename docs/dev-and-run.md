# 开发与运行指南（Dev & Run）

本指南说明如何在本地开发、调试与运行工作流，并给出人机协作（HITL）与日志追踪建议。

## 环境准备

- Node.js LTS（>=18）
- 包管理器：pnpm/npm/yarn（任选其一）
- 模型与服务密钥：`.env` 管理（如 `OPENAI_API_KEY`）

## 依赖与脚手架（建议）

```bash
# 安装依赖（示例）
pnpm add @mastra/core @mastra/deployer @hono/node-server zod pino
```

目录参考见《architecture.md》。建议在 `src/index.ts` 构建 Mastra 实例，并注册工作流：

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
```

## 启动与执行

### 命令行脚本（建议）

```ts
// scripts/run-chapter.ts（示意）
import { mastra } from "../src";

const run = await mastra.getWorkflow("chapter-workflow").createRunAsync();
const res = await run.start({ inputData: { chapterIndex: Number(process.argv[2] || 1) } });
console.dir(res, { depth: null });
```

### 本地 HTTP 服务（可选）

```ts
import { mastra } from "./";
import { serve } from "@hono/node-server";
import { createHonoServer, getToolExports } from "@mastra/deployer/server";
import { tools } from "#tools"; // 若有工具

const app = await createHonoServer(mastra, { tools: getToolExports(tools) });
serve({ fetch: app.fetch, port: 3000 });
```

## 人在环（HITL）与挂起/恢复

- 关键节点可设计为 `SUSPENDED` 状态：保存上下文后等待人工审阅；审阅通过后 `resume`。
- 前端/CLI 提供“approve/reject”接口，拒绝时进入修订分支。

## 日志与追踪

- 使用 Pino（或同类）作为 Logger；记录 `runId/stepId`、输入输出摘要与耗时。
- Mastra `watch`：监听状态，输出实时进度；错误时固定格式上报。

## 测试建议

- 单元测试：
  - 步骤级（Step）纯函数/工具函数可直接 mock 输入验证输出。
  - 提示模板与风格规则：使用“金样本（golden）”对齐期望。
- 集成测试：
  - 子工作流（如 contextExtractionSubflow）在无模型调用下以假数据验证控制流与映射。

