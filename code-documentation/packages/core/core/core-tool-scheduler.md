# Core Tool Scheduler - 核心工具调度器文档

## 概述

`CoreToolScheduler` 是 Gemini CLI 的核心工具调度和执行管理器，负责协调工具调用的完整生命周期。它管理工具调用的验证、确认、执行、状态跟踪和结果处理，支持用户确认流程、实时输出更新、错误处理和取消操作。

## 主要功能

- **工具调用调度**: 管理多个工具调用的并发执行和状态协调
- **用户确认流程**: 支持工具执行前的用户确认和批准机制  
- **状态生命周期管理**: 跟踪工具调用从验证到完成的完整状态变化
- **实时输出更新**: 支持工具执行过程中的实时输出流更新
- **错误处理和恢复**: 完善的错误捕获、报告和恢复机制
- **编辑器集成**: 支持外部编辑器修改工具参数的高级功能
- **并发控制**: 防止冲突的工具调用同时执行

## 核心类型定义

### 工具调用状态类型

```typescript
// 验证中状态
export type ValidatingToolCall = {
  status: 'validating';
  request: ToolCallRequestInfo;
  tool: Tool;
  startTime?: number;
  outcome?: ToolConfirmationOutcome;
};

// 已调度状态
export type ScheduledToolCall = {
  status: 'scheduled';
  request: ToolCallRequestInfo;
  tool: Tool;
  startTime?: number;
  outcome?: ToolConfirmationOutcome;
};

// 执行中状态
export type ExecutingToolCall = {
  status: 'executing';
  request: ToolCallRequestInfo;
  tool: Tool;
  liveOutput?: string;
  startTime?: number;
  outcome?: ToolConfirmationOutcome;
};

// 等待批准状态
export type WaitingToolCall = {
  status: 'awaiting_approval';
  request: ToolCallRequestInfo;
  tool: Tool;
  confirmationDetails: ToolCallConfirmationDetails;
  startTime?: number;
  outcome?: ToolConfirmationOutcome;
};

// 成功完成状态
export type SuccessfulToolCall = {
  status: 'success';
  request: ToolCallRequestInfo;
  tool: Tool;
  response: ToolCallResponseInfo;
  durationMs?: number;
  outcome?: ToolConfirmationOutcome;
};

// 错误状态
export type ErroredToolCall = {
  status: 'error';
  request: ToolCallRequestInfo;
  response: ToolCallResponseInfo;
  durationMs?: number;
  outcome?: ToolConfirmationOutcome;
};

// 取消状态
export type CancelledToolCall = {
  status: 'cancelled';
  request: ToolCallRequestInfo;
  response: ToolCallResponseInfo;
  tool: Tool;
  durationMs?: number;
  outcome?: ToolConfirmationOutcome;
};

// 联合类型
export type ToolCall =
  | ValidatingToolCall
  | ScheduledToolCall
  | ErroredToolCall
  | SuccessfulToolCall
  | ExecutingToolCall
  | CancelledToolCall
  | WaitingToolCall;

export type CompletedToolCall =
  | SuccessfulToolCall
  | CancelledToolCall
  | ErroredToolCall;
```

### 回调函数类型

```typescript
// 确认处理器
export type ConfirmHandler = (
  toolCall: WaitingToolCall,
) => Promise<ToolConfirmationOutcome>;

// 输出更新处理器
export type OutputUpdateHandler = (
  toolCallId: string,
  outputChunk: string,
) => void;

// 所有工具调用完成处理器
export type AllToolCallsCompleteHandler = (
  completedToolCalls: CompletedToolCall[],
) => void;

// 工具调用更新处理器
export type ToolCallsUpdateHandler = (toolCalls: ToolCall[]) => void;
```

## 核心类实现

### `CoreToolScheduler` 类

```typescript
export class CoreToolScheduler {
  private toolRegistry: Promise<ToolRegistry>;
  private toolCalls: ToolCall[] = [];
  private outputUpdateHandler?: OutputUpdateHandler;
  private onAllToolCallsComplete?: AllToolCallsCompleteHandler;
  private onToolCallsUpdate?: ToolCallsUpdateHandler;
  private getPreferredEditor: () => EditorType | undefined;
  private config: Config;

  constructor(options: CoreToolSchedulerOptions);
}
```

**构造函数选项**:
```typescript
interface CoreToolSchedulerOptions {
  toolRegistry: Promise<ToolRegistry>;
  outputUpdateHandler?: OutputUpdateHandler;
  onAllToolCallsComplete?: AllToolCallsCompleteHandler;
  onToolCallsUpdate?: ToolCallsUpdateHandler;
  getPreferredEditor: () => EditorType | undefined;
  config: Config;
}
```

