# Gemini Client - Gemini API 客户端核心文档

## 概述

`GeminiClient` 是 Gemini CLI 的核心客户端类，负责与 Google Gemini API 的所有交互。它管理聊天会话、内容生成、工具调用、历史记录压缩、循环检测等核心功能，是整个系统的中央控制器和 API 交互层。

## 主要功能

- **会话管理**: 完整的聊天会话生命周期管理和状态维护
- **内容生成**: 流式和非流式内容生成，支持多种输出格式
- **工具集成**: 动态工具注册、调用和响应处理
- **智能压缩**: 基于令牌限制的自动历史记录压缩
- **循环检测**: 防止无限循环和重复响应的安全机制
- **IDE 集成**: 支持 IDE 上下文信息的自动注入
- **错误处理**: 完善的重试机制和错误恢复策略
- **模型切换**: 智能模型降级和配额错误处理

## 核心类定义

### `GeminiClient` 类

```typescript
export class GeminiClient {
  private chat?: GeminiChat;
  private contentGenerator?: ContentGenerator;
  private embeddingModel: string;
  private generateContentConfig: GenerateContentConfig = {
    temperature: 0,
    topP: 1,
  };
  private sessionTurnCount = 0;
  private readonly MAX_TURNS = 100;
  private readonly COMPRESSION_TOKEN_THRESHOLD = 0.7;
  private readonly COMPRESSION_PRESERVE_THRESHOLD = 0.3;
  private readonly loopDetector: LoopDetectionService;
  private lastPromptId?: string;

  constructor(private config: Config);
}
```

**核心属性详情**:
- `chat`: GeminiChat 实例，管理对话历史和上下文
- `contentGenerator`: ContentGenerator 实例，处理 API 调用
- `embeddingModel`: 用于生成嵌入向量的模型名称
- `generateContentConfig`: 默认的内容生成配置
- `sessionTurnCount`: 当前会话的对话轮数计数
- `MAX_TURNS`: 防止无限循环的最大轮数限制
- `COMPRESSION_TOKEN_THRESHOLD`: 触发压缩的令牌阈值（70%）
- `COMPRESSION_PRESERVE_THRESHOLD`: 压缩后保留的历史比例（30%）
- `loopDetector`: 循环检测服务实例
- `lastPromptId`: 上一个提示的 ID，用于循环检测

## 核心配置常量

### 压缩和限制配置

```typescript
// 最大对话轮数限制
private readonly MAX_TURNS = 100;

// 压缩令牌阈值 - 当历史记录达到模型令牌限制的70%时触发压缩
private readonly COMPRESSION_TOKEN_THRESHOLD = 0.7;

// 压缩保留阈值 - 压缩后保留最近30%的对话历史
private readonly COMPRESSION_PRESERVE_THRESHOLD = 0.3;
```

### 默认生成配置

```typescript
private generateContentConfig: GenerateContentConfig = {
  temperature: 0,    // 确定性输出
  topP: 1,          // 完整的概率分布
};
```

## 初始化和配置

### 构造函数

```typescript
constructor(private config: Config) {
  // 代理配置
  if (config.getProxy()) {
    setGlobalDispatcher(new ProxyAgent(config.getProxy() as string));
  }

  // 模型配置
  this.embeddingModel = config.getEmbeddingModel();
  
  // 服务初始化
  this.loopDetector = new LoopDetectionService(config);
}
```

**功能**: 初始化客户端，设置代理、模型和服务

### 异步初始化

```typescript
async initialize(contentGeneratorConfig: ContentGeneratorConfig): Promise<void> {
  this.contentGenerator = await createContentGenerator(
    contentGeneratorConfig,
    this.config,
    this.config.getSessionId(),
  );
  this.chat = await this.startChat();
}
```

**执行流程**:
1. **创建内容生成器**: 基于配置创建 ContentGenerator 实例
2. **启动聊天会话**: 初始化带有环境上下文的聊天会话
3. **工具注册**: 自动注册所有可用工具

**使用示例**:
```typescript
const client = new GeminiClient(config);

await client.initialize({
  authType: AuthType.LOGIN_WITH_GOOGLE,
  model: 'gemini-2.0-flash-exp',
  userTier: UserTierId.FREE,
});

console.log('Client initialized:', client.isInitialized());
```

## 会话管理

### 聊天会话创建

```typescript
async startChat(extraHistory?: Content[]): Promise<GeminiChat> {
  const envParts = await this.getEnvironment();
  const toolRegistry = await this.config.getToolRegistry();
  const toolDeclarations = toolRegistry.getFunctionDeclarations();
  const tools: Tool[] = [{ functionDeclarations: toolDeclarations }];
  
  const history: Content[] = [
    {
      role: 'user',
      parts: envParts,
    },
    {
      role: 'model', 
      parts: [{ text: 'Got it. Thanks for the context!' }],
    },
    ...(extraHistory ?? []),
  ];
  
  const userMemory = this.config.getUserMemory();
  const systemInstruction = getCoreSystemPrompt(userMemory);
  
  // 支持思考模式的模型配置
  const generateContentConfigWithThinking = isThinkingSupported(
    this.config.getModel(),
  )
    ? {
        ...this.generateContentConfig,
        thinkingConfig: {
          includeThoughts: true,
        },
      }
    : this.generateContentConfig;
  
  return new GeminiChat(
    this.config,
    this.getContentGenerator(),
    {
      systemInstruction,
      ...generateContentConfigWithThinking,
      tools,
    },
    history,
  );
}
```

