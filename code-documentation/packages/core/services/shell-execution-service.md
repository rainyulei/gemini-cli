# ShellExecutionService Shell 命令执行服务文档

## 概述

`ShellExecutionService` 是 Gemini CLI 的 Shell 命令执行服务，提供跨平台的命令行执行能力。它具有强大的进程管理、流式输出处理、二进制内容检测和优雅的进程终止功能，是系统工具和命令行集成的核心组件。

## 主要功能

- 跨平台 Shell 命令执行（Windows/Unix）
- 流式输出事件处理
- 二进制内容自动检测
- 进程组管理和优雅终止
- 多字节字符编码支持
- AbortSignal 中断控制
- 详细的执行结果结构

## 接口定义

### `ShellExecutionResult`

**功能**: Shell 命令执行结果结构

```typescript
export interface ShellExecutionResult {
  /** 原始未处理的输出缓冲区 */
  rawOutput: Buffer;
  /** 合并并解码的 stdout 和 stderr 字符串 */
  output: string;
  /** 解码后的 stdout 字符串 */
  stdout: string;
  /** 解码后的 stderr 字符串 */
  stderr: string;
  /** 进程退出码，如果被信号终止则为 null */
  exitCode: number | null;
  /** 终止进程的信号（如果有） */
  signal: NodeJS.Signals | null;
  /** 进程启动失败时的错误对象 */ 
  error: Error | null;
  /** 标识命令是否被用户中止 */
  aborted: boolean;
  /** 生成的 shell 进程 ID */
  pid: number | undefined;
}
```

**字段详情**:

- `rawOutput`: 包含完整的原始二进制输出数据
- `output`: 组合的文本输出，格式为 `stdout + "\n" + stderr`（如果有stderr）
- `stdout/stderr`: 分别包含标准输出和错误输出的解码文本
- `exitCode`: 正常退出时的状态码，被信号终止时为 null
- `signal`: 终止信号名称（如 'SIGTERM', 'SIGKILL'）
- `error`: 进程启动阶段的错误（非执行错误）
- `aborted`: 区分用户主动中止和自然结束
- `pid`: 用于进程跟踪和管理

### `ShellExecutionHandle`

**功能**: 正在进行的 Shell 执行句柄

```typescript
export interface ShellExecutionHandle {
  /** 生成的 shell 进程 ID */
  pid: number | undefined;
  /** 解析为完整执行结果的 Promise */
  result: Promise<ShellExecutionResult>;
}
```

**使用场景**:
- 异步命令执行跟踪
- 进程 ID 获取用于外部管理
- 结果等待和处理

### `ShellOutputEvent`

**功能**: Shell 执行期间发出的结构化事件

```typescript
export type ShellOutputEvent =
  | {
      /** 事件包含输出数据块 */
      type: 'data';
      /** 数据来源流 */
      stream: 'stdout' | 'stderr';
      /** 解码后的字符串块 */
      chunk: string;
    }
  | {
      /** 表示输出流被识别为二进制 */
      type: 'binary_detected';
    }
  | {
      /** 为二进制流提供进度更新 */
      type: 'binary_progress';
      /** 到目前为止接收的总字节数 */
      bytesReceived: number;
    };
```

**事件类型详解**:

1. **data 事件**: 提供实时的文本输出块
   - 适用于日志显示、进度跟踪
   - 经过 ANSI 转义序列清理
   - 区分 stdout 和 stderr 来源

2. **binary_detected 事件**: 检测到二进制内容
   - 停止文本流式传输
   - 切换到二进制处理模式
   - 防止二进制数据损坏终端显示

3. **binary_progress 事件**: 二进制内容下载进度
   - 提供字节级进度信息
   - 用于大文件下载的进度条显示

## 核心类定义

### `ShellExecutionService`

**功能**: Shell 命令执行服务主类

```typescript
export class ShellExecutionService {
  /**
   * 使用 spawn 执行 shell 命令，捕获所有输出和生命周期事件
   *
   * @param commandToExecute 要运行的确切命令字符串
   * @param cwd 执行命令的工作目录
   * @param onOutputEvent 用于流式结构化事件的回调，包括数据块和状态更新
   * @param abortSignal 用于终止进程及其子进程的 AbortSignal
   * @returns 包含进程 ID 和完整执行结果 Promise 的对象
   */
  static execute(
    commandToExecute: string,
    cwd: string,
    onOutputEvent: (event: ShellOutputEvent) => void,
    abortSignal: AbortSignal,
  ): ShellExecutionHandle;
}
```

## 核心方法实现

### `execute` 方法详解

```typescript
static execute(
  commandToExecute: string,
  cwd: string,
  onOutputEvent: (event: ShellOutputEvent) => void,
  abortSignal: AbortSignal,
): ShellExecutionHandle {
  // 跨平台 Shell 检测
  const isWindows = os.platform() === 'win32';
  const shell = isWindows ? 'cmd.exe' : 'bash';
  const shellArgs = [isWindows ? '/c' : '-c', commandToExecute];

  // 进程启动配置
  const child = spawn(shell, shellArgs, {
    cwd,
    stdio: ['ignore', 'pipe', 'pipe'],  // 忽略stdin，管道化stdout/stderr
    detached: !isWindows,               // Unix系统使用进程组
    env: {
      ...process.env,
      GEMINI_CLI: '1',                  // 标识CLI环境
    },
  });

  // 返回执行句柄
  return { pid: child.pid, result: promiseResult };
}
```

**跨平台处理**:

1. **Windows**: 使用 `cmd.exe /c` 执行命令
2. **Unix/Linux/macOS**: 使用 `bash -c` 执行命令
3. **进程组**: Unix 系统启用 `detached` 模式便于进程树管理

**环境变量注入**:
- `GEMINI_CLI=1`: 标识当前运行在 Gemini CLI 环境中
- 继承所有现有环境变量
- 便于子进程识别执行上下文

### 输出处理机制

