# GlobTool 文件发现工具文档

## 概述

`GlobTool` 是 Gemini CLI 的高效文件发现工具，专门用于基于 glob 模式匹配快速定位文件。它支持复杂的路径模式匹配（如 `src/**/*.ts`, `**/*.md`），返回按修改时间排序的绝对路径列表。这是在大型代码库中快速定位特定类型文件的理想工具。

## 主要功能

- 高效的 glob 模式文件搜索
- 智能文件排序（最近修改优先）
- Git 忽略文件集成（.gitignore 支持）
- 大小写敏感/不敏感匹配
- 安全的路径验证和边界检查
- 递归目录搜索支持
- 性能优化的文件发现算法

## 接口定义

### `GlobToolParams`

**功能**: GlobTool 工具参数接口

```typescript
export interface GlobToolParams {
  pattern: string;                    // Glob 模式（必需）
  path?: string;                      // 搜索目录路径（可选）
  case_sensitive?: boolean;           // 大小写敏感（可选，默认 false）
  respect_git_ignore?: boolean;       // 是否遵循 .gitignore（可选，默认 true）
}
```

**参数详情**:

- `pattern`: Glob 模式字符串，如 `**/*.py`, `docs/*.md`, `src/**/*.{js,ts}`
- `path`: 搜索的起始目录，相对于目标目录（省略时从根目录搜索）
- `case_sensitive`: 是否大小写敏感匹配（默认 false）
- `respect_git_ignore`: 是否应用 Git 忽略规则（默认 true）

### `GlobPath` 接口

**功能**: Glob 结果路径对象接口

```typescript
export interface GlobPath {
  fullpath(): string;                 // 返回完整路径
  mtimeMs?: number;                   // 修改时间戳（毫秒）
}
```

## 核心方法

### 构造函数

#### `constructor(config: Config)`

**功能**: 创建 GlobTool 实例

```typescript
constructor(private config: Config) {
  super(
    GlobTool.Name,
    'FindFiles',
    'Efficiently finds files matching specific glob patterns...',
    Icon.FileSearch,
    parameterSchema,
  );
}
```

**工具特性**:

- 高效的文件匹配算法
- 智能排序（最近修改优先）
- Git 集成和忽略文件支持
- 安全的路径处理

### 参数验证

#### `validateToolParams(params: GlobToolParams): string | null`

**功能**: 验证工具参数

