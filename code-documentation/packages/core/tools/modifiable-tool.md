# ModifiableTool 可修改工具接口文档

## 概述

`ModifiableTool` 是 Gemini CLI 中支持外部编辑器修改操作的工具接口。它为工具提供了与外部编辑器集成的能力，允许用户在外部编辑器中查看和修改工具提议的内容，然后将修改后的内容反馈到工具执行流程中。这个接口主要用于需要用户交互式修改内容的场景。

## 主要功能

- 外部编辑器集成支持
- 差异对比和可视化
- 临时文件管理和清理
- 修改内容的双向同步
- 类型安全的参数更新机制
- 自动化的差异计算和显示

## 接口定义

### `ModifiableTool<ToolParams>`

**功能**: 支持修改操作的工具接口

```typescript
export interface ModifiableTool<ToolParams> extends Tool<ToolParams> {
  getModifyContext(abortSignal: AbortSignal): ModifyContext<ToolParams>;
}
```

**扩展能力**:

- 继承所有 `Tool<ToolParams>` 的基础功能
- 添加修改上下文获取能力
- 支持用户交互式内容修改

### `ModifyContext<ToolParams>`

**功能**: 修改操作上下文接口

```typescript
export interface ModifyContext<ToolParams> {
  getFilePath: (params: ToolParams) => string;
  getCurrentContent: (params: ToolParams) => Promise<string>;
  getProposedContent: (params: ToolParams) => Promise<string>;
  createUpdatedParams: (
    oldContent: string,
    modifiedProposedContent: string,
    originalParams: ToolParams,
  ) => ToolParams;
}
```

**方法详情**:

- `getFilePath`: 获取目标文件路径
- `getCurrentContent`: 获取当前文件内容
- `getProposedContent`: 获取工具提议的新内容
- `createUpdatedParams`: 基于修改后的内容创建更新的参数

### `ModifyResult<ToolParams>`

**功能**: 修改操作结果接口

```typescript
export interface ModifyResult<ToolParams> {
  updatedParams: ToolParams;           // 更新后的工具参数
  updatedDiff: string;                 // 更新后的差异字符串
}
```

## 核心工具函数

### 类型检查函数

#### `isModifiableTool<TParams>(tool: Tool<TParams>): tool is ModifiableTool<TParams>`

**功能**: 检查工具是否实现了 ModifiableTool 接口

```typescript
export function isModifiableTool<TParams>(
  tool: Tool<TParams>,
): tool is ModifiableTool<TParams> {
  return 'getModifyContext' in tool;
}
```

**使用示例**:

```typescript
if (isModifiableTool(myTool)) {
  // 类型安全地访问 ModifiableTool 特定方法
  const modifyContext = myTool.getModifyContext(abortSignal);
}
```

### 临时文件管理

#### `createTempFilesForModify(currentContent: string, proposedContent: string, file_path: string)`

**功能**: 为修改操作创建临时文件

```typescript
function createTempFilesForModify(
  currentContent: string,
  proposedContent: string,
  file_path: string,
): { oldPath: string; newPath: string } {
  const tempDir = os.tmpdir();
  const diffDir = path.join(tempDir, 'gemini-cli-tool-modify-diffs');

  if (!fs.existsSync(diffDir)) {
    fs.mkdirSync(diffDir, { recursive: true });
  }

  const ext = path.extname(file_path);
  const fileName = path.basename(file_path, ext);
  const timestamp = Date.now();
  
  const tempOldPath = path.join(
    diffDir,
    `gemini-cli-modify-${fileName}-old-${timestamp}${ext}`,
  );
  const tempNewPath = path.join(
    diffDir,
    `gemini-cli-modify-${fileName}-new-${timestamp}${ext}`,
  );

  fs.writeFileSync(tempOldPath, currentContent, 'utf8');
  fs.writeFileSync(tempNewPath, proposedContent, 'utf8');

  return { oldPath: tempOldPath, newPath: tempNewPath };
}
```

**临时文件命名规则**:
- 格式: `gemini-cli-modify-{fileName}-{type}-{timestamp}{ext}`
- 示例: `gemini-cli-modify-app-old-1672531200000.ts`
- 位置: 系统临时目录下的 `gemini-cli-tool-modify-diffs` 文件夹

