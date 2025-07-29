# MemoryTool 会话记忆管理工具文档

## 概述

`MemoryTool` 是 Gemini CLI 的长期记忆管理工具，专门用于保存用户的重要信息和偏好设置，以便在后续的会话中提供个性化和高效的服务。它通过将用户信息写入全局配置文件的方式，实现跨会话的记忆持久化。

## 主要功能

- 长期记忆存储和管理
- 自动化的 Markdown 格式组织
- 全局配置文件集成
- 智能的内容去重和格式化
- 跨会话记忆持久化
- 安全的文件系统操作
- 可配置的记忆文件位置

## 接口定义

### `SaveMemoryParams`

**功能**: MemoryTool 工具参数接口

```typescript
interface SaveMemoryParams {
  fact: string;                       // 要记住的事实或信息（必需）
}
```

**参数详情**:

- `fact`: 要保存的具体事实或信息，应该是清晰、自包含的陈述

### 配置常量

```typescript
export const GEMINI_CONFIG_DIR = '.gemini';           // 配置目录名
export const DEFAULT_CONTEXT_FILENAME = 'GEMINI.md';  // 默认记忆文件名
export const MEMORY_SECTION_HEADER = '## Gemini Added Memories';  // 记忆段落标题
```

### 文件名配置

```typescript
// 当前配置的文件名变量
let currentGeminiMdFilename: string | string[] = DEFAULT_CONTEXT_FILENAME;

// 设置记忆文件名
export function setGeminiMdFilename(newFilename: string | string[]): void;

// 获取当前记忆文件名
export function getCurrentGeminiMdFilename(): string;

// 获取所有记忆文件名
export function getAllGeminiMdFilenames(): string[];
```

## 核心方法

### 构造函数

#### `constructor()`

**功能**: 创建 MemoryTool 实例

```typescript
constructor() {
  super(
    MemoryTool.Name,
    'Save Memory',
    memoryToolDescription,
    Icon.LightBulb,
    memoryToolSchemaData.parameters,
  );
}
```

**工具特性**:

- 基于用户明确请求的记忆保存
- 自动格式化和组织记忆内容
- 安全的文件系统操作
- 错误处理和恢复机制

### 文件路径管理

#### `getGlobalMemoryFilePath(): string`

**功能**: 获取全局记忆文件路径

```typescript
function getGlobalMemoryFilePath(): string {
  return path.join(homedir(), GEMINI_CONFIG_DIR, getCurrentGeminiMdFilename());
}
```

**路径结构**:
- **Unix/Linux/macOS**: `~/.gemini/GEMINI.md`
- **Windows**: `C:\Users\{用户名}\.gemini\GEMINI.md`

### 内容格式化

#### `ensureNewlineSeparation(currentContent: string): string`

**功能**: 确保内容之间有适当的换行分隔

```typescript
function ensureNewlineSeparation(currentContent: string): string {
  if (currentContent.length === 0) return '';
  if (currentContent.endsWith('\n\n') || currentContent.endsWith('\r\n\r\n'))
    return '';
  if (currentContent.endsWith('\n') || currentContent.endsWith('\r\n'))
    return '\n';
  return '\n\n';
}
```

**格式化规则**:

1. **空内容**: 返回空字符串
2. **已有双换行**: 不添加额外换行
3. **已有单换行**: 添加单换行变成双换行
4. **无换行**: 添加双换行

### 核心记忆添加方法

#### `static async performAddMemoryEntry(text: string, memoryFilePath: string, fsAdapter: FsAdapter): Promise<void>`

**功能**: 执行记忆条目添加操作

