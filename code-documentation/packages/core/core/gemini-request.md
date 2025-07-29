# GeminiRequest - Gemini API 请求处理文档

## 概述

`geminiRequest.ts` 是 Gemini CLI 的 API 请求处理模块，定义了与 Gemini API 交互的核心请求数据结构和工具函数。该模块提供简洁但可扩展的接口，用于处理发送到 Gemini API 的请求内容。

## 主要功能

- **请求类型定义**: 定义与 Gemini API 兼容的请求数据结构
- **内容格式转换**: 提供 `PartListUnion` 到字符串的转换功能
- **类型安全**: 基于 TypeScript 的强类型定义确保 API 调用安全
- **可扩展设计**: 为未来扩展请求参数预留空间

## 类型定义

### `GeminiCodeRequest`

```typescript
export type GeminiCodeRequest = PartListUnion;
```

**描述**: Gemini API 请求的核心类型，目前作为 `PartListUnion` 的别名

**设计理念**:
- **简洁性**: 当前实现保持简洁，直接映射到 Google Gemini SDK 的 `PartListUnion` 类型
- **可扩展性**: 类型别名设计允许未来扩展为包含更多参数的复杂对象
- **兼容性**: 与 `@google/genai` SDK 保持完全兼容

**未来扩展可能性**:
```typescript
// 未来可能的扩展形式
export type GeminiCodeRequest = {
  content: PartListUnion;
  model?: string;
  temperature?: number;
  maxTokens?: number;
  stream?: boolean;
  // 其他请求参数...
};
```

### `PartListUnion` 类型说明

`PartListUnion` 来自 `@google/genai` SDK，支持多种内容格式：

- **字符串**: 简单文本内容
- **Part 对象**: 结构化内容部分
- **Part 数组**: 多个内容部分的组合

```typescript
// 示例内容类型
type PartListUnion = 
  | string 
  | Part 
  | Part[];

// Part 可以包含
interface Part {
  text?: string;
  inlineData?: {
    mimeType: string;
    data: string;
  };
  fileData?: {
    mimeType: string;
    fileUri: string;
  };
  functionCall?: {
    name: string;
    args: Record<string, any>;
  };
  functionResponse?: {
    name: string;
    response: any;
  };
}
```

## 核心函数

### `partListUnionToString` - 内容转换函数

```typescript
export function partListUnionToString(value: PartListUnion): string
```

**功能**: 将 `PartListUnion` 类型的内容转换为可读的字符串格式

**参数**:
- `value: PartListUnion` - 需要转换的内容，可以是字符串、Part 对象或 Part 数组

**返回值**: `string` - 转换后的字符串表示

**实现细节**:
```typescript
export function partListUnionToString(value: PartListUnion): string {
  return partToString(value, { verbose: true });
}
```

**特性**:
- **详细模式**: 使用 `verbose: true` 选项，提供完整的内容表示
- **统一接口**: 无论输入是什么格式，都返回统一的字符串格式
- **调试友好**: 适用于日志记录、调试和内容检查

## 依赖关系

### 外部依赖

```typescript
import { type PartListUnion } from '@google/genai';
import { partToString } from '../utils/partUtils.js';
```

**依赖说明**:
- **@google/genai**: Google 官方 Gemini AI SDK，提供核心类型定义
- **partUtils**: 内部工具模块，提供内容转换功能

### 模块关系

```
geminiRequest.ts
├── 被 client.ts 使用（核心 API 客户端）
├── 被 coreToolScheduler.ts 使用（工具调度）
├── 被 ContentGenerator 使用（内容生成）
└── 依赖 partUtils.js（内容转换工具）
```

## 使用场景和示例

### 基本请求创建

```typescript
import { GeminiCodeRequest, partListUnionToString } from './geminiRequest.js';

// 简单文本请求
const textRequest: GeminiCodeRequest = "Hello, how can I help you with coding?";

// 多部分请求
const multiPartRequest: GeminiCodeRequest = [
  { text: "Here is some code to review:" },
  { 
    text: `
