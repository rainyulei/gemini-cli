# EditTool 文件编辑工具文档

## 概述

`EditTool` 是 Gemini CLI 的核心文件编辑工具，负责对文件进行精确的文本替换操作。它支持单个或多个替换、新文件创建、diff 预览、用户确认等功能，同时提供了强大的错误检测和修正机制。

## 主要功能

- 精确文本替换（单个/多个匹配）
- 新文件创建
- 编辑预览和用户确认
- 自动编辑修正和验证
- 路径安全性检查
- 差异可视化显示
- 用户修改跟踪

## 接口定义

### `EditToolParams`

**功能**: 编辑工具参数接口

```typescript
export interface EditToolParams {
  file_path: string;              // 要修改的文件绝对路径
  old_string: string;             // 要替换的文本
  new_string: string;             // 替换后的文本
  expected_replacements?: number; // 期望的替换次数（默认1）
  modified_by_user?: boolean;     // 是否被用户手动修改
}
```

**参数要求**:

1. **`file_path`**: 必须是绝对路径，以 `/` 开头
2. **`old_string`**: 必须是准确的字面文本，包括所有空格、缩进、换行符
3. **`new_string`**: 替换文本，确保结果代码正确且符合规范
4. **`expected_replacements`**: 指定期望替换的次数，用于多重替换

### `CalculatedEdit`

**功能**: 计算编辑结果的内部接口

```typescript
interface CalculatedEdit {
  currentContent: string | null;   // 当前文件内容
  newContent: string;             // 编辑后内容
  occurrences: number;            // 匹配次数
  error?: {                       // 错误信息
    display: string;              // 用户显示信息
    raw: string;                  // 原始错误信息
  };
  isNewFile: boolean;             // 是否为新文件
}
```

## 核心方法

### 构造函数

#### `constructor(config: Config)`

**功能**: 创建 EditTool 实例

**参数**:
- `config: Config` - 应用配置对象

**工具定义**:

```typescript
constructor(private readonly config: Config) {
  super(
    EditTool.Name,
    'Edit',
    `替换文件中的文本。默认替换单个匹配，可通过 expected_replacements 指定多重替换。
    此工具需要提供足够的上下文来确保精确定位。在尝试替换前务必使用 ${ReadFileTool.Name} 工具检查文件内容。`,
    Icon.Pencil,
    // Schema 定义...
  );
}
```

### 参数验证

#### `validateToolParams(params: EditToolParams): string | null`

**功能**: 验证工具参数的有效性

**验证项目**:

1. **Schema 验证**: 使用 SchemaValidator 验证参数结构
2. **路径验证**: 确保是绝对路径
3. **安全验证**: 确保路径在允许的根目录内

```typescript
validateToolParams(params: EditToolParams): string | null {
  const errors = SchemaValidator.validate(this.schema.parameters, params);
  if (errors) return errors;
  
  if (!path.isAbsolute(params.file_path)) {
    return `File path must be absolute: ${params.file_path}`;
  }
  
  if (!isWithinRoot(params.file_path, this.config.getTargetDir())) {
    return `File path must be within the root directory: ${params.file_path}`;
  }
  
  return null;
}
```

### 编辑计算

#### `private async calculateEdit(params: EditToolParams, abortSignal: AbortSignal): Promise<CalculatedEdit>`

**功能**: 计算编辑操作的结果，不实际执行

**执行流程**:

1. **文件存在性检查**:
   ```typescript
   try {
     currentContent = fs.readFileSync(params.file_path, 'utf8');
     currentContent = currentContent.replace(/\r\n/g, '\n'); // 规范化行结尾
     fileExists = true;
   } catch (err) {
     if (err.code !== 'ENOENT') throw err;
     fileExists = false;
   }
   ```

2. **新文件创建逻辑**:
   ```typescript
   if (params.old_string === '' && !fileExists) {
     isNewFile = true;
   } else if (!fileExists) {
     error = {
       display: 'File not found. Use empty old_string to create new file.',
       raw: `File not found: ${params.file_path}`
     };
   }
   ```

3. **编辑修正和验证**:
   ```typescript
   const correctedEdit = await ensureCorrectEdit(
     params.file_path,
     currentContent,
     params,
     this.config.getGeminiClient(),
     abortSignal,
   );
   finalOldString = correctedEdit.params.old_string;
   finalNewString = correctedEdit.params.new_string;
   occurrences = correctedEdit.occurrences;
   ```

4. **错误检查**:
   - 0 个匹配: "无法找到要替换的字符串"
   - 匹配数不符: "期望 X 个匹配但找到 Y 个"
   - 相同内容: "old_string 和 new_string 相同"

### 文本替换

#### `private _applyReplacement(currentContent: string | null, oldString: string, newString: string, isNewFile: boolean): string`

**功能**: 执行实际的文本替换操作

**替换逻辑**:

