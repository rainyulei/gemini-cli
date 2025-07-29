# WebFetchTool 网页获取工具文档

## 概述

`WebFetchTool` 是 Gemini CLI 的网页内容获取和处理工具，能够从 URL 获取内容并根据用户指令进行智能处理。它支持多种获取模式，包括 Gemini 原生 URL 上下文、回退获取机制、GitHub 文件处理和私有网络地址访问。

## 主要功能

- 智能网页内容获取和处理
- 支持多个 URL（最多 20 个）
- GitHub 文件特殊处理（blob URL 转换）
- 私有网络地址支持（localhost 等）
- HTML 到文本转换
- 智能引用和来源标注
- 回退获取机制
- 代理支持

## 接口定义

### `WebFetchToolParams`

**功能**: WebFetch 工具参数接口

```typescript
export interface WebFetchToolParams {
  prompt: string;  // 包含 URL 和处理指令的提示文本
}
```

**参数要求**:

- `prompt`: 必须包含至少一个以 `http://` 或 `https://` 开头的 URL
- 可以包含具体的处理指令（如"总结"、"提取关键信息"等）
- 支持最多 20 个 URL

### 内部接口

#### `GroundingChunkWeb`

**功能**: 网页引用块信息

```typescript
interface GroundingChunkWeb {
  uri?: string;     // 网页 URI
  title?: string;   // 网页标题
}
```

#### `GroundingSupportItem`

**功能**: 引用支持信息

```typescript
interface GroundingSupportItem {
  segment?: GroundingSupportSegment;  // 文本片段
  groundingChunkIndices?: number[];   // 引用块索引
}
```

## 核心方法

### 构造函数

#### `constructor(config: Config)`

**功能**: 创建 WebFetchTool 实例并配置代理

```typescript
constructor(private readonly config: Config) {
  super(
    WebFetchTool.Name,
    'WebFetch',
    "处理嵌入在提示中的 URL 内容，包括本地和私有网络地址。支持最多 20 个 URL 和具体处理指令。",
    Icon.Globe,
    // Schema 定义...
  );
  
  // 配置代理
  const proxy = config.getProxy();
  if (proxy) {
    setGlobalDispatcher(new ProxyAgent(proxy as string));
  }
}
```

### URL 提取和处理

#### `extractUrls(text: string): string[]`

**功能**: 从文本中提取 URL

```typescript
function extractUrls(text: string): string[] {
  const urlRegex = /(https?:\/\/[^\s]+)/g;
  return text.match(urlRegex) || [];
}
```

**使用示例**:

```typescript
const urls = extractUrls("请总结 https://example.com/article 和 https://another.com/data");
// 返回: ["https://example.com/article", "https://another.com/data"]
```

### 参数验证

#### `validateParams(params: WebFetchToolParams): string | null`

**功能**: 验证工具参数

**验证项目**:

1. **Schema 验证**: 验证参数结构
2. **空值检查**: 确保 prompt 不为空
3. **URL 检查**: 确保包含有效 URL

```typescript
validateParams(params: WebFetchToolParams): string | null {
  const errors = SchemaValidator.validate(this.schema.parameters, params);
  if (errors) return errors;
  
  if (!params.prompt || params.prompt.trim() === '') {
    return "The 'prompt' parameter cannot be empty and must contain URL(s) and instructions.";
  }
  
  if (!params.prompt.includes('http://') && !params.prompt.includes('https://')) {
    return "The 'prompt' must contain at least one valid URL.";
  }
  
  return null;
}
```

### 主要执行流程

#### `async execute(params: WebFetchToolParams, signal: AbortSignal): Promise<ToolResult>`

**功能**: 执行网页获取和处理操作

**执行步骤**:

1. **参数验证**:
   ```typescript
   const validationError = this.validateParams(params);
   if (validationError) {
     return {
       llmContent: `Error: Invalid parameters. Reason: ${validationError}`,
       returnDisplay: validationError,
     };
   }
   ```

2. **URL 分析和私有网络检测**:
   ```typescript
   const urls = extractUrls(userPrompt);
   const url = urls[0];
   const isPrivate = isPrivateIp(url);
   
   if (isPrivate) {
     return this.executeFallback(params, signal);
   }
   ```

3. **Gemini 原生处理**:
   ```typescript
   const response = await geminiClient.generateContent(
     [{ role: 'user', parts: [{ text: userPrompt }] }],
     { tools: [{ urlContext: {} }] },  // 启用 URL 上下文工具
     signal,
   );
   ```

4. **错误检测和处理**:
   ```typescript
   let processingError = false;
   
   if (urlContextMeta?.urlMetadata?.length > 0) {
     const allStatuses = urlContextMeta.urlMetadata.map(m => m.urlRetrievalStatus);
     if (allStatuses.every(s => s !== 'URL_RETRIEVAL_STATUS_SUCCESS')) {
       processingError = true;
     }
   }
   
   if (processingError) {
     return this.executeFallback(params, signal);
   }
   ```

