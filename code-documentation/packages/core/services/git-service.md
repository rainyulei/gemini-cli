# GitService Git 版本控制服务文档

## 概述

`GitService` 是 Gemini CLI 的 Git 版本控制服务，专门用于支持检查点（checkpointing）功能。它创建和管理一个隐藏的 Git 仓库来跟踪项目文件的变化，提供文件快照、恢复和版本管理能力，而不会干扰用户的主 Git 仓库。

## 主要功能

- 隐藏 Git 仓库创建和管理
- 项目文件快照和版本控制
- 检查点创建和恢复机制
- Git 可用性验证
- 跨平台兼容性支持
- 用户配置隔离

## 核心类定义

### `GitService`

**功能**: Git 版本控制服务主类

```typescript
export class GitService {
  private projectRoot: string;

  constructor(projectRoot: string) {
    this.projectRoot = path.resolve(projectRoot);
  }
}
```

**构造参数**:
- `projectRoot`: 项目根目录的绝对路径

## 核心方法

### 初始化和验证

#### `async initialize(): Promise<void>`

**功能**: 初始化 Git 服务

```typescript
async initialize(): Promise<void> {
  const gitAvailable = await this.verifyGitAvailability();
  if (!gitAvailable) {
    throw new Error(
      'Checkpointing is enabled, but Git is not installed. Please install Git or disable checkpointing to continue.',
    );
  }
  this.setupShadowGitRepository();
}
```

**执行流程**:

1. **Git 可用性检查**: 验证系统是否安装了 Git
2. **错误处理**: 如果 Git 不可用，抛出明确的错误信息
3. **仓库设置**: 创建和配置隐藏的 Git 仓库

#### `verifyGitAvailability(): Promise<boolean>`

**功能**: 验证 Git 是否可用

```typescript
verifyGitAvailability(): Promise<boolean> {
  return new Promise((resolve) => {
    exec('git --version', (error) => {
      if (error) {
        resolve(false);
      } else {
        resolve(true);
      }
    });
  });
}
```

**检查方式**: 通过执行 `git --version` 命令来检测 Git 安装状态

### 仓库管理

#### `private getHistoryDir(): string`

**功能**: 获取历史记录存储目录

```typescript
private getHistoryDir(): string {
  const hash = getProjectHash(this.projectRoot);
  return path.join(os.homedir(), GEMINI_DIR, 'history', hash);
}
```

**目录结构**:
- **基础路径**: `~/.gemini/history/`
- **项目标识**: 使用项目路径的哈希值作为子目录名
- **最终路径**: `~/.gemini/history/{projectHash}/`

#### `async setupShadowGitRepository(): Promise<void>`

**功能**: 创建和配置隐藏的 Git 仓库

```typescript
async setupShadowGitRepository() {
  const repoDir = this.getHistoryDir();
  const gitConfigPath = path.join(repoDir, '.gitconfig');

  await fs.mkdir(repoDir, { recursive: true });

  // 创建专用的 Git 配置，避免继承用户设置
  const gitConfigContent =
    '[user]\n  name = Gemini CLI\n  email = gemini-cli@google.com\n[commit]\n  gpgsign = false\n';
  await fs.writeFile(gitConfigPath, gitConfigContent);

  const repo = simpleGit(repoDir);
  const isRepoDefined = await repo.checkIsRepo(CheckRepoActions.IS_REPO_ROOT);

  if (!isRepoDefined) {
    await repo.init(false, {
      '--initial-branch': 'main',
    });
    await repo.commit('Initial commit', { '--allow-empty': null });
  }

  // 同步用户的 .gitignore 文件到隐藏仓库
  const userGitIgnorePath = path.join(this.projectRoot, '.gitignore');
  const shadowGitIgnorePath = path.join(repoDir, '.gitignore');

  let userGitIgnoreContent = '';
  try {
    userGitIgnoreContent = await fs.readFile(userGitIgnorePath, 'utf-8');
  } catch (error) {
    if (isNodeError(error) && error.code !== 'ENOENT') {
      throw error;
    }
  }

  await fs.writeFile(shadowGitIgnorePath, userGitIgnoreContent);
}
```