```typescript
const handleOutput = (data: Buffer, stream: 'stdout' | 'stderr') => {
  // 编码检测和解码器初始化
  if (!stdoutDecoder || !stderrDecoder) {
    const encoding = getCachedEncodingForBuffer(data);
    try {
      stdoutDecoder = new TextDecoder(encoding);
      stderrDecoder = new TextDecoder(encoding);
    } catch {
      // 编码不支持时回退到 UTF-8
      stdoutDecoder = new TextDecoder('utf-8');
      stderrDecoder = new TextDecoder('utf-8');
    }
  }

  outputChunks.push(data);

  // 二进制检测逻辑
  if (isStreamingRawContent && sniffedBytes < MAX_SNIFF_SIZE) {
    const sniffBuffer = Buffer.concat(outputChunks.slice(0, 20));
    sniffedBytes = sniffBuffer.length;

    if (isBinary(sniffBuffer)) {
      isStreamingRawContent = false;
      onOutputEvent({ type: 'binary_detected' });
    }
  }

  // 内容解码和清理
  const decodedChunk = stream === 'stdout'
    ? stdoutDecoder.decode(data, { stream: true })
    : stderrDecoder.decode(data, { stream: true });
  const strippedChunk = stripAnsi(decodedChunk);

  // 累积输出
  if (stream === 'stdout') {
    stdout += strippedChunk;
  } else {
    stderr += strippedChunk;
  }

  // 发送适当的事件
  if (isStreamingRawContent) {
    onOutputEvent({ type: 'data', stream, chunk: strippedChunk });
  } else {
    const totalBytes = outputChunks.reduce((sum, chunk) => sum + chunk.length, 0);
    onOutputEvent({ type: 'binary_progress', bytesReceived: totalBytes });
  }
};
```

**处理流程**:

1. **编码检测**: 使用 `getCachedEncodingForBuffer` 智能检测字符编码
2. **解码器管理**: 为 stdout 和 stderr 分别维护解码器
3. **二进制检测**: 前 4KB 内容检测是否为二进制数据
4. **ANSI 清理**: 移除终端控制序列，保持纯文本输出
5. **事件分发**: 根据内容类型发送不同的事件类型

### 进程终止机制

```typescript
const abortHandler = async () => {
  if (child.pid && !exited) {
    if (isWindows) {
      // Windows: 使用 taskkill 强制终止进程树
      spawn('taskkill', ['/pid', child.pid.toString(), '/f', '/t']);
    } else {
      try {
        // Unix: 优雅终止整个进程组
        process.kill(-child.pid, 'SIGTERM');
        await new Promise((res) => setTimeout(res, SIGKILL_TIMEOUT_MS));
        
        if (!exited) {
          process.kill(-child.pid, 'SIGKILL');
        }
      } catch (_e) {
        // 进程组终止失败时回退到单进程终止
        if (!exited) child.kill('SIGKILL');
      }
    }
  }
};

abortSignal.addEventListener('abort', abortHandler, { once: true });
```

**终止策略**:

1. **Windows**: 使用 `taskkill /f /t` 强制终止进程树
2. **Unix**: 分两阶段终止
   - 首先发送 `SIGTERM` 给进程组（负PID）
   - 等待 200ms 后发送 `SIGKILL`
   - 进程组操作失败时回退到单进程终止
3. **清理**: 移除事件监听器防止内存泄漏

## 使用示例

### 基本命令执行

```typescript
import { ShellExecutionService } from './shellExecutionService.js';

// 基本命令执行
async function runCommand(command: string, workingDir: string): Promise<string> {
  const abortController = new AbortController();
  
  const handle = ShellExecutionService.execute(
    command,
    workingDir,
    (event) => {
      switch (event.type) {
        case 'data':
          console.log(`[${event.stream}] ${event.chunk}`);
          break;
        case 'binary_detected':
          console.log('Binary content detected, switching to progress mode...');
          break;
        case 'binary_progress':
          console.log(`Downloaded: ${event.bytesReceived} bytes`);
          break;
      }
    },
    abortController.signal
  );

  console.log(`Started process with PID: ${handle.pid}`);

  try {
    const result = await handle.result;
    
    if (result.aborted) {
      console.log('Command was aborted by user');
      return '';
    }

    if (result.exitCode === 0) {
      console.log('Command completed successfully');
      return result.stdout;
    } else {
      console.error(`Command failed with exit code: ${result.exitCode}`);
      console.error('Error output:', result.stderr);
      return '';
    }
  } catch (error) {
    console.error('Command execution failed:', error);
    return '';
  }
}

// 使用示例
const output = await runCommand('ls -la', '/home/user');
console.log('Command output:', output);
```

### 带超时的命令执行

```typescript
async function runCommandWithTimeout(
  command: string,
  workingDir: string,
  timeoutMs: number = 30000
): Promise<ShellExecutionResult> {
  const abortController = new AbortController();
  
  // 设置超时
  const timeoutId = setTimeout(() => {
    console.log(`Command timeout after ${timeoutMs}ms, aborting...`);
    abortController.abort();
  }, timeoutMs);

  const handle = ShellExecutionService.execute(
    command,
    workingDir,
    (event) => {
      if (event.type === 'data') {
        process.stdout.write(event.chunk);
      }
    },
    abortController.signal
  );

  try {
    const result = await handle.result;
    clearTimeout(timeoutId);
    
    if (result.aborted) {
      throw new Error(`Command was aborted (timeout: ${timeoutMs}ms)`);
    }
    
    return result;
  } catch (error) {
    clearTimeout(timeoutId);
    throw error;
  }
}

// 使用示例
try {
  const result = await runCommandWithTimeout('npm install', '/project/path', 60000);
  console.log('Installation completed:', result.exitCode === 0 ? 'Success' : 'Failed');
} catch (error) {
  console.error('Installation error:', error.message);
}
```

### 交互式命令处理

