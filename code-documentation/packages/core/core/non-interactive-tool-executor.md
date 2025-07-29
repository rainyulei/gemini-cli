# NonInteractiveToolExecutor - 非交互式工具执行器文档

## 概述

`nonInteractiveToolExecutor.ts` 是 Gemini CLI 的非交互式工具执行模块，提供简化的工具调用执行功能。与 `CoreToolScheduler` 不同，该模块专注于直接执行单个工具调用，不处理用户确认、状态管理或实时输出更新，适用于自动化场景和批处理操作。

## 主要功能

- **单一工具执行**: 专注于执行单个工具调用，不支持批量操作
- **非交互式设计**: 跳过用户确认和交互环节，直接执行工具
- **简化错误处理**: 提供基本的错误处理和日志记录
- **性能优化**: 去除交互式功能的开销，提供更快的执行速度
- **完整日志记录**: 记录工具调用的完整生命周期信息

## 核心函数

### `executeToolCall` - 非交互式工具执行

```typescript
export async function executeToolCall(
  config: Config,
  toolCallRequest: ToolCallRequestInfo,
  toolRegistry: ToolRegistry,
  abortSignal?: AbortSignal,
): Promise<ToolCallResponseInfo>
```

**功能**: 非交互式执行单个工具调用，返回执行结果

**参数**:
- `config: Config` - 系统配置对象，用于日志记录配置
- `toolCallRequest: ToolCallRequestInfo` - 工具调用请求信息
- `toolRegistry: ToolRegistry` - 工具注册表，用于查找和获取工具实例
- `abortSignal?: AbortSignal` - 可选的取消信号，用于中断执行

**返回值**: `Promise<ToolCallResponseInfo>` - 工具调用响应信息

**执行流程**:
```typescript
async function executeToolCall(...) {
  // 1. 工具查找和验证
  const tool = toolRegistry.getTool(toolCallRequest.name);
  if (!tool) {
    // 返回工具未找到错误
  }
  
  // 2. 直接执行工具（跳过确认）
  const toolResult = await tool.execute(
    toolCallRequest.args,
    effectiveAbortSignal
    // 无实时输出回调
  );
  
  // 3. 结果转换和返回
  const response = convertToFunctionResponse(
    toolCallRequest.name,
    toolCallRequest.callId,
    toolResult.llmContent
  );
  
  return {
    callId: toolCallRequest.callId,
    responseParts: response,
    resultDisplay: toolResult.returnDisplay,
    error: undefined
  };
}
```

## 核心特性

### 1. 非交互式执行

**设计理念**:
- 跳过所有用户确认步骤
- 不支持实时输出更新
- 适用于自动化和批处理场景

```typescript
// 对比：交互式 vs 非交互式执行

// 交互式执行（CoreToolScheduler）
const scheduler = new CoreToolScheduler({
  // 支持确认处理器
  onConfirm: async (toolCall) => {
    // 用户确认逻辑
    return await getUserConfirmation(toolCall);
  },
  // 支持实时输出更新
  outputUpdateHandler: (callId, chunk) => {
    updateUI(callId, chunk);
  }
});

// 非交互式执行
const result = await executeToolCall(
  config,
  toolCallRequest,
  toolRegistry,
  abortSignal
);
// 直接返回结果，无用户交互
```

### 2. 简化的生命周期

```typescript
// 生命周期状态对比

// CoreToolScheduler 的复杂状态
type ToolCallStatus = 
  | 'validating'
  | 'scheduled' 
  | 'awaiting_approval'
  | 'executing'
  | 'success'
  | 'error'
  | 'cancelled';

// NonInteractiveToolExecutor 的简化状态
// 只有：执行中 → 成功/失败
```

### 3. 性能优化

**优化策略**:
- 无状态管理开销
- 无用户界面更新
- 无复杂的确认流程
- 直接的错误处理