**功能**: 创建新的聊天会话，包含环境上下文和工具配置

**会话初始化包含**:
- **环境信息**: 当前目录、日期、平台、文件夹结构
- **完整上下文**: 可选的完整文件内容加载
- **工具声明**: 所有注册工具的函数声明
- **系统指令**: 基于用户记忆的核心系统提示
- **思考配置**: 支持思考模式的模型的特殊配置

### 环境上下文生成

```typescript
private async getEnvironment(): Promise<Part[]> {
  const cwd = this.config.getWorkingDir();
  const today = new Date().toLocaleDateString(undefined, {
    weekday: 'long',
    year: 'numeric', 
    month: 'long',
    day: 'numeric',
  });
  const platform = process.platform;
  const folderStructure = await getFolderStructure(cwd, {
    fileService: this.config.getFileService(),
  });
  
  const context = `
This is the Gemini CLI. We are setting up the context for our chat.
Today's date is ${today}.
My operating system is: ${platform}
I'm currently working in the directory: ${cwd}
${folderStructure}
    `.trim();

  const initialParts: Part[] = [{ text: context }];
  
  // 完整上下文模式
  if (this.config.getFullContext()) {
    const toolRegistry = await this.config.getToolRegistry();
    const readManyFilesTool = toolRegistry.getTool('read_many_files') as ReadManyFilesTool;
    
    if (readManyFilesTool) {
      try {
        const result = await readManyFilesTool.execute(
          {
            paths: ['**/*'],
            useDefaultExcludes: true,
          },
          AbortSignal.timeout(30000),
        );
        
        if (result.llmContent) {
          initialParts.push({
            text: `\n--- Full File Context ---\n${result.llmContent}`,
          });
        }
      } catch (error) {
        console.error('Error reading full file context:', error);
        initialParts.push({
          text: '\n--- Error reading full file context ---',
        });
      }
    }
  }

  return initialParts;
}
```

**环境上下文包含**:
- **基本信息**: 日期、操作系统、工作目录
- **文件夹结构**: 当前目录的文件树结构
- **完整文件内容**: 可选的所有项目文件内容

## 消息流处理

### 流式消息发送

```typescript
async *sendMessageStream(
  request: PartListUnion,
  signal: AbortSignal,
  prompt_id: string,
  turns: number = this.MAX_TURNS,
  originalModel?: string,
): AsyncGenerator<ServerGeminiStreamEvent, Turn>
```

**功能**: 发送消息并返回流式响应的异步生成器

**主要流程**:

#### 1. 前置检查和初始化
```typescript
// 循环检测重置
if (this.lastPromptId !== prompt_id) {
  this.loopDetector.reset(prompt_id);
  this.lastPromptId = prompt_id;
}

// 会话轮数检查
this.sessionTurnCount++;
if (
  this.config.getMaxSessionTurns() > 0 &&
  this.sessionTurnCount > this.config.getMaxSessionTurns()
) {
  yield { type: GeminiEventType.MaxSessionTurns };
  return new Turn(this.getChat(), prompt_id);
}

// 轮数限制
const boundedTurns = Math.min(turns, this.MAX_TURNS);
const initialModel = originalModel || this.config.getModel();
```

#### 2. 历史压缩检查
```typescript
const compressed = await this.tryCompressChat(prompt_id);
if (compressed) {
  yield { type: GeminiEventType.ChatCompressed, value: compressed };
}
```

#### 3. IDE 上下文注入
```typescript
if (this.config.getIdeMode()) {
  const openFiles = ideContext.getOpenFilesContext();
  if (openFiles) {
    const contextParts: string[] = [];
    
    // 活动文件信息
    if (openFiles.activeFile) {
      contextParts.push(
        `This is the file that the user was most recently looking at:\n- Path: ${openFiles.activeFile}`,
      );
      
      // 光标位置
      if (openFiles.cursor) {
        contextParts.push(
          `This is the cursor position in the file:\n- Cursor Position: Line ${openFiles.cursor.line}, Character ${openFiles.cursor.character}`,
        );
      }
      
      // 选中文本
      if (openFiles.selectedText) {
        contextParts.push(
          `This is the selected text in the active file:\n- ${openFiles.selectedText}`,
        );
      }
    }

    // 最近打开的文件
    if (openFiles.recentOpenFiles && openFiles.recentOpenFiles.length > 0) {
      const recentFiles = openFiles.recentOpenFiles
        .map((file) => `- ${file.filePath}`)
        .join('\n');
      contextParts.push(
        `Here are files the user has recently opened, with the most recent at the top:\n${recentFiles}`,
      );
    }

    if (contextParts.length > 0) {
      request = [
        { text: contextParts.join('\n') },
        ...(Array.isArray(request) ? request : [request]),
      ];
    }
  }
}
```

#### 4. 循环检测和执行
```typescript
const turn = new Turn(this.getChat(), prompt_id);

// 开始前循环检测
const loopDetected = await this.loopDetector.turnStarted(signal);
if (loopDetected) {
  yield { type: GeminiEventType.LoopDetected };
  return turn;
}

// 执行对话轮次
const resultStream = turn.run(request, signal);
for await (const event of resultStream) {
  // 实时循环检测
  if (this.loopDetector.addAndCheck(event)) {
    yield { type: GeminiEventType.LoopDetected };
    return turn;
  }
  yield event;
}
```

