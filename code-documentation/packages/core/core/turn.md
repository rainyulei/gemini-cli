# Turn - 对话回合管理文档

## 概述

`turn.ts` 是 Gemini CLI 的对话回合管理模块，负责管理单个对话回合中与 Gemini API 的交互。该模块实现了复杂的事件流处理系统，处理内容生成、工具调用、思考过程、错误处理等多种类型的响应事件，是整个对话系统的核心组件。

## 主要功能

- **对话回合管理**: 管理单个用户输入到完整响应的整个生命周期
- **事件流处理**: 将 Gemini API 响应转换为结构化的事件流
- **工具调用协调**: 处理 AI 模型发起的工具调用请求
- **思考过程提取**: 识别和处理 AI 模型的思考过程
- **错误处理**: 完善的错误处理和错误报告机制
- **调试支持**: 保存详细的调试信息用于问题排查

## 核心接口和类型

### 事件类型枚举

```typescript
export enum GeminiEventType {
  Content = 'content',                    // 文本内容事件
  ToolCallRequest = 'tool_call_request',  // 工具调用请求事件
  ToolCallResponse = 'tool_call_response', // 工具调用响应事件
  ToolCallConfirmation = 'tool_call_confirmation', // 工具调用确认事件
  UserCancelled = 'user_cancelled',       // 用户取消事件
  Error = 'error',                       // 错误事件
  ChatCompressed = 'chat_compressed',     // 聊天历史压缩事件
  Thought = 'thought',                   // 思考过程事件
  MaxSessionTurns = 'max_session_turns',  // 达到最大回合数事件
  Finished = 'finished',                 // 完成事件
  LoopDetected = 'loop_detected',        // 循环检测事件
}
```

### 核心数据结构

#### `ToolCallRequestInfo` - 工具调用请求信息

```typescript
export interface ToolCallRequestInfo {
  callId: string;                         // 唯一调用标识
  name: string;                          // 工具名称
  args: Record<string, unknown>;         // 工具参数
  isClientInitiated: boolean;            // 是否为客户端发起
  prompt_id: string;                     // 提示词ID
}
```

#### `ToolCallResponseInfo` - 工具调用响应信息

```typescript
export interface ToolCallResponseInfo {
  callId: string;                        // 对应的调用ID
  responseParts: PartListUnion;          // 响应内容部分
  resultDisplay: ToolResultDisplay | undefined; // 显示结果
  error: Error | undefined;              // 错误信息
}
```

#### `ThoughtSummary` - 思考摘要

```typescript
export type ThoughtSummary = {
  subject: string;                       // 思考主题
  description: string;                   // 思考描述
};
```

#### `ServerTool` - 服务器端工具接口

```typescript
export interface ServerTool {
  name: string;                          // 工具名称
  schema: FunctionDeclaration;           // 工具模式定义
  execute(                               // 执行方法
    params: Record<string, unknown>,
    signal?: AbortSignal,
  ): Promise<ToolResult>;
  shouldConfirmExecute(                  // 确认执行方法
    params: Record<string, unknown>,
    abortSignal: AbortSignal,
  ): Promise<ToolCallConfirmationDetails | false>;
}
```

### 事件类型定义

```typescript
// 所有可能的服务器事件类型
export type ServerGeminiStreamEvent =
  | ServerGeminiContentEvent           // 内容事件
  | ServerGeminiToolCallRequestEvent   // 工具调用请求事件
  | ServerGeminiToolCallResponseEvent  // 工具调用响应事件
  | ServerGeminiToolCallConfirmationEvent // 工具调用确认事件
  | ServerGeminiUserCancelledEvent     // 用户取消事件
  | ServerGeminiErrorEvent             // 错误事件
  | ServerGeminiChatCompressedEvent    // 聊天压缩事件
  | ServerGeminiThoughtEvent           // 思考事件
  | ServerGeminiMaxSessionTurnsEvent   // 最大回合事件
  | ServerGeminiFinishedEvent          // 完成事件
  | ServerGeminiLoopDetectedEvent;     // 循环检测事件
```

## Turn 类详解

### 构造函数

```typescript
constructor(
  private readonly chat: GeminiChat,
  private readonly prompt_id: string,
)
```

