# 5. 指令生成系统

## 概述

指令生成系统是 OpenSpec 与 AI 集成的核心，它为 AI 生成结构化的、上下文丰富的指令，指导 AI 创建工件。

## 核心概念

### 指令的组成

一个完整的工件指令包含：

```xml
<artifact id="proposal" change="add-auth" schema="spec-driven">
  <task>创建提案文档</task>
  
  <project_context>
    <!-- 项目背景，AI 不要包含在输出中 -->
  </project_context>
  
  <rules>
    <!-- AI 需要遵守的规则 -->
  </rules>
  
  <dependencies>
    <!-- 需要先读取的文件 -->
  </dependencies>
  
  <output>
    写入到: openspec/changes/add-auth/proposal.md
  </output>
  
  <instruction>
    <!-- Schema 提供的指导 -->
  </instruction>
  
  <template>
    <!-- 模板结构，AI 需要填充 -->
  </template>
  
  <unlocks>
    完成后解锁: design, specs
  </unlocks>
</artifact>
```

### 关键区别

| 元素 | 用途 | AI 是否包含在输出 |
|------|------|-------------------|
| context | 项目背景约束 | ❌ 否 |
| rules | 工件特定规则 | ❌ 否 |
| template | 输出结构 | ✅ 是（填充后） |
| instruction | 创建指导 | ❌ 否（仅作指导） |

## 核心接口

### ArtifactInstructions

```typescript
export interface ArtifactInstructions {
  /** 变更名称 */
  changeName: string;
  
  /** 工件 ID */
  artifactId: string;
  
  /** Schema 名称 */
  schemaName: string;
  
  /** 变更目录完整路径 */
  changeDir: string;
  
  /** 输出路径模式 (如 "proposal.md") */
  outputPath: string;
  
  /** 工件描述 */
  description: string;
  
  /** Schema 的指令字段（如何创建） */
  instruction: string | undefined;
  
  /** 项目上下文（约束，不包含在输出） */
  context: string | undefined;
  
  /** 工件特定规则（约束，不包含在输出） */
  rules: string[] | undefined;
  
  /** 模板内容（输出格式） */
  template: string;
  
  /** 依赖信息（完成状态和路径） */
  dependencies: DependencyInfo[];
  
  /** 完成后解锁的工件 */
  unlocks: string[];
}
```

### DependencyInfo

```typescript
export interface DependencyInfo {
  /** 工件 ID */
  id: string;
  
  /** 是否已完成 */
  done: boolean;
  
  /** 相对输出路径 */
  path: string;
  
  /** 工件描述 */
  description: string;
}
```

## 指令生成流程

### 主函数：generateInstructions

```typescript
export function generateInstructions(
  context: ChangeContext,
  artifactId: string,
  projectRoot?: string
): ArtifactInstructions {
  // 1. 获取工件定义
  const artifact = context.graph.getArtifact(artifactId);
  if (!artifact) {
    throw new Error(
      `Artifact '${artifactId}' not found in schema '${context.schemaName}'`
    );
  }

  // 2. 加载模板
  const templateContent = loadTemplate(
    context.schemaName,
    artifact.template,
    context.projectRoot
  );

  // 3. 收集依赖信息
  const dependencies = getDependencyInfo(
    artifact,
    context.graph,
    context.completed
  );

  // 4. 查找解锁的工件
  const unlocks = getUnlockedArtifacts(context.graph, artifactId);

  // 5. 读取项目配置
  const effectiveProjectRoot = projectRoot ?? context.projectRoot;
  let projectConfig = null;
  if (effectiveProjectRoot) {
    try {
      projectConfig = readProjectConfig(effectiveProjectRoot);
    } catch {
      // 配置读取失败，继续不使用配置
    }
  }

  // 6. 验证规则（如果有）
  if (projectConfig?.rules) {
    const validArtifactIds = new Set(
      context.graph.getAllArtifacts().map(a => a.id)
    );
    const warnings = validateConfigRules(
      projectConfig.rules,
      validArtifactIds,
      context.schemaName
    );
    
    // 显示警告（每个会话只显示一次）
    for (const warning of warnings) {
      if (!shownWarnings.has(warning)) {
        console.warn(warning);
        shownWarnings.add(warning);
      }
    }
  }

  // 7. 提取配置
  const configContext = projectConfig?.context?.trim() || undefined;
  const rulesForArtifact = projectConfig?.rules?.[artifactId];
  const configRules = rulesForArtifact && rulesForArtifact.length > 0 
    ? rulesForArtifact 
    : undefined;

  // 8. 组装指令
  return {
    changeName: context.changeName,
    artifactId: artifact.id,
    schemaName: context.schemaName,
    changeDir: context.changeDir,
    outputPath: artifact.generates,
    description: artifact.description,
    instruction: artifact.instruction,
    context: configContext,
    rules: configRules,
    template: templateContent,
    dependencies,
    unlocks,
  };
}
```

