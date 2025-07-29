# Tools 核心工具接口文档

## 概述

`tools.ts` 是 Gemini CLI 工具系统的核心模块，定义了所有工具必须遵循的基础接口和抽象实现。它提供了统一的工具架构，包括参数验证、执行确认、结果处理和用户交互等核心功能。这个模块是整个工具生态系统的基础。

## 主要功能

- 统一的工具接口规范
- 基础工具抽象实现
- 工具确认和交互机制
- 结果类型和显示格式定义
- 图标和位置信息管理
- 类型安全的参数处理

## 核心接口定义

### `Tool<TParams, TResult>`

**功能**: 工具基础接口，定义所有工具必须实现的方法

```typescript
export interface Tool<
  TParams = unknown,
  TResult extends ToolResult = ToolResult,
> {
  // 基础属性
  name: string;                           // 工具内部名称（用于API调用）
  displayName: string;                    // 用户友好的显示名称
  description: string;                    // 工具功能描述
  icon: Icon;                            // 工具图标
  schema: FunctionDeclaration;           // Gemini API 函数声明
  isOutputMarkdown: boolean;             // 输出是否为 Markdown 格式
  canUpdateOutput: boolean;              // 是否支持实时输出更新

  // 核心方法
  validateToolParams(params: TParams): string | null;
  getDescription(params: TParams): string;
  toolLocations(params: TParams): ToolLocation[];
  shouldConfirmExecute(params: TParams, abortSignal: AbortSignal): Promise<ToolCallConfirmationDetails | false>;
  execute(params: TParams, signal: AbortSignal, updateOutput?: (output: string) => void): Promise<TResult>;
}
```

**接口详解**:

#### 基础属性
- `name`: 工具的唯一标识符，用于 API 调用
- `displayName`: 在用户界面中显示的友好名称
- `description`: 工具功能的详细描述
- `icon`: 工具在界面中显示的图标
- `schema`: 符合 Gemini API 规范的函数声明
- `isOutputMarkdown`: 指示工具输出是否应被渲染为 Markdown
- `canUpdateOutput`: 工具是否支持流式输出更新

#### 核心方法
- `validateToolParams`: 验证工具参数的有效性
- `getDescription`: 生成工具操作的预执行描述
- `toolLocations`: 确定工具将影响的文件系统路径
- `shouldConfirmExecute`: 判断是否需要用户确认执行
- `execute`: 执行工具的核心逻辑

### `BaseTool<TParams, TResult>`

**功能**: 工具基础抽象实现，提供通用功能和默认行为

```typescript
export abstract class BaseTool<
  TParams = unknown,
  TResult extends ToolResult = ToolResult,
> implements Tool<TParams, TResult> {
  constructor(
    readonly name: string,
    readonly displayName: string,
    readonly description: string,
    readonly icon: Icon,
    readonly parameterSchema: Schema,
    readonly isOutputMarkdown: boolean = true,
    readonly canUpdateOutput: boolean = false,
  ) {}

  // 计算属性
  get schema(): FunctionDeclaration {
    return {
      name: this.name,
      description: this.description,
      parameters: this.parameterSchema,
    };
  }

  // 可重写的默认实现
  validateToolParams(params: TParams): string | null {
    return null; // 子类应该重写此方法
  }

  getDescription(params: TParams): string {
    return JSON.stringify(params);
  }

  shouldConfirmExecute(
    params: TParams,
    abortSignal: AbortSignal,
  ): Promise<ToolCallConfirmationDetails | false> {
    return Promise.resolve(false);
  }

  toolLocations(params: TParams): ToolLocation[] {
    return [];
  }

  // 抽象方法，必须由子类实现
  abstract execute(
    params: TParams,
    signal: AbortSignal,
    updateOutput?: (output: string) => void,
  ): Promise<TResult>;
}
```

## 结果类型定义

### `ToolResult`

**功能**: 工具执行结果的标准接口