function fibonacci(n) {
  if (n <= 1) return n;
  return fibonacci(n - 1) + fibonacci(n - 2);
}
    `.trim()
  },
  { text: "Can you optimize this function?" }
];

// 包含函数调用的请求
const functionCallRequest: GeminiCodeRequest = [
  { text: "Please read the file at /path/to/file.js" },
  {
    functionCall: {
      name: "read-file",
      args: { file_path: "/path/to/file.js" }
    }
  }
];
```

### 内容转换和调试

```typescript
// 转换请求内容为字符串（用于日志记录）
const requestString = partListUnionToString(multiPartRequest);
console.log('Request content:', requestString);

// 调试复杂请求结构
function debugRequest(request: GeminiCodeRequest) {
  const readable = partListUnionToString(request);
  console.log('=== Request Debug ===');
  console.log('Type:', Array.isArray(request) ? 'Array' : typeof request);
  console.log('Content:', readable);
  console.log('Length:', readable.length);
  console.log('====================');
}

debugRequest(functionCallRequest);
```

### 与 API 客户端集成

```typescript
import { GeminiClient } from './client.js';
import { GeminiCodeRequest } from './geminiRequest.js';

async function sendRequestExample() {
  const client = new GeminiClient(/* config */);
  
  // 创建请求
  const request: GeminiCodeRequest = [
    { text: "Analyze this TypeScript code:" },
    { text: "interface User { id: number; name: string; }" },
    { text: "What improvements can be made?" }
  ];
  
  // 发送请求（简化示例）
  try {
    const response = await client.sendMessage(request);
    console.log('Response:', response);
  } catch (error) {
    console.error('Request failed:', error);
    console.log('Original request:', partListUnionToString(request));
  }
}
```

### 请求内容验证

```typescript
// 请求内容验证工具
function validateRequest(request: GeminiCodeRequest): {
  valid: boolean;
  errors: string[];
  warnings: string[];
} {
  const errors: string[] = [];
  const warnings: string[] = [];
  
  // 检查空内容
  if (!request) {
    errors.push('Request is null or undefined');
    return { valid: false, errors, warnings };
  }
  
  // 转换为字符串进行内容检查
  const content = partListUnionToString(request);
  
  // 检查内容长度
  if (content.length === 0) {
    errors.push('Request content is empty');
  }
  
  if (content.length > 100000) {
    warnings.push('Request content is very large (>100K chars)');
  }
  
  // 检查潜在的敏感信息
  const sensitivePatterns = [
    /password\s*[:=]\s*["'].*["']/i,
    /api[_-]?key\s*[:=]\s*["'].*["']/i,
    /secret\s*[:=]\s*["'].*["']/i
  ];
  
  for (const pattern of sensitivePatterns) {
    if (pattern.test(content)) {
      warnings.push('Request may contain sensitive information');
      break;
    }
  }
  
  return {
    valid: errors.length === 0,
    errors,
    warnings
  };
}

// 使用验证
const request: GeminiCodeRequest = "Help me debug this API key issue";
const validation = validateRequest(request);

if (!validation.valid) {
  console.error('Request validation failed:', validation.errors);
} else if (validation.warnings.length > 0) {
  console.warn('Request warnings:', validation.warnings);
}
```

### 请求构建器模式

```typescript
// 请求构建器类
class GeminiRequestBuilder {
  private parts: Part[] = [];
  
  addText(text: string): this {
    this.parts.push({ text });
    return this;
  }
  
  addCode(code: string, language?: string): this {
    const formattedCode = language 
      ? `\`\`\`${language}\n${code}\n\`\`\``
      : `\`\`\`\n${code}\n\`\`\``;
    this.parts.push({ text: formattedCode });
    return this;
  }
  
  addFunctionCall(name: string, args: Record<string, any>): this {
    this.parts.push({
      functionCall: { name, args }
    });
    return this;
  }
  
  addFile(content: string, filename: string): this {
    this.parts.push({
      text: `File: ${filename}\n\`\`\`\n${content}\n\`\`\``
    });
    return this;
  }
  
  build(): GeminiCodeRequest {
    if (this.parts.length === 0) {
      throw new Error('Request builder is empty');
    }
    
    return this.parts.length === 1 ? this.parts[0] : this.parts;
  }
  
  preview(): string {
    return partListUnionToString(this.build());
  }
  
  clear(): this {
    this.parts = [];
    return this;
  }
}