**参数**:
- `chat: GeminiChat` - Gemini 聊天实例，用于与 API 通信
- `prompt_id: string` - 当前提示词的唯一标识符

**初始化**:
```typescript
this.pendingToolCalls = [];    // 待处理的工具调用列表
this.debugResponses = [];      // 调试响应列表
```

### 核心方法

#### `run` - 执行对话回合

```typescript
async *run(
  req: PartListUnion,
  signal: AbortSignal,
): AsyncGenerator<ServerGeminiStreamEvent>
```

**功能**: 执行单个对话回合，处理用户请求并生成事件流

**参数**:
- `req: PartListUnion` - 用户输入请求内容
- `signal: AbortSignal` - 取消信号

**返回值**: `AsyncGenerator<ServerGeminiStreamEvent>` - 事件流生成器

**执行流程**:

```typescript
async *run(req: PartListUnion, signal: AbortSignal) {
  try {
    // 1. 发起流式对话请求
    const responseStream = await this.chat.sendMessageStream({
      message: req,
      config: { abortSignal: signal }
    }, this.prompt_id);
    
    // 2. 处理响应流
    for await (const resp of responseStream) {
      // 检查取消信号
      if (signal?.aborted) {
        yield { type: GeminiEventType.UserCancelled };
        return;
      }
      
      // 保存调试信息
      this.debugResponses.push(resp);
      
      // 3. 处理思考内容
      const thoughtPart = resp.candidates?.[0]?.content?.parts?.[0];
      if (thoughtPart?.thought) {
        yield { type: GeminiEventType.Thought, value: extractThought(thoughtPart) };
        continue;
      }
      
      // 4. 处理文本内容
      const text = getResponseText(resp);
      if (text) {
        yield { type: GeminiEventType.Content, value: text };
      }
      
      // 5. 处理工具调用
      const functionCalls = resp.functionCalls ?? [];
      for (const fnCall of functionCalls) {
        const event = this.handlePendingFunctionCall(fnCall);
        if (event) {
          yield event;
        }
      }
      
      // 6. 处理结束条件
      const finishReason = resp.candidates?.[0]?.finishReason;
      if (finishReason) {
        yield { type: GeminiEventType.Finished, value: finishReason };
      }
    }
  } catch (error) {
    // 错误处理和报告
    yield { type: GeminiEventType.Error, value: { error: processError(error) } };
  }
}
```

#### `handlePendingFunctionCall` - 处理待处理的函数调用

```typescript
private handlePendingFunctionCall(
  fnCall: FunctionCall,
): ServerGeminiStreamEvent | null
```

**功能**: 将 Gemini API 的函数调用转换为内部工具调用请求

**实现逻辑**:
```typescript
private handlePendingFunctionCall(fnCall: FunctionCall) {
  // 生成唯一调用ID
  const callId = fnCall.id ?? 
    `${fnCall.name}-${Date.now()}-${Math.random().toString(16).slice(2)}`;
  
  // 构建工具调用请求
  const toolCallRequest: ToolCallRequestInfo = {
    callId,
    name: fnCall.name || 'undefined_tool_name',
    args: (fnCall.args || {}) as Record<string, unknown>,
    isClientInitiated: false,  // AI 发起的调用
    prompt_id: this.prompt_id,
  };
  
  // 添加到待处理列表
  this.pendingToolCalls.push(toolCallRequest);
  
  // 返回工具调用请求事件
  return { 
    type: GeminiEventType.ToolCallRequest, 
    value: toolCallRequest 
  };
}
```

#### `getDebugResponses` - 获取调试响应

```typescript
getDebugResponses(): GenerateContentResponse[]
```

**功能**: 返回本回合中收集的所有调试响应信息

## 事件处理详解

### 1. 内容事件处理

```typescript
// 文本内容提取和处理
const text = getResponseText(resp);
if (text) {
  yield { 
    type: GeminiEventType.Content, 
    value: text 
  };
}
```

**特点**:
- 提取 API 响应中的文本内容
- 过滤空内容
- 直接传递给事件监听器

### 2. 思考事件处理

