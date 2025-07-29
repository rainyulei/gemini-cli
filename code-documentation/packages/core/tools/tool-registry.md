# ToolRegistry 工具注册表文档

## 概述

`ToolRegistry` 是 Gemini CLI 的核心工具管理系统，负责注册、发现、管理和执行各种工具。它支持内置工具注册、命令行工具发现、MCP 服务器工具集成等多种工具来源，提供统一的工具访问接口。

## 主要功能

- 工具注册和管理
- 命令行工具自动发现
- MCP 服务器工具集成
- 工具 Schema 验证和清理
- 工具执行和结果处理
- 工具分组和过滤
- 动态工具加载和卸载

## 核心类

### `ToolRegistry`

**功能**: 工具注册表主类，管理所有工具的生命周期

```typescript
export class ToolRegistry {
  private tools: Map<string, Tool> = new Map();
  private config: Config;
  
  constructor(config: Config) {
    this.config = config;
  }
}
```

### `DiscoveredTool`

**功能**: 从命令行发现的工具包装器

```typescript
export class DiscoveredTool extends BaseTool<ToolParams, ToolResult> {
  constructor(
    private readonly config: Config,
    name: string,
    readonly description: string,
    readonly parameterSchema: Record<string, unknown>,
  ) {
    // 自动生成工具描述，包含发现和调用信息
    super(name, name, enhancedDescription, Icon.Hammer, parameterSchema);
  }
}
```

### `DiscoveredMCPTool`

**功能**: 从 MCP 服务器发现的工具包装器

```typescript
export class DiscoveredMCPTool extends BaseTool<ToolParams, ToolResult> {
  private static readonly allowlist: Set<string> = new Set();
  
  constructor(
    private readonly mcpTool: CallableTool,
    readonly serverName: string,
    readonly serverToolName: string,
    description: string,
    readonly parameterSchemaJson: unknown,
    readonly timeout?: number,
    readonly trust?: boolean,
    nameOverride?: string,
  ) {
    super(
      nameOverride ?? generateValidName(serverToolName),
      `${serverToolName} (${serverName} MCP Server)`,
      description,
      Icon.Hammer,
      { type: Type.OBJECT }, // MCP 工具使用动态 Schema
      true, // isOutputMarkdown
      false, // canUpdateOutput
    );
  }
}
```

**核心特性**:

- **静态允许列表**: 跟踪已授权的服务器和工具
- **完全限定名称**: 支持名称冲突解决
- **信任模式**: 可配置的安全级别
- **动态 Schema**: 使用 `parametersJsonSchema` 而非标准 Schema

#### `asFullyQualifiedTool(): DiscoveredMCPTool`

**功能**: 创建完全限定名称的工具副本，用于解决名称冲突

```typescript
asFullyQualifiedTool(): DiscoveredMCPTool {
  return new DiscoveredMCPTool(
    this.mcpTool,
    this.serverName,
    this.serverToolName,
    this.description,
    this.parameterSchemaJson,
    this.timeout,
    this.trust,
    `${this.serverName}__${this.serverToolName}`, // 完全限定名
  );
}
```

#### `async shouldConfirmExecute(): Promise<ToolCallConfirmationDetails | false>`

**功能**: MCP 工具执行前的确认机制

```typescript
async shouldConfirmExecute(): Promise<ToolCallConfirmationDetails | false> {
  const serverAllowListKey = this.serverName;
  const toolAllowListKey = `${this.serverName}.${this.serverToolName}`;

  if (this.trust) {
    return false; // 信任的服务器无需确认
  }

  if (DiscoveredMCPTool.allowlist.has(serverAllowListKey) || 
      DiscoveredMCPTool.allowlist.has(toolAllowListKey)) {
    return false; // 已在允许列表中
  }

  return {
    type: 'mcp',
    title: 'Confirm MCP Tool Execution',
    serverName: this.serverName,
    toolName: this.serverToolName,
    onConfirm: async (outcome) => {
      if (outcome === ToolConfirmationOutcome.ProceedAlwaysServer) {
        DiscoveredMCPTool.allowlist.add(serverAllowListKey);
      } else if (outcome === ToolConfirmationOutcome.ProceedAlwaysTool) {
        DiscoveredMCPTool.allowlist.add(toolAllowListKey);
      }
    },
  };
}
```

#### `generateValidName(name: string): string`

**功能**: 生成符合 Gemini API 要求的有效工具名称