### 参数更新和差异计算

#### `getUpdatedParams<ToolParams>(...)`

**功能**: 基于修改后的内容计算更新的参数和差异

```typescript
function getUpdatedParams<ToolParams>(
  tmpOldPath: string,
  tempNewPath: string,
  originalParams: ToolParams,
  modifyContext: ModifyContext<ToolParams>,
): { updatedParams: ToolParams; updatedDiff: string } {
  let oldContent = '';
  let newContent = '';

  // 安全地读取临时文件内容
  try {
    oldContent = fs.readFileSync(tmpOldPath, 'utf8');
  } catch (err) {
    if (!isNodeError(err) || err.code !== 'ENOENT') throw err;
    oldContent = '';
  }

  try {
    newContent = fs.readFileSync(tempNewPath, 'utf8');
  } catch (err) {
    if (!isNodeError(err) || err.code !== 'ENOENT') throw err;
    newContent = '';
  }

  // 创建更新的参数
  const updatedParams = modifyContext.createUpdatedParams(
    oldContent,
    newContent,
    originalParams,
  );

  // 计算差异
  const updatedDiff = Diff.createPatch(
    path.basename(modifyContext.getFilePath(originalParams)),
    oldContent,
    newContent,
    'Current',
    'Proposed',
    DEFAULT_DIFF_OPTIONS,
  );

  return { updatedParams, updatedDiff };
}
```

### 临时文件清理

#### `deleteTempFiles(oldPath: string, newPath: string): void`

**功能**: 清理修改操作创建的临时文件

```typescript
function deleteTempFiles(oldPath: string, newPath: string): void {
  try {
    fs.unlinkSync(oldPath);
  } catch {
    console.error(`Error deleting temp diff file: ${oldPath}`);
  }

  try {
    fs.unlinkSync(newPath);
  } catch {
    console.error(`Error deleting temp diff file: ${newPath}`);
  }
}
```

## 主要执行函数

### `modifyWithEditor<ToolParams>(...)`

**功能**: 使用外部编辑器进行内容修改的主要函数

```typescript
export async function modifyWithEditor<ToolParams>(
  originalParams: ToolParams,
  modifyContext: ModifyContext<ToolParams>,
  editorType: EditorType,
  _abortSignal: AbortSignal,
): Promise<ModifyResult<ToolParams>> {
  // 1. 获取当前内容和提议内容
  const currentContent = await modifyContext.getCurrentContent(originalParams);
  const proposedContent = await modifyContext.getProposedContent(originalParams);

  // 2. 创建临时文件
  const { oldPath, newPath } = createTempFilesForModify(
    currentContent,
    proposedContent,
    modifyContext.getFilePath(originalParams),
  );

  try {
    // 3. 打开外部编辑器进行差异对比
    await openDiff(oldPath, newPath, editorType);
    
    // 4. 处理修改后的内容
    const result = getUpdatedParams(
      oldPath,
      newPath,
      originalParams,
      modifyContext,
    );

    return result;
  } finally {
    // 5. 清理临时文件
    deleteTempFiles(oldPath, newPath);
  }
}
```

**执行流程**:

1. **内容获取**: 从修改上下文获取当前内容和提议内容
2. **临时文件创建**: 在系统临时目录创建对比文件
3. **外部编辑器启动**: 使用指定的编辑器打开差异视图
4. **用户交互**: 用户在编辑器中查看和修改内容
5. **内容同步**: 读取用户修改后的内容
6. **参数更新**: 基于修改内容更新工具参数
7. **差异计算**: 计算最终的差异字符串
8. **清理**: 删除临时文件

## 使用示例

### 实现 ModifiableTool