**设置步骤**:

1. **目录创建**: 确保历史记录目录存在
2. **配置文件**: 创建专用的 Git 配置文件，避免用户设置干扰
3. **仓库检查**: 检查是否已存在 Git 仓库
4. **仓库初始化**: 如果不存在，初始化新的 Git 仓库
5. **初始提交**: 创建空的初始提交
6. **忽略文件同步**: 将用户的 .gitignore 复制到隐藏仓库

#### `private get shadowGitRepository(): SimpleGit`

**功能**: 获取配置好的隐藏 Git 仓库实例

```typescript
private get shadowGitRepository(): SimpleGit {
  const repoDir = this.getHistoryDir();
  return simpleGit(this.projectRoot).env({
    GIT_DIR: path.join(repoDir, '.git'),
    GIT_WORK_TREE: this.projectRoot,
    // 防止 Git 使用用户的全局配置
    HOME: repoDir,
    XDG_CONFIG_HOME: repoDir,
  });
}
```

**环境变量配置**:
- `GIT_DIR`: 指向隐藏仓库的 .git 目录
- `GIT_WORK_TREE`: 指向项目根目录作为工作树
- `HOME`: 重定向到仓库目录，避免使用用户配置
- `XDG_CONFIG_HOME`: 同样重定向以隔离配置

### 快照和恢复

#### `async getCurrentCommitHash(): Promise<string>`

**功能**: 获取当前提交的哈希值

```typescript
async getCurrentCommitHash(): Promise<string> {
  const hash = await this.shadowGitRepository.raw('rev-parse', 'HEAD');
  return hash.trim();
}
```

#### `async createFileSnapshot(message: string): Promise<string>`

**功能**: 创建文件快照

```typescript
async createFileSnapshot(message: string): Promise<string> {
  const repo = this.shadowGitRepository;
  await repo.add('.');
  const commitResult = await repo.commit(message);
  return commitResult.commit;
}
```

**执行步骤**:

1. **添加文件**: 将所有文件添加到暂存区
2. **创建提交**: 使用提供的消息创建提交
3. **返回哈希**: 返回新创建提交的哈希值

#### `async restoreProjectFromSnapshot(commitHash: string): Promise<void>`

**功能**: 从快照恢复项目文件

```typescript
async restoreProjectFromSnapshot(commitHash: string): Promise<void> {
  const repo = this.shadowGitRepository;
  await repo.raw(['restore', '--source', commitHash, '.']);
  // 移除快照后引入的未跟踪文件
  await repo.clean('f', ['-d']);
}
```

**恢复步骤**:

1. **文件恢复**: 从指定提交恢复所有文件
2. **清理未跟踪文件**: 删除恢复后新产生的未跟踪文件和目录

## 使用示例

### 基本服务初始化

```typescript
import { GitService } from './gitService.js';

// 创建 Git 服务实例
const gitService = new GitService('/path/to/project');

// 初始化服务
try {
  await gitService.initialize();
  console.log('Git service initialized successfully');
} catch (error) {
  console.error('Failed to initialize Git service:', error.message);
  // 处理 Git 不可用的情况
}
```

### 创建和管理快照

```typescript
// 创建项目快照
async function createProjectCheckpoint(
  gitService: GitService,
  description: string
): Promise<string> {
  try {
    const commitHash = await gitService.createFileSnapshot(
      `Checkpoint: ${description} - ${new Date().toISOString()}`
    );
    
    console.log(`Checkpoint created: ${commitHash}`);
    return commitHash;
  } catch (error) {
    console.error('Failed to create checkpoint:', error);
    throw error;
  }
}

// 使用示例
const checkpointHash = await createProjectCheckpoint(
  gitService,
  'Before major refactoring'
);
```

### 快照恢复