```typescript
export function generateValidName(name: string) {
  // 替换无效字符为下划线
  let validToolname = name.replace(/[^a-zA-Z0-9_.-]/g, '_');
  
  // 长度限制为 63 字符
  if (validToolname.length > 63) {
    validToolname = validToolname.slice(0, 28) + '___' + validToolname.slice(-32);
  }
  return validToolname;
}
```

## 核心方法

### 工具注册

#### `registerTool(tool: Tool): void`

**功能**: 注册单个工具到注册表

```typescript
registerTool(tool: Tool): void {
  if (this.tools.has(tool.name)) {
    if (tool instanceof DiscoveredMCPTool) {
      tool = tool.asFullyQualifiedTool();  // 处理MCP工具命名冲突
    } else {
      console.warn(`Tool "${tool.name}" already registered. Overwriting.`);
    }
  }
  this.tools.set(tool.name, tool);
}
```

**注册流程**:

1. **名称冲突检查**: 检查工具名是否已存在
2. **MCP 工具处理**: 对 MCP 工具使用完全限定名
3. **覆盖警告**: 对重复注册发出警告
4. **存储工具**: 将工具保存到内部 Map

### 工具发现

#### `async discoverAllTools(): Promise<void>`

**功能**: 发现所有可用工具（命令行 + MCP）

```typescript
async discoverAllTools(): Promise<void> {
  // 清理之前发现的工具
  for (const tool of this.tools.values()) {
    if (tool instanceof DiscoveredTool || tool instanceof DiscoveredMCPTool) {
      this.tools.delete(tool.name);
    }
  }
  
  // 发现命令行工具
  await this.discoverAndRegisterToolsFromCommand();
  
  // 发现MCP工具
  await discoverMcpTools(
    this.config.getMcpServers() ?? {},
    this.config.getMcpServerCommand(),
    this,
    this.config.getPromptRegistry(),
    this.config.getDebugMode(),
  );
}
```

**发现流程**:

1. **清理旧工具**: 移除之前发现的工具，保留内置工具
2. **命令行发现**: 执行配置的发现命令
3. **MCP 发现**: 连接 MCP 服务器获取工具列表
4. **注册新工具**: 将发现的工具注册到系统

#### `async discoverMcpTools(): Promise<void>`

**功能**: 仅发现 MCP 服务器工具

```typescript
async discoverMcpTools(): Promise<void> {
  // 仅清理MCP工具
  for (const tool of this.tools.values()) {
    if (tool instanceof DiscoveredMCPTool) {
      this.tools.delete(tool.name);
    }
  }
  
  await discoverMcpTools(/* MCP参数 */);
}
```

#### `async discoverToolsForServer(serverName: string): Promise<void>`

**功能**: 发现特定 MCP 服务器的工具

```typescript
async discoverToolsForServer(serverName: string): Promise<void> {
  // 移除该服务器的旧工具
  for (const [name, tool] of this.tools.entries()) {
    if (tool instanceof DiscoveredMCPTool && tool.serverName === serverName) {
      this.tools.delete(name);
    }
  }
  
  const mcpServers = this.config.getMcpServers() ?? {};
  const serverConfig = mcpServers[serverName];
  if (serverConfig) {
    await discoverMcpTools({ [serverName]: serverConfig }, /* 其他参数 */);
  }
}
```

### 命令行工具发现

#### `private async discoverAndRegisterToolsFromCommand(): Promise<void>`

**功能**: 执行配置的发现命令并注册发现的工具

**执行流程**:

1. **命令解析**:
   ```typescript
   const discoveryCmd = this.config.getToolDiscoveryCommand();
   if (!discoveryCmd) return;
   
   const cmdParts = parse(discoveryCmd);
   if (cmdParts.length === 0) {
     throw new Error('Tool discovery command is empty');
   }
   ```

2. **进程执行**:
   ```typescript
   const proc = spawn(cmdParts[0] as string, cmdParts.slice(1) as string[]);
   let stdout = '';
   let stderr = '';
   const MAX_STDOUT_SIZE = 10 * 1024 * 1024; // 10MB限制
   ```

3. **输出处理**:
   ```typescript
   proc.stdout.on('data', (data) => {
     if (stdoutByteLength + data.length > MAX_STDOUT_SIZE) {
       sizeLimitExceeded = true;
       proc.kill();
       return;
     }
     stdout += stdoutDecoder.write(data);
   });
   ```