## 模板加载

### loadTemplate 函数

```typescript
export function loadTemplate(
  schemaName: string,
  templatePath: string,
  projectRoot?: string
): string {
  // 1. 获取 Schema 目录
  const schemaDir = getSchemaDir(schemaName, projectRoot);
  if (!schemaDir) {
    throw new TemplateLoadError(
      `Schema '${schemaName}' not found`,
      templatePath
    );
  }

  // 2. 构建完整路径
  const fullPath = path.join(schemaDir, 'templates', templatePath);

  // 3. 检查文件存在
  if (!fs.existsSync(fullPath)) {
    throw new TemplateLoadError(
      `Template not found: ${fullPath}`,
      fullPath
    );
  }

  // 4. 读取内容
  try {
    return fs.readFileSync(fullPath, 'utf-8');
  } catch (err) {
    const ioError = err instanceof Error ? err : new Error(String(err));
    throw new TemplateLoadError(
      `Failed to read template: ${ioError.message}`,
      fullPath
    );
  }
}
```

### 模板路径解析

```
Schema 目录: schemas/spec-driven/
  ├── schema.yaml
  └── templates/
      ├── proposal.md      ← artifact.template: "proposal.md"
      ├── spec.md
      ├── design.md
      └── tasks.md
```

## 依赖信息收集

### getDependencyInfo 函数

```typescript
function getDependencyInfo(
  artifact: Artifact,
  graph: ArtifactGraph,
  completed: CompletedSet
): DependencyInfo[] {
  return artifact.requires.map(id => {
    const depArtifact = graph.getArtifact(id);
    
    return {
      id,
      done: completed.has(id),
      path: depArtifact?.generates ?? id,
      description: depArtifact?.description ?? '',
    };
  });
}
```

### 示例输出

```typescript
// 假设 tasks 依赖 specs 和 design
// completed = { specs }

dependencies: [
  {
    id: 'specs',
    done: true,
    path: 'specs/**/*.md',
    description: 'Detailed specifications'
  },
  {
    id: 'design',
    done: false,
    path: 'design.md',
    description: 'Technical design document'
  }
]
```

## 解锁工件查找

### getUnlockedArtifacts 函数

```typescript
function getUnlockedArtifacts(
  graph: ArtifactGraph,
  artifactId: string
): string[] {
  const unlocks: string[] = [];

  // 查找所有依赖当前工件的工件
  for (const artifact of graph.getAllArtifacts()) {
    if (artifact.requires.includes(artifactId)) {
      unlocks.push(artifact.id);
    }
  }

  return unlocks.sort();
}
```

### 示例

```yaml
# Schema 定义
artifacts:
  - id: proposal
    requires: []
  - id: specs
    requires: [proposal]
  - id: design
    requires: [proposal]
```

```typescript
// 完成 proposal 后
getUnlockedArtifacts(graph, 'proposal')
// 返回: ['design', 'specs']
```

## 项目配置

### config.yaml 格式

```yaml
# openspec/config.yaml
context: |
  This is a React TypeScript project using Vite.
  We follow functional programming patterns.
  All components must be accessible (WCAG 2.1 AA).

rules:
  proposal:
    - Keep under 500 words
    - Focus on user value, not implementation
  
  specs:
    - Use SHALL/MUST for requirements
    - Include at least one scenario per requirement
    - Scenarios must use WHEN/THEN format
  
  design:
    - Document all architectural decisions
    - Include alternatives considered
    - Address security and performance
```

### 配置 Schema

```typescript
export const ProjectConfigSchema = z.object({
  context: z.string().optional(),
  rules: z.record(z.array(z.string())).optional(),
});

export type ProjectConfig = z.infer<typeof ProjectConfigSchema>;
```

### 读取配置

```typescript
export function readProjectConfig(projectRoot: string): ProjectConfig {
  const configPath = path.join(projectRoot, 'openspec', 'config.yaml');
  
  if (!fs.existsSync(configPath)) {
    return {};
  }

  const content = fs.readFileSync(configPath, 'utf-8');
  const parsed = parseYaml(content);
  const result = ProjectConfigSchema.safeParse(parsed);

  if (!result.success) {
    throw new Error(`Invalid config: ${formatZodErrors(result.error)}`);
  }

  return result.data;
}
```

### 规则验证

```typescript
export function validateConfigRules(
  rules: Record<string, string[]>,
  validArtifactIds: Set<string>,
  schemaName: string
): string[] {
  const warnings: string[] = [];

  for (const artifactId of Object.keys(rules)) {
    if (!validArtifactIds.has(artifactId)) {
      warnings.push(
        `Warning: config.yaml defines rules for '${artifactId}', ` +
        `but schema '${schemaName}' has no such artifact. ` +
        `These rules will be ignored.`
      );
    }
  }

  return warnings;
}
```