```typescript
class EditTool extends BaseTool<EditToolParams, ToolResult> 
  implements ModifiableTool<EditToolParams> {
  
  // 实现 ModifiableTool 接口
  getModifyContext(abortSignal: AbortSignal): ModifyContext<EditToolParams> {
    return {
      getFilePath: (params: EditToolParams) => {
        return path.resolve(this.config.getTargetDir(), params.file_path);
      },

      getCurrentContent: async (params: EditToolParams) => {
        const filePath = path.resolve(this.config.getTargetDir(), params.file_path);
        try {
          return await fs.promises.readFile(filePath, 'utf-8');
        } catch (error) {
          if (isNodeError(error) && error.code === 'ENOENT') {
            return ''; // 文件不存在，返回空内容
          }
          throw error;
        }
      },

      getProposedContent: async (params: EditToolParams) => {
        // 根据编辑操作类型生成提议内容
        const currentContent = await this.getCurrentContent(params);
        return this.applyEditOperations(currentContent, params.operations);
      },

      createUpdatedParams: (
        oldContent: string,
        modifiedProposedContent: string,
        originalParams: EditToolParams,
      ) => {
        // 基于修改后的内容重新计算编辑操作
        const updatedOperations = this.calculateOperationsFromDiff(
          oldContent,
          modifiedProposedContent,
        );

        return {
          ...originalParams,
          operations: updatedOperations,
        };
      },
    };
  }

  private applyEditOperations(content: string, operations: EditOperation[]): string {
    // 应用编辑操作的逻辑
    let result = content;
    for (const operation of operations) {
      result = this.applyOperation(result, operation);
    }
    return result;
  }

  private calculateOperationsFromDiff(
    oldContent: string,
    newContent: string,
  ): EditOperation[] {
    // 基于差异计算编辑操作的逻辑
    const diff = Diff.diffLines(oldContent, newContent);
    const operations: EditOperation[] = [];

    let lineNumber = 1;
    for (const change of diff) {
      if (change.added) {
        operations.push({
          type: 'insert',
          line: lineNumber,
          content: change.value,
        });
      } else if (change.removed) {
        operations.push({
          type: 'delete',
          line: lineNumber,
          count: change.count || 1,
        });
      }
      if (!change.added) {
        lineNumber += change.count || 1;
      }
    }

    return operations;
  }
}
```

### 使用 modifyWithEditor

```typescript
async function handleToolModification<TParams>(
  tool: ModifiableTool<TParams>,
  params: TParams,
  editorType: EditorType,
  abortSignal: AbortSignal,
): Promise<ModifyResult<TParams>> {
  const modifyContext = tool.getModifyContext(abortSignal);
  
  const result = await modifyWithEditor(
    params,
    modifyContext,
    editorType,
    abortSignal,
  );

  console.log('修改完成:', {
    updatedParams: result.updatedParams,
    diffSummary: result.updatedDiff.split('\n').length + ' lines changed',
  });

  return result;
}

// 使用示例
const editTool = new EditTool(config);
if (isModifiableTool(editTool)) {
  const result = await handleToolModification(
    editTool,
    { file_path: 'src/app.ts', operations: [...] },
    EditorType.VSCode,
    new AbortController().signal,
  );
  
  // 使用修改后的参数重新执行工具
  await editTool.execute(result.updatedParams, new AbortController().signal);
}
```

## 高级功能

### 自定义修改上下文

```typescript
class AdvancedModifyContext<TParams> implements ModifyContext<TParams> {
  constructor(
    private backup: BackupService,
    private validator: ContentValidator,
    private formatter: CodeFormatter,
  ) {}

  async getCurrentContent(params: TParams): Promise<string> {
    const content = await this.readFileContent(params);
    
    // 创建备份
    await this.backup.createBackup(this.getFilePath(params), content);
    
    return content;
  }

  async getProposedContent(params: TParams): Promise<string> {
    const rawContent = await this.generateProposedContent(params);
    
    // 格式化提议内容
    const formattedContent = await this.formatter.format(
      rawContent,
      this.getFileType(params),
    );
    
    // 验证内容
    const validationResult = await this.validator.validate(formattedContent);
    if (!validationResult.isValid) {
      throw new Error(`Proposed content validation failed: ${validationResult.errors.join(', ')}`);
    }
    
    return formattedContent;
  }

  createUpdatedParams(
    oldContent: string,
    modifiedProposedContent: string,
    originalParams: TParams,
  ): TParams {
    // 验证修改后的内容
    this.validateModifiedContent(oldContent, modifiedProposedContent);
    
    // 创建更新的参数
    return this.computeUpdatedParams(
      oldContent,
      modifiedProposedContent,
      originalParams,
    );
  }

  private validateModifiedContent(oldContent: string, newContent: string): void {
    // 检查修改的合理性
    const changeRatio = this.calculateChangeRatio(oldContent, newContent);
    if (changeRatio > 0.8) {
      console.warn('警告：修改比例超过80%，请确认修改正确性');
    }
  }

  private calculateChangeRatio(oldContent: string, newContent: string): number {
    const oldLines = oldContent.split('\n');
    const newLines = newContent.split('\n');
    const diff = Diff.diffLines(oldContent, newContent);
    
    let changedLines = 0;
    for (const change of diff) {
      if (change.added || change.removed) {
        changedLines += change.count || 1;
      }
    }
    
    return changedLines / Math.max(oldLines.length, newLines.length);
  }
}
```

