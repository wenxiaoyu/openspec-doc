# 1. 架构总览

## 系统架构图

```
┌─────────────────────────────────────────────────────────────┐
│                         用户层                               │
│  CLI 命令 (/opsx:new, /opsx:ff, /opsx:apply)               │
└────────────────────┬────────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────────┐
│                      命令路由层                              │
│  Commander.js → 命令解析 → 参数验证 → 遥测                  │
└────────────────────┬────────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────────┐
│                     工作流引擎层                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │ New Change   │  │   Continue   │  │    Apply     │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
└────────────────────┬────────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────────┐
│                   核心引擎层                                 │
│  ┌──────────────────────────────────────────────────┐      │
│  │           Artifact Graph Engine                  │      │
│  │  • 依赖图构建  • 拓扑排序  • 状态检测           │      │
│  └──────────────────────────────────────────────────┘      │
│  ┌──────────────────────────────────────────────────┐      │
│  │           Instruction Loader                     │      │
│  │  • 模板加载  • 上下文注入  • 指令生成           │      │
│  └──────────────────────────────────────────────────┘      │
└────────────────────┬────────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────────┐
│                    数据层                                    │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │   Schema     │  │   Templates  │  │  File System │     │
│  │   (YAML)     │  │  (Markdown)  │  │   (State)    │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
└─────────────────────────────────────────────────────────────┘
```

## 核心模块职责

### 1. CLI 层 (`src/cli/`)

**职责：**
- 解析命令行参数
- 路由到对应的命令处理器
- 处理全局选项（如 `--no-color`）
- 集成遥测系统

**关键文件：**
- `src/cli/index.ts` - 主入口，Commander.js 配置
- `bin/openspec.js` - 可执行文件入口

**实现细节：**

```typescript
// 命令注册模式
program
  .command('new change <name>')
  .option('--schema <name>', 'Workflow schema')
  .action(async (name, options) => {
    await newChangeCommand(name, options);
  });

// Hook 机制
program.hook('preAction', async (thisCommand, actionCommand) => {
  await maybeShowTelemetryNotice();
  await trackCommand(getCommandPath(actionCommand), version);
});
```

### 2. 命令层 (`src/commands/`)

**职责：**
- 实现具体的命令逻辑
- 处理用户交互（提示、选择）
- 调用核心引擎完成任务
- 格式化输出

**关键模块：**
- `workflow/` - 工作流命令（new, status, instructions, apply）
- `change.ts` - 变更管理命令
- `spec.ts` - 规范管理命令
- `validate.ts` - 验证命令

**设计模式：**
- **命令模式** - 每个命令是独立的类或函数
- **策略模式** - 不同的输出格式（JSON/文本）

### 3. 核心引擎层 (`src/core/`)

**职责：**
- 实现核心业务逻辑
- 管理工件依赖关系
- 生成 AI 指令
- 验证文档结构

**关键子系统：**

#### 3.1 Artifact Graph (`artifact-graph/`)
```
artifact-graph/
├── types.ts          # 类型定义
├── graph.ts          # 依赖图实现
├── schema.ts         # Schema 解析
├── state.ts          # 状态检测
├── resolver.ts       # Schema 解析器
└── instruction-loader.ts  # 指令加载
```

#### 3.2 解析器 (`parsers/`)
```
parsers/
├── markdown-parser.ts      # Markdown 解析
├── change-parser.ts        # Change 文档解析
└── requirement-blocks.ts   # Requirement 块解析
```

#### 3.3 验证器 (`validation/`)
```
validation/
├── types.ts       # 验证类型
├── validator.ts   # 验证器实现
└── constants.ts   # 验证规则常量
```

### 4. 工具层 (`src/utils/`)

**职责：**
- 提供通用工具函数
- 文件系统操作
- 交互式提示
- 任务进度跟踪

**关键模块：**
- `file-system.ts` - 文件操作工具
- `change-utils.ts` - 变更管理工具
- `interactive.ts` - 交互式提示
- `task-progress.ts` - 任务进度

## 数据流

### 创建新变更的数据流

```
用户输入: openspec new change add-auth
         ↓
CLI 解析: { command: 'new', subcommand: 'change', name: 'add-auth' }
         ↓
命令处理: newChangeCommand(name, options)
         ↓
验证名称: validateChangeName('add-auth') → valid
         ↓
创建目录: createChange(projectRoot, 'add-auth', { schema })
         ↓
写入元数据: .openspec.yaml { schema: 'spec-driven' }
         ↓
输出结果: "Created change 'add-auth' at openspec/changes/add-auth/"
```