```typescript
// 思考内容识别和提取
const thoughtPart = resp.candidates?.[0]?.content?.parts?.[0];
if (thoughtPart?.thought) {
  // 解析思考格式：**主题** 描述内容
  const rawText = thoughtPart.text ?? '';
  const subjectStringMatches = rawText.match(/\*\*(.*?)\*\*/s);
  const subject = subjectStringMatches 
    ? subjectStringMatches[1].trim() 
    : '';
  const description = rawText.replace(/\*\*(.*?)\*\*/s, '').trim();
  
  const thought: ThoughtSummary = {
    subject,
    description,
  };
  
  yield {
    type: GeminiEventType.Thought,
    value: thought,
  };
}
```

**思考格式规范**:
- 主题用双星号包围：`**主题内容**`
- 主题后跟描述内容
- 支持多行描述

### 3. 工具调用事件处理

```typescript
// 工具调用请求处理
const functionCalls = resp.functionCalls ?? [];
for (const fnCall of functionCalls) {
  const event = this.handlePendingFunctionCall(fnCall);
  if (event) {
    yield event;
  }
}
```

**处理特点**:
- 支持批量工具调用
- 自动生成唯一调用ID
- 维护待处理调用列表

### 4. 错误事件处理

```typescript
// 完整的错误处理流程
catch (e) {
  const error = toFriendlyError(e);
  
  // 特殊错误类型处理
  if (error instanceof UnauthorizedError) {
    throw error; // 直接抛出认证错误
  }
  
  // 用户取消检查
  if (signal.aborted) {
    yield { type: GeminiEventType.UserCancelled };
    return;
  }
  
  // 错误报告和上下文收集
  const contextForReport = [...this.chat.getHistory(true), req];
  await reportError(
    error,
    'Error when talking to Gemini API',
    contextForReport,
    'Turn.run-sendMessageStream',
  );
  
  // 结构化错误信息
  const structuredError: StructuredError = {
    message: getErrorMessage(error),
    status: extractStatusCode(error),
  };
  
  yield { 
    type: GeminiEventType.Error, 
    value: { error: structuredError } 
  };
}
```

## 使用场景和示例

### 1. 基本对话回合执行

```typescript
import { Turn } from './turn.js';
import { GeminiChat } from './geminiChat.js';

async function executeConversationTurn() {
  const chat = new GeminiChat(/* config */);
  const turn = new Turn(chat, 'prompt_123');
  
  const userInput = "Help me write a Python function to calculate fibonacci numbers";
  const abortController = new AbortController();
  
  try {
    for await (const event of turn.run(userInput, abortController.signal)) {
      switch (event.type) {
        case GeminiEventType.Content:
          console.log('AI Response:', event.value);
          break;
          
        case GeminiEventType.ToolCallRequest:
          console.log('Tool Call Requested:', event.value.name);
          console.log('Arguments:', event.value.args);
          break;
          
        case GeminiEventType.Thought:
          console.log('AI Thinking:', event.value.subject);
          console.log('Details:', event.value.description);
          break;
          
        case GeminiEventType.Finished:
          console.log('Turn finished:', event.value);
          break;
          
        case GeminiEventType.Error:
          console.error('Error occurred:', event.value.error.message);
          break;
          
        case GeminiEventType.UserCancelled:
          console.log('User cancelled the operation');
          break;
      }
    }
  } catch (error) {
    console.error('Turn execution failed:', error);
  }
  
  // 获取调试信息
  const debugResponses = turn.getDebugResponses();
  console.log('Debug responses collected:', debugResponses.length);
}
```

### 2. 事件流处理器