```typescript
class InteractiveShellManager {
  private processes = new Map<string, ShellExecutionHandle>();
  private outputBuffers = new Map<string, string>();

  async startCommand(
    id: string,
    command: string,
    workingDir: string,
    onOutput?: (data: string, stream: 'stdout' | 'stderr') => void
  ): Promise<void> {
    const abortController = new AbortController();
    
    const handle = ShellExecutionService.execute(
      command,
      workingDir,
      (event) => {
        switch (event.type) {
          case 'data':
            const bufferId = `${id}_${event.stream}`;
            const currentBuffer = this.outputBuffers.get(bufferId) || '';
            this.outputBuffers.set(bufferId, currentBuffer + event.chunk);
            
            if (onOutput) {
              onOutput(event.chunk, event.stream);
            }
            break;
            
          case 'binary_detected':
            console.log(`[${id}] Binary output detected`);
            break;
            
          case 'binary_progress':
            console.log(`[${id}] Progress: ${event.bytesReceived} bytes`);
            break;
        }
      },
      abortController.signal
    );

    this.processes.set(id, handle);
    console.log(`Started command "${command}" with ID: ${id}, PID: ${handle.pid}`);

    // 异步等待结果
    handle.result.then(
      (result) => {
        console.log(`Command ${id} completed with exit code: ${result.exitCode}`);
        this.processes.delete(id);
      },
      (error) => {
        console.error(`Command ${id} failed:`, error);
        this.processes.delete(id);
      }
    );
  }

  async stopCommand(id: string): Promise<void> {
    const handle = this.processes.get(id);
    if (!handle) {
      throw new Error(`No command found with ID: ${id}`);
    }

    // 触发中止
    const abortController = new AbortController();
    abortController.abort();
    
    try {
      const result = await handle.result;
      console.log(`Command ${id} stopped, was aborted: ${result.aborted}`);
    } catch (error) {
      console.log(`Command ${id} stopped with error:`, error);
    }
  }

  getRunningCommands(): string[] {
    return Array.from(this.processes.keys());
  }

  getOutput(id: string, stream: 'stdout' | 'stderr' = 'stdout'): string {
    return this.outputBuffers.get(`${id}_${stream}`) || '';
  }

  async stopAllCommands(): Promise<void> {
    const stopPromises = Array.from(this.processes.keys()).map(id => 
      this.stopCommand(id).catch(err => 
        console.error(`Failed to stop command ${id}:`, err)
      )
    );
    
    await Promise.all(stopPromises);
  }
}

// 使用示例
const shellManager = new InteractiveShellManager();

// 启动多个命令
await shellManager.startCommand('build', 'npm run build', '/project');
await shellManager.startCommand('test', 'npm test -- --watch', '/project');
await shellManager.startCommand('serve', 'npm run serve', '/project');

// 查看运行中的命令
console.log('Running commands:', shellManager.getRunningCommands());

// 获取特定命令的输出
setTimeout(() => {
  console.log('Build output:', shellManager.getOutput('build'));
}, 5000);

// 停止特定命令
setTimeout(() => {
  shellManager.stopCommand('test');
}, 10000);

// 程序退出时清理
process.on('SIGINT', async () => {
  console.log('Stopping all commands...');
  await shellManager.stopAllCommands();
  process.exit(0);
});
```

## 高级功能

### 命令管道和链式执行

```typescript
class CommandPipeline {
  private commands: Array<{
    command: string;
    cwd: string;
    description?: string;
  }> = [];

  addCommand(command: string, cwd: string, description?: string): this {
    this.commands.push({ command, cwd, description });
    return this;
  }

  async execute(
    onProgress?: (step: number, total: number, description: string) => void
  ): Promise<ShellExecutionResult[]> {
    const results: ShellExecutionResult[] = [];
    
    for (let i = 0; i < this.commands.length; i++) {
      const { command, cwd, description = `Step ${i + 1}` } = this.commands[i];
      
      if (onProgress) {
        onProgress(i + 1, this.commands.length, description);
      }

      const abortController = new AbortController();
      const handle = ShellExecutionService.execute(
        command,
        cwd,
        (event) => {
          if (event.type === 'data') {
            console.log(`[${description}] ${event.chunk}`);
          }
        },
        abortController.signal
      );

      const result = await handle.result;
      results.push(result);

      // 如果命令失败，停止管道
      if (result.exitCode !== 0 && !result.aborted) {
        console.error(`Pipeline failed at step ${i + 1}: ${description}`);
        console.error('Error:', result.stderr);
        break;
      }
    }

    return results;
  }

  async executeParallel(
    maxConcurrent: number = 3
  ): Promise<ShellExecutionResult[]> {
    const results: ShellExecutionResult[] = new Array(this.commands.length);
    const executing: Promise<void>[] = [];

    for (let i = 0; i < this.commands.length; i++) {
      // 限制并发数
      if (executing.length >= maxConcurrent) {
        await Promise.race(executing);
      }

      const commandIndex = i;
      const { command, cwd, description = `Command ${i + 1}` } = this.commands[i];
      
      const executePromise = (async () => {
        try {
          const abortController = new AbortController();
          const handle = ShellExecutionService.execute(
            command,
            cwd,
            (event) => {
              if (event.type === 'data') {
                console.log(`[${description}] ${event.chunk}`);
              }
            },
            abortController.signal
          );

          results[commandIndex] = await handle.result;
        } catch (error) {
          console.error(`Command ${commandIndex + 1} failed:`, error);
        }
      })();

      executing.push(executePromise);
      
      // 移除完成的 Promise
      executePromise.finally(() => {
        const index = executing.indexOf(executePromise);
        if (index > -1) executing.splice(index, 1);
      });
    }

    // 等待所有命令完成
    await Promise.all(executing);
    return results;
  }
}

// 使用示例
const pipeline = new CommandPipeline()
  .addCommand('npm ci', '/project', 'Install dependencies')
  .addCommand('npm run build', '/project', 'Build application')
  .addCommand('npm test', '/project', 'Run tests')
  .addCommand('npm run lint', '/project', 'Lint code');

// 顺序执行
console.log('Running pipeline sequentially...');
const sequentialResults = await pipeline.execute((step, total, desc) => {
  console.log(`Progress: ${step}/${total} - ${desc}`);
});

// 并行执行
console.log('Running pipeline in parallel...');
const parallelResults = await pipeline.executeParallel(2);

console.log('Sequential results:', sequentialResults.map(r => r.exitCode));
console.log('Parallel results:', parallelResults.map(r => r.exitCode));
```