#### 5. 下一轮检查和递归调用
```typescript
if (!turn.pendingToolCalls.length && signal && !signal.aborted) {
  // 模型切换检查
  const currentModel = this.config.getModel();
  if (currentModel !== initialModel) {
    return turn; // 避免模型切换后的意外执行
  }

  // 检查是否需要继续对话
  const nextSpeakerCheck = await checkNextSpeaker(
    this.getChat(),
    this,
    signal,
  );
  
  if (nextSpeakerCheck?.next_speaker === 'model') {
    logFlashDecidedToContinue(
      this.config,
      new FlashDecidedToContinueEvent(prompt_id),
    );
    
    const nextRequest = [{ text: 'Please continue.' }];
    
    // 递归调用继续对话
    yield* this.sendMessageStream(
      nextRequest,
      signal,
      prompt_id,
      boundedTurns - 1,
      initialModel,
    );
  }
}
```

**使用示例**:
```typescript
const client = new GeminiClient(config);
await client.initialize(contentConfig);

const signal = new AbortController().signal;
const promptId = 'unique-prompt-id';

try {
  for await (const event of client.sendMessageStream(
    [{ text: 'Hello, how can you help me?' }],
    signal,
    promptId
  )) {
    switch (event.type) {
      case GeminiEventType.Text:
        console.log('Text chunk:', event.value);
        break;
        
      case GeminiEventType.ToolCall:
        console.log('Tool call:', event.value);
        break;
        
      case GeminiEventType.ChatCompressed:
        console.log('Chat compressed:', event.value);
        break;
        
      case GeminiEventType.LoopDetected:
        console.log('Loop detected, stopping');
        break;
    }
  }
} catch (error) {
  console.error('Stream error:', error);
}
```

## 内容生成

### JSON 内容生成

```typescript
async generateJson(
  contents: Content[],
  schema: SchemaUnion,
  abortSignal: AbortSignal,
  model?: string,
  config: GenerateContentConfig = {},
): Promise<Record<string, unknown>>
```

**功能**: 生成符合指定 JSON 模式的结构化内容

**实现逻辑**:
```typescript
const modelToUse = model || this.config.getModel() || DEFAULT_GEMINI_FLASH_MODEL;

try {
  const userMemory = this.config.getUserMemory();
  const systemInstruction = getCoreSystemPrompt(userMemory);
  
  const requestConfig = {
    abortSignal,
    ...this.generateContentConfig,
    ...config,
  };

  const apiCall = () =>
    this.getContentGenerator().generateContent({
      model: modelToUse,
      config: {
        ...requestConfig,
        systemInstruction,
        responseSchema: schema,
        responseMimeType: 'application/json',
      },
      contents,
    });

  // 重试机制和降级处理
  const result = await retryWithBackoff(apiCall, {
    onPersistent429: async (authType?: string, error?: unknown) =>
      await this.handleFlashFallback(authType, error),
    authType: this.config.getContentGeneratorConfig()?.authType,
  });

  const text = getResponseText(result);
  if (!text) {
    throw new Error('API returned an empty response for generateJson.');
  }

  return JSON.parse(text);
} catch (error) {
  // 错误处理和报告
  await reportError(error, 'Error generating JSON content via API.', contents, 'generateJson-api');
  throw new Error(`Failed to generate JSON content: ${getErrorMessage(error)}`);
}
```

**使用示例**:
```typescript
// 定义 JSON 模式
const schema = {
  type: 'object',
  properties: {
    summary: { type: 'string' },
    keywords: {
      type: 'array',
      items: { type: 'string' }
    },
    confidence: { type: 'number' }
  },
  required: ['summary', 'keywords']
};

// 生成结构化内容
const contents = [
  { role: 'user', parts: [{ text: 'Analyze this text and provide a summary' }] }
];

const result = await client.generateJson(
  contents,
  schema,
  AbortSignal.timeout(30000)
);

console.log('Generated JSON:', result);
// 输出: { summary: "...", keywords: [...], confidence: 0.95 }
```

### 通用内容生成

```typescript
async generateContent(
  contents: Content[],
  generationConfig: GenerateContentConfig,
  abortSignal: AbortSignal,
  model?: string,
): Promise<GenerateContentResponse>
```

**功能**: 生成通用内容，支持自定义配置

**配置选项**:
```typescript
interface GenerateContentConfig {
  temperature?: number;        // 创造性控制 (0-2)
  topP?: number;              // 核采样参数 (0-1)
  topK?: number;              // 候选词数量限制
  maxOutputTokens?: number;   // 最大输出令牌数
  stopSequences?: string[];   // 停止序列
  responseMimeType?: string;  // 响应 MIME 类型
  responseSchema?: SchemaUnion; // 响应模式
}
```

**使用示例**:
```typescript
const contents = [
  { 
    role: 'user', 
    parts: [{ text: 'Write a creative story about AI' }] 
  }
];

const config = {
  temperature: 0.8,     // 高创造性
  maxOutputTokens: 1000,
  topP: 0.9,
};

const response = await client.generateContent(
  contents,
  config,
  AbortSignal.timeout(60000),
  'gemini-pro'
);

console.log('Generated content:', getResponseText(response));
```

### 嵌入向量生成

```typescript
async generateEmbedding(texts: string[]): Promise<number[][]>
```

**功能**: 为文本列表生成嵌入向量

