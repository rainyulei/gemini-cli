# MCP Client - Model Context Protocol 客户端文档

## 概述

`mcp-client.ts` 是 Gemini CLI 的 Model Context Protocol (MCP) 客户端核心实现，负责与 MCP 服务器的连接、认证、工具发现和执行。它支持多种传输协议（Stdio、SSE、HTTP）、OAuth 认证、自动服务器发现，并提供完整的工具和提示管理功能。

## 主要功能

- **多传输协议支持**: Stdio、SSE (Server-Sent Events)、Streamable HTTP
- **OAuth 认证集成**: 自动 OAuth 发现、令牌管理、认证流程
- **服务器状态管理**: 连接状态跟踪、错误恢复、状态变更通知
- **工具和提示发现**: 自动发现和注册 MCP 服务器提供的工具和提示
- **安全验证**: 命令过滤、权限管理、安全策略执行
- **Google 凭据支持**: Google Credentials 认证提供程序集成

## 核心常量和枚举

### 超时配置

```typescript
export const MCP_DEFAULT_TIMEOUT_MSEC = 10 * 60 * 1000; // 10分钟默认超时
```

### 服务器状态枚举

```typescript
export enum MCPServerStatus {
  /** 服务器断开连接或遇到错误 */
  DISCONNECTED = 'disconnected',
  /** 服务器正在连接中 */  
  CONNECTING = 'connecting',
  /** 服务器已连接并准备使用 */
  CONNECTED = 'connected',
}
```

### 发现状态枚举

```typescript
export enum MCPDiscoveryState {
  /** 发现尚未开始 */
  NOT_STARTED = 'not_started',
  /** 发现正在进行中 */
  IN_PROGRESS = 'in_progress', 
  /** 发现已完成（无论是否有错误） */
  COMPLETED = 'completed',
}
```

## 核心接口定义

### `DiscoveredMCPPrompt`

**功能**: 发现的 MCP 提示接口，扩展基础 Prompt 类型

```typescript
export type DiscoveredMCPPrompt = Prompt & {
  /** 提供此提示的服务器名称 */
  serverName: string;
  /** 调用提示的函数 */
  invoke: (params: Record<string, unknown>) => Promise<GetPromptResult>;
};
```

**字段详情**:
- `serverName`: 提供此提示的 MCP 服务器标识
- `invoke`: 异步函数，用于执行提示并返回结果

## 状态管理系统

### 服务器状态跟踪

```typescript
// 服务器状态映射
const serverStatuses: Map<string, MCPServerStatus> = new Map();

// 全局发现状态
let mcpDiscoveryState: MCPDiscoveryState = MCPDiscoveryState.NOT_STARTED;

// OAuth 需求跟踪
export const mcpServerRequiresOAuth: Map<string, boolean> = new Map();
```

### 状态变更监听器

```typescript
type StatusChangeListener = (
  serverName: string,
  status: MCPServerStatus,
) => void;

// 添加状态变更监听器
export function addMCPStatusChangeListener(
  listener: StatusChangeListener,
): void;

// 移除状态变更监听器  
export function removeMCPStatusChangeListener(
  listener: StatusChangeListener,
): void;
```

### 状态查询函数

```typescript
// 获取特定服务器状态
export function getMCPServerStatus(serverName: string): MCPServerStatus;

// 获取所有服务器状态
export function getAllMCPServerStatuses(): Map<string, MCPServerStatus>;

// 获取发现状态
export function getMCPDiscoveryState(): MCPDiscoveryState;
```

## OAuth 认证系统

### 自动 OAuth 处理

```typescript
async function handleAutomaticOAuth(
  mcpServerName: string,
  mcpServerConfig: MCPServerConfig,
  wwwAuthenticate: string,
): Promise<boolean>
```

**功能**: 处理服务器的自动 OAuth 发现和认证流程

**处理流程**:
1. **解析认证头**: 从 `www-authenticate` 头中提取资源元数据 URI
2. **配置发现**: 调用 `OAuthUtils.discoverOAuthConfig()` 发现 OAuth 配置
3. **创建认证配置**: 构建认证 URL、令牌 URL 和作用域配置
4. **执行认证**: 通过 `MCPOAuthProvider.authenticate()` 执行认证流程
5. **返回结果**: 返回认证是否成功