```typescript
// 性能对比分析
const performanceComparison = {
  interactive: {
    overhead: 'High',
    features: ['用户确认', '实时更新', '状态管理', '错误恢复'],
    useCase: '用户界面交互'
  },
  nonInteractive: {
    overhead: 'Low',
    features: ['直接执行', '基本日志', '错误处理'],
    useCase: '自动化批处理'
  }
};
```

## 错误处理机制

### 1. 工具未找到错误

```typescript
// 工具不存在时的处理
if (!tool) {
  const error = new Error(`Tool "${toolCallRequest.name}" not found in registry.`);
  
  // 记录错误日志
  logToolCall(config, {
    'event.name': 'tool_call',
    'event.timestamp': new Date().toISOString(),
    function_name: toolCallRequest.name,
    function_args: toolCallRequest.args,
    duration_ms: Date.now() - startTime,
    success: false,
    error: error.message,
    prompt_id: toolCallRequest.prompt_id,
  });
  
  // 返回标准错误响应
  return {
    callId: toolCallRequest.callId,
    responseParts: [{
      functionResponse: {
        id: toolCallRequest.callId,
        name: toolCallRequest.name,
        response: { error: error.message }
      }
    }],
    resultDisplay: error.message,
    error
  };
}
```

### 2. 执行时错误处理

```typescript
// 工具执行异常的处理
try {
  const toolResult = await tool.execute(
    toolCallRequest.args,
    effectiveAbortSignal
  );
  // 处理成功结果...
} catch (e) {
  const error = e instanceof Error ? e : new Error(String(e));
  
  // 错误日志记录
  logToolCall(config, {
    'event.name': 'tool_call',
    'event.timestamp': new Date().toISOString(),
    function_name: toolCallRequest.name,
    function_args: toolCallRequest.args,
    duration_ms: Date.now() - startTime,
    success: false,
    error: error.message,
    prompt_id: toolCallRequest.prompt_id,
  });
  
  // 返回错误响应
  return {
    callId: toolCallRequest.callId,
    responseParts: [/* 错误响应结构 */],
    resultDisplay: error.message,
    error
  };
}
```

## 日志记录系统

### 日志事件结构

```typescript
interface ToolCallLogEvent {
  'event.name': 'tool_call';
  'event.timestamp': string;        // ISO 时间戳
  function_name: string;            // 工具名称
  function_args: Record<string, unknown>; // 工具参数
  duration_ms: number;              // 执行时长（毫秒）
  success: boolean;                 // 执行是否成功
  error?: string;                   // 错误信息（如果失败）
  prompt_id?: string;               // 提示词ID（如果提供）
}
```

### 日志记录示例

```typescript
// 成功执行的日志
logToolCall(config, {
  'event.name': 'tool_call',
  'event.timestamp': '2025-01-29T10:30:45.123Z',
  function_name: 'read-file',
  function_args: { file_path: '/path/to/file.txt' },
  duration_ms: 150,
  success: true,
  prompt_id: 'prompt_123'
});

// 失败执行的日志
logToolCall(config, {
  'event.name': 'tool_call',
  'event.timestamp': '2025-01-29T10:30:45.456Z',
  function_name: 'write-file',
  function_args: { file_path: '/readonly/file.txt', content: 'data' },
  duration_ms: 50,
  success: false,
  error: 'Permission denied: /readonly/file.txt',
  prompt_id: 'prompt_124'
});
```

## 使用场景和示例

### 1. 自动化脚本中使用

