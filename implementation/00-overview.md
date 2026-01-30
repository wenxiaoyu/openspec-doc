# OpenSpec 实现原理深度解析

## 文档概览

本系列文档深入讲解 OpenSpec 的核心实现原理，适合希望理解系统架构、贡献代码或构建类似系统的开发者。

## 文档结构

1. **[架构总览](./01-architecture.md)** - 系统整体架构和设计哲学
2. **[Artifact Graph 系统](./02-artifact-graph.md)** - 核心依赖管理引擎
3. **[Schema 系统](./03-schema-system.md)** - 工作流定义和解析
4. **[状态管理](./04-state-management.md)** - 基于文件系统的状态跟踪
5. **[指令生成系统](./05-instruction-generation.md)** - 为 AI 生成结构化指令
6. **[工作流命令](./06-workflow-commands.md)** - 用户交互层实现
7. **[解析系统](./07-parsing-system.md)** - Markdown 文档解析
8. **[验证系统](./08-validation-system.md)** - 多层验证机制
9. **[多工具适配](./09-multi-tool-adapters.md)** - 适配器模式实现
10. **[技能生成](./10-skill-generation.md)** - AI 技能模板系统

## 前置知识

阅读本文档前，建议具备以下知识：

- TypeScript 基础
- Node.js 生态系统
- CLI 工具开发经验
- 图论基础（拓扑排序）
- 设计模式（适配器、模板方法）

## 代码约定

文档中的代码示例遵循以下约定：

```typescript
// 简化的伪代码，突出核心逻辑
function example() {
  // 实际实现可能更复杂
}
```

```typescript
// 实际代码片段（来自源码）
export class ArtifactGraph {
  // ...
}
```

## 快速导航

- 想了解整体架构？→ 从 [架构总览](./01-architecture.md) 开始
- 想了解核心算法？→ 直接看 [Artifact Graph](./02-artifact-graph.md)
- 想了解如何扩展？→ 查看 [Schema 系统](./03-schema-system.md) 和 [多工具适配](./09-multi-tool-adapters.md)
- 想了解 AI 集成？→ 阅读 [指令生成](./05-instruction-generation.md) 和 [技能生成](./10-skill-generation.md)