### 输出解析和结构化处理

```typescript
interface CommandResult<T = any> {
  success: boolean;
  data?: T;
  error?: string;
  rawResult: ShellExecutionResult;
}

class StructuredCommandExecutor {
  async executeJSON<T = any>(
    command: string,
    cwd: string,
    parser?: (output: string) => T
  ): Promise<CommandResult<T>> {
    const abortController = new AbortController();
    
    const handle = ShellExecutionService.execute(
      command,
      cwd,
      () => {}, // 静默处理，不输出到控制台
      abortController.signal
    );

    const result = await handle.result;

    if (result.exitCode !== 0) {
      return {
        success: false,
        error: result.stderr || 'Command failed',
        rawResult: result
      };
    }

    try {
      const data = parser ? parser(result.stdout) : JSON.parse(result.stdout);
      return {
        success: true,
        data,
        rawResult: result
      };
    } catch (parseError) {
      return {
        success: false,
        error: `Failed to parse output: ${parseError}`,
        rawResult: result
      };
    }
  }

  async executeWithPattern(
    command: string,
    cwd: string,
    patterns: { [key: string]: RegExp }
  ): Promise<CommandResult<{ [key: string]: string[] }>> {
    const abortController = new AbortController();
    const matches: { [key: string]: string[] } = {};
    
    Object.keys(patterns).forEach(key => {
      matches[key] = [];
    });

    const handle = ShellExecutionService.execute(
      command,
      cwd,
      (event) => {
        if (event.type === 'data') {
          // 实时模式匹配
          Object.entries(patterns).forEach(([key, pattern]) => {
            const found = event.chunk.match(pattern);
            if (found) {
              matches[key].push(...found);
            }
          });
        }
      },
      abortController.signal
    );

    const result = await handle.result;

    return {
      success: result.exitCode === 0,
      data: matches,
      error: result.exitCode !== 0 ? result.stderr : undefined,
      rawResult: result
    };
  }

  async executeWithProgress(
    command: string,
    cwd: string,
    progressPattern: RegExp,
    onProgress: (progress: number, message?: string) => void
  ): Promise<CommandResult> {
    const abortController = new AbortController();
    
    const handle = ShellExecutionService.execute(
      command,
      cwd,
      (event) => {
        if (event.type === 'data') {
          const match = event.chunk.match(progressPattern);
          if (match) {
            const progress = parseInt(match[1] || '0', 10);
            const message = match[2] || undefined;
            onProgress(progress, message);
          }
        }
      },
      abortController.signal
    );

    const result = await handle.result;

    return {
      success: result.exitCode === 0,
      error: result.exitCode !== 0 ? result.stderr : undefined,
      rawResult: result
    };
  }
}

// 使用示例
const executor = new StructuredCommandExecutor();

// JSON 输出解析
interface PackageInfo {
  name: string;
  version: string;
  dependencies: Record<string, string>;
}

const packageResult = await executor.executeJSON<PackageInfo>(
  'npm list --json --depth=0',
  '/project',
  (output) => {
    const parsed = JSON.parse(output);
    return {
      name: parsed.name,
      version: parsed.version,
      dependencies: parsed.dependencies || {}
    };
  }
);

if (packageResult.success) {
  console.log('Package info:', packageResult.data);
} else {
  console.error('Failed to get package info:', packageResult.error);
}

// 模式匹配解析
const testResult = await executor.executeWithPattern(
  'npm test',
  '/project',
  {
    passed: /(\d+) passing/g,
    failed: /(\d+) failing/g,
    testFiles: /\.test\.js/g
  }
);

if (testResult.success) {
  console.log('Test results:', testResult.data);
}

// 进度监控
await executor.executeWithProgress(
  'npm install',
  '/project',
  /(\d+)% - (.+)/,
  (progress, message) => {
    console.log(`Installation progress: ${progress}% - ${message || 'Installing...'}`);
  }
);
```

### 命令缓存和重试机制