```typescript
import { executeToolCall } from './nonInteractiveToolExecutor.js';
import { Config } from '../config/config.js';
import { ToolRegistry } from '../tools/toolRegistry.js';

// 自动化文件处理脚本
async function processFilesAutomatically(
  filePaths: string[],
  config: Config,
  toolRegistry: ToolRegistry
): Promise<void> {
  const abortController = new AbortController();
  
  for (const filePath of filePaths) {
    try {
      // 读取文件
      const readRequest: ToolCallRequestInfo = {
        callId: `read_${Date.now()}`,
        name: 'read-file',
        args: { file_path: filePath },
        prompt_id: 'automation_script'
      };
      
      const readResult = await executeToolCall(
        config,
        readRequest,
        toolRegistry,
        abortController.signal
      );
      
      if (readResult.error) {
        console.error(`Failed to read ${filePath}:`, readResult.error.message);
        continue;
      }
      
      // 处理文件内容...
      const processedContent = processContent(readResult.resultDisplay);
      
      // 写入处理后的文件
      const writeRequest: ToolCallRequestInfo = {
        callId: `write_${Date.now()}`,
        name: 'write-file',
        args: {
          file_path: filePath.replace('.txt', '.processed.txt'),
          content: processedContent
        },
        prompt_id: 'automation_script'
      };
      
      const writeResult = await executeToolCall(
        config,
        writeRequest,
        toolRegistry,
        abortController.signal
      );
      
      if (writeResult.error) {
        console.error(`Failed to write processed file:`, writeResult.error.message);
      } else {
        console.log(`Successfully processed: ${filePath}`);
      }
      
    } catch (error) {
      console.error(`Unexpected error processing ${filePath}:`, error);
    }
  }
}

function processContent(content: string): string {
  // 示例内容处理逻辑
  return content
    .split('\n')
    .map(line => line.trim())
    .filter(line => line.length > 0)
    .join('\n');
}
```

### 2. 批量工具调用执行器

```typescript
// 批量工具调用执行器
class BatchToolExecutor {
  constructor(
    private config: Config,
    private toolRegistry: ToolRegistry,
    private maxConcurrent: number = 5
  ) {}
  
  async executeBatch(
    requests: ToolCallRequestInfo[],
    abortSignal?: AbortSignal
  ): Promise<ToolCallResponseInfo[]> {
    const results: ToolCallResponseInfo[] = [];
    const errors: Error[] = [];
    
    // 分批处理，避免过多并发
    for (let i = 0; i < requests.length; i += this.maxConcurrent) {
      const batch = requests.slice(i, i + this.maxConcurrent);
      
      const batchPromises = batch.map(async (request) => {
        try {
          return await executeToolCall(
            this.config,
            request,
            this.toolRegistry,
            abortSignal
          );
        } catch (error) {
          errors.push(error instanceof Error ? error : new Error(String(error)));
          return null;
        }
      });
      
      const batchResults = await Promise.all(batchPromises);
      results.push(...batchResults.filter(result => result !== null));
      
      // 检查是否被取消
      if (abortSignal?.aborted) {
        throw new Error('Batch execution was aborted');
      }
    }
    
    if (errors.length > 0) {
      console.warn(`Batch execution completed with ${errors.length} errors`);
      errors.forEach((error, index) => {
        console.error(`Error ${index + 1}:`, error.message);
      });
    }
    
    return results;
  }
  
  // 并行执行（适用于独立的工具调用）
  async executeParallel(
    requests: ToolCallRequestInfo[],
    abortSignal?: AbortSignal
  ): Promise<ToolCallResponseInfo[]> {
    const promises = requests.map(request => 
      executeToolCall(
        this.config,
        request,
        this.toolRegistry,
        abortSignal
      )
    );
    
    try {
      return await Promise.all(promises);
    } catch (error) {
      console.error('Parallel execution failed:', error);
      throw error;
    }
  }
  
  // 串行执行（适用于有依赖关系的工具调用）
  async executeSequential(
    requests: ToolCallRequestInfo[],
    abortSignal?: AbortSignal
  ): Promise<ToolCallResponseInfo[]> {
    const results: ToolCallResponseInfo[] = [];
    
    for (const request of requests) {
      if (abortSignal?.aborted) {
        throw new Error('Sequential execution was aborted');
      }
      
      const result = await executeToolCall(
        this.config,
        request,
        this.toolRegistry,
        abortSignal
      );
      
      results.push(result);
      
      // 如果当前工具调用失败，可以选择停止或继续
      if (result.error) {
        console.warn(`Tool call ${request.name} failed:`, result.error.message);
        // 根据需要决定是否继续执行
      }
    }
    
    return results;
  }
}

// 使用示例
async function useBatchExecutor() {
  const batchExecutor = new BatchToolExecutor(config, toolRegistry, 3);
  
  const requests: ToolCallRequestInfo[] = [
    {
      callId: 'call_1',
      name: 'ls',
      args: { path: '/home/user' },
      prompt_id: 'batch_1'
    },
    {
      callId: 'call_2',
      name: 'read-file',
      args: { file_path: '/home/user/config.json' },
      prompt_id: 'batch_1'
    },
    {
      callId: 'call_3',
      name: 'grep',
      args: { pattern: 'error', file_path: '/var/log/app.log' },
      prompt_id: 'batch_1'
    }
  ];
  
  // 并行执行
  const parallelResults = await batchExecutor.executeParallel(requests);
  console.log('Parallel results:', parallelResults.length);
  
  // 串行执行
  const sequentialResults = await batchExecutor.executeSequential(requests);
  console.log('Sequential results:', sequentialResults.length);
}
```

