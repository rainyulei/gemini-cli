# MCP 工具包装器 (mcp-tool.ts)

MCP 工具包装器是 Gemini CLI 中负责封装和管理从 MCP 服务器发现的工具的核心模块。它提供了统一的工具接口、确认机制和执行逻辑。

## 📋 概述

`mcp-tool.ts` 模块的主要职责：
- **工具封装**: 将 MCP 服务器的原生工具封装为 Gemini CLI 可用的工具
- **确认管理**: 实现工具执行前的用户确认机制
- **名称管理**: 处理工具名称冲突和有效性验证
- **结果格式化**: 格式化工具执行结果以适配不同的显示需求
- **权限控制**: 基于信任级别和用户选择的权限管理

## 🏗️ 核心类定义

### `DiscoveredMCPTool` 类

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
  )
}
```

**构造参数详解**:
- `mcpTool`: 底层的 MCP 可调用工具实例
- `serverName`: 提供此工具的 MCP 服务器名称
- `serverToolName`: 工具在 MCP 服务器中的原始名称
- `description`: 工具功能描述
- `parameterSchemaJson`: 工具参数的 JSON Schema
- `timeout`: 工具执行超时时间（可选）
- `trust`: 是否信任此工具（跳过确认）
- `nameOverride`: 自定义工具名称（可选）

## 🔧 核心功能

### 1. 工具名称管理

#### 有效名称生成

**`generateValidName()`** - 生成符合 Gemini API 要求的有效工具名称

```typescript
export function generateValidName(name: string): string {
  // 替换无效字符为下划线
  let validToolname = name.replace(/[^a-zA-Z0-9_.-]/g, '_');

  // 如果长度超过63字符，中间替换为'___'
  if (validToolname.length > 63) {
    validToolname =
      validToolname.slice(0, 28) + '___' + validToolname.slice(-32);
  }
  return validToolname;
}
```

**名称清理规则**:
1. **字符过滤**: 只保留字母、数字、下划线、点和连字符
2. **长度限制**: 最大63字符（Gemini API 限制）
3. **中间截断**: 超长名称中间用 `___` 替换

#### 完全限定名称

**`asFullyQualifiedTool()`** - 创建带服务器前缀的工具副本

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
    `${this.serverName}__${this.serverToolName}`, // 前缀格式
  );
}
```

**用途**: 解决多个服务器提供同名工具时的命名冲突问题

### 2. Schema 管理

#### 动态 Schema 生成

```typescript
override get schema(): FunctionDeclaration {
  return {
    name: this.name,
    description: this.description,
    parametersJsonSchema: this.parameterSchemaJson,
  };
}
```

**特点**:
- **动态生成**: 基于 MCP 服务器提供的原始 schema
- **标准化**: 符合 Gemini API 的 FunctionDeclaration 格式
- **参数保留**: 保持原始参数结构和验证规则

### 3. 执行确认机制

#### 确认逻辑

**`shouldConfirmExecute()`** - 决定是否需要用户确认

```typescript
async shouldConfirmExecute(
  _params: ToolParams,
  _abortSignal: AbortSignal,
): Promise<ToolCallConfirmationDetails | false>
```

**确认决策流程**:
1. **信任检查**: 如果 `trust === true`，跳过确认
2. **允许列表检查**: 检查服务器级或工具级允许列表
3. **生成确认详情**: 创建确认对话框配置

**允许列表键值**:
- **服务器级**: `serverName` - 信任整个服务器
- **工具级**: `serverName.toolName` - 信任特定工具

#### 确认详情配置

```typescript
const confirmationDetails: ToolMcpConfirmationDetails = {
  type: 'mcp',
  title: 'Confirm MCP Tool Execution',
  serverName: this.serverName,
  toolName: this.serverToolName, // 显示原始工具名
  toolDisplayName: this.name,    // 显示注册表中的名称
  onConfirm: async (outcome: ToolConfirmationOutcome) => {
    // 根据用户选择更新允许列表
    if (outcome === ToolConfirmationOutcome.ProceedAlwaysServer) {
      DiscoveredMCPTool.allowlist.add(serverAllowListKey);
    } else if (outcome === ToolConfirmationOutcome.ProceedAlwaysTool) {
      DiscoveredMCPTool.allowlist.add(toolAllowListKey);
    }
  },
};
```

