# 3. Schema 系统

## 概述

Schema 系统是 OpenSpec 的工作流定义语言，它允许用户声明式地定义工件、依赖关系和工作流程。

## Schema 结构

### 完整的 Schema 定义

```yaml
name: spec-driven                    # Schema 名称
version: 1                           # Schema 版本
description: Default workflow        # 描述

artifacts:                           # 工件列表
  - id: proposal                     # 工件 ID
    generates: proposal.md           # 生成的文件
    description: Initial proposal    # 描述
    template: proposal.md            # 模板文件
    instruction: |                   # AI 指令（可选）
      Create the proposal...
    requires: []                     # 依赖列表

  - id: specs
    generates: "specs/**/*.md"       # 支持 glob
    description: Specifications
    template: spec.md
    requires: [proposal]

apply:                               # Apply 阶段配置（可选）
  requires: [tasks]                  # 实现前需要的工件
  tracks: tasks.md                   # 进度跟踪文件
  instruction: |                     # Apply 指令
    Implement tasks...
```

### Zod Schema 定义

```typescript
// Artifact 定义
export const ArtifactSchema = z.object({
  id: z.string().min(1, { error: 'Artifact ID is required' }),
  generates: z.string().min(1, { error: 'generates field is required' }),
  description: z.string(),
  template: z.string().min(1, { error: 'template field is required' }),
  instruction: z.string().optional(),
  requires: z.array(z.string()).default([]),
});

// Apply 阶段配置
export const ApplyPhaseSchema = z.object({
  requires: z.array(z.string()).min(1, { 
    error: 'At least one required artifact' 
  }),
  tracks: z.string().nullable().optional(),
  instruction: z.string().optional(),
});

// 完整 Schema
export const SchemaYamlSchema = z.object({
  name: z.string().min(1, { error: 'Schema name is required' }),
  version: z.number().int().positive({ 
    error: 'Version must be a positive integer' 
  }),
  description: z.string().optional(),
  artifacts: z.array(ArtifactSchema).min(1, { 
    error: 'At least one artifact required' 
  }),
  apply: ApplyPhaseSchema.optional(),
});
```

## Schema 解析器

### 三层解析优先级

```
1. Project-local (项目本地)
   └─ <project>/openspec/schemas/<name>/schema.yaml

2. User-global (用户全局)
   └─ ~/.local/share/openspec/schemas/<name>/schema.yaml
   
3. Package built-in (内置)
   └─ <package>/schemas/<name>/schema.yaml
```

### 解析流程

```typescript
export function resolveSchema(
  name: string, 
  projectRoot?: string
): SchemaYaml {
  // 1. 规范化名称（移除 .yaml 扩展名）
  const normalizedName = name.replace(/\.ya?ml$/, '');

  // 2. 查找 Schema 目录
  const schemaDir = getSchemaDir(normalizedName, projectRoot);
  if (!schemaDir) {
    const availableSchemas = listSchemas(projectRoot);
    throw new Error(
      `Schema '${normalizedName}' not found. ` +
      `Available: ${availableSchemas.join(', ')}`
    );
  }

  // 3. 读取文件
  const schemaPath = path.join(schemaDir, 'schema.yaml');
  const content = fs.readFileSync(schemaPath, 'utf-8');

  // 4. 解析和验证
  return parseSchema(content);
}
```

### 目录查找逻辑

```typescript
export function getSchemaDir(
  name: string,
  projectRoot?: string
): string | null {
  // 1. 检查项目本地目录
  if (projectRoot) {
    const projectDir = path.join(
      projectRoot, 
      'openspec', 
      'schemas', 
      name
    );
    const projectSchemaPath = path.join(projectDir, 'schema.yaml');
    if (fs.existsSync(projectSchemaPath)) {
      return projectDir;
    }
  }

  // 2. 检查用户全局目录
  const userDir = path.join(getUserSchemasDir(), name);
  const userSchemaPath = path.join(userDir, 'schema.yaml');
  if (fs.existsSync(userSchemaPath)) {
    return userDir;
  }

  // 3. 检查内置目录
  const packageDir = path.join(getPackageSchemasDir(), name);
  const packageSchemaPath = path.join(packageDir, 'schema.yaml');
  if (fs.existsSync(packageSchemaPath)) {
    return packageDir;
  }

  return null;
}
```

