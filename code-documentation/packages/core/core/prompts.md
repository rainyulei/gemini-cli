# Prompts - 系统提示词管理文档

## 概述

`prompts.ts` 是 Gemini CLI 的系统提示词管理模块，负责生成和管理与 Gemini AI 模型交互所需的系统指令。它包含核心系统提示词、历史压缩提示词，支持环境变量配置、用户记忆集成，以及动态上下文感知的提示词生成。

## 主要功能

- **核心系统提示词**: 定义 AI 助手的行为准则、工作流程和交互规范
- **环境变量配置**: 支持通过环境变量自定义系统提示词文件
- **用户记忆集成**: 将用户个人偏好和记忆信息融入系统提示
- **上下文感知**: 根据项目环境（Git、沙箱等）动态调整提示内容
- **历史压缩**: 提供专门的历史记录压缩提示词
- **安全考虑**: 包含详细的安全和权限指导原则

## 核心函数

### `getCoreSystemPrompt` - 核心系统提示词生成

```typescript
export function getCoreSystemPrompt(userMemory?: string): string
```

**功能**: 生成包含完整行为指导的核心系统提示词

**参数**:
- `userMemory`: 可选的用户记忆内容，用于个性化提示

**返回值**: 完整的系统提示词字符串

### 环境变量配置系统

#### `GEMINI_SYSTEM_MD` - 系统提示词文件覆盖

**功能**: 允许用户通过外部文件覆盖默认系统提示词

**配置选项**:
```bash
# 禁用自定义系统提示词
export GEMINI_SYSTEM_MD=0
export GEMINI_SYSTEM_MD=false

# 启用默认路径 (~/.gemini/system.md)
export GEMINI_SYSTEM_MD=1
export GEMINI_SYSTEM_MD=true

# 指定自定义文件路径
export GEMINI_SYSTEM_MD="/path/to/custom/system.md"
export GEMINI_SYSTEM_MD="~/my-gemini/custom-prompt.md"
```

**实现逻辑**:
```typescript
let systemMdEnabled = false;
let systemMdPath = path.resolve(path.join(GEMINI_CONFIG_DIR, 'system.md'));
const systemMdVar = process.env.GEMINI_SYSTEM_MD;

if (systemMdVar) {
  const systemMdVarLower = systemMdVar.toLowerCase();
  if (!['0', 'false'].includes(systemMdVarLower)) {
    systemMdEnabled = true;
    
    // 处理自定义路径
    if (!['1', 'true'].includes(systemMdVarLower)) {
      let customPath = systemMdVar;
      if (customPath.startsWith('~/')) {
        customPath = path.join(os.homedir(), customPath.slice(2));
      } else if (customPath === '~') {
        customPath = os.homedir();
      }
      systemMdPath = path.resolve(customPath);
    }
    
    // 验证文件存在
    if (!fs.existsSync(systemMdPath)) {
      throw new Error(`missing system prompt file '${systemMdPath}'`);
    }
  }
}

// 根据配置加载提示词
const basePrompt = systemMdEnabled
  ? fs.readFileSync(systemMdPath, 'utf8')
  : DEFAULT_SYSTEM_PROMPT;
```

#### `GEMINI_WRITE_SYSTEM_MD` - 系统提示词导出

**功能**: 将默认系统提示词导出到文件，便于用户自定义

**配置选项**:
```bash
# 导出到默认路径
export GEMINI_WRITE_SYSTEM_MD=1
export GEMINI_WRITE_SYSTEM_MD=true

# 导出到自定义路径
export GEMINI_WRITE_SYSTEM_MD="/path/to/export/system.md"
export GEMINI_WRITE_SYSTEM_MD="~/my-prompts/base-system.md"
```

**使用示例**:
```bash
# 导出默认系统提示词到文件
export GEMINI_WRITE_SYSTEM_MD=true
gemini-cli --help  # 任何命令都会触发导出

# 检查导出的文件
cat ~/.gemini/system.md

# 自定义并重新加载
export GEMINI_SYSTEM_MD=~/.gemini/system.md
# 编辑文件后重启 CLI
```

## 核心系统提示词结构

### 核心指导原则 (Core Mandates)

