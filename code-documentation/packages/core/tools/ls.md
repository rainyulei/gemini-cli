# LSTool 目录列表工具文档

## 概述

`LSTool` 是 Gemini CLI 的目录列表工具，用于列出指定目录下的文件和子目录。它支持多种过滤选项，包括 glob 模式忽略、Git 忽略文件支持和 Gemini 自定义忽略规则，为文件浏览和管理提供了强大而灵活的功能。

## 主要功能

- 目录内容列表显示
- 文件类型区分（文件/目录）
- 文件大小和修改时间信息
- Glob 模式文件过滤
- Git ignore 规则支持
- Gemini ignore 自定义规则
- 安全路径验证
- 智能排序（目录优先，字母排序）

## 接口定义

### `LSToolParams`

**功能**: LS 工具参数接口

```typescript
export interface LSToolParams {
  path: string;                    // 要列出的目录的绝对路径
  ignore?: string[];               // 可选的 glob 模式忽略列表
  file_filtering_options?: {       // 可选的文件过滤选项
    respect_git_ignore?: boolean;    // 是否遵循 .gitignore 规则
    respect_gemini_ignore?: boolean; // 是否遵循 .geminiignore 规则
  };
}
```

**参数详情**:

- `path`: 必须是绝对路径，指向要列出的目录
- `ignore`: 可选的字符串数组，包含要忽略的 glob 模式
- `file_filtering_options`: 可选的过滤配置
  - `respect_git_ignore`: 默认 true，是否应用 .gitignore 规则
  - `respect_gemini_ignore`: 默认 true，是否应用 .geminiignore 规则

### `FileEntry`

**功能**: 文件条目信息接口

```typescript
export interface FileEntry {
  name: string;           // 文件或目录名称
  path: string;           // 文件或目录的绝对路径
  isDirectory: boolean;   // 是否为目录
  size: number;           // 文件大小（字节），目录为 0
  modifiedTime: Date;     // 最后修改时间
}
```

## 核心方法

### 构造函数

#### `constructor(config: Config)`

**功能**: 创建 LSTool 实例

```typescript
constructor(private config: Config) {
  super(
    LSTool.Name,
    'ReadFolder',
    'Lists the names of files and subdirectories directly within a specified directory path. Can optionally ignore entries matching provided glob patterns.',
    Icon.Folder,
    // Schema 定义...
  );
}
```

### 参数验证

#### `validateToolParams(params: LSToolParams): string | null`

**功能**: 验证工具参数的有效性

**验证项目**:

1. **Schema 验证**: 验证参数结构和类型
2. **绝对路径验证**: 确保路径是绝对路径
3. **安全路径验证**: 确保路径在允许的根目录内

```typescript
validateToolParams(params: LSToolParams): string | null {
  const errors = SchemaValidator.validate(this.schema.parameters, params);
  if (errors) {
    return errors;
  }
  if (!path.isAbsolute(params.path)) {
    return `Path must be absolute: ${params.path}`;
  }
  if (!isWithinRoot(params.path, this.config.getTargetDir())) {
    return `Path must be within the root directory (${this.config.getTargetDir()}): ${params.path}`;
  }
  return null;
}
```

### 文件过滤

#### `private shouldIgnore(filename: string, patterns?: string[]): boolean`

**功能**: 检查文件名是否匹配忽略模式

**Glob 模式转换**:

```typescript
private shouldIgnore(filename: string, patterns?: string[]): boolean {
  if (!patterns || patterns.length === 0) {
    return false;
  }
  for (const pattern of patterns) {
    // 将 glob 模式转换为正则表达式
    const regexPattern = pattern
      .replace(/[.+^${}()|[\]\\]/g, '\\$&')  // 转义特殊字符
      .replace(/\*/g, '.*')                   // * 匹配任意字符
      .replace(/\?/g, '.');                   // ? 匹配单个字符
    const regex = new RegExp(`^${regexPattern}$`);
    if (regex.test(filename)) {
      return true;
    }
  }
  return false;
}
```

