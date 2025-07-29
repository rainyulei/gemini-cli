# FileDiscoveryService 文件发现服务文档

## 概述

`FileDiscoveryService` 是 Gemini CLI 的文件发现和过滤服务，负责管理 Git 忽略规则和 Gemini 忽略规则的解析与应用。它提供统一的文件过滤接口，支持多种过滤选项，是文件操作工具的基础服务。

## 主要功能

- Git 忽略文件（.gitignore）解析和应用
- Gemini 忽略文件（.geminiignore）解析和应用
- 灵活的文件过滤选项配置
- 批量文件过滤能力
- 单文件忽略状态检查
- 模式缓存和性能优化

## 接口定义

### `FilterFilesOptions`

**功能**: 文件过滤选项接口

```typescript
export interface FilterFilesOptions {
  respectGitIgnore?: boolean;           // 是否遵循 Git 忽略规则
  respectGeminiIgnore?: boolean;        // 是否遵循 Gemini 忽略规则
}
```

**默认值**:
- `respectGitIgnore`: `true`
- `respectGeminiIgnore`: `true`

## 核心类定义

### `FileDiscoveryService`

**功能**: 文件发现服务主类

```typescript
export class FileDiscoveryService {
  private gitIgnoreFilter: GitIgnoreFilter | null = null;
  private geminiIgnoreFilter: GitIgnoreFilter | null = null;
  private projectRoot: string;

  constructor(projectRoot: string) {
    this.projectRoot = path.resolve(projectRoot);
    
    // 初始化 Git 忽略过滤器
    if (isGitRepository(this.projectRoot)) {
      const parser = new GitIgnoreParser(this.projectRoot);
      try {
        parser.loadGitRepoPatterns();
      } catch (_error) {
        // 忽略文件不存在或读取失败
      }
      this.gitIgnoreFilter = parser;
    }
    
    // 初始化 Gemini 忽略过滤器
    const gParser = new GitIgnoreParser(this.projectRoot);
    try {
      gParser.loadPatterns(GEMINI_IGNORE_FILE_NAME);
    } catch (_error) {
      // 忽略文件不存在或读取失败
    }
    this.geminiIgnoreFilter = gParser;
  }
}
```

**常量定义**:

```typescript
const GEMINI_IGNORE_FILE_NAME = '.geminiignore';
```

## 核心方法

### 批量文件过滤

#### `filterFiles(filePaths: string[], options?: FilterFilesOptions): string[]`

**功能**: 根据忽略规则过滤文件路径数组

```typescript
filterFiles(
  filePaths: string[],
  options: FilterFilesOptions = {
    respectGitIgnore: true,
    respectGeminiIgnore: true,
  },
): string[] {
  return filePaths.filter((filePath) => {
    if (options.respectGitIgnore && this.shouldGitIgnoreFile(filePath)) {
      return false;
    }
    if (
      options.respectGeminiIgnore &&
      this.shouldGeminiIgnoreFile(filePath)
    ) {
      return false;
    }
    return true;
  });
}
```

**执行流程**:

1. **遍历文件路径**: 对每个文件路径进行检查
2. **Git 忽略检查**: 如果启用且文件被 Git 忽略，则过滤掉
3. **Gemini 忽略检查**: 如果启用且文件被 Gemini 忽略，则过滤掉
4. **返回结果**: 返回通过所有过滤条件的文件路径

### 单文件忽略检查

#### `shouldGitIgnoreFile(filePath: string): boolean`

**功能**: 检查单个文件是否应该被 Git 忽略

```typescript
shouldGitIgnoreFile(filePath: string): boolean {
  if (this.gitIgnoreFilter) {
    return this.gitIgnoreFilter.isIgnored(filePath);
  }
  return false;
}
```

#### `shouldGeminiIgnoreFile(filePath: string): boolean`

**功能**: 检查单个文件是否应该被 Gemini 忽略

```typescript
shouldGeminiIgnoreFile(filePath: string): boolean {
  if (this.geminiIgnoreFilter) {
    return this.geminiIgnoreFilter.isIgnored(filePath);
  }
  return false;
}
```

### 统一忽略检查

#### `shouldIgnoreFile(filePath: string, options?: FilterFilesOptions): boolean`

**功能**: 统一检查文件是否应该被忽略