**实现示例**:
```typescript
// OAuth 配置发现
let oauthConfig;
const resourceMetadataUri = OAuthUtils.parseWWWAuthenticateHeader(wwwAuthenticate);
if (resourceMetadataUri) {
  oauthConfig = await OAuthUtils.discoverOAuthConfig(resourceMetadataUri);
} else if (mcpServerConfig.url) {
  // 后备方案：从基础 URL 发现
  const sseUrl = new URL(mcpServerConfig.url);
  const baseUrl = `${sseUrl.protocol}//${sseUrl.host}`;
  oauthConfig = await OAuthUtils.discoverOAuthConfig(baseUrl);
}

// 创建认证配置
const oauthAuthConfig = {
  enabled: true,
  authorizationUrl: oauthConfig.authorizationUrl,
  tokenUrl: oauthConfig.tokenUrl,
  scopes: oauthConfig.scopes || [],
};

// 执行认证
await MCPOAuthProvider.authenticate(mcpServerName, oauthAuthConfig);
```

### OAuth 传输创建

```typescript
async function createTransportWithOAuth(
  mcpServerName: string,
  mcpServerConfig: MCPServerConfig,
  accessToken: string,
): Promise<StreamableHTTPClientTransport | SSEClientTransport | null>
```

**功能**: 使用 OAuth 令牌创建认证传输

**支持的传输类型**:
- **HTTP 传输**: 使用 `StreamableHTTPClientTransport` 与 Bearer 令牌
- **SSE 传输**: 使用 `SSEClientTransport` 与授权头

## 服务器连接和发现

### 主要发现函数

```typescript
export async function discoverMcpTools(
  mcpServers: Record<string, MCPServerConfig>,
  mcpServerCommand: string | undefined,
  toolRegistry: ToolRegistry,
  promptRegistry: PromptRegistry,
  debugMode: boolean,
): Promise<void>
```

**功能**: 发现所有配置的 MCP 服务器中的工具，并注册到工具注册表

**执行流程**:
1. **状态设置**: 设置发现状态为 `IN_PROGRESS`
2. **服务器准备**: 处理命令行指定的服务器配置
3. **并行连接**: 并行连接到所有配置的服务器
4. **工具注册**: 将发现的工具注册到 `ToolRegistry`
5. **提示注册**: 将发现的提示注册到 `PromptRegistry`
6. **状态完成**: 设置发现状态为 `COMPLETED`

### 服务器连接函数

```typescript
export async function connectAndDiscover(
  mcpServerName: string,
  mcpServerConfig: MCPServerConfig,
  toolRegistry: ToolRegistry,
  promptRegistry: PromptRegistry,
  debugMode: boolean,
): Promise<void>
```

**功能**: 连接到单个 MCP 服务器并发现可用工具和提示

**错误处理机制**:
- **连接错误**: 自动重试和状态恢复
- **认证失败**: OAuth 自动发现和重新认证
- **服务器不可用**: 优雅降级和错误记录
- **工具发现失败**: 继续其他服务器的发现过程

### 工具发现

```typescript
export async function discoverTools(
  mcpServerName: string,
  mcpServerConfig: MCPServerConfig,
  mcpClient: Client,
): Promise<DiscoveredMCPTool[]>
```

**功能**: 从连接的 MCP 客户端发现和清理工具

**发现流程**:
1. **获取函数声明**: 调用 `mcpToTool(mcpClient).tool()` 获取工具
2. **验证声明**: 检查返回的函数声明有效性
3. **过滤工具**: 根据配置过滤启用/禁用的工具
4. **创建工具实例**: 为每个有效工具创建 `DiscoveredMCPTool` 实例
5. **返回结果**: 返回可用工具数组

**工具过滤逻辑**:
```typescript
export function isEnabled(
  funcDecl: FunctionDeclaration,
  mcpServerName: string,
  mcpServerConfig: MCPServerConfig,
): boolean {
  // 检查名称有效性
  if (!funcDecl.name) return false;
  
  const { includeTools, excludeTools } = mcpServerConfig;
  
  // excludeTools 优先于 includeTools
  if (excludeTools && excludeTools.includes(funcDecl.name)) {
    return false;
  }
  
  // 如果没有 includeTools 或工具在包含列表中
  return (
    !includeTools ||
    includeTools.some(
      (tool) => tool === funcDecl.name || tool.startsWith(`${funcDecl.name}(`)
    )
  );
}
```

### 提示发现

```typescript
export async function discoverPrompts(
  mcpServerName: string,
  mcpClient: Client,
  promptRegistry: PromptRegistry,
): Promise<void>
```

**功能**: 发现并注册 MCP 服务器提供的提示

**注册流程**:
1. **请求提示列表**: 调用 `prompts/list` 方法获取可用提示
2. **创建提示对象**: 为每个提示创建包含调用函数的对象
3. **注册提示**: 将提示注册到 `PromptRegistry`
4. **错误处理**: 优雅处理不支持提示的服务器

## 传输层管理

### 传输创建函数

```typescript
export async function createTransport(
  mcpServerName: string,
  mcpServerConfig: MCPServerConfig,
  debugMode: boolean,
): Promise<Transport>
```

**功能**: 根据配置创建适当的传输层

**支持的传输类型**:

#### 1. Google 凭据传输
```typescript
if (mcpServerConfig.authProviderType === AuthProviderType.GOOGLE_CREDENTIALS) {
  const provider = new GoogleCredentialProvider(mcpServerConfig);
  const transportOptions = { authProvider: provider };
  
  if (mcpServerConfig.httpUrl) {
    return new StreamableHTTPClientTransport(
      new URL(mcpServerConfig.httpUrl),
      transportOptions
    );
  } else if (mcpServerConfig.url) {
    return new SSEClientTransport(
      new URL(mcpServerConfig.url),
      transportOptions
    );
  }
}
```

#### 2. OAuth 认证传输
```typescript
// 检查 OAuth 配置或存储的令牌
let accessToken: string | null = null;
let hasOAuthConfig = mcpServerConfig.oauth?.enabled;