**支持的 Glob 模式**:

- `*`: 匹配任意数量的字符
- `?`: 匹配单个字符
- `*.txt`: 匹配所有 .txt 文件
- `temp*`: 匹配以 "temp" 开头的文件
- `node_modules`: 精确匹配

### 主要执行方法

#### `async execute(params: LSToolParams, _signal: AbortSignal): Promise<ToolResult>`

**功能**: 执行目录列表操作

**执行流程**:

1. **参数验证**: 验证输入参数的有效性
2. **目录检查**: 验证路径存在且为目录
3. **文件读取**: 读取目录内容
4. **过滤处理**: 应用各种过滤规则
5. **信息收集**: 获取文件详细信息
6. **排序整理**: 对结果进行排序
7. **格式化输出**: 生成用户友好的显示格式

```typescript
async execute(
  params: LSToolParams,
  _signal: AbortSignal,
): Promise<ToolResult> {
  const validationError = this.validateToolParams(params);
  if (validationError) {
    return this.errorResult(
      `Error: Invalid parameters provided. Reason: ${validationError}`,
      `Failed to execute tool.`,
    );
  }

  try {
    const stats = fs.statSync(params.path);
    if (!stats) {
      return this.errorResult(
        `Error: Directory not found or inaccessible: ${params.path}`,
        `Directory not found or inaccessible.`,
      );
    }
    if (!stats.isDirectory()) {
      return this.errorResult(
        `Error: Path is not a directory: ${params.path}`,
        `Path is not a directory.`,
      );
    }

    const files = fs.readdirSync(params.path);

    // 获取文件过滤配置
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

    const fileDiscovery = this.config.getFileService();
    const entries: FileEntry[] = [];
    let gitIgnoredCount = 0;
    let geminiIgnoredCount = 0;

    if (files.length === 0) {
      return {
        llmContent: `Directory ${params.path} is empty.`,
        returnDisplay: `Directory is empty.`,
      };
    }

    for (const file of files) {
      if (this.shouldIgnore(file, params.ignore)) {
        continue;
      }

      const fullPath = path.join(params.path, file);
      const relativePath = path.relative(
        this.config.getTargetDir(),
        fullPath,
      );

      // 检查 Git 忽略规则
      if (
        fileFilteringOptions.respectGitIgnore &&
        fileDiscovery.shouldGitIgnoreFile(relativePath)
      ) {
        gitIgnoredCount++;
        continue;
      }

      // 检查 Gemini 忽略规则
      if (
        fileFilteringOptions.respectGeminiIgnore &&
        fileDiscovery.shouldGeminiIgnoreFile(relativePath)
      ) {
        geminiIgnoredCount++;
        continue;
      }

      try {
        const stats = fs.statSync(fullPath);
        const isDir = stats.isDirectory();
        entries.push({
          name: file,
          path: fullPath,
          isDirectory: isDir,
          size: isDir ? 0 : stats.size,
          modifiedTime: stats.mtime,
        });
      } catch (error) {
        // 内部记录错误但不中断整个列表操作
        console.error(`Error accessing ${fullPath}: ${error}`);
      }
    }

    // 排序条目（目录优先，然后字母排序）
    entries.sort((a, b) => {
      if (a.isDirectory && !b.isDirectory) return -1;
      if (!a.isDirectory && b.isDirectory) return 1;
      return a.name.localeCompare(b.name);
    });

    // 为 LLM 创建格式化内容
    const directoryContent = entries
      .map((entry) => `${entry.isDirectory ? '[DIR] ' : ''}${entry.name}`)
      .join('\n');

    // 添加统计信息
    let summaryInfo = '';
    if (gitIgnoredCount > 0 || geminiIgnoredCount > 0) {
      const ignoredParts = [];
      if (gitIgnoredCount > 0) {
        ignoredParts.push(`${gitIgnoredCount} git-ignored`);
      }
      if (geminiIgnoredCount > 0) {
        ignoredParts.push(`${geminiIgnoredCount} gemini-ignored`);
      }
      summaryInfo = `\n(${ignoredParts.join(', ')} items hidden)`;
    }

    return {
      llmContent: directoryContent + summaryInfo,
      returnDisplay: `Listed ${entries.length} items`,
    };
  } catch (error) {
    console.error(`Error during LS execution: ${error}`);
    return this.errorResult(
      `Error accessing directory: ${error}`,
      `Failed to access directory.`,
    );
  }
}
```