**确认选项**:
- **一次性执行**: 仅本次执行，不更新允许列表
- **始终允许工具**: 将特定工具添加到允许列表
- **始终允许服务器**: 将整个服务器添加到允许列表
- **取消**: 中止工具执行

### 4. 工具执行

#### 执行流程

**`execute()`** - 执行 MCP 工具并返回格式化结果

```typescript
async execute(params: ToolParams): Promise<ToolResult> {
  const functionCalls: FunctionCall[] = [
    {
      name: this.serverToolName, // 使用原始服务器工具名
      args: params,
    },
  ];

  const responseParts: Part[] = await this.mcpTool.callTool(functionCalls);

  return {
    llmContent: responseParts,     // 原始响应供 LLM 使用
    returnDisplay: getStringifiedResultForDisplay(responseParts), // 格式化显示
  };
}
```

**执行步骤**:
1. **构造调用**: 创建 FunctionCall 对象
2. **调用工具**: 通过底层 `mcpTool` 执行
3. **处理响应**: 获取响应部分
4. **格式化结果**: 分别处理 LLM 内容和用户显示

## 🎨 结果格式化

### 显示格式化函数

**`getStringifiedResultForDisplay()`** - 将工具响应格式化为用户友好的显示格式

```typescript
function getStringifiedResultForDisplay(result: Part[]): string {
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
        // 否则返回内容数组用于 JSON 序列化
        return responseContent;
      }

      // 返回整个 functionResponse 用于检查
      return part.functionResponse;
    }
    return part; // 其他类型的部分
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

**格式化逻辑**:

1. **空结果处理**: 返回空 JSON 数组
2. **函数响应处理**:
   - **纯文本内容**: 直接连接文本部分
   - **混合内容**: 保留数组结构进行 JSON 序列化
   - **其他类型**: 保留原始结构
3. **单一结果**: 直接处理
4. **多个结果**: 映射处理每个部分
5. **最终格式化**: 
   - 字符串结果直接返回
   - 对象结果用 JSON markdown 代码块包装

## 💡 使用示例

### 基本工具创建和使用

```typescript
import { DiscoveredMCPTool } from './mcp-tool.js';
import { CallableTool } from '@google/genai';

// 创建 MCP 工具实例
const mcpTool = mcpToTool(mcpClient); // 从 MCP 客户端创建

const discoveredTool = new DiscoveredMCPTool(
  mcpTool,
  'filesystem-server',      // 服务器名称
  'read_file',             // 原始工具名称
  'Read file contents',    // 描述
  {                        // 参数 schema
    type: 'object',
    properties: {
      path: {
        type: 'string',
        description: 'File path to read'
      }
    },
    required: ['path']
  },
  30000,  // 30秒超时
  false   // 不信任，需要确认
);

// 执行工具
const result = await discoveredTool.execute({
  path: '/path/to/file.txt'
});

console.log('LLM Content:', result.llmContent);
console.log('Display:', result.returnDisplay);
```

### 工具确认流程

```typescript
// 检查是否需要确认
const confirmationDetails = await discoveredTool.shouldConfirmExecute(
  { path: '/sensitive/file.txt' },
  new AbortController().signal
);

if (confirmationDetails) {
  console.log('Confirmation required:');
  console.log(`Server: ${confirmationDetails.serverName}`);
  console.log(`Tool: ${confirmationDetails.toolName}`);
  console.log(`Display Name: ${confirmationDetails.toolDisplayName}`);

  // 模拟用户确认
  await confirmationDetails.onConfirm(
    ToolConfirmationOutcome.ProceedAlwaysTool
  );
  
  console.log('Tool added to allowlist');
}

// 现在执行工具（无需确认）
const result = await discoveredTool.execute({
  path: '/sensitive/file.txt'
});
```

### 名称冲突处理

```typescript
// 两个服务器都提供 'search' 工具
const searchTool1 = new DiscoveredMCPTool(
  mcpTool1,
  'web-server',
  'search',
  'Web search',
  searchSchema,
  15000,
  false
);

const searchTool2 = new DiscoveredMCPTool(
  mcpTool2,
  'docs-server', 
  'search',
  'Document search',
  searchSchema,
  10000,
  false
);

