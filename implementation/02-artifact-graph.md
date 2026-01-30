# 2. Artifact Graph 系统

## 概述

Artifact Graph 是 OpenSpec 的核心创新，它将文档创建过程建模为**有向无环图（DAG）**，自动管理工件之间的依赖关系。

## 核心概念

### 什么是 Artifact？

**Artifact（工件）** 是工作流中需要创建的一个文档或文件。

```typescript
interface Artifact {
  id: string;              // 唯一标识，如 "proposal"
  generates: string;       // 生成的文件路径，如 "proposal.md"
  description: string;     // 描述
  template: string;        // 模板文件名
  instruction?: string;    // AI 生成指令
  requires: string[];      // 依赖的其他工件 ID
}
```

**示例：**
```yaml
- id: proposal
  generates: proposal.md
  description: Initial proposal document
  template: proposal.md
  requires: []  # 无依赖，可以直接开始

- id: specs
  generates: "specs/**/*.md"  # 支持 glob 模式
  description: Detailed specifications
  template: spec.md
  requires: [proposal]  # 依赖 proposal

- id: tasks
  generates: tasks.md
  description: Implementation checklist
  template: tasks.md
  requires: [specs, design]  # 依赖多个工件
```

### 依赖图可视化

```
    proposal (无依赖)
    /      \
specs    design (并行，都依赖 proposal)
    \      /
     tasks (依赖 specs 和 design)
```

## 核心类：ArtifactGraph

### 类结构

```typescript
export class ArtifactGraph {
  private artifacts: Map<string, Artifact>;
  private schema: SchemaYaml;

  // 构造函数（私有）
  private constructor(schema: SchemaYaml);

  // 工厂方法
  static fromYaml(filePath: string): ArtifactGraph;
  static fromYamlContent(yamlContent: string): ArtifactGraph;
  static fromSchema(schema: SchemaYaml): ArtifactGraph;

  // 查询方法
  getArtifact(id: string): Artifact | undefined;
  getAllArtifacts(): Artifact[];
  getName(): string;
  getVersion(): number;

  // 核心算法
  getBuildOrder(): string[];
  getNextArtifacts(completed: CompletedSet): string[];
  isComplete(completed: CompletedSet): boolean;
  getBlocked(completed: CompletedSet): BlockedArtifacts;
}
```

### 核心算法详解

#### 1. 拓扑排序 (Kahn's Algorithm)

**目的：** 计算工件的构建顺序

**算法实现：**

```typescript
getBuildOrder(): string[] {
  // 步骤 1: 初始化入度表和依赖表
  const inDegree = new Map<string, number>();
  const dependents = new Map<string, string[]>();

  for (const artifact of this.artifacts.values()) {
    // 入度 = 依赖的数量
    inDegree.set(artifact.id, artifact.requires.length);
    dependents.set(artifact.id, []);
  }

  // 步骤 2: 构建反向依赖图（谁依赖我）
  for (const artifact of this.artifacts.values()) {
    for (const req of artifact.requires) {
      dependents.get(req)!.push(artifact.id);
    }
  }

  // 步骤 3: 找到所有入度为 0 的节点（无依赖）
  const queue = [...this.artifacts.keys()]
    .filter(id => inDegree.get(id) === 0)
    .sort();  // 排序保证确定性

  const result: string[] = [];

  // 步骤 4: BFS 遍历
  while (queue.length > 0) {
    const current = queue.shift()!;
    result.push(current);

    // 减少依赖者的入度
    const newlyReady: string[] = [];
    for (const dep of dependents.get(current)!) {
      const newDegree = inDegree.get(dep)! - 1;
      inDegree.set(dep, newDegree);
      
      // 入度变为 0，加入队列
      if (newDegree === 0) {
        newlyReady.push(dep);
      }
    }
    
    queue.push(...newlyReady.sort());
  }

  return result;
}
```

**示例执行：**

```
初始状态:
  inDegree: { proposal: 0, specs: 1, design: 1, tasks: 2 }
  queue: [proposal]

迭代 1:
  处理: proposal
  更新: specs (1→0), design (1→0)
  queue: [design, specs]  # 排序后
  result: [proposal]

迭代 2:
  处理: design
  更新: tasks (2→1)
  queue: [specs]
  result: [proposal, design]

迭代 3:
  处理: specs
  更新: tasks (1→0)
  queue: [tasks]
  result: [proposal, design, specs]

迭代 4:
  处理: tasks
  queue: []
  result: [proposal, design, specs, tasks]
```

**时间复杂度：** O(V + E)，其中 V 是工件数，E 是依赖边数

#### 2. 查找可执行工件

**目的：** 找出所有依赖已满足的工件

