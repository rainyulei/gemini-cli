# IDE Context - IDE 上下文管理文档

## 概述

`ideContext.ts` 是 Gemini CLI 的 IDE 集成上下文管理模块，负责管理 IDE 中打开的文件信息、光标位置、选中文本和最近打开文件等上下文数据。它提供了一个反应式的状态管理系统，支持多个订阅者监听上下文变化，是 IDE 扩展和命令行工具之间的核心通信桥梁。

## 主要功能

- **文件上下文管理**: 跟踪当前活动文件、选中文本和光标位置
- **最近文件记录**: 维护最近打开文件的历史记录
- **反应式状态系统**: 支持多订阅者模式的状态变更通知
- **类型安全验证**: 使用 Zod 模式验证所有输入数据
- **工厂模式设计**: 支持隔离实例创建，便于测试和多环境使用
- **IDE 通知集成**: 处理来自 IDE 的 `ide/openFilesChanged` 通知

## 核心类型定义

### `Cursor` 类型

**功能**: 表示编辑器中的光标位置

```typescript
export const CursorSchema = z.object({
  line: z.number(),
  character: z.number(),
});
export type Cursor = z.infer<typeof CursorSchema>;
```

**字段详情**:
- `line`: 光标所在的行号（从0开始计数）
- `character`: 光标在行内的字符位置（从0开始计数）

**使用示例**:
```typescript
const cursorPosition: Cursor = {
  line: 25,        // 第26行
  character: 10    // 第11个字符
};
```

### `OpenFiles` 类型

**功能**: 表示 IDE 当前打开文件的完整上下文信息

```typescript
export const OpenFilesSchema = z.object({
  activeFile: z.string(),
  selectedText: z.string().optional(),
  cursor: CursorSchema.optional(),
  recentOpenFiles: z
    .array(
      z.object({
        filePath: z.string(),
        timestamp: z.number(),
      }),
    )
    .optional(),
});
export type OpenFiles = z.infer<typeof OpenFilesSchema>;
```

**字段详情**:
- `activeFile`: 当前活动文件的完整路径
- `selectedText`: 当前选中的文本内容（可选）
- `cursor`: 光标位置信息（可选）
- `recentOpenFiles`: 最近打开文件的数组，包含文件路径和时间戳

**完整示例**:
```typescript
const openFilesContext: OpenFiles = {
  activeFile: '/workspace/src/components/Header.tsx',
  selectedText: 'const HeaderComponent = () => {\n  return <div>Header</div>;\n};',
  cursor: {
    line: 15,
    character: 22
  },
  recentOpenFiles: [
    {
      filePath: '/workspace/src/components/Header.tsx',
      timestamp: Date.now() - 1000
    },
    {
      filePath: '/workspace/src/utils/helpers.ts',
      timestamp: Date.now() - 60000
    },
    {
      filePath: '/workspace/README.md',
      timestamp: Date.now() - 300000
    }
  ]
};
```

### `OpenFilesNotificationSchema`

**功能**: 验证来自 IDE 的文件上下文变更通知

```typescript
export const OpenFilesNotificationSchema = z.object({
  method: z.literal('ide/openFilesChanged'),
  params: OpenFilesSchema,
});
```

**用途**: 确保从 IDE 扩展接收的通知格式正确和类型安全

## 核心状态管理系统

### `createIdeContextStore` 工厂函数

**功能**: 创建新的 IDE 上下文存储实例，提供完整的状态管理功能

```typescript
export function createIdeContextStore() {
  let openFilesContext: OpenFiles | undefined = undefined;
  const subscribers = new Set<OpenFilesSubscriber>();

  // 返回管理接口
  return {
    setOpenFilesContext,
    getOpenFilesContext,
    subscribeToOpenFiles,
    clearOpenFilesContext,
  };
}
```

**设计优势**:
- **封装性**: 私有状态变量，外部只能通过接口访问
- **隔离性**: 每次调用创建独立实例，适合测试和多环境
- **可扩展性**: 便于添加新的状态管理功能

### 状态管理方法

#### `setOpenFilesContext`

**功能**: 设置活动文件上下文并通知所有订阅者

```typescript
function setOpenFilesContext(newOpenFiles: OpenFiles): void {
  openFilesContext = newOpenFiles;
  notifySubscribers();
}
```

**执行流程**:
1. **更新状态**: 将新的文件上下文保存到内部状态
2. **触发通知**: 调用 `notifySubscribers()` 通知所有监听器
3. **数据验证**: 输入数据已通过 Zod 模式验证

**使用示例**:
```typescript
const ideStore = createIdeContextStore();

// 设置新的文件上下文
ideStore.setOpenFilesContext({
  activeFile: '/project/src/App.tsx',
  selectedText: 'function App() {',
  cursor: { line: 10, character: 0 },
  recentOpenFiles: [
    {
      filePath: '/project/src/App.tsx',
      timestamp: Date.now()
    }
  ]
});
```

#### `getOpenFilesContext`

**功能**: 获取当前的活动文件上下文