// 使用构建器
const builder = new GeminiRequestBuilder();

const codeReviewRequest = builder
  .addText("Please review this TypeScript function:")
  .addCode(`
function processUser(user: any) {
  if (user.name) {
    return user.name.toUpperCase();
  }
  return "Unknown";
}
  `, "typescript")
  .addText("Focus on type safety and error handling.")
  .build();

console.log('Built request:', partListUnionToString(codeReviewRequest));
```

### 请求模板系统

```typescript
// 请求模板定义
interface RequestTemplate {
  name: string;
  description: string;
  template: (params: Record<string, any>) => GeminiCodeRequest;
}

const REQUEST_TEMPLATES: RequestTemplate[] = [
  {
    name: 'code_review',
    description: 'Review code with specific focus areas',
    template: (params) => [
      { text: `Please review this ${params.language || 'code'}:` },
      { text: `\`\`\`${params.language || ''}\n${params.code}\n\`\`\`` },
      { text: params.focus ? `Focus on: ${params.focus}` : 'Provide general feedback.' }
    ]
  },
  {
    name: 'debug_help',
    description: 'Help debug an issue with context',
    template: (params) => [
      { text: `I'm having trouble with: ${params.issue}` },
      ...(params.code ? [{ text: `\`\`\`\n${params.code}\n\`\`\`` }] : []),
      ...(params.error ? [{ text: `Error: ${params.error}` }] : []),
      { text: 'What could be causing this issue?' }
    ]
  },
  {
    name: 'file_analysis',
    description: 'Analyze a file with function calls',
    template: (params) => [
      { text: `Please analyze the file: ${params.filename}` },
      {
        functionCall: {
          name: 'read-file',
          args: { file_path: params.filepath }
        }
      },
      { text: params.analysis_type ? `Focus on: ${params.analysis_type}` : 'Provide comprehensive analysis.' }
    ]
  }
];

// 模板使用函数
function createRequestFromTemplate(
  templateName: string, 
  params: Record<string, any>
): GeminiCodeRequest {
  const template = REQUEST_TEMPLATES.find(t => t.name === templateName);
  if (!template) {
    throw new Error(`Template '${templateName}' not found`);
  }
  
  return template.template(params);
}

// 使用模板
const reviewRequest = createRequestFromTemplate('code_review', {
  language: 'typescript',
  code: 'function add(a, b) { return a + b; }',
  focus: 'type safety'
});

const debugRequest = createRequestFromTemplate('debug_help', {
  issue: 'Function returns undefined',
  code: 'function getData() { /* some code */ }',
  error: 'TypeError: Cannot read property of undefined'
});

const analysisRequest = createRequestFromTemplate('file_analysis', {
  filename: 'auth.ts',
  filepath: '/path/to/auth.ts',
  analysis_type: 'security vulnerabilities'
});
```

### 请求内容压缩

```typescript
// 请求内容压缩工具
class RequestCompressor {
  private maxLength: number;
  
  constructor(maxLength: number = 50000) {
    this.maxLength = maxLength;
  }
  
  compress(request: GeminiCodeRequest): GeminiCodeRequest {
    const content = partListUnionToString(request);
    
    if (content.length <= this.maxLength) {
      return request; // 不需要压缩
    }
    
    console.warn(`Request too long (${content.length} chars), compressing...`);
    
    if (typeof request === 'string') {
      return this.compressText(request);
    }
    
    if (Array.isArray(request)) {
      return this.compressArray(request);
    }
    
    // 单个 Part 对象
    return this.compressPart(request);
  }
  
