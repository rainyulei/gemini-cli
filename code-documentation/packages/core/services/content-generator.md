# ContentGenerator 内容生成服务文档

## 概述

`ContentGenerator` 是 Gemini CLI 的核心内容生成服务，负责与 Google Gemini AI 模型进行交互。它提供了统一的接口来处理内容生成、流式生成、令牌计数和内容嵌入等功能，支持多种认证方式和配置选项。

## 主要功能

- 多种 AI 模型内容生成支持
- 流式内容生成能力
- 令牌计数和使用量统计
- 内容嵌入向量生成
- 多种认证方式集成
- 灵活的配置管理系统
- 用户层级和权限管理

## 接口定义

### `ContentGenerator`

**功能**: 内容生成服务核心接口

```typescript
export interface ContentGenerator {
  generateContent(
    request: GenerateContentParameters,
  ): Promise<GenerateContentResponse>;

  generateContentStream(
    request: GenerateContentParameters,
  ): Promise<AsyncGenerator<GenerateContentResponse>>;

  countTokens(request: CountTokensParameters): Promise<CountTokensResponse>;

  embedContent(request: EmbedContentParameters): Promise<EmbedContentResponse>;

  userTier?: UserTierId;                  // 用户层级信息（可选）
}
```

**方法详情**:

- `generateContent`: 生成单次完整内容响应
- `generateContentStream`: 生成流式内容响应
- `countTokens`: 计算输入内容的令牌数量
- `embedContent`: 生成内容的嵌入向量
- `userTier`: 当前用户的服务层级

### `AuthType`

**功能**: 支持的认证类型枚举

```typescript
export enum AuthType {
  LOGIN_WITH_GOOGLE = 'oauth-personal',    // Google OAuth 登录
  USE_GEMINI = 'gemini-api-key',          // Gemini API 密钥
  USE_VERTEX_AI = 'vertex-ai',            // Vertex AI 认证
  CLOUD_SHELL = 'cloud-shell',            // Cloud Shell 环境
}
```

### `ContentGeneratorConfig`

**功能**: 内容生成器配置接口

```typescript
export type ContentGeneratorConfig = {
  model: string;                          // 使用的模型名称
  apiKey?: string;                        // API 密钥（可选）
  vertexai?: boolean;                     // 是否使用 Vertex AI
  authType?: AuthType | undefined;        // 认证类型
  proxy?: string | undefined;             // 代理设置
};
```

## 核心工厂函数

### `createContentGeneratorConfig`

**功能**: 创建内容生成器配置

```typescript
export function createContentGeneratorConfig(
  config: Config,
  authType: AuthType | undefined,
): ContentGeneratorConfig {
  const geminiApiKey = process.env.GEMINI_API_KEY || undefined;
  const googleApiKey = process.env.GOOGLE_API_KEY || undefined;
  const googleCloudProject = process.env.GOOGLE_CLOUD_PROJECT || undefined;
  const googleCloudLocation = process.env.GOOGLE_CLOUD_LOCATION || undefined;

  // 使用配置中的运行时模型，否则回退到默认模型
  const effectiveModel = config.getModel() || DEFAULT_GEMINI_MODEL;

  const contentGeneratorConfig: ContentGeneratorConfig = {
    model: effectiveModel,
    authType,
    proxy: config?.getProxy(),
  };

  // Google OAuth 或 Cloud Shell 认证
  if (
    authType === AuthType.LOGIN_WITH_GOOGLE ||
    authType === AuthType.CLOUD_SHELL
  ) {
    return contentGeneratorConfig;
  }

  // Gemini API 密钥认证
  if (authType === AuthType.USE_GEMINI && geminiApiKey) {
    contentGeneratorConfig.apiKey = geminiApiKey;
    contentGeneratorConfig.vertexai = false;
    
    // 验证有效模型
    getEffectiveModel(
      contentGeneratorConfig.apiKey,
      contentGeneratorConfig.model,
      contentGeneratorConfig.proxy,
    );

    return contentGeneratorConfig;
  }

  // Vertex AI 认证
  if (
    authType === AuthType.USE_VERTEX_AI &&
    (googleApiKey || (googleCloudProject && googleCloudLocation))
  ) {
    contentGeneratorConfig.apiKey = googleApiKey;
    contentGeneratorConfig.vertexai = true;

    return contentGeneratorConfig;
  }

  return contentGeneratorConfig;
}
```