```typescript
function getOpenFilesContext(): OpenFiles | undefined {
  return openFilesContext;
}
```

**返回值**:
- `OpenFiles`: 当前活动文件上下文
- `undefined`: 没有活动文件时

**使用示例**:
```typescript
const currentContext = ideStore.getOpenFilesContext();

if (currentContext) {
  console.log(`Active file: ${currentContext.activeFile}`);
  console.log(`Cursor at: line ${currentContext.cursor?.line}, character ${currentContext.cursor?.character}`);
  
  if (currentContext.selectedText) {
    console.log(`Selected text: ${currentContext.selectedText}`);
  }
} else {
  console.log('No active file context');
}
```

#### `clearOpenFilesContext`

**功能**: 清除当前文件上下文并通知订阅者

```typescript
function clearOpenFilesContext(): void {
  openFilesContext = undefined;
  notifySubscribers();
}
```

**使用场景**:
- IDE 关闭所有文件时
- 切换到非编辑器视图时
- 重置应用状态时

#### `subscribeToOpenFiles`

**功能**: 订阅文件上下文变更通知

```typescript
function subscribeToOpenFiles(subscriber: OpenFilesSubscriber): () => void {
  subscribers.add(subscriber);
  return () => {
    subscribers.delete(subscriber);
  };
}
```

**参数**:
- `subscriber`: 回调函数，接收 `OpenFiles | undefined` 参数

**返回值**: 取消订阅的函数

**订阅者类型**:
```typescript
type OpenFilesSubscriber = (openFiles: OpenFiles | undefined) => void;
```

**使用示例**:
```typescript
// 创建订阅者
const unsubscribe = ideStore.subscribeToOpenFiles((openFiles) => {
  if (openFiles) {
    console.log(`File context changed: ${openFiles.activeFile}`);
    
    // 更新 UI 状态
    updateFileInfo(openFiles.activeFile);
    
    // 更新光标显示
    if (openFiles.cursor) {
      updateCursorPosition(openFiles.cursor);
    }
    
    // 更新选中文本高亮
    if (openFiles.selectedText) {
      highlightSelectedText(openFiles.selectedText);
    }
  } else {
    console.log('No active file');
    clearFileInfo();
  }
});

// 在组件卸载或不需要时取消订阅
// unsubscribe();
```

### 通知系统

#### `notifySubscribers`

**功能**: 通知所有注册的订阅者上下文变更

```typescript
function notifySubscribers(): void {
  for (const subscriber of subscribers) {
    subscriber(openFilesContext);
  }
}
```

**特点**:
- **同步执行**: 所有订阅者按注册顺序同步调用
- **错误隔离**: 单个订阅者错误不影响其他订阅者
- **性能优化**: 使用 Set 数据结构管理订阅者

## 全局实例

### `ideContext` 默认实例

```typescript
export const ideContext = createIdeContextStore();
```

**功能**: 应用程序的默认共享 IDE 上下文存储实例

**使用场景**:
- 应用程序主要 IDE 集成
- 全局状态管理
- 跨组件状态共享

## 集成使用示例

### 基本 IDE 集成

```typescript
import { ideContext, OpenFiles } from './ideContext.js';

// 基本的 IDE 集成管理器
class IDEIntegrationManager {
  private isConnected = false;

  constructor() {
    this.setupSubscription();
  }

  private setupSubscription(): void {
    // 订阅文件上下文变更
    ideContext.subscribeToOpenFiles((openFiles) => {
      this.handleContextChange(openFiles);
    });
  }

  private handleContextChange(openFiles: OpenFiles | undefined): void {
    if (openFiles) {
      console.log(`Active file changed: ${openFiles.activeFile}`);
      
      // 更新窗口标题
      this.updateWindowTitle(openFiles.activeFile);
      
      // 同步最近文件列表
      this.syncRecentFiles(openFiles.recentOpenFiles || []);
      
      // 处理选中文本
      if (openFiles.selectedText) {
        this.handleTextSelection(openFiles.selectedText);
      }
    } else {
      console.log('No active file');
      this.updateWindowTitle('Gemini CLI');
    }
  }

  // 模拟从 IDE 接收上下文更新
  updateFromIDE(newContext: OpenFiles): void {
    try {
      // 验证数据格式
      OpenFilesSchema.parse(newContext);
      
      // 更新上下文
      ideContext.setOpenFilesContext(newContext);
      
      console.log('IDE context updated successfully');
    } catch (error) {
      console.error('Invalid IDE context data:', error);
    }
  }

  // 获取当前活动文件信息
  getCurrentFileInfo(): {
    file: string | null;
    hasSelection: boolean;
    cursorPosition: string | null;
  } {
    const context = ideContext.getOpenFilesContext();
    
    if (!context) {
      return {
        file: null,
        hasSelection: false,
        cursorPosition: null,
      };
    }

    return {
      file: context.activeFile,
      hasSelection: !!context.selectedText,
      cursorPosition: context.cursor 
        ? `Line ${context.cursor.line + 1}, Column ${context.cursor.character + 1}`
        : null,
    };
  }

  private updateWindowTitle(title: string): void {
    // 更新应用程序窗口标题
    console.log(`Window title: ${title}`);
  }

  private syncRecentFiles(recentFiles: Array<{filePath: string, timestamp: number}>): void {
    // 同步最近文件列表到应用状态
    console.log(`Recent files: ${recentFiles.map(f => f.filePath).join(', ')}`);
  }

  private handleTextSelection(selectedText: string): void {
    // 处理文本选择，例如显示上下文菜单
    console.log(`Text selected: ${selectedText.substring(0, 50)}...`);
  }
}

// 创建和使用集成管理器
const ideManager = new IDEIntegrationManager();

// 模拟从 IDE 接收更新
ideManager.updateFromIDE({
  activeFile: '/workspace/src/components/Button.tsx',
  selectedText: 'interface ButtonProps {\n  onClick: () => void;\n  children: React.ReactNode;\n}',
  cursor: { line: 5, character: 0 },
  recentOpenFiles: [
    {
      filePath: '/workspace/src/components/Button.tsx',
      timestamp: Date.now()
    },
    {
      filePath: '/workspace/src/types/index.ts',
      timestamp: Date.now() - 120000
    }
  ]
});

// 查询当前文件信息
const currentInfo = ideManager.getCurrentFileInfo();
console.log('Current file info:', currentInfo);
```