5. **引用和来源处理**:
   ```typescript
   const sources = groundingMetadata?.groundingChunks;
   const groundingSupports = groundingMetadata?.groundingSupports;
   
   // 生成来源列表
   sources?.forEach((source, index) => {
     const title = source.web?.title || 'Untitled';
     const uri = source.web?.uri || 'Unknown URI';
     sourceListFormatted.push(`[${index + 1}] ${title} (${uri})`);
   });
   
   // 插入引用标记
   if (groundingSupports?.length > 0) {
     const insertions = [];
     groundingSupports.forEach(support => {
       if (support.segment && support.groundingChunkIndices) {
         const citationMarker = support.groundingChunkIndices
           .map(chunkIndex => `[${chunkIndex + 1}]`)
           .join('');
         insertions.push({
           index: support.segment.endIndex,
           marker: citationMarker,
         });
       }
     });
     
     // 按索引倒序插入引用标记
     insertions.sort((a, b) => b.index - a.index);
     const responseChars = responseText.split('');
     insertions.forEach(insertion => {
       responseChars.splice(insertion.index, 0, insertion.marker);
     });
     responseText = responseChars.join('');
   }
   ```

### 回退获取机制

#### `private async executeFallback(params: WebFetchToolParams, signal: AbortSignal): Promise<ToolResult>`

**功能**: 当 Gemini 原生处理失败时的回退方案

**适用场景**:

- 私有网络地址（localhost、内网 IP）
- Gemini URL 上下文处理失败
- 需要更直接的内容获取

**执行流程**:

1. **GitHub URL 转换**:
   ```typescript
   if (url.includes('github.com') && url.includes('/blob/')) {
     url = url
       .replace('github.com', 'raw.githubusercontent.com')
       .replace('/blob/', '/');
   }
   ```

2. **内容获取**:
   ```typescript
   const response = await fetchWithTimeout(url, URL_FETCH_TIMEOUT_MS);
   if (!response.ok) {
     throw new Error(`Request failed with status ${response.status}`);
   }
   const html = await response.text();
   ```

3. **HTML 转文本**:
   ```typescript
   const textContent = convert(html, {
     wordwrap: false,
     selectors: [
       { selector: 'a', options: { ignoreHref: true } },
       { selector: 'img', format: 'skip' },
     ],
   }).substring(0, MAX_CONTENT_LENGTH);
   ```

4. **AI 处理**:
   ```typescript
   const fallbackPrompt = `用户请求：${params.prompt}
   
   我无法直接访问 URL，但已获取页面原始内容。请使用以下内容回答用户请求：
   
   ---
   ${textContent}
   ---`;
   
   const result = await geminiClient.generateContent(
     [{ role: 'user', parts: [{ text: fallbackPrompt }] }],
     {},
     signal,
   );
   ```

### 用户确认

#### `async shouldConfirmExecute(params: WebFetchToolParams): Promise<ToolCallConfirmationDetails | false>`

**功能**: 生成用户确认提示

**确认信息**:

```typescript
const confirmationDetails: ToolCallConfirmationDetails = {
  type: 'info',
  title: 'Confirm Web Fetch',
  prompt: params.prompt,
  urls: transformedUrls,  // 包含 GitHub 转换后的 URL
  onConfirm: async (outcome: ToolConfirmationOutcome) => {
    if (outcome === ToolConfirmationOutcome.ProceedAlways) {
      this.config.setApprovalMode(ApprovalMode.AUTO_EDIT);
    }
  },
};
```

## 使用示例

### 基本网页获取

```typescript
const webFetchTool = new WebFetchTool(config);

// 获取并总结网页内容
const result = await webFetchTool.execute({
  prompt: "请总结 https://example.com/article 的主要内容"
});
```

### 多 URL 处理

```typescript
// 比较多个网页内容
const result = await webFetchTool.execute({
  prompt: `比较以下两篇文章的观点：
  https://site1.com/article1
  https://site2.com/article2
  请指出它们的相同点和不同点`
});
```

### GitHub 文件获取

```typescript
// GitHub 代码文件分析
const result = await webFetchTool.execute({
  prompt: "分析这个 GitHub 文件的代码结构：https://github.com/user/repo/blob/main/src/index.js"
});

// 工具会自动转换为:
// https://raw.githubusercontent.com/user/repo/main/src/index.js
```

### 本地和私有网络

```typescript
// 访问本地服务
const result = await webFetchTool.execute({
  prompt: "获取 http://localhost:3000/api/status 的状态信息"
});

// 访问内网地址
const result = await webFetchTool.execute({
  prompt: "检查内网服务 http://192.168.1.100/health 的健康状态"
});
```

### 特定数据提取

```typescript
// 提取特定信息
const result = await webFetchTool.execute({
  prompt: `从 https://news.example.com/latest 提取：
  1. 标题
  2. 发布时间
  3. 主要新闻摘要`
});
```

## 高级功能

### 代理配置

