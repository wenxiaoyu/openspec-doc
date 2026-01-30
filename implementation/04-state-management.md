# 4. 状态管理

## 设计哲学：状态即文件

OpenSpec 采用**基于文件系统的状态管理**，不使用数据库或外部存储。

### 核心原则

```
工件完成状态 = 文件是否存在
变更元数据 = .openspec.yaml 文件
任务进度 = tasks.md 中的 checkbox 状态
```

### 优势

1. **简单直观** - 状态一目了然
2. **Git 友好** - 状态随代码版本化
3. **无依赖** - 不需要数据库或配置服务
4. **易于调试** - 直接查看文件系统
5. **可移植** - 复制目录即可迁移

## 状态检测系统

### 核心函数：detectCompleted

```typescript
export function detectCompleted(
  graph: ArtifactGraph,
  changeDir: string
): CompletedSet {
  const completed = new Set<string>();

  // 处理缺失的变更目录
  if (!fs.existsSync(changeDir)) {
    return completed;
  }

  // 检查每个工件
  for (const artifact of graph.getAllArtifacts()) {
    if (isArtifactComplete(artifact.generates, changeDir)) {
      completed.add(artifact.id);
    }
  }

  return completed;
}
```

### 工件完成性检查

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
```

## Glob 模式支持

### 模式识别

```typescript
function isGlobPattern(pattern: string): boolean {
  return pattern.includes('*') ||   // 通配符
         pattern.includes('?') ||   // 单字符匹配
         pattern.includes('[');     // 字符集
}
```

### 示例

```yaml
# 简单文件
generates: "proposal.md"
# 检查: openspec/changes/add-auth/proposal.md

# 单层通配符
generates: "specs/*.md"
# 检查: openspec/changes/add-auth/specs/ 下的 .md 文件

# 递归通配符
generates: "specs/**/*.md"
# 检查: openspec/changes/add-auth/specs/ 及子目录下的 .md 文件

# 多个扩展名
generates: "docs/**/*.{md,txt}"
# 检查: .md 或 .txt 文件
```

### Glob 匹配实现

```typescript
function hasGlobMatches(pattern: string): boolean {
  // 规范化路径（Windows 兼容）
  const normalizedPattern = FileSystemUtils.toPosixPath(pattern);
  
  // 使用 fast-glob 查找匹配
  const matches = fg.sync(normalizedPattern, { 
    onlyFiles: true 
  });
  
  return matches.length > 0;
}
```

### 跨平台路径处理

```typescript
// FileSystemUtils.toPosixPath
export class FileSystemUtils {
  static toPosixPath(filePath: string): string {
    // Windows: C:\path\to\file → C:/path/to/file
    return filePath.split(path.sep).join('/');
  }
}
```

## 状态类型

### CompletedSet

```typescript
// 简单的 Set，存储已完成的工件 ID
export type CompletedSet = Set<string>;

// 示例
const completed: CompletedSet = new Set(['proposal', 'specs']);
```

### BlockedArtifacts

```typescript
// 映射：工件 ID → 未满足的依赖列表
export interface BlockedArtifacts {
  [artifactId: string]: string[];
}

// 示例
const blocked: BlockedArtifacts = {
  'tasks': ['design', 'specs'],  // tasks 被 design 和 specs 阻塞
  'docs': ['implementation']     // docs 被 implementation 阻塞
};
```

## 变更上下文

### ChangeContext 接口

```typescript
export interface ChangeContext {
  /** 工件依赖图 */
  graph: ArtifactGraph;
  
  /** 已完成的工件 ID 集合 */
  completed: CompletedSet;
  
  /** 使用的 Schema 名称 */
  schemaName: string;
  
  /** 变更名称 */
  changeName: string;
  
  /** 变更目录路径 */
  changeDir: string;
  