```typescript
// 事件流处理器类
class TurnEventProcessor {
  private contentBuffer: string[] = [];
  private toolCalls: ToolCallRequestInfo[] = [];
  private thoughts: ThoughtSummary[] = [];
  private errors: StructuredError[] = [];
  
  async processTurn(
    turn: Turn,
    userInput: PartListUnion,
    signal: AbortSignal
  ): Promise<{
    content: string;
    toolCalls: ToolCallRequestInfo[];
    thoughts: ThoughtSummary[];
    errors: StructuredError[];
    completed: boolean;
  }> {
    let completed = false;
    
    try {
      for await (const event of turn.run(userInput, signal)) {
        await this.handleEvent(event);
        
        if (event.type === GeminiEventType.Finished) {
          completed = true;
        }
        
        if (event.type === GeminiEventType.Error || 
            event.type === GeminiEventType.UserCancelled) {
          break;
        }
      }
    } catch (error) {
      this.errors.push({
        message: error instanceof Error ? error.message : String(error)
      });
    }
    
    return {
      content: this.contentBuffer.join(''),
      toolCalls: [...this.toolCalls],
      thoughts: [...this.thoughts],
      errors: [...this.errors],
      completed
    };
  }
  
  private async handleEvent(event: ServerGeminiStreamEvent): Promise<void> {
    switch (event.type) {
      case GeminiEventType.Content:
        this.contentBuffer.push(event.value);
        break;
        
      case GeminiEventType.ToolCallRequest:
        this.toolCalls.push(event.value);
        console.log(`Tool call requested: ${event.value.name}`);
        break;
        
      case GeminiEventType.Thought:
        this.thoughts.push(event.value);
        console.log(`AI thinking about: ${event.value.subject}`);
        break;
        
      case GeminiEventType.Error:
        this.errors.push(event.value.error);
        console.error('Turn error:', event.value.error.message);
        break;
        
      case GeminiEventType.ChatCompressed:
        if (event.value) {
          console.log(`Chat compressed: ${event.value.originalTokenCount} -> ${event.value.newTokenCount} tokens`);
        }
        break;
        
      case GeminiEventType.MaxSessionTurns:
        console.warn('Maximum session turns reached');
        break;
        
      case GeminiEventType.LoopDetected:
        console.warn('Loop detected in conversation');
        break;
    }
  }
  
  reset(): void {
    this.contentBuffer = [];
    this.toolCalls = [];
    this.thoughts = [];
    this.errors = [];
  }
}

// 使用事件处理器
async function useEventProcessor() {
  const processor = new TurnEventProcessor();
  const chat = new GeminiChat(/* config */);
  const turn = new Turn(chat, 'session_456');
  
  const result = await processor.processTurn(
    turn,
    "Analyze this code and suggest improvements",
    new AbortController().signal
  );
  
  console.log('Final content:', result.content);
  console.log('Tool calls made:', result.toolCalls.length);
  console.log('Thoughts recorded:', result.thoughts.length);
  console.log('Errors encountered:', result.errors.length);
  console.log('Completed successfully:', result.completed);
}
```

### 3. 实时事件监听

```typescript
// 实时事件监听器
class RealTimeEventListener {
  private eventHandlers: Map<GeminiEventType, Function[]> = new Map();
  
  on(eventType: GeminiEventType, handler: Function): void {
    if (!this.eventHandlers.has(eventType)) {
      this.eventHandlers.set(eventType, []);
    }
    this.eventHandlers.get(eventType)!.push(handler);
  }
  
  off(eventType: GeminiEventType, handler: Function): void {
    const handlers = this.eventHandlers.get(eventType);
    if (handlers) {
      const index = handlers.indexOf(handler);
      if (index > -1) {
        handlers.splice(index, 1);
      }
    }
  }
  
  async listenToTurn(
    turn: Turn,
    userInput: PartListUnion,
    signal: AbortSignal
  ): Promise<void> {
    for await (const event of turn.run(userInput, signal)) {
      const handlers = this.eventHandlers.get(event.type) || [];
      
      for (const handler of handlers) {
        try {
          await handler(event.value);
        } catch (error) {
          console.error(`Event handler error for ${event.type}:`, error);
        }
      }
    }
  }
}

// 使用实时监听器
async function useRealTimeListener() {
  const listener = new RealTimeEventListener();
  
  // 注册事件处理器
  listener.on(GeminiEventType.Content, (content: string) => {
    process.stdout.write(content); // 实时输出内容
  });
  
  listener.on(GeminiEventType.ToolCallRequest, (request: ToolCallRequestInfo) => {
    console.log(`\n[Tool Call] ${request.name}(${JSON.stringify(request.args)})`);
  });
  
  listener.on(GeminiEventType.Thought, (thought: ThoughtSummary) => {
    console.log(`\n[Thinking] ${thought.subject}: ${thought.description}`);
  });
  
  listener.on(GeminiEventType.Error, (error: StructuredError) => {
    console.error(`\n[Error] ${error.message}`);
  });
  
  // 开始监听
  const chat = new GeminiChat(/* config */);
  const turn = new Turn(chat, 'realtime_789');
  
  await listener.listenToTurn(
    turn,
    "Create a web scraper for news articles",
    new AbortController().signal
  );
}
```

