# ShellTool 类文档

## 概述
`ShellTool` 是 Gemini CLI 的 Shell 命令执行工具，负责安全地执行用户输入的 shell 命令。它提供了命令验证、权限确认、流式输出更新和后台进程管理等功能。

## 主要功能
- 安全的 shell 命令执行
- 命令白名单和权限管理
- 实时输出流更新
- 后台进程跟踪和管理
- 二进制输出检测
- 命令取消和信号处理

---

## 常量定义

### `OUTPUT_UPDATE_INTERVAL_MS`
**值**: `1000`
**用途**: 输出更新间隔（毫秒），控制流式输出的更新频率

---

## 接口定义

### `ShellToolParams`
```typescript
interface ShellToolParams {
  command: string;      // 要执行的 shell 命令
  description?: string; // 命令的简要描述（可选）
  directory?: string;   // 执行目录，相对于项目根目录（可选）
}
```

---

## 类属性

### 静态属性
- `static Name: string = 'run_shell_command'` - 工具名称标识符

### 实例属性
- `private allowlist: Set<string>` - 已获得用户许可的命令白名单
- `private readonly config: Config` - 配置对象引用

---

## 构造函数

### `constructor(config: Config)`
**功能**: 初始化 ShellTool 实例
**参数**:
- `config: Config` - 配置对象
**实现**:
- 调用父类构造函数设置工具元数据
- 定义工具的 JSON Schema 参数规范
- 设置工具特性（非 Markdown 输出、支持输出更新）

---

## 核心方法

### `getDescription(params: ShellToolParams): string`
**功能**: 生成命令的用户友好描述
**参数**:
- `params: ShellToolParams` - 工具参数
**返回**: `string` - 格式化的描述文本
**格式**: `{command} [in {directory}] ({description})`
**实现**:
1. 以命令本身作为基础描述
2. 如果指定了目录，添加 `[in directory]`
3. 如果提供了描述，添加 `(description)`，并将换行符替换为空格

### `validateToolParams(params: ShellToolParams): string | null`
**功能**: 验证工具参数的有效性
**参数**:
- `params: ShellToolParams` - 要验证的参数
**返回**: `string | null` - 错误信息或 null（表示验证通过）
**验证规则**:
1. **命令权限检查**: 使用 `isCommandAllowed()` 检查命令是否被允许
2. **Schema 验证**: 验证参数是否符合 JSON Schema 规范
3. **命令非空检查**: 确保命令去除空格后不为空
4. **命令根识别**: 确保能够识别命令根以获取用户权限
5. **目录验证**:
   - 不能是绝对路径
   - 必须存在于文件系统中

### `async shouldConfirmExecute(...): Promise<ToolCallConfirmationDetails | false>`
**功能**: 确定是否需要用户确认执行命令
**参数**:
- `params: ShellToolParams` - 工具参数
- `_abortSignal: AbortSignal` - 中止信号（未使用）
**返回**: `Promise<ToolCallConfirmationDetails | false>` - 确认详情或 false
**实现逻辑**:
1. 如果参数验证失败，跳过确认（执行时会立即失败）
2. 从命令中提取根命令
3. 过滤出未在白名单中的命令
4. 如果所有命令都在白名单中，跳过确认
5. 否则返回确认详情，包括：
   - 确认类型为 'exec'
   - 显示完整命令和根命令
   - 设置确认回调以更新白名单

### `async execute(...): Promise<ToolResult>`
**功能**: 执行 shell 命令
**参数**:
- `params: ShellToolParams` - 工具参数
- `signal: AbortSignal` - 中止信号
- `updateOutput?: (output: string) => void` - 输出更新回调（可选）
**返回**: `Promise<ToolResult>` - 执行结果

#### 执行流程

##### 1. 预处理和验证
```typescript
const strippedCommand = stripShellWrapper(params.command);
const validationError = this.validateToolParams({
  ...params,
  command: strippedCommand,
});
if (validationError) {
  return { llmContent: validationError, returnDisplay: validationError };
}
```

##### 2. 取消检查
```typescript
if (signal.aborted) {
  return {
    llmContent: 'Command was cancelled by user before it could start.',
    returnDisplay: 'Command cancelled by user.',
  };
}
```

##### 3. 平台特定处理
```typescript
const isWindows = os.platform() === 'win32';
const tempFileName = `shell_pgrep_${crypto.randomBytes(6).toString('hex')}.tmp`;
const tempFilePath = path.join(os.tmpdir(), tempFileName);
```

##### 4. 命令包装（非 Windows 系统）
```typescript
const commandToExecute = isWindows
  ? strippedCommand
  : (() => {
      let command = strippedCommand.trim();
      if (!command.endsWith('&')) command += ';';
      return `{ ${command} }; __code=$?; pgrep -g 0 >${tempFilePath} 2>&1; exit $__code;`;
    })();
```

##### 5. 执行和流式输出处理
```typescript
const { result: resultPromise } = ShellExecutionService.execute(
  commandToExecute,
  cwd,
  (event: ShellOutputEvent) => {
    // 处理不同类型的输出事件
    switch (event.type) {
      case 'data':
        // 累积标准输出和标准错误
        // 定期更新显示输出
        break;
      case 'binary_detected':
        // 检测到二进制输出，停止文本处理
        break;
      case 'binary_progress':
        // 显示二进制数据接收进度
        break;
    }
  },
  signal,
);
```