```typescript
shouldIgnoreFile(
  filePath: string,
  options: FilterFilesOptions = {},
): boolean {
  const { respectGitIgnore = true, respectGeminiIgnore = true } = options;

  if (respectGitIgnore && this.shouldGitIgnoreFile(filePath)) {
    return true;
  }
  if (respectGeminiIgnore && this.shouldGeminiIgnoreFile(filePath)) {
    return true;
  }
  return false;
}
```

### 模式获取

#### `getGeminiIgnorePatterns(): string[]`

**功能**: 获取已加载的 Gemini 忽略模式

```typescript
getGeminiIgnorePatterns(): string[] {
  return this.geminiIgnoreFilter?.getPatterns() ?? [];
}
```

## 使用示例

### 基本文件过滤

```typescript
import { FileDiscoveryService } from './fileDiscoveryService.js';

// 创建服务实例
const fileService = new FileDiscoveryService('/path/to/project');

// 准备文件列表
const allFiles = [
  'src/index.ts',
  'src/utils.ts',
  'node_modules/package.json',
  '.git/config',
  'build/output.js',
  'temp/cache.json',
  'README.md'
];

// 使用默认选项过滤
const filteredFiles = fileService.filterFiles(allFiles);
console.log('Filtered files:', filteredFiles);
// 输出: ['src/index.ts', 'src/utils.ts', 'README.md']
```

### 自定义过滤选项

```typescript
// 仅应用 Git 忽略规则
const gitOnlyFiltered = fileService.filterFiles(allFiles, {
  respectGitIgnore: true,
  respectGeminiIgnore: false
});

// 仅应用 Gemini 忽略规则
const geminiOnlyFiltered = fileService.filterFiles(allFiles, {
  respectGitIgnore: false,
  respectGeminiIgnore: true
});

// 禁用所有过滤
const unfiltered = fileService.filterFiles(allFiles, {
  respectGitIgnore: false,
  respectGeminiIgnore: false
});
```

### 单文件检查

```typescript
// 检查特定文件的忽略状态
const filePath = 'src/components/Button.tsx';

if (fileService.shouldGitIgnoreFile(filePath)) {
  console.log('File is git-ignored');
}

if (fileService.shouldGeminiIgnoreFile(filePath)) {
  console.log('File is gemini-ignored');
}

// 统一检查
if (fileService.shouldIgnoreFile(filePath)) {
  console.log('File should be ignored');
} else {
  console.log('File should be processed');
}
```

### 与工具集成

```typescript
class FileOperationTool {
  constructor(
    private fileService: FileDiscoveryService,
    private config: ToolConfig
  ) {}

  async processFiles(patterns: string[]): Promise<ProcessResult> {
    // 使用 glob 查找文件
    const allFiles = await this.findFilesByPatterns(patterns);
    
    // 应用忽略规则过滤
    const filteredFiles = this.fileService.filterFiles(allFiles, {
      respectGitIgnore: this.config.respectGitIgnore,
      respectGeminiIgnore: this.config.respectGeminiIgnore
    });

    // 处理过滤后的文件
    const results = [];
    for (const file of filteredFiles) {
      try {
        const result = await this.processFile(file);
        results.push(result);
      } catch (error) {
        console.error(`Failed to process ${file}:`, error);
      }
    }

    return {
      processedCount: results.length,
      skippedCount: allFiles.length - filteredFiles.length,
      results
    };
  }

  private async findFilesByPatterns(patterns: string[]): Promise<string[]> {
    // 实现 glob 模式匹配
    return [];
  }

  private async processFile(filePath: string): Promise<any> {
    // 实现文件处理逻辑
    return {};
  }
}
```

## 高级功能

### 动态忽略规则管理