### 批量修改支持

```typescript
class BatchModifiableTool<TParams> {
  constructor(private tools: ModifiableTool<TParams>[]) {}

  async modifyMultiple(
    paramsArray: TParams[],
    editorType: EditorType,
    abortSignal: AbortSignal,
  ): Promise<ModifyResult<TParams>[]> {
    const results: ModifyResult<TParams>[] = [];

    for (let i = 0; i < this.tools.length; i++) {
      const tool = this.tools[i];
      const params = paramsArray[i];

      if (abortSignal.aborted) {
        throw new Error('批量修改操作被取消');
      }

      try {
        const result = await modifyWithEditor(
          params,
          tool.getModifyContext(abortSignal),
          editorType,
          abortSignal,
        );
        results.push(result);
      } catch (error) {
        console.error(`修改第 ${i + 1} 个工具时出错:`, error);
        throw error;
      }
    }

    return results;
  }

  async previewChanges(
    paramsArray: TParams[]
  ): Promise<Array<{ tool: string; diff: string }>> {
    const previews: Array<{ tool: string; diff: string }> = [];

    for (let i = 0; i < this.tools.length; i++) {
      const tool = this.tools[i];
      const params = paramsArray[i];
      const modifyContext = tool.getModifyContext(new AbortController().signal);

      const currentContent = await modifyContext.getCurrentContent(params);
      const proposedContent = await modifyContext.getProposedContent(params);

      const diff = Diff.createPatch(
        `tool-${i}`,
        currentContent,
        proposedContent,
        'Current',
        'Proposed',
        DEFAULT_DIFF_OPTIONS,
      );

      previews.push({
        tool: tool.name,
        diff,
      });
    }

    return previews;
  }
}
```

### 安全修改包装器

```typescript
class SecureModifyWrapper<TParams> implements ModifiableTool<TParams> {
  constructor(
    private wrappedTool: ModifiableTool<TParams>,
    private security: SecurityService,
  ) {}

  // 实现基础 Tool 接口
  name = this.wrappedTool.name;
  displayName = this.wrappedTool.displayName;
  description = this.wrappedTool.description;
  icon = this.wrappedTool.icon;
  schema = this.wrappedTool.schema;
  isOutputMarkdown = this.wrappedTool.isOutputMarkdown;
  canUpdateOutput = this.wrappedTool.canUpdateOutput;

  validateToolParams = this.wrappedTool.validateToolParams.bind(this.wrappedTool);
  getDescription = this.wrappedTool.getDescription.bind(this.wrappedTool);
  toolLocations = this.wrappedTool.toolLocations.bind(this.wrappedTool);
  shouldConfirmExecute = this.wrappedTool.shouldConfirmExecute.bind(this.wrappedTool);
  execute = this.wrappedTool.execute.bind(this.wrappedTool);

  getModifyContext(abortSignal: AbortSignal): ModifyContext<TParams> {
    const originalContext = this.wrappedTool.getModifyContext(abortSignal);

    return {
      getFilePath: originalContext.getFilePath,

      getCurrentContent: async (params: TParams) => {
        // 安全检查：验证文件访问权限
        const filePath = originalContext.getFilePath(params);
        await this.security.checkFileAccess(filePath, 'read');
        
        return originalContext.getCurrentContent(params);
      },

      getProposedContent: async (params: TParams) => {
        const content = await originalContext.getProposedContent(params);
        
        // 安全检查：扫描恶意内容
        await this.security.scanContent(content);
        
        return content;
      },

      createUpdatedParams: (oldContent, modifiedContent, originalParams) => {
        // 安全检查：验证修改内容
        this.security.validateContentChange(oldContent, modifiedContent);
        
        return originalContext.createUpdatedParams(
          oldContent,
          modifiedContent,
          originalParams,
        );
      },
    };
  }
}

// 使用示例
const secureEditTool = new SecureModifyWrapper(
  new EditTool(config),
  new SecurityService(),
);
```

