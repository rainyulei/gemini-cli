# WebSearchTool 网络搜索工具文档

## 概述

`WebSearchTool` 是 Gemini CLI 的网络搜索工具，通过 Google Search（via Gemini API）进行网络搜索并返回结果。它提供了智能的信息检索能力，包括搜索结果、来源引用和支撑证据，是获取最新网络信息的强大工具。

## 主要功能

- 基于 Gemini API 的 Google 搜索集成
- 智能搜索结果处理和格式化
- 自动引用标记和来源链接
- 支撑证据和可信度评分
- 结构化的搜索结果展示
- 错误处理和异常恢复
- 可中断的异步搜索操作

## 接口定义

### `WebSearchToolParams`

**功能**: WebSearchTool 工具参数接口

```typescript
export interface WebSearchToolParams {
  query: string;                      // 搜索查询（必需）
}
```

**参数详情**:

- `query`: 搜索查询字符串，用于在网络上查找信息

### `WebSearchToolResult`

**功能**: 扩展的搜索结果接口

```typescript
export interface WebSearchToolResult extends ToolResult {
  sources?: GroundingMetadata extends { groundingChunks: GroundingChunkItem[] }
    ? GroundingMetadata['groundingChunks']
    : GroundingChunkItem[];
}
```

### 支撑数据结构

#### `GroundingChunkWeb`

**功能**: 网络来源信息结构

```typescript
interface GroundingChunkWeb {
  uri?: string;                       // 网页链接
  title?: string;                     // 网页标题
}
```

#### `GroundingChunkItem`

**功能**: 引用来源条目结构

```typescript
interface GroundingChunkItem {
  web?: GroundingChunkWeb;           // 网络来源信息
  // 未来可能扩展其他属性
}
```

#### `GroundingSupportSegment`

**功能**: 支撑文本段落结构

```typescript
interface GroundingSupportSegment {
  startIndex: number;                 // 段落开始位置
  endIndex: number;                   // 段落结束位置
  text?: string;                      // 段落文本（可选）
}
```

#### `GroundingSupportItem`

**功能**: 支撑证据条目结构

```typescript
interface GroundingSupportItem {
  segment?: GroundingSupportSegment;          // 支撑文本段落
  groundingChunkIndices?: number[];           // 引用来源索引数组
  confidenceScores?: number[];                // 可信度评分（可选）
}
```

## 核心方法

### 构造函数

#### `constructor(config: Config)`

**功能**: 创建 WebSearchTool 实例

```typescript
constructor(private readonly config: Config) {
  super(
    WebSearchTool.Name,
    'GoogleSearch',
    'Performs a web search using Google Search (via the Gemini API)...',
    Icon.Globe,
    parameterSchema,
  );
}
```

**工具特性**:

- 集成 Gemini API 的 Google 搜索功能
- 自动引用标记和来源处理
- 结构化搜索结果格式化
- 异步操作和错误处理

### 参数验证

#### `validateParams(params: WebSearchToolParams): string | null`

**功能**: 验证搜索参数

```typescript
validateParams(params: WebSearchToolParams): string | null {
  const errors = SchemaValidator.validate(this.schema.parameters, params);
  if (errors) {
    return errors;
  }

  if (!params.query || params.query.trim() === '') {
    return "The 'query' parameter cannot be empty.";
  }
  return null;
}
```

**验证项目**:

1. **Schema 验证**: 验证参数结构和类型
2. **查询内容**: 确保搜索查询非空且有意义

### 描述生成

#### `getDescription(params: WebSearchToolParams): string`

**功能**: 生成搜索操作描述

```typescript
getDescription(params: WebSearchToolParams): string {
  return `Searching the web for: "${params.query}"`;
}
```

### 主要执行方法

#### `async execute(params: WebSearchToolParams, signal: AbortSignal): Promise<WebSearchToolResult>`

**功能**: 执行网络搜索操作

**执行流程**:

1. **参数验证**: 验证搜索查询参数
2. **API 调用**: 通过 Gemini API 执行搜索
3. **结果处理**: 提取和格式化搜索结果
4. **引用标记**: 添加来源引用和支撑证据
5. **结果组装**: 构建完整的搜索结果