## 主要方法

### 工具调用调度

#### `schedule` - 调度工具调用

```typescript
async schedule(
  request: ToolCallRequestInfo | ToolCallRequestInfo[],
  signal: AbortSignal,
): Promise<void>
```

**功能**: 调度一个或多个工具调用执行

**执行流程**:

1. **并发检查**: 确保没有正在运行的工具调用
2. **工具验证**: 验证工具是否在注册表中存在
3. **创建工具调用**: 初始化工具调用状态为 `validating`
4. **确认检查**: 根据批准模式决定是否需要用户确认
5. **状态更新**: 更新工具调用状态并通知监听器

**实现逻辑**:
```typescript
async schedule(
  request: ToolCallRequestInfo | ToolCallRequestInfo[],
  signal: AbortSignal,
): Promise<void> {
  // 1. 检查是否有正在运行的工具调用
  if (this.isRunning()) {
    throw new Error(
      'Cannot schedule new tool calls while other tool calls are actively running'
    );
  }

  const requestsToProcess = Array.isArray(request) ? request : [request];
  const toolRegistry = await this.toolRegistry;

  // 2. 创建新的工具调用
  const newToolCalls: ToolCall[] = requestsToProcess.map((reqInfo): ToolCall => {
    const toolInstance = toolRegistry.getTool(reqInfo.name);
    if (!toolInstance) {
      return {
        status: 'error',
        request: reqInfo,
        response: createErrorResponse(
          reqInfo,
          new Error(`Tool "${reqInfo.name}" not found in registry.`),
        ),
        durationMs: 0,
      };
    }
    return {
      status: 'validating',
      request: reqInfo,
      tool: toolInstance,
      startTime: Date.now(),
    };
  });

  this.toolCalls = this.toolCalls.concat(newToolCalls);
  this.notifyToolCallsUpdate();

  // 3. 处理每个工具调用
  for (const toolCall of newToolCalls) {
    if (toolCall.status !== 'validating') continue;

    const { request: reqInfo, tool: toolInstance } = toolCall;
    try {
      // 检查批准模式
      if (this.config.getApprovalMode() === ApprovalMode.YOLO) {
        this.setStatusInternal(reqInfo.callId, 'scheduled');
      } else {
        // 获取确认详情
        const confirmationDetails = await toolInstance.shouldConfirmExecute(
          reqInfo.args,
          signal,
        );

        if (confirmationDetails) {
          // 需要用户确认
          const wrappedConfirmationDetails: ToolCallConfirmationDetails = {
            ...confirmationDetails,
            onConfirm: (outcome, payload) =>
              this.handleConfirmationResponse(
                reqInfo.callId,
                confirmationDetails.onConfirm,
                outcome,
                signal,
                payload,
              ),
          };
          this.setStatusInternal(
            reqInfo.callId,
            'awaiting_approval',
            wrappedConfirmationDetails,
          );
        } else {
          // 无需确认，直接调度
          this.setStatusInternal(reqInfo.callId, 'scheduled');
        }
      }
    } catch (error) {
      this.setStatusInternal(
        reqInfo.callId,
        'error',
        createErrorResponse(reqInfo, error),
      );
    }
  }

  // 4. 尝试执行已调度的工具调用
  this.attemptExecutionOfScheduledCalls(signal);
  this.checkAndNotifyCompletion();
}
```

**使用示例**:
```typescript
const scheduler = new CoreToolScheduler({
  toolRegistry: Promise.resolve(toolRegistry),
  config,
  outputUpdateHandler: (callId, output) => {
    console.log(`Tool ${callId}: ${output}`);
  },
  onAllToolCallsComplete: (completedCalls) => {
    console.log(`Completed ${completedCalls.length} tool calls`);
  },
  getPreferredEditor: () => EditorType.VSCode,
});

// 调度单个工具调用
await scheduler.schedule({
  callId: 'call-1',
  name: 'read_file',
  args: { file_path: '/path/to/file.txt' }
}, signal);

// 调度多个工具调用
await scheduler.schedule([
  {
    callId: 'call-2',
    name: 'grep',
    args: { pattern: 'function', path: 'src/' }
  },
  {
    callId: 'call-3', 
    name: 'edit',
    args: { file_path: '/path/to/edit.js', new_content: 'updated content' }
  }
], signal);
```

### 确认处理

#### `handleConfirmationResponse` - 处理用户确认响应