### 3. API 服务中的工具调用

```typescript
// Express.js API 端点示例
import express from 'express';

const app = express();
app.use(express.json());

// 非交互式工具调用 API 端点
app.post('/api/tools/execute', async (req, res) => {
  try {
    const { toolName, args, callId } = req.body;
    
    // 验证请求
    if (!toolName || !args || !callId) {
      return res.status(400).json({
        error: 'Missing required fields: toolName, args, callId'
      });
    }
    
    // 创建工具调用请求
    const toolCallRequest: ToolCallRequestInfo = {
      callId,
      name: toolName,
      args,
      prompt_id: `api_${Date.now()}`
    };
    
    // 执行工具调用
    const result = await executeToolCall(
      config,
      toolCallRequest,
      toolRegistry
    );
    
    // 返回结果
    if (result.error) {
      res.status(500).json({
        success: false,
        error: result.error.message,
        callId: result.callId
      });
    } else {
      res.json({
        success: true,
        callId: result.callId,
        result: result.resultDisplay,
        responseParts: result.responseParts
      });
    }
    
  } catch (error) {
    console.error('API tool execution error:', error);
    res.status(500).json({
      success: false,
      error: 'Internal server error'
    });
  }
});

// 批量工具调用 API 端点
app.post('/api/tools/batch', async (req, res) => {
  try {
    const { requests } = req.body;
    
    if (!Array.isArray(requests)) {
      return res.status(400).json({
        error: 'requests must be an array'
      });
    }
    
    const batchExecutor = new BatchToolExecutor(config, toolRegistry);
    const results = await batchExecutor.executeBatch(requests);
    
    res.json({
      success: true,
      count: results.length,
      results: results.map(result => ({
        callId: result.callId,
        success: !result.error,
        error: result.error?.message,
        result: result.resultDisplay
      }))
    });
    
  } catch (error) {
    console.error('Batch API execution error:', error);
    res.status(500).json({
      success: false,
      error: 'Batch execution failed'
    });
  }
});
```

### 4. 测试和调试工具