```typescript
async execute(
  params: WebSearchToolParams,
  signal: AbortSignal,
): Promise<WebSearchToolResult> {
  const validationError = this.validateParams(params);
  if (validationError) {
    return {
      llmContent: `Error: Invalid parameters. Reason: ${validationError}`,
      returnDisplay: validationError,
    };
  }

  const geminiClient = this.config.getGeminiClient();

  try {
    // 执行 Gemini API 搜索
    const response = await geminiClient.generateContent(
      [{ role: 'user', parts: [{ text: params.query }] }],
      { tools: [{ googleSearch: {} }] },
      signal,
    );

    // 提取响应内容和元数据
    const responseText = getResponseText(response);
    const groundingMetadata = response.candidates?.[0]?.groundingMetadata;
    const sources = groundingMetadata?.groundingChunks as GroundingChunkItem[] | undefined;
    const groundingSupports = groundingMetadata?.groundingSupports as GroundingSupportItem[] | undefined;

    // 检查搜索结果
    if (!responseText || !responseText.trim()) {
      return {
        llmContent: `No search results found for query: "${params.query}"`,
        returnDisplay: 'No information found.',
      };
    }

    let modifiedResponseText = responseText;
    const sourceListFormatted: string[] = [];

    // 处理来源信息
    if (sources && sources.length > 0) {
      sources.forEach((source: GroundingChunkItem, index: number) => {
        const title = source.web?.title || 'Untitled';
        const uri = source.web?.uri || 'No URI';
        sourceListFormatted.push(`[${index + 1}] ${title} (${uri})`);
      });

      // 添加引用标记
      if (groundingSupports && groundingSupports.length > 0) {
        const insertions: Array<{ index: number; marker: string }> = [];
        
        groundingSupports.forEach((support: GroundingSupportItem) => {
          if (support.segment && support.groundingChunkIndices) {
            const citationMarker = support.groundingChunkIndices
              .map((chunkIndex: number) => `[${chunkIndex + 1}]`)
              .join('');
            insertions.push({
              index: support.segment.endIndex,
              marker: citationMarker,
            });
          }
        });

        // 按索引降序排序以避免索引偏移
        insertions.sort((a, b) => b.index - a.index);

        const responseChars = modifiedResponseText.split('');
        insertions.forEach((insertion) => {
          responseChars.splice(insertion.index, 0, insertion.marker);
        });
        modifiedResponseText = responseChars.join('');
      }

      // 添加来源列表
      if (sourceListFormatted.length > 0) {
        modifiedResponseText += '\n\nSources:\n' + sourceListFormatted.join('\n');
      }
    }

    return {
      llmContent: `Web search results for "${params.query}":\n\n${modifiedResponseText}`,
      returnDisplay: `Search results for "${params.query}" returned.`,
      sources,
    };
  } catch (error: unknown) {
    const errorMessage = `Error during web search for query "${params.query}": ${getErrorMessage(error)}`;
    console.error(errorMessage, error);
    return {
      llmContent: `Error: ${errorMessage}`,
      returnDisplay: 'Error performing web search.',
    };
  }
}
```

## 使用示例

### 基本网络搜索

```typescript
const webSearchTool = new WebSearchTool(config);

// 技术问题搜索
const result = await webSearchTool.execute({
  query: "TypeScript best practices 2024"
}, new AbortController().signal);

// 新闻和时事搜索
const result = await webSearchTool.execute({
  query: "latest JavaScript framework trends"
}, new AbortController().signal);
```

### 技术文档搜索

```typescript
// API 文档搜索
const result = await webSearchTool.execute({
  query: "React 18 concurrent features documentation"
});

// 库的使用方法搜索
const result = await webSearchTool.execute({
  query: "how to use Prisma with PostgreSQL"
});

// 错误解决方案搜索
const result = await webSearchTool.execute({
  query: "TypeScript error TS2307 Cannot find module"
});
```

### 市场和趋势研究