  /** 项目根目录 */
  projectRoot: string;
}
```

### 加载变更上下文

```typescript
export function loadChangeContext(
  projectRoot: string,
  changeName: string,
  schemaName?: string
): ChangeContext {
  const changeDir = path.join(
    projectRoot, 
    'openspec', 
    'changes', 
    changeName
  );

  // Schema 解析优先级：
  // 1. 显式参数
  // 2. .openspec.yaml 元数据
  // 3. 默认 'spec-driven'
  const resolvedSchemaName = resolveSchemaForChange(
    changeDir, 
    schemaName
  );

  // 加载 Schema 并构建依赖图
  const schema = resolveSchema(resolvedSchemaName, projectRoot);
  const graph = ArtifactGraph.fromSchema(schema);
  
  // 检测完成状态
  const completed = detectCompleted(graph, changeDir);

  return {
    graph,
    completed,
    schemaName: resolvedSchemaName,
    changeName,
    changeDir,
    projectRoot,
  };
}
```

## 状态查询

### 查询可执行工件

```typescript
// 使用 ChangeContext
const context = loadChangeContext(projectRoot, 'add-auth');
const nextArtifacts = context.graph.getNextArtifacts(context.completed);

console.log('Ready to create:', nextArtifacts);
// 输出: ['design', 'specs']  // 假设 proposal 已完成
```

### 查询阻塞状态

```typescript
const blocked = context.graph.getBlocked(context.completed);

for (const [id, deps] of Object.entries(blocked)) {
  console.log(`${id} is blocked by: ${deps.join(', ')}`);
}
// 输出: tasks is blocked by: design, specs
```

### 检查完成状态

```typescript
if (context.graph.isComplete(context.completed)) {
  console.log('All artifacts complete!');
} else {
  const total = context.graph.getAllArtifacts().length;
  const done = context.completed.size;
  console.log(`Progress: ${done}/${total}`);
}
```

## 任务进度跟踪

### Checkbox 格式

```markdown
## 1. Setup
- [ ] 1.1 Create module structure
- [x] 1.2 Add dependencies
- [ ] 1.3 Configure build

## 2. Implementation
- [x] 2.1 Implement core function
- [ ] 2.2 Add error handling
```

### 解析任务文件

```typescript
export interface TaskItem {
  id: string;
  description: string;
  done: boolean;
}

function parseTasksFile(content: string): TaskItem[] {
  const tasks: TaskItem[] = [];
  const lines = content.split('\n');
  let taskIndex = 0;

  for (const line of lines) {
    // 匹配 checkbox: - [ ] 或 - [x] 或 - [X]
    const checkboxMatch = line.match(/^[-*]\s*\[([ xX])\]\s*(.+)\s*$/);
    
    if (checkboxMatch) {
      taskIndex++;
      const done = checkboxMatch[1].toLowerCase() === 'x';
      const description = checkboxMatch[2].trim();
      
      tasks.push({
        id: `${taskIndex}`,
        description,
        done,
      });
    }
  }

  return tasks;
}
```

### 计算进度

```typescript
function calculateProgress(tasks: TaskItem[]) {
  const total = tasks.length;
  const complete = tasks.filter(t => t.done).length;
  const remaining = total - complete;
  
  return { total, complete, remaining };
}

// 使用示例
const content = fs.readFileSync('tasks.md', 'utf-8');
const tasks = parseTasksFile(content);
const progress = calculateProgress(tasks);