**配置生成流程**:

1. **环境变量读取**: 读取相关的 API 密钥和配置信息
2. **模型选择**: 确定要使用的有效模型
3. **认证配置**: 根据认证类型设置相应的配置项
4. **验证**: 对配置进行必要的验证和检查

### `createContentGenerator`

**功能**: 创建内容生成器实例

```typescript
export async function createContentGenerator(
  config: ContentGeneratorConfig,
  gcConfig: Config,
  sessionId?: string,
): Promise<ContentGenerator> {
  const version = process.env.CLI_VERSION || process.version;
  const httpOptions = {
    headers: {
      'User-Agent': `GeminiCLI/${version} (${process.platform}; ${process.arch})`,
    },
  };

  // Code Assist 内容生成器（OAuth 和 Cloud Shell）
  if (
    config.authType === AuthType.LOGIN_WITH_GOOGLE ||
    config.authType === AuthType.CLOUD_SHELL
  ) {
    return createCodeAssistContentGenerator(
      httpOptions,
      config.authType,
      gcConfig,
      sessionId,
    );
  }

  // Google GenAI 内容生成器（API 密钥和 Vertex AI）
  if (
    config.authType === AuthType.USE_GEMINI ||
    config.authType === AuthType.USE_VERTEX_AI
  ) {
    const googleGenAI = new GoogleGenAI({
      apiKey: config.apiKey === '' ? undefined : config.apiKey,
      vertexai: config.vertexai,
      httpOptions,
    });

    return googleGenAI.models;
  }

  throw new Error(
    `Error creating contentGenerator: Unsupported authType: ${config.authType}`,
  );
}
```

**创建流程**:

1. **用户代理设置**: 构建包含版本和平台信息的 HTTP 头
2. **认证类型判断**: 根据认证类型选择相应的生成器
3. **实例创建**: 创建具体的内容生成器实例
4. **错误处理**: 处理不支持的认证类型

## 使用示例

### 基本内容生成

```typescript
import { createContentGenerator, createContentGeneratorConfig, AuthType } from './contentGenerator.js';
import { Config } from '../config/config.js';

// 创建配置
const config = new Config();
const authType = AuthType.USE_GEMINI;

// 生成内容生成器配置
const generatorConfig = createContentGeneratorConfig(config, authType);

// 创建内容生成器
const contentGenerator = await createContentGenerator(
  generatorConfig,
  config,
  'session-123'
);

// 生成内容
const response = await contentGenerator.generateContent({
  contents: [
    {
      role: 'user',
      parts: [{ text: 'Explain quantum computing in simple terms' }]
    }
  ],
  generationConfig: {
    maxOutputTokens: 1000,
    temperature: 0.7,
  }
});

console.log('Generated content:', response.candidates?.[0]?.content);
```

### 流式内容生成

```typescript
// 流式生成内容
const streamGenerator = await contentGenerator.generateContentStream({
  contents: [
    {
      role: 'user',
      parts: [{ text: 'Write a detailed explanation of machine learning' }]
    }
  ],
  generationConfig: {
    maxOutputTokens: 2000,
    temperature: 0.8,
  }
});

// 处理流式响应
for await (const chunk of streamGenerator) {
  const text = chunk.candidates?.[0]?.content?.parts?.[0]?.text;
  if (text) {
    process.stdout.write(text);
  }
}
```

### 令牌计数

```typescript
// 计算令牌数量
const tokenCount = await contentGenerator.countTokens({
  contents: [
    {
      role: 'user',
      parts: [{ text: 'This is a sample text for token counting' }]
    }
  ]
});

console.log('Token count:', tokenCount.totalTokens);
console.log('Input tokens:', tokenCount.candidatesTokenCount);
```

### 内容嵌入

