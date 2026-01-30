# OpenSpec 实现原理快速入门

5 分钟了解 OpenSpec 核心实现。

## 🎯 核心思想

OpenSpec 将文档创建过程建模为**依赖图**，自动管理工件之间的依赖关系。

```
    proposal (提案)
    /      \
specs    design (并行)
    \      /
     tasks (任务)
```

## 🔑 三大核心系统

### 1. Artifact Graph（依赖图）

**作用：** 管理工件依赖关系

**核心算法：**
```typescript
// 拓扑排序 - 计算构建顺序
graph.getBuildOrder()
// → ['proposal', 'design', 'specs', 'tasks']

// 查找可执行工件
graph.getNextArtifacts(completed)
// → ['design', 'specs']  // 当 proposal 完成时
```

**详细文档：** [Artifact Graph 系统](./02-artifact-graph.md)

### 2. Schema 系统

**作用：** 声明式定义工作流

**示例：**
```yaml
name: spec-driven
artifacts:
  - id: proposal
    generates: proposal.md
    requires: []
  
  - id: specs
    generates: "specs/**/*.md"  # 支持 glob
    requires: [proposal]
```

**详细文档：** [Schema 系统](./03-schema-system.md)

### 3. 状态管理

**作用：** 基于文件系统跟踪状态

**原理：**
```typescript
// 工件完成 = 文件存在
detectCompleted(graph, changeDir)
// 检查: proposal.md 存在? specs/**/*.md 有文件?
```

**详细文档：** [状态管理](./04-state-management.md)

## 📊 数据流示例

### 创建变更

```
openspec new change add-auth
  ↓
创建目录: openspec/changes/add-auth/
  ↓
写入元数据: .openspec.yaml
  ↓
完成
```

### 生成指令

```
openspec instructions proposal --change add-auth
  ↓
加载 Schema → 构建依赖图 → 检测状态
  ↓
加载模板 → 读取配置 → 收集依赖
  ↓
输出结构化指令（给 AI）
```

### 检测状态

```
openspec status --change add-auth
  ↓
检测文件存在性
  ↓
输出:
  ✓ proposal
  ✓ specs
  → design
  ○ tasks (blocked by: design)
```

## 🏗️ 架构层次

```
CLI 层 (Commander.js)
  ↓
命令层 (用户交互)
  ↓
核心引擎层
  ├─ Artifact Graph (依赖管理)
  ├─ Instruction Loader (指令生成)
  └─ Validator (验证)
  ↓
数据层 (文件系统)
```

## 💡 关键设计

### 1. 状态即文件

```
工件完成状态 = 文件是否存在
变更元数据 = .openspec.yaml
任务进度 = tasks.md 中的 checkbox
```

**优势：** 简单、Git 友好、无需数据库

### 2. 声明式工作流

```yaml
# 声明依赖关系，引擎自动处理
artifacts:
  - id: tasks
    requires: [specs, design]
```

**优势：** 灵活、可扩展、用户可自定义

### 3. 结构化指令

```xml
<artifact id="proposal">
  <context>项目背景（约束）</context>
  <rules>工件规则（约束）</rules>
  <template>输出结构</template>
  <dependencies>依赖信息</dependencies>
</artifact>
```

**优势：** AI 清楚知道什么是约束，什么是输出

## 🔧 核心 API

### Artifact Graph

```typescript
// 创建
const graph = ArtifactGraph.fromYaml('schema.yaml');

// 查询
graph.getBuildOrder()              // 构建顺序
graph.getNextArtifacts(completed)  // 可执行工件
graph.isComplete(completed)        // 是否完成
graph.getBlocked(completed)        // 阻塞工件
```

### 状态管理

```typescript
// 检测状态
const completed = detectCompleted(graph, changeDir);

// 加载上下文
const context = loadChangeContext(projectRoot, changeName);

// 格式化输出
const status = formatChangeStatus(context);
```

### 指令生成

```typescript
// 生成工件指令
const instructions = generateInstructions(context, artifactId);

// 生成 Apply 指令
const applyInstructions = await generateApplyInstructions(
  projectRoot,
  changeName
);
```

## 📁 代码位置

```
src/core/
├── artifact-graph/          # 依赖图系统
│   ├── graph.ts            # 核心算法
│   ├── schema.ts           # Schema 解析
│   ├── state.ts            # 状态检测
│   ├── resolver.ts         # Schema 解析器
│   └── instruction-loader.ts  # 指令生成
├── parsers/                # 解析器
├── validation/             # 验证器
└── command-generation/     # 命令生成
```

## 🎓 学习路径

### 5 分钟快速了解

1. ✅ 阅读本文档
2. 查看 [架构总览](./01-architecture.md)

### 30 分钟深入理解

1. [Artifact Graph 系统](./02-artifact-graph.md) - 核心算法
2. [Schema 系统](./03-schema-system.md) - 工作流定义
3. [状态管理](./04-state-management.md) - 状态跟踪

### 2 小时掌握实现

1. 阅读所有文档
2. 运行项目，跟踪代码执行
3. 阅读源码，理解细节

## 🔍 常见问题

### Q: 为什么使用依赖图？

**A:** 自动管理依赖关系，支持并行工作流，灵活可扩展。

### Q: 为什么不用数据库？

**A:** 状态即文件，简单直观，Git 友好，无需额外依赖。

### Q: 如何自定义工作流？

**A:** 创建自定义 Schema，定义工件和依赖关系。详见 [Schema 系统](./03-schema-system.md#自定义-schema)

### Q: 如何添加项目特定规则？

**A:** 创建 `openspec/config.yaml`，定义 context 和 rules。详见 [指令生成](./05-instruction-generation.md#项目配置)

## 🚀 下一步

### 想了解算法细节？

→ [Artifact Graph 系统](./02-artifact-graph.md)

### 想自定义工作流？

→ [Schema 系统](./03-schema-system.md)

### 想理解 AI 集成？

→ [指令生成系统](./05-instruction-generation.md)

### 想查看完整文档？

→ [文档索引](./README.md)

## 📚 相关资源

- **完整文档：** [docs/implementation/](./README.md)
- **快速索引：** [INDEX.md](./INDEX.md)
- **总结文档：** [implementation-summary.md](../implementation-summary.md)
- **源代码：** [src/core/](../../src/core/)

---

**5 分钟了解完毕！** 现在你已经掌握了 OpenSpec 的核心实现原理。

继续深入学习：[完整文档索引](./README.md)