### 辅助方法

#### `getDescription(params: LSToolParams): string`

**功能**: 获取操作描述信息

```typescript
getDescription(params: LSToolParams): string {
  const relativePath = makeRelative(params.path, this.config.getTargetDir());
  return shortenPath(relativePath);
}
```

#### `private errorResult(llmContent: string, returnDisplay: string): ToolResult`

**功能**: 统一的错误结果格式化

```typescript
private errorResult(llmContent: string, returnDisplay: string): ToolResult {
  return {
    llmContent,
    returnDisplay: `Error: ${returnDisplay}`,
  };
}
```

## 使用示例

### 基本目录列表

```typescript
const lsTool = new LSTool(config);

// 列出当前目录
const result = await lsTool.execute({
  path: "/Users/user/project"
});

// 列出特定子目录
const result = await lsTool.execute({
  path: "/Users/user/project/src"
});
```

### 带过滤的目录列表

```typescript
// 忽略特定文件模式
const result = await lsTool.execute({
  path: "/Users/user/project",
  ignore: ["*.log", "temp*", "node_modules"]
});

// 忽略 .DS_Store 和隐藏文件
const result = await lsTool.execute({
  path: "/Users/user/project",
  ignore: [".*", "*.tmp"]
});
```

### 配置忽略规则

```typescript
// 禁用 Git 忽略规则
const result = await lsTool.execute({
  path: "/Users/user/project",
  file_filtering_options: {
    respect_git_ignore: false,
    respect_gemini_ignore: true
  }
});

// 禁用所有自动忽略
const result = await lsTool.execute({
  path: "/Users/user/project",
  file_filtering_options: {
    respect_git_ignore: false,
    respect_gemini_ignore: false
  }
});
```

### 复杂过滤场景

```typescript
// 仅显示源代码文件
const result = await lsTool.execute({
  path: "/Users/user/project/src",
  ignore: ["*.test.*", "*.spec.*", "*.d.ts"]
});

// 忽略构建和缓存目录
const result = await lsTool.execute({
  path: "/Users/user/project",
  ignore: ["dist", "build", ".cache", "coverage"]
});
```

## 输出格式

### 标准输出格式

```
[DIR] components
[DIR] utils
App.tsx
index.ts
package.json
README.md
```

### 带统计信息的输出

```
[DIR] src
[DIR] tests
main.js
package.json
(5 git-ignored, 2 gemini-ignored items hidden)
```

### 空目录输出

```
Directory /path/to/empty/dir is empty.
```

## 文件过滤系统

### Glob 模式支持

#### 基本通配符

```typescript
// 单字符匹配
ignore: ["?.tmp"]      // 匹配 "a.tmp", "1.tmp" 等

// 多字符匹配  
ignore: ["*.log"]      // 匹配所有 .log 文件
ignore: ["temp*"]      // 匹配以 "temp" 开头的文件

// 精确匹配
ignore: ["node_modules"] // 仅匹配 "node_modules"
```

#### 复杂模式组合

```typescript
// 多种文件类型
ignore: ["*.{log,tmp,cache}"]

// 特定前缀和后缀
ignore: ["test_*.js", "*_backup.*"]

// 系统文件
ignore: [".DS_Store", "Thumbs.db", "desktop.ini"]
```

### Git 忽略集成

#### 自动 .gitignore 支持

LSTool 会自动读取并应用 `.gitignore` 文件中的规则：