```typescript
// 工具调用测试套件
class ToolCallTester {
  constructor(
    private config: Config,
    private toolRegistry: ToolRegistry
  ) {}
  
  async testTool(
    toolName: string,
    testCases: Array<{
      name: string;
      args: Record<string, unknown>;
      expectedSuccess: boolean;
      expectedContent?: string;
    }>
  ): Promise<void> {
    console.log(`\n=== Testing Tool: ${toolName} ===`);
    
    for (const testCase of testCases) {
      console.log(`\nTest Case: ${testCase.name}`);
      
      const request: ToolCallRequestInfo = {
        callId: `test_${Date.now()}`,
        name: toolName,
        args: testCase.args,
        prompt_id: 'test_suite'
      };
      
      try {
        const startTime = Date.now();
        const result = await executeToolCall(
          this.config,
          request,
          this.toolRegistry
        );
        const duration = Date.now() - startTime;
        
        const success = !result.error;
        const statusIcon = success ? '✅' : '❌';
        
        console.log(`  ${statusIcon} Status: ${success ? 'SUCCESS' : 'FAILED'}`);
        console.log(`  ⏱️  Duration: ${duration}ms`);
        
        if (result.error) {
          console.log(`  ❌ Error: ${result.error.message}`);
        } else {
          console.log(`  📄 Result: ${result.resultDisplay?.substring(0, 100)}...`);
        }
        
        // 验证期望结果
        if (testCase.expectedSuccess !== success) {
          console.log(`  ⚠️  Expected success: ${testCase.expectedSuccess}, got: ${success}`);
        }
        
        if (testCase.expectedContent && result.resultDisplay) {
          const containsExpected = result.resultDisplay.includes(testCase.expectedContent);
          if (!containsExpected) {
            console.log(`  ⚠️  Expected content not found: "${testCase.expectedContent}"`);
          }
        }
        
      } catch (error) {
        console.log(`  💥 Unexpected error: ${error}`);
      }
    }
  }
  
  async benchmarkTool(
    toolName: string,
    args: Record<string, unknown>,
    iterations: number = 10
  ): Promise<{
    averageMs: number;
    minMs: number;
    maxMs: number;
    successRate: number;
  }> {
    console.log(`\n=== Benchmarking Tool: ${toolName} (${iterations} iterations) ===`);
    
    const durations: number[] = [];
    let successCount = 0;
    
    for (let i = 0; i < iterations; i++) {
      const request: ToolCallRequestInfo = {
        callId: `bench_${i}`,
        name: toolName,
        args,
        prompt_id: 'benchmark'
      };
      
      try {
        const startTime = Date.now();
        const result = await executeToolCall(
          this.config,
          request,
          this.toolRegistry
        );
        const duration = Date.now() - startTime;
        
        durations.push(duration);
        if (!result.error) {
          successCount++;
        }
        
      } catch (error) {
        durations.push(0); // 记录失败的调用
      }
    }
    
    const averageMs = durations.reduce((a, b) => a + b, 0) / durations.length;
    const minMs = Math.min(...durations);
    const maxMs = Math.max(...durations);
    const successRate = successCount / iterations;
    
    console.log(`  📊 Average: ${averageMs.toFixed(2)}ms`);
    console.log(`  📊 Min: ${minMs}ms`);
    console.log(`  📊 Max: ${maxMs}ms`);
    console.log(`  📊 Success Rate: ${(successRate * 100).toFixed(1)}%`);
    
    return { averageMs, minMs, maxMs, successRate };
  }
}

// 使用测试工具
async function runToolTests() {
  const tester = new ToolCallTester(config, toolRegistry);
  
  // 测试文件读取工具
  await tester.testTool('read-file', [
    {
      name: 'Read existing file',
      args: { file_path: '/etc/passwd' },
      expectedSuccess: true,
      expectedContent: 'root'
    },
    {
      name: 'Read non-existent file',
      args: { file_path: '/nonexistent/file.txt' },
      expectedSuccess: false
    }
  ]);
  
  // 性能基准测试
  await tester.benchmarkTool('ls', { path: '/' }, 20);
}
```

## 性能优化和最佳实践

### 1. 并发控制

```typescript
// 并发限制器
class ConcurrencyLimiter {
  private running = 0;
  private queue: Array<() => void> = [];
  
  constructor(private maxConcurrent: number) {}
  
  async execute<T>(fn: () => Promise<T>): Promise<T> {
    return new Promise((resolve, reject) => {
      const task = async () => {
        try {
          this.running++;
          const result = await fn();
          resolve(result);
        } catch (error) {
          reject(error);
        } finally {
          this.running--;
          this.processQueue();
        }
      };
      
      if (this.running < this.maxConcurrent) {
        task();
      } else {
        this.queue.push(task);
      }
    });
  }
  
  private processQueue(): void {
    if (this.queue.length > 0 && this.running < this.maxConcurrent) {
      const nextTask = this.queue.shift()!;
      nextTask();
    }
  }
}

// 使用并发限制器
const limiter = new ConcurrencyLimiter(3);

async function executeConcurrentTools(requests: ToolCallRequestInfo[]) {
  const promises = requests.map(request => 
    limiter.execute(() => 
      executeToolCall(config, request, toolRegistry)
    )
  );
  
  return Promise.all(promises);
}
```