```typescript
export interface ToolResult {
  summary?: string;                      // 简短的一行结果摘要
  llmContent: PartListUnion;            // 用于 LLM 历史记录的内容
  returnDisplay: ToolResultDisplay;      // 用户显示的 Markdown 内容
}
```

**字段说明**:

- `summary`: 工具执行结果的简短摘要，如 "Read 5 files", "Wrote 256 bytes to foo.txt"
- `llmContent`: 包含在 LLM 对话历史中的实际内容，可以是文本或多媒体内容
- `returnDisplay`: 向用户显示的格式化内容，支持 Markdown 渲染

### `ToolResultDisplay`

**功能**: 工具结果显示格式的联合类型

```typescript
export type ToolResultDisplay = string | FileDiff;
```

### `FileDiff`

**功能**: 文件差异显示结构

```typescript
export interface FileDiff {
  fileDiff: string;                      // 差异内容字符串
  fileName: string;                      // 文件名
  originalContent: string | null;        // 原始内容
  newContent: string;                    // 新内容
}
```

## 确认机制定义

### `ToolCallConfirmationDetails`

**功能**: 工具执行确认详情的联合类型

```typescript
export type ToolCallConfirmationDetails =
  | ToolEditConfirmationDetails
  | ToolExecuteConfirmationDetails
  | ToolMcpConfirmationDetails
  | ToolInfoConfirmationDetails;
```

### 确认类型详解

#### `ToolEditConfirmationDetails`

**功能**: 文件编辑确认详情

```typescript
export interface ToolEditConfirmationDetails {
  type: 'edit';
  title: string;                         // 确认对话框标题
  onConfirm: (outcome: ToolConfirmationOutcome, payload?: ToolConfirmationPayload) => Promise<void>;
  fileName: string;                      // 被编辑的文件名
  fileDiff: string;                      // 文件差异
  originalContent: string | null;        // 原始文件内容
  newContent: string;                    // 新文件内容
  isModifying?: boolean;                 // 是否为修改操作
}
```

#### `ToolExecuteConfirmationDetails`

**功能**: 命令执行确认详情

```typescript
export interface ToolExecuteConfirmationDetails {
  type: 'exec';
  title: string;                         // 确认标题
  onConfirm: (outcome: ToolConfirmationOutcome) => Promise<void>;
  command: string;                       // 要执行的命令
  rootCommand: string;                   // 根命令
}
```

#### `ToolMcpConfirmationDetails`

**功能**: MCP 工具确认详情

```typescript
export interface ToolMcpConfirmationDetails {
  type: 'mcp';
  title: string;                         // 确认标题
  serverName: string;                    // MCP 服务器名称
  toolName: string;                      // 工具名称
  toolDisplayName: string;               // 工具显示名称
  onConfirm: (outcome: ToolConfirmationOutcome) => Promise<void>;
}
```

#### `ToolInfoConfirmationDetails`

**功能**: 信息确认详情

```typescript
export interface ToolInfoConfirmationDetails {
  type: 'info';
  title: string;                         // 确认标题
  onConfirm: (outcome: ToolConfirmationOutcome) => Promise<void>;
  prompt: string;                        // 提示内容
  urls?: string[];                       // 相关链接
}
```

### `ToolConfirmationOutcome`

**功能**: 工具确认操作结果枚举

```typescript
export enum ToolConfirmationOutcome {
  ProceedOnce = 'proceed_once',              // 仅此次执行
  ProceedAlways = 'proceed_always',          // 始终执行
  ProceedAlwaysServer = 'proceed_always_server',  // 对此服务器始终执行
  ProceedAlwaysTool = 'proceed_always_tool',      // 对此工具始终执行
  ModifyWithEditor = 'modify_with_editor',        // 使用编辑器修改
  Cancel = 'cancel',                             // 取消操作
}
```

### `ToolConfirmationPayload`

**功能**: 工具确认负载数据