### 路径解析

```typescript
// 获取内置 Schema 目录
export function getPackageSchemasDir(): string {
  const currentFile = fileURLToPath(import.meta.url);
  // 从 dist/core/artifact-graph/ 导航到 schemas/
  return path.join(
    path.dirname(currentFile), 
    '..', '..', '..', 
    'schemas'
  );
}

// 获取用户全局目录
export function getUserSchemasDir(): string {
  return path.join(getGlobalDataDir(), 'schemas');
}

// 获取项目本地目录
export function getProjectSchemasDir(projectRoot: string): string {
  return path.join(projectRoot, 'openspec', 'schemas');
}
```

## Schema 验证

### 多层验证机制

```typescript
export function parseSchema(yamlContent: string): SchemaYaml {
  // 1. YAML 解析
  const parsed = parseYaml(yamlContent);

  // 2. Zod Schema 验证
  const result = SchemaYamlSchema.safeParse(parsed);
  if (!result.success) {
    const errors = result.error.issues
      .map(e => `${e.path.join('.')}: ${e.message}`)
      .join(', ');
    throw new SchemaValidationError(`Invalid schema: ${errors}`);
  }

  const schema = result.data;

  // 3. 业务规则验证
  validateNoDuplicateIds(schema.artifacts);
  validateRequiresReferences(schema.artifacts);
  validateNoCycles(schema.artifacts);

  return schema;
}
```

### 验证规则详解

#### 1. 重复 ID 检测

```typescript
function validateNoDuplicateIds(artifacts: Artifact[]): void {
  const seen = new Set<string>();
  
  for (const artifact of artifacts) {
    if (seen.has(artifact.id)) {
      throw new SchemaValidationError(
        `Duplicate artifact ID: ${artifact.id}`
      );
    }
    seen.add(artifact.id);
  }
}
```

#### 2. 引用有效性检查

```typescript
function validateRequiresReferences(artifacts: Artifact[]): void {
  const validIds = new Set(artifacts.map(a => a.id));

  for (const artifact of artifacts) {
    for (const req of artifact.requires) {
      if (!validIds.has(req)) {
        throw new SchemaValidationError(
          `Invalid dependency in '${artifact.id}': ` +
          `'${req}' does not exist`
        );
      }
    }
  }
}
```

#### 3. 循环依赖检测