```typescript
class DynamicFileDiscoveryService extends FileDiscoveryService {
  private customIgnorePatterns: string[] = [];
  private tempIgnorePatterns: string[] = [];

  // 添加自定义忽略模式
  addCustomIgnorePattern(pattern: string): void {
    this.customIgnorePatterns.push(pattern);
  }

  // 移除自定义忽略模式
  removeCustomIgnorePattern(pattern: string): void {
    const index = this.customIgnorePatterns.indexOf(pattern);
    if (index > -1) {
      this.customIgnorePatterns.splice(index, 1);
    }
  }

  // 添加临时忽略模式（会话级别）
  addTempIgnorePattern(pattern: string): void {
    this.tempIgnorePatterns.push(pattern);
  }

  // 清理临时忽略模式
  clearTempIgnorePatterns(): void {
    this.tempIgnorePatterns = [];
  }

  // 重写忽略检查方法，包含自定义规则
  shouldIgnoreFile(
    filePath: string,
    options: FilterFilesOptions = {}
  ): boolean {
    // 检查基础规则
    if (super.shouldIgnoreFile(filePath, options)) {
      return true;
    }

    // 检查自定义规则
    if (this.matchesPatterns(filePath, this.customIgnorePatterns)) {
      return true;
    }

    // 检查临时规则
    if (this.matchesPatterns(filePath, this.tempIgnorePatterns)) {
      return true;
    }

    return false;
  }

  private matchesPatterns(filePath: string, patterns: string[]): boolean {
    return patterns.some(pattern => {
      // 简单的模式匹配，可以使用更复杂的 glob 匹配
      return filePath.includes(pattern) || 
             filePath.match(new RegExp(pattern.replace('*', '.*')));
    });
  }

  // 获取所有有效的忽略模式
  getAllIgnorePatterns(): {
    git: string[];
    gemini: string[];
    custom: string[];
    temp: string[];
  } {
    return {
      git: this.gitIgnoreFilter?.getPatterns() ?? [],
      gemini: this.getGeminiIgnorePatterns(),
      custom: [...this.customIgnorePatterns],
      temp: [...this.tempIgnorePatterns]
    };
  }
}
```

### 性能优化缓存

```typescript
class CachedFileDiscoveryService extends FileDiscoveryService {
  private cache = new Map<string, boolean>();
  private cacheHits = 0;
  private cacheMisses = 0;

  shouldIgnoreFile(
    filePath: string,
    options: FilterFilesOptions = {}
  ): boolean {
    const cacheKey = `${filePath}:${JSON.stringify(options)}`;
    
    if (this.cache.has(cacheKey)) {
      this.cacheHits++;
      return this.cache.get(cacheKey)!;
    }

    this.cacheMisses++;
    const result = super.shouldIgnoreFile(filePath, options);
    this.cache.set(cacheKey, result);
    
    // 限制缓存大小
    if (this.cache.size > 10000) {
      this.clearOldCacheEntries();
    }

    return result;
  }

  private clearOldCacheEntries(): void {
    // 简单的 LRU 策略：删除一半的条目
    const entries = Array.from(this.cache.entries());
    const halfSize = Math.floor(entries.length / 2);
    
    this.cache.clear();
    
    // 保留后一半的条目
    for (let i = halfSize; i < entries.length; i++) {
      this.cache.set(entries[i][0], entries[i][1]);
    }
  }

  getCacheStats(): { hits: number; misses: number; hitRate: number } {
    const total = this.cacheHits + this.cacheMisses;
    return {
      hits: this.cacheHits,
      misses: this.cacheMisses,
      hitRate: total > 0 ? this.cacheHits / total : 0
    };
  }

  clearCache(): void {
    this.cache.clear();
    this.cacheHits = 0;
    this.cacheMisses = 0;
  }
}
```

### 批量操作优化