```typescript
validateToolParams(params: GlobToolParams): string | null {
  const errors = SchemaValidator.validate(this.schema.parameters, params);
  if (errors) {
    return errors;
  }

  const searchDirAbsolute = path.resolve(
    this.config.getTargetDir(),
    params.path || '.',
  );

  if (!isWithinRoot(searchDirAbsolute, this.config.getTargetDir())) {
    return `Search path resolves outside the tool's root directory`;
  }

  // 目录存在性和权限验证
  try {
    if (!fs.existsSync(targetDir)) {
      return `Search path does not exist ${targetDir}`;
    }
    if (!fs.statSync(targetDir).isDirectory()) {
      return `Search path is not a directory: ${targetDir}`;
    }
  } catch (e: unknown) {
    return `Error accessing search path: ${e}`;
  }

  return null;
}
```

**验证项目**:

1. **Schema 验证**: 验证参数结构和类型
2. **路径安全**: 确保搜索路径在允许范围内
3. **目录存在**: 验证搜索目录存在且可访问
4. **模式有效性**: 确保 glob 模式非空

### 描述生成

#### `getDescription(params: GlobToolParams): string`

**功能**: 生成操作描述信息

```typescript
getDescription(params: GlobToolParams): string {
  let description = `'${params.pattern}'`;
  if (params.path) {
    const searchDir = path.resolve(
      this.config.getTargetDir(),
      params.path || '.',
    );
    const relativePath = makeRelative(searchDir, this.config.getTargetDir());
    description += ` within ${shortenPath(relativePath)}`;
  }
  return description;
}
```

### 智能排序算法

#### `sortFileEntries(entries: GlobPath[], nowTimestamp: number, recencyThresholdMs: number): GlobPath[]`

**功能**: 基于修改时间和字母顺序的智能文件排序

```typescript
export function sortFileEntries(
  entries: GlobPath[],
  nowTimestamp: number,
  recencyThresholdMs: number,
): GlobPath[] {
  const sortedEntries = [...entries];
  sortedEntries.sort((a, b) => {
    const mtimeA = a.mtimeMs ?? 0;
    const mtimeB = b.mtimeMs ?? 0;
    const aIsRecent = nowTimestamp - mtimeA < recencyThresholdMs;
    const bIsRecent = nowTimestamp - mtimeB < recencyThresholdMs;

    if (aIsRecent && bIsRecent) {
      return mtimeB - mtimeA;  // 最近文件：新到老
    } else if (aIsRecent) {
      return -1;               // 最近文件优先
    } else if (bIsRecent) {
      return 1;                // 最近文件优先
    } else {
      return a.fullpath().localeCompare(b.fullpath());  // 字母顺序
    }
  });
  return sortedEntries;
}
```

**排序策略**:

1. **最近文件优先**: 24小时内修改的文件排在最前面
2. **时间倒序**: 最近文件内按修改时间倒序排列
3. **字母顺序**: 非最近文件按路径字母顺序排列

### 主要执行方法

#### `async execute(params: GlobToolParams, signal: AbortSignal): Promise<ToolResult>`

**功能**: 执行 glob 文件搜索操作

**执行流程**:

1. **参数验证**: 验证输入参数和路径安全性
2. **搜索配置**: 设置 glob 搜索选项
3. **文件发现**: 执行 glob 模式匹配
4. **Git 过滤**: 应用 Git 忽略规则
5. **智能排序**: 按修改时间和路径排序
6. **结果生成**: 生成格式化的文件列表

```typescript
async execute(
  params: GlobToolParams,
  signal: AbortSignal,
): Promise<ToolResult> {
  const validationError = this.validateToolParams(params);
  if (validationError) {
    return {
      llmContent: `Error: Invalid parameters. Reason: ${validationError}`,
      returnDisplay: validationError,
    };
  }

  try {
    const searchDirAbsolute = path.resolve(
      this.config.getTargetDir(),
      params.path || '.',
    );

    // 获取文件发现服务和配置
    const respectGitIgnore =
      params.respect_git_ignore ??
      this.config.getFileFilteringRespectGitIgnore();
    const fileDiscovery = this.config.getFileService();

    // 执行 glob 搜索
    const entries = (await glob(params.pattern, {
      cwd: searchDirAbsolute,
      withFileTypes: true,
      nodir: true,                    // 仅文件，不包括目录
      stat: true,                     // 获取文件统计信息
      nocase: !params.case_sensitive, // 大小写敏感控制
      dot: true,                      // 包括隐藏文件
      ignore: ['**/node_modules/**', '**/.git/**'],  // 默认忽略
      follow: false,                  // 不跟随符号链接
      signal,                        // 支持取消操作
    })) as GlobPath[];

    // 应用 Git 忽略过滤
    let filteredEntries = entries;
    let gitIgnoredCount = 0;

    if (respectGitIgnore) {
      const relativePaths = entries.map((p) =>
        path.relative(this.config.getTargetDir(), p.fullpath()),
      );
      const filteredRelativePaths = fileDiscovery.filterFiles(relativePaths, {
        respectGitIgnore,
      });
      const filteredAbsolutePaths = new Set(
        filteredRelativePaths.map((p) =>
          path.resolve(this.config.getTargetDir(), p),
        ),
      );

      filteredEntries = entries.filter((entry) =>
        filteredAbsolutePaths.has(entry.fullpath()),
      );
      gitIgnoredCount = entries.length - filteredEntries.length;
    }

    // 检查结果
    if (!filteredEntries || filteredEntries.length === 0) {
      let message = `No files found matching pattern "${params.pattern}"`;
      if (gitIgnoredCount > 0) {
        message += ` (${gitIgnoredCount} files were git-ignored)`;
      }
      return {
        llmContent: message,
        returnDisplay: 'No files found',
      };
    }

    // 智能排序
    const oneDayInMs = 24 * 60 * 60 * 1000;
    const nowTimestamp = new Date().getTime();

    const sortedEntries = sortFileEntries(
      filteredEntries,
      nowTimestamp,
      oneDayInMs,
    );

    // 生成结果
    const sortedAbsolutePaths = sortedEntries.map((entry) =>
      entry.fullpath(),
    );
    const fileListDescription = sortedAbsolutePaths.join('\n');
    const fileCount = sortedAbsolutePaths.length;

    let resultMessage = `Found ${fileCount} file(s) matching "${params.pattern}"`;
    if (gitIgnoredCount > 0) {
      resultMessage += ` (${gitIgnoredCount} additional files were git-ignored)`;
    }
    resultMessage += `, sorted by modification time (newest first):\n${fileListDescription}`;

    return {
      llmContent: resultMessage,
      returnDisplay: `Found ${fileCount} matching file(s)`,
    };
  } catch (error) {
    const errorMessage = error instanceof Error ? error.message : String(error);
    return {
      llmContent: `Error during glob search operation: ${errorMessage}`,
      returnDisplay: 'Error: An unexpected error occurred.',
    };
  }
}
```

## 使用示例

### 基本文件查找

```typescript
const globTool = new GlobTool(config);