```typescript
export interface ToolConfirmationPayload {
  newContent: string;                    // 修改后的新内容
}
```

## 辅助类型定义

### `Icon`

**功能**: 工具图标枚举

```typescript
export enum Icon {
  FileSearch = 'fileSearch',             // 文件搜索图标
  Folder = 'folder',                     // 文件夹图标
  Globe = 'globe',                       // 全球/网络图标
  Hammer = 'hammer',                     // 工具/构建图标
  LightBulb = 'lightBulb',              // 灯泡/想法图标
  Pencil = 'pencil',                     // 编辑图标
  Regex = 'regex',                       // 正则表达式图标
  Terminal = 'terminal',                 // 终端图标
}
```

### `ToolLocation`

**功能**: 工具操作位置信息

```typescript
export interface ToolLocation {
  path: string;                          // 文件绝对路径
  line?: number;                         // 行号（如果已知）
}
```

## 实现示例

### 基础工具实现

```typescript
import { BaseTool, Icon, ToolResult } from './tools.js';
import { Type } from '@google/genai';

interface MyToolParams {
  input: string;
  options?: {
    verbose?: boolean;
    format?: 'json' | 'text';
  };
}

class MyTool extends BaseTool<MyToolParams, ToolResult> {
  constructor() {
    super(
      'my_tool',                         // name
      'My Custom Tool',                  // displayName
      'A custom tool that processes input and returns formatted output',
      Icon.Hammer,                       // icon
      {                                  // parameterSchema
        type: Type.OBJECT,
        properties: {
          input: {
            type: Type.STRING,
            description: 'The input text to process'
          },
          options: {
            type: Type.OBJECT,
            properties: {
              verbose: {
                type: Type.BOOLEAN,
                description: 'Enable verbose output'
              },
              format: {
                type: Type.STRING,
                enum: ['json', 'text'],
                description: 'Output format'
              }
            }
          }
        },
        required: ['input']
      },
      true,                              // isOutputMarkdown
      false                              // canUpdateOutput
    );
  }

  validateToolParams(params: MyToolParams): string | null {
    if (!params.input || params.input.trim().length === 0) {
      return 'Input parameter cannot be empty';
    }
    
    if (params.options?.format && !['json', 'text'].includes(params.options.format)) {
      return 'Format must be either "json" or "text"';
    }
    
    return null;
  }

  getDescription(params: MyToolParams): string {
    const format = params.options?.format || 'text';
    const verbose = params.options?.verbose ? ' (verbose)' : '';
    return `Processing "${params.input}" as ${format}${verbose}`;
  }

  toolLocations(params: MyToolParams): ToolLocation[] {
    // 如果工具操作特定文件，返回文件位置
    return [];
  }

  async shouldConfirmExecute(
    params: MyToolParams,
    abortSignal: AbortSignal
  ): Promise<ToolCallConfirmationDetails | false> {
    // 对于敏感操作可能需要确认
    if (params.input.includes('delete') || params.input.includes('remove')) {
      return {
        type: 'info',
        title: 'Confirm Potentially Destructive Operation',
        prompt: `Are you sure you want to process: "${params.input}"?`,
        onConfirm: async (outcome) => {
          if (outcome === ToolConfirmationOutcome.Cancel) {
            throw new Error('Operation cancelled by user');
          }
        }
      };
    }
    
    return false;
  }

  async execute(
    params: MyToolParams,
    signal: AbortSignal,
    updateOutput?: (output: string) => void
  ): Promise<ToolResult> {
    const validationError = this.validateToolParams(params);
    if (validationError) {
      return {
        llmContent: `Error: ${validationError}`,
        returnDisplay: `## Error\n\n${validationError}`
      };
    }

    try {
      // 模拟处理过程
      if (updateOutput) {
        updateOutput('Processing input...');
      }

      // 检查是否被取消
      if (signal.aborted) {
        throw new Error('Operation was cancelled');
      }

      // 实际处理逻辑
      const result = await this.processInput(params.input, params.options);

      // 格式化结果
      const output = params.options?.format === 'json' 
        ? JSON.stringify(result, null, 2)
        : result.toString();

      return {
        summary: `Processed input with ${result.length} characters`,
        llmContent: output,
        returnDisplay: `## Processing Result\n\n\`\`\`${params.options?.format || 'text'}\n${output}\n\`\`\``
      };
    } catch (error) {
      const errorMessage = error instanceof Error ? error.message : String(error);
      return {
        llmContent: `Error processing input: ${errorMessage}`,
        returnDisplay: `## Error\n\n${errorMessage}`
      };
    }
  }

  private async processInput(input: string, options?: MyToolParams['options']): Promise<string> {
    // 实际的处理逻辑
    let result = input.toUpperCase();
    
    if (options?.verbose) {
      result = `[VERBOSE] Processing: ${result} [Length: ${input.length}]`;
    }
    
    return result;
  }
}
```

### 支持流式输出的工具

```typescript
class StreamingTool extends BaseTool<{ query: string }, ToolResult> {
  constructor() {
    super(
      'streaming_tool',
      'Streaming Tool',
      'A tool that provides streaming output',
      Icon.Terminal,
      {
        type: Type.OBJECT,
        properties: {
          query: {
            type: Type.STRING,
            description: 'Query to process'
          }
        },
        required: ['query']
      },
      true,  // isOutputMarkdown
      true   // canUpdateOutput - 支持流式输出
    );
  }