```typescript
// 生成内容嵌入
const embedding = await contentGenerator.embedContent({
  content: {
    parts: [{ text: 'Generate embedding for this text' }]
  },
  taskType: 'RETRIEVAL_DOCUMENT'
});

console.log('Embedding vector:', embedding.embedding?.values);
console.log('Vector dimensions:', embedding.embedding?.values?.length);
```

## 认证方式详解

### Google OAuth 认证

```typescript
// 使用 Google OAuth 登录
const config = createContentGeneratorConfig(
  baseConfig,
  AuthType.LOGIN_WITH_GOOGLE
);

const generator = await createContentGenerator(config, baseConfig, sessionId);

// 此方式会使用 Code Assist 内容生成器
// 支持个人账户和企业账户
// 自动处理令牌刷新和权限管理
```

### API 密钥认证

```typescript
// 设置环境变量
process.env.GEMINI_API_KEY = 'your-api-key-here';

const config = createContentGeneratorConfig(
  baseConfig,
  AuthType.USE_GEMINI
);

const generator = await createContentGenerator(config, baseConfig);

// 直接使用 Gemini API
// 需要有效的 API 密钥
// 支持所有公开的 Gemini 模型
```

### Vertex AI 认证

```typescript
// 设置 Google Cloud 环境变量
process.env.GOOGLE_CLOUD_PROJECT = 'your-project-id';
process.env.GOOGLE_CLOUD_LOCATION = 'us-central1';
process.env.GOOGLE_API_KEY = 'your-service-account-key';

const config = createContentGeneratorConfig(
  baseConfig,
  AuthType.USE_VERTEX_AI
);

const generator = await createContentGenerator(config, baseConfig);

// 使用 Vertex AI 端点
// 支持企业级功能和控制
// 需要 Google Cloud 项目设置
```

### Cloud Shell 认证

```typescript
// 在 Google Cloud Shell 环境中运行
const config = createContentGeneratorConfig(
  baseConfig,
  AuthType.CLOUD_SHELL
);

const generator = await createContentGenerator(config, baseConfig);

// 自动使用 Cloud Shell 凭据
// 无需额外配置
// 适合 Cloud Shell 环境使用
```

## 高级功能

### 自定义内容生成器包装器

```typescript
class CustomContentGenerator implements ContentGenerator {
  constructor(
    private baseGenerator: ContentGenerator,
    private customConfig: {
      enableLogging?: boolean;
      rateLimiting?: boolean;
      caching?: boolean;
    }
  ) {}

  async generateContent(
    request: GenerateContentParameters,
  ): Promise<GenerateContentResponse> {
    if (this.customConfig.enableLogging) {
      console.log('Generating content with request:', request);
    }

    // 速率限制
    if (this.customConfig.rateLimiting) {
      await this.checkRateLimit();
    }

    // 缓存检查
    if (this.customConfig.caching) {
      const cached = await this.checkCache(request);
      if (cached) return cached;
    }

    const response = await this.baseGenerator.generateContent(request);

    // 缓存响应
    if (this.customConfig.caching) {
      await this.cacheResponse(request, response);
    }

    return response;
  }

  async generateContentStream(
    request: GenerateContentParameters,
  ): Promise<AsyncGenerator<GenerateContentResponse>> {
    // 流式生成的自定义逻辑
    const streamGenerator = await this.baseGenerator.generateContentStream(request);
    
    return this.wrapStreamGenerator(streamGenerator);
  }

  private async *wrapStreamGenerator(
    generator: AsyncGenerator<GenerateContentResponse>
  ): AsyncGenerator<GenerateContentResponse> {
    for await (const chunk of generator) {
      // 可以在这里添加流式处理逻辑
      if (this.customConfig.enableLogging) {
        console.log('Stream chunk received');
      }
      yield chunk;
    }
  }

  async countTokens(request: CountTokensParameters): Promise<CountTokensResponse> {
    return this.baseGenerator.countTokens(request);
  }

  async embedContent(request: EmbedContentParameters): Promise<EmbedContentResponse> {
    return this.baseGenerator.embedContent(request);
  }

  get userTier() {
    return this.baseGenerator.userTier;
  }

  private async checkRateLimit(): Promise<void> {
    // 实现速率限制逻辑
  }

  private async checkCache(request: GenerateContentParameters): Promise<GenerateContentResponse | null> {
    // 实现缓存检查逻辑
    return null;
  }

  private async cacheResponse(
    request: GenerateContentParameters,
    response: GenerateContentResponse
  ): Promise<void> {
    // 实现响应缓存逻辑
  }
}
```

