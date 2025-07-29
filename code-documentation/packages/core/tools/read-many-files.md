# ReadManyFilesTool 批量文件读取工具文档

## 概述

`ReadManyFilesTool` 是 Gemini CLI 的批量文件读取工具，专门用于一次性读取多个文件的内容并将其合并为单个输出。它支持 glob 模式匹配、智能文件过滤、多种文件格式处理（文本、图片、PDF），是分析代码库、文档集合或配置文件的强大工具。

## 主要功能

- 批量文件读取和内容合并
- Glob 模式匹配支持（`**/*.js`, `src/**/*.ts` 等）
- 智能文件类型检测和处理
- 多种文件格式支持（文本、图片、PDF）
- 默认排除模式（node_modules、.git、二进制文件等）
- Git/Gemini 忽略文件集成
- 递归目录遍历
- 文件内容分隔符自定义
- 遥测和性能监控

## 接口定义

### `ReadManyFilesParams`

**功能**: ReadManyFiles 工具参数接口

```typescript
export interface ReadManyFilesParams {
  paths: string[];                    // 文件路径或目录路径数组（必需）
  include?: string[];                 // 额外包含的 glob 模式（可选）
  exclude?: string[];                 // 排除的 glob 模式（可选）
  recursive?: boolean;                // 是否递归搜索（可选，默认 true）
  useDefaultExcludes?: boolean;       // 是否应用默认排除模式（可选，默认 true）
  file_filtering_options?: {          // 文件过滤选项（可选）
    respect_git_ignore?: boolean;       // 是否遵循 .gitignore（默认 true）
    respect_gemini_ignore?: boolean;    // 是否遵循 .geminiignore（默认 true）
  };
}
```

**参数详情**:

- `paths`: 相对于目标目录的文件路径或 glob 模式数组
- `include`: 额外的包含模式，与 paths 合并
- `exclude`: 排除模式，添加到默认排除列表中
- `recursive`: 是否递归搜索（主要通过 `**` 控制）
- `useDefaultExcludes`: 是否应用默认排除模式
- `file_filtering_options`: Git/Gemini 忽略文件配置

### 默认排除模式

```typescript
const DEFAULT_EXCLUDES: string[] = [
  '**/node_modules/**',    // Node.js 依赖
  '**/.git/**',           // Git 目录
  '**/.vscode/**',        // VS Code 配置
  '**/.idea/**',          // IntelliJ IDEA 配置
  '**/dist/**',           // 构建输出
  '**/build/**',          // 构建目录
  '**/coverage/**',       // 测试覆盖率
  '**/__pycache__/**',    // Python 缓存
  '**/*.pyc',             // Python 编译文件
  '**/*.bin',             // 二进制文件
  '**/*.exe',             // 可执行文件
  '**/*.dll',             // 动态链接库
  '**/*.zip',             // 压缩文件
  '**/*.tar',             // 归档文件
  '**/*.doc',             // Office 文档
  '**/*.docx',            // Word 文档
  '**/.DS_Store',         // macOS 系统文件
  '**/.env',              // 环境变量文件
];
```

## 核心方法

### 构造函数

#### `constructor(config: Config)`

**功能**: 创建 ReadManyFilesTool 实例

```typescript
constructor(private config: Config) {
  super(
    ReadManyFilesTool.Name,
    'ReadManyFiles',
    `Reads content from multiple files specified by paths or glob patterns within a configured target directory...`,
    Icon.FileSearch,
    parameterSchema,
  );
}
```

**工具特性**:

- 支持文本文件内容合并
- 图片和 PDF 文件的特殊处理
- 智能编码检测和转换
- 内容分隔符格式化

### 参数验证

#### `validateParams(params: ReadManyFilesParams): string | null`

**功能**: 验证工具参数

```typescript
validateParams(params: ReadManyFilesParams): string | null {
  const errors = SchemaValidator.validate(this.schema.parameters, params);
  if (errors) {
    return errors;
  }
  return null;
}
```

**验证项目**:

1. **Schema 验证**: 验证参数结构和类型
2. **路径格式**: 确保路径格式正确
3. **模式有效性**: 验证 glob 模式语法

### 描述生成

#### `getDescription(params: ReadManyFilesParams): string`

**功能**: 生成操作描述信息