```typescript
async handleConfirmationResponse(
  callId: string,
  originalOnConfirm: (outcome: ToolConfirmationOutcome) => Promise<void>,
  outcome: ToolConfirmationOutcome,
  signal: AbortSignal,
  payload?: ToolConfirmationPayload,
): Promise<void>
```

**功能**: 处理用户对工具调用的确认响应

**确认结果处理**:

1. **取消 (`Cancel`)**: 将工具调用标记为已取消
2. **批准 (`ProceedOnce/ProceedAlways`)**: 调度工具执行
3. **编辑器修改 (`ModifyWithEditor`)**: 打开外部编辑器修改参数
4. **内联修改**: 应用用户提供的新内容

**实现逻辑**:
```typescript
async handleConfirmationResponse(
  callId: string,
  originalOnConfirm: (outcome: ToolConfirmationOutcome) => Promise<void>,
  outcome: ToolConfirmationOutcome,
  signal: AbortSignal,
  payload?: ToolConfirmationPayload,
): Promise<void> {
  const toolCall = this.toolCalls.find(
    (c) => c.request.callId === callId && c.status === 'awaiting_approval',
  );

  if (toolCall && toolCall.status === 'awaiting_approval') {
    await originalOnConfirm(outcome);
  }

  // 更新工具调用的确认结果
  this.toolCalls = this.toolCalls.map((call) => {
    if (call.request.callId !== callId) return call;
    return { ...call, outcome };
  });

  if (outcome === ToolConfirmationOutcome.Cancel || signal.aborted) {
    // 用户取消
    this.setStatusInternal(callId, 'cancelled', 'User did not allow tool call');
  } else if (outcome === ToolConfirmationOutcome.ModifyWithEditor) {
    // 编辑器修改
    const waitingToolCall = toolCall as WaitingToolCall;
    if (isModifiableTool(waitingToolCall.tool)) {
      const modifyContext = waitingToolCall.tool.getModifyContext(signal);
      const editorType = this.getPreferredEditor();
      
      if (!editorType) return;

      // 标记为修改中
      this.setStatusInternal(callId, 'awaiting_approval', {
        ...waitingToolCall.confirmationDetails,
        isModifying: true,
      });

      // 使用编辑器修改
      const { updatedParams, updatedDiff } = await modifyWithEditor(
        waitingToolCall.request.args,
        modifyContext,
        editorType,
        signal,
      );

      // 更新参数和差异
      this.setArgsInternal(callId, updatedParams);
      this.setStatusInternal(callId, 'awaiting_approval', {
        ...waitingToolCall.confirmationDetails,
        fileDiff: updatedDiff,
        isModifying: false,
      });
    }
  } else {
    // 批准执行
    if (payload?.newContent && toolCall) {
      await this._applyInlineModify(toolCall as WaitingToolCall, payload, signal);
    }
    this.setStatusInternal(callId, 'scheduled');
  }

  this.attemptExecutionOfScheduledCalls(signal);
}
```

### 工具执行

#### `attemptExecutionOfScheduledCalls` - 执行已调度的工具调用

```typescript
private attemptExecutionOfScheduledCalls(signal: AbortSignal): void
```

**功能**: 执行所有状态为 `scheduled` 的工具调用

**执行条件**: 所有工具调用都处于最终状态或已调度状态时才开始执行

**执行流程**:
```typescript
private attemptExecutionOfScheduledCalls(signal: AbortSignal): void {
  // 检查是否所有调用都是最终状态或已调度
  const allCallsFinalOrScheduled = this.toolCalls.every(
    (call) =>
      call.status === 'scheduled' ||
      call.status === 'cancelled' ||
      call.status === 'success' ||
      call.status === 'error',
  );

  if (allCallsFinalOrScheduled) {
    const callsToExecute = this.toolCalls.filter(
      (call) => call.status === 'scheduled',
    );

    callsToExecute.forEach((toolCall) => {
      if (toolCall.status !== 'scheduled') return;

      const scheduledCall = toolCall;
      const { callId, name: toolName } = scheduledCall.request;
      
      // 更新状态为执行中
      this.setStatusInternal(callId, 'executing');

      // 设置实时输出回调
      const liveOutputCallback =
        scheduledCall.tool.canUpdateOutput && this.outputUpdateHandler
          ? (outputChunk: string) => {
              if (this.outputUpdateHandler) {
                this.outputUpdateHandler(callId, outputChunk);
              }
              // 更新工具调用的实时输出
              this.toolCalls = this.toolCalls.map((tc) =>
                tc.request.callId === callId && tc.status === 'executing'
                  ? { ...tc, liveOutput: outputChunk }
                  : tc,
              );
              this.notifyToolCallsUpdate();
            }
          : undefined;

      // 执行工具
      scheduledCall.tool
        .execute(scheduledCall.request.args, signal, liveOutputCallback)
        .then(async (toolResult: ToolResult) => {
          if (signal.aborted) {
            this.setStatusInternal(callId, 'cancelled', 'User cancelled tool execution.');
            return;
          }

          // 转换为函数响应格式
          const response = convertToFunctionResponse(
            toolName,
            callId,
            toolResult.llmContent,
          );
          
          const successResponse: ToolCallResponseInfo = {
            callId,
            responseParts: response,
            resultDisplay: toolResult.returnDisplay,
            error: undefined,
          };

          this.setStatusInternal(callId, 'success', successResponse);
        })
        .catch((executionError: Error) => {
          this.setStatusInternal(
            callId,
            'error',
            createErrorResponse(scheduledCall.request, executionError),
          );
        });
    });
  }
}
```