```typescript
interface CachedCommandOptions {
  cacheKey?: string;
  cacheTTL?: number; // 缓存时间（毫秒）
  maxRetries?: number;
  retryDelay?: number;
  retryOn?: (result: ShellExecutionResult) => boolean;
}

class CachedShellExecutor {
  private cache = new Map<string, {
    result: ShellExecutionResult;
    timestamp: number;
    ttl: number;
  }>();

  async execute(
    command: string,
    cwd: string,
    options: CachedCommandOptions = {}
  ): Promise<ShellExecutionResult> {
    const {
      cacheKey = `${command}:${cwd}`,
      cacheTTL = 300000, // 5分钟默认缓存
      maxRetries = 3,
      retryDelay = 1000,
      retryOn = (result) => result.exitCode !== 0 && !result.aborted
    } = options;

    // 检查缓存
    const cached = this.cache.get(cacheKey);
    if (cached && Date.now() - cached.timestamp < cached.ttl) {
      console.log(`Using cached result for: ${command}`);
      return cached.result;
    }

    // 执行命令（带重试）
    let lastResult: ShellExecutionResult;
    let attempt = 1;

    while (attempt <= maxRetries) {
      try {
        console.log(`Executing command (attempt ${attempt}/${maxRetries}): ${command}`);
        
        const abortController = new AbortController();
        const handle = ShellExecutionService.execute(
          command,
          cwd,
          (event) => {
            if (event.type === 'data') {
              process.stdout.write(event.chunk);
            }
          },
          abortController.signal
        );

        lastResult = await handle.result;

        // 检查是否需要重试
        if (!retryOn(lastResult) || attempt === maxRetries) {
          break;
        }

        console.log(`Command failed, retrying in ${retryDelay}ms...`);
        await new Promise(resolve => setTimeout(resolve, retryDelay * attempt));
        attempt++;
        
      } catch (error) {
        if (attempt === maxRetries) {
          throw error;
        }
        
        console.log(`Command error, retrying in ${retryDelay}ms...`);
        await new Promise(resolve => setTimeout(resolve, retryDelay * attempt));
        attempt++;
      }
    }

    // 缓存成功的结果
    if (lastResult!.exitCode === 0) {
      this.cache.set(cacheKey, {
        result: lastResult!,
        timestamp: Date.now(),
        ttl: cacheTTL
      });
    }

    return lastResult!;
  }

  clearCache(pattern?: string): void {
    if (pattern) {
      const regex = new RegExp(pattern);
      for (const [key] of this.cache) {
        if (regex.test(key)) {
          this.cache.delete(key);
        }
      }
    } else {
      this.cache.clear();
    }
  }

  getCacheStats(): {
    size: number;
    keys: string[];
    oldestEntry: number;
    newestEntry: number;
  } {
    const entries = Array.from(this.cache.entries());
    const timestamps = entries.map(([, value]) => value.timestamp);
    
    return {
      size: this.cache.size,
      keys: entries.map(([key]) => key),
      oldestEntry: timestamps.length > 0 ? Math.min(...timestamps) : 0,
      newestEntry: timestamps.length > 0 ? Math.max(...timestamps) : 0
    };
  }
}

// 使用示例
const cachedExecutor = new CachedShellExecutor();

// 带缓存和重试的命令执行
const result = await cachedExecutor.execute(
  'npm test',
  '/project',
  {
    cacheKey: 'npm-test-project',
    cacheTTL: 600000, // 10分钟缓存
    maxRetries: 3,
    retryDelay: 2000,
    retryOn: (result) => {
      // 只对网络相关错误重试
      return result.stderr.includes('network') || 
             result.stderr.includes('timeout') ||
             result.exitCode === 1;
    }
  }
);

console.log('Test result:', result.exitCode === 0 ? 'Passed' : 'Failed');

// 查看缓存状态
const stats = cachedExecutor.getCacheStats();
console.log('Cache stats:', stats);

// 清理特定模式的缓存
cachedExecutor.clearCache('npm-test-.*');
```

## 错误处理

### 常见错误场景

```typescript
// 命令不存在
Error: Command not found: invalid-command

// 权限错误
Error: Permission denied: /protected/directory

// 工作目录不存在
Error: ENOENT: no such file or directory, chdir '/nonexistent'

// 进程启动失败
Error: spawn cmd.exe ENOENT

// 网络超时
Error: Command timeout after 30000ms

// 磁盘空间不足
Error: ENOSPC: no space left on device

// 内存不足
Error: Cannot allocate memory
```

### 错误恢复策略