if (hasOAuthConfig && mcpServerConfig.oauth) {
  accessToken = await MCPOAuthProvider.getValidToken(
    mcpServerName,
    mcpServerConfig.oauth
  );
}

// 使用令牌创建传输
if (hasOAuthConfig && accessToken) {
  transportOptions.requestInit = {
    headers: {
      ...mcpServerConfig.headers,
      Authorization: `Bearer ${accessToken}`,
    },
  };
}
```

#### 3. Stdio 传输
```typescript
if (mcpServerConfig.command) {
  const transport = new StdioClientTransport({
    command: mcpServerConfig.command,
    args: mcpServerConfig.args || [],
    env: {
      ...process.env,
      ...(mcpServerConfig.env || {}),
    } as Record<string, string>,
    cwd: mcpServerConfig.cwd,
    stderr: 'pipe',
  });
  
  // 调试模式下监听 stderr
  if (debugMode) {
    transport.stderr!.on('data', (data) => {
      const stderrStr = data.toString().trim();
      console.debug(`[DEBUG] [MCP STDERR (${mcpServerName})]: `, stderrStr);
    });
  }
  
  return transport;
}
```

### 服务器连接函数

```typescript
export async function connectToMcpServer(
  mcpServerName: string,
  mcpServerConfig: MCPServerConfig,
  debugMode: boolean,
): Promise<Client>
```

**功能**: 创建并连接 MCP 客户端到服务器

**连接流程**:
1. **创建客户端**: 初始化 MCP 客户端实例
2. **超时补丁**: 修补 `callTool` 方法以支持请求超时
3. **创建传输**: 根据配置创建适当的传输层
4. **建立连接**: 连接客户端到传输
5. **错误处理**: 处理认证错误和连接失败

**超时补丁实现**:
```typescript
// 修补 Client.callTool 以使用请求超时
if ('callTool' in mcpClient) {
  const origCallTool = mcpClient.callTool.bind(mcpClient);
  mcpClient.callTool = function (params, resultSchema, options) {
    return origCallTool(params, resultSchema, {
      ...options,
      timeout: mcpServerConfig.timeout ?? MCP_DEFAULT_TIMEOUT_MSEC,
    });
  };
}
```

**认证错误处理**:
```typescript
// 检查 401 错误并触发 OAuth 流程
if (errorString.includes('401') && 
    (mcpServerConfig.httpUrl || mcpServerConfig.url)) {
  
  mcpServerRequiresOAuth.set(mcpServerName, true);
  
  // 提取 www-authenticate 头
  let wwwAuthenticate = extractWWWAuthenticateHeader(errorString);
  
  if (wwwAuthenticate) {
    // 尝试自动 OAuth 发现和认证
    const oauthSuccess = await handleAutomaticOAuth(
      mcpServerName,
      mcpServerConfig,
      wwwAuthenticate
    );
    
    if (oauthSuccess) {
      // 使用 OAuth 令牌重试连接
      // ... 重试逻辑
    }
  }
}
```

## 使用示例

### 基本 MCP 服务器配置

```typescript
import { discoverMcpTools, connectToMcpServer } from './mcp-client.js';
import { ToolRegistry } from './tool-registry.js';
import { PromptRegistry } from '../prompts/prompt-registry.js';