## 状态管理

### 状态生命周期

```
[初始化] → validating → awaiting_approval → scheduled → executing → success/error/cancelled
                  ↓                ↓
               error          cancelled
```

**状态转换说明**:
- `validating`: 验证工具是否存在和参数是否有效
- `awaiting_approval`: 等待用户确认工具执行
- `scheduled`: 已调度等待执行
- `executing`: 正在执行中
- `success`: 执行成功完成
- `error`: 执行过程中发生错误
- `cancelled`: 用户取消或系统中止

### 状态更新方法

```typescript
private setStatusInternal(
  targetCallId: string,
  newStatus: Status,
  auxiliaryData?: unknown,
): void {
  this.toolCalls = this.toolCalls.map((currentCall) => {
    if (
      currentCall.request.callId !== targetCallId ||
      currentCall.status === 'success' ||
      currentCall.status === 'error' ||
      currentCall.status === 'cancelled'
    ) {
      return currentCall;
    }

    const existingStartTime = currentCall.startTime;
    const toolInstance = currentCall.tool;
    const outcome = currentCall.outcome;

    switch (newStatus) {
      case 'success': {
        const durationMs = existingStartTime
          ? Date.now() - existingStartTime
          : undefined;
        return {
          request: currentCall.request,
          tool: toolInstance,
          status: 'success',
          response: auxiliaryData as ToolCallResponseInfo,
          durationMs,
          outcome,
        } as SuccessfulToolCall;
      }
      case 'error': {
        const durationMs = existingStartTime
          ? Date.now() - existingStartTime
          : undefined;
        return {
          request: currentCall.request,
          status: 'error',
          response: auxiliaryData as ToolCallResponseInfo,
          durationMs,
          outcome,
        } as ErroredToolCall;
      }
      // ... 其他状态处理
    }
  });
  
  this.notifyToolCallsUpdate();
  this.checkAndNotifyCompletion();
}
```

## 响应格式转换

### `convertToFunctionResponse` - 转换为函数响应格式

```typescript
export function convertToFunctionResponse(
  toolName: string,
  callId: string,
  llmContent: PartListUnion,
): PartListUnion
```

**功能**: 将工具执行结果转换为 Gemini API 的函数响应格式

**转换规则**:
```typescript
export function convertToFunctionResponse(
  toolName: string,
  callId: string,
  llmContent: PartListUnion,
): PartListUnion {
  const contentToProcess =
    Array.isArray(llmContent) && llmContent.length === 1
      ? llmContent[0]
      : llmContent;

  // 字符串内容
  if (typeof contentToProcess === 'string') {
    return createFunctionResponsePart(callId, toolName, contentToProcess);
  }

  // 数组内容
  if (Array.isArray(contentToProcess)) {
    const functionResponse = createFunctionResponsePart(
      callId,
      toolName,
      'Tool execution succeeded.',
    );
    return [functionResponse, ...contentToProcess];
  }

  // 单个 Part 对象
  if (contentToProcess.functionResponse) {
    if (contentToProcess.functionResponse.response?.content) {
      const stringifiedOutput =
        getResponseTextFromParts(
          contentToProcess.functionResponse.response.content as Part[],
        ) || '';
      return createFunctionResponsePart(callId, toolName, stringifiedOutput);
    }
    return contentToProcess;
  }

  // 二进制数据
  if (contentToProcess.inlineData || contentToProcess.fileData) {
    const mimeType =
      contentToProcess.inlineData?.mimeType ||
      contentToProcess.fileData?.mimeType ||
      'unknown';
    const functionResponse = createFunctionResponsePart(
      callId,
      toolName,
      `Binary content of type ${mimeType} was processed.`,
    );
    return [functionResponse, contentToProcess];
  }

  // 文本内容
  if (contentToProcess.text !== undefined) {
    return createFunctionResponsePart(callId, toolName, contentToProcess.text);
  }

  // 默认情况
  return createFunctionResponsePart(
    callId,
    toolName,
    'Tool execution succeeded.',
  );
}

function createFunctionResponsePart(
  callId: string,
  toolName: string,
  output: string,
): Part {
  return {
    functionResponse: {
      id: callId,
      name: toolName,
      response: { output },
    },
  };
}
```

