# GeminiChat 类文档

## 概述

`GeminiChat` 是 Gemini CLI 的聊天会话管理类，负责维护与 Google Gemini AI 模型的对话上下文，处理消息发送、响应接收和历史记录管理。它是 Google GenAI SDK 的扩展实现，修复了原版在函数响应处理方面的关键缺陷。

## 主要功能

- 聊天会话状态管理和历史记录维护
- 消息发送和流式响应处理
- 自动函数调用（AFC）历史管理
- Flash 模型回退机制
- API 调用遥测和错误处理
- 配额错误处理和重试逻辑

## 核心工具函数

### `isValidResponse(response: GenerateContentResponse): boolean`

**功能**: 验证 API 响应是否有效

**参数**:

- `response: GenerateContentResponse` - API 响应对象

**返回**: `boolean` - 响应是否有效

**验证逻辑**:

1. 检查 `candidates` 数组是否存在且非空
2. 验证第一个候选项的 `content` 是否存在
3. 调用 `isValidContent` 验证内容有效性

### `isValidContent(content: Content): boolean`

**功能**: 验证内容对象是否有效

**参数**:

- `content: Content` - 内容对象

**返回**: `boolean` - 内容是否有效

**验证规则**:

1. `parts` 数组必须存在且非空
2. 每个 `part` 对象必须存在且非空
3. 非思考部分的文本内容不能为空字符串

### `validateHistory(history: Content[]): void`

**功能**: 验证历史记录的角色正确性

**参数**:

- `history: Content[]` - 历史记录数组

**抛出异常**:

- 如果角色不是 'user' 或 'model'

### `extractCuratedHistory(comprehensiveHistory: Content[]): Content[]`

**功能**: 从完整历史中提取有效的历史记录

**参数**:

- `comprehensiveHistory: Content[]` - 完整的历史记录

**返回**: `Content[]` - 经过筛选的有效历史记录

**筛选逻辑**:

1. 保留所有用户输入
2. 验证模型输出的有效性
3. 如果模型输出无效，移除对应的用户输入
4. 确保历史记录的完整性

## 类属性

### 私有属性

- `private sendPromise: Promise<void>` - 当前消息发送的 Promise 状态
- `private readonly config: Config` - 配置对象
- `private readonly contentGenerator: ContentGenerator` - 内容生成器
- `private readonly generationConfig: GenerateContentConfig` - 生成配置
- `private history: Content[]` - 聊天历史记录

## 构造函数

### `constructor(...)`

```typescript
constructor(
  private readonly config: Config,
  private readonly contentGenerator: ContentGenerator,
  private readonly generationConfig: GenerateContentConfig = {},
  private history: Content[] = [],
)
```

**功能**: 初始化 GeminiChat 实例

**参数**:

- `config: Config` - 配置对象
- `contentGenerator: ContentGenerator` - 内容生成器实例
- `generationConfig?: GenerateContentConfig` - 生成配置（可选）
- `history?: Content[]` - 初始历史记录（可选）

**实现**: 验证初始历史记录的角色正确性

## 核心方法

### 历史记录管理

#### `getHistory(curated: boolean = false): Content[]`

**功能**: 获取聊天历史记录

**参数**:

- `curated: boolean` - 是否返回筛选后的历史（默认 false）

**返回**: `Content[]` - 历史记录数组

**实现**:

- `curated = true`: 返回 `extractCuratedHistory()` 处理后的历史
- `curated = false`: 返回完整的原始历史记录

#### `setHistory(history: Content[]): void`

**功能**: 设置聊天历史记录

**参数**:

- `history: Content[]` - 新的历史记录

**实现**: 验证并替换当前历史记录

#### `addHistory(content: Content): void`

**功能**: 添加单个内容到历史记录

**参数**:

- `content: Content` - 要添加的内容

#### `setTools(tools: Tool[]): void`

**功能**: 设置可用工具

**参数**:

- `tools: Tool[]` - 工具数组

**实现**: 更新生成配置中的工具设置

### API 调用和日志记录

#### `private _logApiRequest(...)`

**功能**: 记录 API 请求遥测数据

**参数**:

- `contents: Content[]` - 请求内容
- `model: string` - 模型名称
- `prompt_id: string` - 提示 ID

**实现**: 使用遥测系统记录请求信息

#### `private _logApiResponse(...)`

**功能**: 记录 API 响应遥测数据

**参数**:

- `durationMs: number` - 请求持续时间
- `prompt_id: string` - 提示 ID
- `usageMetadata?: GenerateContentResponseUsageMetadata` - 使用元数据
- `responseText?: string` - 响应文本

#### `private _logApiError(...)`

**功能**: 记录 API 错误遥测数据

**参数**:

- `durationMs: number` - 请求持续时间
- `error: unknown` - 错误对象
- `prompt_id: string` - 提示 ID

### Flash 回退机制

#### `private async handleFlashFallback(...): Promise<string | null>`

**功能**: 处理 Flash 模型回退

**参数**:

- `authType?: string` - 认证类型
- `error?: unknown` - 错误对象

**返回**: `Promise<string | null>` - 回退模型名称或 null

**回退逻辑**:

1. 仅为 OAuth 用户处理回退
2. 检查当前模型是否已是 Flash 模型
3. 调用配置的回退处理器
4. 根据处理结果决定是否切换模型

### 消息发送

#### `async sendMessage(...): Promise<GenerateContentResponse>`

**功能**: 发送消息并获取完整响应

**参数**:

- `params: SendMessageParameters` - 消息参数
- `prompt_id: string` - 提示 ID

**返回**: `Promise<GenerateContentResponse>` - 完整的 API 响应