```typescript
getDescription(params: ReadManyFilesParams): string {
  const allPatterns = [...params.paths, ...(params.include || [])];
  const pathDesc = `using patterns: \`${allPatterns.join('`, `')}\` (within target directory: \`${this.config.getTargetDir()}\`)`;

  // 确定最终的排除模式列表
  const paramExcludes = params.exclude || [];
  const paramUseDefaultExcludes = params.useDefaultExcludes !== false;
  const geminiIgnorePatterns = this.config
    .getFileService()
    .getGeminiIgnorePatterns();
  const finalExclusionPatternsForDescription: string[] =
    paramUseDefaultExcludes
      ? [...DEFAULT_EXCLUDES, ...paramExcludes, ...geminiIgnorePatterns]
      : [...paramExcludes, ...geminiIgnorePatterns];

  let excludeDesc = `Excluding: ${finalExclusionPatternsForDescription.length > 0 ? `patterns like \`${finalExclusionPatternsForDescription.slice(0, 2).join('`, `')}${finalExclusionPatternsForDescription.length > 2 ? '...`' : '`'}` : 'none specified'}`;

  return `Will attempt to read and concatenate files ${pathDesc}. ${excludeDesc}. File encoding: ${DEFAULT_ENCODING}. Separator: "${DEFAULT_OUTPUT_SEPARATOR_FORMAT.replace('{filePath}', 'path/to/file.ext')}".`;
}
```

### 主要执行方法

#### `async execute(params: ReadManyFilesParams, signal: AbortSignal): Promise<ToolResult>`

**功能**: 执行批量文件读取操作

**执行流程**:

1. **参数验证**: 验证输入参数
2. **模式解析**: 处理 glob 模式和包含/排除规则
3. **文件发现**: 使用 glob 匹配查找文件
4. **文件过滤**: 应用忽略规则和默认排除
5. **内容读取**: 批量读取文件内容
6. **内容合并**: 使用分隔符合并所有内容
7. **遥测记录**: 记录操作指标

```typescript
async execute(
  params: ReadManyFilesParams,
  signal: AbortSignal,
): Promise<ToolResult> {
  const validationError = this.validateParams(params);
  if (validationError) {
    return {
      llmContent: `Error: Invalid parameters for ${this.displayName}. Reason: ${validationError}`,
      returnDisplay: `## Parameter Error\n\n${validationError}`,
    };
  }

  const {
    paths: inputPatterns,
    include = [],
    exclude = [],
    useDefaultExcludes = true,
  } = params;

  // 配置文件过滤选项
  const defaultFileIgnores =
    this.config.getFileFilteringOptions() ?? DEFAULT_FILE_FILTERING_OPTIONS;

  const fileFilteringOptions = {
    respectGitIgnore:
      params.file_filtering_options?.respect_git_ignore ??
      defaultFileIgnores.respectGitIgnore,
    respectGeminiIgnore:
      params.file_filtering_options?.respect_gemini_ignore ??
      defaultFileIgnores.respectGeminiIgnore,
  };

  // 获取文件发现服务
  const fileDiscovery = this.config.getFileService();

  const filesToConsider = new Set<string>();
  const skippedFiles: Array<{ path: string; reason: string }> = [];
  const processedFilesRelativePaths: string[] = [];
  const contentParts: PartListUnion = [];

  // 构建有效的排除模式
  const effectiveExcludes = useDefaultExcludes
    ? [...DEFAULT_EXCLUDES, ...exclude]
    : [...exclude];

  const searchPatterns = [...inputPatterns, ...include];
  if (searchPatterns.length === 0) {
    return {
      llmContent: 'No search paths or include patterns provided.',
      returnDisplay: 'No files to read',
    };
  }

  // 使用 glob 查找文件
  for (const pattern of searchPatterns) {
    try {
      const resolvedPattern = path.resolve(this.config.getTargetDir(), pattern);
      const foundFiles = await glob(resolvedPattern, {
        ignore: effectiveExcludes,
        nodir: true, // 仅文件，不包括目录
        absolute: true,
        followSymbolicLinks: false,
      });

      for (const file of foundFiles) {
        filesToConsider.add(file);
      }
    } catch (error) {
      const errorMessage = getErrorMessage(error);
      skippedFiles.push({
        path: pattern,
        reason: `Glob error: ${errorMessage}`,
      });
    }
  }

  // 应用 Git/Gemini 忽略规则
  const filesToProcess = Array.from(filesToConsider).filter((file) => {
    const relativePath = path.relative(this.config.getTargetDir(), file);

    if (
      fileFilteringOptions.respectGitIgnore &&
      fileDiscovery.shouldGitIgnoreFile(relativePath)
    ) {
      skippedFiles.push({
        path: relativePath,
        reason: 'Git ignored',
      });
      return false;
    }

    if (
      fileFilteringOptions.respectGeminiIgnore &&
      fileDiscovery.shouldGeminiIgnoreFile(relativePath)
    ) {
      skippedFiles.push({
        path: relativePath,
        reason: 'Gemini ignored',
      });
      return false;
    }

    return true;
  });

  // 处理每个文件
  for (const filePath of filesToProcess) {
    if (signal.aborted) {
      break;
    }

    try {
      const relativePath = path.relative(this.config.getTargetDir(), filePath);
      
      // 检测文件类型
      const fileType = await detectFileType(filePath);
      
      // 处理文件内容
      const processedContent = await processSingleFileContent(
        filePath,
        fileType,
        DEFAULT_ENCODING,
      );

      if (processedContent && processedContent.length > 0) {
        // 添加文件分隔符
        const separator = DEFAULT_OUTPUT_SEPARATOR_FORMAT.replace(
          '{filePath}',
          relativePath,
        );
        
        contentParts.push({ text: `\n${separator}\n` });
        contentParts.push(...processedContent);
        
        processedFilesRelativePaths.push(relativePath);

        // 记录遥测数据
        const lines = processedContent
          .filter(part => part.text)
          .map(part => part.text!.split('\n').length)
          .reduce((a, b) => a + b, 0);
        
        const mimetype = getSpecificMimeType(filePath);
        const extension = path.extname(filePath);
        
        recordFileOperationMetric(
          this.config,
          FileOperation.READ,
          lines,
          mimetype,
          extension,
        );
      }
    } catch (error) {
      const errorMessage = getErrorMessage(error);
      const relativePath = path.relative(this.config.getTargetDir(), filePath);
      skippedFiles.push({
        path: relativePath,
        reason: `Read error: ${errorMessage}`,
      });
    }
  }

  // 生成结果摘要
  let resultSummary = `Successfully read ${processedFilesRelativePaths.length} files`;
  if (skippedFiles.length > 0) {
    resultSummary += `, skipped ${skippedFiles.length} files`;
  }

  const llmContent = contentParts.length > 0 
    ? contentParts
    : 'No file content was successfully read.';

  return {
    llmContent,
    returnDisplay: resultSummary,
  };
}
```

## 使用示例

### 基本文件读取

```typescript
const readManyFilesTool = new ReadManyFilesTool(config);