// 基本配置示例
const mcpServers = {
  'filesystem': {
    command: 'npx',
    args: ['@modelcontextprotocol/server-filesystem', '/tmp'],
    timeout: 30000,
  },
  'web-search': {
    url: 'https://api.example.com/mcp/sse',
    headers: {
      'User-Agent': 'gemini-cli/1.0',
    },
    oauth: {
      enabled: true,
      authorizationUrl: 'https://auth.example.com/oauth/authorize',
      tokenUrl: 'https://auth.example.com/oauth/token',
      scopes: ['read', 'search'],
    },
  },
  'http-api': {
    httpUrl: 'https://api.example.com/mcp/http',
    headers: {
      'Content-Type': 'application/json',
    },
  },
};

// 发现工具
const toolRegistry = new ToolRegistry();
const promptRegistry = new PromptRegistry();

await discoverMcpTools(
  mcpServers,
  undefined, // 无命令行服务器
  toolRegistry,
  promptRegistry,
  false // 非调试模式
);

console.log('Discovered tools:', toolRegistry.getAvailableTools());
```

### OAuth 认证集成

```typescript
import { 
  connectToMcpServer, 
  addMCPStatusChangeListener,
  getMCPServerStatus,
  MCPServerStatus 
} from './mcp-client.js';

// OAuth 配置
const oauthServerConfig = {
  url: 'https://secure-api.example.com/mcp',
  oauth: {
    enabled: true,
    authorizationUrl: 'https://auth.example.com/oauth/authorize',
    tokenUrl: 'https://auth.example.com/oauth/token',
    clientId: 'your-client-id',
    scopes: ['mcp:read', 'mcp:execute'],
  },
  timeout: 60000,
};

// 状态监听
addMCPStatusChangeListener((serverName, status) => {
  console.log(`Server ${serverName} status changed to: ${status}`);
  
  switch (status) {
    case MCPServerStatus.CONNECTING:
      console.log('Connecting to server...');
      break;
    case MCPServerStatus.CONNECTED:
      console.log('Server connected successfully');
      break;
    case MCPServerStatus.DISCONNECTED:
      console.log('Server disconnected, attempting reconnection...');
      // 实现重连逻辑
      break;
  }
});

// 连接到需要 OAuth 的服务器
try {
  const client = await connectToMcpServer(
    'secure-api',
    oauthServerConfig,
    true // 启用调试模式
  );
  
  console.log('Connected to OAuth-enabled MCP server');
} catch (error) {
  console.error('Failed to connect:', error);
  
  // 检查是否需要用户认证
  if (error.message.includes('OAuth authentication')) {
    console.log('Please run: /mcp auth secure-api');
  }
}
```

### 动态服务器发现

```typescript
import { discoverMcpTools, populateMcpServerCommand } from './mcp-client.js';

// 处理命令行指定的服务器
const commandLineServer = 'python /path/to/custom-mcp-server.py --verbose';

// 基础配置
let mcpServers = {
  'built-in-server': {
    command: 'mcp-server',
    args: ['--config', 'default.json'],
  },
};

// 添加命令行服务器
mcpServers = populateMcpServerCommand(mcpServers, commandLineServer);

// 发现所有服务器的工具
const toolRegistry = new ToolRegistry();
const promptRegistry = new PromptRegistry();

await discoverMcpTools(
  mcpServers,
  undefined,
  toolRegistry,
  promptRegistry,
  true // 调试模式
);

// 列出所有发现的工具
for (const tool of toolRegistry.getAvailableTools()) {
  console.log(`Tool: ${tool.name} from ${tool.serverName}`);
  console.log(`Description: ${tool.description}`);
}
```

### 工具过滤配置

```typescript
// 高级服务器配置，包含工具过滤
const advancedMcpServers = {
  'filesystem': {
    command: 'mcp-fs-server',
    args: ['/workspace'],
    // 只包含特定工具
    includeTools: ['read_file', 'write_file', 'list_directory'],
    timeout: 15000,
  },
  'web-tools': {
    url: 'https://web-tools.example.com/mcp',
    // 排除危险工具
    excludeTools: ['delete_all', 'format_disk', 'system_shutdown'],
    trust: false, // 不信任此服务器的工具
    headers: {
      'Authorization': 'Bearer your-api-key',
    },
  },
  'trusted-tools': {
    httpUrl: 'https://trusted.example.com/mcp',
    trust: true, // 信任此服务器的所有工具
    timeout: 120000, // 长超时用于复杂操作
  },
};

