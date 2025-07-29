# GeminiClient 类文档

## 概述
`GeminiClient` 是 Gemini CLI 的核心客户端类，负责与 Google Gemini AI API 的交互、聊天会话管理、内容生成和消息流处理。

## 主要功能
- AI 聊天会话管理
- 内容生成（文本、JSON、嵌入向量）
- 消息流处理和工具调用
- 聊天历史压缩和优化
- 模型回退机制（Flash fallback）
- 循环检测和防护

---

## 类属性

### 私有属性
- `chat?: GeminiChat` - 聊天会话实例
- `contentGenerator?: ContentGenerator` - 内容生成器实例
- `embeddingModel: string` - 嵌入模型名称
- `generateContentConfig: GenerateContentConfig` - 内容生成配置
- `sessionTurnCount: number` - 会话轮次计数
- `readonly MAX_TURNS: number = 100` - 最大轮次限制
- `readonly COMPRESSION_TOKEN_THRESHOLD: number = 0.7` - 压缩令牌阈值
- `readonly COMPRESSION_PRESERVE_THRESHOLD: number = 0.3` - 压缩保留阈值
- `readonly loopDetector: LoopDetectionService` - 循环检测服务
- `lastPromptId?: string` - 最后一个提示ID

---

## 工具函数

### `isThinkingSupported(model: string): boolean`
**功能**: 检查指定模型是否支持思考模式
**参数**:
- `model: string` - 模型名称
**返回**: `boolean` - 是否支持思考模式
**实现**:
```typescript
function isThinkingSupported(model: string) {
  if (model.startsWith('gemini-2.5')) return true;
  return false;
}
```

### `findIndexAfterFraction(history: Content[], fraction: number): number`
**功能**: 查找历史记录中指定分数位置后的内容索引
**参数**:
- `history: Content[]` - 历史记录数组
- `fraction: number` - 分数值（0-1之间）
**返回**: `number` - 目标索引位置
**用途**: 用于聊天历史压缩时确定保留部分的起始位置
**异常**: 当分数不在0-1范围内时抛出错误

---

## 构造函数

### `constructor(private config: Config)`
**功能**: 初始化GeminiClient实例
**参数**:
- `config: Config` - 配置对象
**实现**:
- 设置代理配置（如果存在）
- 初始化嵌入模型
- 创建循环检测服务实例

---

## 核心方法

### 初始化方法

#### `async initialize(contentGeneratorConfig: ContentGeneratorConfig): Promise<void>`
**功能**: 异步初始化客户端
**参数**:
- `contentGeneratorConfig: ContentGeneratorConfig` - 内容生成器配置
**实现**:
1. 创建内容生成器实例
2. 启动聊天会话

#### `async startChat(extraHistory?: Content[]): Promise<GeminiChat>`
**功能**: 启动新的聊天会话
**参数**:
- `extraHistory?: Content[]` - 额外的历史记录（可选）
**返回**: `Promise<GeminiChat>` - 聊天实例
**实现**:
1. 获取环境信息
2. 设置工具声明
3. 构建初始历史记录
4. 配置系统指令和思考模式
5. 创建并返回GeminiChat实例

### 访问器方法

#### `getContentGenerator(): ContentGenerator`
**功能**: 获取内容生成器实例
**返回**: `ContentGenerator` - 内容生成器
**异常**: 当内容生成器未初始化时抛出错误

#### `getUserTier(): UserTierId | undefined`
**功能**: 获取用户层级
**返回**: `UserTierId | undefined` - 用户层级信息

#### `getChat(): GeminiChat`
**功能**: 获取聊天实例
**返回**: `GeminiChat` - 聊天实例
**异常**: 当聊天未初始化时抛出错误

#### `isInitialized(): boolean`
**功能**: 检查客户端是否已初始化
**返回**: `boolean` - 初始化状态

### 历史记录管理

#### `async addHistory(content: Content): Promise<void>`
**功能**: 添加历史记录
**参数**:
- `content: Content` - 要添加的内容

#### `getHistory(): Content[]`
**功能**: 获取聊天历史记录
**返回**: `Content[]` - 历史记录数组

#### `setHistory(history: Content[]): void`
**功能**: 设置聊天历史记录
**参数**:
- `history: Content[]` - 新的历史记录

### 工具管理

#### `async setTools(): Promise<void>`
**功能**: 设置可用工具
**实现**:
1. 从工具注册表获取函数声明
2. 将工具设置到聊天实例中

#### `async resetChat(): Promise<void>`
**功能**: 重置聊天会话
**实现**: 创建新的聊天实例

---

## 消息处理方法