// 查找所有 TypeScript 文件
const result = await globTool.execute({
  pattern: "**/*.ts"
});

// 查找特定目录下的 JavaScript 文件
const result = await globTool.execute({
  pattern: "*.js",
  path: "src/components"
});
```

### 代码库分析

```typescript
// 查找所有源代码文件
const result = await globTool.execute({
  pattern: "**/*.{js,ts,tsx,jsx}",
  respect_git_ignore: true
});

// 查找测试文件
const result = await globTool.execute({
  pattern: "**/*.{test,spec}.{js,ts}",
  path: "tests"
});
```

### 配置文件发现

```typescript
// 查找配置文件
const result = await globTool.execute({
  pattern: "*.{json,yaml,yml,toml}",
  case_sensitive: false
});

// 查找特定配置
const result = await globTool.execute({
  pattern: "*config*",
  path: "."
});
```

### 文档文件搜索

```typescript
// 查找所有 Markdown 文件
const result = await globTool.execute({
  pattern: "**/*.md"
});

// 查找文档目录下的文件
const result = await globTool.execute({
  pattern: "*.{md,rst,txt}",
  path: "docs"
});
```

### 大小写敏感搜索

```typescript
// 大小写敏感的文件查找
const result = await globTool.execute({
  pattern: "**/README*",
  case_sensitive: true
});

// 大小写不敏感（默认）
const result = await globTool.execute({
  pattern: "**/readme*",
  case_sensitive: false
});
```

## Glob 模式语法

### 基本模式

| 模式 | 描述 | 示例 |
|------|------|------|
| `*` | 匹配任意字符（除路径分隔符） | `*.js` 匹配所有 JS 文件 |
| `**` | 递归匹配目录 | `**/*.ts` 匹配所有子目录中的 TS 文件 |
| `?` | 匹配单个字符 | `file?.txt` 匹配 file1.txt, fileA.txt |
| `[]` | 字符类匹配 | `file[0-9].js` 匹配 file0.js 到 file9.js |
| `{}` | 选择匹配 | `*.{js,ts}` 匹配 .js 和 .ts 文件 |

### 高级模式

```typescript
// 复杂的选择模式
const result = await globTool.execute({
  pattern: "src/**/*.{js,ts,tsx,jsx}"
});

// 排除特定目录
const result = await globTool.execute({
  pattern: "**/*.js",
  // 注意：排除通过 Git 忽略或默认忽略列表处理
});

// 字符类和范围
const result = await globTool.execute({
  pattern: "test[0-9]*.spec.js"
});

// 嵌套目录匹配
const result = await globTool.execute({
  pattern: "**/components/**/*.tsx"
});
```

### 实际应用模式

```typescript
// Web 项目源文件
"src/**/*.{js,jsx,ts,tsx,vue,svelte}"

// 配置文件
"*.{json,yaml,yml,toml,ini,conf}"

// 样式文件
"**/*.{css,scss,sass,less,styl}"

// 文档文件
"**/*.{md,mdx,rst,txt,adoc}"

// 图片文件
"**/*.{png,jpg,jpeg,gif,svg,webp}"

// 测试文件
"**/*.{test,spec}.{js,ts,jsx,tsx}"

// 构建文件
"{webpack,rollup,vite,esbuild}.config.{js,ts}"
```

## 性能优化

### 模式优化策略

```typescript
// 推荐：具体的路径前缀
const result = await globTool.execute({
  pattern: "src/components/**/*.tsx"  // 好
});

// 避免：过于宽泛的模式
const result = await globTool.execute({
  pattern: "**/*.tsx"  // 可能较慢
});
```

### 大型项目优化

```typescript
// 使用具体的目录路径
const result = await globTool.execute({
  pattern: "*.ts",
  path: "src/specific-module"  // 限制搜索范围
});