// 创建完全限定版本以避免冲突
const qualifiedTool1 = searchTool1.asFullyQualifiedTool();
const qualifiedTool2 = searchTool2.asFullyQualifiedTool();

console.log(qualifiedTool1.name); // "web-server__search"
console.log(qualifiedTool2.name); // "docs-server__search"

// 在工具注册表中注册
toolRegistry.registerTool(searchTool1);     // 第一个注册获得原名 "search"
toolRegistry.registerTool(qualifiedTool2);  // 第二个使用限定名 "docs-server__search"
```

### 信任级别管理

```typescript
// 创建不同信任级别的工具
const trustedTool = new DiscoveredMCPTool(
  mcpTool,
  'internal-server',
  'safe_operation',
  'Safe internal operation',
  schema,
  5000,
  true  // 信任此工具，跳过确认
);

const untrustedTool = new DiscoveredMCPTool(
  mcpTool,
  'external-server',
  'dangerous_operation', 
  'Potentially dangerous operation',
  schema,
  10000,
  false // 不信任，需要确认
);

// 信任工具直接执行
const trustedResult = await trustedTool.execute({ param: 'value' });

// 不信任工具需要确认
const confirmation = await untrustedTool.shouldConfirmExecute({ param: 'value' }, signal);
if (confirmation) {
  // 显示确认对话框...
  await confirmation.onConfirm(ToolConfirmationOutcome.ProceedOnce);
}
const untrustedResult = await untrustedTool.execute({ param: 'value' });
```

### 高级结果处理

```typescript
// 处理不同类型的工具响应
class AdvancedMCPToolHandler {
  async handleToolResult(tool: DiscoveredMCPTool, params: ToolParams): Promise<void> {
    try {
      const result = await tool.execute(params);
      
      // 分析 LLM 内容
      for (const part of result.llmContent) {
        if (part.functionResponse) {
          const response = part.functionResponse.response;
          
          if (response?.content) {
            console.log(`Tool ${tool.name} returned:`, response.content);
            
            // 检查错误
            if (response.isError) {
              console.error(`Tool error: ${response.content}`);
              return;
            }
            
            // 处理不同内容类型
            for (const contentPart of response.content) {
              if (contentPart.text) {
                console.log('Text response:', contentPart.text);
              } else if (contentPart.image) {
                console.log('Image response received');
              } else if (contentPart.fileData) {
                console.log('File data response received');
              }
            }
          }
        }
      }
      
      // 显示用户友好的结果
      console.log('Formatted result:');
      console.log(result.returnDisplay);
      
    } catch (error) {
      console.error(`Tool execution failed: ${error.message}`);
      
      // 处理超时错误
      if (error.message.includes('timeout')) {
        console.log(`Tool ${tool.name} timed out after ${tool.timeout}ms`);
      }
      
      // 处理权限错误
      if (error.message.includes('permission') || error.message.includes('unauthorized')) {
        console.log('Tool execution was denied or requires additional permissions');
      }
    }
  }
}

// 使用高级处理器
const handler = new AdvancedMCPToolHandler();
await handler.handleToolResult(discoveredTool, { path: '/example/file.txt' });
```

### 批量工具管理

```typescript
// 批量管理多个 MCP 工具
class MCPToolManager {
  private tools = new Map<string, DiscoveredMCPTool>();
  private allowedServers = new Set<string>();
  
  registerTool(tool: DiscoveredMCPTool): void {
    // 检查名称冲突
    if (this.tools.has(tool.name)) {
      console.warn(`Tool name conflict: ${tool.name}`);
      // 使用完全限定名称
      const qualifiedTool = tool.asFullyQualifiedTool();
      this.tools.set(qualifiedTool.name, qualifiedTool);
    } else {
      this.tools.set(tool.name, tool);
    }
  }
  
  async executeToolSafely(toolName: string, params: ToolParams): Promise<ToolResult | null> {
    const tool = this.tools.get(toolName);
    if (!tool) {
      console.error(`Tool not found: ${toolName}`);
      return null;
    }
    
    // 检查服务器是否在允许列表中
    if (!this.allowedServers.has(tool.serverName) && !tool.trust) {
      const confirmation = await tool.shouldConfirmExecute(params, new AbortController().signal);
      
      if (confirmation) {
        console.log(`Tool ${toolName} from ${tool.serverName} requires confirmation`);
        // 在实际应用中，这里会显示确认对话框
        // 这里假设用户总是允许
        await confirmation.onConfirm(ToolConfirmationOutcome.ProceedAlwaysServer);
        this.allowedServers.add(tool.serverName);
      }
    }
    
    try {
      return await tool.execute(params);
    } catch (error) {
      console.error(`Tool ${toolName} execution failed:`, error);
      return null;
    }
  }
  