```typescript
class BatchFileDiscoveryService extends FileDiscoveryService {
  // 批量过滤，提供详细的过滤信息
  filterFilesWithDetails(
    filePaths: string[],
    options: FilterFilesOptions = {}
  ): {
    included: string[];
    excluded: Array<{
      path: string;
      reason: 'git-ignored' | 'gemini-ignored' | 'both';
    }>;
    stats: {
      total: number;
      included: number;
      gitIgnored: number;
      geminiIgnored: number;
      bothIgnored: number;
    };
  } {
    const included: string[] = [];
    const excluded: Array<{
      path: string;
      reason: 'git-ignored' | 'gemini-ignored' | 'both';
    }> = [];

    let gitIgnoredCount = 0;
    let geminiIgnoredCount = 0;
    let bothIgnoredCount = 0;

    for (const filePath of filePaths) {
      const gitIgnored = options.respectGitIgnore && 
                        this.shouldGitIgnoreFile(filePath);
      const geminiIgnored = options.respectGeminiIgnore && 
                           this.shouldGeminiIgnoreFile(filePath);

      if (gitIgnored || geminiIgnored) {
        let reason: 'git-ignored' | 'gemini-ignored' | 'both';
        
        if (gitIgnored && geminiIgnored) {
          reason = 'both';
          bothIgnoredCount++;
        } else if (gitIgnored) {
          reason = 'git-ignored';
          gitIgnoredCount++;
        } else {
          reason = 'gemini-ignored';
          geminiIgnoredCount++;
        }

        excluded.push({ path: filePath, reason });
      } else {
        included.push(filePath);
      }
    }

    return {
      included,
      excluded,
      stats: {
        total: filePaths.length,
        included: included.length,
        gitIgnored: gitIgnoredCount,
        geminiIgnored: geminiIgnoredCount,
        bothIgnored: bothIgnoredCount,
      }
    };
  }

  // 并行文件检查（对于大量文件）
  async filterFilesParallel(
    filePaths: string[],
    options: FilterFilesOptions = {},
    batchSize: number = 100
  ): Promise<string[]> {
    const results: string[] = [];
    
    // 分批处理
    for (let i = 0; i < filePaths.length; i += batchSize) {
      const batch = filePaths.slice(i, i + batchSize);
      
      // 并行检查当前批次
      const batchPromises = batch.map(async (filePath) => {
        // 模拟异步检查（如果需要网络调用或复杂计算）
        const shouldIgnore = await new Promise<boolean>(resolve => {
          setImmediate(() => {
            resolve(this.shouldIgnoreFile(filePath, options));
          });
        });
        
        return shouldIgnore ? null : filePath;
      });

      const batchResults = await Promise.all(batchPromises);
      
      // 收集非空结果
      for (const result of batchResults) {
        if (result !== null) {
          results.push(result);
        }
      }
    }

    return results;
  }
}
```

### 监控和调试

```typescript
class InstrumentedFileDiscoveryService extends FileDiscoveryService {
  private metrics = {
    filterCalls: 0,
    gitIgnoreChecks: 0,
    geminiIgnoreChecks: 0,
    totalFilesProcessed: 0,
    totalFilesFiltered: 0,
    averageFilterTime: 0,
  };

  filterFiles(
    filePaths: string[],
    options: FilterFilesOptions = {}
  ): string[] {
    const startTime = Date.now();
    this.metrics.filterCalls++;
    this.metrics.totalFilesProcessed += filePaths.length;

    const result = super.filterFiles(filePaths, options);
    
    this.metrics.totalFilesFiltered += (filePaths.length - result.length);
    
    // 更新平均过滤时间
    const filterTime = Date.now() - startTime;
    this.updateAverageFilterTime(filterTime);

    return result;
  }

  shouldGitIgnoreFile(filePath: string): boolean {
    this.metrics.gitIgnoreChecks++;
    return super.shouldGitIgnoreFile(filePath);
  }

  shouldGeminiIgnoreFile(filePath: string): boolean {
    this.metrics.geminiIgnoreChecks++;
    return super.shouldGeminiIgnoreFile(filePath);
  }

  private updateAverageFilterTime(newTime: number): void {
    const currentAverage = this.metrics.averageFilterTime;
    const count = this.metrics.filterCalls;
    
    this.metrics.averageFilterTime = 
      (currentAverage * (count - 1) + newTime) / count;
  }

  getMetrics(): typeof this.metrics & {
    filterEfficiency: number;
    checksPerFile: number;
  } {
    return {
      ...this.metrics,
      filterEfficiency: this.metrics.totalFilesProcessed > 0 
        ? this.metrics.totalFilesFiltered / this.metrics.totalFilesProcessed
        : 0,
      checksPerFile: this.metrics.totalFilesProcessed > 0
        ? (this.metrics.gitIgnoreChecks + this.metrics.geminiIgnoreChecks) / this.metrics.totalFilesProcessed
        : 0,
    };
  }

  resetMetrics(): void {
    this.metrics = {
      filterCalls: 0,
      gitIgnoreChecks: 0,
      geminiIgnoreChecks: 0,
      totalFilesProcessed: 0,
      totalFilesFiltered: 0,
      averageFilterTime: 0,
    };
  }

  // 调试模式下的详细日志
  enableDebugMode(): void {
    const originalFilterFiles = this.filterFiles.bind(this);
    
    this.filterFiles = (filePaths: string[], options: FilterFilesOptions = {}) => {
      console.log(`[FileDiscoveryService] Filtering ${filePaths.length} files with options:`, options);
      
      const result = originalFilterFiles(filePaths, options);
      
      console.log(`[FileDiscoveryService] Filtered result: ${result.length} files included, ${filePaths.length - result.length} files excluded`);
      
      return result;
    };
  }
}
```