4. **结果解析**:
   ```typescript
   const discoveredItems = JSON.parse(stdout.trim());
   if (!Array.isArray(discoveredItems)) {
     throw new Error('Tool discovery must return JSON array');
   }
   ```

5. **工具注册**:
   ```typescript
   const functions: FunctionDeclaration[] = [];
   for (const tool of discoveredItems) {
     if (tool && typeof tool === 'object') {
       if (Array.isArray(tool['function_declarations'])) {
         functions.push(...tool['function_declarations']);
       } else if (Array.isArray(tool['functionDeclarations'])) {
         functions.push(...tool['functionDeclarations']);
       } else if (tool['name']) {
         functions.push(tool as FunctionDeclaration);
       }
     }
   }
   
   for (const func of functions) {
     if (!func.name) {
       console.warn('Discovered a tool with no name. Skipping.');
       continue;
     }
     
     const parameters = func.parameters &&
       typeof func.parameters === 'object' &&
       !Array.isArray(func.parameters)
         ? (func.parameters as Schema)
         : {};
     sanitizeParameters(parameters);
     this.registerTool(new DiscoveredTool(
       this.config,
       func.name,
       func.description ?? '',
       parameters as Record<string, unknown>,
     ));
   }
   ```

### 工具访问

#### `getTool(name: string): Tool | undefined`

**功能**: 获取指定名称的工具

```typescript
getTool(name: string): Tool | undefined {
  return this.tools.get(name);
}
```

#### `getAllTools(): Tool[]`

**功能**: 获取所有工具，按显示名称排序

```typescript
getAllTools(): Tool[] {
  return Array.from(this.tools.values()).sort((a, b) =>
    a.displayName.localeCompare(b.displayName),
  );
}
```

#### `getToolsByServer(serverName: string): Tool[]`

**功能**: 获取特定 MCP 服务器的工具

```typescript
getToolsByServer(serverName: string): Tool[] {
  const serverTools: Tool[] = [];
  for (const tool of this.tools.values()) {
    if ((tool as DiscoveredMCPTool)?.serverName === serverName) {
      serverTools.push(tool);
    }
  }
  return serverTools.sort((a, b) => a.name.localeCompare(b.name));
}
```

#### `getFunctionDeclarations(): FunctionDeclaration[]`

**功能**: 获取所有工具的函数声明，用于 Gemini API

```typescript
getFunctionDeclarations(): FunctionDeclaration[] {
  const declarations: FunctionDeclaration[] = [];
  this.tools.forEach((tool) => {
    declarations.push(tool.schema);
  });
  return declarations;
}
```

## DiscoveredTool 实现

### 构造函数增强

```typescript
constructor(
  private readonly config: Config,
  name: string,
  readonly description: string,
  readonly parameterSchema: Record<string, unknown>,
) {
  const discoveryCmd = config.getToolDiscoveryCommand()!;
  const callCommand = config.getToolCallCommand()!;
  
  // 增强描述信息
  description += `