### 4. 批量回合处理

```typescript
// 批量回合处理器
class BatchTurnProcessor {
  constructor(
    private chat: GeminiChat,
    private maxConcurrent: number = 3
  ) {}
  
  async processBatch(
    requests: Array<{
      input: PartListUnion;
      promptId: string;
      metadata?: any;
    }>,
    signal: AbortSignal
  ): Promise<Array<{
    input: PartListUnion;
    result: TurnResult;
    metadata?: any;
    error?: Error;
  }>> {
    const results: Array<any> = [];
    const semaphore = new Array(this.maxConcurrent).fill(null);
    
    const processRequest = async (request: any, index: number) => {
      try {
        const turn = new Turn(this.chat, request.promptId);
        const result = await this.processSingleTurn(
          turn,
          request.input,
          signal
        );
        
        results[index] = {
          input: request.input,
          result,
          metadata: request.metadata
        };
      } catch (error) {
        results[index] = {
          input: request.input,
          result: null,
          metadata: request.metadata,
          error: error instanceof Error ? error : new Error(String(error))
        };
      }
    };
    
    // 并发处理请求
    const promises: Promise<void>[] = [];
    for (let i = 0; i < requests.length; i++) {
      promises.push(processRequest(requests[i], i));
      
      // 限制并发数
      if (promises.length >= this.maxConcurrent) {
        await Promise.race(promises);
        promises.splice(0, 1);
      }
    }
    
    // 等待所有请求完成
    await Promise.all(promises);
    
    return results;
  }
  
  private async processSingleTurn(
    turn: Turn,
    input: PartListUnion,
    signal: AbortSignal
  ): Promise<TurnResult> {
    const content: string[] = [];
    const toolCalls: ToolCallRequestInfo[] = [];
    const thoughts: ThoughtSummary[] = [];
    const errors: StructuredError[] = [];
    let finished = false;
    
    for await (const event of turn.run(input, signal)) {
      switch (event.type) {
        case GeminiEventType.Content:
          content.push(event.value);
          break;
        case GeminiEventType.ToolCallRequest:
          toolCalls.push(event.value);
          break;
        case GeminiEventType.Thought:
          thoughts.push(event.value);
          break;
        case GeminiEventType.Error:
          errors.push(event.value.error);
          break;
        case GeminiEventType.Finished:
          finished = true;
          break;
      }
    }
    
    return {
      content: content.join(''),
      toolCalls,
      thoughts,
      errors,
      finished,
      debugResponses: turn.getDebugResponses()
    };
  }
}

interface TurnResult {
  content: string;
  toolCalls: ToolCallRequestInfo[];
  thoughts: ThoughtSummary[];
  errors: StructuredError[];
  finished: boolean;
  debugResponses: any[];
}

// 使用批量处理器
async function useBatchProcessor() {
  const chat = new GeminiChat(/* config */);
  const processor = new BatchTurnProcessor(chat, 5);
  
  const requests = [
    {
      input: "Explain quantum computing",
      promptId: "batch_1",
      metadata: { topic: "physics" }
    },
    {
      input: "Write a sorting algorithm",
      promptId: "batch_2", 
      metadata: { topic: "algorithms" }
    },
    {
      input: "Create a REST API design",
      promptId: "batch_3",
      metadata: { topic: "architecture" }
    }
  ];
  
  const results = await processor.processBatch(
    requests,
    new AbortController().signal
  );
  
  results.forEach((result, index) => {
    console.log(`\n=== Request ${index + 1} ===`);
    console.log('Topic:', result.metadata?.topic);
    
    if (result.error) {
      console.error('Error:', result.error.message);
    } else {
      console.log('Content length:', result.result.content.length);
      console.log('Tool calls:', result.result.toolCalls.length);
      console.log('Thoughts:', result.result.thoughts.length);
      console.log('Completed:', result.result.finished);
    }
  });
}
```

