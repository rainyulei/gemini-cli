# GrepTool 文件内容搜索工具文档

## 概述

`GrepTool` 是 Gemini CLI 的文件内容搜索工具，使用正则表达式在指定目录下的文件中搜索匹配模式。它支持多种搜索策略，包括系统原生 grep、git grep 和纯 JavaScript 实现的回退方案，提供强大且灵活的文本搜索功能。

## 主要功能

- 正则表达式模式匹配
- 递归目录搜索
- 文件模式过滤（glob 支持）
- 多种搜索引擎（git grep, 系统 grep, JavaScript 回退）
- 安全路径验证
- 搜索结果按文件分组显示
- 行号和内容显示

## 接口定义

### `GrepToolParams`

**功能**: Grep 工具参数接口

```typescript
export interface GrepToolParams {
  pattern: string;  // 要搜索的正则表达式模式
  path?: string;    // 搜索目录路径（可选，默认当前目录）
  include?: string; // 文件过滤模式（可选，如 "*.js", "*.{ts,tsx}"）
}
```

**参数详情**:

- `pattern`: 正则表达式字符串，用于匹配文件内容
- `path`: 可选的搜索路径，支持绝对路径和相对路径
- `include`: 可选的 glob 模式，用于过滤要搜索的文件类型

### `GrepMatch`

**功能**: 单个搜索匹配结果

```typescript
interface GrepMatch {
  filePath: string;    // 匹配文件的路径
  lineNumber: number;  // 匹配行的行号
  line: string;        // 匹配行的完整内容
}
```

## 核心方法

### 构造函数

#### `constructor(config: Config)`

**功能**: 创建 GrepTool 实例

```typescript
constructor(private readonly config: Config) {
  super(
    GrepTool.Name,
    'SearchText',
    'Searches for a regular expression pattern within the content of files in a specified directory (or current working directory). Can filter files by a glob pattern. Returns the lines containing matches, along with their file paths and line numbers.',
    Icon.Regex,
    // Schema 定义...
  );
}
```

### 参数验证

#### `validateToolParams(params: GrepToolParams): string | null`

**功能**: 验证工具参数的有效性

**验证项目**:

1. **Schema 验证**: 验证参数结构和类型
2. **正则表达式验证**: 确保 pattern 是有效的正则表达式
3. **路径验证**: 验证搜索路径的安全性和可访问性

```typescript
validateToolParams(params: GrepToolParams): string | null {
  const errors = SchemaValidator.validate(this.schema.parameters, params);
  if (errors) {
    return errors;
  }

  try {
    new RegExp(params.pattern);
  } catch (error) {
    return `Invalid regular expression pattern provided: ${params.pattern}. Error: ${getErrorMessage(error)}`;
  }

  try {
    this.resolveAndValidatePath(params.path);
  } catch (error) {
    return getErrorMessage(error);
  }

  return null; // 参数有效
}
```

### 路径安全验证

#### `private resolveAndValidatePath(relativePath?: string): string`

**功能**: 安全地解析和验证搜索路径

**安全检查**:

1. **路径遍历防护**: 确保路径在允许的根目录内
2. **存在性检查**: 验证路径是否存在
3. **类型检查**: 确保路径是目录而非文件

```typescript
private resolveAndValidatePath(relativePath?: string): string {
  const targetPath = path.resolve(
    this.config.getTargetDir(),
    relativePath || '.',
  );

  // 安全检查：确保解析的路径仍在根目录内
  if (
    !targetPath.startsWith(this.config.getTargetDir()) &&
    targetPath !== this.config.getTargetDir()
  ) {
    throw new Error(
      `Path validation failed: Attempted path "${relativePath || '.'}" resolves outside the allowed root directory "${this.config.getTargetDir()}".`,
    );
  }

  // 检查存在性和类型
  try {
    const stats = fs.statSync(targetPath);
    if (!stats.isDirectory()) {
      throw new Error(`Path is not a directory: ${targetPath}`);
    }
  } catch (error: unknown) {
    if (isNodeError(error) && error.code !== 'ENOENT') {
      throw new Error(`Path does not exist: ${targetPath}`);
    }
    throw new Error(
      `Failed to access path stats for ${targetPath}: ${error}`,
    );
  }

  return targetPath;
}
```