## 使用示例

### 基本工具调度

```typescript
import { CoreToolScheduler } from './coreToolScheduler.js';
import { ToolRegistry } from '../tools/tool-registry.js';
import { Config } from '../config/config.js';

// 基本使用示例
async function basicToolSchedulingExample() {
  const config = new Config({
    approvalMode: ApprovalMode.AUTO, // 自动批准模式
  });
  
  const toolRegistry = new ToolRegistry();
  // 注册一些工具...

  const scheduler = new CoreToolScheduler({
    toolRegistry: Promise.resolve(toolRegistry),
    config,
    outputUpdateHandler: (callId, outputChunk) => {
      console.log(`[${callId}] Output: ${outputChunk}`);
    },
    onAllToolCallsComplete: (completedCalls) => {
      console.log(`\nAll ${completedCalls.length} tool calls completed:`);
      completedCalls.forEach(call => {
        console.log(`- ${call.request.name}: ${call.status}`);
        if (call.status === 'success') {
          console.log(`  Duration: ${call.durationMs}ms`);
        }
      });
    },
    onToolCallsUpdate: (toolCalls) => {
      const statusCounts = toolCalls.reduce((acc, call) => {
        acc[call.status] = (acc[call.status] || 0) + 1;
        return acc;
      }, {} as Record<string, number>);
      console.log('Status update:', statusCounts);
    },
    getPreferredEditor: () => EditorType.VSCode,
  });

  const signal = new AbortController().signal;

  try {
    // 调度多个工具调用
    await scheduler.schedule([
      {
        callId: 'read-package',
        name: 'read_file',
        args: { file_path: '/project/package.json' }
      },
      {
        callId: 'search-functions',
        name: 'grep',
        args: { pattern: 'function', path: 'src/' }
      },
      {
        callId: 'list-files',
        name: 'ls',
        args: { path: '/project/src' }
      }
    ], signal);

    console.log('Tool calls scheduled successfully');
  } catch (error) {
    console.error('Failed to schedule tool calls:', error);
  }
}

basicToolSchedulingExample();
```

### 带用户确认的工具调度

```typescript
// 用户确认示例
async function userConfirmationExample() {
  const config = new Config({
    approvalMode: ApprovalMode.CONFIRM_ALWAYS, // 总是确认模式
  });

  const scheduler = new CoreToolScheduler({
    toolRegistry: Promise.resolve(toolRegistry),
    config,
    outputUpdateHandler: (callId, outputChunk) => {
      process.stdout.write(`[${callId}] ${outputChunk}`);
    },
    onAllToolCallsComplete: (completedCalls) => {
      console.log('\n=== Execution Summary ===');
      completedCalls.forEach(call => {
        const duration = call.durationMs ? `(${call.durationMs}ms)` : '';
        console.log(`${call.request.name}: ${call.status} ${duration}`);
        
        if (call.status === 'success' && call.response.resultDisplay) {
          console.log(`  Result: ${call.response.resultDisplay}`);
        } else if (call.status === 'error') {
          console.log(`  Error: ${call.response.error?.message}`);
        }
      });
    },
    getPreferredEditor: () => process.env.EDITOR === 'code' 
      ? EditorType.VSCode 
      : EditorType.Vim,
  });

  const signal = new AbortController().signal;
  
  // 模拟用户确认处理（实际应用中会由 UI 处理）
  const handleToolConfirmation = async (toolCall: WaitingToolCall) => {
    console.log(`\n=== Tool Confirmation Required ===`);
    console.log(`Tool: ${toolCall.request.name}`);
    console.log(`Args:`, JSON.stringify(toolCall.request.args, null, 2));
    
    if (toolCall.confirmationDetails.type === 'edit') {
      console.log('File changes:');
      console.log(toolCall.confirmationDetails.fileDiff);
    }
    
    // 模拟用户输入（实际应用中会显示 UI 确认对话框）
    const mockUserChoice = Math.random() > 0.3 
      ? ToolConfirmationOutcome.ProceedOnce 
      : ToolConfirmationOutcome.Cancel;
    
    console.log(`User choice: ${mockUserChoice}`);
    return mockUserChoice;
  };

  try {
    await scheduler.schedule({
      callId: 'dangerous-edit',
      name: 'edit',
      args: {
        file_path: '/project/src/important.js',
        new_content: 'console.log("Modified by AI");',
      }
    }, signal);

    console.log('Tool call scheduled, awaiting user confirmation...');
  } catch (error) {
    console.error('Tool scheduling failed:', error);
  }
}
```