```typescript
getNextArtifacts(completed: CompletedSet): string[] {
  const ready: string[] = [];

  for (const artifact of this.artifacts.values()) {
    // 跳过已完成的
    if (completed.has(artifact.id)) {
      continue;
    }

    // 检查所有依赖是否都已完成
    const allDepsCompleted = artifact.requires.every(
      req => completed.has(req)
    );

    if (allDepsCompleted) {
      ready.push(artifact.id);
    }
  }

  return ready.sort();  // 排序保证确定性
}
```

**示例：**

```typescript
// 假设 completed = { proposal }
getNextArtifacts(completed)
// 返回: [design, specs]  // 两者都只依赖 proposal

// 假设 completed = { proposal, specs, design }
getNextArtifacts(completed)
// 返回: [tasks]  // tasks 依赖 specs 和 design
```

#### 3. 检查完成状态

```typescript
isComplete(completed: CompletedSet): boolean {
  for (const artifact of this.artifacts.values()) {
    if (!completed.has(artifact.id)) {
      return false;
    }
  }
  return true;
}
```

#### 4. 查找阻塞工件

**目的：** 找出被阻塞的工件及其未满足的依赖

```typescript
getBlocked(completed: CompletedSet): BlockedArtifacts {
  const blocked: BlockedArtifacts = {};

  for (const artifact of this.artifacts.values()) {
    // 跳过已完成的
    if (completed.has(artifact.id)) {
      continue;
    }

    // 找出未满足的依赖
    const unmetDeps = artifact.requires.filter(
      req => !completed.has(req)
    );

    if (unmetDeps.length > 0) {
      blocked[artifact.id] = unmetDeps.sort();
    }
  }

  return blocked;
}
```

**示例：**

```typescript
// 假设 completed = { proposal }
getBlocked(completed)
// 返回: { tasks: ['design', 'specs'] }
// tasks 被 design 和 specs 阻塞
```

## Schema 解析

### Schema 文件格式

```yaml
name: spec-driven
version: 1
description: Default OpenSpec workflow
artifacts:
  - id: proposal
    generates: proposal.md
    description: Initial proposal
    template: proposal.md
    instruction: |
      Create the proposal document...
    requires: []

  - id: specs
    generates: "specs/**/*.md"
    description: Detailed specifications
    template: spec.md
    requires: [proposal]

apply:
  requires: [tasks]
  tracks: tasks.md
  instruction: |
    Implement tasks...
```

### 解析流程

```typescript
export function parseSchema(yamlContent: string): SchemaYaml {
  // 1. 解析 YAML
  const parsed = parseYaml(yamlContent);

  // 2. Zod 验证
  const result = SchemaYamlSchema.safeParse(parsed);
  if (!result.success) {
    throw new SchemaValidationError(formatErrors(result.error));
  }

  const schema = result.data;

  // 3. 业务规则验证
  validateNoDuplicateIds(schema.artifacts);
  validateRequiresReferences(schema.artifacts);
  validateNoCycles(schema.artifacts);

  return schema;
}
```

### 循环依赖检测

**使用 DFS 检测环：**

```typescript
function validateNoCycles(artifacts: Artifact[]): void {
  const artifactMap = new Map(artifacts.map(a => [a.id, a]));
  const visited = new Set<string>();
  const inStack = new Set<string>();  // 当前 DFS 路径
  const parent = new Map<string, string>();

  function dfs(id: string): string | null {
    visited.add(id);
    inStack.add(id);

    const artifact = artifactMap.get(id);
    if (!artifact) return null;

    for (const dep of artifact.requires) {
      if (!visited.has(dep)) {
        parent.set(dep, id);
        const cycle = dfs(dep);
        if (cycle) return cycle;
      } else if (inStack.has(dep)) {
        // 发现环！重建路径
        const cyclePath = [dep];
        let current = id;
        while (current !== dep) {
          cyclePath.unshift(current);
          current = parent.get(current)!;
        }
        cyclePath.unshift(dep);
        return cyclePath.join(' → ');
      }
    }

    inStack.delete(id);
    return null;
  }

  // 检查所有连通分量
  for (const artifact of artifacts) {
    if (!visited.has(artifact.id)) {
      const cycle = dfs(artifact.id);
      if (cycle) {
        throw new SchemaValidationError(
          `Cyclic dependency detected: ${cycle}`
        );
      }
    }
  }
}
```

**示例：**

```yaml
# 错误的 Schema（有环）
artifacts:
  - id: a
    requires: [b]
  - id: b
    requires: [c]
  - id: c
    requires: [a]  # 环！

# 错误信息：
# Cyclic dependency detected: a → b → c → a
```

## 状态检测

### 基于文件系统的状态

```typescript
export function detectCompleted(
  graph: ArtifactGraph,
  changeDir: string
): CompletedSet {
  const completed = new Set<string>();

  // 检查目录是否存在
  if (!fs.existsSync(changeDir)) {
    return completed;
  }

  for (const artifact of graph.getAllArtifacts()) {
    if (isArtifactComplete(artifact.generates, changeDir)) {
      completed.add(artifact.id);
    }
  }

  return completed;
}
```