console.log(`Progress: ${progress.complete}/${progress.total}`);
// 输出: Progress: 3/5
```

## 状态格式化

### ArtifactStatus

```typescript
export interface ArtifactStatus {
  id: string;
  outputPath: string;
  status: 'done' | 'ready' | 'blocked';
  missingDeps?: string[];  // 仅 blocked 状态
}
```

### ChangeStatus

```typescript
export interface ChangeStatus {
  changeName: string;
  schemaName: string;
  isComplete: boolean;
  applyRequires: string[];  // Apply 阶段需要的工件
  artifacts: ArtifactStatus[];
}
```

### 格式化状态

```typescript
export function formatChangeStatus(
  context: ChangeContext
): ChangeStatus {
  // 加载 Schema 获取 apply 配置
  const schema = resolveSchema(context.schemaName, context.projectRoot);
  const applyRequires = schema.apply?.requires ?? 
                        schema.artifacts.map(a => a.id);

  const artifacts = context.graph.getAllArtifacts();
  const ready = new Set(
    context.graph.getNextArtifacts(context.completed)
  );
  const blocked = context.graph.getBlocked(context.completed);

  // 为每个工件生成状态
  const artifactStatuses: ArtifactStatus[] = artifacts.map(artifact => {
    if (context.completed.has(artifact.id)) {
      return {
        id: artifact.id,
        outputPath: artifact.generates,
        status: 'done' as const,
      };
    }

    if (ready.has(artifact.id)) {
      return {
        id: artifact.id,
        outputPath: artifact.generates,
        status: 'ready' as const,
      };
    }

    return {
      id: artifact.id,
      outputPath: artifact.generates,
      status: 'blocked' as const,
      missingDeps: blocked[artifact.id] ?? [],
    };
  });

  // 按构建顺序排序
  const buildOrder = context.graph.getBuildOrder();
  const orderMap = new Map(buildOrder.map((id, idx) => [id, idx]));
  artifactStatuses.sort(
    (a, b) => (orderMap.get(a.id) ?? 0) - (orderMap.get(b.id) ?? 0)
  );

  return {
    changeName: context.changeName,
    schemaName: context.schemaName,
    isComplete: context.graph.isComplete(context.completed),
    applyRequires,
    artifacts: artifactStatuses,
  };
}
```

## 状态输出

### 文本格式

```typescript
export function printStatusText(status: ChangeStatus): void {
  const doneCount = status.artifacts.filter(a => a.status === 'done').length;
  const total = status.artifacts.length;

  console.log(`Change: ${status.changeName}`);
  console.log(`Schema: ${status.schemaName}`);
  console.log(`Progress: ${doneCount}/${total} artifacts complete`);
  console.log();

  for (const artifact of status.artifacts) {
    const indicator = getStatusIndicator(artifact.status);
    const color = getStatusColor(artifact.status);
    let line = `${indicator} ${artifact.id}`;

    if (artifact.status === 'blocked' && artifact.missingDeps) {
      line += color(` (blocked by: ${artifact.missingDeps.join(', ')})`);
    }

    console.log(line);
  }

  if (status.isComplete) {
    console.log();
    console.log(chalk.green('All artifacts complete!'));
  }
}

function getStatusIndicator(status: string): string {
  switch (status) {
    case 'done': return '✓';
    case 'ready': return '→';
    case 'blocked': return '○';
    default: return '?';
  }
}

function getStatusColor(status: string) {
  switch (status) {
    case 'done': return chalk.green;
    case 'ready': return chalk.yellow;
    case 'blocked': return chalk.gray;
    default: return chalk.white;
  }
}
```

### 输出示例

```
Change: add-auth
Schema: spec-driven
Progress: 2/4 artifacts complete