### 实时输出处理

```typescript
// 实时输出处理示例
async function realTimeOutputExample() {
  const scheduler = new CoreToolScheduler({
    toolRegistry: Promise.resolve(toolRegistry),
    config,
    outputUpdateHandler: (callId, outputChunk) => {
      // 实时显示工具输出
      const timestamp = new Date().toISOString().slice(11, 23);
      process.stdout.write(`[${timestamp}] [${callId}] ${outputChunk}`);
    },
    onToolCallsUpdate: (toolCalls) => {
      // 显示执行进度
      const executing = toolCalls.filter(c => c.status === 'executing');
      if (executing.length > 0) {
        console.log(`\nCurrently executing ${executing.length} tool(s):`);
        executing.forEach(call => {
          const executingCall = call as ExecutingToolCall;
          const duration = executingCall.startTime 
            ? Date.now() - executingCall.startTime 
            : 0;
          console.log(`  - ${call.request.name} (${duration}ms)`);
          if (executingCall.liveOutput) {
            const preview = executingCall.liveOutput.slice(-50);
            console.log(`    Latest: ...${preview}`);
          }
        });
      }
    },
    getPreferredEditor: () => EditorType.VSCode,
  });

  const signal = new AbortController().signal;

  // 调度一个长时间运行的工具
  await scheduler.schedule({
    callId: 'long-running-task',
    name: 'shell',
    args: {
      command: 'npm install && npm run build && npm test',
      background: false
    }
  }, signal);

  console.log('Long-running task scheduled with real-time output...');
}
```

### 错误处理和恢复

```typescript
// 错误处理示例
async function errorHandlingExample() {
  const scheduler = new CoreToolScheduler({
    toolRegistry: Promise.resolve(toolRegistry),
    config,
    onAllToolCallsComplete: (completedCalls) => {
      // 分析执行结果
      const results = {
        successful: completedCalls.filter(c => c.status === 'success').length,
        failed: completedCalls.filter(c => c.status === 'error').length,
        cancelled: completedCalls.filter(c => c.status === 'cancelled').length,
      };

      console.log('\n=== Execution Results ===');
      console.log(`✅ Successful: ${results.successful}`);
      console.log(`❌ Failed: ${results.failed}`);
      console.log(`🚫 Cancelled: ${results.cancelled}`);

      // 处理错误
      const erroredCalls = completedCalls.filter(c => c.status === 'error') as ErroredToolCall[];
      if (erroredCalls.length > 0) {
        console.log('\n=== Error Details ===');
        erroredCalls.forEach(call => {
          console.log(`Tool: ${call.request.name}`);
          console.log(`Error: ${call.response.error?.message}`);
          console.log(`Duration: ${call.durationMs}ms`);
          console.log('---');
        });

        // 错误恢复策略
        console.log('\n=== Recovery Suggestions ===');
        erroredCalls.forEach(call => {
          const toolName = call.request.name;
          if (toolName === 'read_file') {
            console.log('- Check file path and permissions');
          } else if (toolName === 'shell') {
            console.log('- Verify command syntax and dependencies');
          } else if (toolName === 'edit') {
            console.log('- Ensure file is writable and backup exists');
          }
        });
      }
    },
    getPreferredEditor: () => EditorType.VSCode,
  });

  const signal = new AbortController().signal;

  // 设置超时处理
  const timeoutId = setTimeout(() => {
    console.log('\nTimeout reached, aborting tool execution...');
    signal.abort();
  }, 30000); // 30秒超时

  try {
    // 调度可能失败的工具调用
    await scheduler.schedule([
      {
        callId: 'read-nonexistent',
        name: 'read_file',
        args: { file_path: '/nonexistent/file.txt' }
      },
      {
        callId: 'invalid-shell',
        name: 'shell',
        args: { command: 'invalid-command-that-does-not-exist' }
      },
      {
        callId: 'valid-operation',
        name: 'ls',
        args: { path: '.' }
      }
    ], signal);

  } catch (error) {
    console.error('Scheduling error:', error);
  } finally {
    clearTimeout(timeoutId);
  }
}
```

### 编辑器集成示例