```typescript
// 恢复到特定快照
async function restoreToCheckpoint(
  gitService: GitService,
  commitHash: string
): Promise<void> {
  try {
    console.log(`Restoring to checkpoint: ${commitHash}`);
    await gitService.restoreProjectFromSnapshot(commitHash);
    console.log('Project restored successfully');
  } catch (error) {
    console.error('Failed to restore checkpoint:', error);
    throw error;
  }
}

// 使用示例
await restoreToCheckpoint(gitService, checkpointHash);
```

### 获取当前状态

```typescript
// 获取当前快照信息
async function getCurrentSnapshot(gitService: GitService): Promise<string> {
  try {
    const currentHash = await gitService.getCurrentCommitHash();
    console.log(`Current snapshot: ${currentHash}`);
    return currentHash;
  } catch (error) {
    console.error('Failed to get current snapshot:', error);
    throw error;
  }
}
```

## 高级功能

### 检查点管理器

```typescript
class CheckpointManager {
  private gitService: GitService;
  private checkpoints: Map<string, CheckpointInfo> = new Map();

  constructor(projectRoot: string) {
    this.gitService = new GitService(projectRoot);
  }

  async initialize(): Promise<void> {
    await this.gitService.initialize();
    await this.loadExistingCheckpoints();
  }

  async createCheckpoint(
    name: string,
    description?: string
  ): Promise<CheckpointInfo> {
    const timestamp = new Date().toISOString();
    const message = `${name}: ${description || 'No description'} (${timestamp})`;
    
    const commitHash = await this.gitService.createFileSnapshot(message);
    
    const checkpointInfo: CheckpointInfo = {
      name,
      description,
      commitHash,
      timestamp,
      created: new Date(),
    };

    this.checkpoints.set(name, checkpointInfo);
    await this.saveCheckpointMetadata();

    return checkpointInfo;
  }

  async restoreCheckpoint(name: string): Promise<void> {
    const checkpoint = this.checkpoints.get(name);
    if (!checkpoint) {
      throw new Error(`Checkpoint '${name}' not found`);
    }

    await this.gitService.restoreProjectFromSnapshot(checkpoint.commitHash);
  }

  async listCheckpoints(): Promise<CheckpointInfo[]> {
    return Array.from(this.checkpoints.values()).sort(
      (a, b) => b.created.getTime() - a.created.getTime()
    );
  }

  async deleteCheckpoint(name: string): Promise<void> {
    if (!this.checkpoints.has(name)) {
      throw new Error(`Checkpoint '${name}' not found`);
    }

    this.checkpoints.delete(name);
    await this.saveCheckpointMetadata();
  }

  private async loadExistingCheckpoints(): Promise<void> {
    // 实现从存储中加载检查点信息的逻辑
  }

  private async saveCheckpointMetadata(): Promise<void> {
    // 实现保存检查点元数据的逻辑
  }
}

interface CheckpointInfo {
  name: string;
  description?: string;
  commitHash: string;
  timestamp: string;
  created: Date;
}
```

### 自动检查点系统