详见 [Artifact Graph 文档](./02-artifact-graph.md#循环依赖检测)

## 内置 Schema: spec-driven

### 工作流概览

```
proposal → (specs + design) → tasks → apply
```

### 完整定义

```yaml
name: spec-driven
version: 1
description: Default OpenSpec workflow

artifacts:
  # 1. Proposal - 为什么做这个变更
  - id: proposal
    generates: proposal.md
    description: Initial proposal document
    template: proposal.md
    instruction: |
      Create the proposal document that establishes WHY.
      Sections: Why, What Changes, Capabilities, Impact
    requires: []

  # 2. Specs - 详细的需求规范
  - id: specs
    generates: "specs/**/*.md"
    description: Detailed specifications
    template: spec.md
    instruction: |
      Create specification files that define WHAT.
      One spec per capability listed in proposal.
      Format: Requirements with Scenarios
    requires: [proposal]

  # 3. Design - 技术设计
  - id: design
    generates: design.md
    description: Technical design document
    template: design.md
    instruction: |
      Create the design document that explains HOW.
      Sections: Context, Goals, Decisions, Risks, Migration
    requires: [proposal]

  # 4. Tasks - 实现清单
  - id: tasks
    generates: tasks.md
    description: Implementation checklist
    template: tasks.md
    instruction: |
      Create the task list with checkboxes.
      Format: - [ ] X.Y Task description
    requires: [specs, design]

# Apply 阶段配置
apply:
  requires: [tasks]
  tracks: tasks.md
  instruction: |
    Read context files, work through tasks, mark complete.
```

### 工件详解

#### Proposal

**目的：** 回答"为什么"

**内容：**
- Why - 问题或机会
- What Changes - 具体变更
- Capabilities - 新增/修改的能力
- Impact - 影响范围

**模板：**
```markdown
## Why
<!-- 解释动机 -->

## What Changes
<!-- 描述变更 -->

## Capabilities
### New Capabilities
- `<name>`: <描述>

### Modified Capabilities
- `<existing-name>`: <变更>

## Impact
<!-- 影响范围 -->
```

#### Specs

**目的：** 定义"做什么"

**内容：**
- Requirements - 规范化需求
- Scenarios - 可测试的场景

**格式：**
```markdown
## ADDED Requirements

### Requirement: Feature Name
The system SHALL do something.

#### Scenario: Basic case
- **WHEN** user does X
- **THEN** system does Y
```

#### Design

**目的：** 解释"怎么做"

**内容：**
- Context - 背景和约束
- Goals / Non-Goals - 目标和非目标
- Decisions - 技术决策
- Risks - 风险和权衡
- Migration Plan - 迁移计划

#### Tasks

**目的：** 分解实现工作

**格式：**
```markdown
## 1. Setup
- [ ] 1.1 Create module structure
- [ ] 1.2 Add dependencies

## 2. Implementation
- [ ] 2.1 Implement core function
- [ ] 2.2 Add tests
```

## 自定义 Schema

### 创建新 Schema

**步骤 1: 创建目录结构**

```bash
# 项目本地
mkdir -p openspec/schemas/my-workflow/templates

# 用户全局
mkdir -p ~/.local/share/openspec/schemas/my-workflow/templates
```

**步骤 2: 定义 schema.yaml**

```yaml
name: my-workflow
version: 1
description: Custom workflow for my project

artifacts:
  - id: spec
    generates: spec.md
    description: Technical specification
    template: spec.md
    requires: []

  - id: tests
    generates: "tests/**/*.test.ts"
    description: Test cases
    template: test.ts
    requires: [spec]

  - id: implementation
    generates: "src/**/*.ts"
    description: Implementation
    template: implementation.ts
    requires: [spec, tests]

  - id: docs
    generates: docs.md
    description: Documentation
    template: docs.md
    requires: [implementation]

apply:
  requires: [tests, implementation]
  tracks: null  # 不跟踪进度
  instruction: |
    Run tests, ensure they pass, then implement.
```

**步骤 3: 创建模板文件**

```bash
# templates/spec.md
## Purpose
<!-- What this component does -->

## Requirements
### Requirement: <Name>
<!-- Description -->

# templates/test.ts
describe('<Component>', () => {
  it('should <behavior>', () => {
    // Test implementation
  });
});

# templates/implementation.ts
export class <Component> {
  // Implementation
}

# templates/docs.md
# <Component>

## Overview
<!-- Description -->

## API
<!-- API documentation -->
```

**步骤 4: 使用自定义 Schema**

```bash
openspec new change my-feature --schema my-workflow
```

### Schema 最佳实践

#### 1. 保持简单

```yaml
# 好：清晰的线性流程
artifacts:
  - id: spec
    requires: []
  - id: implementation
    requires: [spec]
  - id: tests
    requires: [implementation]

# 避免：过于复杂的依赖
artifacts:
  - id: a
    requires: [b, c, d]
  - id: b
    requires: [e, f]
  - id: c
    requires: [f, g]
  # ... 太复杂
```

#### 2. 使用有意义的 ID

```yaml
# 好：清晰的命名
artifacts:
  - id: api-spec
  - id: database-schema
  - id: integration-tests

# 避免：模糊的命名
artifacts:
  - id: doc1
  - id: doc2
  - id: file3
```

#### 3. 提供详细的 instruction

```yaml
# 好：具体的指导
instruction: |
  Create the API specification using OpenAPI 3.0 format.
  Include all endpoints, request/response schemas, and error codes.
  Reference the requirements document for expected behavior.

# 避免：模糊的指导
instruction: |
  Write the spec.
```

#### 4. 合理使用 glob 模式

```yaml
# 好：明确的模式
generates: "specs/**/*.md"      # 所有 spec 文件
generates: "tests/**/*.test.ts" # 所有测试文件

# 避免：过于宽泛
generates: "**/*"  # 太宽泛，难以验证
```

## Schema 列表和查询

### 列出可用 Schema

```typescript
export function listSchemas(projectRoot?: string): string[] {
  const schemas = new Set<string>();

  // 收集所有来源的 Schema
  // 1. 内置
  addSchemasFromDir(getPackageSchemasDir(), schemas);
  
  // 2. 用户全局
  addSchemasFromDir(getUserSchemasDir(), schemas);
  
  // 3. 项目本地
  if (projectRoot) {
    addSchemasFromDir(getProjectSchemasDir(projectRoot), schemas);
  }

  return Array.from(schemas).sort();
}

function addSchemasFromDir(dir: string, schemas: Set<string>): void {
  if (!fs.existsSync(dir)) return;

  for (const entry of fs.readdirSync(dir, { withFileTypes: true })) {
    if (entry.isDirectory()) {
      const schemaPath = path.join(dir, entry.name, 'schema.yaml');
      if (fs.existsSync(schemaPath)) {
        schemas.add(entry.name);
      }
    }
  }
}
```

### 获取 Schema 信息

```typescript
export interface SchemaInfo {
  name: string;
  description: string;
  artifacts: string[];
  source: 'project' | 'user' | 'package';
}

export function listSchemasWithInfo(
  projectRoot?: string
): SchemaInfo[] {
  const schemas: SchemaInfo[] = [];
  const seenNames = new Set<string>();

  // 按优先级收集（project > user > package）
  // 项目本地
  if (projectRoot) {
    collectSchemasFromDir(
      getProjectSchemasDir(projectRoot),
      'project',
      schemas,
      seenNames
    );
  }

  // 用户全局
  collectSchemasFromDir(
    getUserSchemasDir(),
    'user',
    schemas,
    seenNames
  );

  // 内置
  collectSchemasFromDir(
    getPackageSchemasDir(),
    'package',
    schemas,
    seenNames
  );

  return schemas.sort((a, b) => a.name.localeCompare(b.name));
}
```

## 变更元数据

### .openspec.yaml 文件

每个变更目录包含一个元数据文件：

```yaml
# openspec/changes/add-auth/.openspec.yaml
schema: spec-driven
created: 2025-01-30
```

### 元数据 Schema

```typescript
export const ChangeMetadataSchema = z.object({
  schema: z.string().min(1, { message: 'schema is required' }),
  created: z
    .string()
    .regex(/^\d{4}-\d{2}-\d{2}$/, {
      message: 'created must be YYYY-MM-DD format',
    })
    .optional(),
});
```

### Schema 解析优先级

```typescript
export function resolveSchemaForChange(
  changeDir: string,
  explicitSchema?: string
): string {
  // 1. 显式指定的 Schema
  if (explicitSchema) {
    return explicitSchema;
  }

  // 2. 从 .openspec.yaml 读取
  const metadataPath = path.join(changeDir, '.openspec.yaml');
  if (fs.existsSync(metadataPath)) {
    const content = fs.readFileSync(metadataPath, 'utf-8');
    const parsed = parseYaml(content);
    const result = ChangeMetadataSchema.safeParse(parsed);
    if (result.success) {
      return result.data.schema;
    }
  }

  // 3. 默认 Schema
  return 'spec-driven';
}
```

## 使用示例

### 基本使用

```typescript
// 解析 Schema
const schema = resolveSchema('spec-driven', projectRoot);
console.log(schema.name);        // 'spec-driven'
console.log(schema.version);     // 1
console.log(schema.artifacts.length); // 4

// 创建依赖图
const graph = ArtifactGraph.fromSchema(schema);
const buildOrder = graph.getBuildOrder();
console.log(buildOrder); // ['proposal', 'design', 'specs', 'tasks']
```

### 列出可用 Schema

```typescript
const schemas = listSchemas(projectRoot);
console.log('Available schemas:', schemas);
// ['spec-driven', 'my-workflow']

const schemasWithInfo = listSchemasWithInfo(projectRoot);
for (const info of schemasWithInfo) {
  console.log(`${info.name} (${info.source})`);
  console.log(`  ${info.description}`);
  console.log(`  Artifacts: ${info.artifacts.join(', ')}`);
}
```

### 验证自定义 Schema

```typescript
try {
  const content = fs.readFileSync('my-schema.yaml', 'utf-8');
  const schema = parseSchema(content);
  console.log('Schema is valid!');
} catch (error) {
  if (error instanceof SchemaValidationError) {
    console.error('Validation error:', error.message);
  }
}
```

## 下一步

- 了解 [状态管理](./04-state-management.md) 如何跟踪工件完成状态
- 学习 [指令生成](./05-instruction-generation.md) 如何使用 Schema