```markdown
# Core Mandates

- **Conventions:** 严格遵循现有项目约定
- **Libraries/Frameworks:** 从不假设库/框架可用性
- **Style & Structure:** 模仿现有代码的风格和结构
- **Idiomatic Changes:** 确保变更自然且符合语言习惯
- **Comments:** 谨慎添加代码注释，专注于"为什么"而非"什么"
- **Proactiveness:** 彻底完成用户请求，包括合理的后续行动
- **Confirm Ambiguity/Expansion:** 在超出明确范围前确认用户意图
- **Explaining Changes:** 完成代码修改后不提供摘要除非被要求
- **Path Construction:** 构建绝对路径用于文件系统工具
- **Do Not revert changes:** 除非明确要求否则不回滚变更
```

### 主要工作流程 (Primary Workflows)

#### 软件工程任务流程

```markdown
## Software Engineering Tasks

1. **Understand:** 理解用户请求和相关代码库上下文
   - 广泛使用搜索工具 (grep, glob)
   - 使用 read-file 和 read-many-files 理解上下文
   
2. **Plan:** 构建连贯且有根据的计划
   - 与用户分享简洁清晰的计划
   - 包含自验证循环（单元测试）
   
3. **Implement:** 使用可用工具执行计划
   - 严格遵循项目约定
   - 使用 edit, write-file, shell 等工具
   
4. **Verify (Tests):** 验证变更
   - 识别正确的测试命令和框架
   - 检查 README、package.json 等配置
   
5. **Verify (Standards):** 执行项目特定的构建、代码检查和类型检查
   - 运行 tsc, npm run lint, ruff check 等命令
```

#### 新应用开发流程

```markdown
## New Applications

**Goal:** 自主实现功能完整、视觉吸引的原型

1. **Understand Requirements:** 分析核心功能、UX、视觉美学
2. **Propose Plan:** 制定开发计划并呈现给用户
3. **User Approval:** 获得用户批准
4. **Implementation:** 自主实现每个功能和设计元素
5. **Verify:** 检查工作质量，修复错误和偏差
6. **Solicit Feedback:** 提供启动说明并请求用户反馈
```

**推荐技术栈**:
```markdown
- **Websites (Frontend):** React (JavaScript/TypeScript) + Bootstrap CSS + Material Design
- **Back-End APIs:** Node.js + Express.js 或 Python + FastAPI
- **Full-stack:** Next.js 或 Python (Django/Flask) + React/Vue.js
- **CLIs:** Python 或 Go
- **Mobile App:** Compose Multiplatform (Kotlin) 或 Flutter (Dart)
- **3D Games:** HTML/CSS/JavaScript + Three.js
- **2D Games:** HTML/CSS/JavaScript
```

### 操作指导 (Operational Guidelines)

#### 语调和风格

```markdown
## Tone and Style (CLI Interaction)

- **Concise & Direct:** 专业、直接、简洁的 CLI 环境语调
- **Minimal Output:** 每次响应少于3行文本输出
- **Clarity over Brevity:** 必要时优先考虑清晰度
- **No Chitchat:** 避免对话填充和前后缀
- **Formatting:** 使用 GitHub 风格的 Markdown
- **Tools vs. Text:** 工具用于行动，文本仅用于沟通
```

#### 安全和安全规则

```markdown
## Security and Safety Rules

- **Explain Critical Commands:** 执行系统修改命令前简要解释目的和影响
- **Security First:** 始终应用安全最佳实践
- 从不引入暴露、记录或提交机密信息的代码
```

#### 工具使用指南

```markdown
## Tool Usage

- **File Paths:** 始终使用绝对路径
- **Parallelism:** 可行时并行执行多个独立工具调用
- **Command Execution:** 使用 shell 工具运行 shell 命令
- **Background Processes:** 对不太可能自行停止的命令使用后台进程
- **Interactive Commands:** 避免需要用户交互的 shell 命令
- **Remembering Facts:** 使用 memory 工具记住用户相关事实
- **Respect User Confirmations:** 尊重用户的确认选择
```

### 动态上下文集成

#### 沙箱环境检测

```typescript
// 沙箱状态检测
const isSandboxExec = process.env.SANDBOX === 'sandbox-exec';
const isGenericSandbox = !!process.env.SANDBOX;

if (isSandboxExec) {
  // macOS Seatbelt 特定指导
  return `# macOS Seatbelt
You are running under macos seatbelt with limited access...`;
} else if (isGenericSandbox) {
  // 通用沙箱指导
  return `# Sandbox
You are running in a sandbox container...`;
} else {
  // 非沙箱环境警告
  return `# Outside of Sandbox
You are running outside of a sandbox container...`;
}
```

#### Git 仓库集成

