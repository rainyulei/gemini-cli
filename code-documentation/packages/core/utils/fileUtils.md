# fileUtils 工具函数文档

## 概述

`fileUtils` 模块提供了一组用于文件操作和处理的核心工具函数，主要用于 Gemini CLI 的文件读取、类型检测和内容处理功能。这些函数涵盖了从基本的路径验证到复杂的文件内容分析等各种文件操作需求。

## 主要功能

- 文件类型检测和 MIME 类型识别
- 路径安全性验证
- 二进制文件检测
- 文件内容处理和格式化
- 多媒体文件支持
- 安全的文件系统操作

## 常量定义

### 文本文件处理常量

```typescript
const DEFAULT_MAX_LINES_TEXT_FILE = 2000;  // 默认最大文本行数
const MAX_LINE_LENGTH_TEXT_FILE = 2000;    // 最大行长度
export const DEFAULT_ENCODING: BufferEncoding = 'utf-8'; // 默认编码
```

## 核心函数

### `getSpecificMimeType(filePath: string): string | undefined`

**功能**: 查找文件路径对应的具体 MIME 类型

**参数**:

- `filePath: string` - 文件路径

**返回**: `string | undefined` - 具体的 MIME 类型字符串或 undefined

**实现**:

```typescript
export function getSpecificMimeType(filePath: string): string | undefined {
  const lookedUpMime = mime.lookup(filePath);
  return typeof lookedUpMime === 'string' ? lookedUpMime : undefined;
}
```

**使用示例**:

```typescript
const mimeType = getSpecificMimeType('/path/to/file.py');
// 返回: 'text/x-python'

const jsType = getSpecificMimeType('/path/to/script.js');
// 返回: 'application/javascript'
```

### `isWithinRoot(pathToCheck: string, rootDirectory: string): boolean`

**功能**: 检查指定路径是否在给定根目录内（安全性验证）

**参数**:

- `pathToCheck: string` - 要检查的绝对路径
- `rootDirectory: string` - 根目录的绝对路径

**返回**: `boolean` - 如果路径在根目录内返回 true，否则返回 false

**安全特性**:

1. **路径规范化**: 使用 `path.resolve()` 规范化路径
2. **分隔符处理**: 正确处理不同操作系统的路径分隔符
3. **边界情况**: 处理根路径等特殊情况

**实现细节**:

```typescript
export function isWithinRoot(pathToCheck: string, rootDirectory: string): boolean {
  const normalizedPathToCheck = path.resolve(pathToCheck);
  const normalizedRootDirectory = path.resolve(rootDirectory);

  // 确保根目录路径以分隔符结尾，用于正确的 startsWith 比较
  const rootWithSeparator =
    normalizedRootDirectory === path.sep ||
    normalizedRootDirectory.endsWith(path.sep)
      ? normalizedRootDirectory
      : normalizedRootDirectory + path.sep;

  return (
    normalizedPathToCheck === normalizedRootDirectory ||
    normalizedPathToCheck.startsWith(rootWithSeparator)
  );
}
```

**使用示例**:

```typescript
// 正常情况
isWithinRoot('/home/user/project/file.txt', '/home/user/project');
// 返回: true

// 路径遍历攻击防护
isWithinRoot('/home/user/project/../../../etc/passwd', '/home/user/project');
// 返回: false

// 相同路径
isWithinRoot('/home/user/project', '/home/user/project');
// 返回: true
```

### `async isBinaryFile(filePath: string): Promise<boolean>`

**功能**: 通过内容采样判断文件是否为二进制文件

**参数**:

- `filePath: string` - 文件路径

**返回**: `Promise<boolean>` - 如果文件为二进制文件返回 true

**检测算法**:

1. **文件大小检查**: 空文件不被视为二进制文件
2. **采样读取**: 读取最多 4KB 的文件开头内容
3. **空字节检测**: 存在空字节（0x00）强烈表明是二进制文件
4. **非打印字符统计**: 超过 30% 的非打印字符视为二进制文件

**实现细节**:

```typescript
export async function isBinaryFile(filePath: string): Promise<boolean> {
  let fileHandle: fs.promises.FileHandle | undefined;
  try {
    fileHandle = await fs.promises.open(filePath, 'r');
    
    const stats = await fileHandle.stat();
    const fileSize = stats.size;
    if (fileSize === 0) {
      return false; // 空文件不被视为二进制
    }
    
    // 读取最多 4KB 或文件大小（取较小值）
    const bufferSize = Math.min(4096, fileSize);
    const buffer = Buffer.alloc(bufferSize);
    const result = await fileHandle.read(buffer, 0, buffer.length, 0);
    const bytesRead = result.bytesRead;

    if (bytesRead === 0) return false;

    let nonPrintableCount = 0;
    for (let i = 0; i < bytesRead; i++) {
      if (buffer[i] === 0) return true; // 空字节是强指标
      if (buffer[i] < 9 || (buffer[i] > 13 && buffer[i] < 32)) {
        nonPrintableCount++;
      }
    }
    
    // 如果超过 30% 的非打印字符，视为二进制
    return nonPrintableCount / bytesRead > 0.3;
  } catch (error) {
    console.warn(`Failed to check if file is binary: ${filePath}`, error);
    return false; // 出错时默认为非二进制
  } finally {
    if (fileHandle) {
      await fileHandle.close();
    }
  }
}
```