**实现特点**:
- **批量处理**: 支持多个文本的批量嵌入生成
- **验证机制**: 确保输入输出数量匹配
- **错误检查**: 验证嵌入向量的有效性

**使用示例**:
```typescript
const texts = [
  "This is the first document",
  "Here is another piece of text", 
  "And this is the third document"
];

const embeddings = await client.generateEmbedding(texts);

console.log(`Generated ${embeddings.length} embeddings`);
console.log(`First embedding dimension: ${embeddings[0].length}`);

// 计算文本相似度
function cosineSimilarity(a: number[], b: number[]): number {
  const dotProduct = a.reduce((sum, val, i) => sum + val * b[i], 0);
  const magnitudeA = Math.sqrt(a.reduce((sum, val) => sum + val * val, 0));
  const magnitudeB = Math.sqrt(b.reduce((sum, val) => sum + val * val, 0));
  return dotProduct / (magnitudeA * magnitudeB);
}

const similarity = cosineSimilarity(embeddings[0], embeddings[1]);
console.log(`Similarity between texts: ${similarity}`);
```

## 历史记录压缩

### 智能压缩系统

```typescript
async tryCompressChat(
  prompt_id: string,
  force: boolean = false,
): Promise<ChatCompressionInfo | null>
```

**功能**: 当对话历史过长时自动压缩，保持性能和上下文相关性

**压缩逻辑**:

#### 1. 压缩条件检查
```typescript
const curatedHistory = this.getChat().getHistory(true);

// 空历史不压缩
if (curatedHistory.length === 0) {
  return null;
}

const model = this.config.getModel();
const { totalTokens: originalTokenCount } = 
  await this.getContentGenerator().countTokens({
    model,
    contents: curatedHistory,
  });

// 检查是否达到压缩阈值 (70% 的模型令牌限制)
if (
  !force &&
  originalTokenCount < this.COMPRESSION_TOKEN_THRESHOLD * tokenLimit(model)
) {
  return null;
}
```

#### 2. 压缩点确定
```typescript
// 找到需要压缩的分割点
let compressBeforeIndex = findIndexAfterFraction(
  curatedHistory,
  1 - this.COMPRESSION_PRESERVE_THRESHOLD, // 保留最后30%
);

// 确保从用户消息开始新的轮次
while (
  compressBeforeIndex < curatedHistory.length &&
  (curatedHistory[compressBeforeIndex]?.role === 'model' ||
   isFunctionResponse(curatedHistory[compressBeforeIndex]))
) {
  compressBeforeIndex++;
}

const historyToCompress = curatedHistory.slice(0, compressBeforeIndex);
const historyToKeep = curatedHistory.slice(compressBeforeIndex);
```

#### 3. 压缩执行
```typescript
// 设置待压缩历史
this.getChat().setHistory(historyToCompress);

// 生成压缩摘要
const { text: summary } = await this.getChat().sendMessage(
  {
    message: {
      text: 'First, reason in your scratchpad. Then, generate the <state_snapshot>.',
    },
    config: {
      systemInstruction: { text: getCompressionPrompt() },
    },
  },
  prompt_id,
);

// 创建新的聊天会话，包含压缩摘要和保留的历史
this.chat = await this.startChat([
  {
    role: 'user',
    parts: [{ text: summary }],
  },
  {
    role: 'model',
    parts: [{ text: 'Got it. Thanks for the additional context!' }],
  },
  ...historyToKeep,
]);
```

#### 4. 压缩效果统计
```typescript
const { totalTokens: newTokenCount } = 
  await this.getContentGenerator().countTokens({
    model: this.config.getModel(),
    contents: this.getChat().getHistory(),
  });

return {
  originalTokenCount,
  newTokenCount,
};
```

**压缩效果示例**:
```typescript
const compressionInfo = await client.tryCompressChat('prompt-123');

if (compressionInfo) {
  console.log(`压缩前令牌数: ${compressionInfo.originalTokenCount}`);
  console.log(`压缩后令牌数: ${compressionInfo.newTokenCount}`);
  console.log(`压缩率: ${(1 - compressionInfo.newTokenCount / compressionInfo.originalTokenCount) * 100}%`);
}
```

### 分割点查找算法

```typescript
export function findIndexAfterFraction(
  history: Content[],
  fraction: number,
): number {
  if (fraction <= 0 || fraction >= 1) {
    throw new Error('Fraction must be between 0 and 1');
  }

  // 计算每个内容的字符长度
  const contentLengths = history.map(
    (content) => JSON.stringify(content).length,
  );

  const totalCharacters = contentLengths.reduce(
    (sum, length) => sum + length,
    0,
  );
  const targetCharacters = totalCharacters * fraction;

  // 找到目标字符数对应的索引
  let charactersSoFar = 0;
  for (let i = 0; i < contentLengths.length; i++) {
    charactersSoFar += contentLengths[i];
    if (charactersSoFar >= targetCharacters) {
      return i;
    }
  }
  return contentLengths.length;
}
```

**功能**: 根据字符数比例找到历史记录的分割点

**使用示例**:
```typescript
const history = [
  { role: 'user', parts: [{ text: 'Hello' }] },
  { role: 'model', parts: [{ text: 'Hi there!' }] },
  { role: 'user', parts: [{ text: 'How are you?' }] },
  { role: 'model', parts: [{ text: 'I am doing well, thank you!' }] },
];

// 找到70%位置的索引
const index = findIndexAfterFraction(history, 0.7);
console.log(`分割点索引: ${index}`);

const toCompress = history.slice(0, index);
const toKeep = history.slice(index);
console.log(`压缩部分: ${toCompress.length} 条`);
console.log(`保留部分: ${toKeep.length} 条`);
```