### 生成指令的数据流

```
用户输入: openspec instructions proposal --change add-auth
         ↓
加载上下文: loadChangeContext(projectRoot, 'add-auth')
         ├─ 解析 Schema: resolveSchema('spec-driven')
         ├─ 构建依赖图: ArtifactGraph.fromSchema(schema)
         └─ 检测完成状态: detectCompleted(graph, changeDir)
         ↓
生成指令: generateInstructions(context, 'proposal')
         ├─ 加载模板: loadTemplate('spec-driven', 'proposal.md')
         ├─ 读取配置: readProjectConfig(projectRoot)
         ├─ 收集依赖: getDependencyInfo(artifact, graph, completed)
         └─ 组装指令: { template, context, rules, dependencies }
         ↓
格式化输出: printInstructionsText(instructions)
```

## 设计哲学

### 1. 状态即文件 (State as Files)

**原则：** 不使用数据库，所有状态通过文件系统表达

**实现：**
- 工件完成状态 = 文件是否存在
- 变更元数据 = `.openspec.yaml` 文件
- 任务进度 = `tasks.md` 中的 checkbox

**优势：**
- 易于理解和调试
- 与 Git 完美集成
- 无需额外的存储系统

### 2. 声明式工作流 (Declarative Workflows)

**原则：** 通过 Schema 声明工作流，而非硬编码

**实现：**
```yaml
# Schema 定义工作流
artifacts:
  - id: proposal
    requires: []
  - id: specs
    requires: [proposal]
```

**优势：**
- 用户可自定义工作流
- 易于扩展和修改
- 工作流逻辑与引擎分离

### 3. 适配器模式 (Adapter Pattern)

**原则：** 工具无关的内容 + 工具特定的适配器

**实现：**
```typescript
// 工具无关
CommandContent { id, name, body }

// 工具特定
claudeAdapter.formatFile(content) → Claude 格式
cursorAdapter.formatFile(content) → Cursor 格式
```

**优势：**
- 单一数据源
- 易于添加新工具
- 维护成本低

### 4. 渐进式增强 (Progressive Enhancement)

**原则：** 核心功能简单，高级功能可选

**实现：**
- 基础：文件 + 模板
- 增强：项目配置（context, rules）
- 高级：自定义 Schema

**优势：**
- 学习曲线平缓
- 适应不同复杂度的项目
- 不强制使用所有功能

## 技术栈

### 核心依赖

| 依赖 | 版本 | 用途 |
|------|------|------|
| TypeScript | ^5.9.3 | 类型安全 |
| Commander.js | ^14.0.0 | CLI 框架 |
| Zod | ^4.0.17 | Schema 验证 |
| YAML | ^2.8.2 | YAML 解析 |
| Inquirer | ^7.8.0 | 交互式提示 |
| Chalk | ^5.5.0 | 终端颜色 |
| Ora | ^8.2.0 | 加载动画 |

### 开发依赖

| 依赖 | 用途 |
|------|------|
| Vitest | 单元测试 |
| ESLint | 代码检查 |
| Changesets | 版本管理 |

## 目录结构

```
openspec/
├── bin/                    # 可执行文件
│   └── openspec.js
├── src/
│   ├── cli/               # CLI 入口
│   ├── commands/          # 命令实现
│   │   └── workflow/      # 工作流命令
│   ├── core/              # 核心引擎
│   │   ├── artifact-graph/  # 依赖图系统
│   │   ├── parsers/         # 解析器
│   │   ├── validation/      # 验证器
│   │   ├── command-generation/  # 命令生成
│   │   └── templates/       # 模板系统
│   ├── utils/             # 工具函数
│   └── telemetry/         # 遥测系统
├── schemas/               # 内置 Schema
│   └── spec-driven/
│       ├── schema.yaml
│       └── templates/
├── test/                  # 测试文件
└── docs/                  # 文档
```

## 扩展点

系统提供以下扩展点：

1. **自定义 Schema** - 定义新的工作流
2. **自定义模板** - 修改工件模板
3. **项目配置** - 添加 context 和 rules
4. **工具适配器** - 支持新的 AI 工具
5. **验证规则** - 添加自定义验证

## 下一步

- 深入了解 [Artifact Graph 系统](./02-artifact-graph.md)
- 学习如何 [自定义 Schema](./03-schema-system.md)