  async execute(
    params: { query: string },
    signal: AbortSignal,
    updateOutput?: (output: string) => void
  ): Promise<ToolResult> {
    let currentOutput = '';
    const steps = ['Initializing...', 'Processing query...', 'Generating results...', 'Finalizing...'];

    for (let i = 0; i < steps.length; i++) {
      if (signal.aborted) {
        throw new Error('Operation cancelled');
      }

      currentOutput += `\n${steps[i]}`;
      
      // 更新实时输出
      if (updateOutput) {
        updateOutput(`## Progress\n${currentOutput}`);
      }

      // 模拟处理时间
      await new Promise(resolve => setTimeout(resolve, 1000));
    }

    const finalResult = `Query "${params.query}" processed successfully!`;
    currentOutput += `\n\n**Result:** ${finalResult}`;

    return {
      summary: `Processed query: ${params.query}`,
      llmContent: finalResult,
      returnDisplay: `## Final Result\n${currentOutput}`
    };
  }
}
```

### 需要用户确认的工具

```typescript
class ConfirmableTool extends BaseTool<{ action: string; target: string }, ToolResult> {
  constructor() {
    super(
      'confirmable_tool',
      'Confirmable Tool',
      'A tool that requires user confirmation for certain actions',
      Icon.Hammer,
      {
        type: Type.OBJECT,
        properties: {
          action: {
            type: Type.STRING,
            description: 'Action to perform'
          },
          target: {
            type: Type.STRING,
            description: 'Target of the action'
          }
        },
        required: ['action', 'target']
      }
    );
  }

  async shouldConfirmExecute(
    params: { action: string; target: string },
    abortSignal: AbortSignal
  ): Promise<ToolCallConfirmationDetails | false> {
    const dangerousActions = ['delete', 'remove', 'destroy', 'wipe'];
    
    if (dangerousActions.some(action => params.action.toLowerCase().includes(action))) {
      return {
        type: 'exec',
        title: 'Confirm Destructive Action',
        command: `${params.action} ${params.target}`,
        rootCommand: params.action,
        onConfirm: async (outcome) => {
          switch (outcome) {
            case ToolConfirmationOutcome.Cancel:
              throw new Error('Action cancelled by user');
            case ToolConfirmationOutcome.ProceedOnce:
              console.log('User confirmed action for this execution');
              break;
            case ToolConfirmationOutcome.ProceedAlways:
              console.log('User confirmed action for all future executions');
              break;
          }
        }
      };
    }

    return false;
  }