### 批量内容生成

```typescript
class BatchContentGenerator {
  constructor(
    private contentGenerator: ContentGenerator,
    private options: {
      batchSize?: number;
      delayMs?: number;
      maxConcurrent?: number;
    } = {}
  ) {}

  async generateMultiple(
    requests: GenerateContentParameters[],
  ): Promise<GenerateContentResponse[]> {
    const { batchSize = 5, delayMs = 1000, maxConcurrent = 3 } = this.options;
    const results: GenerateContentResponse[] = [];

    // 分批处理
    for (let i = 0; i < requests.length; i += batchSize) {
      const batch = requests.slice(i, i + batchSize);
      
      // 限制并发数
      const concurrent = Math.min(batch.length, maxConcurrent);
      const batches = [];
      
      for (let j = 0; j < batch.length; j += concurrent) {
        const concurrentBatch = batch.slice(j, j + concurrent);
        batches.push(concurrentBatch);
      }

      for (const concurrentBatch of batches) {
        const batchPromises = concurrentBatch.map(request =>
          this.contentGenerator.generateContent(request)
        );

        const batchResults = await Promise.allSettled(batchPromises);
        
        batchResults.forEach((result, index) => {
          if (result.status === 'fulfilled') {
            results.push(result.value);
          } else {
            console.error(`Request failed:`, result.reason);
            // 可以添加默认的错误响应
          }
        });

        // 批次间延迟
        if (i + batchSize < requests.length) {
          await new Promise(resolve => setTimeout(resolve, delayMs));
        }
      }
    }

    return results;
  }

  async generateWithRetry(
    request: GenerateContentParameters,
    maxRetries: number = 3,
    retryDelay: number = 1000,
  ): Promise<GenerateContentResponse> {
    let lastError: Error | null = null;

    for (let attempt = 1; attempt <= maxRetries; attempt++) {
      try {
        return await this.contentGenerator.generateContent(request);
      } catch (error) {
        lastError = error instanceof Error ? error : new Error(String(error));
        
        if (attempt < maxRetries) {
          console.warn(`Attempt ${attempt} failed, retrying in ${retryDelay}ms...`);
          await new Promise(resolve => setTimeout(resolve, retryDelay * attempt));
        }
      }
    }

    throw new Error(`All ${maxRetries} attempts failed. Last error: ${lastError?.message}`);
  }
}
```

### 内容生成监控