```typescript
// Git 仓库检测和指导
if (isGitRepository(process.cwd())) {
  return `# Git Repository
- The current working directory is being managed by a git repository.
- When asked to commit changes:
  - \`git status\` to ensure files are tracked and staged
  - \`git diff HEAD\` to review all changes
  - \`git log -n 3\` to review recent commit messages
- Always propose a draft commit message
- Never push changes without explicit user request`;
}
```

### 用户记忆集成

```typescript
// 用户记忆后缀添加
const memorySuffix =
  userMemory && userMemory.trim().length > 0
    ? `\n\n---\n\n${userMemory.trim()}`
    : '';

return `${basePrompt}${memorySuffix}`;
```

**用户记忆示例**:
```
---

User prefers:
- TypeScript over JavaScript
- Functional programming style
- Jest for testing
- Detailed commit messages

User's project context:
- Primary framework: React with Next.js
- Database: PostgreSQL with Prisma
- Deployment: Vercel
```

## 历史压缩提示词

### `getCompressionPrompt` - 历史压缩提示

```typescript
export function getCompressionPrompt(): string
```

**功能**: 生成用于对话历史压缩的专门提示词

**压缩流程**:
1. **私有思考**: 在 `<scratchpad>` 中分析整个历史记录
2. **信息提取**: 识别用户目标、代理行动、工具输出、文件修改
3. **结构化输出**: 生成密集信息的 XML 快照

**输出结构**:
```xml
<state_snapshot>
    <overall_goal>
        <!-- 用户高级目标的单句描述 -->
    </overall_goal>

    <key_knowledge>
        <!-- 基于对话历史的关键事实、约定和约束 -->
        <!-- 示例:
         - Build Command: `npm run build`
         - Testing: Tests are run with `npm test`
         - API Endpoint: `https://api.example.com/v2`
        -->
    </key_knowledge>

    <file_system_state>
        <!-- 已创建、读取、修改或删除的文件列表 -->
        <!-- 示例:
         - CWD: `/home/user/project/src`
         - READ: `package.json` - Confirmed 'axios' dependency
         - MODIFIED: `services/auth.ts` - Replaced 'jsonwebtoken'
         - CREATED: `tests/new-feature.test.ts` - Initial test structure
        -->
    </file_system_state>

    <recent_actions>
        <!-- 最近几个重要代理行动及其结果的摘要 -->
        <!-- 示例:
         - Ran `grep 'old_function'` - returned 3 results in 2 files
         - Ran `npm run test` - failed due to snapshot mismatch
         - Ran `ls -F static/` - discovered .webp image assets
        -->
    </recent_actions>

    <current_plan>
        <!-- 代理的逐步计划，标记已完成的步骤 -->
        <!-- 示例:
         1. [DONE] Identify files using deprecated 'UserAPI'
         2. [IN PROGRESS] Refactor `UserProfile.tsx`
         3. [TODO] Refactor remaining files
         4. [TODO] Update tests
        -->
    </current_plan>
</state_snapshot>
```

## 使用示例

### 基本系统提示词使用

```typescript
import { getCoreSystemPrompt } from './prompts.js';

// 基本使用
const systemPrompt = getCoreSystemPrompt();
console.log('System prompt length:', systemPrompt.length);

// 带用户记忆
const userMemory = `
User preferences:
- Prefers TypeScript over JavaScript
- Uses functional programming style
- Likes detailed commit messages

Project context:
- Main framework: React with Next.js
- Testing: Jest and React Testing Library
- Styling: Tailwind CSS
`;

const personalizedPrompt = getCoreSystemPrompt(userMemory);
console.log('Personalized prompt includes user memory');
```

### 环境变量配置示例

```bash
#!/bin/bash

# 示例 1: 使用自定义系统提示词文件
echo "Creating custom system prompt..."
mkdir -p ~/.gemini
cat > ~/.gemini/custom-system.md << 'EOF'
You are a specialized TypeScript development assistant.

Your primary focus is:
- Writing type-safe TypeScript code
- Following functional programming principles
- Using modern ES6+ features
- Implementing comprehensive testing

Always prioritize code quality and maintainability.
EOF

# 启用自定义提示词
export GEMINI_SYSTEM_MD=~/.gemini/custom-system.md

# 运行 CLI
gemini-cli "Help me refactor this function to be more type-safe"
```

```bash
# 示例 2: 导出默认提示词进行自定义
export GEMINI_WRITE_SYSTEM_MD=~/my-prompts/base-system.md
gemini-cli --version  # 触发导出