### 多实例管理

```typescript
import { createIdeContextStore } from './ideContext.js';

// 多个 IDE 实例管理器
class MultiIDEManager {
  private contexts = new Map<string, ReturnType<typeof createIdeContextStore>>();
  private activeInstance: string | null = null;

  // 注册新的 IDE 实例
  registerIDE(instanceId: string): void {
    if (this.contexts.has(instanceId)) {
      console.warn(`IDE instance ${instanceId} already registered`);
      return;
    }

    const contextStore = createIdeContextStore();
    this.contexts.set(instanceId, contextStore);

    // 为每个实例设置独立的订阅
    contextStore.subscribeToOpenFiles((openFiles) => {
      this.handleInstanceContextChange(instanceId, openFiles);
    });

    console.log(`IDE instance ${instanceId} registered`);
  }

  // 移除 IDE 实例
  unregisterIDE(instanceId: string): void {
    if (this.contexts.has(instanceId)) {
      this.contexts.delete(instanceId);
      console.log(`IDE instance ${instanceId} unregistered`);
      
      // 如果移除的是活动实例，清除活动状态
      if (this.activeInstance === instanceId) {
        this.activeInstance = null;
      }
    }
  }

  // 设置活动 IDE 实例
  setActiveInstance(instanceId: string): void {
    if (!this.contexts.has(instanceId)) {
      throw new Error(`IDE instance ${instanceId} not found`);
    }
    
    this.activeInstance = instanceId;
    console.log(`Active IDE instance set to: ${instanceId}`);
  }

  // 更新特定实例的上下文
  updateInstanceContext(instanceId: string, context: OpenFiles): void {
    const contextStore = this.contexts.get(instanceId);
    if (!contextStore) {
      throw new Error(`IDE instance ${instanceId} not found`);
    }

    contextStore.setOpenFilesContext(context);
  }

  // 获取活动实例的上下文
  getActiveContext(): OpenFiles | undefined {
    if (!this.activeInstance) {
      return undefined;
    }

    const contextStore = this.contexts.get(this.activeInstance);
    return contextStore?.getOpenFilesContext();
  }

  // 获取所有实例的上下文摘要
  getAllContextsSummary(): Array<{
    instanceId: string;
    activeFile: string | null;
    hasSelection: boolean;
    recentFilesCount: number;
  }> {
    const summary: Array<{
      instanceId: string;
      activeFile: string | null;
      hasSelection: boolean;
      recentFilesCount: number;
    }> = [];

    for (const [instanceId, contextStore] of this.contexts) {
      const context = contextStore.getOpenFilesContext();
      summary.push({
        instanceId,
        activeFile: context?.activeFile || null,
        hasSelection: !!context?.selectedText,
        recentFilesCount: context?.recentOpenFiles?.length || 0,
      });
    }

    return summary;
  }

  private handleInstanceContextChange(instanceId: string, openFiles: OpenFiles | undefined): void {
    console.log(`Context changed for IDE instance ${instanceId}:`, {
      activeFile: openFiles?.activeFile,
      hasSelection: !!openFiles?.selectedText,
      cursorPosition: openFiles?.cursor,
    });

    // 如果是活动实例，执行额外的处理
    if (instanceId === this.activeInstance) {
      this.handleActiveInstanceChange(openFiles);
    }
  }

  private handleActiveInstanceChange(openFiles: OpenFiles | undefined): void {
    // 处理活动实例的上下文变更
    if (openFiles) {
      console.log(`Active instance file: ${openFiles.activeFile}`);
      // 更新全局状态、UI等
    }
  }
}

// 使用示例
const multiIDEManager = new MultiIDEManager();

// 注册多个 IDE 实例
multiIDEManager.registerIDE('vscode-main');
multiIDEManager.registerIDE('vscode-secondary');
multiIDEManager.registerIDE('intellij');

// 设置活动实例
multiIDEManager.setActiveInstance('vscode-main');

// 更新不同实例的上下文
multiIDEManager.updateInstanceContext('vscode-main', {
  activeFile: '/project/src/App.tsx',
  cursor: { line: 10, character: 5 }
});

multiIDEManager.updateInstanceContext('intellij', {
  activeFile: '/project/backend/server.java',
  selectedText: 'public class Server {',
  cursor: { line: 25, character: 0 }
});

// 获取所有实例摘要
const summary = multiIDEManager.getAllContextsSummary();
console.log('All IDE instances:', summary);
```