// 读取所有 TypeScript 文件
const result = await readManyFilesTool.execute({
  paths: ["src/**/*.ts"]
});

// 读取特定文件
const result = await readManyFilesTool.execute({
  paths: ["package.json", "README.md", "src/index.ts"]
});
```

### 代码库分析

```typescript
// 分析整个源代码目录
const result = await readManyFilesTool.execute({
  paths: ["src/**/*.{js,ts,tsx,jsx}"],
  exclude: ["**/*.test.*", "**/*.spec.*"]
});

// 读取配置文件
const result = await readManyFilesTool.execute({
  paths: ["*.json", "*.yaml", "*.yml", ".env.example"],
  useDefaultExcludes: false
});
```

### 文档集合处理

```typescript
// 读取所有文档文件
const result = await readManyFilesTool.execute({
  paths: ["docs/**/*.md", "*.md"],
  include: ["CHANGELOG.md", "CONTRIBUTING.md"]
});

// 读取API文档
const result = await readManyFilesTool.execute({
  paths: ["docs/api/**/*.md"],
  exclude: ["docs/api/deprecated/**"]
});
```

### 高级过滤和包含

```typescript
// 复杂的包含和排除规则
const result = await readManyFilesTool.execute({
  paths: ["src/**/*"],
  include: ["tests/**/*.test.ts"],
  exclude: ["**/*.min.js", "**/*.bundle.js", "src/legacy/**"],
  useDefaultExcludes: true
});

// 禁用默认排除
const result = await readManyFilesTool.execute({
  paths: ["**/*"],
  exclude: ["*.log"],
  useDefaultExcludes: false
});
```

### 文件过滤选项

```typescript
// 自定义忽略文件处理
const result = await readManyFilesTool.execute({
  paths: ["**/*.js"],
  file_filtering_options: {
    respect_git_ignore: false,  // 忽略 .gitignore 规则
    respect_gemini_ignore: true // 仍然应用 .geminiignore 规则
  }
});