```typescript
// 编辑器集成示例
async function editorIntegrationExample() {
  const scheduler = new CoreToolScheduler({
    toolRegistry: Promise.resolve(toolRegistry),
    config: new Config({
      approvalMode: ApprovalMode.CONFIRM_ALWAYS,
    }),
    getPreferredEditor: () => {
      // 根据环境选择编辑器
      if (process.env.VSCODE_INJECTION) {
        return EditorType.VSCode;
      } else if (process.env.VIM) {
        return EditorType.Vim;
      } else {
        return EditorType.VSCode; // 默认
      }
    },
    onToolCallsUpdate: (toolCalls) => {
      const waitingForApproval = toolCalls.filter(c => c.status === 'awaiting_approval');
      if (waitingForApproval.length > 0) {
        console.log(`\n${waitingForApproval.length} tool(s) awaiting approval:`);
        waitingForApproval.forEach(call => {
          const waitingCall = call as WaitingToolCall;
          if (waitingCall.confirmationDetails.isModifying) {
            console.log(`  - ${call.request.name}: Editing in external editor...`);
          } else {
            console.log(`  - ${call.request.name}: Awaiting user confirmation`);
          }
        });
      }
    },
  });

  const signal = new AbortController().signal;

  // 模拟确认处理器，支持编辑器修改
  const mockConfirmationHandler = async (toolCall: WaitingToolCall) => {
    console.log(`\nConfirmation required for: ${toolCall.request.name}`);
    
    if (toolCall.confirmationDetails.type === 'edit') {
      console.log('File edit detected. Options:');
      console.log('1. Approve as-is');
      console.log('2. Modify with editor');
      console.log('3. Cancel');
      
      // 模拟用户选择编辑器修改
      const choice = Math.random();
      if (choice < 0.4) {
        return ToolConfirmationOutcome.ProceedOnce;
      } else if (choice < 0.8) {
        return ToolConfirmationOutcome.ModifyWithEditor;
      } else {
        return ToolConfirmationOutcome.Cancel;
      }
    }
    
    return ToolConfirmationOutcome.ProceedOnce;
  };

  try {
    await scheduler.schedule({
      callId: 'edit-with-confirmation',
      name: 'edit',
      args: {
        file_path: '/project/src/component.tsx',
        old_string: 'const Component = () => {',
        new_string: 'const Component: React.FC = () => {',
      }
    }, signal);

    console.log('Edit tool scheduled with editor integration...');
  } catch (error) {
    console.error('Editor integration error:', error);
  }
}
```

## 高级功能

### 内联修改处理

```typescript
// 内联修改功能
private async _applyInlineModify(
  toolCall: WaitingToolCall,
  payload: ToolConfirmationPayload,
  signal: AbortSignal,
): Promise<void> {
  if (
    toolCall.confirmationDetails.type !== 'edit' ||
    !isModifiableTool(toolCall.tool)
  ) {
    return;
  }

  const modifyContext = toolCall.tool.getModifyContext(signal);
  const currentContent = await modifyContext.getCurrentContent(
    toolCall.request.args,
  );

  const updatedParams = modifyContext.createUpdatedParams(
    currentContent,
    payload.newContent,
    toolCall.request.args,
  );
  
  const updatedDiff = Diff.createPatch(
    modifyContext.getFilePath(toolCall.request.args),
    currentContent,
    payload.newContent,
    'Current',
    'Proposed',
  );

  this.setArgsInternal(toolCall.request.callId, updatedParams);
  this.setStatusInternal(toolCall.request.callId, 'awaiting_approval', {
    ...toolCall.confirmationDetails,
    fileDiff: updatedDiff,
  });
}
```

### 并发控制

```typescript
// 并发控制检查
private isRunning(): boolean {
  return this.toolCalls.some(
    (call) =>
      call.status === 'executing' || call.status === 'awaiting_approval',
  );
}
```

**功能**: 防止在有工具正在执行或等待确认时调度新的工具调用

### 完成检查和通知

```typescript
private checkAndNotifyCompletion(): void {
  const allCallsAreTerminal = this.toolCalls.every(
    (call) =>
      call.status === 'success' ||
      call.status === 'error' ||
      call.status === 'cancelled',
  );

  if (this.toolCalls.length > 0 && allCallsAreTerminal) {
    const completedCalls = [...this.toolCalls] as CompletedToolCall[];
    this.toolCalls = [];

    // 记录工具调用日志
    for (const call of completedCalls) {
      logToolCall(this.config, new ToolCallEvent(call));
    }

    // 通知完成
    if (this.onAllToolCallsComplete) {
      this.onAllToolCallsComplete(completedCalls);
    }
    this.notifyToolCallsUpdate();
  }
}
```

## 最佳实践