```typescript
class MonitoredContentGenerator implements ContentGenerator {
  private metrics = {
    totalRequests: 0,
    successfulRequests: 0,
    failedRequests: 0,
    totalTokensGenerated: 0,
    averageResponseTime: 0,
  };

  constructor(private baseGenerator: ContentGenerator) {}

  async generateContent(
    request: GenerateContentParameters,
  ): Promise<GenerateContentResponse> {
    const startTime = Date.now();
    this.metrics.totalRequests++;

    try {
      const response = await this.baseGenerator.generateContent(request);
      
      // 记录成功指标
      this.metrics.successfulRequests++;
      this.updateResponseTime(Date.now() - startTime);
      
      // 估算生成的令牌数
      const tokens = this.estimateTokenCount(response);
      this.metrics.totalTokensGenerated += tokens;

      return response;
    } catch (error) {
      this.metrics.failedRequests++;
      throw error;
    }
  }

  async generateContentStream(
    request: GenerateContentParameters,
  ): Promise<AsyncGenerator<GenerateContentResponse>> {
    const generator = await this.baseGenerator.generateContentStream(request);
    return this.monitorStream(generator);
  }

  private async *monitorStream(
    generator: AsyncGenerator<GenerateContentResponse>
  ): AsyncGenerator<GenerateContentResponse> {
    let tokenCount = 0;
    
    try {
      for await (const chunk of generator) {
        tokenCount += this.estimateTokenCount(chunk);
        yield chunk;
      }
      
      this.metrics.successfulRequests++;
      this.metrics.totalTokensGenerated += tokenCount;
    } catch (error) {
      this.metrics.failedRequests++;
      throw error;
    }
  }

  async countTokens(request: CountTokensParameters): Promise<CountTokensResponse> {
    return this.baseGenerator.countTokens(request);
  }

  async embedContent(request: EmbedContentParameters): Promise<EmbedContentResponse> {
    return this.baseGenerator.embedContent(request);
  }

  get userTier() {
    return this.baseGenerator.userTier;
  }

  getMetrics() {
    return {
      ...this.metrics,
      successRate: this.metrics.totalRequests > 0 
        ? this.metrics.successfulRequests / this.metrics.totalRequests 
        : 0,
      failureRate: this.metrics.totalRequests > 0 
        ? this.metrics.failedRequests / this.metrics.totalRequests 
        : 0,
    };
  }

  private updateResponseTime(responseTime: number): void {
    const currentAverage = this.metrics.averageResponseTime;
    const count = this.metrics.successfulRequests;
    
    this.metrics.averageResponseTime = 
      (currentAverage * (count - 1) + responseTime) / count;
  }

  private estimateTokenCount(response: GenerateContentResponse): number {
    // 简单的令牌估算，实际应用中可能需要更精确的计算
    const text = response.candidates?.[0]?.content?.parts?.[0]?.text || '';
    return Math.ceil(text.length / 4); // 粗略估算：4 字符 ≈ 1 令牌
  }
}
```

## 错误处理

### 常见错误场景

```typescript
// API 密钥相关错误
Error: Invalid API key provided
Error: API key has been revoked
Error: API quota exceeded

// 模型相关错误
Error: Model not found: invalid-model-name
Error: Model access denied for current user tier

// 网络相关错误
Error: Network request failed
Error: Connection timeout
Error: Service temporarily unavailable

// 内容相关错误
Error: Content too long for model context
Error: Content violates safety policies
Error: Invalid content format
```

### 错误恢复策略

```typescript
class RobustContentGenerator implements ContentGenerator {
  constructor(
    private baseGenerator: ContentGenerator,
    private fallbackGenerator?: ContentGenerator,
  ) {}

  async generateContent(
    request: GenerateContentParameters,
  ): Promise<GenerateContentResponse> {
    try {
      return await this.baseGenerator.generateContent(request);
    } catch (error) {
      return this.handleGenerationError(error, request);
    }
  }

  private async handleGenerationError(
    error: unknown,
    request: GenerateContentParameters,
  ): Promise<GenerateContentResponse> {
    const errorMessage = error instanceof Error ? error.message : String(error);

    // API 限额错误 - 使用备用生成器
    if (errorMessage.includes('quota exceeded') && this.fallbackGenerator) {
      console.warn('Primary generator quota exceeded, switching to fallback...');
      return this.fallbackGenerator.generateContent(request);
    }

    // 内容长度错误 - 尝试分段处理
    if (errorMessage.includes('too long')) {
      console.warn('Content too long, attempting to split...');
      return this.handleLongContent(request);
    }

    // 安全策略错误 - 清理内容后重试
    if (errorMessage.includes('safety') || errorMessage.includes('policy')) {
      console.warn('Content safety issue, attempting to sanitize...');
      const sanitizedRequest = this.sanitizeContent(request);
      return this.baseGenerator.generateContent(sanitizedRequest);
    }

    // 网络错误 - 重试
    if (errorMessage.includes('network') || errorMessage.includes('timeout')) {
      console.warn('Network error, retrying...');
      await new Promise(resolve => setTimeout(resolve, 2000));
      return this.baseGenerator.generateContent(request);
    }

    // 其他错误 - 抛出原始错误
    throw error;
  }

  private async handleLongContent(
    request: GenerateContentParameters,
  ): Promise<GenerateContentResponse> {
    // 简化内容或分段处理
    const simplifiedRequest = {
      ...request,
      contents: request.contents.map(content => ({
        ...content,
        parts: content.parts.map(part => {
          if (part.text && part.text.length > 10000) {
            return { ...part, text: part.text.substring(0, 10000) + '...' };
          }
          return part;
        })
      }))
    };

    return this.baseGenerator.generateContent(simplifiedRequest);
  }

  private sanitizeContent(
    request: GenerateContentParameters,
  ): GenerateContentParameters {
    // 移除可能触发安全策略的内容
    const sanitizedContents = request.contents.map(content => ({
      ...content,
      parts: content.parts.map(part => {
        if (part.text) {
          // 简单的内容清理逻辑
          let cleanText = part.text
            .replace(/\b(violence|harmful|inappropriate)\b/gi, '[content filtered]')
            .replace(/[^\w\s.,!?-]/g, ''); // 移除特殊字符
          
          return { ...part, text: cleanText };
        }
        return part;
      })
    }));

    return { ...request, contents: sanitizedContents };
  }

  // 实现其他接口方法...
  async generateContentStream(request: GenerateContentParameters) {
    return this.baseGenerator.generateContentStream(request);
  }

  async countTokens(request: CountTokensParameters) {
    return this.baseGenerator.countTokens(request);
  }

  async embedContent(request: EmbedContentParameters) {
    return this.baseGenerator.embedContent(request);
  }

  get userTier() {
    return this.baseGenerator.userTier;
  }
}
```