// 发现带过滤的工具
await discoverMcpTools(
  advancedMcpServers,
  undefined,
  toolRegistry,
  promptRegistry,
  false
);

// 验证过滤结果
const fileSystemTools = toolRegistry.getAvailableTools()
  .filter(tool => tool.serverName === 'filesystem');

console.log('Filesystem tools:', fileSystemTools.map(t => t.name));
// 输出: ['read_file', 'write_file', 'list_directory']
```

### 提示系统集成

```typescript
import { invokeMcpPrompt, discoverPrompts } from './mcp-client.js';

// 发现并使用提示
const client = await connectToMcpServer('prompt-server', {
  command: 'mcp-prompt-server',
  args: ['--templates', 'all'],
}, false);

// 发现提示
await discoverPrompts('prompt-server', client, promptRegistry);

// 获取可用提示
const availablePrompts = promptRegistry.getAvailablePrompts();
console.log('Available prompts:', availablePrompts.map(p => p.name));

// 调用特定提示
for (const prompt of availablePrompts) {
  if (prompt.name === 'code-review') {
    try {
      const result = await prompt.invoke({
        code: 'function hello() { return "world"; }',
        language: 'javascript',
        focus: 'best-practices',
      });
      
      console.log('Code review result:', result);
    } catch (error) {
      console.error('Failed to invoke prompt:', error);
    }
    break;
  }
}
```

## 错误处理和恢复

### 连接错误处理

```typescript
// 自定义错误处理器
class MCPConnectionManager {
  private retryAttempts = new Map<string, number>();
  private maxRetries = 3;
  private retryDelay = 5000;

  async connectWithRetry(
    serverName: string,
    config: MCPServerConfig
  ): Promise<Client | null> {
    const attempts = this.retryAttempts.get(serverName) || 0;
    
    try {
      const client = await connectToMcpServer(serverName, config, false);
      this.retryAttempts.delete(serverName); // 重置重试计数
      return client;
    } catch (error) {
      console.error(`Connection attempt ${attempts + 1} failed for ${serverName}:`, error);
      
      if (attempts < this.maxRetries) {
        this.retryAttempts.set(serverName, attempts + 1);
        
        // 指数退避
        const delay = this.retryDelay * Math.pow(2, attempts);
        console.log(`Retrying connection to ${serverName} in ${delay}ms...`);
        
        await new Promise(resolve => setTimeout(resolve, delay));
        return this.connectWithRetry(serverName, config);
      } else {
        console.error(`Max retries exceeded for server ${serverName}`);
        this.retryAttempts.delete(serverName);
        return null;
      }
    }
  }
}

// 使用示例
const connectionManager = new MCPConnectionManager();

for (const [serverName, config] of Object.entries(mcpServers)) {
  const client = await connectionManager.connectWithRetry(serverName, config);
  if (client) {
    console.log(`Successfully connected to ${serverName}`);
    // 继续工具发现
  } else {
    console.warn(`Failed to connect to ${serverName} after all retries`);
  }
}
```

### OAuth 错误恢复

```typescript
// OAuth 错误恢复管理器
class OAuthRecoveryManager {
  private tokenRefreshInProgress = new Set<string>();

  async handleOAuthError(
    serverName: string,
    config: MCPServerConfig,
    error: Error
  ): Promise<boolean> {
    if (this.tokenRefreshInProgress.has(serverName)) {
      // 避免并发刷新
      return false;
    }

    this.tokenRefreshInProgress.add(serverName);

    try {
      if (error.message.includes('401') || error.message.includes('unauthorized')) {
        console.log(`Attempting OAuth token refresh for ${serverName}...`);
        
        // 尝试刷新令牌
        const credentials = await MCPOAuthTokenStorage.getToken(serverName);
        if (credentials && credentials.refreshToken) {
          const newToken = await MCPOAuthProvider.refreshToken(
            serverName,
            credentials.refreshToken
          );
          
          if (newToken) {
            console.log(`OAuth token refreshed for ${serverName}`);
            return true;
          }
        }
        
        // 如果刷新失败，尝试重新认证
        console.log(`OAuth refresh failed, requiring re-authentication for ${serverName}`);
        console.log(`Please run: /mcp auth ${serverName}`);
        return false;
      }
      
      return false;
    } finally {
      this.tokenRefreshInProgress.delete(serverName);
    }
  }
}