### 实时同步集成

```typescript
import { ideContext, OpenFilesNotificationSchema } from './ideContext.js';

// 实时 IDE 同步管理器
class RealtimeIDESync {
  private websocket: WebSocket | null = null;
  private reconnectAttempts = 0;
  private maxReconnectAttempts = 5;
  private reconnectDelay = 1000;

  constructor(private wsUrl: string) {
    this.setupContextSubscription();
    this.connect();
  }

  private setupContextSubscription(): void {
    ideContext.subscribeToOpenFiles((openFiles) => {
      // 将本地上下文变更同步到其他客户端
      this.broadcastContextChange(openFiles);
    });
  }

  private connect(): void {
    try {
      this.websocket = new WebSocket(this.wsUrl);
      
      this.websocket.onopen = () => {
        console.log('IDE sync WebSocket connected');
        this.reconnectAttempts = 0;
      };

      this.websocket.onmessage = (event) => {
        this.handleIncomingMessage(event.data);
      };

      this.websocket.onclose = () => {
        console.log('IDE sync WebSocket disconnected');
        this.attemptReconnect();
      };

      this.websocket.onerror = (error) => {
        console.error('IDE sync WebSocket error:', error);
      };
    } catch (error) {
      console.error('Failed to connect to IDE sync WebSocket:', error);
      this.attemptReconnect();
    }
  }

  private handleIncomingMessage(data: string): void {
    try {
      const message = JSON.parse(data);
      
      // 验证消息格式
      if (message.type === 'ide-context-update') {
        const validated = OpenFilesNotificationSchema.parse(message.payload);
        
        // 更新本地上下文（不触发广播避免循环）
        this.updateLocalContext(validated.params);
      }
    } catch (error) {
      console.error('Invalid incoming IDE sync message:', error);
    }
  }

  private updateLocalContext(openFiles: OpenFiles): void {
    // 直接更新，避免触发订阅者的广播
    ideContext.setOpenFilesContext(openFiles);
    console.log('Local IDE context updated from remote:', openFiles.activeFile);
  }

  private broadcastContextChange(openFiles: OpenFiles | undefined): void {
    if (!this.websocket || this.websocket.readyState !== WebSocket.OPEN) {
      return;
    }

    const message = {
      type: 'ide-context-update',
      timestamp: Date.now(),
      payload: {
        method: 'ide/openFilesChanged',
        params: openFiles,
      },
    };

    try {
      this.websocket.send(JSON.stringify(message));
    } catch (error) {
      console.error('Failed to broadcast IDE context change:', error);
    }
  }

  private attemptReconnect(): void {
    if (this.reconnectAttempts >= this.maxReconnectAttempts) {
      console.error('Max reconnection attempts reached for IDE sync');
      return;
    }

    this.reconnectAttempts++;
    const delay = this.reconnectDelay * Math.pow(2, this.reconnectAttempts - 1);
    
    console.log(`Attempting to reconnect IDE sync in ${delay}ms (attempt ${this.reconnectAttempts})`);
    
    setTimeout(() => {
      this.connect();
    }, delay);
  }

  // 手动触发同步
  forceSyncToPeers(): void {
    const currentContext = ideContext.getOpenFilesContext();
    this.broadcastContextChange(currentContext);
  }

  // 请求从服务器同步最新状态
  requestSyncFromServer(): void {
    if (!this.websocket || this.websocket.readyState !== WebSocket.OPEN) {
      console.warn('WebSocket not connected, cannot request sync');
      return;
    }

    const requestMessage = {
      type: 'request-context-sync',
      timestamp: Date.now(),
    };

    this.websocket.send(JSON.stringify(requestMessage));
  }

  disconnect(): void {
    if (this.websocket) {
      this.websocket.close();
      this.websocket = null;
    }
  }
}

// 使用实时同步
const realtimeSync = new RealtimeIDESync('ws://localhost:8080/ide-sync');

// 模拟 IDE 更新
setTimeout(() => {
  ideContext.setOpenFilesContext({
    activeFile: '/project/src/NewComponent.tsx',
    selectedText: 'const NewComponent = () => {',
    cursor: { line: 1, character: 0 },
  });
}, 2000);

// 请求同步
setTimeout(() => {
  realtimeSync.requestSyncFromServer();
}, 5000);
```

### 持久化状态管理