##### 6. 后台进程检测（非 Windows）
```typescript
const backgroundPIDs: number[] = [];
if (os.platform() !== 'win32') {
  if (fs.existsSync(tempFilePath)) {
    const pgrepLines = fs
      .readFileSync(tempFilePath, 'utf8')
      .split('\n')
      .filter(Boolean);
    for (const line of pgrepLines) {
      const pid = Number(line);
      if (pid !== result.pid) {
        backgroundPIDs.push(pid);
      }
    }
  }
}
```

##### 7. 结果格式化
```typescript
let llmContent = '';
if (result.aborted) {
  llmContent = 'Command was cancelled by user before it could complete.';
  // 添加取消前的输出信息
} else {
  llmContent = [
    `Command: ${params.command}`,
    `Directory: ${params.directory || '(root)'}`,
    `Stdout: ${result.stdout || '(empty)'}`,
    `Stderr: ${result.stderr || '(empty)'}`,
    `Error: ${finalError}`,
    `Exit Code: ${result.exitCode ?? '(none)'}`,
    `Signal: ${result.signal ?? '(none)'}`,
    `Background PIDs: ${backgroundPIDs.length ? backgroundPIDs.join(', ') : '(none)'}`,
    `Process Group PGID: ${result.pid ?? '(none)'}`,
  ].join('\n');
}
```

##### 8. 输出摘要化（如果配置）
```typescript
const summarizeConfig = this.config.getSummarizeToolOutputConfig();
if (summarizeConfig && summarizeConfig[this.name]) {
  const summary = await summarizeToolOutput(
    llmContent,
    this.config.getGeminiClient(),
    signal,
    summarizeConfig[this.name].tokenBudget,
  );
  return { llmContent: summary, returnDisplay: returnDisplayMessage };
}
```

---

## 输出事件处理

### 数据事件处理
- **累积输出**: 分别累积 stdout 和 stderr
- **节流更新**: 每秒最多更新一次显示输出
- **合并显示**: 将 stdout 和 stderr 合并显示

### 二进制输出处理
- **自动检测**: 检测到二进制输出时自动切换模式
- **进度显示**: 显示已接收的字节数
- **流量统计**: 使用 `formatMemoryUsage()` 格式化字节数

---

## 安全特性

### 命令验证
- **权限检查**: 使用 `isCommandAllowed()` 验证命令权限
- **根命令提取**: 使用 `getCommandRoots()` 提取需要确认的根命令
- **路径验证**: 确保目录参数不是绝对路径

### 用户确认
- **动态白名单**: 用户确认后将命令加入白名单
- **批量确认**: 支持 "总是允许" 选项
- **命令分组**: 按根命令分组进行确认

### 进程管理
- **进程组**: 使用进程组管理子进程
- **后台跟踪**: 跟踪后台进程 PID
- **信号处理**: 支持信号中止和清理

---

## 错误处理

### 验证错误
- 参数验证失败时返回具体错误信息
- 不执行命令，避免安全风险

### 执行错误
- 捕获并格式化执行错误
- 区分不同类型的失败（取消、信号、退出码）
- 保留部分输出信息以供调试

### 资源清理
```typescript
finally {
  if (fs.existsSync(tempFilePath)) {
    fs.unlinkSync(tempFilePath);
  }
}
```

---

## 使用示例

### 基本用法
```typescript
const shellTool = new ShellTool(config);

// 执行简单命令
const result = await shellTool.execute(
  { command: 'ls -la' },
  abortSignal
);

// 在指定目录执行命令
const result = await shellTool.execute(
  { 
    command: 'npm install',
    directory: 'frontend',
    description: 'Install frontend dependencies'
  },
  abortSignal,
  (output) => console.log('Progress:', output)
);
```

### 权限确认流程
```typescript
// 检查是否需要确认
const confirmationDetails = await shellTool.shouldConfirmExecute(params, signal);

if (confirmationDetails) {
  // 显示确认对话框
  const outcome = await showConfirmationDialog(confirmationDetails);
  
  // 根据用户选择更新白名单
  await confirmationDetails.onConfirm(outcome);
}

// 执行命令
const result = await shellTool.execute(params, signal, updateCallback);
```

---

## 依赖关系

### 外部依赖
- `fs`, `path`, `os`, `crypto` - Node.js 核心模块
- `@google/genai` - Google GenAI SDK（类型定义）

### 内部依赖
- `./tools.js` - 基础工具类和接口
- `../config/config.js` - 配置管理
- `../services/shellExecutionService.js` - Shell 执行服务
- `../utils/*` - 各种工具函数

---

## 特殊功能

### 跨平台支持
- **Windows**: 不支持后台进程跟踪（pgrep 不可用）
- **Unix/Linux**: 完整的进程组和后台进程管理

### 性能优化
- **流式输出**: 实时显示命令输出
- **节流更新**: 避免过于频繁的 UI 更新
- **二进制检测**: 自动处理二进制输出以避免性能问题

### 调试支持
- **详细输出**: 调试模式下显示完整的执行信息
- **进程信息**: 包含进程组 ID 和后台进程列表
- **错误上下文**: 提供丰富的错误上下文信息