```typescript
// 技术趋势搜索
const result = await webSearchTool.execute({
  query: "web development trends 2024 survey"
});

// 工具比较搜索
const result = await webSearchTool.execute({
  query: "Vite vs Webpack performance comparison 2024"
});

// 最佳实践搜索
const result = await webSearchTool.execute({
  query: "Node.js security best practices"
});
```

### 学习和教程搜索

```typescript
// 教程搜索
const result = await webSearchTool.execute({
  query: "advanced React hooks tutorial with examples"
});

// 概念解释搜索
const result = await webSearchTool.execute({
  query: "what is serverless architecture advantages disadvantages"
});

// 实践指南搜索
const result = await webSearchTool.execute({
  query: "Docker containerization guide for Node.js applications"
});
```

## 搜索结果格式

### 标准结果格式

```
Web search results for "TypeScript best practices 2024":

TypeScript has evolved significantly in 2024, with several key best practices emerging for modern development[1][2]. Here are the most important guidelines:

1. **Use Strict Mode**: Always enable strict mode in your tsconfig.json[1]
2. **Leverage Type Guards**: Implement proper type guards for runtime validation[2][3]
3. **Embrace Utility Types**: Use built-in utility types like Partial, Required, and Pick[1]

Sources:
[1] TypeScript Best Practices 2024 (https://typescript-guide.com/best-practices)
[2] Modern TypeScript Development (https://dev.to/typescript-2024)
[3] Advanced TypeScript Patterns (https://typescript-handbook.org/advanced)
```

### 结果组件说明

#### 1. 主要内容
- **搜索结果文本**: AI 处理后的结构化信息
- **引用标记**: `[1][2][3]` 形式的来源引用
- **内容组织**: 清晰的段落和列表结构

#### 2. 来源列表
```
Sources:
[1] 标题 (URL)
[2] 标题 (URL)
[3] 标题 (URL)
```

#### 3. 返回值结构
```typescript
{
  llmContent: "完整的搜索结果内容",
  returnDisplay: "简短的状态描述",
  sources: [
    {
      web: {
        title: "网页标题",
        uri: "https://example.com"
      }
    }
  ]
}
```

## 高级功能

### 带取消支持的搜索

```typescript
class CancellableWebSearchTool extends WebSearchTool {
  async executeWithTimeout(
    params: WebSearchToolParams,
    timeoutMs: number = 30000
  ): Promise<WebSearchToolResult> {
    const controller = new AbortController();
    
    // 设置超时
    const timeoutId = setTimeout(() => {
      controller.abort();
    }, timeoutMs);
    
    try {
      const result = await this.execute(params, controller.signal);
      clearTimeout(timeoutId);
      return result;
    } catch (error) {
      clearTimeout(timeoutId);
      if (controller.signal.aborted) {
        return {
          llmContent: `Search timeout after ${timeoutMs}ms for query: "${params.query}"`,
          returnDisplay: 'Search timeout',
        };
      }
      throw error;
    }
  }
}
```

### 搜索结果缓存

```typescript
class CachedWebSearchTool extends WebSearchTool {
  private cache = new Map<string, { result: WebSearchToolResult, timestamp: number }>();
  private cacheTimeout = 10 * 60 * 1000; // 10 分钟

  async execute(
    params: WebSearchToolParams,
    signal: AbortSignal
  ): Promise<WebSearchToolResult> {
    const cacheKey = params.query.toLowerCase().trim();
    const cached = this.cache.get(cacheKey);
    
    if (cached && Date.now() - cached.timestamp < this.cacheTimeout) {
      return {
        ...cached.result,
        returnDisplay: `${cached.result.returnDisplay} (cached)`
      };
    }
    
    const result = await super.execute(params, signal);
    this.cache.set(cacheKey, { result, timestamp: Date.now() });
    
    return result;
  }
  
  clearCache(): void {
    this.cache.clear();
  }
  
  getCacheStats(): { size: number, oldest: number } {
    if (this.cache.size === 0) {
      return { size: 0, oldest: 0 };
    }
    
    const timestamps = Array.from(this.cache.values()).map(entry => entry.timestamp);
    return {
      size: this.cache.size,
      oldest: Math.min(...timestamps)
    };
  }
}
```