## 配置管理

### 环境变量配置

```typescript
// 环境变量设置示例
export const CONTENT_GENERATOR_ENV = {
  // Gemini API 配置
  GEMINI_API_KEY: process.env.GEMINI_API_KEY,
  GEMINI_MODEL: process.env.GEMINI_MODEL || 'gemini-1.5-pro',
  
  // Google Cloud 配置
  GOOGLE_API_KEY: process.env.GOOGLE_API_KEY,
  GOOGLE_CLOUD_PROJECT: process.env.GOOGLE_CLOUD_PROJECT,
  GOOGLE_CLOUD_LOCATION: process.env.GOOGLE_CLOUD_LOCATION || 'us-central1',
  
  // 代理配置
  HTTPS_PROXY: process.env.HTTPS_PROXY,
  HTTP_PROXY: process.env.HTTP_PROXY,
  
  // 调试配置
  GEMINI_DEBUG: process.env.GEMINI_DEBUG === 'true',
  CLI_VERSION: process.env.CLI_VERSION,
};

// 配置验证函数
export function validateContentGeneratorEnv(): {
  isValid: boolean;
  errors: string[];
} {
  const errors: string[] = [];

  // 检查必需的环境变量
  if (!CONTENT_GENERATOR_ENV.GEMINI_API_KEY && 
      !CONTENT_GENERATOR_ENV.GOOGLE_API_KEY) {
    errors.push('Either GEMINI_API_KEY or GOOGLE_API_KEY must be set');
  }

  if (CONTENT_GENERATOR_ENV.GOOGLE_CLOUD_PROJECT && 
      !CONTENT_GENERATOR_ENV.GOOGLE_CLOUD_LOCATION) {
    errors.push('GOOGLE_CLOUD_LOCATION must be set when using GOOGLE_CLOUD_PROJECT');
  }

  return {
    isValid: errors.length === 0,
    errors,
  };
}
```

### 配置文件管理