// 避免在根目录使用 **
const result = await globTool.execute({
  pattern: "components/**/*.vue",  // 好
  path: "src"
});
```

### 批量搜索模式

```typescript
// 实现批量模式搜索
class BatchGlobTool extends GlobTool {
  async executeMultiple(
    patterns: string[], 
    basePath?: string
  ): Promise<ToolResult[]> {
    const results: ToolResult[] = [];
    
    for (const pattern of patterns) {
      const result = await this.execute({
        pattern,
        path: basePath
      }, new AbortController().signal);
      results.push(result);
    }
    
    return results;
  }
}

// 使用示例
const batchGlob = new BatchGlobTool(config);
const results = await batchGlob.executeMultiple([
  "**/*.ts",
  "**/*.js", 
  "**/*.json"
], "src");
```

## 高级功能

### 自定义排序

```typescript
// 自定义排序策略
class CustomSortGlobTool extends GlobTool {
  protected customSort(entries: GlobPath[], sortBy: 'name' | 'size' | 'type'): GlobPath[] {
    return entries.sort((a, b) => {
      switch (sortBy) {
        case 'name':
          return a.fullpath().localeCompare(b.fullpath());
        case 'size':
          // 需要获取文件大小信息
          return 0;
        case 'type':
          const extA = path.extname(a.fullpath());
          const extB = path.extname(b.fullpath());
          return extA.localeCompare(extB);
        default:
          return 0;
      }
    });
  }
}
```

### 过滤增强

```typescript
// 添加自定义过滤器
class FilteredGlobTool extends GlobTool {
  async executeWithFilter(
    params: GlobToolParams & { 
      minSize?: number;
      maxSize?: number;
      extensions?: string[];
    },
    signal: AbortSignal
  ): Promise<ToolResult> {
    const baseResult = await super.execute(params, signal);
    
    if (typeof baseResult.llmContent !== 'string') {
      return baseResult;
    }
    
    // 提取文件路径并应用额外过滤
    const lines = baseResult.llmContent.split('\n');
    const fileLines = lines.filter(line => line.startsWith('/'));
    
    const filteredFiles = await this.applyCustomFilters(
      fileLines, 
      params
    );
    
    return {
      llmContent: filteredFiles.join('\n'),
      returnDisplay: `Found ${filteredFiles.length} filtered file(s)`
    };
  }
  
  private async applyCustomFilters(
    files: string[], 
    filters: any
  ): Promise<string[]> {
    const result: string[] = [];
    
    for (const file of files) {
      try {
        const stats = await fs.promises.stat(file);
        
        // 大小过滤
        if (filters.minSize && stats.size < filters.minSize) continue;
        if (filters.maxSize && stats.size > filters.maxSize) continue;
        
        // 扩展名过滤
        if (filters.extensions) {
          const ext = path.extname(file).toLowerCase();
          if (!filters.extensions.includes(ext)) continue;
        }
        
        result.push(file);
      } catch (error) {
        // 忽略无法访问的文件
        continue;
      }
    }
    
    return result;
  }
}
```

### 结果缓存

```typescript
// 实现结果缓存
class CachedGlobTool extends GlobTool {
  private cache = new Map<string, { result: ToolResult, timestamp: number }>();
  private cacheTimeout = 5 * 60 * 1000; // 5 分钟
  
  async execute(params: GlobToolParams, signal: AbortSignal): Promise<ToolResult> {
    const cacheKey = JSON.stringify(params);
    const cached = this.cache.get(cacheKey);
    
    if (cached && Date.now() - cached.timestamp < this.cacheTimeout) {
      return cached.result;
    }
    
    const result = await super.execute(params, signal);
    this.cache.set(cacheKey, { result, timestamp: Date.now() });
    
    return result;
  }
  