### 批量搜索功能

```typescript
class BatchWebSearchTool extends WebSearchTool {
  async executeMultiple(
    queries: string[],
    options: {
      maxConcurrent?: number;
      delayMs?: number;
    } = {}
  ): Promise<WebSearchToolResult[]> {
    const { maxConcurrent = 3, delayMs = 1000 } = options;
    const results: WebSearchToolResult[] = [];
    
    // 分批处理以避免过载
    for (let i = 0; i < queries.length; i += maxConcurrent) {
      const batch = queries.slice(i, i + maxConcurrent);
      
      const batchPromises = batch.map(query =>
        this.execute({ query }, new AbortController().signal)
      );
      
      const batchResults = await Promise.allSettled(batchPromises);
      
      batchResults.forEach((result, index) => {
        if (result.status === 'fulfilled') {
          results.push(result.value);
        } else {
          results.push({
            llmContent: `Error searching for "${batch[index]}": ${result.reason}`,
            returnDisplay: 'Search failed',
          });
        }
      });
      
      // 批次间延迟
      if (i + maxConcurrent < queries.length) {
        await new Promise(resolve => setTimeout(resolve, delayMs));
      }
    }
    
    return results;
  }
}

// 使用示例
const batchSearch = new BatchWebSearchTool(config);
const queries = [
  "React 18 new features",
  "TypeScript 5.0 updates", 
  "Node.js performance tips"
];

const results = await batchSearch.executeMultiple(queries, {
  maxConcurrent: 2,
  delayMs: 500
});
```

### 搜索结果分析器

```typescript
class AnalyzedWebSearchTool extends WebSearchTool {
  async executeWithAnalysis(
    params: WebSearchToolParams,
    signal: AbortSignal
  ): Promise<WebSearchToolResult & { analysis: SearchAnalysis }> {
    const result = await super.execute(params, signal);
    
    const analysis = this.analyzeSearchResult(result);
    
    return {
      ...result,
      analysis
    };
  }
  
  private analyzeSearchResult(result: WebSearchToolResult): SearchAnalysis {
    const analysis: SearchAnalysis = {
      sourceCount: 0,
      avgConfidence: 0,
      domainDistribution: {},
      contentLength: 0,
      hasRecentSources: false
    };
    
    if (typeof result.llmContent === 'string') {
      analysis.contentLength = result.llmContent.length;
    }
    
    if (result.sources) {
      analysis.sourceCount = result.sources.length;
      
      // 分析域名分布
      result.sources.forEach(source => {
        if (source.web?.uri) {
          try {
            const domain = new URL(source.web.uri).hostname;
            analysis.domainDistribution[domain] = 
              (analysis.domainDistribution[domain] || 0) + 1;
          } catch (error) {
            // 忽略无效 URL
          }
        }
      });
    }
    
    return analysis;
  }
}

interface SearchAnalysis {
  sourceCount: number;
  avgConfidence: number;
  domainDistribution: Record<string, number>;
  contentLength: number;
  hasRecentSources: boolean;
}
```

## 错误处理

### 常见错误场景

#### API 调用错误

```typescript
// 网络连接错误
Error: Error during web search for query "...": Network request failed

// API 限额错误
Error: Error during web search for query "...": Rate limit exceeded

// 认证错误
Error: Error during web search for query "...": Invalid API key
```

#### 参数验证错误

```typescript
// 空查询
Error: Invalid parameters. Reason: The 'query' parameter cannot be empty.

// 无效参数类型
Error: Invalid parameters. Reason: Parameter validation failed
```

#### 搜索结果错误

```typescript
// 无搜索结果
No search results found for query: "very specific obscure term"

// 结果处理错误
Error: Failed to process search results
```

### 错误恢复策略