### 2. 结果缓存

```typescript
// 工具调用结果缓存
class ToolCallCache {
  private cache = new Map<string, ToolCallResponseInfo>();
  private readonly maxSize = 1000;
  
  private generateKey(request: ToolCallRequestInfo): string {
    return `${request.name}:${JSON.stringify(request.args)}`;
  }
  
  get(request: ToolCallRequestInfo): ToolCallResponseInfo | undefined {
    return this.cache.get(this.generateKey(request));
  }
  
  set(request: ToolCallRequestInfo, response: ToolCallResponseInfo): void {
    const key = this.generateKey(request);
    
    // 限制缓存大小
    if (this.cache.size >= this.maxSize) {
      const firstKey = this.cache.keys().next().value;
      this.cache.delete(firstKey);
    }
    
    this.cache.set(key, response);
  }
  
  clear(): void {
    this.cache.clear();
  }
}

// 带缓存的工具执行器
const toolCache = new ToolCallCache();

async function executeToolCallWithCache(
  config: Config,
  toolCallRequest: ToolCallRequestInfo,
  toolRegistry: ToolRegistry,
  abortSignal?: AbortSignal
): Promise<ToolCallResponseInfo> {
  // 检查缓存
  const cached = toolCache.get(toolCallRequest);
  if (cached) {
    console.log(`Cache hit for ${toolCallRequest.name}`);
    return cached;
  }
  
  // 执行工具调用
  const result = await executeToolCall(
    config,
    toolCallRequest,
    toolRegistry,
    abortSignal
  );
  
  // 缓存成功结果
  if (!result.error) {
    toolCache.set(toolCallRequest, result);
  }
  
  return result;
}
```

### 3. 错误重试机制

```typescript
// 重试配置
interface RetryConfig {
  maxAttempts: number;
  delayMs: number;
  backoffMultiplier: number;
  retryableErrors: string[];
}

const defaultRetryConfig: RetryConfig = {
  maxAttempts: 3,
  delayMs: 1000,
  backoffMultiplier: 2,
  retryableErrors: ['ECONNRESET', 'ETIMEDOUT', 'ENOTFOUND']
};

// 带重试的工具执行器
async function executeToolCallWithRetry(
  config: Config,
  toolCallRequest: ToolCallRequestInfo,
  toolRegistry: ToolRegistry,
  abortSignal?: AbortSignal,
  retryConfig: RetryConfig = defaultRetryConfig
): Promise<ToolCallResponseInfo> {
  let lastError: Error | undefined;
  
  for (let attempt = 1; attempt <= retryConfig.maxAttempts; attempt++) {
    try {
      const result = await executeToolCall(
        config,
        toolCallRequest,
        toolRegistry,
        abortSignal
      );
      
      // 如果有错误但不在可重试列表中，直接返回
      if (result.error && !isRetryableError(result.error, retryConfig)) {
        return result;
      }
      
      // 成功或不可重试错误，返回结果
      if (!result.error || attempt === retryConfig.maxAttempts) {
        return result;
      }
      
      lastError = result.error;
      
    } catch (error) {
      lastError = error instanceof Error ? error : new Error(String(error));
      
      if (!isRetryableError(lastError, retryConfig) || 
          attempt === retryConfig.maxAttempts) {
        throw lastError;
      }
    }
    
    // 等待后重试
    const delay = retryConfig.delayMs * Math.pow(retryConfig.backoffMultiplier, attempt - 1);
    console.log(`Attempt ${attempt} failed, retrying in ${delay}ms...`);
    await new Promise(resolve => setTimeout(resolve, delay));
    
    // 检查是否被取消
    if (abortSignal?.aborted) {
      throw new Error('Execution was aborted during retry');
    }
  }
  
  throw lastError || new Error('All retry attempts failed');
}

function isRetryableError(error: Error, config: RetryConfig): boolean {
  return config.retryableErrors.some(retryableError => 
    error.message.includes(retryableError)
  );
}
```