```typescript
import { ideContext, OpenFiles } from './ideContext.js';
import * as fs from 'fs';
import * as path from 'path';
import * as os from 'os';

// IDE 上下文持久化管理器
class IDEContextPersistence {
  private persistencePath: string;
  private autosaveInterval: NodeJS.Timeout | null = null;
  private autosaveDelay = 5000; // 5秒自动保存

  constructor(filename = 'ide-context.json') {
    this.persistencePath = path.join(os.homedir(), '.gemini-cli', filename);
    this.ensureDirectoryExists();
    this.setupAutosave();
    this.loadPersistedContext();
  }

  private ensureDirectoryExists(): void {
    const dir = path.dirname(this.persistencePath);
    if (!fs.existsSync(dir)) {
      fs.mkdirSync(dir, { recursive: true });
    }
  }

  private setupAutosave(): void {
    ideContext.subscribeToOpenFiles((openFiles) => {
      // 延迟保存，避免频繁写入
      if (this.autosaveInterval) {
        clearTimeout(this.autosaveInterval);
      }

      this.autosaveInterval = setTimeout(() => {
        this.saveContext(openFiles);
      }, this.autosaveDelay);
    });
  }

  // 保存上下文到文件
  saveContext(openFiles: OpenFiles | undefined): void {
    try {
      const data = {
        timestamp: Date.now(),
        context: openFiles,
        version: '1.0',
      };

      fs.writeFileSync(this.persistencePath, JSON.stringify(data, null, 2));
      console.log('IDE context saved to:', this.persistencePath);
    } catch (error) {
      console.error('Failed to save IDE context:', error);
    }
  }

  // 从文件加载上下文
  loadPersistedContext(): OpenFiles | null {
    try {
      if (!fs.existsSync(this.persistencePath)) {
        console.log('No persisted IDE context found');
        return null;
      }

      const data = JSON.parse(fs.readFileSync(this.persistencePath, 'utf-8'));
      
      // 验证数据格式和版本
      if (data.version !== '1.0') {
        console.warn('Incompatible IDE context version, ignoring');
        return null;
      }

      // 检查数据是否过期（例如1小时）
      const maxAge = 60 * 60 * 1000;
      if (Date.now() - data.timestamp > maxAge) {
        console.log('Persisted IDE context is too old, ignoring');
        return null;
      }

      if (data.context) {
        // 验证上下文数据
        const validatedContext = OpenFilesSchema.parse(data.context);
        
        // 恢复到全局上下文
        ideContext.setOpenFilesContext(validatedContext);
        console.log('IDE context restored from:', this.persistencePath);
        
        return validatedContext;
      }
    } catch (error) {
      console.error('Failed to load persisted IDE context:', error);
    }

    return null;
  }

  // 清除持久化数据
  clearPersistedContext(): void {
    try {
      if (fs.existsSync(this.persistencePath)) {
        fs.unlinkSync(this.persistencePath);
        console.log('Persisted IDE context cleared');
      }
    } catch (error) {
      console.error('Failed to clear persisted IDE context:', error);
    }
  }

  // 手动保存当前上下文
  saveCurrentContext(): void {
    const currentContext = ideContext.getOpenFilesContext();
    this.saveContext(currentContext);
  }

  // 获取持久化文件信息
  getPersistenceInfo(): {
    path: string;
    exists: boolean;
    size: number;
    lastModified: Date | null;
  } {
    const exists = fs.existsSync(this.persistencePath);
    let size = 0;
    let lastModified: Date | null = null;

    if (exists) {
      try {
        const stats = fs.statSync(this.persistencePath);
        size = stats.size;
        lastModified = stats.mtime;
      } catch (error) {
        console.error('Failed to get persistence file stats:', error);
      }
    }

    return {
      path: this.persistencePath,
      exists,
      size,
      lastModified,
    };
  }

  destroy(): void {
    if (this.autosaveInterval) {
      clearTimeout(this.autosaveInterval);
      this.autosaveInterval = null;
    }
  }
}

// 使用持久化管理器
const persistence = new IDEContextPersistence();

// 获取持久化信息
const info = persistence.getPersistenceInfo();
console.log('Persistence info:', info);

// 手动保存
persistence.saveCurrentContext();

// 程序退出时清理
process.on('exit', () => {
  persistence.destroy();
});
```

## 错误处理和验证

### 数据验证