## 模型降级处理

### Flash 模型降级

```typescript
private async handleFlashFallback(
  authType?: string,
  error?: unknown,
): Promise<string | null>
```

**功能**: 处理持续 429 错误时的模型降级

**降级条件**:
- 仅对 OAuth 用户生效
- 当前模型不是 Flash 模型
- 配置了降级处理器

**降级流程**:
```typescript
// 只处理 OAuth 用户的降级
if (authType !== AuthType.LOGIN_WITH_GOOGLE) {
  return null;
}

const currentModel = this.config.getModel();
const fallbackModel = DEFAULT_GEMINI_FLASH_MODEL;

// 已经是 Flash 模型则不降级
if (currentModel === fallbackModel) {
  return null;
}

// 检查配置的降级处理器
const fallbackHandler = this.config.flashFallbackHandler;
if (typeof fallbackHandler === 'function') {
  try {
    const accepted = await fallbackHandler(
      currentModel,
      fallbackModel,
      error,
    );
    
    if (accepted !== false && accepted !== null) {
      this.config.setModel(fallbackModel);
      return fallbackModel;
    }
    
    // 检查处理器是否手动切换了模型
    if (this.config.getModel() === fallbackModel) {
      return null; // 模型已切换但不继续当前提示
    }
  } catch (error) {
    console.warn('Flash fallback handler failed:', error);
  }
}

return null;
```

**降级处理器示例**:
```typescript
const fallbackHandler = async (
  currentModel: string,
  fallbackModel: string,
  error: unknown
) => {
  console.log(`模型 ${currentModel} 遇到配额限制，是否切换到 ${fallbackModel}？`);
  
  // 自动同意切换（生产环境可能需要用户确认）
  if (currentModel.includes('pro') && fallbackModel.includes('flash')) {
    console.log('自动切换到 Flash 模型以避免配额限制');
    return true;
  }
  
  return false;
};

// 在配置中设置降级处理器
config.flashFallbackHandler = fallbackHandler;
```

## 状态管理和查询

### 初始化状态检查

```typescript
isInitialized(): boolean {
  return this.chat !== undefined && this.contentGenerator !== undefined;
}
```

### 用户层级获取

```typescript
getUserTier(): UserTierId | undefined {
  return this.contentGenerator?.userTier;
}
```

### 历史记录管理

```typescript
// 获取对话历史
getHistory(): Content[] {
  return this.getChat().getHistory();
}

// 设置对话历史
setHistory(history: Content[]): void {
  this.getChat().setHistory(history);
}

// 添加历史记录
async addHistory(content: Content): Promise<void> {
  this.getChat().addHistory(content);
}
```

### 工具管理

```typescript
async setTools(): Promise<void> {
  const toolRegistry = await this.config.getToolRegistry();
  const toolDeclarations = toolRegistry.getFunctionDeclarations();
  const tools: Tool[] = [{ functionDeclarations: toolDeclarations }];
  this.getChat().setTools(tools);
}
```

### 聊天重置

```typescript
async resetChat(): Promise<void> {
  this.chat = await this.startChat();
}
```

## 使用示例

### 基本聊天会话

```typescript
import { GeminiClient } from './client.js';
import { Config } from '../config/config.js';
import { AuthType } from './contentGenerator.js';

// 基本聊天会话示例
async function basicChatExample() {
  const config = new Config({
    workingDir: process.cwd(),
    model: 'gemini-2.0-flash-exp',
    maxSessionTurns: 50,
  });

  const client = new GeminiClient(config);
  
  // 初始化客户端
  await client.initialize({
    authType: AuthType.LOGIN_WITH_GOOGLE,
    model: 'gemini-2.0-flash-exp',
  });

  console.log('Client initialized:', client.isInitialized());
  console.log('User tier:', client.getUserTier());

  // 发送消息并处理流式响应
  const signal = new AbortController().signal;
  const promptId = `chat-${Date.now()}`;

  try {
    const stream = client.sendMessageStream(
      [{ text: 'Hello! Can you help me understand TypeScript generics?' }],
      signal,
      promptId
    );

    for await (const event of stream) {
      switch (event.type) {
        case 'text':
          process.stdout.write(event.value);
          break;
          
        case 'toolCall':
          console.log('\n[Tool Call]:', event.value.name);
          break;
          
        case 'toolResult': 
          console.log('[Tool Result]:', event.value.summary);
          break;
          
        case 'chatCompressed':
          console.log('\n[Compression]:', 
            `${event.value.originalTokenCount} → ${event.value.newTokenCount} tokens`);
          break;
          
        case 'loopDetected':
          console.log('\n[Warning]: Loop detected, stopping conversation');
          break;
      }
    }
  } catch (error) {
    console.error('Chat error:', error);
  }
}
```

### JSON 结构化生成