```typescript
class RobustShellExecutor {
  private readonly DEFAULT_TIMEOUT = 30000;
  private readonly MAX_RETRIES = 3;
  private readonly RETRY_DELAY = 1000;

  async executeRobust(
    command: string,
    cwd: string,
    options: {
      timeout?: number;
      maxRetries?: number;
      retryDelay?: number;
      fallbackCommands?: string[];
      onError?: (error: Error, attempt: number) => void;
    } = {}
  ): Promise<ShellExecutionResult> {
    const {
      timeout = this.DEFAULT_TIMEOUT,
      maxRetries = this.MAX_RETRIES,
      retryDelay = this.RETRY_DELAY,
      fallbackCommands = [],
      onError
    } = options;

    const allCommands = [command, ...fallbackCommands];
    let lastError: Error | null = null;

    for (let cmdIndex = 0; cmdIndex < allCommands.length; cmdIndex++) {
      const currentCommand = allCommands[cmdIndex];
      
      for (let attempt = 1; attempt <= maxRetries; attempt++) {
        try {
          const result = await this.executeWithTimeout(
            currentCommand,
            cwd,
            timeout
          );

          // 成功执行
          if (result.exitCode === 0 || result.aborted) {
            return result;
          }

          // 命令执行失败，但不是错误（比如测试失败）
          if (attempt === maxRetries && cmdIndex === allCommands.length - 1) {
            return result;
          }

        } catch (error) {
          lastError = error instanceof Error ? error : new Error(String(error));
          
          if (onError) {
            onError(lastError, attempt);
          }

          // 特定错误处理
          if (this.isRetryableError(lastError)) {
            if (attempt < maxRetries) {
              console.warn(`Attempt ${attempt} failed, retrying in ${retryDelay}ms...`);
              await new Promise(resolve => setTimeout(resolve, retryDelay * attempt));
              continue;
            }
          } else {
            // 不可重试的错误，立即尝试下一个命令
            break;
          }
        }
      }
    }

    throw lastError || new Error('All command attempts failed');
  }

  private async executeWithTimeout(
    command: string,
    cwd: string,
    timeoutMs: number
  ): Promise<ShellExecutionResult> {
    const abortController = new AbortController();
    
    const timeoutId = setTimeout(() => {
      abortController.abort();
    }, timeoutMs);

    try {
      // 预检查
      await this.preExecutionCheck(command, cwd);

      const handle = ShellExecutionService.execute(
        command,
        cwd,
        (event) => {
          if (event.type === 'data') {
            process.stdout.write(event.chunk);
          }
        },
        abortController.signal
      );

      const result = await handle.result;
      clearTimeout(timeoutId);

      if (result.aborted) {
        throw new Error(`Command timeout after ${timeoutMs}ms`);
      }

      return result;
    } catch (error) {
      clearTimeout(timeoutId);
      throw this.enrichError(error, command, cwd);
    }
  }

  private async preExecutionCheck(command: string, cwd: string): Promise<void> {
    // 检查工作目录是否存在
    try {
      const fs = await import('fs/promises');
      const stat = await fs.stat(cwd);
      if (!stat.isDirectory()) {
        throw new Error(`Working directory is not a directory: ${cwd}`);
      }
    } catch (error) {
      throw new Error(`Working directory not accessible: ${cwd}`);
    }

    // 检查磁盘空间（Unix系统）
    if (process.platform !== 'win32') {
      try {
        const handle = ShellExecutionService.execute(
          'df -h .',
          cwd,
          () => {},
          new AbortController().signal
        );
        
        const result = await handle.result;
        if (result.stdout.includes('100%')) {
          console.warn('Warning: Disk space may be full');
        }
      } catch {
        // 忽略磁盘空间检查失败
      }
    }
  }

  private isRetryableError(error: Error): boolean {
    const retryablePatterns = [
      /network/i,
      /timeout/i,
      /ECONNRESET/i,
      /ECONNREFUSED/i,
      /ETIMEDOUT/i,
      /temporary failure/i,
      /service unavailable/i
    ];

    return retryablePatterns.some(pattern => pattern.test(error.message));
  }

  private enrichError(error: unknown, command: string, cwd: string): Error {
    const baseError = error instanceof Error ? error : new Error(String(error));
    
    // 添加上下文信息
    const contextInfo = {
      command,
      cwd,
      platform: process.platform,
      nodeVersion: process.version,
      timestamp: new Date().toISOString()
    };

    const enrichedMessage = `${baseError.message}\nContext: ${JSON.stringify(contextInfo, null, 2)}`;
    
    const enrichedError = new Error(enrichedMessage);
    enrichedError.stack = baseError.stack;
    
    return enrichedError;
  }

  // 健康检查
  async healthCheck(): Promise<{
    isHealthy: boolean;
    issues: string[];
    systemInfo: {
      platform: string;
      shell: string;
      diskSpace?: string;
      memory?: string;
    };
  }> {
    const issues: string[] = [];
    const systemInfo = {
      platform: process.platform,
      shell: process.platform === 'win32' ? 'cmd.exe' : 'bash',
    };

    try {
      // 测试基本命令执行
      const testCommand = process.platform === 'win32' ? 'echo test' : 'echo test';
      const handle = ShellExecutionService.execute(
        testCommand,
        process.cwd(),
        () => {},
        new AbortController().signal
      );
      
      const result = await handle.result;
      if (result.exitCode !== 0) {
        issues.push('Basic command execution failed');
      }
    } catch (error) {
      issues.push(`Shell execution service unavailable: ${error}`);
    }

    // 检查系统资源
    try {
      if (process.platform !== 'win32') {
        // Unix系统资源检查
        const dfHandle = ShellExecutionService.execute(
          'df -h /',
          '/',
          () => {},
          new AbortController().signal
        );
        const dfResult = await dfHandle.result;
        systemInfo.diskSpace = dfResult.stdout.split('\n')[1] || 'Unknown';

        const freeHandle = ShellExecutionService.execute(
          'free -h',
          '/',
          () => {},
          new AbortController().signal
        );
        const freeResult = await freeHandle.result;
        systemInfo.memory = freeResult.stdout.split('\n')[1] || 'Unknown';
      }
    } catch {
      // 忽略系统信息收集失败
    }

    return {
      isHealthy: issues.length === 0,
      issues,
      systemInfo: systemInfo as any
    };
  }
}

// 使用示例
const robustExecutor = new RobustShellExecutor();

// 健壮的命令执行
try {
  const result = await robustExecutor.executeRobust(
    'npm install',
    '/project',
    {
      timeout: 120000, // 2分钟超时
      maxRetries: 3,
      retryDelay: 5000,
      fallbackCommands: [
        'npm install --no-package-lock',
        'npm install --legacy-peer-deps',
        'yarn install'
      ],
      onError: (error, attempt) => {
        console.warn(`Attempt ${attempt} failed:`, error.message);
      }
    }
  );

  console.log('Installation result:', result.exitCode === 0 ? 'Success' : 'Failed');
} catch (error) {
  console.error('All installation attempts failed:', error.message);
}

// 系统健康检查
const health = await robustExecutor.healthCheck();
console.log('System health:', health);
```

## 性能优化

### 进程池管理

```typescript
class ShellProcessPool {
  private activeProcesses = new Map<string, ShellExecutionHandle>();
  private processQueue: Array<{
    id: string;
    command: string;
    cwd: string;
    onOutput: (event: ShellOutputEvent) => void;
    abortSignal: AbortSignal;
    resolve: (result: ShellExecutionResult) => void;
    reject: (error: Error) => void;
  }> = [];

  constructor(
    private maxConcurrentProcesses: number = 5,
    private processTimeout: number = 300000 // 5分钟默认超时
  ) {}

  async execute(
    command: string,
    cwd: string,
    onOutput: (event: ShellOutputEvent) => void = () => {},
    abortSignal?: AbortSignal
  ): Promise<ShellExecutionResult> {
    return new Promise((resolve, reject) => {
      const id = `${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
      const effectiveAbortSignal = abortSignal || new AbortController().signal;

      const queueItem = {
        id,
        command,
        cwd,
        onOutput,
        abortSignal: effectiveAbortSignal,
        resolve,
        reject
      };

      this.processQueue.push(queueItem);
      this.processNext();
    });
  }

  private processNext(): void {
    if (this.activeProcesses.size >= this.maxConcurrentProcesses || 
        this.processQueue.length === 0) {
      return;
    }

    const queueItem = this.processQueue.shift()!;
    this.executeProcess(queueItem);
  }

  private async executeProcess(queueItem: {
    id: string;
    command: string;
    cwd: string;
    onOutput: (event: ShellOutputEvent) => void;
    abortSignal: AbortSignal;
    resolve: (result: ShellExecutionResult) => void;
    reject: (error: Error) => void;
  }): Promise<void> {
    const { id, command, cwd, onOutput, abortSignal, resolve, reject } = queueItem;

    try {
      // 创建超时控制
      const timeoutController = new AbortController();
      const timeoutId = setTimeout(() => {
        timeoutController.abort();
      }, this.processTimeout);

      // 合并中止信号
      const combinedAbortController = new AbortController();
      const abortHandler = () => combinedAbortController.abort();
      
      abortSignal.addEventListener('abort', abortHandler, { once: true });
      timeoutController.signal.addEventListener('abort', abortHandler, { once: true });

      const handle = ShellExecutionService.execute(
        command,
        cwd,
        onOutput,
        combinedAbortController.signal
      );

      this.activeProcesses.set(id, handle);

      const result = await handle.result;
      clearTimeout(timeoutId);
      
      // 清理
      this.activeProcesses.delete(id);
      abortSignal.removeEventListener('abort', abortHandler);
      
      resolve(result);
      
    } catch (error) {
      this.activeProcesses.delete(id);
      reject(error instanceof Error ? error : new Error(String(error)));
    } finally {
      // 处理下一个排队的命令
      this.processNext();
    }
  }

  getPoolStats(): {
    active: number;
    queued: number;
    maxConcurrency: number;
    activeCommands: Array<{ id: string; pid?: number }>;
  } {
    return {
      active: this.activeProcesses.size,
      queued: this.processQueue.length,
      maxConcurrency: this.maxConcurrentProcesses,
      activeCommands: Array.from(this.activeProcesses.entries()).map(([id, handle]) => ({
        id,
        pid: handle.pid
      }))
    };
  }

  async abortAll(): Promise<void> {
    const abortPromises = Array.from(this.activeProcesses.entries()).map(
      async ([id, handle]) => {
        try {
          // 等待进程结束（已被中止）
          await handle.result;
        } catch {
          // 忽略中止造成的错误
        }
      }
    );

    // 清空队列
    this.processQueue.length = 0;
    
    await Promise.all(abortPromises);
    this.activeProcesses.clear();
  }

  setMaxConcurrency(max: number): void {
    this.maxConcurrentProcesses = max;
    // 如果增加了并发数，尝试处理更多命令
    while (this.activeProcesses.size < this.maxConcurrentProcesses && 
           this.processQueue.length > 0) {
      this.processNext();
    }
  }
}