# 编辑导出的文件
nano ~/my-prompts/base-system.md

# 使用编辑后的文件
export GEMINI_SYSTEM_MD=~/my-prompts/base-system.md
gemini-cli "Let's start coding"
```

### 动态提示词生成

```typescript
// 模拟不同环境下的提示词生成
function demonstrateContextualPrompts() {
  // 模拟 Git 仓库环境
  const originalCwd = process.cwd();
  
  // 情况 1: 非 Git 环境
  console.log('=== Non-Git Environment ===');
  const nonGitPrompt = getCoreSystemPrompt();
  console.log('Git section included:', nonGitPrompt.includes('Git Repository'));
  
  // 情况 2: 沙箱环境
  console.log('\n=== Sandbox Environment ===');
  process.env.SANDBOX = 'true';
  const sandboxPrompt = getCoreSystemPrompt();
  console.log('Sandbox section included:', sandboxPrompt.includes('Sandbox'));
  delete process.env.SANDBOX;
  
  // 情况 3: macOS Seatbelt 环境
  console.log('\n=== macOS Seatbelt Environment ===');
  process.env.SANDBOX = 'sandbox-exec';
  const seatbeltPrompt = getCoreSystemPrompt();
  console.log('Seatbelt section included:', seatbeltPrompt.includes('macOS Seatbelt'));
  delete process.env.SANDBOX;
  
  // 情况 4: 带用户记忆
  console.log('\n=== With User Memory ===');
  const userMemory = 'User prefers Python over JavaScript for backend development.';
  const memoryPrompt = getCoreSystemPrompt(userMemory);
  console.log('User memory included:', memoryPrompt.includes(userMemory));
}

demonstrateContextualPrompts();
```

### 历史压缩使用

```typescript
import { getCompressionPrompt } from './prompts.js';

// 历史压缩提示词使用
async function demonstrateCompression() {
  const compressionPrompt = getCompressionPrompt();
  
  console.log('Compression prompt structure:');
  console.log('- Contains scratchpad instruction:', 
    compressionPrompt.includes('<scratchpad>'));
  console.log('- Contains state_snapshot structure:', 
    compressionPrompt.includes('<state_snapshot>'));
  console.log('- Includes all required sections:', 
    ['overall_goal', 'key_knowledge', 'file_system_state', 'recent_actions', 'current_plan']
      .every(section => compressionPrompt.includes(section)));
  
  // 模拟压缩过程中的使用
  const mockChatHistory = [
    { role: 'user', content: 'Help me refactor auth.js to use async/await' },
    { role: 'assistant', content: 'I can help with that. Let me first read the file...' },
    // ... more history
  ];
  
  console.log('\nSimulating compression for chat with', mockChatHistory.length, 'messages');
  console.log('Compression prompt ready for use in chat compression.');
}

demonstrateCompression();
```

### 提示词自定义最佳实践

```typescript
// 创建项目特定的提示词管理器
class ProjectPromptManager {
  private projectRoot: string;
  private customPromptPath: string;

  constructor(projectRoot: string) {
    this.projectRoot = projectRoot;
    this.customPromptPath = path.join(projectRoot, '.gemini', 'project-prompt.md');
  }

  // 生成项目特定的系统提示词
  async generateProjectPrompt(): Promise<string> {
    const basePrompt = getCoreSystemPrompt();
    const projectContext = await this.analyzeProjectContext();
    
    return `${basePrompt}

# Project-Specific Context

${projectContext}`;
  }

  // 分析项目上下文
  private async analyzeProjectContext(): Promise<string> {
    const context: string[] = [];
    
    // 检查包管理器
    if (fs.existsSync(path.join(this.projectRoot, 'package.json'))) {
      const packageJson = JSON.parse(
        fs.readFileSync(path.join(this.projectRoot, 'package.json'), 'utf8')
      );
      
      context.push(`## Node.js Project`);
      context.push(`- Package manager: npm/yarn`);
      
      if (packageJson.scripts) {
        context.push(`- Available scripts: ${Object.keys(packageJson.scripts).join(', ')}`);
      }
      
      if (packageJson.dependencies?.typescript || packageJson.devDependencies?.typescript) {
        context.push(`- TypeScript project`);
      }
      
      if (packageJson.dependencies?.react || packageJson.devDependencies?.react) {
        context.push(`- React framework`);
      }
    }
    
    // 检查 Python 项目
    if (fs.existsSync(path.join(this.projectRoot, 'requirements.txt')) ||
        fs.existsSync(path.join(this.projectRoot, 'pyproject.toml'))) {
      context.push(`## Python Project`);
      context.push(`- Python package management detected`);
    }
    
    // 检查测试框架
    const testFiles = this.findTestFiles();
    if (testFiles.length > 0) {
      context.push(`## Testing`);
      context.push(`- Test files found: ${testFiles.length}`);
      context.push(`- Test patterns: ${this.detectTestPatterns(testFiles)}`);
    }
    
    return context.join('\n');
  }

