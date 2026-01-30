# OpenSpec 实现原理文档

深入讲解 OpenSpec 核心实现的技术文档系列。

## 📚 文档列表

### 已完成

1. **[概览](./00-overview.md)** - 文档导航和前置知识
2. **[架构总览](./01-architecture.md)** - 系统整体架构、模块职责、数据流
3. **[Artifact Graph 系统](./02-artifact-graph.md)** - 依赖图算法、拓扑排序、状态检测
4. **[Schema 系统](./03-schema-system.md)** - 工作流定义、解析优先级、自定义 Schema
5. **[状态管理](./04-state-management.md)** - 基于文件系统的状态、Glob 支持、进度跟踪

### 待完成

6. **指令生成系统** - 模板加载、上下文注入、依赖信息收集
7. **工作流命令** - new/continue/apply 命令实现、用户交互
8. **解析系统** - Markdown 解析、Requirement 提取、Delta 处理
9. **验证系统** - 多层验证、Schema 验证、业务规则
10. **多工具适配** - 适配器模式、命令生成、工具特定格式
11. **技能生成** - AI 技能模板、指令结构、工作流集成

## 🎯 快速导航

### 按主题

**核心算法**
- [Artifact Graph](./02-artifact-graph.md) - 依赖图和拓扑排序
- [状态管理](./04-state-management.md) - 状态检测和跟踪

**扩展性**
- [Schema 系统](./03-schema-system.md) - 自定义工作流
- 多工具适配 - 支持新的 AI 工具

**AI 集成**
- 指令生成系统 - 为 AI 生成指令
- 技能生成 - AI 技能模板

**数据处理**
- 解析系统 - Markdown 文档解析
- 验证系统 - 文档验证

### 按角色

**想了解整体架构？**
→ [架构总览](./01-architecture.md)

**想贡献代码？**
→ [Artifact Graph](./02-artifact-graph.md) + [Schema 系统](./03-schema-system.md)

**想自定义工作流？**
→ [Schema 系统](./03-schema-system.md)

**想集成新工具？**
→ 多工具适配

**想了解 AI 集成？**
→ 指令生成系统 + 技能生成

## 📖 阅读建议

### 初学者路径

1. [概览](./00-overview.md) - 了解文档结构
2. [架构总览](./01-architecture.md) - 理解系统设计
3. [Artifact Graph](./02-artifact-graph.md) - 掌握核心算法
4. [Schema 系统](./03-schema-system.md) - 学习工作流定义

### 进阶路径

1. [状态管理](./04-state-management.md) - 深入状态系统
2. 指令生成系统 - 理解 AI 集成
3. 解析系统 - 了解文档处理
4. 验证系统 - 掌握质量保证

### 扩展开发路径

1. [Schema 系统](./03-schema-system.md) - 自定义工作流
2. 多工具适配 - 添加新工具支持
3. 技能生成 - 创建 AI 技能

## 🔧 代码示例

每个文档都包含：
- 核心代码片段
- 实际使用示例
- 最佳实践建议

## 🤝 贡献

发现文档问题或有改进建议？
- 提交 Issue
- 发起 Pull Request
- 加入 Discord 讨论

## 📝 文档约定

### 代码块标记

```typescript
// 简化的伪代码
function example() {
  // 突出核心逻辑
}
```

```typescript
// 实际源码片段
export class ArtifactGraph {
  // 来自 src/core/artifact-graph/graph.ts
}
```

### 图表

使用 ASCII 图表展示架构和流程：

```
┌─────────────┐
│   Component │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│   Another   │
└─────────────┘
```

## 🔗 相关资源

- [用户文档](../README.md)
- [API 参考](../../src/)
- [测试用例](../../test/)
- [示例项目](../../examples/)

## 📅 更新日志

- 2025-01-30: 创建文档系列，完成前 5 章