```typescript
static async performAddMemoryEntry(
  text: string,
  memoryFilePath: string,
  fsAdapter: {
    readFile: (path: string, encoding: 'utf-8') => Promise<string>;
    writeFile: (path: string, data: string, encoding: 'utf-8') => Promise<void>;
    mkdir: (path: string, options: { recursive: boolean }) => Promise<string | undefined>;
  },
): Promise<void> {
  let processedText = text.trim();
  // 移除开头的连字符，避免被误解为 Markdown 列表项
  processedText = processedText.replace(/^(-+\s*)+/, '').trim();
  const newMemoryItem = `- ${processedText}`;

  try {
    // 确保目录存在
    await fsAdapter.mkdir(path.dirname(memoryFilePath), { recursive: true });
    
    let content = '';
    try {
      content = await fsAdapter.readFile(memoryFilePath, 'utf-8');
    } catch (_e) {
      // 文件不存在，将创建新文件
    }

    const headerIndex = content.indexOf(MEMORY_SECTION_HEADER);

    if (headerIndex === -1) {
      // 头部不存在，添加头部和条目
      const separator = ensureNewlineSeparation(content);
      content += `${separator}${MEMORY_SECTION_HEADER}\n${newMemoryItem}\n`;
    } else {
      // 头部存在，找到插入新记忆条目的位置
      const startOfSectionContent = headerIndex + MEMORY_SECTION_HEADER.length;
      let endOfSectionIndex = content.indexOf('\n## ', startOfSectionContent);
      if (endOfSectionIndex === -1) {
        endOfSectionIndex = content.length; // 文件结尾
      }

      const beforeSectionMarker = content.substring(0, startOfSectionContent).trimEnd();
      let sectionContent = content.substring(startOfSectionContent, endOfSectionIndex).trimEnd();
      const afterSectionMarker = content.substring(endOfSectionIndex);

      sectionContent += `\n${newMemoryItem}`;
      content = `${beforeSectionMarker}\n${sectionContent.trimStart()}\n${afterSectionMarker}`.trimEnd() + '\n';
    }
    
    await fsAdapter.writeFile(memoryFilePath, content, 'utf-8');
  } catch (error) {
    throw new Error(`Failed to add memory entry: ${error instanceof Error ? error.message : String(error)}`);
  }
}
```

**操作流程**:

1. **文本预处理**: 清理输入文本，移除多余的连字符
2. **格式化**: 转换为 Markdown 列表项格式
3. **目录创建**: 确保记忆文件目录存在
4. **文件读取**: 尝试读取现有文件内容
5. **头部检查**: 查找记忆段落标题
6. **内容插入**: 在适当位置插入新记忆条目
7. **文件写入**: 保存更新后的内容

### 主要执行方法

#### `async execute(params: SaveMemoryParams, _signal: AbortSignal): Promise<ToolResult>`

**功能**: 执行记忆保存操作

```typescript
async execute(
  params: SaveMemoryParams,
  _signal: AbortSignal,
): Promise<ToolResult> {
  const { fact } = params;

  if (!fact || typeof fact !== 'string' || fact.trim() === '') {
    const errorMessage = 'Parameter "fact" must be a non-empty string.';
    return {
      llmContent: JSON.stringify({ success: false, error: errorMessage }),
      returnDisplay: `Error: ${errorMessage}`,
    };
  }

  try {
    await MemoryTool.performAddMemoryEntry(fact, getGlobalMemoryFilePath(), {
      readFile: fs.readFile,
      writeFile: fs.writeFile,
      mkdir: fs.mkdir,
    });
    
    const successMessage = `Okay, I've remembered that: "${fact}"`;
    return {
      llmContent: JSON.stringify({ success: true, message: successMessage }),
      returnDisplay: successMessage,
    };
  } catch (error) {
    const errorMessage = error instanceof Error ? error.message : String(error);
    console.error(`[MemoryTool] Error executing save_memory for fact "${fact}": ${errorMessage}`);
    
    return {
      llmContent: JSON.stringify({
        success: false,
        error: `Failed to save memory. Detail: ${errorMessage}`,
      }),
      returnDisplay: `Error saving memory: ${errorMessage}`,
    };
  }
}
```

**执行结果**:

- **成功**: 返回确认消息和 JSON 格式的成功状态
- **失败**: 返回错误信息和详细的错误描述

## 使用示例

### 基本记忆保存

```typescript
const memoryTool = new MemoryTool();

// 保存用户偏好
const result = await memoryTool.execute({
  fact: "用户喜欢使用 TypeScript 进行开发"
}, new AbortController().signal);

// 保存项目信息
const result = await memoryTool.execute({
  fact: "当前项目使用 React 18 和 Vite 构建工具"
}, new AbortController().signal);
```

### 个人偏好记忆

```typescript
// 代码风格偏好
await memoryTool.execute({
  fact: "用户偏好使用函数式组件而不是类组件"
});

// 工具链偏好
await memoryTool.execute({
  fact: "用户喜欢使用 ESLint 和 Prettier 进行代码格式化"
});