```typescript
class AutoCheckpointSystem {
  private gitService: GitService;
  private checkpointManager: CheckpointManager;
  private autoCheckpointInterval: number;
  private maxAutoCheckpoints: number;

  constructor(
    projectRoot: string,
    options: {
      intervalMinutes?: number;
      maxCheckpoints?: number;
    } = {}
  ) {
    this.gitService = new GitService(projectRoot);
    this.checkpointManager = new CheckpointManager(projectRoot);
    this.autoCheckpointInterval = (options.intervalMinutes || 15) * 60 * 1000;
    this.maxAutoCheckpoints = options.maxCheckpoints || 10;
  }

  async start(): Promise<void> {
    await this.gitService.initialize();
    await this.checkpointManager.initialize();

    // 创建初始检查点
    await this.createAutoCheckpoint('Auto: System started');

    // 启动定时检查点
    setInterval(async () => {
      try {
        await this.createAutoCheckpoint('Auto: Periodic checkpoint');
        await this.cleanupOldCheckpoints();
      } catch (error) {
        console.error('Auto checkpoint failed:', error);
      }
    }, this.autoCheckpointInterval);
  }

  private async createAutoCheckpoint(description: string): Promise<void> {
    const name = `auto_${Date.now()}`;
    await this.checkpointManager.createCheckpoint(name, description);
  }

  private async cleanupOldCheckpoints(): Promise<void> {
    const checkpoints = await this.checkpointManager.listCheckpoints();
    const autoCheckpoints = checkpoints.filter(cp => cp.name.startsWith('auto_'));

    if (autoCheckpoints.length > this.maxAutoCheckpoints) {
      const toDelete = autoCheckpoints.slice(this.maxAutoCheckpoints);
      for (const checkpoint of toDelete) {
        await this.checkpointManager.deleteCheckpoint(checkpoint.name);
      }
    }
  }

  async createManualCheckpoint(name: string, description?: string): Promise<void> {
    await this.checkpointManager.createCheckpoint(name, description);
  }

  async restoreToCheckpoint(name: string): Promise<void> {
    await this.checkpointManager.restoreCheckpoint(name);
  }

  async listAllCheckpoints(): Promise<CheckpointInfo[]> {
    return this.checkpointManager.listCheckpoints();
  }
}
```

### 快照比较工具

```typescript
class SnapshotComparator {
  private gitService: GitService;

  constructor(projectRoot: string) {
    this.gitService = new GitService(projectRoot);
  }

  async compareSnapshots(
    fromHash: string,
    toHash: string
  ): Promise<SnapshotDiff> {
    const repo = this.gitService['shadowGitRepository'];
    
    // 获取两个快照之间的差异
    const diffSummary = await repo.diffSummary([fromHash, toHash]);
    
    // 获取详细的文件差异
    const fileDiffs: FileDiff[] = [];
    for (const file of diffSummary.files) {
      const diff = await repo.diff([fromHash, toHash, '--', file.file]);
      fileDiffs.push({
        file: file.file,
        changes: file.changes,
        insertions: file.insertions,
        deletions: file.deletions,
        diff: diff
      });
    }

    return {
      summary: {
        totalFiles: diffSummary.files.length,
        totalChanges: diffSummary.changes,
        totalInsertions: diffSummary.insertions,
        totalDeletions: diffSummary.deletions,
      },
      files: fileDiffs,
      fromHash,
      toHash,
    };
  }

  async getSnapshotInfo(commitHash: string): Promise<SnapshotInfo> {
    const repo = this.gitService['shadowGitRepository'];
    
    const commitInfo = await repo.show(['--format=fuller', '--no-patch', commitHash]);
    const fileList = await repo.raw(['ls-tree', '-r', '--name-only', commitHash]);
    
    return {
      hash: commitHash,
      message: this.extractCommitMessage(commitInfo),
      timestamp: this.extractCommitDate(commitInfo),
      files: fileList.trim().split('\n').filter(f => f.length > 0),
    };
  }

  private extractCommitMessage(commitInfo: string): string {
    // 实现提取提交消息的逻辑
    const lines = commitInfo.split('\n');
    const messageStart = lines.findIndex(line => line.trim() === '') + 1;
    return lines.slice(messageStart).join('\n').trim();
  }

  private extractCommitDate(commitInfo: string): Date {
    // 实现提取提交日期的逻辑
    const dateMatch = commitInfo.match(/CommitDate:\s+(.+)/);
    return dateMatch ? new Date(dateMatch[1]) : new Date();
  }
}

interface SnapshotDiff {
  summary: {
    totalFiles: number;
    totalChanges: number;
    totalInsertions: number;
    totalDeletions: number;
  };
  files: FileDiff[];
  fromHash: string;
  toHash: string;
}

interface FileDiff {
  file: string;
  changes: number;
  insertions: number;
  deletions: number;
  diff: string;
}

interface SnapshotInfo {
  hash: string;
  message: string;
  timestamp: Date;
  files: string[];
}
```

## 错误处理

### 常见错误场景