// 使用示例
const processPool = new ShellProcessPool(3, 60000); // 最多3个并发，1分钟超时

// 批量执行命令
const commands = [
  { cmd: 'npm run build', cwd: '/project1' },
  { cmd: 'npm test', cwd: '/project1' },
  { cmd: 'npm run lint', cwd: '/project1' },
  { cmd: 'npm run build', cwd: '/project2' },
  { cmd: 'npm test', cwd: '/project2' },
];

const executePromises = commands.map((command, index) => 
  processPool.execute(
    command.cmd,
    command.cwd,
    (event) => {
      if (event.type === 'data') {
        console.log(`[Command ${index + 1}] ${event.chunk}`);
      }
    }
  )
);

// 监控进程池状态
const statsInterval = setInterval(() => {
  const stats = processPool.getPoolStats();
  console.log(`Pool stats: ${stats.active} active, ${stats.queued} queued`);
  
  if (stats.active === 0 && stats.queued === 0) {
    clearInterval(statsInterval);
  }
}, 1000);

try {
  const results = await Promise.all(executePromises);
  console.log('All commands completed');
  results.forEach((result, index) => {
    console.log(`Command ${index + 1} exit code: ${result.exitCode}`);
  });
} catch (error) {
  console.error('Some commands failed:', error);
} finally {
  clearInterval(statsInterval);
}

// 程序退出时清理
process.on('SIGINT', async () => {
  console.log('Aborting all processes...');
  await processPool.abortAll();
  process.exit(0);
});
```

## 最佳实践

### 命令构建和安全性

```typescript
// ✅ 推荐：安全的命令构建
class SafeCommandBuilder {
  private command: string[] = [];
  private workingDir: string = process.cwd();
  private environment: Record<string, string> = {};

  static create(): SafeCommandBuilder {
    return new SafeCommandBuilder();
  }

  setCommand(cmd: string): this {
    // 验证命令中不包含危险字符
    if (cmd.includes(';') || cmd.includes('&&') || cmd.includes('||')) {
      throw new Error('Command chaining not allowed for security');
    }
    this.command = [cmd];
    return this;
  }

  addArgument(arg: string): this {
    // 自动转义参数
    const escaped = this.escapeArgument(arg);
    this.command.push(escaped);
    return this;
  }

  addFlag(flag: string, value?: string): this {
    this.command.push(flag);
    if (value !== undefined) {
      this.command.push(this.escapeArgument(value));
    }
    return this;
  }

  setWorkingDirectory(dir: string): this {
    // 验证目录路径
    if (!path.isAbsolute(dir)) {
      throw new Error('Working directory must be absolute path');
    }
    this.workingDir = dir;
    return this;
  }

  setEnvironment(env: Record<string, string>): this {
    // 过滤敏感环境变量
    const filtered = Object.fromEntries(
      Object.entries(env).filter(([key]) => 
        !key.toLowerCase().includes('password') &&
        !key.toLowerCase().includes('secret') &&
        !key.toLowerCase().includes('token')
      )
    );
    this.environment = filtered;
    return this;
  }

  private escapeArgument(arg: string): string {
    if (process.platform === 'win32') {
      // Windows 转义
      return `"${arg.replace(/"/g, '""')}"`;
    } else {
      // Unix 转义
      return `'${arg.replace(/'/g, "'\"'\"'")}'`;
    }
  }

  build(): string {
    if (this.command.length === 0) {
      throw new Error('No command specified');
    }
    return this.command.join(' ');
  }

  async execute(
    onOutput?: (event: ShellOutputEvent) => void
  ): Promise<ShellExecutionResult> {
    const command = this.build();
    const abortController = new AbortController();

    const handle = ShellExecutionService.execute(
      command,
      this.workingDir,
      onOutput || (() => {}),
      abortController.signal
    );

    return handle.result;
  }
}

// 使用示例
const result = await SafeCommandBuilder.create()
  .setCommand('npm')
  .addArgument('install')
  .addArgument('@types/node')
  .addFlag('--save-dev')
  .addFlag('--registry', 'https://registry.npmjs.org/')
  .setWorkingDirectory('/project')
  .setEnvironment({ NODE_ENV: 'development' })
  .execute((event) => {
    if (event.type === 'data') {
      console.log(event.chunk);
    }
  });

// ❌ 避免：直接字符串拼接
// const dangerousCommand = `rm -rf ${userInput}`; // 危险！