```typescript
import { OpenFilesSchema, OpenFilesNotificationSchema } from './ideContext.js';

// IDE 数据验证器
class IDEDataValidator {
  // 验证 OpenFiles 数据
  static validateOpenFiles(data: unknown): OpenFiles | null {
    try {
      return OpenFilesSchema.parse(data);
    } catch (error) {
      console.error('Invalid OpenFiles data:', error);
      return null;
    }
  }

  // 验证 IDE 通知数据
  static validateNotification(data: unknown): boolean {
    try {
      OpenFilesNotificationSchema.parse(data);
      return true;
    } catch (error) {
      console.error('Invalid IDE notification:', error);
      return false;
    }
  }

  // 验证文件路径
  static validateFilePath(filePath: string): boolean {
    // 检查路径格式
    if (typeof filePath !== 'string' || filePath.trim().length === 0) {
      return false;
    }

    // 检查路径安全性（防止路径遍历攻击）
    const normalizedPath = path.normalize(filePath);
    if (normalizedPath.includes('..')) {
      console.warn('Suspicious file path detected:', filePath);
      return false;
    }

    return true;
  }

  // 验证光标位置
  static validateCursor(cursor: unknown): boolean {
    if (!cursor || typeof cursor !== 'object') {
      return false;
    }

    const c = cursor as any;
    return (
      typeof c.line === 'number' && 
      typeof c.character === 'number' &&
      c.line >= 0 && 
      c.character >= 0
    );
  }

  // 清理和标准化数据
  static sanitizeOpenFiles(data: Partial<OpenFiles>): OpenFiles | null {
    try {
      // 确保必需字段
      if (!data.activeFile || !this.validateFilePath(data.activeFile)) {
        return null;
      }

      const sanitized: OpenFiles = {
        activeFile: path.normalize(data.activeFile),
      };

      // 可选字段处理
      if (data.selectedText && typeof data.selectedText === 'string') {
        // 限制选中文本长度
        sanitized.selectedText = data.selectedText.substring(0, 10000);
      }

      if (data.cursor && this.validateCursor(data.cursor)) {
        sanitized.cursor = data.cursor;
      }

      if (Array.isArray(data.recentOpenFiles)) {
        sanitized.recentOpenFiles = data.recentOpenFiles
          .filter(item => 
            item && 
            typeof item.filePath === 'string' && 
            typeof item.timestamp === 'number' &&
            this.validateFilePath(item.filePath)
          )
          .map(item => ({
            filePath: path.normalize(item.filePath),
            timestamp: item.timestamp,
          }))
          .slice(0, 20); // 限制最近文件数量
      }

      return sanitized;
    } catch (error) {
      console.error('Failed to sanitize OpenFiles data:', error);
      return null;
    }
  }
}

// 错误恢复的上下文管理器
class RobustIDEContextManager {
  private fallbackContext: OpenFiles | null = null;
  private errorCount = 0;
  private maxErrors = 5;

  constructor(private contextStore = ideContext) {
    this.setupErrorHandling();
  }

  private setupErrorHandling(): void {
    // 包装原始方法以添加错误处理
    const originalSet = this.contextStore.setOpenFilesContext;
    
    this.contextStore.setOpenFilesContext = (newOpenFiles: OpenFiles) => {
      try {
        // 验证数据
        const validated = IDEDataValidator.validateOpenFiles(newOpenFiles);
        if (!validated) {
          throw new Error('Invalid OpenFiles data');
        }

        // 调用原始方法
        originalSet.call(this.contextStore, validated);
        
        // 重置错误计数
        this.errorCount = 0;
        
        // 更新后备上下文
        this.fallbackContext = validated;
      } catch (error) {
        this.handleError(error as Error, newOpenFiles);
      }
    };
  }

  private handleError(error: Error, attemptedData: OpenFiles): void {
    this.errorCount++;
    console.error(`IDE context error ${this.errorCount}/${this.maxErrors}:`, error);

    if (this.errorCount >= this.maxErrors) {
      console.error('Too many IDE context errors, falling back to safe mode');
      this.enterSafeMode();
      return;
    }

    // 尝试恢复部分数据
    const sanitized = IDEDataValidator.sanitizeOpenFiles(attemptedData);
    if (sanitized) {
      console.log('Attempting recovery with sanitized data');
      try {
        this.contextStore.setOpenFilesContext(sanitized);
      } catch (recoveryError) {
        console.error('Recovery attempt failed:', recoveryError);
        this.useFallbackContext();
      }
    } else {
      this.useFallbackContext();
    }
  }

  private useFallbackContext(): void {
    if (this.fallbackContext) {
      console.log('Using fallback context');
      try {
        this.contextStore.setOpenFilesContext(this.fallbackContext);
      } catch (error) {
        console.error('Fallback context also failed:', error);
        this.enterSafeMode();
      }
    } else {
      this.enterSafeMode();
    }
  }

  private enterSafeMode(): void {
    console.log('Entering safe mode - clearing IDE context');
    try {
      this.contextStore.clearOpenFilesContext();
    } catch (error) {
      console.error('Failed to clear context in safe mode:', error);
    }
    
    // 通知用户
    console.warn('IDE integration is in safe mode due to repeated errors');
  }

  // 健康检查
  healthCheck(): {
    status: 'healthy' | 'degraded' | 'safe-mode';
    errorCount: number;
    hasContext: boolean;
    contextValid: boolean;
  } {
    const currentContext = this.contextStore.getOpenFilesContext();
    const hasContext = !!currentContext;
    const contextValid = hasContext ? 
      IDEDataValidator.validateOpenFiles(currentContext) !== null : 
      true;

    let status: 'healthy' | 'degraded' | 'safe-mode' = 'healthy';
    if (this.errorCount >= this.maxErrors) {
      status = 'safe-mode';
    } else if (this.errorCount > 0) {
      status = 'degraded';
    }

    return {
      status,
      errorCount: this.errorCount,
      hasContext,
      contextValid,
    };
  }

  // 重置错误状态
  resetErrorState(): void {
    this.errorCount = 0;
    console.log('IDE context error state reset');
  }
}

// 使用健壮的上下文管理器
const robustManager = new RobustIDEContextManager();

// 定期健康检查
setInterval(() => {
  const health = robustManager.healthCheck();
  if (health.status !== 'healthy') {
    console.log('IDE context health:', health);
  }
}, 30000);
```