## 错误处理机制

### 1. 错误分类和处理

```typescript
// 错误处理策略
class TurnErrorHandler {
  static async handleTurnError(
    error: unknown,
    context: {
      turn: Turn;
      input: PartListUnion;
      signal: AbortSignal;
    }
  ): Promise<ServerGeminiStreamEvent> {
    // 转换为友好错误
    const friendlyError = toFriendlyError(error);
    
    // 认证错误直接抛出
    if (friendlyError instanceof UnauthorizedError) {
      throw friendlyError;
    }
    
    // 用户取消检查
    if (context.signal.aborted) {
      return { type: GeminiEventType.UserCancelled };
    }
    
    // 收集错误上下文
    const errorContext = {
      input: context.input,
      history: context.turn.chat?.getHistory?.(true) || [],
      debugResponses: context.turn.getDebugResponses(),
      timestamp: new Date().toISOString(),
      userAgent: navigator?.userAgent || 'unknown'
    };
    
    // 报告错误
    await reportError(
      friendlyError,
      'Turn execution error',
      errorContext,
      'TurnErrorHandler.handleTurnError'
    );
    
    // 返回结构化错误事件
    return {
      type: GeminiEventType.Error,
      value: {
        error: {
          message: getErrorMessage(friendlyError),
          status: extractStatusCode(friendlyError),
        }
      }
    };
  }
  
  static isRetryableError(error: unknown): boolean {
    const message = getErrorMessage(error);
    const retryableMessages = [
      'network error',
      'timeout',
      'rate limit',
      'service unavailable',
      'temporary failure'
    ];
    
    return retryableMessages.some(retryableMsg => 
      message.toLowerCase().includes(retryableMsg)
    );
  }
  
  static shouldReportError(error: unknown): boolean {
    // 不报告用户取消和认证错误
    return !(error instanceof UnauthorizedError) && 
           !getErrorMessage(error).includes('aborted');
  }
}
```

### 2. 错误恢复策略

```typescript
// 错误恢复机制
class TurnRecoveryManager {
  constructor(
    private maxRetries: number = 3,
    private retryDelayMs: number = 1000
  ) {}
  
  async executeWithRecovery(
    turn: Turn,
    input: PartListUnion,
    signal: AbortSignal
  ): Promise<AsyncGenerator<ServerGeminiStreamEvent>> {
    let attempt = 0;
    let lastError: unknown;
    
    while (attempt < this.maxRetries) {
      try {
        return turn.run(input, signal);
      } catch (error) {
        lastError = error;
        attempt++;
        
        // 检查是否可重试
        if (!TurnErrorHandler.isRetryableError(error) || 
            attempt >= this.maxRetries ||
            signal.aborted) {
          throw error;
        }
        
        // 等待后重试
        const delay = this.retryDelayMs * Math.pow(2, attempt - 1);
        console.warn(`Turn failed (attempt ${attempt}), retrying in ${delay}ms...`);
        await new Promise(resolve => setTimeout(resolve, delay));
      }
    }
    
    throw lastError;
  }
  
  async *executeWithFallback(
    primaryTurn: Turn,
    fallbackTurn: Turn,
    input: PartListUnion,
    signal: AbortSignal
  ): AsyncGenerator<ServerGeminiStreamEvent> {
    try {
      for await (const event of primaryTurn.run(input, signal)) {
        yield event;
      }
    } catch (error) {
      console.warn('Primary turn failed, switching to fallback:', error);
      
      yield {
        type: GeminiEventType.Content,
        value: '\n[Switching to fallback model...]\n'
      };
      
      for await (const event of fallbackTurn.run(input, signal)) {
        yield event;
      }
    }
  }
}
```

## 性能优化

### 1. 内存管理