  clearCache(): void {
    this.cache.clear();
  }
}
```

## 错误处理

### 常见错误场景

#### 模式语法错误

```typescript
// 无效的 glob 模式
Error: Invalid glob pattern: [unclosed

// 空模式
Error: The 'pattern' parameter cannot be empty.
```

#### 路径访问错误

```typescript
// 路径不存在
Error: Search path does not exist /nonexistent/path

// 权限不足
Error: Error accessing search path: EACCES: permission denied
```

#### 路径安全错误

```typescript
// 路径超出边界
Error: Search path resolves outside the tool's root directory
```

### 错误恢复策略

```typescript
// 实现错误恢复
class RobustGlobTool extends GlobTool {
  async execute(
    params: GlobToolParams, 
    signal: AbortSignal
  ): Promise<ToolResult> {
    try {
      return await super.execute(params, signal);
    } catch (error) {
      // 记录错误但提供降级方案
      console.error('GlobTool error:', error);
      
      // 尝试简化模式
      if (params.pattern.includes('**')) {
        const simplifiedParams = {
          ...params,
          pattern: params.pattern.replace('**/', '')
        };
        
        try {
          const result = await super.execute(simplifiedParams, signal);
          return {
            ...result,
            llmContent: `Warning: Used simplified pattern due to error.\n${result.llmContent}`
          };
        } catch (fallbackError) {
          // 最终降级
          return {
            llmContent: `Error: Could not execute glob search. ${error.message}`,
            returnDisplay: 'Glob search failed'
          };
        }
      }
      
      throw error;
    }
  }
}
```

## 集成特性

### 与其他工具协作

#### 与 ReadManyFilesTool 配合

```typescript
// 先查找文件，再读取内容
const globResult = await globTool.execute({
  pattern: "src/**/*.ts"
});

// 提取文件路径
const filePaths = extractFilePathsFromGlobResult(globResult);

// 批量读取文件
const readResult = await readManyFilesTool.execute({
  paths: filePaths
});
```

#### 与 GrepTool 配合

```typescript
// 先查找特定类型文件，再搜索内容
const globResult = await globTool.execute({
  pattern: "**/*.{js,ts}"
});

const files = extractFilePathsFromGlobResult(globResult);
for (const file of files) {
  const grepResult = await grepTool.execute({
    pattern: "function.*export",
    path: file
  });
}
```

### 配置集成

#### 项目级配置

```typescript
// 在 .geminiconfig 中配置
{
  "globTool": {
    "defaultIgnorePatterns": ["**/*.log", "temp/**"],
    "caseSensitive": false,
    "respectGitIgnore": true,
    "recencyThreshold": "24h"
  }
}
```

#### 用户级配置

```typescript
// 全局配置文件
{
  "tools": {
    "glob": {
      "defaultPatterns": {
        "source": "**/*.{js,ts,jsx,tsx}",
        "tests": "**/*.{test,spec}.{js,ts}",
        "docs": "**/*.{md,rst,txt}"
      }
    }
  }
}
```

## 最佳实践

### 模式设计最佳实践

```typescript
// 推荐：具体而精确的模式
"src/components/**/*.tsx"          // 好
"src/**/*.{js,ts}"                // 好  
"**/*.test.{js,ts}"               // 好

// 避免：过于宽泛的模式
"**/*"                           // 避免
"*"                              // 避免（在大目录中）
```

### 性能最佳实践

```typescript
// 推荐：限制搜索范围
const result = await globTool.execute({
  pattern: "*.config.js",
  path: "config"                   // 限制在特定目录
});

// 推荐：使用具体的文件扩展名
pattern: "**/*.{js,ts}"           // 好
pattern: "**/*"                   // 避免
```

### 安全最佳实践

```typescript
// 推荐：总是验证路径
if (!params.path || params.path.includes('..')) {
  return { error: 'Invalid path parameter' };
}

// 推荐：使用配置的目标目录
const safePath = path.resolve(this.config.getTargetDir(), params.path || '.');

// 推荐：启用 Git 忽略
const result = await globTool.execute({
  pattern: "**/*.js",
  respect_git_ignore: true         // 推荐总是启用
});
```

### 可维护性建议

```typescript
// 推荐：定义常用模式常量
const COMMON_PATTERNS = {
  SOURCE_FILES: "**/*.{js,ts,jsx,tsx,vue}",
  TEST_FILES: "**/*.{test,spec}.{js,ts}",
  CONFIG_FILES: "*.{json,yaml,yml,toml}",
  DOC_FILES: "**/*.{md,rst,txt}",
  STYLE_FILES: "**/*.{css,scss,sass,less}"
};

// 推荐：创建语义化的搜索方法
class SemanticGlobTool extends GlobTool {
  async findSourceFiles(basePath?: string) {
    return this.execute({
      pattern: COMMON_PATTERNS.SOURCE_FILES,
      path: basePath
    }, new AbortController().signal);
  }
  
  async findTestFiles(basePath?: string) {
    return this.execute({
      pattern: COMMON_PATTERNS.TEST_FILES,
      path: basePath
    }, new AbortController().signal);
  }
  
  async findConfigFiles(basePath?: string) {
    return this.execute({
      pattern: COMMON_PATTERNS.CONFIG_FILES,
      path: basePath
    }, new AbortController().signal);
  }
}
```