## 与 CoreToolScheduler 的对比

### 功能对比表

| 特性 | NonInteractiveToolExecutor | CoreToolScheduler |
|------|---------------------------|-------------------|
| 用户确认 | ❌ 不支持 | ✅ 完整支持 |
| 实时输出 | ❌ 不支持 | ✅ 支持回调 |
| 状态管理 | ❌ 简化 | ✅ 完整状态机 |
| 批量执行 | ❌ 单个调用 | ✅ 支持批量 |
| 修改支持 | ❌ 不支持 | ✅ 外部编辑器 |
| 性能开销 | ✅ 低 | ❌ 高 |
| 适用场景 | 自动化/批处理 | 用户交互 |
| 错误处理 | ✅ 基本 | ✅ 完整 |
| 日志记录 | ✅ 支持 | ✅ 支持 |

### 选择指南

**使用 NonInteractiveToolExecutor 当**:
- 构建自动化脚本或批处理系统
- 不需要用户交互和确认
- 对性能有较高要求
- 简单的工具调用场景

**使用 CoreToolScheduler 当**:
- 构建用户界面应用
- 需要用户确认和交互
- 需要实时输出更新
- 复杂的工具调用编排

```typescript
// 使用选择示例
class ToolExecutionService {
  constructor(
    private config: Config,
    private toolRegistry: ToolRegistry
  ) {}
  
  // 用户界面模式：使用 CoreToolScheduler
  async executeInteractive(
    requests: ToolCallRequestInfo[],
    handlers: {
      onConfirm: ConfirmHandler;
      onUpdate: ToolCallsUpdateHandler;
      onComplete: AllToolCallsCompleteHandler;
    }
  ): Promise<void> {
    const scheduler = new CoreToolScheduler({
      toolRegistry: Promise.resolve(this.toolRegistry),
      onToolCallsUpdate: handlers.onUpdate,
      onAllToolCallsComplete: handlers.onComplete,
      getPreferredEditor: () => 'vscode',
      config: this.config
    });
    
    await scheduler.schedule(requests, new AbortController().signal);
  }
  
  // 自动化模式：使用 NonInteractiveToolExecutor
  async executeAutomated(
    requests: ToolCallRequestInfo[],
    abortSignal?: AbortSignal
  ): Promise<ToolCallResponseInfo[]> {
    const results: ToolCallResponseInfo[] = [];
    
    for (const request of requests) {
      const result = await executeToolCall(
        this.config,
        request,
        this.toolRegistry,
        abortSignal
      );
      results.push(result);
    }
    
    return results;
  }
}
```

## 测试示例

### 单元测试