```typescript
// 内存优化的 Turn 扩展
class OptimizedTurn extends Turn {
  private maxDebugResponses = 100;
  private responseBuffer: GenerateContentResponse[] = [];
  
  protected addDebugResponse(response: GenerateContentResponse): void {
    this.responseBuffer.push(response);
    
    // 限制调试响应数量
    if (this.responseBuffer.length > this.maxDebugResponses) {
      this.responseBuffer.shift();
    }
  }
  
  getDebugResponses(): GenerateContentResponse[] {
    return [...this.responseBuffer]; // 返回副本
  }
  
  cleanup(): void {
    this.responseBuffer = [];
    this.pendingToolCalls.length = 0;
  }
}
```

### 2. 事件流优化

```typescript
// 事件流缓冲优化
class BufferedTurnProcessor {
  constructor(
    private bufferSize: number = 10,
    private flushIntervalMs: number = 100
  ) {}
  
  async *processWithBuffering(
    turn: Turn,
    input: PartListUnion,
    signal: AbortSignal
  ): AsyncGenerator<ServerGeminiStreamEvent[]> {
    const buffer: ServerGeminiStreamEvent[] = [];
    let lastFlush = Date.now();
    
    for await (const event of turn.run(input, signal)) {
      buffer.push(event);
      
      const shouldFlush = 
        buffer.length >= this.bufferSize ||
        Date.now() - lastFlush >= this.flushIntervalMs ||
        event.type === GeminiEventType.Finished ||
        event.type === GeminiEventType.Error;
      
      if (shouldFlush) {
        yield [...buffer];
        buffer.length = 0;
        lastFlush = Date.now();
      }
    }
    
    // 刷新剩余事件
    if (buffer.length > 0) {
      yield buffer;
    }
  }
}
```

## 测试示例

### 单元测试

```typescript
describe('Turn', () => {
  let mockChat: jest.Mocked<GeminiChat>;
  let turn: Turn;
  
  beforeEach(() => {
    mockChat = {
      sendMessageStream: jest.fn(),
      getHistory: jest.fn().mockReturnValue([])
    } as any;
    
    turn = new Turn(mockChat, 'test-prompt');
  });
  
  describe('run', () => {
    it('should yield content events', async () => {
      const mockResponse = {
        candidates: [{
          content: { parts: [{ text: 'Test response' }] }
        }]
      };
      
      mockChat.sendMessageStream.mockImplementation(async function* () {
        yield mockResponse;
      });
      
      const events: ServerGeminiStreamEvent[] = [];
      for await (const event of turn.run('test input', new AbortController().signal)) {
        events.push(event);
      }
      
      expect(events).toHaveLength(1);
      expect(events[0].type).toBe(GeminiEventType.Content);
      expect(events[0].value).toBe('Test response');
    });
    
    it('should handle tool call requests', async () => {
      const mockResponse = {
        functionCalls: [{
          id: 'call-123',
          name: 'test-tool',
          args: { param: 'value' }
        }]
      };
      
      mockChat.sendMessageStream.mockImplementation(async function* () {
        yield mockResponse;
      });
      
      const events: ServerGeminiStreamEvent[] = [];
      for await (const event of turn.run('test input', new AbortController().signal)) {
        events.push(event);
      }
      
      expect(events).toHaveLength(1);
      expect(events[0].type).toBe(GeminiEventType.ToolCallRequest);
      expect(events[0].value.name).toBe('test-tool');
      expect(events[0].value.callId).toBe('call-123');
    });
    
    it('should handle thought events', async () => {
      const mockResponse = {
        candidates: [{
          content: {
            parts: [{
              thought: true,
              text: '**Planning** I need to create a function'
            }]
          }
        }]
      };
      
      mockChat.sendMessageStream.mockImplementation(async function* () {
        yield mockResponse;
      });
      
      const events: ServerGeminiStreamEvent[] = [];
      for await (const event of turn.run('test input', new AbortController().signal)) {
        events.push(event);
      }
      
      expect(events).toHaveLength(1);
      expect(events[0].type).toBe(GeminiEventType.Thought);
      expect(events[0].value.subject).toBe('Planning');
      expect(events[0].value.description).toBe('I need to create a function');
    });
    
    it('should handle errors gracefully', async () => {
      const testError = new Error('API Error');
      mockChat.sendMessageStream.mockRejectedValue(testError);
      
      const events: ServerGeminiStreamEvent[] = [];
      for await (const event of turn.run('test input', new AbortController().signal)) {
        events.push(event);
      }
      
      expect(events).toHaveLength(1);
      expect(events[0].type).toBe(GeminiEventType.Error);
      expect(events[0].value.error.message).toContain('API Error');
    });
    
    it('should handle user cancellation', async () => {
      const abortController = new AbortController();
      
      mockChat.sendMessageStream.mockImplementation(async function* () {
        // 模拟异步操作中的取消
        yield new Promise(resolve => setTimeout(resolve, 100));
      });
      
      // 立即取消
      abortController.abort();
      
      const events: ServerGeminiStreamEvent[] = [];
      for await (const event of turn.run('test input', abortController.signal)) {
        events.push(event);
      }
      
      expect(events).toHaveLength(1);
      expect(events[0].type).toBe(GeminiEventType.UserCancelled);
    });
  });
  
  describe('handlePendingFunctionCall', () => {
    it('should generate unique call IDs', () => {
      const fnCall1 = { name: 'tool1', args: {} };
      const fnCall2 = { name: 'tool2', args: {} };
      
      const event1 = turn['handlePendingFunctionCall'](fnCall1);
      const event2 = turn['handlePendingFunctionCall'](fnCall2);
      
      expect(event1?.value.callId).toBeDefined();
      expect(event2?.value.callId).toBeDefined();
      expect(event1?.value.callId).not.toBe(event2?.value.callId);
    });
    
    it('should use provided call ID when available', () => {
      const fnCall = { 
        id: 'custom-id',
        name: 'tool1', 
        args: {} 
      };
      
      const event = turn['handlePendingFunctionCall'](fnCall);
      
      expect(event?.value.callId).toBe('custom-id');
    });
  });
  
  describe('getDebugResponses', () => {
    it('should return collected debug responses', async () => {
      const mockResponse1 = { candidates: [{ content: { parts: [{ text: 'Response 1' }] } }] };
      const mockResponse2 = { candidates: [{ content: { parts: [{ text: 'Response 2' }] } }] };
      
      mockChat.sendMessageStream.mockImplementation(async function* () {
        yield mockResponse1;
        yield mockResponse2;
      });
      
      // 执行 turn
      const events: ServerGeminiStreamEvent[] = [];
      for await (const event of turn.run('test input', new AbortController().signal)) {
        events.push(event);
      }
      
      const debugResponses = turn.getDebugResponses();
      expect(debugResponses).toHaveLength(2);
      expect(debugResponses[0]).toBe(mockResponse1);
      expect(debugResponses[1]).toBe(mockResponse2);
    });
  });
});
```