```typescript
class RobustWebSearchTool extends WebSearchTool {
  private maxRetries = 3;
  private retryDelayMs = 1000;
  
  async execute(
    params: WebSearchToolParams,
    signal: AbortSignal
  ): Promise<WebSearchToolResult> {
    let lastError: Error | null = null;
    
    for (let attempt = 1; attempt <= this.maxRetries; attempt++) {
      try {
        return await super.execute(params, signal);
      } catch (error) {
        lastError = error instanceof Error ? error : new Error(String(error));
        
        // 检查是否为可重试的错误
        if (!this.isRetryableError(lastError) || attempt === this.maxRetries) {
          break;
        }
        
        // 指数退避延迟
        const delay = this.retryDelayMs * Math.pow(2, attempt - 1);
        await new Promise(resolve => setTimeout(resolve, delay));
        
        console.warn(`Search attempt ${attempt} failed, retrying in ${delay}ms...`);
      }
    }
    
    // 所有重试都失败，尝试降级方案
    return this.fallbackSearch(params, lastError!);
  }
  
  private isRetryableError(error: Error): boolean {
    const retryableMessages = [
      'network request failed',
      'timeout',
      'rate limit',
      'service unavailable'
    ];
    
    const errorMessage = error.message.toLowerCase();
    return retryableMessages.some(msg => errorMessage.includes(msg));
  }
  
  private async fallbackSearch(
    params: WebSearchToolParams,
    originalError: Error
  ): Promise<WebSearchToolResult> {
    // 尝试简化查询
    const simplifiedQuery = this.simplifyQuery(params.query);
    
    if (simplifiedQuery !== params.query) {
      try {
        const result = await super.execute({ query: simplifiedQuery }, new AbortController().signal);
        return {
          ...result,
          llmContent: `Note: Used simplified query due to errors.\n\n${result.llmContent}`,
          returnDisplay: `Partial results (simplified query): ${result.returnDisplay}`
        };
      } catch (fallbackError) {
        // 降级方案也失败
      }
    }
    
    // 最终降级：返回错误信息
    return {
      llmContent: `Unable to complete web search for "${params.query}". Error: ${originalError.message}`,
      returnDisplay: `Search failed: ${originalError.message}`,
    };
  }
  
  private simplifyQuery(query: string): string {
    // 移除特殊字符和复杂语法
    return query
      .replace(/[^\w\s]/g, ' ')
      .replace(/\s+/g, ' ')
      .trim()
      .split(' ')
      .slice(0, 5) // 限制为前5个词
      .join(' ');
  }
}
```

## 集成特性

### 与其他工具协作

#### 与 MemoryTool 配合

```typescript
// 搜索并记住重要信息
const searchResult = await webSearchTool.execute({
  query: "React 18 concurrent features best practices"
});

// 提取关键信息并保存到记忆中
if (searchResult.llmContent) {
  const keyInsights = extractKeyInsights(searchResult.llmContent);
  for (const insight of keyInsights) {
    await memoryTool.execute({ fact: insight });
  }
}

function extractKeyInsights(content: string): string[] {
  // 简单的关键信息提取逻辑
  const insights: string[] = [];
  const lines = content.split('\n');
  
  lines.forEach(line => {
    if (line.includes('best practice') || line.includes('recommended')) {
      insights.push(line.trim());
    }
  });
  
  return insights;
}
```

#### 与文档工具配合

```typescript
// 搜索并生成文档
class DocumentationSearchTool extends WebSearchTool {
  async searchAndDocument(
    topic: string,
    documentPath: string
  ): Promise<void> {
    const searchResult = await this.execute({
      query: `${topic} documentation guide tutorial`
    }, new AbortController().signal);
    
    if (searchResult.llmContent) {
      const documentation = this.formatAsDocumentation(
        topic,
        searchResult.llmContent,
        searchResult.sources
      );
      
      await fs.writeFile(documentPath, documentation, 'utf-8');
    }
  }
  
  private formatAsDocumentation(
    topic: string,
    content: string,
    sources?: GroundingChunkItem[]
  ): string {
    let doc = `# ${topic} 文档\n\n`;
    doc += `> 基于最新网络搜索结果生成\n\n`;
    doc += content;
    
    if (sources && sources.length > 0) {
      doc += '\n\n## 参考资料\n\n';
      sources.forEach((source, index) => {
        if (source.web?.title && source.web?.uri) {
          doc += `${index + 1}. [${source.web.title}](${source.web.uri})\n`;
        }
      });
    }
    
    return doc;
  }
}
```

### 配置集成

#### 搜索配置管理

```typescript
interface WebSearchConfig {
  timeout: number;
  maxRetries: number;
  cacheEnabled: boolean;
  cacheTtl: number;
  defaultRegion?: string;
  defaultLanguage?: string;
}