## 错误处理

### 常见错误场景

```typescript
// Git 仓库不存在或无法访问
Error: Not a git repository
Error: Permission denied reading .gitignore

// 忽略文件格式错误  
Error: Invalid pattern in .gitignore: [invalid-pattern]
Error: Malformed .geminiignore file

// 文件系统错误
Error: ENOENT: no such file or directory
Error: EMFILE: too many open files
```

### 错误恢复策略

```typescript
class RobustFileDiscoveryService extends FileDiscoveryService {
  constructor(projectRoot: string) {
    try {
      super(projectRoot);
    } catch (error) {
      console.warn('FileDiscoveryService initialization warning:', error);
      this.initializeFallback(projectRoot);
    }
  }

  private initializeFallback(projectRoot: string): void {
    this.projectRoot = path.resolve(projectRoot);
    
    // 创建基本的过滤器，即使 Git 仓库不存在
    this.gitIgnoreFilter = null;
    this.geminiIgnoreFilter = null;
    
    console.log('FileDiscoveryService initialized in fallback mode');
  }

  shouldGitIgnoreFile(filePath: string): boolean {
    try {
      return super.shouldGitIgnoreFile(filePath);
    } catch (error) {
      console.warn(`Git ignore check failed for ${filePath}:`, error);
      return false; // 默认不忽略
    }
  }

  shouldGeminiIgnoreFile(filePath: string): boolean {
    try {
      return super.shouldGeminiIgnoreFile(filePath);
    } catch (error) {
      console.warn(`Gemini ignore check failed for ${filePath}:`, error);
      return false; // 默认不忽略
    }
  }

  filterFiles(
    filePaths: string[],
    options: FilterFilesOptions = {}
  ): string[] {
    try {
      return super.filterFiles(filePaths, options);
    } catch (error) {
      console.error('Batch filtering failed, falling back to individual checks:', error);
      
      // 降级为逐个检查
      return filePaths.filter(filePath => {
        try {
          return !this.shouldIgnoreFile(filePath, options);
        } catch (fileError) {
          console.warn(`Individual file check failed for ${filePath}:`, fileError);
          return true; // 默认包含文件
        }
      });
    }
  }

  // 重新加载忽略文件
  reloadIgnoreFiles(): { success: boolean; errors: string[] } {
    const errors: string[] = [];
    let success = true;

    try {
      // 重新加载 Git 忽略文件
      if (isGitRepository(this.projectRoot)) {
        const parser = new GitIgnoreParser(this.projectRoot);
        parser.loadGitRepoPatterns();
        this.gitIgnoreFilter = parser;
      }
    } catch (error) {
      errors.push(`Failed to reload .gitignore: ${error}`);
      success = false;
    }

    try {
      // 重新加载 Gemini 忽略文件
      const gParser = new GitIgnoreParser(this.projectRoot);
      gParser.loadPatterns(GEMINI_IGNORE_FILE_NAME);
      this.geminiIgnoreFilter = gParser;
    } catch (error) {
      errors.push(`Failed to reload .geminiignore: ${error}`);
      success = false;
    }

    return { success, errors };
  }
}
```

## 集成示例

### 与 Config 系统集成

```typescript
class ConfigurableFileDiscoveryService extends FileDiscoveryService {
  constructor(
    projectRoot: string,
    private config: Config
  ) {
    super(projectRoot);
  }

  filterFiles(
    filePaths: string[],
    options?: FilterFilesOptions
  ): string[] {
    // 使用配置中的默认选项
    const effectiveOptions: FilterFilesOptions = {
      respectGitIgnore: options?.respectGitIgnore ?? 
                       this.config.getFileFilteringRespectGitIgnore(),
      respectGeminiIgnore: options?.respectGeminiIgnore ?? 
                          this.config.getFileFilteringRespectGeminiIgnore(),
    };

    return super.filterFiles(filePaths, effectiveOptions);
  }

  // 获取配置中的忽略模式
  getConfigIgnorePatterns(): string[] {
    return this.config.getGlobalIgnorePatterns() || [];
  }

  // 检查文件是否被配置忽略
  isConfigIgnored(filePath: string): boolean {
    const patterns = this.getConfigIgnorePatterns();
    return patterns.some(pattern => {
      return filePath.match(new RegExp(pattern.replace('*', '.*')));
    });
  }

  // 重写统一检查方法，包含配置规则
  shouldIgnoreFile(
    filePath: string,
    options: FilterFilesOptions = {}
  ): boolean {
    // 检查基础规则
    if (super.shouldIgnoreFile(filePath, options)) {
      return true;
    }

    // 检查配置规则
    if (this.isConfigIgnored(filePath)) {
      return true;
    }

    return false;
  }
}
```