### Glob 模式支持

```typescript
function isArtifactComplete(
  generates: string,
  changeDir: string
): boolean {
  const fullPattern = path.join(changeDir, generates);

  // 检查是否是 glob 模式
  if (isGlobPattern(generates)) {
    return hasGlobMatches(fullPattern);
  }

  // 简单文件路径
  return fs.existsSync(fullPattern);
}

function isGlobPattern(pattern: string): boolean {
  return pattern.includes('*') || 
         pattern.includes('?') || 
         pattern.includes('[');
}

function hasGlobMatches(pattern: string): boolean {
  const normalizedPattern = FileSystemUtils.toPosixPath(pattern);
  const matches = fg.sync(normalizedPattern, { onlyFiles: true });
  return matches.length > 0;
}
```

**示例：**

```typescript
// 简单文件
generates: "proposal.md"
// 检查: openspec/changes/add-auth/proposal.md 是否存在

// Glob 模式
generates: "specs/**/*.md"
// 检查: openspec/changes/add-auth/specs/ 下是否有 .md 文件
```

## 使用示例

### 创建依赖图

```typescript
// 从文件加载
const graph = ArtifactGraph.fromYaml('schemas/spec-driven/schema.yaml');

// 从内容加载
const yamlContent = fs.readFileSync('schema.yaml', 'utf-8');
const graph = ArtifactGraph.fromYamlContent(yamlContent);

// 从已解析的 Schema
const schema = parseSchema(yamlContent);
const graph = ArtifactGraph.fromSchema(schema);
```

### 查询工件信息

```typescript
// 获取单个工件
const proposal = graph.getArtifact('proposal');
console.log(proposal.description);

// 获取所有工件
const allArtifacts = graph.getAllArtifacts();
console.log(`Total artifacts: ${allArtifacts.length}`);

// 获取构建顺序
const buildOrder = graph.getBuildOrder();
console.log('Build order:', buildOrder);
// 输出: ['proposal', 'design', 'specs', 'tasks']
```

### 工作流控制

```typescript
// 检测已完成的工件
const completed = detectCompleted(graph, changeDir);
console.log('Completed:', Array.from(completed));

// 查找下一步可执行的工件
const next = graph.getNextArtifacts(completed);
console.log('Ready to create:', next);

// 检查是否全部完成
if (graph.isComplete(completed)) {
  console.log('All artifacts complete!');
}

// 查看阻塞状态
const blocked = graph.getBlocked(completed);
for (const [id, deps] of Object.entries(blocked)) {
  console.log(`${id} is blocked by: ${deps.join(', ')}`);
}
```

## 设计优势

### 1. 声明式依赖管理

**传统方式（硬编码）：**
```typescript
// 硬编码的顺序
async function createArtifacts() {
  await createProposal();
  await createSpecs();
  await createDesign();
  await createTasks();
}
```

**Artifact Graph 方式：**
```yaml
# 声明依赖关系
artifacts:
  - id: specs
    requires: [proposal]
  - id: design
    requires: [proposal]
  - id: tasks
    requires: [specs, design]
```

**优势：**
- 依赖关系清晰可见
- 自动计算执行顺序
- 支持并行执行（specs 和 design）

### 2. 灵活的工作流

**支持多种工作流模式：**

```yaml
# 线性工作流
proposal → specs → design → tasks

# 并行工作流
proposal → (specs + design) → tasks

# 复杂工作流
proposal → specs → (design + tests) → implementation → docs
```

### 3. 可扩展性

**添加新工件无需修改代码：**

```yaml
# 添加新的工件类型
artifacts:
  - id: security-review
    generates: security-review.md
    requires: [design]
  
  - id: tasks
    requires: [specs, design, security-review]  # 更新依赖
```

## 性能考虑

### 时间复杂度

| 操作 | 复杂度 | 说明 |
|------|--------|------|
| getBuildOrder | O(V + E) | V=工件数, E=依赖边数 |
| getNextArtifacts | O(V × D) | D=平均依赖数 |
| isComplete | O(V) | 遍历所有工件 |
| getBlocked | O(V × D) | 遍历并检查依赖 |

### 空间复杂度

- 依赖图存储：O(V + E)
- 完成状态：O(V)

### 优化策略

1. **缓存构建顺序** - 构建顺序不变，可以缓存
2. **增量更新** - 只检查状态变化的工件
3. **并行检测** - 文件存在性检查可以并行

## 下一步

- 了解 [Schema 系统](./03-schema-system.md) 如何定义工作流
- 学习 [指令生成系统](./05-instruction-generation.md) 如何使用依赖图