  async execute(
    params: { action: string; target: string },
    signal: AbortSignal
  ): Promise<ToolResult> {
    // 执行实际操作
    const result = `Performed ${params.action} on ${params.target}`;
    
    return {
      summary: result,
      llmContent: result,
      returnDisplay: `## Action Completed\n\n${result}`
    };
  }
}
```

## 高级功能

### 工具装饰器模式

```typescript
// 基础装饰器
abstract class ToolDecorator<TParams, TResult extends ToolResult> implements Tool<TParams, TResult> {
  constructor(protected wrappedTool: Tool<TParams, TResult>) {}

  // 委托所有属性到被包装的工具
  get name() { return this.wrappedTool.name; }
  get displayName() { return this.wrappedTool.displayName; }
  get description() { return this.wrappedTool.description; }
  get icon() { return this.wrappedTool.icon; }
  get schema() { return this.wrappedTool.schema; }
  get isOutputMarkdown() { return this.wrappedTool.isOutputMarkdown; }
  get canUpdateOutput() { return this.wrappedTool.canUpdateOutput; }

  validateToolParams(params: TParams): string | null {
    return this.wrappedTool.validateToolParams(params);
  }

  getDescription(params: TParams): string {
    return this.wrappedTool.getDescription(params);
  }

  toolLocations(params: TParams): ToolLocation[] {
    return this.wrappedTool.toolLocations(params);
  }

  shouldConfirmExecute(
    params: TParams,
    abortSignal: AbortSignal
  ): Promise<ToolCallConfirmationDetails | false> {
    return this.wrappedTool.shouldConfirmExecute(params, abortSignal);
  }

  abstract execute(
    params: TParams,
    signal: AbortSignal,
    updateOutput?: (output: string) => void
  ): Promise<TResult>;
}

// 日志装饰器
class LoggingToolDecorator<TParams, TResult extends ToolResult> 
  extends ToolDecorator<TParams, TResult> {
  
  async execute(
    params: TParams,
    signal: AbortSignal,
    updateOutput?: (output: string) => void
  ): Promise<TResult> {
    console.log(`[${this.name}] Starting execution with params:`, params);
    const startTime = Date.now();
    
    try {
      const result = await this.wrappedTool.execute(params, signal, updateOutput);
      const duration = Date.now() - startTime;
      console.log(`[${this.name}] Completed successfully in ${duration}ms`);
      return result;
    } catch (error) {
      const duration = Date.now() - startTime;
      console.error(`[${this.name}] Failed after ${duration}ms:`, error);
      throw error;
    }
  }
}

// 重试装饰器
class RetryToolDecorator<TParams, TResult extends ToolResult> 
  extends ToolDecorator<TParams, TResult> {
  
  constructor(
    wrappedTool: Tool<TParams, TResult>,
    private maxRetries: number = 3,
    private retryDelay: number = 1000
  ) {
    super(wrappedTool);
  }

  async execute(
    params: TParams,
    signal: AbortSignal,
    updateOutput?: (output: string) => void
  ): Promise<TResult> {
    let lastError: Error | null = null;
    
    for (let attempt = 1; attempt <= this.maxRetries; attempt++) {
      try {
        return await this.wrappedTool.execute(params, signal, updateOutput);
      } catch (error) {
        lastError = error instanceof Error ? error : new Error(String(error));
        
        if (attempt < this.maxRetries && !signal.aborted) {
          console.warn(`[${this.name}] Attempt ${attempt} failed, retrying in ${this.retryDelay}ms...`);
          await new Promise(resolve => setTimeout(resolve, this.retryDelay));
        }
      }
    }
    
    throw new Error(`Tool failed after ${this.maxRetries} attempts. Last error: ${lastError?.message}`);
  }
}