```typescript
// JSON 结构化内容生成示例
async function structuredContentExample() {
  const client = new GeminiClient(config);
  await client.initialize(contentConfig);

  // 定义代码分析的 JSON 模式
  const codeAnalysisSchema = {
    type: 'object',
    properties: {
      summary: {
        type: 'string',
        description: 'Brief summary of the code'
      },
      functions: {
        type: 'array',
        items: {
          type: 'object',
          properties: {
            name: { type: 'string' },
            purpose: { type: 'string' },
            parameters: { type: 'array', items: { type: 'string' } },
            returnType: { type: 'string' }
          },
          required: ['name', 'purpose']
        }
      },
      complexity: {
        type: 'string',
        enum: ['low', 'medium', 'high']
      },
      suggestions: {
        type: 'array',
        items: { type: 'string' }
      }
    },
    required: ['summary', 'functions', 'complexity']
  };

  const codeToAnalyze = `
    function fibonacci(n: number): number {
      if (n <= 1) return n;
      return fibonacci(n - 1) + fibonacci(n - 2);
    }
    
    function memoizedFibonacci(n: number, memo: Map<number, number> = new Map()): number {
      if (memo.has(n)) return memo.get(n)!;
      if (n <= 1) return n;
      
      const result = memoizedFibonacci(n - 1, memo) + memoizedFibonacci(n - 2, memo);
      memo.set(n, result);
      return result;
    }
  `;

  const contents = [
    {
      role: 'user',
      parts: [{
        text: `Please analyze this TypeScript code and provide a structured response:\n\n${codeToAnalyze}`
      }]
    }
  ];

  try {
    const analysis = await client.generateJson(
      contents,
      codeAnalysisSchema,
      AbortSignal.timeout(30000)
    );

    console.log('Code Analysis Result:');
    console.log('Summary:', analysis.summary);
    console.log('Complexity:', analysis.complexity);
    console.log('Functions:');
    analysis.functions?.forEach((func: any, index: number) => {
      console.log(`  ${index + 1}. ${func.name}: ${func.purpose}`);
      if (func.parameters?.length > 0) {
        console.log(`     Parameters: ${func.parameters.join(', ')}`);
      }
    });
    
    if (analysis.suggestions?.length > 0) {
      console.log('Suggestions:');
      analysis.suggestions.forEach((suggestion: string, index: number) => {
        console.log(`  ${index + 1}. ${suggestion}`);
      });
    }
  } catch (error) {
    console.error('Analysis failed:', error);
  }
}
```

### 嵌入向量和相似度搜索

```typescript
// 嵌入向量和相似度搜索示例
async function embeddingSearchExample() {
  const client = new GeminiClient(config);
  await client.initialize(contentConfig);

  // 文档集合
  const documents = [
    "TypeScript is a strongly typed programming language that builds on JavaScript",
    "React is a JavaScript library for building user interfaces",
    "Node.js is a JavaScript runtime built on Chrome's V8 JavaScript engine", 
    "Vue.js is a progressive framework for building user interfaces",
    "Angular is a platform for building mobile and desktop web applications",
    "Python is a high-level programming language with dynamic semantics",
    "Go is an open source programming language that makes it easy to build simple and reliable software"
  ];

  console.log('Generating embeddings for documents...');
  const embeddings = await client.generateEmbedding(documents);

  // 查询
  const query = "What is a JavaScript framework for UI development?";
  console.log(`\nQuery: "${query}"`);
  
  const queryEmbedding = await client.generateEmbedding([query]);

  // 计算相似度
  function cosineSimilarity(a: number[], b: number[]): number {
    const dotProduct = a.reduce((sum, val, i) => sum + val * b[i], 0);
    const magnitudeA = Math.sqrt(a.reduce((sum, val) => sum + val * val, 0));
    const magnitudeB = Math.sqrt(b.reduce((sum, val) => sum + val * val, 0));
    return dotProduct / (magnitudeA * magnitudeB);
  }

  // 计算每个文档与查询的相似度
  const similarities = embeddings.map((docEmbedding, index) => ({
    document: documents[index],
    similarity: cosineSimilarity(queryEmbedding[0], docEmbedding),
    index
  }));

  // 按相似度排序
  similarities.sort((a, b) => b.similarity - a.similarity);

  console.log('\nTop 3 most similar documents:');
  similarities.slice(0, 3).forEach((result, rank) => {
    console.log(`${rank + 1}. Similarity: ${result.similarity.toFixed(4)}`);
    console.log(`   Document: "${result.document}"`);
    console.log();
  });
}
```

### 带工具调用的复杂会话

```typescript
// 复杂工具调用会话示例
async function complexToolCallExample() {
  const client = new GeminiClient(config);
  await client.initialize(contentConfig);

  // 设置工具
  await client.setTools();

  const signal = new AbortController().signal;
  const promptId = `complex-${Date.now()}`;

  try {
    console.log('Starting complex conversation with tool calls...');
    
    const stream = client.sendMessageStream(
      [{
        text: "I need to analyze all TypeScript files in this project. Can you read them and provide a summary of the codebase architecture?"
      }],
      signal,
      promptId
    );

    let currentToolCall: string | null = null;
    let responseText = '';

    for await (const event of stream) {
      switch (event.type) {
        case 'text':
          responseText += event.value;
          process.stdout.write(event.value);
          break;
          
        case 'toolCall':
          currentToolCall = event.value.name;
          console.log(`\n\n🔧 Calling tool: ${event.value.name}`);
          if (event.value.input) {
            console.log('📋 Input:', JSON.stringify(event.value.input, null, 2));
          }
          break;
          
        case 'toolResult':
          if (currentToolCall) {
            console.log(`✅ Tool ${currentToolCall} completed`);
            console.log('📄 Summary:', event.value.summary);
            console.log('---');
            currentToolCall = null;
          }
          break;
          
        case 'chatCompressed':
          console.log(`\n💾 Chat compressed: ${event.value.originalTokenCount} → ${event.value.newTokenCount} tokens`);
          break;
          
        case 'loopDetected':
          console.log('\n⚠️  Loop detected - stopping conversation');
          break;
          
        case 'maxSessionTurns':
          console.log('\n⏰ Maximum session turns reached');
          break;
      }
    }

    console.log('\n\n✅ Conversation completed');
    console.log('📊 Final response length:', responseText.length, 'characters');
    
    // 获取最终历史记录
    const history = client.getHistory();
    console.log('💬 Total conversation turns:', history.length);
    
  } catch (error) {
    console.error('❌ Complex conversation failed:', error);
  }
}
```