### 1. 错误处理策略
```typescript
// ✅ 推荐：完善的错误处理
const scheduler = new CoreToolScheduler({
  // ... 其他配置
  onAllToolCallsComplete: (completedCalls) => {
    const errors = completedCalls.filter(c => c.status === 'error');
    if (errors.length > 0) {
      // 记录错误日志
      console.error(`${errors.length} tool calls failed`);
      
      // 实现重试逻辑
      const retryableCalls = errors.filter(call => 
        isRetryableError(call.response.error)
      );
      
      if (retryableCalls.length > 0) {
        console.log(`Retrying ${retryableCalls.length} failed calls...`);
        // 重新调度
      }
    }
  }
});

function isRetryableError(error?: Error): boolean {
  return error?.message.includes('timeout') || 
         error?.message.includes('network') ||
         error?.message.includes('temporary');
}
```

### 2. 性能监控
```typescript
// ✅ 推荐：性能监控和优化
class PerformanceTracker {
  private metrics = {
    totalCalls: 0,
    averageDuration: 0,
    successRate: 0,
  };

  trackCompletion(completedCalls: CompletedToolCall[]) {
    this.metrics.totalCalls += completedCalls.length;
    
    const durations = completedCalls
      .filter(c => c.durationMs)
      .map(c => c.durationMs!);
    
    if (durations.length > 0) {
      this.metrics.averageDuration = 
        durations.reduce((sum, d) => sum + d, 0) / durations.length;
    }
    
    const successful = completedCalls.filter(c => c.status === 'success').length;
    this.metrics.successRate = successful / completedCalls.length;
    
    console.log('Performance metrics:', this.metrics);
  }
}

const performanceTracker = new PerformanceTracker();
const scheduler = new CoreToolScheduler({
  // ... 其他配置
  onAllToolCallsComplete: (completedCalls) => {
    performanceTracker.trackCompletion(completedCalls);
  }
});
```

### 3. 资源管理
```typescript
// ✅ 推荐：资源管理和清理
class ManagedToolScheduler {
  private scheduler: CoreToolScheduler;
  private activeSignals: AbortController[] = [];

  constructor(options: CoreToolSchedulerOptions) {
    this.scheduler = new CoreToolScheduler(options);
  }

  async scheduleWithTimeout(
    request: ToolCallRequestInfo | ToolCallRequestInfo[],
    timeoutMs: number = 30000
  ): Promise<void> {
    const controller = new AbortController();
    this.activeSignals.push(controller);

    const timeoutId = setTimeout(() => {
      console.log('Tool execution timeout, aborting...');
      controller.abort();
    }, timeoutMs);

    try {
      await this.scheduler.schedule(request, controller.signal);
    } finally {
      clearTimeout(timeoutId);
      this.activeSignals = this.activeSignals.filter(c => c !== controller);
    }
  }

  dispose(): void {
    // 清理所有活动的信号
    this.activeSignals.forEach(controller => controller.abort());
    this.activeSignals = [];
  }
}
```

### 4. 状态持久化
```typescript
// ✅ 推荐：状态持久化和恢复
interface ToolCallSnapshot {
  toolCalls: ToolCall[];
  timestamp: number;
  sessionId: string;
}

class PersistentToolScheduler {
  private scheduler: CoreToolScheduler;
  private snapshotPath: string;

  constructor(options: CoreToolSchedulerOptions, snapshotPath: string) {
    this.snapshotPath = snapshotPath;
    this.scheduler = new CoreToolScheduler({
      ...options,
      onToolCallsUpdate: (toolCalls) => {
        this.saveSnapshot(toolCalls);
        options.onToolCallsUpdate?.(toolCalls);
      }
    });
  }

  private saveSnapshot(toolCalls: ToolCall[]): void {
    const snapshot: ToolCallSnapshot = {
      toolCalls,
      timestamp: Date.now(),
      sessionId: process.env.SESSION_ID || 'default',
    };

    fs.writeFileSync(this.snapshotPath, JSON.stringify(snapshot, null, 2));
  }

  loadSnapshot(): ToolCallSnapshot | null {
    try {
      if (fs.existsSync(this.snapshotPath)) {
        return JSON.parse(fs.readFileSync(this.snapshotPath, 'utf8'));
      }
    } catch (error) {
      console.warn('Failed to load snapshot:', error);
    }
    return null;
  }
}
```

`CoreToolScheduler` 是 Gemini CLI 工具执行系统的核心协调器，提供了完整的工具调用生命周期管理、用户确认流程、错误处理和状态跟踪功能。通过合理使用这个调度器，可以构建强大且用户友好的 AI 工具执行系统。