## 测试支持

### 测试工具函数

```typescript
// IDE 上下文测试辅助工具
export class IDEContextTestHelper {
  // 创建测试用的 OpenFiles 对象
  static createTestOpenFiles(overrides: Partial<OpenFiles> = {}): OpenFiles {
    return {
      activeFile: '/test/src/TestFile.tsx',
      selectedText: 'const testFunction = () => {};',
      cursor: { line: 10, character: 5 },
      recentOpenFiles: [
        {
          filePath: '/test/src/TestFile.tsx',
          timestamp: Date.now(),
        },
        {
          filePath: '/test/src/AnotherFile.ts',
          timestamp: Date.now() - 60000,
        },
      ],
      ...overrides,
    };
  }

  // 创建隔离的测试上下文存储
  static createTestContextStore() {
    return createIdeContextStore();
  }

  // 模拟 IDE 通知
  static createTestNotification(openFiles: OpenFiles) {
    return {
      method: 'ide/openFilesChanged' as const,
      params: openFiles,
    };
  }

  // 等待订阅者被调用
  static async waitForSubscriber(
    contextStore: ReturnType<typeof createIdeContextStore>,
    timeout = 1000
  ): Promise<OpenFiles | undefined> {
    return new Promise((resolve, reject) => {
      const timer = setTimeout(() => {
        reject(new Error('Subscriber timeout'));
      }, timeout);

      const unsubscribe = contextStore.subscribeToOpenFiles((openFiles) => {
        clearTimeout(timer);
        unsubscribe();
        resolve(openFiles);
      });
    });
  }

  // 验证上下文数据结构
  static validateContextStructure(context: any): boolean {
    try {
      OpenFilesSchema.parse(context);
      return true;
    } catch {
      return false;
    }
  }
}

// 测试示例
describe('IDE Context Management', () => {
  let testStore: ReturnType<typeof createIdeContextStore>;

  beforeEach(() => {
    testStore = IDEContextTestHelper.createTestContextStore();
  });

  it('should set and get context correctly', () => {
    const testContext = IDEContextTestHelper.createTestOpenFiles();
    
    testStore.setOpenFilesContext(testContext);
    const retrieved = testStore.getOpenFilesContext();
    
    expect(retrieved).toEqual(testContext);
  });

  it('should notify subscribers on context change', async () => {
    const testContext = IDEContextTestHelper.createTestOpenFiles();
    
    const subscriberPromise = IDEContextTestHelper.waitForSubscriber(testStore);
    testStore.setOpenFilesContext(testContext);
    
    const notifiedContext = await subscriberPromise;
    expect(notifiedContext).toEqual(testContext);
  });

  it('should handle context clearing', () => {
    const testContext = IDEContextTestHelper.createTestOpenFiles();
    
    testStore.setOpenFilesContext(testContext);
    testStore.clearOpenFilesContext();
    
    const retrieved = testStore.getOpenFilesContext();
    expect(retrieved).toBeUndefined();
  });

  it('should support multiple subscribers', () => {
    const subscriber1 = jest.fn();
    const subscriber2 = jest.fn();
    
    testStore.subscribeToOpenFiles(subscriber1);
    testStore.subscribeToOpenFiles(subscriber2);
    
    const testContext = IDEContextTestHelper.createTestOpenFiles();
    testStore.setOpenFilesContext(testContext);
    
    expect(subscriber1).toHaveBeenCalledWith(testContext);
    expect(subscriber2).toHaveBeenCalledWith(testContext);
  });

  it('should handle unsubscription correctly', () => {
    const subscriber = jest.fn();
    const unsubscribe = testStore.subscribeToOpenFiles(subscriber);
    
    unsubscribe();
    
    const testContext = IDEContextTestHelper.createTestOpenFiles();
    testStore.setOpenFilesContext(testContext);
    
    expect(subscriber).not.toHaveBeenCalled();
  });

  it('should validate notification schema', () => {
    const testContext = IDEContextTestHelper.createTestOpenFiles();
    const notification = IDEContextTestHelper.createTestNotification(testContext);
    
    const isValid = IDEContextTestHelper.validateContextStructure(notification);
    expect(isValid).toBe(true);
  });
});
```

## 最佳实践

### 性能优化