  private findTestFiles(): string[] {
    // 简化的测试文件查找
    const testPatterns = ['**/*.test.*', '**/*.spec.*', 'test/**/*', 'tests/**/*'];
    const testFiles: string[] = [];
    
    // 实际实现会使用 glob 库
    // 这里仅作示例
    return testFiles;
  }

  private detectTestPatterns(testFiles: string[]): string {
    const patterns = new Set<string>();
    
    testFiles.forEach(file => {
      if (file.includes('.test.')) patterns.add('*.test.*');
      if (file.includes('.spec.')) patterns.add('*.spec.*');
      if (file.startsWith('test/')) patterns.add('test/**');
      if (file.startsWith('tests/')) patterns.add('tests/**');
    });
    
    return Array.from(patterns).join(', ');
  }

  // 保存自定义提示词
  async saveCustomPrompt(prompt: string): Promise<void> {
    const dir = path.dirname(this.customPromptPath);
    if (!fs.existsSync(dir)) {
      fs.mkdirSync(dir, { recursive: true });
    }
    
    fs.writeFileSync(this.customPromptPath, prompt, 'utf8');
    console.log(`Custom prompt saved to: ${this.customPromptPath}`);
  }

  // 加载自定义提示词
  loadCustomPrompt(): string | null {
    if (fs.existsSync(this.customPromptPath)) {
      return fs.readFileSync(this.customPromptPath, 'utf8');
    }
    return null;
  }
}

// 使用示例
async function useProjectPromptManager() {
  const projectManager = new ProjectPromptManager(process.cwd());
  
  // 生成项目特定提示词
  const projectPrompt = await projectManager.generateProjectPrompt();
  
  // 保存到文件
  await projectManager.saveCustomPrompt(projectPrompt);
  
  // 设置环境变量使用自定义提示词
  process.env.GEMINI_SYSTEM_MD = path.join(process.cwd(), '.gemini', 'project-prompt.md');
  
  // 现在使用自定义提示词
  const customizedPrompt = getCoreSystemPrompt();
  console.log('Using project-specific prompt:', customizedPrompt.length, 'characters');
}
```

### 提示词版本管理

```typescript
// 提示词版本管理系统
class PromptVersionManager {
  private versionHistory: Map<string, string> = new Map();
  private currentVersion: string = '1.0.0';

  // 保存提示词版本
  saveVersion(version: string, prompt: string): void {
    this.versionHistory.set(version, prompt);
    this.currentVersion = version;
    
    console.log(`Prompt version ${version} saved`);
  }

  // 获取指定版本的提示词
  getVersion(version: string): string | undefined {
    return this.versionHistory.get(version);
  }

  // 比较版本差异
  compareVersions(version1: string, version2: string): {
    added: string[];
    removed: string[];
    changed: string[];
  } {
    const prompt1 = this.getVersion(version1);
    const prompt2 = this.getVersion(version2);
    
    if (!prompt1 || !prompt2) {
      throw new Error('One or both versions not found');
    }

    // 简化的差异分析
    const lines1 = prompt1.split('\n');
    const lines2 = prompt2.split('\n');
    
    const added = lines2.filter(line => !lines1.includes(line));
    const removed = lines1.filter(line => !lines2.includes(line));
    const changed = added.filter(line => 
      removed.some(removedLine => 
        line.substring(0, 20) === removedLine.substring(0, 20)
      )
    );

    return { added, removed, changed };
  }

  // 回滚到指定版本
  rollbackTo(version: string): string {
    const prompt = this.getVersion(version);
    if (!prompt) {
      throw new Error(`Version ${version} not found`);
    }

    this.currentVersion = version;
    console.log(`Rolled back to version ${version}`);
    return prompt;
  }

  // 列出所有版本
  listVersions(): string[] {
    return Array.from(this.versionHistory.keys()).sort();
  }
}