```typescript
// Git 不可用
Error: Checkpointing is enabled, but Git is not installed.

// 仓库初始化失败
Error: Failed to initialize shadow Git repository

// 快照创建失败
Error: Failed to create file snapshot: working tree dirty

// 快照恢复失败
Error: Invalid commit hash: abc123

// 权限错误
Error: EACCES: permission denied, mkdir ~/.gemini/history
```

### 错误恢复策略

```typescript
class RobustGitService extends GitService {
  private fallbackMode = false;

  async initialize(): Promise<void> {
    try {
      await super.initialize();
    } catch (error) {
      console.warn('Git service initialization failed, entering fallback mode:', error.message);
      this.fallbackMode = true;
      this.initializeFallbackMode();
    }
  }

  private initializeFallbackMode(): void {
    console.log('Checkpointing disabled due to Git unavailability');
    // 可以实现基于文件系统的简单备份机制
  }

  async createFileSnapshot(message: string): Promise<string> {
    if (this.fallbackMode) {
      return this.createFallbackSnapshot(message);
    }

    try {
      return await super.createFileSnapshot(message);
    } catch (error) {
      console.error('Failed to create Git snapshot:', error);
      
      // 尝试修复常见问题
      if (error.message.includes('working tree dirty')) {
        // 尝试清理工作树
        await this.cleanWorkingTree();
        return await super.createFileSnapshot(message);
      }
      
      throw error;
    }
  }

  async restoreProjectFromSnapshot(commitHash: string): Promise<void> {
    if (this.fallbackMode) {
      return this.restoreFromFallbackSnapshot(commitHash);
    }

    try {
      await super.restoreProjectFromSnapshot(commitHash);
    } catch (error) {
      console.error('Failed to restore from Git snapshot:', error);
      
      // 尝试验证提交哈希
      if (error.message.includes('Invalid commit hash')) {
        const validHash = await this.findSimilarCommit(commitHash);
        if (validHash) {
          console.warn(`Using similar commit: ${validHash}`);
          await super.restoreProjectFromSnapshot(validHash);
          return;
        }
      }
      
      throw error;
    }
  }

  private async createFallbackSnapshot(message: string): Promise<string> {
    // 实现基于文件系统的快照机制
    const timestamp = Date.now().toString();
    const snapshotId = `fallback_${timestamp}`;
    
    // 创建快照目录和文件副本
    // ... 实现细节
    
    return snapshotId;
  }

  private async restoreFromFallbackSnapshot(snapshotId: string): Promise<void> {
    // 实现从文件系统快照恢复
    // ... 实现细节
  }

  private async cleanWorkingTree(): Promise<void> {
    try {
      const repo = this['shadowGitRepository'];
      await repo.clean('f', ['-d']);
      await repo.reset(['--hard']);
    } catch (error) {
      console.warn('Failed to clean working tree:', error);
    }
  }

  private async findSimilarCommit(invalidHash: string): Promise<string | null> {
    try {
      const repo = this['shadowGitRepository'];
      const commits = await repo.log({ maxCount: 10 });
      
      // 查找最相似的提交（基于前缀匹配）
      for (const commit of commits.all) {
        if (commit.hash.startsWith(invalidHash.substring(0, 6))) {
          return commit.hash;
        }
      }
    } catch (error) {
      console.warn('Failed to find similar commit:', error);
    }
    
    return null;
  }

  // 健康检查方法
  async healthCheck(): Promise<{
    isHealthy: boolean;
    issues: string[];
    suggestions: string[];
  }> {
    const issues: string[] = [];
    const suggestions: string[] = [];

    // 检查 Git 可用性
    const gitAvailable = await this.verifyGitAvailability();
    if (!gitAvailable) {
      issues.push('Git is not installed or not available in PATH');
      suggestions.push('Install Git or add it to your system PATH');
    }

    // 检查仓库状态
    try {
      const historyDir = this.getHistoryDir();
      const exists = await fs.access(historyDir).then(() => true, () => false);
      
      if (!exists) {
        issues.push('Shadow Git repository not found');
        suggestions.push('Run git service initialization');
      }
    } catch (error) {
      issues.push(`Cannot access history directory: ${error.message}`);
      suggestions.push('Check file system permissions');
    }

    return {
      isHealthy: issues.length === 0,
      issues,
      suggestions,
    };
  }
}
```