// 缓存装饰器
class CachingToolDecorator<TParams, TResult extends ToolResult> 
  extends ToolDecorator<TParams, TResult> {
  
  private cache = new Map<string, { result: TResult; timestamp: number }>();
  
  constructor(
    wrappedTool: Tool<TParams, TResult>,
    private ttl: number = 300000 // 5 minutes
  ) {
    super(wrappedTool);
  }

  async execute(
    params: TParams,
    signal: AbortSignal,
    updateOutput?: (output: string) => void
  ): Promise<TResult> {
    const cacheKey = JSON.stringify(params);
    const cached = this.cache.get(cacheKey);
    const now = Date.now();
    
    if (cached && (now - cached.timestamp) < this.ttl) {
      console.log(`[${this.name}] Cache hit for params:`, params);
      return cached.result;
    }
    
    const result = await this.wrappedTool.execute(params, signal, updateOutput);
    this.cache.set(cacheKey, { result, timestamp: now });
    
    return result;
  }
  
  clearCache(): void {
    this.cache.clear();
  }
}

// 使用装饰器
const myTool = new MyTool();
const decoratedTool = new LoggingToolDecorator(
  new RetryToolDecorator(
    new CachingToolDecorator(myTool, 600000), // 10 分钟缓存
    3, // 最多重试3次
    2000 // 重试间隔2秒
  )
);
```

### 工具工厂模式

```typescript
interface ToolFactory {
  createTool(type: string, config: any): Tool<any, any>;
  registerToolType(type: string, factory: (config: any) => Tool<any, any>): void;
}

class DefaultToolFactory implements ToolFactory {
  private factories = new Map<string, (config: any) => Tool<any, any>>();

  registerToolType(type: string, factory: (config: any) => Tool<any, any>): void {
    this.factories.set(type, factory);
  }

  createTool(type: string, config: any): Tool<any, any> {
    const factory = this.factories.get(type);
    if (!factory) {
      throw new Error(`Unknown tool type: ${type}`);
    }
    return factory(config);
  }
}

// 注册工具类型
const toolFactory = new DefaultToolFactory();

toolFactory.registerToolType('my_tool', (config) => new MyTool());
toolFactory.registerToolType('streaming_tool', (config) => new StreamingTool());
toolFactory.registerToolType('confirmable_tool', (config) => new ConfirmableTool());

// 使用工厂创建工具
const toolConfig = { type: 'my_tool', options: {} };
const tool = toolFactory.createTool(toolConfig.type, toolConfig.options);
```

### 工具组合模式

```typescript
class CompositeTool<TParams, TResult extends ToolResult> extends BaseTool<TParams, TResult> {
  constructor(
    name: string,
    displayName: string,
    description: string,
    icon: Icon,
    parameterSchema: Schema,
    private tools: Tool<any, any>[]
  ) {
    super(name, displayName, description, icon, parameterSchema);
  }

  async execute(
    params: TParams,
    signal: AbortSignal,
    updateOutput?: (output: string) => void
  ): Promise<TResult> {
    const results: any[] = [];
    let combinedOutput = '';

    for (let i = 0; i < this.tools.length; i++) {
      if (signal.aborted) {
        throw new Error('Composite operation cancelled');
      }

      const tool = this.tools[i];
      if (updateOutput) {
        updateOutput(`## Step ${i + 1}: ${tool.displayName}\n${combinedOutput}`);
      }

      try {
        const result = await tool.execute(
          this.extractParamsForTool(params, i),
          signal,
          updateOutput
        );
        results.push(result);
        combinedOutput += `\n### ${tool.displayName} Result\n${result.returnDisplay}\n`;
      } catch (error) {
        combinedOutput += `\n### ${tool.displayName} Error\n${error}\n`;
        throw error;
      }
    }

    return {
      summary: `Executed ${this.tools.length} tools successfully`,
      llmContent: results.map(r => r.llmContent).join('\n\n'),
      returnDisplay: `## Composite Tool Results\n${combinedOutput}`
    } as TResult;
  }