### 历史压缩演示

```typescript
// 历史压缩演示
async function compressionExample() {
  const client = new GeminiClient(config);
  await client.initialize(contentConfig);

  const signal = new AbortController().signal;
  let promptId = `compression-demo-${Date.now()}`;

  console.log('Building up conversation history...');

  // 构建长对话历史
  const topics = [
    "Explain TypeScript generics",
    "How do React hooks work?", 
    "What are the benefits of functional programming?",
    "Describe the difference between SQL and NoSQL databases",
    "How does machine learning work?",
    "What are microservices?",
    "Explain the concept of containerization",
    "What is the difference between synchronous and asynchronous programming?"
  ];

  for (const topic of topics) {
    console.log(`\n💬 Discussing: ${topic}`);
    
    const stream = client.sendMessageStream(
      [{ text: topic }],
      signal,
      promptId
    );

    let responseLength = 0;
    for await (const event of stream) {
      if (event.type === 'text') {
        responseLength += event.value.length;
      } else if (event.type === 'chatCompressed') {
        console.log(`📦 Compression occurred: ${event.value.originalTokenCount} → ${event.value.newTokenCount} tokens`);
      }
    }
    
    console.log(`   Response length: ${responseLength} characters`);
    
    const history = client.getHistory();
    console.log(`   History length: ${history.length} turns`);
    
    // 尝试手动压缩检查
    const compressionInfo = await client.tryCompressChat(promptId);
    if (compressionInfo) {
      console.log(`   Manual compression available: ${compressionInfo.originalTokenCount} → ${compressionInfo.newTokenCount} tokens`);
    }
  }

  console.log('\n✅ Compression demonstration completed');
}
```

## 错误处理和最佳实践

### 错误处理模式

```typescript
// 错误处理和重试示例
async function robustClientUsage() {
  const client = new GeminiClient(config);
  
  try {
    await client.initialize(contentConfig);
  } catch (error) {
    console.error('Initialization failed:', error);
    // 可能需要检查认证、网络连接等
    return;
  }

  const maxRetries = 3;
  let retryCount = 0;

  while (retryCount < maxRetries) {
    try {
      const signal = AbortSignal.timeout(60000); // 60秒超时
      const promptId = `retry-${retryCount}-${Date.now()}`;

      const stream = client.sendMessageStream(
        [{ text: 'Hello, world!' }],
        signal,
        promptId
      );

      for await (const event of stream) {
        // 处理事件
        if (event.type === 'error') {
          throw new Error(`Stream error: ${event.value}`);
        }
      }

      console.log('✅ Success!');
      break;

    } catch (error) {
      retryCount++;
      console.error(`❌ Attempt ${retryCount} failed:`, error);
      
      if (retryCount >= maxRetries) {
        console.error('Max retries exceeded');
        throw error;
      }

      // 指数退避
      const delay = Math.pow(2, retryCount) * 1000;
      console.log(`⏳ Retrying in ${delay}ms...`);
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }
}
```

### 资源管理

```typescript
// 资源管理最佳实践
class ManagedGeminiClient {
  private client: GeminiClient;
  private abortController: AbortController;

  constructor(config: Config) {
    this.client = new GeminiClient(config);
    this.abortController = new AbortController();
  }

  async initialize(contentConfig: ContentGeneratorConfig) {
    await this.client.initialize(contentConfig);
  }

  async sendMessage(message: string): Promise<string> {
    const promptId = `managed-${Date.now()}`;
    let responseText = '';

    try {
      const stream = this.client.sendMessageStream(
        [{ text: message }],
        this.abortController.signal,
        promptId
      );

      for await (const event of stream) {
        if (event.type === 'text') {
          responseText += event.value;
        }
      }

      return responseText;
    } catch (error) {
      if (error.name === 'AbortError') {
        console.log('Request was cancelled');
        return '';
      }
      throw error;
    }
  }

  // 优雅关闭
  dispose() {
    this.abortController.abort();
    // 清理其他资源
  }
}

// 使用示例
const managedClient = new ManagedGeminiClient(config);
try {
  await managedClient.initialize(contentConfig);
  const response = await managedClient.sendMessage('Hello!');
  console.log('Response:', response);
} finally {
  managedClient.dispose(); // 确保资源清理
}
```

### 性能监控