## 错误处理

### 常见错误场景

#### 临时文件操作错误

```typescript
// 临时目录创建失败
Error: EACCES: permission denied, mkdir '/tmp/gemini-cli-tool-modify-diffs'

// 临时文件写入失败
Error: ENOSPC: no space left on device

// 临时文件读取失败
Error: ENOENT: no such file or directory
```

#### 外部编辑器错误

```typescript
// 编辑器启动失败
Error: Editor command not found: code

// 编辑器进程异常退出
Error: Editor process exited with code 1

// 用户取消编辑操作
Error: User cancelled the modification
```

#### 内容验证错误

```typescript
// 修改内容格式错误
Error: Modified content contains syntax errors

// 修改内容过大
Error: Modified content exceeds size limit

// 修改内容包含危险操作
Error: Modified content contains potentially harmful operations
```

### 错误恢复策略

```typescript
class RobustModifyWithEditor {
  static async execute<TParams>(
    originalParams: TParams,
    modifyContext: ModifyContext<TParams>,
    editorType: EditorType,
    abortSignal: AbortSignal,
    options: {
      maxRetries?: number;
      backupEnabled?: boolean;
      validateBeforeApply?: boolean;
    } = {},
  ): Promise<ModifyResult<TParams>> {
    const { maxRetries = 3, backupEnabled = true, validateBeforeApply = true } = options;
    
    let lastError: Error | null = null;
    let tempFiles: { oldPath: string; newPath: string } | null = null;

    // 创建备份
    let backupPath: string | null = null;
    if (backupEnabled) {
      const filePath = modifyContext.getFilePath(originalParams);
      const currentContent = await modifyContext.getCurrentContent(originalParams);
      backupPath = await this.createBackup(filePath, currentContent);
    }

    try {
      for (let attempt = 1; attempt <= maxRetries; attempt++) {
        try {
          const currentContent = await modifyContext.getCurrentContent(originalParams);
          const proposedContent = await modifyContext.getProposedContent(originalParams);

          tempFiles = createTempFilesForModify(
            currentContent,
            proposedContent,
            modifyContext.getFilePath(originalParams),
          );

          await openDiff(tempFiles.oldPath, tempFiles.newPath, editorType);

          const result = getUpdatedParams(
            tempFiles.oldPath,
            tempFiles.newPath,
            originalParams,
            modifyContext,
          );

          // 验证修改结果
          if (validateBeforeApply) {
            await this.validateModificationResult(result, modifyContext);
          }

          return result;
        } catch (error) {
          lastError = error instanceof Error ? error : new Error(String(error));
          
          if (attempt < maxRetries) {
            console.warn(`修改尝试 ${attempt} 失败，重试中... 错误: ${lastError.message}`);
            await new Promise(resolve => setTimeout(resolve, 1000 * attempt));
          }
        } finally {
          if (tempFiles) {
            deleteTempFiles(tempFiles.oldPath, tempFiles.newPath);
            tempFiles = null;
          }
        }
      }

      // 所有重试都失败
      throw new Error(`修改操作失败，已重试 ${maxRetries} 次。最后错误: ${lastError?.message}`);
    } catch (error) {
      // 恢复备份
      if (backupPath && backupEnabled) {
        await this.restoreFromBackup(
          modifyContext.getFilePath(originalParams),
          backupPath,
        );
      }
      throw error;
    } finally {
      // 清理备份
      if (backupPath) {
        await this.cleanupBackup(backupPath);
      }
    }
  }

  private static async createBackup(filePath: string, content: string): Promise<string> {
    const backupDir = path.join(os.tmpdir(), 'gemini-cli-backups');
    await fs.promises.mkdir(backupDir, { recursive: true });
    
    const timestamp = Date.now();
    const backupPath = path.join(
      backupDir,
      `${path.basename(filePath)}-backup-${timestamp}`,
    );
    
    await fs.promises.writeFile(backupPath, content, 'utf-8');
    return backupPath;
  }

  private static async restoreFromBackup(filePath: string, backupPath: string): Promise<void> {
    try {
      const backupContent = await fs.promises.readFile(backupPath, 'utf-8');
      await fs.promises.writeFile(filePath, backupContent, 'utf-8');
      console.log(`已从备份恢复文件: ${filePath}`);
    } catch (error) {
      console.error(`恢复备份失败: ${error}`);
    }
  }

  private static async cleanupBackup(backupPath: string): Promise<void> {
    try {
      await fs.promises.unlink(backupPath);
    } catch (error) {
      // 忽略清理失败
    }
  }

  private static async validateModificationResult<TParams>(
    result: ModifyResult<TParams>,
    modifyContext: ModifyContext<TParams>,
  ): Promise<void> {
    // 基本验证：检查差异是否合理
    const diffLines = result.updatedDiff.split('\n');
    const hasChanges = diffLines.some(line => line.startsWith('+') || line.startsWith('-'));
    
    if (!hasChanges) {
      throw new Error('修改操作没有产生任何变化');
    }

    // 大小验证：防止意外的大量修改
    const addedLines = diffLines.filter(line => line.startsWith('+')).length;
    const removedLines = diffLines.filter(line => line.startsWith('-')).length;
    const totalChanges = addedLines + removedLines;

    if (totalChanges > 1000) {
      console.warn(`警告：修改涉及 ${totalChanges} 行，请确认修改正确性`);
    }
  }
}
```