  private extractParamsForTool(params: TParams, toolIndex: number): any {
    // 根据工具索引提取相应的参数
    // 这里需要根据具体的参数结构来实现
    return params;
  }
}
```

## 错误处理最佳实践

### 标准化错误处理

```typescript
class ErrorHandlingTool extends BaseTool<any, ToolResult> {
  constructor() {
    super(
      'error_handling_tool',
      'Error Handling Tool',
      'Demonstrates proper error handling patterns',
      Icon.Hammer,
      {
        type: Type.OBJECT,
        properties: {
          input: { type: Type.STRING }
        },
        required: ['input']
      }
    );
  }

  async execute(
    params: { input: string },
    signal: AbortSignal,
    updateOutput?: (output: string) => void
  ): Promise<ToolResult> {
    try {
      // 参数验证
      const validationError = this.validateToolParams(params);
      if (validationError) {
        return this.createErrorResult(
          'Parameter Validation Error',
          validationError,
          'validation'
        );
      }

      // 取消检查
      if (signal.aborted) {
        return this.createErrorResult(
          'Operation Cancelled',
          'The operation was cancelled by the user',
          'cancellation'
        );
      }

      // 实际处理逻辑
      const result = await this.performOperation(params.input, signal, updateOutput);
      
      return {
        summary: `Successfully processed: ${params.input}`,
        llmContent: result,
        returnDisplay: `## Success\n\n${result}`
      };

    } catch (error) {
      return this.handleExecutionError(error);
    }
  }

  private createErrorResult(
    title: string,
    message: string,
    category: 'validation' | 'execution' | 'cancellation' | 'timeout'
  ): ToolResult {
    const errorIcon = {
      validation: '⚠️',
      execution: '❌',
      cancellation: '🚫',
      timeout: '⏰'
    }[category];

    return {
      summary: `Error: ${title}`,
      llmContent: `Error: ${message}`,
      returnDisplay: `## ${errorIcon} ${title}\n\n${message}`
    };
  }

  private handleExecutionError(error: unknown): ToolResult {
    if (error instanceof Error) {
      // 根据错误类型进行分类处理
      if (error.name === 'AbortError') {
        return this.createErrorResult(
          'Operation Aborted',
          'The operation was aborted',
          'cancellation'
        );
      }

      if (error.name === 'TimeoutError') {
        return this.createErrorResult(
          'Operation Timeout',
          'The operation timed out',
          'timeout'
        );
      }

      // 通用错误处理
      return this.createErrorResult(
        'Execution Error',
        error.message,
        'execution'
      );
    }

    // 未知错误类型
    return this.createErrorResult(
      'Unknown Error',
      String(error),
      'execution'
    );
  }

  private async performOperation(
    input: string,
    signal: AbortSignal,
    updateOutput?: (output: string) => void
  ): Promise<string> {
    // 模拟操作实现
    return `Processed: ${input}`;
  }
}
```

## 性能优化策略

### 懒加载工具

```typescript
class LazyTool<TParams, TResult extends ToolResult> implements Tool<TParams, TResult> {
  private toolInstance: Tool<TParams, TResult> | null = null;

  constructor(
    private toolFactory: () => Tool<TParams, TResult>,
    public readonly name: string,
    public readonly displayName: string,
    public readonly description: string,
    public readonly icon: Icon,
    public readonly schema: FunctionDeclaration,
    public readonly isOutputMarkdown: boolean = true,
    public readonly canUpdateOutput: boolean = false
  ) {}

  private ensureToolLoaded(): Tool<TParams, TResult> {
    if (!this.toolInstance) {
      this.toolInstance = this.toolFactory();
    }
    return this.toolInstance;
  }

  validateToolParams(params: TParams): string | null {
    return this.ensureToolLoaded().validateToolParams(params);
  }