### 主要执行方法

#### `async execute(params: GrepToolParams, signal: AbortSignal): Promise<ToolResult>`

**功能**: 执行文件内容搜索

**执行流程**:

1. **参数验证**: 验证输入参数的有效性
2. **路径解析**: 安全地解析搜索目录
3. **搜索执行**: 使用最优搜索策略执行搜索
4. **结果处理**: 格式化和分组搜索结果
5. **输出生成**: 生成用户友好的结果显示

```typescript
async execute(
  params: GrepToolParams,
  signal: AbortSignal,
): Promise<ToolResult> {
  const validationError = this.validateToolParams(params);
  if (validationError) {
    return {
      llmContent: `Error: Invalid parameters provided. Reason: ${validationError}`,
      returnDisplay: `Model provided invalid parameters. Error: ${validationError}`,
    };
  }

  let searchDirAbs: string;
  try {
    searchDirAbs = this.resolveAndValidatePath(params.path);
    const searchDirDisplay = params.path || '.';

    const matches: GrepMatch[] = await this.performGrepSearch({
      pattern: params.pattern,
      path: searchDirAbs,
      include: params.include,
      signal,
    });

    if (matches.length === 0) {
      const noMatchMsg = `No matches found for pattern "${params.pattern}" in path "${searchDirDisplay}"${params.include ? ` (filter: "${params.include}")` : ''}.`;
      return { llmContent: noMatchMsg, returnDisplay: `No matches found` };
    }

    // 按文件分组结果
    const matchesByFile = matches.reduce(
      (acc, match) => {
        const relativeFilePath =
          path.relative(searchDirAbs, path.resolve(searchDirAbs, match.filePath)) || 
          path.basename(match.filePath);
        if (!acc[relativeFilePath]) {
          acc[relativeFilePath] = [];
        }
        acc[relativeFilePath].push(match);
        acc[relativeFilePath].sort((a, b) => a.lineNumber - b.lineNumber);
        return acc;
      },
      {} as Record<string, GrepMatch[]>,
    );

    const matchCount = matches.length;
    const matchTerm = matchCount === 1 ? 'match' : 'matches';

    let llmContent = `Found ${matchCount} ${matchTerm} for pattern "${params.pattern}" in path "${searchDirDisplay}"${params.include ? ` (filter: "${params.include}")` : ''}:\n---\n`;

    for (const filePath in matchesByFile) {
      llmContent += `File: ${filePath}\n`;
      matchesByFile[filePath].forEach((match) => {
        const trimmedLine = match.line.trim();
        llmContent += `L${match.lineNumber}: ${trimmedLine}\n`;
      });
      llmContent += '---\n';
    }

    return {
      llmContent: llmContent.trim(),
      returnDisplay: `Found ${matchCount} ${matchTerm}`,
    };
  } catch (error) {
    console.error(`Error during GrepLogic execution: ${error}`);
    const errorMessage = getErrorMessage(error);
    return {
      llmContent: `Error during grep search operation: ${errorMessage}`,
      returnDisplay: `Error: ${errorMessage}`,
    };
  }
}
```

## 搜索引擎策略

### 多层搜索策略

GrepTool 使用分层的搜索策略以获得最佳性能：

#### 1. Git Grep（首选）

**适用条件**: 在 Git 仓库内且 git 命令可用

**优势**:
- 最快的搜索性能
- 自动忽略 `.gitignore` 文件
- 原生支持正则表达式

#### 2. 系统 Grep（备选）

**适用条件**: 系统安装了 grep 命令

**优势**:
- 高性能的原生实现
- 支持复杂的正则表达式
- 广泛的平台支持

#### 3. JavaScript 回退（保底）

**适用条件**: 前两种方法不可用时