This tool was discovered from the project by executing the command \`${discoveryCmd}\` on project root.
When called, this tool will execute the command \`${callCommand} ${name}\` on project root.
Tool discovery and call commands can be configured in project or user settings.

When called, the tool call command is executed as a subprocess.
On success, tool output is returned as a json string.
Otherwise, the following information is returned:

Stdout: Output on stdout stream. Can be \`(empty)\` or partial.
Stderr: Output on stderr stream. Can be \`(empty)\` or partial.
Error: Error or \`(none)\` if no error was reported for the subprocess.
Exit Code: Exit code or \`(none)\` if terminated by signal.
Signal: Signal number or \`(none)\` if no signal was received.`;
  
  super(
    name,
    name,
    description,
    Icon.Hammer,
    parameterSchema,
    false, // isOutputMarkdown
    false, // canUpdateOutput
  );
}
```

### 工具执行

#### `async execute(params: ToolParams): Promise<ToolResult>`

**功能**: 执行发现的工具

```typescript
async execute(params: ToolParams): Promise<ToolResult> {
  const callCommand = this.config.getToolCallCommand()!;
  const child = spawn(callCommand, [this.name]);
  child.stdin.write(JSON.stringify(params));
  child.stdin.end();

  let stdout = '';
  let stderr = '';
  let error: Error | null = null;
  let code: number | null = null;
  let signal: NodeJS.Signals | null = null;

  await new Promise<void>((resolve) => {
    const onStdout = (data: Buffer) => {
      stdout += data?.toString();
    };

    const onStderr = (data: Buffer) => {
      stderr += data?.toString();
    };

    const onError = (err: Error) => {
      error = err;
    };

    const onClose = (
      _code: number | null,
      _signal: NodeJS.Signals | null,
    ) => {
      code = _code;
      signal = _signal;
      cleanup();
      resolve();
    };

    const cleanup = () => {
      child.stdout.removeListener('data', onStdout);
      child.stderr.removeListener('data', onStderr);
      child.removeListener('error', onError);
      child.removeListener('close', onClose);
      if (child.connected) {
        child.disconnect();
      }
    };

    child.stdout.on('data', onStdout);
    child.stderr.on('data', onStderr);
    child.on('error', onError);
    child.on('close', onClose);
  });

  // 如果有任何错误、非零退出码、信号或stderr，返回错误详情
  if (error || code !== 0 || signal || stderr) {
    const llmContent = [
      `Stdout: ${stdout || '(empty)'}`,
      `Stderr: ${stderr || '(empty)'}`,
      `Error: ${error ?? '(none)'}`,
      `Exit Code: ${code ?? '(none)'}`,
      `Signal: ${signal ?? '(none)'}`,
    ].join('\n');
    return {
      llmContent,
      returnDisplay: llmContent,
    };
  }

  return {
    llmContent: stdout,
    returnDisplay: stdout,
  };
}
```

**关键特性**:

- **事件监听器清理**: 防止内存泄漏的完整清理机制
- **连接管理**: 正确处理子进程连接状态
- **错误聚合**: 统一的错误信息格式
- **JSON 参数传递**: 通过 stdin 传递结构化参数

## DiscoveredMCPTool 实现

### MCP 工具执行

#### `async execute(params: ToolParams): Promise<ToolResult>`

**功能**: 执行 MCP 服务器工具

```typescript
async execute(params: ToolParams): Promise<ToolResult> {
  const functionCalls: FunctionCall[] = [
    {
      name: this.serverToolName,  // 使用原始工具名
      args: params,
    },
  ];

  const responseParts: Part[] = await this.mcpTool.callTool(functionCalls);

  return {
    llmContent: responseParts,
    returnDisplay: getStringifiedResultForDisplay(responseParts),
  };
}
```

### MCP 响应处理

#### `getStringifiedResultForDisplay(result: Part[]): string`

**功能**: 处理 MCP 工具执行结果，生成用户友好的显示格式

```typescript
function getStringifiedResultForDisplay(result: Part[]) {
  if (!result || result.length === 0) {
    return '```json\n[]\n```';
  }

  const processFunctionResponse = (part: Part) => {
    if (part.functionResponse) {
      const responseContent = part.functionResponse.response?.content;
      if (responseContent && Array.isArray(responseContent)) {
        // 检查是否全部为简单文本部分
        const allTextParts = responseContent.every(
          (p: Part) => p.text !== undefined,
        );
        if (allTextParts) {
          return responseContent.map((p: Part) => p.text).join('');
        }
        // 如果包含其他类型，返回结构化内容
        return responseContent;
      }
      // 返回完整的 functionResponse 对象
      return part.functionResponse;
    }
    return part; // 其他类型的 Part
  };

  const processedResults =
    result.length === 1
      ? processFunctionResponse(result[0])
      : result.map(processFunctionResponse);

  if (typeof processedResults === 'string') {
    return processedResults;
  }

  return '```json\n' + JSON.stringify(processedResults, null, 2) + '\n```';
}
```

**处理逻辑**:

1. **空结果处理**: 返回空 JSON 数组
2. **文本响应优化**: 纯文本响应直接返回字符串
3. **结构化数据**: 复杂数据以 JSON 格式显示
4. **Markdown 格式**: 使用代码块包装 JSON 输出

## Schema 清理功能

### `sanitizeParameters(schema?: Schema)`

**功能**: 清理 Schema 以兼容 Gemini API

```typescript
export function sanitizeParameters(schema?: Schema) {
  _sanitizeParameters(schema, new Set<Schema>());
}