```typescript
// 性能监控包装器
class PerformanceMonitoredClient {
  private client: GeminiClient;
  private metrics = {
    totalRequests: 0,
    totalTokens: 0,
    totalTime: 0,
    compressions: 0,
    errors: 0,
  };

  constructor(config: Config) {
    this.client = new GeminiClient(config);
  }

  async initialize(contentConfig: ContentGeneratorConfig) {
    await this.client.initialize(contentConfig);
  }

  async sendMessageWithMetrics(message: string): Promise<{
    response: string;
    metrics: {
      duration: number;
      tokenCount: number;
      compressed: boolean;
    };
  }> {
    const startTime = Date.now();
    const promptId = `perf-${Date.now()}`;
    
    let responseText = '';
    let compressed = false;
    
    try {
      this.metrics.totalRequests++;
      
      const stream = this.client.sendMessageStream(
        [{ text: message }],
        AbortSignal.timeout(120000),
        promptId
      );

      for await (const event of stream) {
        switch (event.type) {
          case 'text':
            responseText += event.value;
            break;
          case 'chatCompressed':
            compressed = true;
            this.metrics.compressions++;
            break;
        }
      }

      const duration = Date.now() - startTime;
      this.metrics.totalTime += duration;

      // 估算令牌数（简化）
      const tokenCount = Math.ceil((message.length + responseText.length) / 4);
      this.metrics.totalTokens += tokenCount;

      return {
        response: responseText,
        metrics: {
          duration,
          tokenCount,
          compressed,
        },
      };

    } catch (error) {
      this.metrics.errors++;
      throw error;
    }
  }

  getPerformanceStats() {
    const avgTime = this.metrics.totalRequests > 0 
      ? this.metrics.totalTime / this.metrics.totalRequests 
      : 0;
    
    const avgTokens = this.metrics.totalRequests > 0
      ? this.metrics.totalTokens / this.metrics.totalRequests
      : 0;

    return {
      totalRequests: this.metrics.totalRequests,
      totalTokens: this.metrics.totalTokens,
      totalTime: this.metrics.totalTime,
      compressions: this.metrics.compressions,
      errors: this.metrics.errors,
      averageResponseTime: avgTime,
      averageTokensPerRequest: avgTokens,
      compressionRate: this.metrics.totalRequests > 0 
        ? (this.metrics.compressions / this.metrics.totalRequests) * 100 
        : 0,
      errorRate: this.metrics.totalRequests > 0
        ? (this.metrics.errors / this.metrics.totalRequests) * 100
        : 0,
    };
  }
}

// 使用性能监控
const perfClient = new PerformanceMonitoredClient(config);
await perfClient.initialize(contentConfig);

// 发送多个消息进行测试
const testMessages = [
  "What is TypeScript?",
  "Explain React components", 
  "How does async/await work?",
];

for (const message of testMessages) {
  const result = await perfClient.sendMessageWithMetrics(message);
  console.log(`Message: "${message}"`);
  console.log(`Duration: ${result.metrics.duration}ms`);
  console.log(`Tokens: ${result.metrics.tokenCount}`);
  console.log(`Compressed: ${result.metrics.compressed}`);
  console.log('---');
}

// 查看总体性能统计
const stats = perfClient.getPerformanceStats();
console.log('Performance Stats:', stats);
```

## 思考模式支持

### 思考模式检测

```typescript
function isThinkingSupported(model: string): boolean {
  if (model.startsWith('gemini-2.5')) return true;
  return false;
}
```

**功能**: 检测模型是否支持思考模式（内部推理过程可见）

**支持的模型**:
- `gemini-2.5-*` 系列模型

### 思考配置应用

在 `startChat` 方法中自动应用思考配置：

```typescript
const generateContentConfigWithThinking = isThinkingSupported(
  this.config.getModel(),
)
  ? {
      ...this.generateContentConfig,
      thinkingConfig: {
        includeThoughts: true,
      },
    }
  : this.generateContentConfig;
```

**思考模式的优势**:
- **透明推理**: 可以看到模型的思考过程
- **更好的解释**: 提供推理步骤和决策依据
- **调试友好**: 便于理解模型的决策逻辑

## 最佳实践总结

### 1. 初始化模式
```typescript
// ✅ 推荐：完整初始化检查
const client = new GeminiClient(config);
await client.initialize(contentConfig);

if (!client.isInitialized()) {
  throw new Error('Client initialization failed');
}
```

### 2. 错误处理
```typescript
// ✅ 推荐：包装错误处理
try {
  const stream = client.sendMessageStream(request, signal, promptId);
  // ... 处理流
} catch (error) {
  if (error.name === 'AbortError') {
    // 处理取消
  } else {
    // 处理其他错误
  }
}
```

### 3. 资源管理
```typescript
// ✅ 推荐：使用 AbortController
const controller = new AbortController();
const timeoutId = setTimeout(() => controller.abort(), 60000);

try {
  // ... 使用 controller.signal
} finally {
  clearTimeout(timeoutId);
}
```

### 4. 性能优化
```typescript
// ✅ 推荐：监控压缩和令牌使用
const compressionInfo = await client.tryCompressChat(promptId);
if (compressionInfo) {
  console.log(`压缩节省: ${compressionInfo.originalTokenCount - compressionInfo.newTokenCount} tokens`);
}
```

### 5. 状态维护
```typescript
// ✅ 推荐：定期检查状态
const history = client.getHistory();
if (history.length > 100) {
  // 考虑重置或压缩
  await client.tryCompressChat(promptId, true);
}
```

`GeminiClient` 是整个 Gemini CLI 系统的核心，提供了与 Google Gemini API 交互的所有必要功能，包括智能的资源管理、错误处理和性能优化。正确使用这个客户端是构建稳定、高效的 AI 应用程序的关键。