✓ proposal
✓ specs
→ design
○ tasks (blocked by: design)
```

### JSON 格式

```typescript
// 直接序列化 ChangeStatus
const status = formatChangeStatus(context);
console.log(JSON.stringify(status, null, 2));
```

```json
{
  "changeName": "add-auth",
  "schemaName": "spec-driven",
  "isComplete": false,
  "applyRequires": ["tasks"],
  "artifacts": [
    {
      "id": "proposal",
      "outputPath": "proposal.md",
      "status": "done"
    },
    {
      "id": "specs",
      "outputPath": "specs/**/*.md",
      "status": "done"
    },
    {
      "id": "design",
      "outputPath": "design.md",
      "status": "ready"
    },
    {
      "id": "tasks",
      "outputPath": "tasks.md",
      "status": "blocked",
      "missingDeps": ["design"]
    }
  ]
}
```

## 状态持久化

### 元数据文件

```yaml
# openspec/changes/add-auth/.openspec.yaml
schema: spec-driven
created: 2025-01-30
```

### 创建元数据

```typescript
export async function createChange(
  projectRoot: string,
  name: string,
  options: { schema?: string }
): Promise<{ schema: string }> {
  const changeDir = path.join(projectRoot, 'openspec', 'changes', name);
  
  // 创建目录
  await fs.mkdir(changeDir, { recursive: true });

  // 确定 Schema
  const schema = options.schema ?? 'spec-driven';

  // 写入元数据
  const metadata = {
    schema,
    created: new Date().toISOString().split('T')[0], // YYYY-MM-DD
  };

  const metadataPath = path.join(changeDir, '.openspec.yaml');
  await fs.writeFile(
    metadataPath,
    stringify(metadata),
    'utf-8'
  );

  return { schema };
}
```

### 读取元数据

```typescript
export function readChangeMetadata(
  changeDir: string
): ChangeMetadata | null {
  const metadataPath = path.join(changeDir, '.openspec.yaml');
  
  if (!fs.existsSync(metadataPath)) {
    return null;
  }

  try {
    const content = fs.readFileSync(metadataPath, 'utf-8');
    const parsed = parseYaml(content);
    const result = ChangeMetadataSchema.safeParse(parsed);
    
    return result.success ? result.data : null;
  } catch {
    return null;
  }
}
```

## 性能优化

### 缓存策略

```typescript
class StateCache {
  private cache = new Map<string, CompletedSet>();
  private timestamps = new Map<string, number>();

  get(changeDir: string, maxAge: number = 5000): CompletedSet | null {
    const cached = this.cache.get(changeDir);
    const timestamp = this.timestamps.get(changeDir);

    if (cached && timestamp) {
      const age = Date.now() - timestamp;
      if (age < maxAge) {
        return cached;
      }
    }

    return null;
  }

  set(changeDir: string, completed: CompletedSet): void {
    this.cache.set(changeDir, completed);
    this.timestamps.set(changeDir, Date.now());
  }

  invalidate(changeDir: string): void {
    this.cache.delete(changeDir);
    this.timestamps.delete(changeDir);
  }
}
```

### 增量检测

```typescript
// 只检查特定工件
function detectArtifactComplete(
  artifact: Artifact,
  changeDir: string
): boolean {
  return isArtifactComplete(artifact.generates, changeDir);
}

// 增量更新
function updateCompleted(
  completed: CompletedSet,
  artifact: Artifact,
  changeDir: string
): boolean {
  const isComplete = detectArtifactComplete(artifact, changeDir);
  
  if (isComplete && !completed.has(artifact.id)) {
    completed.add(artifact.id);
    return true;  // 状态变化
  }
  
  return false;  // 无变化
}
```

## 使用示例

### 基本状态查询

```typescript
// 加载上下文
const context = loadChangeContext(
  process.cwd(),
  'add-auth'
);

// 查看完成状态
console.log('Completed:', Array.from(context.completed));

// 查看下一步
const next = context.graph.getNextArtifacts(context.completed);
console.log('Next:', next);

// 格式化输出
const status = formatChangeStatus(context);
printStatusText(status);
```

### 监控进度

```typescript
async function monitorProgress(changeName: string) {
  const context = loadChangeContext(process.cwd(), changeName);
  
  while (!context.graph.isComplete(context.completed)) {
    const status = formatChangeStatus(context);
    console.clear();
    printStatusText(status);
    
    // 等待 1 秒后重新检测
    await new Promise(resolve => setTimeout(resolve, 1000));
    
    // 重新检测状态
    context.completed = detectCompleted(context.graph, context.changeDir);
  }
  
  console.log('All artifacts complete!');
}
```

## 下一步

- 了解 [指令生成系统](./05-instruction-generation.md) 如何使用状态信息
- 学习 [工作流命令](./06-workflow-commands.md) 如何展示状态