// 完全禁用忽略文件
const result = await readManyFilesTool.execute({
  paths: ["**/*"],
  file_filtering_options: {
    respect_git_ignore: false,
    respect_gemini_ignore: false
  },
  useDefaultExcludes: false
});
```

## 文件类型处理

### 文本文件

**支持的格式**:
- 源代码文件（.js, .ts, .py, .java, .cpp, .go 等）
- 配置文件（.json, .yaml, .xml, .ini 等）
- 文档文件（.md, .txt, .rst 等）
- 样式文件（.css, .scss, .less 等）

**处理方式**:
- 使用 UTF-8 编码读取
- 自动检测编码格式
- 保留原始格式和换行符

### 图片文件

**支持的格式**: .png, .jpg, .jpeg, .gif, .bmp, .webp

**处理方式**:
```typescript
// 图片文件自动转换为 base64
const result = await readManyFilesTool.execute({
  paths: ["assets/images/*.png", "docs/screenshots/*.jpg"]
});
```

### PDF 文件

**处理方式**:
```typescript
// PDF 文件内容提取
const result = await readManyFilesTool.execute({
  paths: ["docs/**/*.pdf"]
});
```

### 二进制文件

**默认行为**: 自动跳过二进制文件，除非明确指定

**强制包含**:
```typescript
// 明确指定要处理的二进制文件
const result = await readManyFilesTool.execute({
  paths: ["config.bin", "data.dat"],
  useDefaultExcludes: false
});
```

## 性能优化

### 文件数量限制

```typescript
// 对于大型代码库，使用更具体的模式
const result = await readManyFilesTool.execute({
  paths: ["src/components/**/*.tsx"], // 具体目录
  exclude: ["**/*.test.*"]            // 排除测试文件
});
```

### 内存管理

```typescript
// 分批处理大量文件
class BatchReadManyFilesTool extends ReadManyFilesTool {
  async executeBatch(
    patterns: string[], 
    batchSize: number = 50
  ): Promise<ToolResult[]> {
    const results: ToolResult[] = [];
    
    for (let i = 0; i < patterns.length; i += batchSize) {
      const batch = patterns.slice(i, i + batchSize);
      const result = await this.execute({ paths: batch }, new AbortController().signal);
      results.push(result);
    }
    
    return results;
  }
}
```

### 缓存策略

```typescript
// 实现文件内容缓存
class CachedReadManyFilesTool extends ReadManyFilesTool {
  private cache = new Map<string, { content: PartListUnion, mtime: number }>();
  
  protected async readFileWithCache(filePath: string): Promise<PartListUnion> {
    const stats = await fs.promises.stat(filePath);
    const cached = this.cache.get(filePath);
    
    if (cached && cached.mtime === stats.mtimeMs) {
      return cached.content;
    }
    
    const content = await this.readFileContent(filePath);
    this.cache.set(filePath, { content, mtime: stats.mtimeMs });
    return content;
  }
}
```

## 高级功能

### 自定义分隔符

```typescript
// 自定义文件分隔符格式
const CUSTOM_SEPARATOR_FORMAT = '=== File: {filePath} ===';