function _sanitizeParameters(schema: Schema | undefined, visited: Set<Schema>) {
  if (!schema || visited.has(schema)) return;
  visited.add(schema);
  
  // 处理anyOf冲突
  if (schema.anyOf) {
    schema.default = undefined;  // Vertex AI对anyOf+default组合有问题
    for (const item of schema.anyOf) {
      if (typeof item !== 'boolean') {
        _sanitizeParameters(item, visited);
      }
    }
  }
  
  // 递归处理嵌套schema
  if (schema.items && typeof schema.items !== 'boolean') {
    _sanitizeParameters(schema.items, visited);
  }
  
  if (schema.properties) {
    for (const item of Object.values(schema.properties)) {
      if (typeof item !== 'boolean') {
        _sanitizeParameters(item, visited);
      }
    }
  }
  
  // 处理enum值
  if (schema.enum && Array.isArray(schema.enum)) {
    if (schema.type !== Type.STRING) {
      schema.type = Type.STRING;  // Gemini API仅支持STRING类型的enum
    }
    // 过滤null/undefined并转换为字符串
    schema.enum = schema.enum
      .filter((value: unknown) => value !== null && value !== undefined)
      .map((value: unknown) => String(value));
  }
}
```

**清理操作**:

1. **anyOf 处理**: 移除与 anyOf 冲突的 default 属性
2. **递归清理**: 处理嵌套的 schema 结构
3. **枚举规范化**: 将枚举值转换为字符串类型
4. **循环引用防护**: 使用 visited 集合防止无限递归

## 使用示例

### 基本工具注册

```typescript
const toolRegistry = new ToolRegistry(config);

// 注册内置工具
toolRegistry.registerTool(new ReadFileTool(config));
toolRegistry.registerTool(new EditTool(config));
toolRegistry.registerTool(new ShellTool(config));

// 发现项目特定工具
await toolRegistry.discoverAllTools();
```

### 配置工具发现

```typescript
// 在config中设置发现命令
const config = new Config({
  // ... 其他配置
  toolDiscoveryCommand: 'npm run discover-tools',
  toolCallCommand: 'npm run call-tool',
});

const toolRegistry = new ToolRegistry(config);
await toolRegistry.discoverAndRegisterToolsFromCommand();
```

### MCP 服务器工具

```typescript
// 配置MCP服务器
const config = new Config({
  mcpServers: {
    'filesystem': {
      command: 'node',
      args: ['filesystem-mcp-server.js'],
    },
    'web-search': {
      url: 'http://localhost:3001/mcp',
    },
  },
});

const toolRegistry = new ToolRegistry(config);
await toolRegistry.discoverMcpTools();

// 获取特定服务器的工具
const filesystemTools = toolRegistry.getToolsByServer('filesystem');
```

### 工具查询和使用

```typescript
// 获取所有工具
const allTools = toolRegistry.getAllTools();
console.log(`Available tools: ${allTools.map(t => t.name).join(', ')}`);

// 获取特定工具
const readTool = toolRegistry.getTool('read_file');
if (readTool) {
  const result = await readTool.execute({ file_path: '/path/to/file.txt' });
  console.log(result.llmContent);
}

// 获取函数声明（用于Gemini API）
const declarations = toolRegistry.getFunctionDeclarations();
const response = await geminiClient.generateContent(
  messages,
  { tools: [{ functionDeclarations: declarations }] }
);
```

### 动态工具管理

```typescript
// 重新发现特定服务器的工具
await toolRegistry.discoverToolsForServer('filesystem');

// 检查工具是否存在
if (toolRegistry.getTool('custom_tool')) {
  console.log('Custom tool is available');
}

// 获取工具统计
const allTools = toolRegistry.getAllTools();
const builtinTools = allTools.filter(t => !(t instanceof DiscoveredTool) && !(t instanceof DiscoveredMCPTool));
const discoveredTools = allTools.filter(t => t instanceof DiscoveredTool);
const mcpTools = allTools.filter(t => t instanceof DiscoveredMCPTool);

console.log(`Builtin: ${builtinTools.length}, Discovered: ${discoveredTools.length}, MCP: ${mcpTools.length}`);
```

## 错误处理

### 工具发现错误

```typescript
try {
  await toolRegistry.discoverAllTools();
} catch (error) {
  console.error('Tool discovery failed:', error);
  // 继续使用内置工具
}
```

### 工具执行错误

```typescript
const tool = toolRegistry.getTool('some_tool');
if (tool) {
  try {
    const result = await tool.execute(params);
    if (result.llmContent.includes('Error:')) {
      console.error('Tool execution failed:', result.returnDisplay);
    }
  } catch (error) {
    console.error('Unexpected tool error:', error);
  }
}
```

### Schema 验证错误

```typescript
// 发现的工具可能有无效的schema
try {
  const declarations = toolRegistry.getFunctionDeclarations();
  // 验证schema是否有效
  for (const decl of declarations) {
    if (!decl.name || typeof decl.name !== 'string') {
      console.warn('Invalid tool declaration:', decl);
    }
  }
} catch (error) {
  console.error('Schema validation failed:', error);
}
```

## 性能考虑

### 工具发现优化

```typescript
// 缓存发现结果，避免重复发现
class CachedToolRegistry extends ToolRegistry {
  private discoveryCache = new Map<string, { tools: Tool[], timestamp: number }>();
  