class ConfigurableWebSearchTool extends WebSearchTool {
  private searchConfig: WebSearchConfig;
  
  constructor(config: Config, searchConfig?: Partial<WebSearchConfig>) {
    super(config);
    
    this.searchConfig = {
      timeout: 30000,
      maxRetries: 3,
      cacheEnabled: true,
      cacheTtl: 600000, // 10 minutes
      ...searchConfig
    };
  }
  
  async execute(
    params: WebSearchToolParams,
    signal: AbortSignal
  ): Promise<WebSearchToolResult> {
    // 应用配置的超时设置
    const timeoutSignal = AbortSignal.timeout(this.searchConfig.timeout);
    const combinedSignal = AbortSignal.any([signal, timeoutSignal]);
    
    return super.execute(params, combinedSignal);
  }
}
```

## 性能优化

### 请求优化

```typescript
class OptimizedWebSearchTool extends WebSearchTool {
  private requestQueue: Array<{
    params: WebSearchToolParams;
    resolve: (result: WebSearchToolResult) => void;
    reject: (error: Error) => void;
  }> = [];
  
  private processing = false;
  private readonly maxQueueSize = 10;
  private readonly processDelayMs = 500;
  
  async execute(
    params: WebSearchToolParams,
    signal: AbortSignal
  ): Promise<WebSearchToolResult> {
    // 检查队列大小
    if (this.requestQueue.length >= this.maxQueueSize) {
      throw new Error('Search queue is full, please try again later');
    }
    
    // 添加到队列并返回 Promise
    return new Promise((resolve, reject) => {
      this.requestQueue.push({ params, resolve, reject });
      this.processQueue();
    });
  }
  
  private async processQueue(): Promise<void> {
    if (this.processing || this.requestQueue.length === 0) {
      return;
    }
    
    this.processing = true;
    
    while (this.requestQueue.length > 0) {
      const request = this.requestQueue.shift()!;
      
      try {
        const result = await super.execute(
          request.params,
          new AbortController().signal
        );
        request.resolve(result);
      } catch (error) {
        request.reject(error instanceof Error ? error : new Error(String(error)));
      }
      
      // 请求间延迟
      if (this.requestQueue.length > 0) {
        await new Promise(resolve => setTimeout(resolve, this.processDelayMs));
      }
    }
    
    this.processing = false;
  }
}
```

### 结果压缩

```typescript
class CompactWebSearchTool extends WebSearchTool {
  async execute(
    params: WebSearchToolParams,
    signal: AbortSignal
  ): Promise<WebSearchToolResult> {
    const result = await super.execute(params, signal);
    
    // 压缩长结果
    if (typeof result.llmContent === 'string' && result.llmContent.length > 5000) {
      result.llmContent = this.compressContent(result.llmContent);
      result.returnDisplay += ' (compressed)';
    }
    
    return result;
  }
  
  private compressContent(content: string): string {
    const lines = content.split('\n');
    const importantLines: string[] = [];
    
    lines.forEach(line => {
      const trimmed = line.trim();
      
      // 保留重要内容
      if (
        trimmed.length > 0 &&
        (trimmed.includes('important') ||
         trimmed.includes('key') ||
         trimmed.includes('best practice') ||
         trimmed.startsWith('#') ||
         trimmed.startsWith('-') ||
         trimmed.startsWith('*'))
      ) {
        importantLines.push(line);
      }
    });
    
    // 如果压缩后太短，返回原始内容的摘要
    if (importantLines.length < 5) {
      return content.substring(0, 2000) + '\n\n[Content truncated for brevity]';
    }
    
    return importantLines.join('\n');
  }
}
```

## 最佳实践

### 查询优化最佳实践

```typescript
// ✅ 推荐：具体而清晰的查询
const goodQueries = [
  "TypeScript 5.0 new features tutorial",
  "React Server Components best practices 2024",
  "Node.js performance optimization guide",
  "PostgreSQL indexing strategies for large datasets",
  "Docker multi-stage build optimization"
];