## 性能优化

### 快照优化策略

```typescript
class OptimizedGitService extends GitService {
  private snapshotCache = new Map<string, string>();
  private compressionEnabled = true;

  async createFileSnapshot(message: string): Promise<string> {
    // 检查是否有实际变化
    const currentState = await this.calculateProjectHash();
    const cachedHash = this.snapshotCache.get(currentState);
    
    if (cachedHash) {
      console.log('No changes detected, reusing existing snapshot');
      return cachedHash;
    }

    // 创建新快照
    const commitHash = await super.createFileSnapshot(message);
    this.snapshotCache.set(currentState, commitHash);

    // 定期清理缓存
    if (this.snapshotCache.size > 100) {
      this.pruneCache();
    }

    return commitHash;
  }

  private async calculateProjectHash(): Promise<string> {
    // 简单的项目状态哈希计算
    const crypto = await import('crypto');
    const hash = crypto.createHash('sha256');
    
    // 可以基于文件修改时间、大小等计算快速哈希
    // ... 实现细节
    
    return hash.digest('hex');
  }

  private pruneCache(): void {
    // 保留最近的 50 个缓存条目
    const entries = Array.from(this.snapshotCache.entries());
    const toKeep = entries.slice(-50);
    
    this.snapshotCache.clear();
    toKeep.forEach(([key, value]) => {
      this.snapshotCache.set(key, value);
    });
  }

  // 异步快照创建
  async createSnapshotAsync(message: string): Promise<void> {
    // 在后台创建快照，不阻塞主流程
    setImmediate(async () => {
      try {
        await this.createFileSnapshot(message);
      } catch (error) {
        console.error('Background snapshot creation failed:', error);
      }
    });
  }
}
```

## 最佳实践

### 服务初始化

```typescript
// ✅ 推荐：优雅的服务初始化
async function initializeGitService(projectRoot: string): Promise<GitService | null> {
  const gitService = new GitService(projectRoot);
  
  try {
    await gitService.initialize();
    console.log('Git service initialized successfully');
    return gitService;
  } catch (error) {
    console.warn('Git service unavailable:', error.message);
    console.log('Continuing without checkpointing functionality');
    return null;
  }
}

// 使用示例
const gitService = await initializeGitService('/path/to/project');
if (gitService) {
  // 使用检查点功能
}
```

### 快照管理

```typescript
// ✅ 推荐：有意义的快照消息
const meaningfulMessages = [
  'Before refactoring UserService class',
  'After implementing authentication middleware',
  'Pre-deployment checkpoint',
  'Before experimental feature implementation',
];

// ✅ 推荐：定期清理策略
async function maintainSnapshots(gitService: GitService): Promise<void> {
  // 定期清理超过 30 天的自动快照
  const thirtyDaysAgo = new Date(Date.now() - 30 * 24 * 60 * 60 * 1000);
  
  // ... 实现清理逻辑
}

// ✅ 推荐：快照验证
async function verifySnapshot(
  gitService: GitService,
  commitHash: string
): Promise<boolean> {
  try {
    const currentHash = await gitService.getCurrentCommitHash();
    return commitHash === currentHash;
  } catch (error) {
    return false;
  }
}
```

### 错误处理

```typescript
// ✅ 推荐：全面的错误处理
async function safeCreateSnapshot(
  gitService: GitService,
  message: string
): Promise<string | null> {
  try {
    return await gitService.createFileSnapshot(message);
  } catch (error) {
    console.error('Snapshot creation failed:', error.message);
    
    // 记录错误但不中断主流程
    if (error.message.includes('Git not installed')) {
      console.log('Consider installing Git for checkpointing functionality');
    }
    
    return null;
  }
}
```