// 集成到连接流程
const oauthRecovery = new OAuthRecoveryManager();

try {
  const client = await connectToMcpServer(serverName, config, debugMode);
  return client;
} catch (error) {
  const recovered = await oauthRecovery.handleOAuthError(serverName, config, error);
  
  if (recovered) {
    // 重试连接
    return await connectToMcpServer(serverName, config, debugMode);
  } else {
    throw error;
  }
}
```

## 安全考虑

### 工具验证和过滤

```typescript
// 安全工具过滤器
class MCPSecurityFilter {
  private dangerousPatterns = [
    /rm\s+-rf/i,
    /delete|remove|destroy/i,
    /format|mkfs/i,
    /sudo|admin/i,
    /system|exec|eval/i,
  ];

  private trustedServers = new Set(['localhost', 'internal.company.com']);

  validateTool(tool: DiscoveredMCPTool): boolean {
    // 检查服务器信任级别
    if (!this.trustedServers.has(tool.serverName) && !tool.trusted) {
      console.warn(`Tool ${tool.name} from untrusted server ${tool.serverName}`);
      
      // 检查危险模式
      const toolInfo = `${tool.name} ${tool.description}`;
      for (const pattern of this.dangerousPatterns) {
        if (pattern.test(toolInfo)) {
          console.error(`Blocking dangerous tool: ${tool.name}`);
          return false;
        }
      }
    }

    return true;
  }

  filterDiscoveredTools(tools: DiscoveredMCPTool[]): DiscoveredMCPTool[] {
    return tools.filter(tool => this.validateTool(tool));
  }
}

// 在工具发现中使用安全过滤
const securityFilter = new MCPSecurityFilter();

export async function secureDiscoverTools(
  mcpServerName: string,
  mcpServerConfig: MCPServerConfig, 
  mcpClient: Client,
): Promise<DiscoveredMCPTool[]> {
  const allTools = await discoverTools(mcpServerName, mcpServerConfig, mcpClient);
  return securityFilter.filterDiscoveredTools(allTools);
}
```

### 传输安全

```typescript
// 安全传输配置
const secureTransportOptions = {
  // HTTPS/WSS 强制
  enforceSecure: true,
  // 证书验证
  rejectUnauthorized: true,
  // 超时设置
  timeout: 30000,
  // 自定义证书 pinning
  certificateFingerprints: [
    'sha256:1234567890abcdef...',
  ],
};

// 验证服务器证书
function validateServerCertificate(serverName: string, cert: any): boolean {
  // 实现证书验证逻辑
  return true;
}

// 创建安全传输
export async function createSecureTransport(
  mcpServerName: string,
  mcpServerConfig: MCPServerConfig,
): Promise<Transport> {
  // 强制 HTTPS 用于生产环境
  if (process.env.NODE_ENV === 'production') {
    if (mcpServerConfig.url && !mcpServerConfig.url.startsWith('https://')) {
      throw new Error(`Insecure URL not allowed in production: ${mcpServerConfig.url}`);
    }
    if (mcpServerConfig.httpUrl && !mcpServerConfig.httpUrl.startsWith('https://')) {
      throw new Error(`Insecure URL not allowed in production: ${mcpServerConfig.httpUrl}`);
    }
  }

  return createTransport(mcpServerName, mcpServerConfig, false);
}
```

## 性能优化

### 连接池管理

```typescript
// MCP 连接池
class MCPConnectionPool {
  private connections = new Map<string, Client>();
  private connectionPromises = new Map<string, Promise<Client>>();
  private maxConnections = 10;

  async getConnection(
    serverName: string,
    config: MCPServerConfig
  ): Promise<Client> {
    // 检查现有连接
    const existing = this.connections.get(serverName);
    if (existing) {
      return existing;
    }

    // 检查正在进行的连接
    const pending = this.connectionPromises.get(serverName);
    if (pending) {
      return pending;
    }

    // 创建新连接
    const connectionPromise = this.createConnection(serverName, config);
    this.connectionPromises.set(serverName, connectionPromise);

    try {
      const client = await connectionPromise;
      this.connections.set(serverName, client);
      return client;
    } finally {
      this.connectionPromises.delete(serverName);
    }
  }