### 与工具系统集成

```typescript
// 在工具中使用 FileDiscoveryService
class FileReadTool extends BaseTool<FileReadParams, ToolResult> {
  constructor(
    private config: Config,
    private fileService: FileDiscoveryService
  ) {
    super(/* tool parameters */);
  }

  async execute(
    params: FileReadParams,
    signal: AbortSignal
  ): Promise<ToolResult> {
    try {
      // 查找匹配的文件
      const allFiles = await this.findFiles(params.patterns);
      
      // 应用过滤规则
      const filteredFiles = this.fileService.filterFiles(allFiles, {
        respectGitIgnore: params.respectGitIgnore,
        respectGeminiIgnore: params.respectGeminiIgnore
      });

      // 处理过滤后的文件
      const results = await this.processFiles(filteredFiles, signal);
      
      return {
        summary: `Processed ${filteredFiles.length} files`,
        llmContent: results.content,
        returnDisplay: this.formatResults(results, allFiles.length - filteredFiles.length)
      };
    } catch (error) {
      return this.handleError(error);
    }
  }

  private formatResults(results: any, skippedCount: number): string {
    let display = `## File Processing Results\n\n`;
    display += `**Processed**: ${results.processedCount} files\n`;
    
    if (skippedCount > 0) {
      display += `**Skipped**: ${skippedCount} files (filtered by ignore rules)\n`;
    }
    
    return display;
  }
}
```

## 最佳实践

### 性能优化

```typescript
// ✅ 推荐：预过滤大量文件
async function efficientFileProcessing(
  patterns: string[],
  fileService: FileDiscoveryService
): Promise<void> {
  // 先使用快速模式获取文件列表
  const allFiles = await fastGlob(patterns, {
    onlyFiles: true,
    ignore: ['node_modules/**', '.git/**'] // 基础过滤
  });

  // 然后应用详细的忽略规则
  const filteredFiles = fileService.filterFiles(allFiles);
  
  // 处理过滤后的文件
  await processFiles(filteredFiles);
}

// ✅ 推荐：批量检查减少调用开销
const batchedFiles = [
  'src/file1.ts', 'src/file2.ts', 'src/file3.ts'
];

// 好：一次性过滤
const filtered = fileService.filterFiles(batchedFiles);

// 避免：逐个检查
// batchedFiles.forEach(file => fileService.shouldIgnoreFile(file));
```

### 错误处理

```typescript
// ✅ 推荐：优雅的错误处理
function safeFileFilter(
  files: string[],
  fileService: FileDiscoveryService
): { filtered: string[]; errors: string[] } {
  const filtered: string[] = [];
  const errors: string[] = [];

  try {
    return {
      filtered: fileService.filterFiles(files),
      errors: []
    };
  } catch (error) {
    // 降级为单文件检查
    for (const file of files) {
      try {
        if (!fileService.shouldIgnoreFile(file)) {
          filtered.push(file);
        }
      } catch (fileError) {
        errors.push(`Failed to check ${file}: ${fileError}`);
        filtered.push(file); // 默认包含
      }
    }

    return { filtered, errors };
  }
}
```

### 配置管理

```typescript
// ✅ 推荐：集中配置管理
interface FileDiscoveryConfig {
  respectGitIgnore: boolean;
  respectGeminiIgnore: boolean;
  customIgnorePatterns: string[];
  enableCache: boolean;
  cacheSize: number;
}

class ConfiguredFileDiscoveryService {
  constructor(
    projectRoot: string,
    private config: FileDiscoveryConfig
  ) {
    // 根据配置初始化服务
  }

  createFilterOptions(): FilterFilesOptions {
    return {
      respectGitIgnore: this.config.respectGitIgnore,
      respectGeminiIgnore: this.config.respectGeminiIgnore
    };
  }
}
```