## 指令格式化

### 文本格式输出

```typescript
export function printInstructionsText(
  instructions: ArtifactInstructions,
  isBlocked: boolean
): void {
  const {
    artifactId,
    changeName,
    schemaName,
    changeDir,
    outputPath,
    description,
    instruction,
    context,
    rules,
    template,
    dependencies,
    unlocks,
  } = instructions;

  // 1. 开始标签
  console.log(`<artifact id="${artifactId}" change="${changeName}" schema="${schemaName}">`);
  console.log();

  // 2. 阻塞警告
  if (isBlocked) {
    const missing = dependencies.filter(d => !d.done).map(d => d.id);
    console.log('<warning>');
    console.log('This artifact has unmet dependencies.');
    console.log(`Missing: ${missing.join(', ')}`);
    console.log('</warning>');
    console.log();
  }

  // 3. 任务描述
  console.log('<task>');
  console.log(`Create the ${artifactId} artifact for change "${changeName}".`);
  console.log(description);
  console.log('</task>');
  console.log();

  // 4. 项目上下文（约束）
  if (context) {
    console.log('<project_context>');
    console.log('<!-- Background for you. Do NOT include in output. -->');
    console.log(context);
    console.log('</project_context>');
    console.log();
  }

  // 5. 规则（约束）
  if (rules && rules.length > 0) {
    console.log('<rules>');
    console.log('<!-- Constraints for you. Do NOT include in output. -->');
    for (const rule of rules) {
      console.log(`- ${rule}`);
    }
    console.log('</rules>');
    console.log();
  }

  // 6. 依赖文件
  if (dependencies.length > 0) {
    console.log('<dependencies>');
    console.log('Read these files for context:');
    console.log();
    for (const dep of dependencies) {
      const status = dep.done ? 'done' : 'missing';
      const fullPath = path.join(changeDir, dep.path);
      console.log(`<dependency id="${dep.id}" status="${status}">`);
      console.log(`  <path>${fullPath}</path>`);
      console.log(`  <description>${dep.description}</description>`);
      console.log('</dependency>');
    }
    console.log('</dependencies>');
    console.log();
  }

  // 7. 输出位置
  console.log('<output>');
  console.log(`Write to: ${path.join(changeDir, outputPath)}`);
  console.log('</output>');
  console.log();

  // 8. 指令
  if (instruction) {
    console.log('<instruction>');
    console.log(instruction.trim());
    console.log('</instruction>');
    console.log();
  }

  // 9. 模板
  console.log('<template>');
  console.log('<!-- Use this as the structure. Fill in the sections. -->');
  console.log(template.trim());
  console.log('</template>');
  console.log();

  // 10. 解锁信息
  if (unlocks.length > 0) {
    console.log('<unlocks>');
    console.log(`Completing this enables: ${unlocks.join(', ')}`);
    console.log('</unlocks>');
    console.log();
  }

  // 11. 结束标签
  console.log('</artifact>');
}
```

### JSON 格式输出

```typescript
// 直接序列化 ArtifactInstructions
const instructions = generateInstructions(context, 'proposal');
console.log(JSON.stringify(instructions, null, 2));
```

```json
{
  "changeName": "add-auth",
  "artifactId": "proposal",
  "schemaName": "spec-driven",
  "changeDir": "/project/openspec/changes/add-auth",
  "outputPath": "proposal.md",
  "description": "Initial proposal document",
  "instruction": "Create the proposal document...",
  "context": "This is a React TypeScript project...",
  "rules": [
    "Keep under 500 words",
    "Focus on user value"
  ],
  "template": "## Why\n...",
  "dependencies": [],
  "unlocks": ["design", "specs"]
}
```

## Apply 指令生成

### ApplyInstructions 接口

```typescript
export interface ApplyInstructions {
  changeName: string;
  changeDir: string;
  schemaName: string;
  contextFiles: Record<string, string>;  // artifactId → 文件路径
  progress: {
    total: number;
    complete: number;
    remaining: number;
  };
  tasks: TaskItem[];
  state: 'blocked' | 'ready' | 'all_done';
  missingArtifacts?: string[];
  instruction: string;
}
```

### generateApplyInstructions 函数