// ✅ 推荐：使用构建器
const safeCleanup = await SafeCommandBuilder.create()
  .setCommand('rm')
  .addFlag('-rf')
  .addArgument(path.join('/tmp', 'safe-directory'))
  .execute();
```

### 资源管理和监控

```typescript
// ✅ 推荐：完整的资源管理
class ManagedShellExecutor {
  private activeProcesses = new Set<number>();
  private resourceLimits = {
    maxMemoryMB: 512,
    maxConcurrentProcesses: 10,
    maxExecutionTimeMs: 300000
  };

  async executeManaged(
    command: string,
    cwd: string,
    options: {
      memoryLimit?: number;
      timeLimit?: number;
      priority?: 'low' | 'normal' | 'high';
    } = {}
  ): Promise<ShellExecutionResult> {
    // 检查资源限制
    if (this.activeProcesses.size >= this.resourceLimits.maxConcurrentProcesses) {
      throw new Error('Too many concurrent processes');
    }

    const abortController = new AbortController();
    let processId: number | undefined;

    try {
      // 设置超时
      const timeLimit = options.timeLimit || this.resourceLimits.maxExecutionTimeMs;
      const timeoutId = setTimeout(() => {
        abortController.abort();
      }, timeLimit);

      const handle = ShellExecutionService.execute(
        command,
        cwd,
        (event) => {
          if (event.type === 'data') {
            // 可以在这里添加内存监控
            this.monitorMemoryUsage(processId);
          }
        },
        abortController.signal
      );

      processId = handle.pid;
      if (processId) {
        this.activeProcesses.add(processId);
      }

      const result = await handle.result;
      clearTimeout(timeoutId);

      return result;

    } finally {
      if (processId) {
        this.activeProcesses.delete(processId);
      }
    }
  }

  private monitorMemoryUsage(pid?: number): void {
    if (!pid) return;

    // 在生产环境中，这里可以实现真正的内存监控
    // 例如使用 ps 命令或系统API
  }

  getResourceStats(): {
    activeProcesses: number;
    maxConcurrentProcesses: number;
    memoryLimitMB: number;
  } {
    return {
      activeProcesses: this.activeProcesses.size,
      maxConcurrentProcesses: this.resourceLimits.maxConcurrentProcesses,
      memoryLimitMB: this.resourceLimits.maxMemoryMB
    };
  }

  async cleanup(): Promise<void> {
    // 清理所有活动进程
    this.activeProcesses.clear();
  }
}
```

### 日志和调试

```typescript
// ✅ 推荐：结构化日志
interface ExecutionLog {
  id: string;
  command: string;
  cwd: string;
  startTime: Date;
  endTime?: Date;
  duration?: number;
  exitCode?: number;
  pid?: number;
  outputLength: number;
  errorLength: number;
}

class LoggingShellExecutor {
  private executionLogs: ExecutionLog[] = [];
  private maxLogEntries = 1000;

  async executeWithLogging(
    command: string,
    cwd: string,
    onOutput?: (event: ShellOutputEvent) => void
  ): Promise<ShellExecutionResult> {
    const id = `exec_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
    const log: ExecutionLog = {
      id,
      command,
      cwd,
      startTime: new Date(),
      outputLength: 0,
      errorLength: 0
    };

    this.executionLogs.push(log);
    this.trimLogs();

    console.log(`[${id}] Starting command: ${command}`);
    console.log(`[${id}] Working directory: ${cwd}`);

    const abortController = new AbortController();

    try {
      const handle = ShellExecutionService.execute(
        command,
        cwd,
        (event) => {
          if (event.type === 'data') {
            log.outputLength += event.chunk.length;
            if (event.stream === 'stderr') {
              log.errorLength += event.chunk.length;
            }
          }
          
          if (onOutput) {
            onOutput(event);
          }
        },
        abortController.signal
      );

      log.pid = handle.pid;
      console.log(`[${id}] Process started with PID: ${handle.pid}`);

      const result = await handle.result;
      
      log.endTime = new Date();
      log.duration = log.endTime.getTime() - log.startTime.getTime();
      log.exitCode = result.exitCode;

      console.log(`[${id}] Command completed in ${log.duration}ms with exit code: ${result.exitCode}`);
      console.log(`[${id}] Output: ${log.outputLength} chars, Errors: ${log.errorLength} chars`);

      return result;

    } catch (error) {
      log.endTime = new Date();
      log.duration = log.endTime.getTime() - log.startTime.getTime();
      
      console.error(`[${id}] Command failed after ${log.duration}ms:`, error);
      throw error;
    }
  }

  private trimLogs(): void {
    if (this.executionLogs.length > this.maxLogEntries) {
      this.executionLogs = this.executionLogs.slice(-this.maxLogEntries);
    }
  }

  getExecutionHistory(): ExecutionLog[] {
    return [...this.executionLogs].reverse(); // 最新的在前
  }

  exportLogs(format: 'json' | 'csv' = 'json'): string {
    if (format === 'csv') {
      const headers = ['id', 'command', 'cwd', 'startTime', 'duration', 'exitCode', 'outputLength', 'errorLength'];
      const rows = this.executionLogs.map(log => [
        log.id,
        `"${log.command.replace(/"/g, '""')}"`,
        log.cwd,
        log.startTime.toISOString(),
        log.duration || 0,
        log.exitCode || 'N/A',
        log.outputLength,
        log.errorLength
      ]);
      
      return [headers.join(','), ...rows.map(row => row.join(','))].join('\n');
    }
    
    return JSON.stringify(this.executionLogs, null, 2);
  }
}

// 使用示例
const loggingExecutor = new LoggingShellExecutor();

// 执行命令并记录日志
const result = await loggingExecutor.executeWithLogging(
  'npm run build',
  '/project',
  (event) => {
    if (event.type === 'data') {
      process.stdout.write(event.chunk);
    }
  }
);

// 查看执行历史
const history = loggingExecutor.getExecutionHistory();
console.log('Recent executions:', history.slice(0, 5));

// 导出日志
const csvLogs = loggingExecutor.exportLogs('csv');
console.log('CSV logs:', csvLogs);
```