### `async *sendMessageStream(...): AsyncGenerator<ServerGeminiStreamEvent, Turn>`
**功能**: 发送消息并获取流式响应
**参数**:
- `request: PartListUnion` - 请求内容
- `signal: AbortSignal` - 中止信号
- `prompt_id: string` - 提示ID
- `turns: number = this.MAX_TURNS` - 最大轮次
- `originalModel?: string` - 原始模型（可选）
**返回**: `AsyncGenerator<ServerGeminiStreamEvent, Turn>` - 流式事件生成器
**实现**:
1. 循环检测和重置
2. 会话轮次限制检查
3. 聊天历史压缩（如需要）
4. IDE模式上下文添加
5. 执行对话轮次
6. 下一个发言者检查和递归调用

### `async generateJson(...): Promise<Record<string, unknown>>`
**功能**: 生成JSON格式的内容
**参数**:
- `contents: Content[]` - 输入内容
- `schema: SchemaUnion` - JSON模式
- `abortSignal: AbortSignal` - 中止信号
- `model?: string` - 模型名称（可选）
- `config: GenerateContentConfig = {}` - 生成配置
**返回**: `Promise<Record<string, unknown>>` - 解析后的JSON对象
**实现**:
1. 选择使用的模型
2. 配置系统指令和响应格式
3. 执行API调用（带重试）
4. 解析JSON响应
5. 错误处理和报告

### `async generateContent(...): Promise<GenerateContentResponse>`
**功能**: 生成普通文本内容
**参数**:
- `contents: Content[]` - 输入内容
- `generationConfig: GenerateContentConfig` - 生成配置
- `abortSignal: AbortSignal` - 中止信号
- `model?: string` - 模型名称（可选）
**返回**: `Promise<GenerateContentResponse>` - 生成响应
**实现**:
1. 合并配置选项
2. 设置系统指令
3. 执行API调用（带重试和回退）
4. 错误处理

### `async generateEmbedding(texts: string[]): Promise<number[][]>`
**功能**: 生成文本嵌入向量
**参数**:
- `texts: string[]` - 输入文本数组
**返回**: `Promise<number[][]>` - 嵌入向量数组
**实现**:
1. 验证输入参数
2. 调用嵌入API
3. 验证响应格式和数量
4. 提取并返回向量值

---

## 私有方法

### `private async getEnvironment(): Promise<Part[]>`
**功能**: 获取环境上下文信息
**返回**: `Promise<Part[]>` - 环境信息部分
**实现**:
1. 获取当前工作目录
2. 获取当前日期和平台信息
3. 获取文件夹结构
4. 可选地读取完整文件上下文
5. 构建环境描述文本

### `async tryCompressChat(prompt_id: string, force: boolean = false): Promise<ChatCompressionInfo | null>`
**功能**: 尝试压缩聊天历史
**参数**:
- `prompt_id: string` - 提示ID
- `force: boolean = false` - 是否强制压缩
**返回**: `Promise<ChatCompressionInfo | null>` - 压缩信息或null
**实现**:
1. 获取策划过的历史记录
2. 计算当前令牌数量
3. 检查是否需要压缩
4. 确定压缩和保留的边界
5. 执行压缩操作
6. 重新启动聊天会话
7. 返回压缩统计信息

### `private async handleFlashFallback(authType?: string, error?: unknown): Promise<string | null>`
**功能**: 处理Flash模型回退机制
**参数**:
- `authType?: string` - 认证类型
- `error?: unknown` - 错误信息
**返回**: `Promise<string | null>` - 回退模型名称或null
**实现**:
1. 检查是否为OAuth用户
2. 验证当前模型是否已是Flash模型
3. 调用配置的回退处理器
4. 根据处理结果更新模型配置

---

## 使用示例

```typescript
// 初始化客户端
const client = new GeminiClient(config);
await client.initialize(contentGeneratorConfig);

// 发送消息并处理响应
const messageStream = client.sendMessageStream(
  [{ text: "Hello, how can you help me?" }],
  abortSignal,
  "prompt-123"
);

for await (const event of messageStream) {
  console.log('Event:', event);
}

// 生成JSON内容
const jsonResult = await client.generateJson(
  [{ role: 'user', parts: [{ text: 'Generate a JSON object with user info' }] }],
  { type: 'object', properties: { name: { type: 'string' } } },
  abortSignal
);

// 生成嵌入向量
const embeddings = await client.generateEmbedding([
  "This is a sample text",
  "Another text for embedding"
]);
```

---

## 错误处理

该类实现了全面的错误处理机制：
- 使用 `reportError` 函数记录和报告错误
- 实现重试机制处理临时性错误
- 支持模型回退以处理配额限制
- 提供循环检测防止无限递归
- 包含会话轮次限制防护

## 依赖关系

- `@google/genai` - Google Gemini API SDK
- `./turn.js` - 对话轮次管理
- `./geminiChat.js` - 聊天会话实现
- `../config/config.js` - 配置管理
- `../utils/*` - 各种工具函数
- `../services/*` - 服务层组件