```gitignore
# .gitignore 示例
node_modules/
*.log
.env
dist/
coverage/
```

这些文件和目录会自动从列表中排除（除非显式禁用）。

#### Git 仓库检测

- 自动检测是否在 Git 仓库中
- 仅在 Git 仓库中应用 .gitignore 规则
- 支持嵌套的 .gitignore 文件

### Gemini 自定义忽略

#### .geminiignore 文件

创建 `.geminiignore` 文件来定义 Gemini 特定的忽略规则：

```gitignore
# .geminiignore 示例
*.ai-generated
temp_*
.gemini-cache/
experimental/
```

#### 优先级和组合

1. 用户指定的 `ignore` 参数（最高优先级）
2. .geminiignore 规则
3. .gitignore 规则（最低优先级）

### 过滤统计

工具会统计被各种规则过滤的文件数量：

```
(3 git-ignored, 1 gemini-ignored items hidden)
```

## 安全性考虑

### 路径安全验证

#### 绝对路径要求

```typescript
// 正确 - 绝对路径
path: "/Users/user/project/src"

// 错误 - 相对路径会被拒绝
path: "../sensitive/dir"
```

#### 根目录限制

```typescript
// 自动验证路径在允许的根目录内
if (!isWithinRoot(params.path, this.config.getTargetDir())) {
  return `Path must be within the root directory`;
}
```

### 访问权限处理

```typescript
// 优雅处理权限错误
try {
  const stats = fs.statSync(fullPath);
  // 处理文件信息...
} catch (error) {
  // 记录错误但继续处理其他文件
  console.error(`Error accessing ${fullPath}: ${error}`);
}
```

### 错误恢复策略

- 单个文件访问失败不会中断整个列表操作
- 详细的错误日志记录
- 用户友好的错误信息

## 性能优化

### 批量文件操作

```typescript
// 使用同步操作提高性能
const files = fs.readdirSync(params.path);
for (const file of files) {
  const stats = fs.statSync(fullPath);
  // 处理每个文件...
}
```

### 智能过滤顺序

1. **用户 glob 过滤**: 最快，基于文件名
2. **Git 忽略检查**: 中等速度，基于路径模式
3. **文件系统访问**: 最慢，获取详细信息

### 内存优化

```typescript
// 流式处理大目录
const entries: FileEntry[] = [];
for (const file of files) {
  // 逐个处理而非全部加载到内存
  if (shouldProcess(file)) {
    entries.push(processFile(file));
  }
}
```

## 高级功能

### 排序策略

#### 默认排序

```typescript
entries.sort((a, b) => {
  if (a.isDirectory && !b.isDirectory) return -1;  // 目录优先
  if (!a.isDirectory && b.isDirectory) return 1;
  return a.name.localeCompare(b.name);             // 字母排序
});
```

#### 自定义排序扩展

```typescript
class CustomLSTool extends LSTool {
  protected sortEntries(entries: FileEntry[]): FileEntry[] {
    return entries.sort((a, b) => {
      // 自定义排序逻辑
      // 例如：按修改时间排序
      return b.modifiedTime.getTime() - a.modifiedTime.getTime();
    });
  }
}
```

### 文件信息增强

#### 详细文件信息

```typescript
interface ExtendedFileEntry extends FileEntry {
  permissions?: string;  // 文件权限
  owner?: string;        // 文件所有者
  group?: string;        // 文件组
  type?: string;         // MIME 类型
}
```

#### 符号链接处理

```typescript
// 检测和处理符号链接
if (stats.isSymbolicLink()) {
  const linkTarget = fs.readlinkSync(fullPath);
  entry.linkTarget = linkTarget;
  entry.isSymlink = true;
}
```

### 缓存和优化

#### 目录缓存