  async discoverAllTools(): Promise<void> {
    const cacheKey = this.generateDiscoveryCacheKey();
    const cached = this.discoveryCache.get(cacheKey);
    
    if (cached && !this.isCacheExpired(cached.timestamp)) {
      // 使用缓存的工具
      for (const tool of cached.tools) {
        this.registerTool(tool);
      }
      return;
    }
    
    await super.discoverAllTools();
    
    // 缓存结果
    const discoveredTools = this.getAllTools().filter(t => 
      t instanceof DiscoveredTool || t instanceof DiscoveredMCPTool
    );
    this.discoveryCache.set(cacheKey, {
      tools: discoveredTools,
      timestamp: Date.now()
    });
  }
}
```

### 内存管理

```typescript
// 定期清理不活跃的工具
class ManagedToolRegistry extends ToolRegistry {
  private toolUsage = new Map<string, number>();
  
  registerTool(tool: Tool): void {
    super.registerTool(tool);
    this.toolUsage.set(tool.name, Date.now());
  }
  
  async cleanupInactiveTools(): Promise<void> {
    const now = Date.now();
    const inactiveThreshold = 24 * 60 * 60 * 1000; // 24小时
    
    for (const [name, lastUsed] of this.toolUsage.entries()) {
      if (now - lastUsed > inactiveThreshold) {
        const tool = this.getTool(name);
        if (tool && (tool instanceof DiscoveredTool || tool instanceof DiscoveredMCPTool)) {
          this.tools.delete(name);
          this.toolUsage.delete(name);
        }
      }
    }
  }
}
```

## 扩展性

### 自定义工具发现

```typescript
interface ToolDiscoverer {
  discoverTools(): Promise<Tool[]>;
}

class CustomToolDiscoverer implements ToolDiscoverer {
  async discoverTools(): Promise<Tool[]> {
    // 自定义发现逻辑
    return [];
  }
}

class ExtendedToolRegistry extends ToolRegistry {
  private discoverers: ToolDiscoverer[] = [];
  
  addDiscoverer(discoverer: ToolDiscoverer): void {
    this.discoverers.push(discoverer);
  }
  
  async discoverAllTools(): Promise<void> {
    await super.discoverAllTools();
    
    // 运行自定义发现器
    for (const discoverer of this.discoverers) {
      const tools = await discoverer.discoverTools();
      for (const tool of tools) {
        this.registerTool(tool);
      }
    }
  }
}
```

### 工具生命周期管理

```typescript
interface ToolLifecycleListener {
  onToolRegistered(tool: Tool): void;
  onToolUnregistered(tool: Tool): void;
}

class LifecycleAwareToolRegistry extends ToolRegistry {
  private listeners: ToolLifecycleListener[] = [];
  
  addLifecycleListener(listener: ToolLifecycleListener): void {
    this.listeners.push(listener);
  }
  
  registerTool(tool: Tool): void {
    super.registerTool(tool);
    this.listeners.forEach(l => l.onToolRegistered(tool));
  }
  
  unregisterTool(name: string): boolean {
    const tool = this.getTool(name);
    if (tool) {
      this.tools.delete(name);
      this.listeners.forEach(l => l.onToolUnregistered(tool));
      return true;
    }
    return false;
  }
}
```

## 安全考虑

### 工具隔离

- 发现的工具在独立进程中执行
- 限制输出大小防止资源耗尽
- 超时机制防止长时间阻塞

### 权限控制

```typescript
class SecureToolRegistry extends ToolRegistry {
  private allowedTools = new Set<string>();
  
  setAllowedTools(tools: string[]): void {
    this.allowedTools = new Set(tools);
  }
  
  registerTool(tool: Tool): void {
    if (this.allowedTools.size > 0 && !this.allowedTools.has(tool.name)) {
      console.warn(`Tool ${tool.name} not in allowed list, skipping`);
      return;
    }
    super.registerTool(tool);
  }
}
```

### Schema 安全

- 清理潜在危险的 Schema 属性
- 验证工具参数类型
- 防止 Schema 注入攻击