**执行流程**:

##### 1. 序列化处理

```typescript
await this.sendPromise;
const userContent = createUserContent(params.message);
const requestContents = this.getHistory(true).concat(userContent);
```

确保消息按顺序处理，构建包含历史记录的完整请求

##### 2. API 调用和重试

```typescript
const apiCall = () => {
  const modelToUse = this.config.getModel() || DEFAULT_GEMINI_FLASH_MODEL;
  
  // 防止配额错误后立即调用 Flash 模型
  if (this.config.getQuotaErrorOccurred() && modelToUse === DEFAULT_GEMINI_FLASH_MODEL) {
    throw new Error('Please submit a new query to continue with the Flash model.');
  }
  
  return this.contentGenerator.generateContent({
    model: modelToUse,
    contents: requestContents,
    config: { ...this.generationConfig, ...params.config },
  });
};

response = await retryWithBackoff(apiCall, {
  shouldRetry: (error: Error) => {
    if (error?.message?.includes('429')) return true;
    if (error?.message?.match(/5\d{2}/)) return true;
    return false;
  },
  onPersistent429: async (authType?: string, error?: unknown) =>
    await this.handleFlashFallback(authType, error),
  authType: this.config.getContentGeneratorConfig()?.authType,
});
```

##### 3. 历史记录更新

```typescript
this.sendPromise = (async () => {
  const outputContent = response.candidates?.[0]?.content;
  const automaticFunctionCallingHistory = 
    response.automaticFunctionCallingHistory?.slice(this.getHistory(true).length) ?? [];
  const modelOutput = outputContent ? [outputContent] : [];
  
  this.recordHistory(userContent, modelOutput, automaticFunctionCallingHistory);
})();
```

#### `async sendMessageStream(...): Promise<AsyncGenerator<GenerateContentResponse>>`

**功能**: 发送消息并获取流式响应

**参数**:

- `params: SendMessageParameters` - 消息参数
- `prompt_id: string` - 提示 ID

**返回**: `Promise<AsyncGenerator<GenerateContentResponse>>` - 流式响应生成器

**实现特点**:

- 与 `sendMessage` 类似的错误处理和重试逻辑
- 支持实时流式输出
- 完整的历史记录管理

### 历史记录操作

#### `private recordHistory(...)`

**功能**: 记录对话历史

**参数**:

- `userContent: Content` - 用户输入内容
- `modelOutput: Content[]` - 模型输出内容
- `automaticFunctionCallingHistory: Content[]` - 自动函数调用历史

**实现**: 将用户输入、模型输出和函数调用历史按顺序添加到历史记录中

## 错误处理机制

### 重试策略

- **429 错误**: 速率限制错误，支持重试
- **5xx 错误**: 服务器错误，支持重试
- **持续 429 错误**: 触发 Flash 模型回退

### 配额错误处理

```typescript
if (this.config.getQuotaErrorOccurred() && modelToUse === DEFAULT_GEMINI_FLASH_MODEL) {
  throw new Error('Please submit a new query to continue with the Flash model.');
}
```

防止在配额错误后立即使用 Flash 模型，要求用户提交新查询

### 异常恢复

- 自动重置 `sendPromise` 状态
- 完整的错误遥测记录
- 优雅的错误传播

## 使用示例

### 基本聊天会话

```typescript
const chat = new GeminiChat(config, contentGenerator);

// 发送消息并获取完整响应
const response = await chat.sendMessage({
  message: 'Hello, how can you help me?'
}, 'prompt-123');

console.log(response.text);
```

### 流式聊天

```typescript
// 发送消息并获取流式响应
const responseStream = await chat.sendMessageStream({
  message: 'Tell me a long story'
}, 'prompt-456');

for await (const chunk of responseStream) {
  process.stdout.write(chunk.text || '');
}
```

### 带工具的聊天

```typescript
// 设置可用工具
chat.setTools([
  {
    functionDeclarations: [
      {
        name: 'get_weather',
        description: 'Get weather information',
        parameters: {
          type: 'object',
          properties: {
            location: { type: 'string', description: 'City name' }
          }
        }
      }
    ]
  }
]);

// 发送可能触发工具调用的消息
const response = await chat.sendMessage({
  message: 'What is the weather in New York?'
}, 'prompt-789');
```

### 历史记录管理

```typescript
// 获取完整历史
const fullHistory = chat.getHistory();

// 获取筛选后的有效历史
const curatedHistory = chat.getHistory(true);

// 设置新的历史记录
chat.setHistory([
  { role: 'user', parts: [{ text: 'Previous conversation...' }] },
  { role: 'model', parts: [{ text: 'Model response...' }] }
]);
```

## 依赖关系

### 外部依赖

- `@google/genai` - Google GenAI SDK
- `../utils/retry.js` - 重试机制工具
- `../utils/messageInspectors.js` - 消息检查工具

### 内部依赖

- `./contentGenerator.js` - 内容生成器
- `../config/config.js` - 配置管理
- `../telemetry/loggers.js` - 遥测日志记录
- `../config/models.js` - 模型配置

## 特殊功能

### 自动函数调用（AFC）

- 自动处理函数调用历史
- 去重复化历史记录
- 完整的调用上下文维护

### 并发控制

- 使用 `sendPromise` 确保消息按顺序处理
- 防止并发请求导致的状态混乱
- 异常情况下的状态重置

### 遥测集成

- 完整的 API 调用记录
- 性能指标收集
- 错误统计和分析

### 配额管理

- 智能的 Flash 模型回退
- 用户友好的配额错误处理
- 透明的模型切换机制