  private compressText(text: string): string {
    // 简单截断策略
    if (text.length <= this.maxLength) return text;
    
    const truncated = text.substring(0, this.maxLength - 100);
    const lastSpace = truncated.lastIndexOf(' ');
    const cutPoint = lastSpace > this.maxLength * 0.8 ? lastSpace : truncated.length;
    
    return truncated.substring(0, cutPoint) + 
      `\n\n[Content truncated - original length: ${text.length} chars]`;
  }
  
  private compressArray(parts: Part[]): Part[] {
    const compressed: Part[] = [];
    let currentLength = 0;
    
    for (const part of parts) {
      const partContent = partToString(part, { verbose: false });
      
      if (currentLength + partContent.length > this.maxLength) {
        // 添加截断说明
        compressed.push({
          text: `[${parts.length - compressed.length} parts truncated due to length limit]`
        });
        break;
      }
      
      compressed.push(part);
      currentLength += partContent.length;
    }
    
    return compressed;
  }
  
  private compressPart(part: Part): Part {
    if (part.text) {
      return { ...part, text: this.compressText(part.text) };
    }
    return part; // 其他类型的 Part 暂不压缩
  }
}

// 使用压缩器
const compressor = new RequestCompressor(30000);
const longRequest: GeminiCodeRequest = [
  { text: "Very long text content..." },
  // ... many more parts
];

const compressedRequest = compressor.compress(longRequest);
console.log('Original:', partListUnionToString(longRequest).length);
console.log('Compressed:', partListUnionToString(compressedRequest).length);
```

## 性能考虑

### 内存使用优化

```typescript
// 大型请求的内存优化处理
function createStreamableRequest(content: string): GeminiCodeRequest {
  // 对于大型内容，分块处理
  const CHUNK_SIZE = 10000;
  
  if (content.length <= CHUNK_SIZE) {
    return content;
  }
  
  const chunks: Part[] = [];
  for (let i = 0; i < content.length; i += CHUNK_SIZE) {
    chunks.push({
      text: content.substring(i, i + CHUNK_SIZE)
    });
  }
  
  return chunks;
}
```

### 请求缓存策略

```typescript
// 请求内容缓存
const requestCache = new Map<string, string>();

function getCachedRequestString(request: GeminiCodeRequest): string {
  const key = JSON.stringify(request);
  
  if (requestCache.has(key)) {
    return requestCache.get(key)!;
  }
  
  const result = partListUnionToString(request);
  requestCache.set(key, result);
  
  // 限制缓存大小
  if (requestCache.size > 1000) {
    const firstKey = requestCache.keys().next().value;
    requestCache.delete(firstKey);
  }
  
  return result;
}
```

## 错误处理

### 请求验证错误

```typescript
class RequestValidationError extends Error {
  constructor(
    message: string,
    public readonly request: GeminiCodeRequest,
    public readonly validationErrors: string[]
  ) {
    super(message);
    this.name = 'RequestValidationError';
  }
}