**使用示例**:

```typescript
// 检查各种文件类型
const isTextBinary = await isBinaryFile('/path/to/file.txt');
// 返回: false

const isImageBinary = await isBinaryFile('/path/to/image.png');
// 返回: true

const isExecutableBinary = await isBinaryFile('/path/to/program.exe');
// 返回: true
```

### `async processSingleFileContent(...): Promise<FileContentResult>`

**功能**: 处理单个文件的内容，支持多种文件类型

**参数**:

- `filePath: string` - 文件的绝对路径
- `rootDirectory: string` - 项目根目录
- `offset?: number` - 起始行号（可选）
- `limit?: number` - 读取行数限制（可选）

**返回**: `Promise<FileContentResult>` - 文件内容处理结果

**支持的文件类型**:

1. **文本文件**: 直接读取和格式化
2. **图像文件**: PNG, JPG, GIF, WEBP, SVG, BMP
3. **PDF 文件**: 文本提取和结构分析
4. **二进制文件**: 提供文件类型信息

**处理流程**:

1. **安全性检查**: 验证文件路径在根目录内
2. **存在性检查**: 确认文件存在且可访问
3. **类型检测**: 判断文件类型并选择处理方式
4. **内容处理**: 根据类型执行相应的处理逻辑
5. **格式化输出**: 生成 LLM 友好的输出格式

### `formatFileContentForDisplay(...): string`

**功能**: 格式化文件内容用于显示

**特性**:

- **行号添加**: 为文本文件添加行号
- **长度限制**: 截断过长的行
- **编码处理**: 正确处理各种字符编码
- **格式保持**: 保持原始格式和缩进

## 高级功能

### 多媒体文件处理

**图像文件处理**:

```typescript
// 自动检测图像类型并生成适当的描述
const imageResult = await processSingleFileContent('/path/to/image.png', rootDir);
// 结果包含图像元数据和 GenAI 兼容的部分
```

**PDF 文件处理**:

```typescript
// 提取 PDF 文本内容和结构信息
const pdfResult = await processSingleFileContent('/path/to/document.pdf', rootDir);
// 结果包含提取的文本和页面信息
```

### 分页读取

**大文件分页处理**:

```typescript
// 读取从第 100 行开始的 50 行内容
const partialResult = await processSingleFileContent(
  '/path/to/large-file.txt',
  rootDir,
  100,  // offset
  50    // limit
);
```

### 错误处理

**全面的错误处理机制**:

1. **文件不存在**: 提供清晰的错误消息
2. **权限错误**: 处理访问权限问题
3. **编码错误**: 处理字符编码问题
4. **内存限制**: 防止大文件导致的内存问题

## 安全特性

### 路径遍历防护

```typescript
// 防护措施
if (!isWithinRoot(filePath, rootDirectory)) {
  throw new Error('File path is outside the allowed directory');
}
```

### 资源管理

- **文件句柄管理**: 确保文件句柄正确关闭
- **内存控制**: 限制读取的数据量
- **超时处理**: 防止长时间阻塞操作

## 性能优化

### 文件读取优化

1. **流式读取**: 对大文件使用流式处理
2. **缓冲控制**: 合理设置读取缓冲区大小
3. **延迟加载**: 仅在需要时加载文件内容

### 类型检测优化

1. **扩展名优先**: 首先基于扩展名判断类型
2. **内容采样**: 仅读取文件开头进行类型检测
3. **缓存结果**: 缓存文件类型检测结果

## 使用示例

### 基本文件处理

```typescript
import { processSingleFileContent, isWithinRoot, isBinaryFile } from './fileUtils';

// 安全性检查
if (!isWithinRoot(filePath, projectRoot)) {
  throw new Error('File outside project directory');
}

// 二进制文件检查
if (await isBinaryFile(filePath)) {
  console.log('Binary file detected');
}

// 处理文件内容
const result = await processSingleFileContent(filePath, projectRoot);
console.log(result.llmContent);
```

### 批量文件处理

```typescript
const files = ['/path/to/file1.txt', '/path/to/file2.py'];
const results = await Promise.all(
  files.map(file => processSingleFileContent(file, projectRoot))
);
```

### 错误处理示例

```typescript
try {
  const result = await processSingleFileContent(filePath, rootDir);
  if (result.error) {
    console.error('File processing error:', result.error);
  } else {
    console.log('Success:', result.llmContent);
  }
} catch (error) {
  console.error('Unexpected error:', error);
}
```

## 依赖关系

### 外部依赖

- `node:fs` - Node.js 文件系统 API
- `node:path` - Node.js 路径处理
- `@google/genai` - Google GenAI SDK 类型
- `mime-types` - MIME 类型查找

### 内部依赖

- 其他工具函数模块
- 配置管理系统
- 错误处理机制

## 测试覆盖

### 单元测试

- 路径验证测试
- 文件类型检测测试
- 内容处理测试
- 错误处理测试

### 集成测试

- 多种文件类型处理
- 大文件处理性能
- 安全性验证

## 扩展性

### 新文件类型支持

1. 在类型检测逻辑中添加新的 MIME 类型
2. 实现对应的内容处理函数
3. 添加相应的测试用例

### 性能改进

1. 实现更高效的二进制检测算法
2. 添加文件内容缓存机制
3. 优化大文件处理策略