```typescript
export async function generateApplyInstructions(
  projectRoot: string,
  changeName: string,
  schemaName?: string
): Promise<ApplyInstructions> {
  // 1. 加载上下文
  const context = loadChangeContext(projectRoot, changeName, schemaName);
  const changeDir = path.join(projectRoot, 'openspec', 'changes', changeName);

  // 2. 获取 Schema 的 apply 配置
  const schema = resolveSchema(context.schemaName, projectRoot);
  const applyConfig = schema.apply;

  // 3. 确定必需的工件
  const requiredArtifactIds = applyConfig?.requires ?? 
                              schema.artifacts.map(a => a.id);
  const tracksFile = applyConfig?.tracks ?? null;
  const schemaInstruction = applyConfig?.instruction ?? null;

  // 4. 检查缺失的工件
  const missingArtifacts: string[] = [];
  for (const artifactId of requiredArtifactIds) {
    const artifact = schema.artifacts.find(a => a.id === artifactId);
    if (artifact && !artifactOutputExists(changeDir, artifact.generates)) {
      missingArtifacts.push(artifactId);
    }
  }

  // 5. 构建上下文文件映射
  const contextFiles: Record<string, string> = {};
  for (const artifact of schema.artifacts) {
    if (artifactOutputExists(changeDir, artifact.generates)) {
      contextFiles[artifact.id] = path.join(changeDir, artifact.generates);
    }
  }

  // 6. 解析任务（如果有跟踪文件）
  let tasks: TaskItem[] = [];
  let tracksFileExists = false;
  if (tracksFile) {
    const tracksPath = path.join(changeDir, tracksFile);
    tracksFileExists = fs.existsSync(tracksPath);
    if (tracksFileExists) {
      const tasksContent = await fs.promises.readFile(tracksPath, 'utf-8');
      tasks = parseTasksFile(tasksContent);
    }
  }

  // 7. 计算进度
  const total = tasks.length;
  const complete = tasks.filter(t => t.done).length;
  const remaining = total - complete;

  // 8. 确定状态和指令
  let state: ApplyInstructions['state'];
  let instruction: string;

  if (missingArtifacts.length > 0) {
    state = 'blocked';
    instruction = `Missing artifacts: ${missingArtifacts.join(', ')}. ` +
                  `Use openspec-continue-change to create them first.`;
  } else if (tracksFile && !tracksFileExists) {
    state = 'blocked';
    instruction = `The ${path.basename(tracksFile)} file is missing. ` +
                  `Use openspec-continue-change to generate it.`;
  } else if (tracksFile && tracksFileExists && total === 0) {
    state = 'blocked';
    instruction = `The ${path.basename(tracksFile)} file has no tasks. ` +
                  `Add tasks or regenerate it.`;
  } else if (tracksFile && remaining === 0 && total > 0) {
    state = 'all_done';
    instruction = 'All tasks complete! Ready to archive.';
  } else if (!tracksFile) {
    state = 'ready';
    instruction = schemaInstruction?.trim() ?? 
                  'All required artifacts complete. Proceed with implementation.';
  } else {
    state = 'ready';
    instruction = schemaInstruction?.trim() ?? 
                  'Work through pending tasks, mark complete as you go.';
  }

  return {
    changeName,
    changeDir,
    schemaName: context.schemaName,
    contextFiles,
    progress: { total, complete, remaining },
    tasks,
    state,
    missingArtifacts: missingArtifacts.length > 0 ? missingArtifacts : undefined,
    instruction,
  };
}
```

## 使用示例

### 生成工件指令

```typescript
// 加载变更上下文
const context = loadChangeContext(
  process.cwd(),
  'add-auth'
);

// 生成 proposal 指令
const instructions = generateInstructions(context, 'proposal');

// 文本格式输出
const isBlocked = instructions.dependencies.some(d => !d.done);
printInstructionsText(instructions, isBlocked);

// JSON 格式输出
console.log(JSON.stringify(instructions, null, 2));
```

### 生成 Apply 指令

```typescript
const applyInstructions = await generateApplyInstructions(
  process.cwd(),
  'add-auth'
);

if (applyInstructions.state === 'blocked') {
  console.error('Cannot apply:', applyInstructions.instruction);
} else {
  console.log('Context files:', applyInstructions.contextFiles);
  console.log('Progress:', applyInstructions.progress);
  console.log('Tasks:', applyInstructions.tasks);
}
```

## 设计优势

### 1. 分离约束和输出

**约束（不包含在输出）：**
- `context` - 项目背景
- `rules` - 工件规则
- `instruction` - 创建指导

**输出（AI 填充）：**
- `template` - 结构模板

这种分离让 AI 清楚地知道什么是指导，什么是输出格式。

### 2. 丰富的上下文

指令包含：
- 依赖文件路径和状态
- 项目特定的约束
- 完成后的影响（unlocks）

### 3. 结构化格式

使用 XML 风格的标签，便于：
- AI 解析和理解
- 人类阅读和调试
- 工具处理和验证

## 下一步

- 了解 [工作流命令](./06-workflow-commands.md) 如何使用指令
- 学习 [技能生成](./10-skill-generation.md) 如何封装指令