function validateAndProcess(request: GeminiCodeRequest): void {
  const validation = validateRequest(request);
  
  if (!validation.valid) {
    throw new RequestValidationError(
      'Request validation failed',
      request,
      validation.errors
    );
  }
  
  if (validation.warnings.length > 0) {
    console.warn('Request validation warnings:', validation.warnings);
  }
}
```

### 转换错误处理

```typescript
function safePartListUnionToString(value: PartListUnion): string {
  try {
    return partListUnionToString(value);
  } catch (error) {
    console.error('Failed to convert PartListUnion to string:', error);
    
    // 提供备用转换策略
    if (typeof value === 'string') {
      return value;
    }
    
    if (Array.isArray(value)) {
      return value.map(part => 
        part.text || '[Non-text content]'
      ).join('\n');
    }
    
    return '[Unconvertible content]';
  }
}
```

## 测试示例

### 单元测试

```typescript
describe('geminiRequest', () => {
  describe('GeminiCodeRequest type', () => {
    it('should accept string content', () => {
      const request: GeminiCodeRequest = "Hello world";
      expect(typeof request).toBe('string');
    });
    
    it('should accept Part array', () => {
      const request: GeminiCodeRequest = [
        { text: "Part 1" },
        { text: "Part 2" }
      ];
      expect(Array.isArray(request)).toBe(true);
    });
  });
  
  describe('partListUnionToString', () => {
    it('should convert string input', () => {
      const result = partListUnionToString("test string");
      expect(result).toBe("test string");
    });
    
    it('should convert Part array', () => {
      const input = [
        { text: "Hello" },
        { text: "World" }
      ];
      const result = partListUnionToString(input);
      expect(result).toContain("Hello");
      expect(result).toContain("World");
    });
    
    it('should handle function calls', () => {
      const input = {
        functionCall: {
          name: "test-function",
          args: { param: "value" }
        }
      };
      const result = partListUnionToString(input);
      expect(result).toContain("test-function");
    });
  });
});
```

### 集成测试

```typescript
describe('Request Integration', () => {
  it('should work with GeminiClient', async () => {
    const request: GeminiCodeRequest = "Test request";
    const client = new GeminiClient(testConfig);
    
    // Mock the client response
    jest.spyOn(client, 'sendMessage').mockResolvedValue(mockResponse);
    
    const response = await client.sendMessage(request);
    expect(response).toBeDefined();
  });
  
  it('should handle request building pipeline', () => {
    const builder = new GeminiRequestBuilder();
    
    const request = builder
      .addText("Test")
      .addCode("console.log('hello')", "javascript")
      .build();
    
    const stringified = partListUnionToString(request);
    expect(stringified).toContain("Test");
    expect(stringified).toContain("console.log");
  });
});
```

## 最佳实践

### 1. 类型使用建议

```typescript
// ✅ 推荐：明确使用 GeminiCodeRequest 类型
function processRequest(request: GeminiCodeRequest): void {
  // 类型安全的处理
}

// ❌ 避免：使用 any 类型
function processRequest(request: any): void {
  // 失去类型安全性
}
```

### 2. 内容验证

```typescript
// ✅ 推荐：在发送前验证请求内容
function sendSafeRequest(request: GeminiCodeRequest): Promise<Response> {
  const validation = validateRequest(request);
  if (!validation.valid) {
    throw new Error(`Invalid request: ${validation.errors.join(', ')}`);
  }
  
  return client.sendMessage(request);
}
```

### 3. 错误处理

```typescript
// ✅ 推荐：完整的错误处理
async function handleRequest(request: GeminiCodeRequest): Promise<void> {
  try {
    const response = await client.sendMessage(request);
    // 处理响应
  } catch (error) {
    console.error('Request failed:', error);
    console.log('Failed request content:', 
      safePartListUnionToString(request));
    // 错误恢复逻辑
  }
}
```

### 4. 性能优化

```typescript
// ✅ 推荐：缓存重复的字符串转换
const requestStringCache = new WeakMap<object, string>();

function getCachedString(request: GeminiCodeRequest): string {
  if (typeof request === 'string') return request;
  
  if (requestStringCache.has(request)) {
    return requestStringCache.get(request)!;
  }
  
  const result = partListUnionToString(request);
  requestStringCache.set(request, result);
  return result;
}
```

## 未来发展方向

### 计划中的功能扩展

1. **增强请求参数**:
   ```typescript
   export interface GeminiCodeRequest {
     content: PartListUnion;
     model?: string;
     temperature?: number;
     maxTokens?: number;
     stopSequences?: string[];
     stream?: boolean;
   }
   ```

2. **请求中间件系统**:
   ```typescript
   interface RequestMiddleware {
     process(request: GeminiCodeRequest): GeminiCodeRequest;
   }
   ```

3. **内容类型验证器**:
   ```typescript
   interface ContentValidator {
     validate(content: PartListUnion): ValidationResult;
   }
   ```

4. **请求序列化优化**:
   ```typescript
   interface RequestSerializer {
     serialize(request: GeminiCodeRequest): Buffer;
     deserialize(data: Buffer): GeminiCodeRequest;
   }
   ```

该模块虽然目前实现简洁，但为 Gemini CLI 提供了坚实的 API 请求基础，并为未来的功能扩展预留了充足的设计空间。通过类型别名和工具函数的设计，确保了系统的可维护性和可扩展性。