**优势**:
- 纯 JavaScript 实现，不依赖外部工具
- 跨平台兼容性
- 完全可控的搜索逻辑

### 命令检测

#### `private isCommandAvailable(command: string): Promise<boolean>`

**功能**: 检查系统命令是否可用

```typescript
private isCommandAvailable(command: string): Promise<boolean> {
  return new Promise((resolve) => {
    const checkCommand = process.platform === 'win32' ? 'where' : 'command';
    const checkArgs = process.platform === 'win32' ? [command] : ['-v', command];
    try {
      const child = spawn(checkCommand, checkArgs, {
        stdio: 'ignore',
        shell: process.platform === 'win32',
      });
      child.on('close', (code) => resolve(code === 0));
      child.on('error', () => resolve(false));
    } catch {
      resolve(false);
    }
  });
}
```

### 输出解析

#### `private parseGrepOutput(output: string, basePath: string): GrepMatch[]`

**功能**: 解析 grep 命令的标准输出

**解析格式**: `filePath:lineNumber:lineContent`

```typescript
private parseGrepOutput(output: string, basePath: string): GrepMatch[] {
  const results: GrepMatch[] = [];
  if (!output) return results;

  const lines = output.split(EOL);

  for (const line of lines) {
    if (!line.trim()) continue;

    // 找到第一个冒号
    const firstColonIndex = line.indexOf(':');
    if (firstColonIndex === -1) continue;

    // 找到第二个冒号
    const secondColonIndex = line.indexOf(':', firstColonIndex + 1);
    if (secondColonIndex === -1) continue;

    // 提取各部分
    const filePathRaw = line.substring(0, firstColonIndex);
    const lineNumberStr = line.substring(firstColonIndex + 1, secondColonIndex);
    const lineContent = line.substring(secondColonIndex + 1);

    const lineNumber = parseInt(lineNumberStr, 10);

    if (!isNaN(lineNumber)) {
      const absoluteFilePath = path.resolve(basePath, filePathRaw);
      results.push({
        filePath: absoluteFilePath,
        lineNumber,
        line: lineContent,
      });
    }
  }

  return results;
}
```

## 使用示例

### 基本文本搜索

```typescript
const grepTool = new GrepTool(config);

// 搜索函数定义
const result = await grepTool.execute({
  pattern: "function\\s+\\w+",
  path: "src"
});

// 搜索导入语句
const result = await grepTool.execute({
  pattern: "import\\s+.*from\\s+['\"].*['\"]",
  include: "*.{js,ts,tsx}"
});
```

### 具体文件类型搜索

```typescript
// 仅搜索 TypeScript 文件
const result = await grepTool.execute({
  pattern: "interface\\s+\\w+",
  path: "src",
  include: "*.ts"
});

// 搜索多种文件类型
const result = await grepTool.execute({
  pattern: "TODO|FIXME",
  include: "*.{js,ts,tsx,jsx,md}"
});
```

### 复杂正则表达式搜索

```typescript
// 搜索类定义
const result = await grepTool.execute({
  pattern: "class\\s+\\w+\\s*(?:extends\\s+\\w+)?\\s*\\{"
});

// 搜索异步函数
const result = await grepTool.execute({
  pattern: "async\\s+function\\s+\\w+|\\w+\\s*:\\s*async\\s*\\("
});

// 搜索错误处理
const result = await grepTool.execute({
  pattern: "catch\\s*\\(\\s*\\w+\\s*\\)|throw\\s+new\\s+\\w+"
});
```

### 特定目录搜索

```typescript
// 搜索测试文件
const result = await grepTool.execute({
  pattern: "describe\\s*\\(|it\\s*\\(|test\\s*\\(",
  path: "tests",
  include: "*.test.{js,ts}"
});

// 搜索配置文件
const result = await grepTool.execute({
  pattern: "export\\s+default|module\\.exports",
  path: "config",
  include: "*.{js,json}"
});
```

## 高级功能

### 安全性考虑

#### 路径遍历防护

```typescript
// 自动阻止危险路径
const result = await grepTool.execute({
  pattern: "secret",
  path: "../../../etc/passwd" // 会被拒绝
});
```