  private async createConnection(
    serverName: string,
    config: MCPServerConfig
  ): Promise<Client> {
    // 检查连接数限制
    if (this.connections.size >= this.maxConnections) {
      throw new Error('Maximum connections exceeded');
    }

    const client = await connectToMcpServer(serverName, config, false);
    
    // 设置错误处理
    client.onerror = (error) => {
      console.error(`Connection error for ${serverName}:`, error);
      this.connections.delete(serverName);
    };

    return client;
  }

  closeConnection(serverName: string): void {
    const client = this.connections.get(serverName);
    if (client) {
      client.close();
      this.connections.delete(serverName);
    }
  }

  closeAllConnections(): void {
    for (const [serverName, client] of this.connections) {
      client.close();
    }
    this.connections.clear();
  }
}

// 全局连接池
const connectionPool = new MCPConnectionPool();

// 在工具发现中使用连接池
export async function pooledConnectAndDiscover(
  mcpServerName: string,
  mcpServerConfig: MCPServerConfig,
  toolRegistry: ToolRegistry,
  promptRegistry: PromptRegistry,
): Promise<void> {
  const client = await connectionPool.getConnection(mcpServerName, mcpServerConfig);
  
  try {
    await discoverPrompts(mcpServerName, client, promptRegistry);
    const tools = await discoverTools(mcpServerName, mcpServerConfig, client);
    
    for (const tool of tools) {
      toolRegistry.registerTool(tool);
    }
  } catch (error) {
    console.error(`Discovery failed for ${mcpServerName}:`, error);
    connectionPool.closeConnection(mcpServerName);
    throw error;
  }
}
```

### 缓存优化

```typescript
// 发现结果缓存
class MCPDiscoveryCache {
  private toolCache = new Map<string, DiscoveredMCPTool[]>();
  private promptCache = new Map<string, DiscoveredMCPPrompt[]>();
  private cacheTimeout = 5 * 60 * 1000; // 5分钟
  private cacheTimestamps = new Map<string, number>();

  getCachedTools(serverName: string): DiscoveredMCPTool[] | null {
    if (this.isCacheValid(serverName)) {
      return this.toolCache.get(serverName) || null;
    }
    return null;
  }

  setCachedTools(serverName: string, tools: DiscoveredMCPTool[]): void {
    this.toolCache.set(serverName, tools);
    this.cacheTimestamps.set(serverName, Date.now());
  }

  private isCacheValid(serverName: string): boolean {
    const timestamp = this.cacheTimestamps.get(serverName);
    if (!timestamp) return false;
    
    return Date.now() - timestamp < this.cacheTimeout;
  }

  clearCache(serverName?: string): void {
    if (serverName) {
      this.toolCache.delete(serverName);
      this.promptCache.delete(serverName);
      this.cacheTimestamps.delete(serverName);
    } else {
      this.toolCache.clear();
      this.promptCache.clear();
      this.cacheTimestamps.clear();
    }
  }
}

// 使用缓存的发现函数
const discoveryCache = new MCPDiscoveryCache();

export async function cachedDiscoverTools(
  mcpServerName: string,
  mcpServerConfig: MCPServerConfig,
  mcpClient: Client,
): Promise<DiscoveredMCPTool[]> {
  // 尝试从缓存获取
  const cached = discoveryCache.getCachedTools(mcpServerName);
  if (cached) {
    console.log(`Using cached tools for ${mcpServerName}`);
    return cached;
  }

  // 执行发现
  const tools = await discoverTools(mcpServerName, mcpServerConfig, mcpClient);
  
  // 缓存结果
  discoveryCache.setCachedTools(mcpServerName, tools);
  
  return tools;
}
```

## 最佳实践

### 配置管理

```typescript
// ✅ 推荐：结构化配置验证
interface ValidatedMCPServerConfig extends MCPServerConfig {
  validated: boolean;
  securityLevel: 'low' | 'medium' | 'high';
}

function validateMCPConfig(
  serverName: string,
  config: MCPServerConfig
): ValidatedMCPServerConfig {
  // 验证必需字段
  if (!config.command && !config.url && !config.httpUrl) {
    throw new Error(`Invalid MCP config for ${serverName}: missing connection method`);
  }

  // 评估安全级别
  let securityLevel: 'low' | 'medium' | 'high' = 'medium';
  
  if (config.command && config.command.includes('sudo')) {
    securityLevel = 'high';
  } else if (config.trust === false) {
    securityLevel = 'low';
  }

  return {
    ...config,
    validated: true,
    securityLevel,
  };
}