// ❌ 避免：过于宽泛或模糊的查询
const badQueries = [
  "programming",                    // 太宽泛
  "how to code",                   // 太模糊
  "best language ever",            // 主观性太强
  "fix my bug",                    // 缺乏上下文
  ""                              // 空查询
];
```

### 结果处理最佳实践

```typescript
// 推荐：结构化处理搜索结果
class StructuredSearchProcessor {
  static processSearchResult(result: WebSearchToolResult): ProcessedSearchResult {
    return {
      summary: this.extractSummary(result.llmContent),
      keyPoints: this.extractKeyPoints(result.llmContent),
      sources: this.organizeSources(result.sources),
      reliability: this.assessReliability(result.sources),
      timestamp: new Date().toISOString()
    };
  }
  
  private static extractSummary(content: unknown): string {
    if (typeof content !== 'string') return '';
    
    const lines = content.split('\n');
    const firstParagraph = lines.find(line => 
      line.trim().length > 50 && 
      !line.includes('[') && 
      !line.startsWith('#')
    );
    
    return firstParagraph?.substring(0, 200) + '...' || '';
  }
  
  private static extractKeyPoints(content: unknown): string[] {
    if (typeof content !== 'string') return [];
    
    const keyPoints: string[] = [];
    const lines = content.split('\n');
    
    lines.forEach(line => {
      const trimmed = line.trim();
      if (
        (trimmed.startsWith('- ') || trimmed.startsWith('* ')) &&
        trimmed.length > 20
      ) {
        keyPoints.push(trimmed.substring(2));
      }
    });
    
    return keyPoints.slice(0, 5); // 限制为前5个要点
  }
}

interface ProcessedSearchResult {
  summary: string;
  keyPoints: string[];
  sources: SourceInfo[];
  reliability: number;
  timestamp: string;
}
```

### 安全性最佳实践

```typescript
// 推荐：验证和清理搜索查询
class SecureWebSearchTool extends WebSearchTool {
  private sanitizeQuery(query: string): string {
    // 移除潜在危险字符
    const sanitized = query
      .replace(/[<>]/g, '')           // 移除HTML标签字符
      .replace(/javascript:/gi, '')    // 移除JavaScript协议
      .replace(/data:/gi, '')         // 移除data协议
      .trim();
    
    // 限制查询长度
    return sanitized.substring(0, 500);
  }
  
  async execute(
    params: WebSearchToolParams,
    signal: AbortSignal
  ): Promise<WebSearchToolResult> {
    // 清理查询
    const sanitizedParams = {
      ...params,
      query: this.sanitizeQuery(params.query)
    };
    
    return super.execute(sanitizedParams, signal);
  }
}
```

### 可维护性建议

```typescript
// 推荐：模块化搜索策略
class ModularWebSearchTool extends WebSearchTool {
  private searchStrategies = {
    technical: (query: string) => `${query} tutorial documentation guide 2024`,
    news: (query: string) => `${query} latest news updates recent`,
    comparison: (query: string) => `${query} comparison vs alternatives pros cons`,
    troubleshooting: (query: string) => `${query} error fix solution troubleshoot`
  };
  
  async searchWithStrategy(
    query: string,
    strategy: keyof typeof this.searchStrategies
  ): Promise<WebSearchToolResult> {
    const enhancedQuery = this.searchStrategies[strategy](query);
    
    return this.execute({ query: enhancedQuery }, new AbortController().signal);
  }
}

// 使用示例
const modularSearch = new ModularWebSearchTool(config);

// 技术文档搜索
const techResult = await modularSearch.searchWithStrategy(
  'React hooks',
  'technical'
);

// 问题解决搜索
const troubleResult = await modularSearch.searchWithStrategy(
  'npm install fails',
  'troubleshooting'
);
```