// 架构偏好
await memoryTool.execute({
  fact: "用户倾向于使用微服务架构"
});
```

### 项目特定信息

```typescript
// 项目配置
await memoryTool.execute({
  fact: "项目使用 PostgreSQL 数据库，端口 5432"
});

// 环境配置
await memoryTool.execute({
  fact: "开发环境 API 地址: https://dev-api.example.com"
});

// 团队约定
await memoryTool.execute({
  fact: "团队使用 Conventional Commits 规范"
});
```

### 技术栈记忆

```typescript
// 前端技术栈
await memoryTool.execute({
  fact: "前端技术栈: React + TypeScript + Tailwind CSS"
});

// 后端技术栈
await memoryTool.execute({
  fact: "后端技术栈: Node.js + Express + Prisma ORM"
});

// 部署信息
await memoryTool.execute({
  fact: "部署平台: Vercel (前端) + Railway (后端)"
});
```

## 文件格式和结构

### Markdown 文件结构

```markdown
# 项目文档

这里是项目的其他内容...

## Gemini Added Memories
- 用户喜欢使用 TypeScript 进行开发
- 当前项目使用 React 18 和 Vite 构建工具
- 用户偏好使用函数式组件而不是类组件
- 项目使用 PostgreSQL 数据库，端口 5432
- 前端技术栈: React + TypeScript + Tailwind CSS

## 其他章节

其他文档内容...
```

### 记忆条目格式

**标准格式**:
```markdown
## Gemini Added Memories
- 这是第一个记忆条目
- 这是第二个记忆条目
- 这是第三个记忆条目
```

**自动格式化**:
```typescript
// 输入: "-- 用户喜欢 Vue.js"
// 处理后: "- 用户喜欢 Vue.js"

// 输入: "项目使用 Docker"
// 处理后: "- 项目使用 Docker"
```

## 高级功能

### 多文件名支持

```typescript
// 设置多个记忆文件
setGeminiMdFilename(['GEMINI.md', 'CONTEXT.md', 'MEMORY.md']);

// 获取当前使用的文件名
const currentFile = getCurrentGeminiMdFilename(); // 返回第一个

// 获取所有文件名
const allFiles = getAllGeminiMdFilenames(); // 返回数组
```

### 自定义记忆工具

```typescript
// 扩展基础 MemoryTool
class ProjectMemoryTool extends MemoryTool {
  private projectContext: string;
  
  constructor(projectContext: string) {
    super();
    this.projectContext = projectContext;
  }
  
  async execute(params: SaveMemoryParams, signal: AbortSignal): Promise<ToolResult> {
    // 为记忆添加项目上下文
    const contextualFact = `[${this.projectContext}] ${params.fact}`;
    
    return super.execute({ fact: contextualFact }, signal);
  }
}

// 使用示例
const projectMemory = new ProjectMemoryTool('E-commerce Platform');
await projectMemory.execute({
  fact: "使用 Stripe 作为支付处理器"
});
// 保存为: "[E-commerce Platform] 使用 Stripe 作为支付处理器"
```

### 记忆分类管理

```typescript
class CategorizedMemoryTool extends MemoryTool {
  async saveToCategory(category: string, fact: string): Promise<ToolResult> {
    const categorizedFact = `${category}: ${fact}`;
    return this.execute({ fact: categorizedFact }, new AbortController().signal);
  }
}

const categorizedMemory = new CategorizedMemoryTool();

// 技术偏好分类
await categorizedMemory.saveToCategory('Tech Preference', '喜欢使用 Vue.js');

// 项目配置分类  
await categorizedMemory.saveToCategory('Project Config', 'API 端口 3000');

// 个人信息分类
await categorizedMemory.saveToCategory('Personal', '时区: UTC+8');
```

### 记忆去重功能

```typescript
class DeduplicatedMemoryTool extends MemoryTool {
  private memoryCache = new Set<string>();
  
  async execute(params: SaveMemoryParams, signal: AbortSignal): Promise<ToolResult> {
    const normalizedFact = params.fact.trim().toLowerCase();
    
    if (this.memoryCache.has(normalizedFact)) {
      return {
        llmContent: JSON.stringify({ 
          success: false, 
          error: 'This fact is already remembered' 
        }),
        returnDisplay: 'Already remembered this fact',
      };
    }
    
    this.memoryCache.add(normalizedFact);
    return super.execute(params, signal);
  }
  