```typescript
private _applyReplacement(
  currentContent: string | null,
  oldString: string,
  newString: string,
  isNewFile: boolean,
): string {
  if (isNewFile) {
    return newString;  // 新文件直接返回新内容
  }
  
  if (currentContent === null) {
    return oldString === '' ? newString : '';
  }
  
  // 空字符串且非新文件时不修改
  if (oldString === '' && !isNewFile) {
    return currentContent;
  }
  
  return currentContent.replaceAll(oldString, newString);
}
```

### 用户确认

#### `async shouldConfirmExecute(params: EditToolParams, abortSignal: AbortSignal): Promise<ToolCallConfirmationDetails | false>`

**功能**: 生成用户确认提示，显示编辑预览

**确认条件**:

```typescript
// 自动编辑模式跳过确认
if (this.config.getApprovalMode() === ApprovalMode.AUTO_EDIT) {
  return false;
}
```

**差异生成**:

```typescript
const fileDiff = Diff.createPatch(
  fileName,
  editData.currentContent ?? '',
  editData.newContent,
  'Current',
  'Proposed',
  DEFAULT_DIFF_OPTIONS,
);
```

**确认详情**:

```typescript
const confirmationDetails: ToolEditConfirmationDetails = {
  type: 'edit',
  title: `Confirm Edit: ${shortenPath(makeRelative(params.file_path, this.config.getTargetDir()))}`,
  fileName,
  fileDiff,
  originalContent: editData.currentContent,
  newContent: editData.newContent,
  onConfirm: async (outcome: ToolConfirmationOutcome) => {
    if (outcome === ToolConfirmationOutcome.ProceedAlways) {
      this.config.setApprovalMode(ApprovalMode.AUTO_EDIT);
    }
  },
};
```

### 工具执行

#### `async execute(params: EditToolParams, signal: AbortSignal): Promise<ToolResult>`

**功能**: 执行编辑操作并返回结果

**执行步骤**:

1. **参数验证**:
   ```typescript
   const validationError = this.validateToolParams(params);
   if (validationError) {
     return {
       llmContent: `Error: Invalid parameters. Reason: ${validationError}`,
       returnDisplay: `Error: ${validationError}`,
     };
   }
   ```

2. **编辑计算**:
   ```typescript
   let editData: CalculatedEdit;
   try {
     editData = await this.calculateEdit(params, signal);
   } catch (error) {
     return {
       llmContent: `Error preparing edit: ${error.message}`,
       returnDisplay: `Error preparing edit: ${error.message}`,
     };
   }
   ```

3. **错误检查**:
   ```typescript
   if (editData.error) {
     return {
       llmContent: editData.error.raw,
       returnDisplay: `Error: ${editData.error.display}`,
     };
   }
   ```

4. **文件写入**:
   ```typescript
   try {
     this.ensureParentDirectoriesExist(params.file_path);
     fs.writeFileSync(params.file_path, editData.newContent, 'utf8');
   } catch (error) {
     return {
       llmContent: `Error executing edit: ${error.message}`,
       returnDisplay: `Error writing file: ${error.message}`,
     };
   }
   ```

5. **结果生成**:
   ```typescript
   // 新文件
   if (editData.isNewFile) {
     displayResult = `Created ${shortenPath(makeRelative(params.file_path, this.config.getTargetDir()))}`;
   } else {
     // 生成差异显示
     const fileDiff = Diff.createPatch(/* ... */);
     displayResult = { fileDiff, fileName, originalContent, newContent };
   }
   ```

### 辅助方法

#### `ensureParentDirectoriesExist(filePath: string): void`

**功能**: 确保父目录存在

```typescript
private ensureParentDirectoriesExist(filePath: string): void {
  const dirName = path.dirname(filePath);
  if (!fs.existsSync(dirName)) {
    fs.mkdirSync(dirName, { recursive: true });
  }
}
```

#### `toolLocations(params: EditToolParams): ToolLocation[]`

**功能**: 返回工具影响的文件位置

```typescript
toolLocations(params: EditToolParams): ToolLocation[] {
  return [{ path: params.file_path }];
}
```

#### `getDescription(params: EditToolParams): string`

**功能**: 生成操作描述

```typescript
getDescription(params: EditToolParams): string {
  const relativePath = makeRelative(params.file_path, this.config.getTargetDir());
  
  if (params.old_string === '') {
    return `Create ${shortenPath(relativePath)}`;
  }
  
  const oldSnippet = params.old_string.split('\n')[0].substring(0, 30) + '...';
  const newSnippet = params.new_string.split('\n')[0].substring(0, 30) + '...';
  
  return `${shortenPath(relativePath)}: ${oldSnippet} => ${newSnippet}`;
}
```

## ModifiableTool 实现

### `getModifyContext(abortSignal: AbortSignal): ModifyContext<EditToolParams>`

**功能**: 提供用户修改工具参数的上下文