```typescript
interface ContentGeneratorSettings {
  defaultModel: string;
  authType: AuthType;
  proxy?: string;
  requestTimeout: number;
  maxRetries: number;
  retryDelay: number;
  enableLogging: boolean;
  rateLimiting: {
    enabled: boolean;
    requestsPerMinute: number;
  };
}

class ContentGeneratorConfigManager {
  private settings: ContentGeneratorSettings;

  constructor(configPath?: string) {
    this.settings = this.loadSettings(configPath);
  }

  private loadSettings(configPath?: string): ContentGeneratorSettings {
    const defaultSettings: ContentGeneratorSettings = {
      defaultModel: 'gemini-1.5-pro',
      authType: AuthType.USE_GEMINI,
      requestTimeout: 30000,
      maxRetries: 3,
      retryDelay: 1000,
      enableLogging: false,
      rateLimiting: {
        enabled: false,
        requestsPerMinute: 60,
      },
    };

    if (configPath && fs.existsSync(configPath)) {
      try {
        const fileContent = fs.readFileSync(configPath, 'utf-8');
        const userSettings = JSON.parse(fileContent);
        return { ...defaultSettings, ...userSettings };
      } catch (error) {
        console.warn('Failed to load config file, using defaults:', error);
      }
    }

    return defaultSettings;
  }

  getSettings(): ContentGeneratorSettings {
    return { ...this.settings };
  }

  updateSettings(updates: Partial<ContentGeneratorSettings>): void {
    this.settings = { ...this.settings, ...updates };
  }

  saveSettings(configPath: string): void {
    try {
      fs.writeFileSync(configPath, JSON.stringify(this.settings, null, 2));
    } catch (error) {
      console.error('Failed to save config file:', error);
    }
  }
}
```

## 最佳实践

### 性能优化

```typescript
// ✅ 推荐：使用适当的生成配置
const optimizedRequest: GenerateContentParameters = {
  contents: [{ role: 'user', parts: [{ text: prompt }] }],
  generationConfig: {
    maxOutputTokens: 1000,        // 根据需求设置合理的令牌限制
    temperature: 0.7,             // 适中的创造性
    topP: 0.8,                   // 控制输出质量
    topK: 40,                    // 限制候选词汇
    stopSequences: ['END'],       // 设置停止序列
  },
  safetySettings: [             // 配置安全设置
    {
      category: 'HARM_CATEGORY_HARASSMENT',
      threshold: 'BLOCK_MEDIUM_AND_ABOVE'
    }
  ]
};

// ✅ 推荐：使用流式生成处理长内容
async function generateLongContent(prompt: string) {
  const streamGenerator = await contentGenerator.generateContentStream({
    contents: [{ role: 'user', parts: [{ text: prompt }] }],
    generationConfig: { maxOutputTokens: 4000 }
  });

  let fullContent = '';
  for await (const chunk of streamGenerator) {
    const text = chunk.candidates?.[0]?.content?.parts?.[0]?.text || '';
    fullContent += text;
    
    // 实时处理内容块
    processContentChunk(text);
  }

  return fullContent;
}
```

### 错误处理

```typescript
// ✅ 推荐：全面的错误处理
async function safeGenerateContent(
  request: GenerateContentParameters
): Promise<GenerateContentResponse | null> {
  try {
    return await contentGenerator.generateContent(request);
  } catch (error) {
    if (error instanceof Error) {
      // 记录具体错误信息
      console.error('Content generation failed:', {
        message: error.message,
        request: JSON.stringify(request, null, 2),
        timestamp: new Date().toISOString(),
      });

      // 根据错误类型进行处理
      if (error.message.includes('quota')) {
        // 处理配额错误
        return null;
      }
      
      if (error.message.includes('safety')) {
        // 处理安全策略错误
        console.warn('Content filtered by safety policies');
        return null;
      }
    }
    
    // 重新抛出未处理的错误
    throw error;
  }
}
```

### 资源管理

```typescript
// ✅ 推荐：实现连接池和资源管理
class ManagedContentGenerator implements ContentGenerator {
  private connectionPool: ContentGenerator[] = [];
  private currentIndex = 0;

  constructor(
    private configs: ContentGeneratorConfig[],
    private maxConnections: number = 5
  ) {
    this.initializePool();
  }

  private async initializePool(): Promise<void> {
    for (let i = 0; i < Math.min(this.configs.length, this.maxConnections); i++) {
      const generator = await createContentGenerator(this.configs[i], baseConfig);
      this.connectionPool.push(generator);
    }
  }

  private getNextGenerator(): ContentGenerator {
    const generator = this.connectionPool[this.currentIndex];
    this.currentIndex = (this.currentIndex + 1) % this.connectionPool.length;
    return generator;
  }

  async generateContent(request: GenerateContentParameters): Promise<GenerateContentResponse> {
    const generator = this.getNextGenerator();
    return generator.generateContent(request);
  }

  // 实现其他方法...
}
```