### 集成测试

```typescript
describe('Turn Integration', () => {
  it('should work with real GeminiChat', async () => {
    // 需要真实的 API 密钥进行集成测试
    const chat = new GeminiChat(/* real config */);
    const turn = new Turn(chat, 'integration-test');
    
    const events: ServerGeminiStreamEvent[] = [];
    
    try {
      for await (const event of turn.run(
        'What is the capital of France?',
        new AbortController().signal
      )) {
        events.push(event);
        
        if (event.type === GeminiEventType.Finished) {
          break;
        }
      }
    } catch (error) {
      // 集成测试可能因为网络或API限制失败
      console.warn('Integration test failed:', error);
      return;
    }
    
    expect(events.length).toBeGreaterThan(0);
    expect(events.some(e => e.type === GeminiEventType.Content)).toBe(true);
  });
});
```

## 最佳实践

### 1. 事件处理
- 始终处理所有事件类型
- 使用适当的缓冲策略处理高频事件
- 实现优雅的错误恢复机制

### 2. 内存管理
- 限制调试响应的数量
- 及时清理不再需要的数据
- 避免在事件处理器中创建大量对象

### 3. 错误处理
- 区分可重试和不可重试的错误
- 提供详细的错误上下文信息
- 实现适当的错误报告机制

### 4. 性能优化
- 使用事件流缓冲减少I/O开销
- 实现并发控制避免资源过载
- 监控内存使用和事件处理延迟

该模块是 Gemini CLI 对话系统的核心，提供了完整的事件驱动架构，支持复杂的 AI 交互场景和工具调用编排。