  async loadExistingMemories(): Promise<void> {
    try {
      const content = await fs.readFile(getGlobalMemoryFilePath(), 'utf-8');
      const memorySection = this.extractMemorySection(content);
      
      memorySection.forEach(fact => {
        this.memoryCache.add(fact.toLowerCase());
      });
    } catch (error) {
      // 文件不存在或读取失败，忽略
    }
  }
  
  private extractMemorySection(content: string): string[] {
    const headerIndex = content.indexOf(MEMORY_SECTION_HEADER);
    if (headerIndex === -1) return [];
    
    const startOfSection = headerIndex + MEMORY_SECTION_HEADER.length;
    let endOfSection = content.indexOf('\n## ', startOfSection);
    if (endOfSection === -1) endOfSection = content.length;
    
    const sectionContent = content.substring(startOfSection, endOfSection);
    
    return sectionContent
      .split('\n')
      .filter(line => line.trim().startsWith('- '))
      .map(line => line.replace(/^- /, '').trim())
      .filter(fact => fact.length > 0);
  }
}
```

## 工具使用指南

### 何时使用 MemoryTool

**应该使用的场景**:

```typescript
// 1. 用户明确要求记住某事
"请记住我喜欢使用深色主题"
await memoryTool.execute({ fact: "用户喜欢使用深色主题" });

// 2. 重要的个人偏好
"我总是使用 yarn 而不是 npm"
await memoryTool.execute({ fact: "用户偏好使用 yarn 而不是 npm" });

// 3. 项目特定配置
"我们的 API 基础 URL 是 https://api.myproject.com"
await memoryTool.execute({ fact: "项目 API 基础 URL: https://api.myproject.com" });

// 4. 工作流偏好
"我喜欢先写测试再写实现代码"
await memoryTool.execute({ fact: "用户偏好测试驱动开发 (TDD)" });
```

**不应该使用的场景**:

```typescript
// 1. 临时的会话信息
// ❌ 不要保存: "我刚才问了关于 React hooks 的问题"

// 2. 长篇复杂的文本
// ❌ 不要保存整个文档或代码块

// 3. 不确定的信息
// ❌ 不要保存: "我可能会使用 MongoDB"

// 4. 过于具体的临时信息
// ❌ 不要保存: "今天我在调试第 45 行的错误"
```

### 最佳实践

#### 记忆内容格式

```typescript
// ✅ 推荐：清晰、简洁的陈述
await memoryTool.execute({ fact: "用户使用 macOS 开发环境" });
await memoryTool.execute({ fact: "项目部署在 AWS EC2" });
await memoryTool.execute({ fact: "团队使用 Git Flow 工作流" });

// ❌ 避免：模糊或冗长的描述
// "用户说他们可能在某些情况下会考虑使用 macOS，但也不确定..."
```

#### 分类组织

```typescript
// 按类型组织记忆
const categories = {
  personal: "个人偏好: ",
  technical: "技术栈: ",
  project: "项目配置: ",
  workflow: "工作流: "
};

await memoryTool.execute({ 
  fact: `${categories.personal}喜欢使用函数式编程` 
});

await memoryTool.execute({ 
  fact: `${categories.technical}前端使用 Next.js` 
});
```

#### 版本化记忆

```typescript
// 为记忆添加时间戳或版本信息
const timestamp = new Date().toISOString().split('T')[0];

await memoryTool.execute({ 
  fact: `项目配置 (${timestamp}): 使用 Node.js 18.x` 
});
```

## 错误处理

### 常见错误场景

#### 参数验证错误

```typescript
// 空事实
Error: Parameter "fact" must be a non-empty string.

// 无效类型
Error: Parameter "fact" must be a non-empty string.
```

#### 文件系统错误

```typescript
// 权限不足
Error: Failed to add memory entry: EACCES: permission denied

// 磁盘空间不足
Error: Failed to add memory entry: ENOSPC: no space left on device

// 路径不存在
Error: Failed to add memory entry: ENOENT: no such file or directory
```

### 错误恢复策略

```typescript
class RobustMemoryTool extends MemoryTool {
  async execute(params: SaveMemoryParams, signal: AbortSignal): Promise<ToolResult> {
    // 多次重试机制
    const maxRetries = 3;
    let lastError: Error | null = null;
    
    for (let attempt = 1; attempt <= maxRetries; attempt++) {
      try {
        return await super.execute(params, signal);
      } catch (error) {
        lastError = error instanceof Error ? error : new Error(String(error));
        
        if (attempt < maxRetries) {
          // 等待后重试
          await new Promise(resolve => setTimeout(resolve, 1000 * attempt));
          continue;
        }
      }
    }
    
    // 所有重试都失败，尝试备用方案
    return this.fallbackSave(params.fact, lastError!);
  }
  