// ✅ 推荐：环境特定配置
const getEnvironmentConfig = () => {
  const isDevelopment = process.env.NODE_ENV === 'development';
  const isProduction = process.env.NODE_ENV === 'production';

  return {
    defaultTimeout: isDevelopment ? 30000 : 10000,
    debugMode: isDevelopment,
    enforceHttps: isProduction,
    maxRetries: isProduction ? 3 : 1,
  };
};
```

### 监控和日志

```typescript
// ✅ 推荐：结构化日志记录
class MCPLogger {
  private logLevel: 'debug' | 'info' | 'warn' | 'error';

  constructor(logLevel: 'debug' | 'info' | 'warn' | 'error' = 'info') {
    this.logLevel = logLevel;
  }

  logConnection(serverName: string, status: 'connecting' | 'connected' | 'disconnected', details?: any): void {
    const logEntry = {
      timestamp: new Date().toISOString(),
      component: 'mcp-client',
      server: serverName,
      event: 'connection',
      status,
      details,
    };

    console.log(JSON.stringify(logEntry));
  }

  logToolDiscovery(serverName: string, toolCount: number, errors?: string[]): void {
    const logEntry = {
      timestamp: new Date().toISOString(),
      component: 'mcp-client',
      server: serverName,
      event: 'tool-discovery',
      toolCount,
      errors,
    };

    console.log(JSON.stringify(logEntry));
  }

  logOAuthFlow(serverName: string, stage: string, success: boolean, error?: string): void {
    const logEntry = {
      timestamp: new Date().toISOString(),
      component: 'mcp-client',
      server: serverName,
      event: 'oauth',
      stage,
      success,
      error,
    };

    console.log(JSON.stringify(logEntry));
  }
}

// 全局日志记录器
const mcpLogger = new MCPLogger(
  process.env.NODE_ENV === 'development' ? 'debug' : 'info'
);

// 在关键函数中使用
export async function loggedConnectToMcpServer(
  mcpServerName: string,
  mcpServerConfig: MCPServerConfig,
  debugMode: boolean,
): Promise<Client> {
  mcpLogger.logConnection(mcpServerName, 'connecting');
  
  try {
    const client = await connectToMcpServer(mcpServerName, mcpServerConfig, debugMode);
    mcpLogger.logConnection(mcpServerName, 'connected');
    return client;
  } catch (error) {
    mcpLogger.logConnection(mcpServerName, 'disconnected', {
      error: error.message,
    });
    throw error;
  }
}
```

### 测试支持

```typescript
// ✅ 推荐：测试辅助函数
export class MCPTestHelper {
  static createMockServerConfig(overrides: Partial<MCPServerConfig> = {}): MCPServerConfig {
    return {
      command: 'mock-mcp-server',
      args: ['--test-mode'],
      timeout: 5000,
      trust: true,
      ...overrides,
    };
  }

  static createMockClient(): Client {
    const mockClient = {
      connect: jest.fn().mockResolvedValue(undefined),
      close: jest.fn(),
      callTool: jest.fn(),
      request: jest.fn(),
      onerror: null,
    } as any;

    return mockClient;
  }

  static async setupTestEnvironment(): Promise<{
    toolRegistry: ToolRegistry;
    promptRegistry: PromptRegistry;
    mockServers: Record<string, MCPServerConfig>;
  }> {
    const toolRegistry = new ToolRegistry();
    const promptRegistry = new PromptRegistry();
    
    const mockServers = {
      'test-server': this.createMockServerConfig(),
    };

    return { toolRegistry, promptRegistry, mockServers };
  }
}

// 测试示例
describe('MCP Client Tests', () => {
  let testEnv: Awaited<ReturnType<typeof MCPTestHelper.setupTestEnvironment>>;

  beforeEach(async () => {
    testEnv = await MCPTestHelper.setupTestEnvironment();
  });

  it('should discover tools from mock server', async () => {
    // 模拟工具发现
    const mockTools = [
      new DiscoveredMCPTool(
        {} as any, // mockCallableTool
        'test-server',
        'test-tool',
        'A test tool',
        { type: 'object', properties: {} },
        5000,
        true
      ),
    ];

    // 测试工具注册
    for (const tool of mockTools) {
      testEnv.toolRegistry.registerTool(tool);
    }

    const availableTools = testEnv.toolRegistry.getAvailableTools();
    expect(availableTools).toHaveLength(1);
    expect(availableTools[0].name).toBe('test-tool');
  });
});
```