```typescript
class CachedLSTool extends LSTool {
  private cache = new Map<string, { entries: FileEntry[], timestamp: number }>();
  
  async execute(params: LSToolParams, signal: AbortSignal) {
    const cacheKey = params.path;
    const cached = this.cache.get(cacheKey);
    
    if (cached && !this.isCacheExpired(cached.timestamp)) {
      return this.formatCachedResult(cached.entries);
    }
    
    const result = await super.execute(params, signal);
    this.updateCache(cacheKey, result);
    return result;
  }
}
```

#### 异步优化

```typescript
// 并行处理文件信息
const entryPromises = files.map(async (file) => {
  const fullPath = path.join(params.path, file);
  try {
    const stats = await fs.promises.stat(fullPath);
    return this.createFileEntry(file, fullPath, stats);
  } catch (error) {
    console.error(`Error accessing ${fullPath}: ${error}`);
    return null;
  }
});

const entries = (await Promise.all(entryPromises))
  .filter(entry => entry !== null);
```

## 集成特性

### 与其他工具协作

#### ReadFileTool 集成

```typescript
// 列出目录后读取特定文件
const lsResult = await lsTool.execute({ path: "/project/src" });
const readResult = await readFileTool.execute({ 
  file_path: "/project/src/index.ts" 
});
```

#### GrepTool 集成

```typescript
// 在列出的文件中搜索内容
const lsResult = await lsTool.execute({ path: "/project/src" });
const grepResult = await grepTool.execute({
  pattern: "function.*",
  path: "/project/src",
  include: "*.ts"
});
```

### 配置集成

#### 全局文件过滤配置

```typescript
// 在 Config 中设置默认过滤选项
const config = new Config({
  fileFilteringOptions: {
    respectGitIgnore: true,
    respectGeminiIgnore: true
  }
});
```

#### 项目特定配置

```typescript
// 项目级别的 .geminiconfig
{
  "fileFiltering": {
    "defaultIgnorePatterns": ["*.log", "temp*"],
    "respectGitIgnore": true,
    "respectGeminiIgnore": true
  }
}
```

## 错误处理和调试

### 常见错误场景

#### 路径错误

```typescript
// 相对路径错误
Error: Path must be absolute: ./relative/path

// 路径超出根目录
Error: Path must be within the root directory

// 路径不存在
Error: Directory not found or inaccessible
```

#### 权限错误

```typescript
// 访问权限不足
Error: Permission denied accessing directory

// 文件系统错误  
Error: File system error occurred
```

### 调试功能

#### 详细日志模式

```typescript
// 启用调试模式查看详细信息
const result = await lsTool.execute({
  path: "/project",
  debug: true  // 显示过滤详情和性能信息
});
```

#### 过滤统计

```typescript
// 查看过滤统计信息
{
  totalFiles: 20,
  visibleFiles: 15,
  gitIgnored: 3,
  geminiIgnored: 1,
  globIgnored: 1
}
```

## 最佳实践

### 性能最佳实践

```typescript
// 推荐：使用具体路径
path: "/project/src/components"

// 避免：在根目录列出大量文件
path: "/", ignore: []

// 推荐：使用适当的过滤
ignore: ["node_modules", "*.log", ".cache"]
```

### 安全最佳实践

```typescript
// 推荐：始终使用绝对路径
path: path.resolve(process.cwd(), "src")

// 推荐：验证路径安全性
if (!isWithinAllowedDirectory(targetPath)) {
  throw new Error("Path not allowed");
}

// 推荐：处理敏感文件
ignore: [".env*", "*.key", "*.pem", "secrets.*"]
```

### 可维护性建议

```typescript
// 推荐：使用配置文件管理忽略规则
// .geminiignore
node_modules/
*.log
.env*
dist/
coverage/

// 推荐：组织相关的忽略模式
const DEVELOPMENT_IGNORES = ["*.log", ".cache", "temp*"];
const BUILD_IGNORES = ["dist", "build", "coverage"];
const SYSTEM_IGNORES = [".DS_Store", "Thumbs.db"];

ignore: [...DEVELOPMENT_IGNORES, ...BUILD_IGNORES, ...SYSTEM_IGNORES]
```