  listTools(): Array<{name: string, server: string, description: string, trusted: boolean}> {
    return Array.from(this.tools.values()).map(tool => ({
      name: tool.name,
      server: tool.serverName,
      description: tool.description,
      trusted: tool.trust || this.allowedServers.has(tool.serverName),
    }));
  }
  
  getToolsByServer(serverName: string): DiscoveredMCPTool[] {
    return Array.from(this.tools.values())
      .filter(tool => tool.serverName === serverName);
  }
}

// 使用工具管理器
const toolManager = new MCPToolManager();

// 注册工具
const tools = [
  new DiscoveredMCPTool(mcpTool1, 'fs-server', 'read', 'Read file', schema1, 5000, false),
  new DiscoveredMCPTool(mcpTool2, 'web-server', 'search', 'Web search', schema2, 10000, false),
  new DiscoveredMCPTool(mcpTool3, 'fs-server', 'write', 'Write file', schema3, 5000, false),
];

for (const tool of tools) {
  toolManager.registerTool(tool);
}

// 列出所有工具
console.log('Available tools:');
for (const toolInfo of toolManager.listTools()) {
  console.log(`- ${toolInfo.name} (${toolInfo.server}): ${toolInfo.description} [${toolInfo.trusted ? 'Trusted' : 'Requires confirmation'}]`);
}

// 执行工具
const readResult = await toolManager.executeToolSafely('read', { path: '/etc/hosts' });
if (readResult) {
  console.log('File contents:', readResult.returnDisplay);
}
```

## 🔒 安全考虑

### 1. 工具执行安全

```typescript
// 安全工具执行包装器
class SecureMCPToolExecutor {
  private executionTimeout = 30000; // 30秒默认超时
  private maxConcurrentExecutions = 5;
  private currentExecutions = 0;
  
  async executeWithSafeguards(
    tool: DiscoveredMCPTool,
    params: ToolParams
  ): Promise<ToolResult> {
    // 检查并发限制
    if (this.currentExecutions >= this.maxConcurrentExecutions) {
      throw new Error('Too many concurrent tool executions');
    }
    
    this.currentExecutions++;
    
    try {
      // 创建超时控制器
      const abortController = new AbortController();
      const timeoutId = setTimeout(() => {
        abortController.abort();
      }, tool.timeout || this.executionTimeout);
      
      // 参数验证
      this.validateParameters(params, tool.parameterSchemaJson);
      
      // 执行工具
      const result = await tool.execute(params);
      
      clearTimeout(timeoutId);
      
      // 结果验证
      this.validateResult(result);
      
      return result;
    } finally {
      this.currentExecutions--;
    }
  }
  
  private validateParameters(params: ToolParams, schema: unknown): void {
    // 实现参数验证逻辑
    if (!params || typeof params !== 'object') {
      throw new Error('Invalid parameters: must be an object');
    }
    
    // 这里可以添加更详细的 JSON Schema 验证
  }
  
  private validateResult(result: ToolResult): void {
    if (!result || !result.llmContent) {
      throw new Error('Invalid tool result: missing llmContent');
    }
  }
}
```

### 2. 权限管理

```typescript
// 权限管理系统
class MCPPermissionManager {
  private permissions = new Map<string, Set<string>>(); // server -> allowed tools
  private globallyDeniedTools = new Set<string>();
  
  setServerPermissions(serverName: string, allowedTools: string[]): void {
    this.permissions.set(serverName, new Set(allowedTools));
  }
  
  denyToolGlobally(toolName: string): void {
    this.globallyDeniedTools.add(toolName);
  }
  
  isToolAllowed(tool: DiscoveredMCPTool): boolean {
    // 检查全局拒绝列表
    if (this.globallyDeniedTools.has(tool.serverToolName)) {
      return false;
    }
    
    // 检查服务器权限
    const serverPermissions = this.permissions.get(tool.serverName);
    if (serverPermissions) {
      return serverPermissions.has(tool.serverToolName);
    }
    
    // 默认允许信任的工具
    return tool.trust || false;
  }
  