  private async fallbackSave(fact: string, originalError: Error): Promise<ToolResult> {
    try {
      // 尝试保存到临时位置
      const tempPath = path.join(require('os').tmpdir(), 'gemini-memory-backup.md');
      const backupContent = `# Backup Memory\n\n## Failed Saves\n- ${fact}\n`;
      
      await fs.writeFile(tempPath, backupContent, 'utf-8');
      
      return {
        llmContent: JSON.stringify({ 
          success: false, 
          error: `Primary save failed, backed up to: ${tempPath}`,
          originalError: originalError.message
        }),
        returnDisplay: `Memory backed up due to error: ${originalError.message}`,
      };
    } catch (backupError) {
      return {
        llmContent: JSON.stringify({ 
          success: false, 
          error: `Complete save failure: ${originalError.message}` 
        }),
        returnDisplay: `Unable to save memory: ${originalError.message}`,
      };
    }
  }
}
```

## 集成特性

### 与配置系统集成

```typescript
// 从配置文件读取记忆文件设置
class ConfigurableMemoryTool extends MemoryTool {
  constructor(private config: Config) {
    super();
    
    // 从配置中读取自定义文件名
    const customFilename = config.getMemoryFilename();
    if (customFilename) {
      setGeminiMdFilename(customFilename);
    }
  }
  
  async execute(params: SaveMemoryParams, signal: AbortSignal): Promise<ToolResult> {
    // 检查是否启用记忆功能
    if (!this.config.isMemoryEnabled()) {
      return {
        llmContent: JSON.stringify({ 
          success: false, 
          error: 'Memory feature is disabled' 
        }),
        returnDisplay: 'Memory feature is disabled',
      };
    }
    
    return super.execute(params, signal);
  }
}
```

### 与其他工具协作

#### 与文档工具集成

```typescript
// 结合文档生成自动创建记忆
class DocumentationMemoryTool extends MemoryTool {
  async rememberFromDocumentation(docContent: string): Promise<ToolResult[]> {
    const results: ToolResult[] = [];
    
    // 提取重要信息
    const techStack = this.extractTechStack(docContent);
    const projectConfig = this.extractProjectConfig(docContent);
    const conventions = this.extractConventions(docContent);
    
    // 保存提取的信息
    for (const tech of techStack) {
      const result = await this.execute({ fact: `技术栈: ${tech}` }, new AbortController().signal);
      results.push(result);
    }
    
    for (const config of projectConfig) {
      const result = await this.execute({ fact: `项目配置: ${config}` }, new AbortController().signal);
      results.push(result);
    }
    
    return results;
  }
  
  private extractTechStack(content: string): string[] {
    // 实现技术栈提取逻辑
    const techPatterns = [
      /使用\s+(React|Vue|Angular|Node\.js|Express|FastAPI)/gi,
      /基于\s+(TypeScript|JavaScript|Python|Java)/gi,
      /采用\s+(PostgreSQL|MySQL|MongoDB|Redis)/gi,
    ];
    
    const matches: string[] = [];
    techPatterns.forEach(pattern => {
      const found = content.match(pattern);
      if (found) {
        matches.push(...found);
      }
    });
    
    return matches;
  }
  
  private extractProjectConfig(content: string): string[] {
    // 实现配置提取逻辑
    return [];
  }
  