```typescript
// ✅ 推荐：防抖订阅者
class DebouncedSubscriber {
  private timeoutId: NodeJS.Timeout | null = null;
  private delay: number;

  constructor(
    private callback: (openFiles: OpenFiles | undefined) => void,
    delay = 300
  ) {
    this.delay = delay;
  }

  notify(openFiles: OpenFiles | undefined): void {
    if (this.timeoutId) {
      clearTimeout(this.timeoutId);
    }

    this.timeoutId = setTimeout(() => {
      this.callback(openFiles);
      this.timeoutId = null;
    }, this.delay);
  }

  cancel(): void {
    if (this.timeoutId) {
      clearTimeout(this.timeoutId);
      this.timeoutId = null;
    }
  }
}

// 使用防抖订阅者
const debouncedHandler = new DebouncedSubscriber((openFiles) => {
  console.log('Debounced context update:', openFiles?.activeFile);
}, 500);

ideContext.subscribeToOpenFiles((openFiles) => {
  debouncedHandler.notify(openFiles);
});

// ✅ 推荐：内存优化的最近文件管理
class OptimizedRecentFiles {
  private maxFiles = 10;
  private maxAge = 24 * 60 * 60 * 1000; // 24小时

  filterRecentFiles(recentFiles: Array<{filePath: string, timestamp: number}>): Array<{filePath: string, timestamp: number}> {
    const now = Date.now();
    
    return recentFiles
      .filter(file => now - file.timestamp < this.maxAge) // 过滤过期文件
      .sort((a, b) => b.timestamp - a.timestamp) // 按时间排序
      .slice(0, this.maxFiles) // 限制数量
      .filter((file, index, arr) => 
        arr.findIndex(f => f.filePath === file.filePath) === index // 去重
      );
  }

  addRecentFile(
    recentFiles: Array<{filePath: string, timestamp: number}>,
    newFile: string
  ): Array<{filePath: string, timestamp: number}> {
    const updatedFiles = [
      { filePath: newFile, timestamp: Date.now() },
      ...recentFiles.filter(file => file.filePath !== newFile)
    ];

    return this.filterRecentFiles(updatedFiles);
  }
}
```

### 错误处理

```typescript
// ✅ 推荐：优雅的错误处理
class SafeIDEContextManager {
  private static readonly MAX_TEXT_LENGTH = 10000;
  private static readonly MAX_RECENT_FILES = 20;

  static sanitizeContext(context: any): OpenFiles | null {
    try {
      // 基本验证
      if (!context || typeof context.activeFile !== 'string') {
        return null;
      }

      const sanitized: OpenFiles = {
        activeFile: this.sanitizePath(context.activeFile),
      };

      // 安全地处理可选字段
      if (context.selectedText && typeof context.selectedText === 'string') {
        sanitized.selectedText = context.selectedText.substring(0, this.MAX_TEXT_LENGTH);
      }

      if (this.isValidCursor(context.cursor)) {
        sanitized.cursor = context.cursor;
      }

      if (Array.isArray(context.recentOpenFiles)) {
        sanitized.recentOpenFiles = this.sanitizeRecentFiles(context.recentOpenFiles);
      }

      return sanitized;
    } catch (error) {
      console.error('Context sanitization failed:', error);
      return null;
    }
  }

  private static sanitizePath(filePath: string): string {
    return path.normalize(filePath.trim());
  }

  private static isValidCursor(cursor: any): boolean {
    return cursor && 
           typeof cursor.line === 'number' && 
           typeof cursor.character === 'number' &&
           cursor.line >= 0 && 
           cursor.character >= 0;
  }

  private static sanitizeRecentFiles(files: any[]): Array<{filePath: string, timestamp: number}> {
    return files
      .filter(file => 
        file && 
        typeof file.filePath === 'string' && 
        typeof file.timestamp === 'number'
      )
      .map(file => ({
        filePath: this.sanitizePath(file.filePath),
        timestamp: file.timestamp,
      }))
      .slice(0, this.MAX_RECENT_FILES);
  }
}
```

### 类型安全

```typescript
// ✅ 推荐：严格的类型定义
interface StrictOpenFiles {
  readonly activeFile: string;
  readonly selectedText?: string;
  readonly cursor?: Readonly<{
    readonly line: number;
    readonly character: number;
  }>;
  readonly recentOpenFiles?: ReadonlyArray<Readonly<{
    readonly filePath: string;
    readonly timestamp: number;
  }>>;
}

// ✅ 推荐：类型守卫函数
function isOpenFiles(value: unknown): value is OpenFiles {
  try {
    OpenFilesSchema.parse(value);
    return true;
  } catch {
    return false;
  }
}

function isCursor(value: unknown): value is Cursor {
  try {
    CursorSchema.parse(value);
    return true;
  } catch {
    return false;
  }
}

// 使用类型守卫
function handleUnknownData(data: unknown): void {
  if (isOpenFiles(data)) {
    // 这里 TypeScript 知道 data 是 OpenFiles 类型
    console.log('Active file:', data.activeFile);
    
    if (data.cursor && isCursor(data.cursor)) {
      console.log(`Cursor at ${data.cursor.line}:${data.cursor.character}`);
    }
  } else {
    console.error('Invalid data format');
  }
}
```