## 集成特性

### 与编辑器集成

```typescript
// 支持多种编辑器类型
enum EditorType {
  VSCode = 'vscode',
  Vim = 'vim',
  Emacs = 'emacs',
  Sublime = 'sublime',
  Atom = 'atom',
  Default = 'default',
}

// 编辑器配置管理
class EditorManager {
  private static editorCommands: Record<EditorType, string[]> = {
    [EditorType.VSCode]: ['code', '--diff'],
    [EditorType.Vim]: ['vim', '-d'],
    [EditorType.Emacs]: ['emacs', '--eval', '(ediff-files "file1" "file2")'],
    [EditorType.Sublime]: ['subl', '--command', 'sftp_diff'],
    [EditorType.Atom]: ['atom', '--diff'],
    [EditorType.Default]: ['diff'],
  };

  static async detectAvailableEditor(): Promise<EditorType> {
    for (const [editorType, command] of Object.entries(this.editorCommands)) {
      try {
        const { execFile } = await import('child_process');
        await new Promise((resolve, reject) => {
          execFile('which', [command[0]], (error) => {
            if (error) reject(error);
            else resolve(null);
          });
        });
        return editorType as EditorType;
      } catch {
        continue;
      }
    }
    return EditorType.Default;
  }
}
```

### 与配置系统集成

```typescript
interface ModifyToolConfig {
  defaultEditor: EditorType;
  enableBackup: boolean;
  maxRetries: number;
  tempFileRetention: number; // 小时
  validationEnabled: boolean;
}

class ConfigurableModifiableTool<TParams> implements ModifiableTool<TParams> {
  constructor(
    private baseTool: ModifiableTool<TParams>,
    private config: ModifyToolConfig,
  ) {}

  async executeModify(
    params: TParams,
    abortSignal: AbortSignal,
  ): Promise<ModifyResult<TParams>> {
    return RobustModifyWithEditor.execute(
      params,
      this.getModifyContext(abortSignal),
      this.config.defaultEditor,
      abortSignal,
      {
        maxRetries: this.config.maxRetries,
        backupEnabled: this.config.enableBackup,
        validateBeforeApply: this.config.validationEnabled,
      },
    );
  }
}
```

## 最佳实践

### 安全性最佳实践