class CustomSeparatorReadTool extends ReadManyFilesTool {
  protected getFileSeparator(filePath: string): string {
    return CUSTOM_SEPARATOR_FORMAT.replace('{filePath}', filePath);
  }
}
```

### 内容预处理

```typescript
// 添加内容预处理功能
class PreprocessingReadTool extends ReadManyFilesTool {
  protected async preprocessContent(
    content: string, 
    filePath: string
  ): Promise<string> {
    // 移除注释
    if (filePath.endsWith('.js') || filePath.endsWith('.ts')) {
      return content.replace(/\/\*[\s\S]*?\*\//g, '').replace(/\/\/.*$/gm, '');
    }
    
    // 压缩空白行
    return content.replace(/\n\s*\n\s*\n/g, '\n\n');
  }
}
```

### 文件统计

```typescript
// 添加详细统计信息
interface FileStatistics {
  totalFiles: number;
  processedFiles: number;
  skippedFiles: number;
  totalSize: number;
  fileTypes: Map<string, number>;
}

class StatisticsReadTool extends ReadManyFilesTool {
  async executeWithStats(
    params: ReadManyFilesParams,
    signal: AbortSignal
  ): Promise<{ result: ToolResult; stats: FileStatistics }> {
    const stats: FileStatistics = {
      totalFiles: 0,
      processedFiles: 0,
      skippedFiles: 0,
      totalSize: 0,
      fileTypes: new Map(),
    };
    
    // 执行读取并收集统计
    const result = await this.execute(params, signal);
    
    return { result, stats };
  }
}
```

## 错误处理

### 常见错误场景

#### Glob 模式错误

```typescript
// 无效的 glob 模式
Error: Invalid glob pattern: [invalid

// 路径不存在
Error: Pattern matches no files: nonexistent/**/*.js
```

#### 文件访问错误

```typescript
// 权限错误
Error: Permission denied reading file: /restricted/file.txt

// 文件过大
Error: File too large to process: large-file.json (>10MB)
```

#### 编码问题

```typescript
// 字符编码错误
Error: Invalid character encoding in file: binary-file.dat

// 内容格式错误
Error: Unable to process file content: corrupted-file.pdf
```

### 错误恢复策略

```typescript
// 实现错误恢复
class RobustReadManyFilesTool extends ReadManyFilesTool {
  async execute(
    params: ReadManyFilesParams,
    signal: AbortSignal
  ): Promise<ToolResult> {
    try {
      return await super.execute(params, signal);
    } catch (error) {
      // 记录错误但继续处理其他文件
      console.error('ReadManyFiles error:', error);
      
      // 返回部分结果
      return {
        llmContent: `Partial read completed. Error: ${error.message}`,
        returnDisplay: 'Partial read with errors'
      };
    }
  }
}
```

## 集成特性

### 与其他工具协作

#### 与 GrepTool 配合

```typescript
// 先搜索再读取
const grepResult = await grepTool.execute({
  pattern: "function.*export",
  path: "src",
  include: "*.ts"
});

// 基于搜索结果读取相关文件
const readResult = await readManyFilesTool.execute({
  paths: extractFilePathsFromGrepResult(grepResult)
});
```

#### 与 EditTool 配合

```typescript
// 读取多个文件，然后批量编辑
const readResult = await readManyFilesTool.execute({
  paths: ["src/**/*.js"]
});

const filesToEdit = analyzeContentForEdits(readResult.llmContent);
for (const fileEdit of filesToEdit) {
  await editTool.execute(fileEdit);
}
```

### 配置集成

#### 项目级配置

```typescript
// 在 .geminiconfig 中配置
{
  "readManyFiles": {
    "defaultExcludes": ["**/*.log", "temp/**"],
    "maxFiles": 100,
    "maxFileSize": "1MB"
  }
}
```

#### 用户级配置

```typescript
// 全局配置文件
{
  "tools": {
    "readManyFiles": {
      "respectGitIgnore": true,
      "useDefaultExcludes": true,
      "encoding": "utf-8"
    }
  }
}
```

## 最佳实践

### 性能最佳实践

```typescript
// 推荐：使用具体的模式
paths: ["src/components/**/*.tsx", "src/utils/**/*.ts"]

// 避免：过于宽泛的模式
paths: ["**/*"]

// 推荐：排除不必要的文件
exclude: ["**/*.test.*", "**/node_modules/**", "**/*.min.js"]

// 推荐：使用合适的递归设置
recursive: false  // 当只需要直接子文件时
```

### 安全最佳实践

```typescript
// 推荐：限制文件大小
if (fileSize > MAX_FILE_SIZE) {
  skipFile(filePath, 'File too large');
}

// 推荐：验证文件类型
const allowedExtensions = ['.js', '.ts', '.md', '.json'];
if (!allowedExtensions.includes(path.extname(filePath))) {
  skipFile(filePath, 'File type not allowed');
}

// 推荐：使用安全的编码
const content = await fs.readFile(filePath, { encoding: 'utf8' });
```

### 可维护性建议

```typescript
// 推荐：使用描述性的模式
const CONFIG_FILES = ["*.json", "*.yaml", "*.yml"];
const SOURCE_FILES = ["src/**/*.{js,ts,tsx,jsx}"];
const DOC_FILES = ["docs/**/*.md", "README.md"];

await readManyFilesTool.execute({
  paths: [...CONFIG_FILES, ...SOURCE_FILES, ...DOC_FILES]
});

// 推荐：模块化排除规则
const COMMON_EXCLUDES = ["**/*.test.*", "**/*.spec.*"];
const BUILD_EXCLUDES = ["**/dist/**", "**/build/**"];
const CACHE_EXCLUDES = ["**/.cache/**", "**/node_modules/**"];

const ALL_EXCLUDES = [...COMMON_EXCLUDES, ...BUILD_EXCLUDES, ...CACHE_EXCLUDES];
```