  getDescription(params: TParams): string {
    return this.ensureToolLoaded().getDescription(params);
  }

  toolLocations(params: TParams): ToolLocation[] {
    return this.ensureToolLoaded().toolLocations(params);
  }

  shouldConfirmExecute(
    params: TParams,
    abortSignal: AbortSignal
  ): Promise<ToolCallConfirmationDetails | false> {
    return this.ensureToolLoaded().shouldConfirmExecute(params, abortSignal);
  }

  execute(
    params: TParams,
    signal: AbortSignal,
    updateOutput?: (output: string) => void
  ): Promise<TResult> {
    return this.ensureToolLoaded().execute(params, signal, updateOutput);
  }
}

// 使用懒加载工具
const lazyTool = new LazyTool(
  () => new MyTool(), // 工具工厂函数
  'my_tool',
  'My Tool',
  'A lazily loaded tool',
  Icon.Hammer,
  {
    name: 'my_tool',
    description: 'A lazily loaded tool',
    parameters: { type: Type.OBJECT, properties: {} }
  }
);
```

## 最佳实践总结

### 工具设计原则

1. **单一职责**: 每个工具应该专注于单一明确的功能
2. **可测试性**: 工具应该易于单元测试和集成测试
3. **错误处理**: 提供清晰的错误信息和恢复机制
4. **用户体验**: 提供直观的描述和适当的确认机制
5. **性能考虑**: 对于耗时操作提供进度反馈和取消机制

### 实现建议

```typescript
// ✅ 推荐：清晰的参数类型定义
interface WellDefinedParams {
  requiredField: string;
  optionalField?: number;
  options?: {
    mode: 'fast' | 'thorough';
    includeMetadata: boolean;
  };
}

// ✅ 推荐：详细的参数验证
validateToolParams(params: WellDefinedParams): string | null {
  if (!params.requiredField?.trim()) {
    return 'requiredField cannot be empty';
  }
  
  if (params.optionalField !== undefined && params.optionalField < 0) {
    return 'optionalField must be non-negative';
  }
  
  return null;
}

// ✅ 推荐：有意义的描述生成
getDescription(params: WellDefinedParams): string {
  const mode = params.options?.mode || 'fast';
  const metadata = params.options?.includeMetadata ? ' with metadata' : '';
  return `Processing "${params.requiredField}" in ${mode} mode${metadata}`;
}

// ✅ 推荐：结构化的结果返回
async execute(params: WellDefinedParams, signal: AbortSignal): Promise<ToolResult> {
  try {
    const result = await this.performProcessing(params, signal);
    
    return {
      summary: `Processed ${params.requiredField} successfully`,
      llmContent: result.data,
      returnDisplay: this.formatResultDisplay(result)
    };
  } catch (error) {
    return this.handleError(error, params);
  }
}
```

### 测试策略

```typescript
// 工具测试示例
describe('MyTool', () => {
  let tool: MyTool;
  
  beforeEach(() => {
    tool = new MyTool();
  });

  describe('validateToolParams', () => {
    it('should reject empty input', () => {
      const result = tool.validateToolParams({ input: '' });
      expect(result).toContain('cannot be empty');
    });

    it('should accept valid params', () => {
      const result = tool.validateToolParams({ input: 'valid input' });
      expect(result).toBeNull();
    });
  });

  describe('execute', () => {
    it('should process input successfully', async () => {
      const result = await tool.execute(
        { input: 'test input' },
        new AbortController().signal
      );
      
      expect(result.summary).toContain('successfully');
      expect(result.llmContent).toBeDefined();
      expect(result.returnDisplay).toBeDefined();
    });

    it('should handle cancellation', async () => {
      const controller = new AbortController();
      controller.abort();
      
      const result = await tool.execute(
        { input: 'test input' },
        controller.signal
      );
      
      expect(result.llmContent).toContain('cancelled');
    });
  });
});
```