  createSecureTool(tool: DiscoveredMCPTool): DiscoveredMCPTool | null {
    if (!this.isToolAllowed(tool)) {
      console.warn(`Tool ${tool.name} from ${tool.serverName} is not allowed`);
      return null;
    }
    
    return tool;
  }
}

// 使用权限管理器
const permissionManager = new MCPPermissionManager();

// 配置权限
permissionManager.setServerPermissions('fs-server', ['read', 'list']);
permissionManager.setServerPermissions('web-server', ['search', 'fetch']);
permissionManager.denyToolGlobally('delete_all'); // 全局禁止危险操作

// 创建安全工具
const secureTool = permissionManager.createSecureTool(discoveredTool);
if (secureTool) {
  const result = await secureTool.execute(params);
}
```

## 🧪 测试支持

### 工具测试助手

```typescript
// MCP 工具测试助手
export class MCPToolTestHelper {
  static createMockCallableTool(
    responses: Record<string, any> = {}
  ): CallableTool {
    return {
      callTool: jest.fn().mockImplementation(async (calls: FunctionCall[]) => {
        const call = calls[0];
        const response = responses[call.name] || { success: true };
        
        return [{
          functionResponse: {
            name: call.name,
            response: {
              content: [{ text: JSON.stringify(response) }]
            }
          }
        }];
      })
    } as any;
  }
  
  static createTestTool(
    serverName = 'test-server',
    toolName = 'test-tool',
    overrides: Partial<ConstructorParameters<typeof DiscoveredMCPTool>> = {}
  ): DiscoveredMCPTool {
    const mockCallableTool = this.createMockCallableTool();
    
    return new DiscoveredMCPTool(
      mockCallableTool,
      serverName,
      toolName,
      'Test tool description',
      { type: 'object', properties: {} },
      5000,
      false,
      ...overrides
    );
  }
  
  static async testToolExecution(
    tool: DiscoveredMCPTool,
    params: ToolParams,
    expectedResponse?: any
  ): Promise<void> {
    const result = await tool.execute(params);
    
    expect(result).toBeDefined();
    expect(result.llmContent).toBeDefined();
    expect(result.returnDisplay).toBeDefined();
    
    if (expectedResponse) {
      const displayContent = JSON.parse(
        result.returnDisplay.replace(/```json\n|\n```/g, '')
      );
      expect(displayContent).toEqual(expectedResponse);
    }
  }
}

// 测试示例
describe('DiscoveredMCPTool', () => {
  it('should execute tool and return formatted result', async () => {
    const tool = MCPToolTestHelper.createTestTool('test-server', 'test-tool');
    
    await MCPToolTestHelper.testToolExecution(
      tool,
      { input: 'test' },
      { success: true }
    );
  });
  
  it('should handle confirmation for untrusted tools', async () => {
    const tool = MCPToolTestHelper.createTestTool('untrusted-server', 'dangerous-tool');
    
    const confirmationDetails = await tool.shouldConfirmExecute(
      { action: 'delete' },
      new AbortController().signal
    );
    
    expect(confirmationDetails).toBeTruthy();
    expect(confirmationDetails.type).toBe('mcp');
    expect(confirmationDetails.serverName).toBe('untrusted-server');
    expect(confirmationDetails.toolName).toBe('dangerous-tool');
  });
  
  it('should skip confirmation for trusted tools', async () => {
    const trustedTool = new DiscoveredMCPTool(
      MCPToolTestHelper.createMockCallableTool(),
      'trusted-server',
      'safe-tool',
      'Safe operation',
      { type: 'object' },
      5000,
      true // trusted
    );
    
    const confirmationDetails = await trustedTool.shouldConfirmExecute(
      { action: 'read' },
      new AbortController().signal
    );
    
    expect(confirmationDetails).toBe(false);
  });
});
```

这个 MCP 工具包装器是 Gemini CLI 中 MCP 集成的关键组件，它提供了统一的工具接口、灵活的确认机制和强大的结果格式化功能，确保 MCP 服务器的工具能够安全、有效地集成到 Gemini CLI 中。