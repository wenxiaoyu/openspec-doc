# 实现文档快速索引

## 按主题查找

### 核心算法

| 主题 | 文档 | 关键内容 |
|------|------|----------|
| 依赖图管理 | [Artifact Graph](./02-artifact-graph.md) | 拓扑排序、依赖检测、循环检测 |
| 状态检测 | [状态管理](./04-state-management.md) | 文件系统状态、Glob 支持 |
| 指令生成 | [指令生成](./05-instruction-generation.md) | 模板加载、上下文注入 |

### 系统架构

| 主题 | 文档 | 关键内容 |
|------|------|----------|
| 整体架构 | [架构总览](./01-architecture.md) | 模块职责、数据流、设计哲学 |
| 工作流定义 | [Schema 系统](./03-schema-system.md) | Schema 格式、解析优先级 |

### 扩展开发

| 主题 | 文档 | 关键内容 |
|------|------|----------|
| 自定义工作流 | [Schema 系统](./03-schema-system.md#自定义-schema) | 创建 Schema、模板定义 |
| 项目配置 | [指令生成](./05-instruction-generation.md#项目配置) | config.yaml 格式 |

## 按代码位置查找

### src/core/artifact-graph/

| 文件 | 文档 | 说明 |
|------|------|------|
| `types.ts` | [Artifact Graph](./02-artifact-graph.md#核心概念) | 类型定义 |
| `graph.ts` | [Artifact Graph](./02-artifact-graph.md#核心类artifactgraph) | 依赖图实现 |
| `schema.ts` | [Artifact Graph](./02-artifact-graph.md#schema-解析) | Schema 解析 |
| `state.ts` | [状态管理](./04-state-management.md#状态检测系统) | 状态检测 |
| `resolver.ts` | [Schema 系统](./03-schema-system.md#schema-解析器) | Schema 解析器 |
| `instruction-loader.ts` | [指令生成](./05-instruction-generation.md#指令生成流程) | 指令加载 |

### src/commands/workflow/

| 文件 | 说明 |
|------|------|
| `new-change.ts` | 创建新变更 |
| `status.ts` | 状态查询 |
| `instructions.ts` | 指令生成命令 |

### src/core/parsers/

| 文件 | 说明 |
|------|------|
| `markdown-parser.ts` | Markdown 解析 |
| `change-parser.ts` | Change 文档解析 |

## 按功能查找

### 创建变更

1. [架构总览 - 数据流](./01-architecture.md#数据流) - 整体流程
2. [状态管理 - 元数据](./04-state-management.md#状态持久化) - .openspec.yaml

### 生成指令

1. [指令生成 - 主函数](./05-instruction-generation.md#指令生成流程) - generateInstructions
2. [指令生成 - 模板加载](./05-instruction-generation.md#模板加载) - loadTemplate
3. [指令生成 - 项目配置](./05-instruction-generation.md#项目配置) - config.yaml

### 检测状态

1. [状态管理 - 状态检测](./04-state-management.md#状态检测系统) - detectCompleted
2. [状态管理 - Glob 支持](./04-state-management.md#glob-模式支持) - 模式匹配
3. [状态管理 - 格式化](./04-state-management.md#状态格式化) - formatChangeStatus

### 自定义工作流

1. [Schema 系统 - 自定义](./03-schema-system.md#自定义-schema) - 创建步骤
2. [Schema 系统 - 最佳实践](./03-schema-system.md#schema-最佳实践) - 设计建议
3. [指令生成 - 模板](./05-instruction-generation.md#模板加载) - 模板系统

## 按学习路径查找

### 初学者路径

1. [概览](./00-overview.md) - 了解文档结构
2. [架构总览](./01-architecture.md) - 理解系统设计
3. [Artifact Graph](./02-artifact-graph.md) - 掌握核心算法
4. [Schema 系统](./03-schema-system.md) - 学习工作流定义

### 进阶路径

1. [状态管理](./04-state-management.md) - 深入状态系统
2. [指令生成](./05-instruction-generation.md) - 理解 AI 集成

### 扩展开发路径

1. [Schema 系统 - 自定义](./03-schema-system.md#自定义-schema) - 自定义工作流
2. [指令生成 - 项目配置](./05-instruction-generation.md#项目配置) - 项目定制

## 常见问题快速查找

### 如何创建自定义工作流？

→ [Schema 系统 - 自定义 Schema](./03-schema-system.md#自定义-schema)

### 如何理解依赖图算法？

→ [Artifact Graph - 核心算法详解](./02-artifact-graph.md#核心算法详解)

### 状态是如何检测的？

→ [状态管理 - 状态检测系统](./04-state-management.md#状态检测系统)

### 指令是如何生成的？

→ [指令生成 - 指令生成流程](./05-instruction-generation.md#指令生成流程)

### 如何添加项目特定的规则？

→ [指令生成 - 项目配置](./05-instruction-generation.md#项目配置)

### Glob 模式是如何工作的？

→ [状态管理 - Glob 模式支持](./04-state-management.md#glob-模式支持)

### Schema 解析优先级是什么？

→ [Schema 系统 - 三层解析优先级](./03-schema-system.md#schema-解析器)

## 代码示例快速查找

### 使用 Artifact Graph

```typescript
// 创建依赖图
const graph = ArtifactGraph.fromYaml('schema.yaml');

// 获取构建顺序
const buildOrder = graph.getBuildOrder();

// 查找可执行工件
const next = graph.getNextArtifacts(completed);
```

→ [Artifact Graph - 使用示例](./02-artifact-graph.md#使用示例)

### 检测状态

```typescript
// 检测完成状态
const completed = detectCompleted(graph, changeDir);

// 加载变更上下文
const context = loadChangeContext(projectRoot, changeName);

// 格式化状态
const status = formatChangeStatus(context);
```

→ [状态管理 - 使用示例](./04-state-management.md#使用示例)

### 生成指令

```typescript
// 生成工件指令
const instructions = generateInstructions(context, 'proposal');

// 生成 Apply 指令
const applyInstructions = await generateApplyInstructions(
  projectRoot,
  changeName
);
```

→ [指令生成 - 使用示例](./05-instruction-generation.md#使用示例)

## 图表快速查找

### 系统架构图

→ [架构总览 - 系统架构图](./01-architecture.md#系统架构图)

### 依赖图可视化

→ [Artifact Graph - 依赖图可视化](./02-artifact-graph.md#依赖图可视化)

### 数据流图

→ [架构总览 - 数据流](./01-architecture.md#数据流)

## 性能相关

### 时间复杂度

→ [Artifact Graph - 性能考虑](./02-artifact-graph.md#性能考虑)

### 优化策略

→ [状态管理 - 性能优化](./04-state-management.md#性能优化)

## 设计模式

### 工厂模式

→ [架构总览 - 设计模式](./01-architecture.md#设计模式)

### 适配器模式

→ [架构总览 - 设计哲学](./01-architecture.md#设计哲学)

### 策略模式

→ [架构总览 - 设计模式](./01-architecture.md#设计模式)

---

**返回：** [文档首页](./README.md) | [总结](../implementation-summary.md)