// 使用版本管理
const versionManager = new PromptVersionManager();

// 保存不同版本的提示词
versionManager.saveVersion('1.0.0', getCoreSystemPrompt());

// 创建用户特定版本
const userMemory = 'User prefers functional programming';
versionManager.saveVersion('1.1.0-user', getCoreSystemPrompt(userMemory));

// 比较版本
const diff = versionManager.compareVersions('1.0.0', '1.1.0-user');
console.log('Version differences:', diff);

// 列出所有版本
console.log('Available versions:', versionManager.listVersions());
```

## 最佳实践

### 1. 环境变量配置
```bash
# ✅ 推荐：渐进式自定义流程
# 1. 先导出默认提示词
export GEMINI_WRITE_SYSTEM_MD=~/my-prompts/base.md
gemini-cli --version

# 2. 编辑和自定义
nano ~/my-prompts/base.md

# 3. 启用自定义提示词
export GEMINI_SYSTEM_MD=~/my-prompts/base.md

# 4. 测试自定义效果
gemini-cli "Test custom prompt behavior"
```

### 2. 项目特定配置
```typescript
// ✅ 推荐：项目级别的提示词配置
// 在项目根目录创建 .gemini/system.md
const projectPromptPath = path.join(process.cwd(), '.gemini', 'system.md');

if (fs.existsSync(projectPromptPath)) {
  process.env.GEMINI_SYSTEM_MD = projectPromptPath;
}
```

### 3. 用户记忆最佳实践
```typescript
// ✅ 推荐：结构化用户记忆
const structuredUserMemory = `
## Coding Preferences
- Language: TypeScript preferred over JavaScript
- Style: Functional programming with immutable data
- Testing: Jest with comprehensive test coverage
- Documentation: JSDoc comments for public APIs

## Project Context  
- Framework: React with Next.js
- Database: PostgreSQL with Prisma ORM
- Styling: Tailwind CSS with component library
- Deployment: Vercel with automated CI/CD

## Communication Style
- Prefers detailed explanations for complex topics
- Likes step-by-step implementation guides
- Values security and performance considerations
`;

const prompt = getCoreSystemPrompt(structuredUserMemory);
```

### 4. 压缩提示词优化
```typescript
// ✅ 推荐：定期评估压缩质量
function evaluateCompressionQuality(
  originalHistory: any[],
  compressedState: string
): {
  informationRetention: number;
  contextPreservation: number;
  actionableContent: number;
} {
  // 实现压缩质量评估逻辑
  const keyElements = [
    'overall_goal',
    'key_knowledge', 
    'file_system_state',
    'recent_actions',
    'current_plan'
  ];

  const retention = keyElements.filter(element => 
    compressedState.includes(element)
  ).length / keyElements.length;

  return {
    informationRetention: retention,
    contextPreservation: 0.85, // 基于内容分析
    actionableContent: 0.90,    // 基于计划完整性
  };
}
```

### 5. 安全考虑
```typescript
// ✅ 推荐：提示词安全验证
function validatePromptSecurity(prompt: string): {
  safe: boolean;
  issues: string[];
} {
  const issues: string[] = [];
  
  // 检查敏感信息泄露
  if (prompt.includes('password') || prompt.includes('secret')) {
    issues.push('Contains potentially sensitive keywords');
  }
  
  // 检查系统命令注入风险
  if (prompt.includes('rm -rf') || prompt.includes('sudo')) {
    issues.push('Contains potentially dangerous system commands');
  }
  
  // 检查路径遍历风险
  if (prompt.includes('../') || prompt.includes('..\\')) {
    issues.push('Contains path traversal patterns');
  }

  return {
    safe: issues.length === 0,
    issues
  };
}

// 使用安全验证
const prompt = getCoreSystemPrompt();
const security = validatePromptSecurity(prompt);

if (!security.safe) {
  console.warn('Prompt security issues:', security.issues);
}
```

## 提示词架构说明

Gemini CLI 的提示词系统采用分层架构：

1. **基础层**: 核心行为准则和工作流程定义
2. **环境层**: 基于运行环境的动态内容注入
3. **项目层**: 项目特定的上下文和约定
4. **用户层**: 个人偏好和记忆信息
5. **压缩层**: 历史记录压缩的专门提示词

这种架构确保了提示词的灵活性、可扩展性和个性化能力，同时保持了系统的一致性和安全性。通过环境变量配置和用户记忆集成，每个用户都可以拥有量身定制的 AI 助手体验。