  private extractConventions(content: string): string[] {
    // 实现约定提取逻辑
    return [];
  }
}
```

## 性能优化

### 批量记忆操作

```typescript
class BatchMemoryTool extends MemoryTool {
  async executeBatch(facts: string[]): Promise<ToolResult[]> {
    const results: ToolResult[] = [];
    
    // 批量处理以减少文件 I/O
    const memoryFilePath = getGlobalMemoryFilePath();
    
    try {
      // 一次性读取文件
      let content = '';
      try {
        content = await fs.readFile(memoryFilePath, 'utf-8');
      } catch (_e) {
        // 文件不存在
      }
      
      // 批量添加所有事实
      for (const fact of facts) {
        if (!fact || fact.trim() === '') continue;
        
        const processedText = fact.trim().replace(/^(-+\s*)+/, '').trim();
        const newMemoryItem = `- ${processedText}`;
        
        // 添加到内容中
        const headerIndex = content.indexOf(MEMORY_SECTION_HEADER);
        if (headerIndex === -1) {
          const separator = this.ensureNewlineSeparation(content);
          content += `${separator}${MEMORY_SECTION_HEADER}\n${newMemoryItem}\n`;
        } else {
          // 找到插入位置并添加
          const insertPosition = this.findInsertPosition(content, headerIndex);
          const before = content.substring(0, insertPosition);
          const after = content.substring(insertPosition);
          content = `${before}${newMemoryItem}\n${after}`;
        }
        
        results.push({
          llmContent: JSON.stringify({ success: true, message: `Remembered: ${fact}` }),
          returnDisplay: `Remembered: ${fact}`,
        });
      }
      
      // 一次性写入文件
      await fs.mkdir(path.dirname(memoryFilePath), { recursive: true });
      await fs.writeFile(memoryFilePath, content, 'utf-8');
      
    } catch (error) {
      const errorMessage = error instanceof Error ? error.message : String(error);
      results.push({
        llmContent: JSON.stringify({ success: false, error: errorMessage }),
        returnDisplay: `Batch save error: ${errorMessage}`,
      });
    }
    
    return results;
  }
  
  private findInsertPosition(content: string, headerIndex: number): number {
    const startOfSectionContent = headerIndex + MEMORY_SECTION_HEADER.length;
    let endOfSectionIndex = content.indexOf('\n## ', startOfSectionContent);
    if (endOfSectionIndex === -1) {
      endOfSectionIndex = content.length;
    }
    return endOfSectionIndex;
  }
  
  private ensureNewlineSeparation(content: string): string {
    // 实现换行分隔逻辑
    return ensureNewlineSeparation(content);
  }
}
```

### 内存缓存优化

```typescript
class CachedMemoryTool extends MemoryTool {
  private static fileContentCache = new Map<string, { content: string, mtime: number }>();
  
  protected async readFileWithCache(filePath: string): Promise<string> {
    try {
      const stats = await fs.stat(filePath);
      const cached = CachedMemoryTool.fileContentCache.get(filePath);
      
      if (cached && cached.mtime === stats.mtimeMs) {
        return cached.content;
      }
      
      const content = await fs.readFile(filePath, 'utf-8');
      CachedMemoryTool.fileContentCache.set(filePath, {
        content,
        mtime: stats.mtimeMs
      });
      
      return content;
    } catch (error) {
      throw error;
    }
  }
  
  static clearCache(): void {
    CachedMemoryTool.fileContentCache.clear();
  }
}
```

## 最佳实践总结

### 内容质量最佳实践

```typescript
// ✅ 推荐的记忆格式
const goodMemories = [
  "用户偏好使用 TypeScript 开发",
  "项目使用 PostgreSQL 数据库",
  "团队遵循 Conventional Commits 规范",
  "部署环境: AWS EC2 + Docker",
  "API 基础路径: /api/v1"
];

// ❌ 避免的记忆格式
const badMemories = [
  "用户刚才问了一个关于 React 的问题",  // 临时信息
  "这里是一个很长的代码片段...",      // 过长内容
  "用户可能会喜欢 Vue.js",           // 不确定信息
  "今天调试了一个奇怪的 bug"         // 过于具体的临时信息
];
```

### 分类管理最佳实践

```typescript
// 使用前缀进行分类
const categorizedMemories = {
  personal: "个人偏好: 喜欢使用深色主题",
  technical: "技术选择: 使用 React + TypeScript",
  project: "项目配置: MongoDB 连接端口 27017",
  workflow: "工作流程: 使用 Git Flow 分支策略",
  environment: "环境设置: 开发环境使用 Docker"
};
```

### 安全性最佳实践

```typescript
// ❌ 避免保存敏感信息
const sensitiveInfo = [
  "API 密钥: sk-1234567890abcdef",    // API 密钥
  "数据库密码: mypassword123",        // 密码
  "用户邮箱: user@example.com"       // 个人身份信息
];

// ✅ 推荐保存的配置信息
const safeInfo = [
  "使用环境变量管理敏感配置",
  "数据库连接使用连接池",
  "API 认证使用 JWT 令牌"
];
```