```typescript
getModifyContext(_: AbortSignal): ModifyContext<EditToolParams> {
  return {
    getFilePath: (params) => params.file_path,
    
    getCurrentContent: async (params) => {
      try {
        return fs.readFileSync(params.file_path, 'utf8');
      } catch (err) {
        if (err.code !== 'ENOENT') throw err;
        return '';
      }
    },
    
    getProposedContent: async (params) => {
      try {
        const currentContent = fs.readFileSync(params.file_path, 'utf8');
        return this._applyReplacement(
          currentContent,
          params.old_string,
          params.new_string,
          params.old_string === '' && currentContent === ''
        );
      } catch (err) {
        if (err.code !== 'ENOENT') throw err;
        return '';
      }
    },
    
    createUpdatedParams: (oldContent, modifiedContent, originalParams) => ({
      ...originalParams,
      old_string: oldContent,
      new_string: modifiedContent,
      modified_by_user: true,
    }),
  };
}
```

## 使用示例

### 基本文件编辑

```typescript
const editTool = new EditTool(config);

// 替换单行代码
const result = await editTool.execute({
  file_path: '/project/src/index.js',
  old_string: 'console.log("Hello");',
  new_string: 'console.log("Hello, World!");',
});
```

### 多行文本替换

```typescript
// 替换函数实现
const result = await editTool.execute({
  file_path: '/project/src/utils.js',
  old_string: `function oldFunction(param) {
  return param + 1;
}`,
  new_string: `function newFunction(param) {
  const result = param + 1;
  return result;
}`,
});
```

### 多重替换

```typescript
// 替换所有匹配项
const result = await editTool.execute({
  file_path: '/project/src/config.js',
  old_string: 'var ',
  new_string: 'const ',
  expected_replacements: 5,  // 期望替换5个匹配项
});
```

### 创建新文件

```typescript
// 创建新文件
const result = await editTool.execute({
  file_path: '/project/src/newFile.js',
  old_string: '',  // 空字符串表示新文件
  new_string: `// 新文件内容
export default function newFunction() {
  return "Hello, World!";
}`,
});
```

### 带确认的编辑

```typescript
// 检查是否需要用户确认
const confirmationDetails = await editTool.shouldConfirmExecute(params, abortSignal);

if (confirmationDetails) {
  // 显示确认对话框
  const userApproved = await showConfirmationDialog(confirmationDetails);
  
  if (userApproved) {
    const result = await editTool.execute(params, abortSignal);
  }
}
```

## 错误处理

### 常见错误类型

1. **文件不存在**:
   ```
   File not found. Cannot apply edit. Use an empty old_string to create a new file.
   ```

2. **匹配失败**:
   ```
   Failed to edit, could not find the string to replace.
   ```

3. **匹配数量不符**:
   ```
   Failed to edit, expected 1 occurrence but found 3.
   ```

4. **相同内容**:
   ```
   No changes to apply. The old_string and new_string are identical.
   ```

### 错误处理策略

```typescript
try {
  const result = await editTool.execute(params, abortSignal);
  
  if (result.llmContent.startsWith('Error:')) {
    console.error('Edit failed:', result.returnDisplay);
    // 处理编辑失败
  } else {
    console.log('Edit successful:', result.returnDisplay);
  }
} catch (error) {
  console.error('Unexpected error:', error);
  // 处理意外错误
}
```

## 最佳实践

### 精确匹配原则

1. **包含上下文**: 在 old_string 中包含至少 3 行上下文
2. **保持格式**: 准确匹配空格、缩进和换行符
3. **避免转义**: 使用字面文本，避免不必要的转义

### 安全编辑

1. **路径验证**: 确保编辑路径在项目根目录内
2. **备份策略**: 重要编辑前考虑创建备份
3. **渐进式修改**: 大量修改分解为小步骤

### 性能优化

1. **批量编辑**: 使用多重替换减少工具调用次数
2. **内容缓存**: 避免重复读取相同文件
3. **异步处理**: 利用 AbortSignal 支持取消操作

## 集成特性

### 与 Config 集成

- 审批模式控制 (DEFAULT/AUTO_EDIT/YOLO)
- 根目录路径验证
- Gemini 客户端集成用于编辑修正

### 与其他工具协作

- **ReadFileTool**: 编辑前检查文件内容
- **ensureCorrectEdit**: AI 辅助的编辑修正
- **Diff**: 变更可视化显示

### 遥测和监控

- 编辑操作统计
- 错误率监控
- 用户行为分析

## 扩展性

### 自定义编辑策略

可以扩展 EditTool 以支持更复杂的编辑模式:

```typescript
class AdvancedEditTool extends EditTool {
  async executeRegexReplace(params: RegexEditParams) {
    // 正则表达式替换实现
  }
  
  async executeBlockReplace(params: BlockEditParams) {
    // 代码块替换实现
  }
}
```

### 插件系统集成

EditTool 可以与插件系统集成，支持自定义编辑处理器和验证器。