```typescript
// 在配置中设置代理
const config = new Config({
  // ... 其他配置
  proxy: 'http://proxy.company.com:8080'
});

const webFetchTool = new WebFetchTool(config);
// 所有网络请求将通过代理
```

### 引用和来源

WebFetchTool 自动处理 Gemini 的引用系统：

```typescript
// 获取的内容会包含引用标记
const result = await webFetchTool.execute({
  prompt: "总结这篇文章：https://example.com/article"
});

// 结果可能类似：
// 这篇文章讨论了人工智能的发展[1]。主要观点包括...
// 
// Sources:
// [1] AI Development Trends (https://example.com/article)
```

### 错误处理策略

```typescript
try {
  const result = await webFetchTool.execute(params, abortSignal);
  
  if (result.llmContent.startsWith('Error:')) {
    console.error('Web fetch failed:', result.returnDisplay);
    // 处理错误，可能尝试备用方案
  } else {
    console.log('Content processed:', result.llmContent);
  }
} catch (error) {
  console.error('Unexpected error:', error);
  // 处理网络错误、超时等
}
```

## 配置选项

### 超时设置

```typescript
const URL_FETCH_TIMEOUT_MS = 10000;  // 10秒超时
```

### 内容长度限制

```typescript
const MAX_CONTENT_LENGTH = 100000;  // 最大100KB文本内容
```

### HTML 转换选项

```typescript
const htmlToTextOptions = {
  wordwrap: false,
  selectors: [
    { selector: 'a', options: { ignoreHref: true } },  // 忽略链接href
    { selector: 'img', format: 'skip' },               // 跳过图片
  ],
};
```

## 安全考虑

### 私有网络检测

```typescript
const isPrivate = isPrivateIp(url);
if (isPrivate) {
  // 使用回退机制，避免通过 Gemini 访问私有地址
  return this.executeFallback(params, signal);
}
```

### URL 验证

- 仅允许 `http://` 和 `https://` 协议
- 自动检测和处理恶意 URL
- 代理配置的安全控制

### 内容安全

- 限制获取内容的最大长度
- HTML 内容的安全转换
- 敏感信息过滤

## 性能优化

### 缓存策略

虽然当前实现不包含缓存，但可以扩展：

```typescript
class CachedWebFetchTool extends WebFetchTool {
  private cache = new Map<string, { content: string; timestamp: number }>();
  
  async execute(params: WebFetchToolParams, signal: AbortSignal) {
    const cacheKey = this.generateCacheKey(params);
    const cached = this.cache.get(cacheKey);
    
    if (cached && !this.isCacheExpired(cached.timestamp)) {
      return { llmContent: cached.content, returnDisplay: 'From cache' };
    }
    
    const result = await super.execute(params, signal);
    this.cache.set(cacheKey, { content: result.llmContent, timestamp: Date.now() });
    return result;
  }
}
```

### 并发控制

```typescript
// 限制并发请求数量
const semaphore = new Semaphore(5);  // 最多5个并发请求

async execute(params: WebFetchToolParams, signal: AbortSignal) {
  await semaphore.acquire();
  try {
    return await this.executeInternal(params, signal);
  } finally {
    semaphore.release();
  }
}
```

## 扩展功能

### 自定义处理器

```typescript
interface ContentProcessor {
  canProcess(url: string): boolean;
  process(content: string, url: string): string;
}

class PDFProcessor implements ContentProcessor {
  canProcess(url: string): boolean {
    return url.endsWith('.pdf');
  }
  
  process(content: string, url: string): string {
    // PDF 特殊处理逻辑
    return extractTextFromPDF(content);
  }
}
```

### 多媒体支持

```typescript
// 扩展支持图片、视频等多媒体内容
class MultimediaWebFetchTool extends WebFetchTool {
  async processMultimedia(url: string): Promise<string> {
    if (this.isImageUrl(url)) {
      return await this.processImage(url);
    } else if (this.isVideoUrl(url)) {
      return await this.processVideo(url);
    }
    return super.executeFallback({ prompt: url }, new AbortController().signal);
  }
}
```

## 错误类型和处理

### 网络错误

- **连接超时**: 10秒超时机制
- **DNS 解析失败**: 自动回退到错误报告
- **HTTP 错误状态**: 详细状态码报告

### 内容处理错误

- **HTML 解析失败**: 优雅降级到原始文本
- **字符编码问题**: 自动检测和转换
- **内容过大**: 自动截断到限制长度

### Gemini API 错误

- **配额限制**: 自动回退到直接获取
- **上下文长度限制**: 内容分片处理
- **API 不可用**: 完全回退模式

## 集成特性

### 与其他工具协作

- **ReadFileTool**: 本地文件和网络内容统一处理
- **WebSearchTool**: 搜索结果的详细内容获取
- **EditTool**: 基于网络内容的代码生成

### 配置集成

- 审批模式控制用户确认
- 代理设置的全局配置
- 调试模式的详细日志

### 遥测和监控

- URL 访问统计
- 成功/失败率监控
- 性能指标收集