```typescript
import { executeToolCall } from './nonInteractiveToolExecutor.js';
import { Config } from '../config/config.js';
import { ToolRegistry } from '../tools/toolRegistry.js';

describe('NonInteractiveToolExecutor', () => {
  let config: Config;
  let toolRegistry: ToolRegistry;
  let mockTool: any;
  
  beforeEach(() => {
    config = new Config();
    toolRegistry = new ToolRegistry();
    
    mockTool = {
      execute: jest.fn(),
      canUpdateOutput: false
    };
  });
  
  describe('executeToolCall', () => {
    it('should execute tool successfully', async () => {
      toolRegistry.registerTool('test-tool', mockTool);
      mockTool.execute.mockResolvedValue({
        llmContent: 'Test result',
        returnDisplay: 'Display result'
      });
      
      const request: ToolCallRequestInfo = {
        callId: 'test-call',
        name: 'test-tool',
        args: { param: 'value' },
        prompt_id: 'test-prompt'
      };
      
      const result = await executeToolCall(
        config,
        request,
        toolRegistry
      );
      
      expect(result.error).toBeUndefined();
      expect(result.callId).toBe('test-call');
      expect(result.resultDisplay).toBe('Display result');
      expect(mockTool.execute).toHaveBeenCalledWith(
        { param: 'value' },
        expect.any(AbortSignal),
        undefined // 无实时输出回调
      );
    });
    
    it('should handle tool not found', async () => {
      const request: ToolCallRequestInfo = {
        callId: 'test-call',
        name: 'nonexistent-tool',
        args: {},
        prompt_id: 'test-prompt'
      };
      
      const result = await executeToolCall(
        config,
        request,
        toolRegistry
      );
      
      expect(result.error).toBeDefined();
      expect(result.error?.message).toContain('not found in registry');
      expect(result.callId).toBe('test-call');
    });
    
    it('should handle tool execution error', async () => {
      toolRegistry.registerTool('failing-tool', mockTool);
      mockTool.execute.mockRejectedValue(new Error('Tool execution failed'));
      
      const request: ToolCallRequestInfo = {
        callId: 'test-call',
        name: 'failing-tool',
        args: {},
        prompt_id: 'test-prompt'
      };
      
      const result = await executeToolCall(
        config,
        request,
        toolRegistry
      );
      
      expect(result.error).toBeDefined();
      expect(result.error?.message).toBe('Tool execution failed');
      expect(result.resultDisplay).toBe('Tool execution failed');
    });
    
    it('should respect abort signal', async () => {
      toolRegistry.registerTool('slow-tool', mockTool);
      
      const abortController = new AbortController();
      
      // 模拟慢工具执行
      mockTool.execute.mockImplementation(async (args, signal) => {
        return new Promise((resolve, reject) => {
          const timeout = setTimeout(() => {
            resolve({ llmContent: 'slow result', returnDisplay: 'slow display' });
          }, 1000);
          
          signal.addEventListener('abort', () => {
            clearTimeout(timeout);
            reject(new Error('Aborted'));
          });
        });
      });
      
      const request: ToolCallRequestInfo = {
        callId: 'test-call',
        name: 'slow-tool',
        args: {},
        prompt_id: 'test-prompt'
      };
      
      // 立即取消
      setTimeout(() => abortController.abort(), 100);
      
      const result = await executeToolCall(
        config,
        request,
        toolRegistry,
        abortController.signal
      );
      
      expect(result.error).toBeDefined();
      expect(result.error?.message).toBe('Aborted');
    });
  });
});
```

### 集成测试

```typescript
describe('Integration Tests', () => {
  it('should work with real tools', async () => {
    const config = new Config();
    const toolRegistry = new ToolRegistry();
    
    // 注册真实工具
    toolRegistry.registerTool('ls', new LSTool());
    
    const request: ToolCallRequestInfo = {
      callId: 'integration-test',
      name: 'ls',
      args: { path: '.' },
      prompt_id: 'integration'
    };
    
    const result = await executeToolCall(
      config,
      request,
      toolRegistry
    );
    
    expect(result.error).toBeUndefined();
    expect(result.resultDisplay).toBeDefined();
    expect(typeof result.resultDisplay).toBe('string');
  });
});
```

## 最佳实践总结

### 1. 使用场景选择
- **自动化脚本**: 使用 NonInteractiveToolExecutor
- **用户界面**: 使用 CoreToolScheduler
- **API 服务**: 使用 NonInteractiveToolExecutor
- **批处理**: 结合使用 BatchToolExecutor

### 2. 错误处理
- 始终检查 `result.error`
- 使用适当的重试策略
- 记录详细的错误信息

### 3. 性能优化
- 实现并发控制
- 使用结果缓存
- 避免不必要的工具调用

### 4. 监控和日志
- 利用内置日志记录
- 监控执行时间和成功率
- 设置适当的告警

该模块为 Gemini CLI 提供了高效的非交互式工具执行能力，是构建自动化和批处理系统的核心组件。