#### 权限控制

- 仅允许在配置的根目录内搜索
- 自动处理符号链接和路径规范化
- 防止访问系统敏感目录

### 性能优化

#### 搜索策略选择

```typescript
// 自动选择最优搜索引擎
// 1. Git 仓库内优先使用 git grep
// 2. 系统有 grep 时使用系统 grep
// 3. 最后回退到 JavaScript 实现
```

#### 文件过滤优化

```typescript
// 使用 glob 模式预过滤文件
const result = await grepTool.execute({
  pattern: "console\\.log",
  include: "src/**/*.{js,ts}" // 仅搜索源代码目录
});
```

### 结果格式化

#### 按文件分组显示

搜索结果自动按文件分组，每个文件内的匹配按行号排序：

```
Found 5 matches for pattern "function\s+\w+" in path "src":
---
File: utils/helpers.js
L15: function formatDate(date) {
L28: function parseConfig(config) {
---
File: components/Button.tsx
L10: function Button({ children, onClick }) {
L45: function handleClick(event) {
---
```

### 错误处理

#### 常见错误场景

```typescript
// 无效正则表达式
const result = await grepTool.execute({
  pattern: "[invalid regex"  // 会返回错误信息
});

// 路径不存在
const result = await grepTool.execute({
  pattern: "text",
  path: "nonexistent/path"  // 会返回错误信息
});

// 权限不足
const result = await grepTool.execute({
  pattern: "secret",
  path: "/root"  // 会被安全检查阻止
});
```

#### 错误恢复策略

- 参数验证失败时提供详细错误信息
- 搜索引擎不可用时自动回退
- 文件访问失败时继续搜索其他文件

### 扩展功能

#### 自定义搜索引擎

```typescript
// 可以扩展支持其他搜索工具
class ExtendedGrepTool extends GrepTool {
  async performGrepSearch(params) {
    // 尝试使用 ripgrep (rg) 如果可用
    if (await this.isCommandAvailable('rg')) {
      return this.performRipgrepSearch(params);
    }
    return super.performGrepSearch(params);
  }
}
```

#### 结果后处理

```typescript
// 可以添加结果过滤和高亮功能
class HighlightGrepTool extends GrepTool {
  async execute(params, signal) {
    const result = await super.execute(params, signal);
    
    // 添加语法高亮
    result.llmContent = this.addSyntaxHighlight(result.llmContent, params.pattern);
    
    return result;
  }
}
```

## 配置和定制

### 环境配置

- **搜索引擎优先级**: 可通过配置调整搜索引擎选择策略
- **默认包含模式**: 设置默认的文件过滤规则
- **搜索超时**: 配置搜索操作的超时时间

### 集成特性

#### 与其他工具协作

- **ReadFileTool**: 搜索到文件后读取具体内容
- **EditTool**: 基于搜索结果进行批量编辑
- **ShellTool**: 使用系统命令进行复杂搜索

#### Git 集成

- 自动检测 Git 仓库
- 遵循 `.gitignore` 规则
- 支持 Git 仓库子目录搜索

### 监控和调试

- 搜索性能统计
- 搜索引擎使用情况
- 错误率和成功率监控
- 调试模式详细日志

## 最佳实践

### 正则表达式建议

```typescript
// 推荐：使用具体的模式
pattern: "function\\s+\\w+"

// 避免：过于宽泛的模式
pattern: ".*"

// 推荐：转义特殊字符
pattern: "console\\.log"

// 推荐：使用词边界
pattern: "\\bclass\\b"
```

### 性能优化建议

```typescript
// 推荐：使用文件过滤
include: "*.{js,ts,tsx}"

// 推荐：指定具体路径
path: "src/components"

// 避免：在根目录搜索所有文件
path: ".", include: undefined
```

### 安全性建议

- 始终验证用户输入的路径
- 使用相对路径而非绝对路径
- 定期审查搜索模式和结果
- 避免搜索包含敏感信息的目录