```typescript
// ✅ 推荐：安全的文件路径处理
class SecureModifyContext<TParams> implements ModifyContext<TParams> {
  getFilePath(params: TParams): string {
    const filePath = this.extractFilePath(params);
    
    // 验证路径安全性
    if (filePath.includes('..') || path.isAbsolute(filePath)) {
      throw new Error('不安全的文件路径');
    }
    
    // 限制在工作目录内
    const resolvedPath = path.resolve(this.workingDir, filePath);
    if (!resolvedPath.startsWith(this.workingDir)) {
      throw new Error('文件路径超出工作目录范围');
    }
    
    return resolvedPath;
  }
}

// ✅ 推荐：内容验证
async getProposedContent(params: TParams): Promise<string> {
  const content = await this.generateContent(params);
  
  // 验证内容大小
  if (content.length > MAX_CONTENT_SIZE) {
    throw new Error('提议内容过大');
  }
  
  // 验证内容安全性
  if (this.containsMaliciousPattern(content)) {
    throw new Error('提议内容包含潜在危险操作');
  }
  
  return content;
}
```

### 性能最佳实践

```typescript
// ✅ 推荐：异步内容处理
class OptimizedModifyContext<TParams> implements ModifyContext<TParams> {
  private contentCache = new Map<string, string>();
  
  async getCurrentContent(params: TParams): Promise<string> {
    const filePath = this.getFilePath(params);
    const cacheKey = `current:${filePath}`;
    
    if (this.contentCache.has(cacheKey)) {
      return this.contentCache.get(cacheKey)!;
    }
    
    const content = await fs.promises.readFile(filePath, 'utf-8');
    this.contentCache.set(cacheKey, content);
    
    return content;
  }
  
  clearCache(): void {
    this.contentCache.clear();
  }
}

// ✅ 推荐：批量操作优化
class BatchOptimizedModify<TParams> {
  async modifyMultiple(
    operations: Array<{ tool: ModifiableTool<TParams>; params: TParams }>,
    editorType: EditorType,
  ): Promise<ModifyResult<TParams>[]> {
    // 并行准备所有修改上下文
    const contexts = await Promise.all(
      operations.map(async ({ tool, params }) => ({
        tool,
        params,
        context: tool.getModifyContext(new AbortController().signal),
        currentContent: await tool.getModifyContext(new AbortController().signal).getCurrentContent(params),
        proposedContent: await tool.getModifyContext(new AbortController().signal).getProposedContent(params),
      }))
    );
    
    // 序列化执行修改以避免编辑器冲突
    const results: ModifyResult<TParams>[] = [];
    for (const ctx of contexts) {
      const result = await modifyWithEditor(
        ctx.params,
        ctx.context,
        editorType,
        new AbortController().signal,
      );
      results.push(result);
    }
    
    return results;
  }
}
```

### 可维护性建议

```typescript
// ✅ 推荐：模块化修改上下文
abstract class BaseModifyContext<TParams> implements ModifyContext<TParams> {
  abstract getFilePath(params: TParams): string;
  abstract extractOperations(params: TParams): unknown[];
  abstract applyOperations(content: string, operations: unknown[]): string;
  
  async getCurrentContent(params: TParams): Promise<string> {
    const filePath = this.getFilePath(params);
    return this.readFileWithRetry(filePath);
  }
  
  async getProposedContent(params: TParams): Promise<string> {
    const currentContent = await this.getCurrentContent(params);
    const operations = this.extractOperations(params);
    return this.applyOperations(currentContent, operations);
  }
  
  protected async readFileWithRetry(
    filePath: string,
    maxRetries: number = 3,
  ): Promise<string> {
    for (let i = 0; i < maxRetries; i++) {
      try {
        return await fs.promises.readFile(filePath, 'utf-8');
      } catch (error) {
        if (i === maxRetries - 1) throw error;
        await new Promise(resolve => setTimeout(resolve, 100 * (i + 1)));
      }
    }
    throw new Error('Unreachable');
  }
}

// 具体实现
class EditModifyContext extends BaseModifyContext<EditToolParams> {
  getFilePath(params: EditToolParams): string {
    return path.resolve(this.baseDir, params.file_path);
  }
  
  extractOperations(params: EditToolParams): EditOperation[] {
    return params.operations;
  }
  
  applyOperations(content: string, operations: EditOperation[]): string {
    return this.editService.applyOperations(content, operations);
  }
}
```