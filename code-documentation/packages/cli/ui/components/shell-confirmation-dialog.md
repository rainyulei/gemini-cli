# ShellConfirmationDialog Shell 命令确认对话框文档

## 概述

`ShellConfirmationDialog` 是 Gemini CLI 的 Shell 命令执行确认对话框组件，当自定义命令或工具需要执行 Shell 命令时显示。它提供安全的用户交互界面，让用户能够审查将要执行的命令并做出明智的决定，支持一次性批准、会话级批准和取消操作。

## 主要功能

- Shell 命令预览和确认
- 多种批准模式（一次性、会话级、取消）
- 键盘导航和快捷键支持
- 安全的命令展示和边框样式
- 与 Ink 终端 UI 框架完全集成
- 响应式布局和颜色主题

## 接口定义

### `ShellConfirmationRequest`

**功能**: Shell 命令确认请求接口

```typescript
export interface ShellConfirmationRequest {
  /** 要执行的命令列表 */
  commands: string[];
  /** 确认结果回调函数 */
  onConfirm: (
    outcome: ToolConfirmationOutcome,
    approvedCommands?: string[],
  ) => void;
}
```

**字段详情**:

- `commands`: 将要执行的 Shell 命令数组，每个命令作为独立字符串
- `onConfirm`: 用户做出选择后的回调函数
  - `outcome`: 确认结果类型（ProceedOnce、ProceedAlways、Cancel）
  - `approvedCommands`: 当用户批准时，返回批准的命令列表

### `ShellConfirmationDialogProps`

**功能**: 对话框组件属性接口

```typescript
export interface ShellConfirmationDialogProps {
  /** 确认请求对象 */
  request: ShellConfirmationRequest;
}
```

## 核心组件实现

### `ShellConfirmationDialog` 组件

```typescript
export const ShellConfirmationDialog: React.FC<ShellConfirmationDialogProps> = ({ 
  request 
}) => {
  const { commands, onConfirm } = request;

  // 键盘输入处理
  useInput((_, key) => {
    if (key.escape) {
      onConfirm(ToolConfirmationOutcome.Cancel);
    }
  });

  // 选择处理逻辑
  const handleSelect = (item: ToolConfirmationOutcome) => {
    if (item === ToolConfirmationOutcome.Cancel) {
      onConfirm(item);
    } else {
      // 对于 ProceedOnce 和 ProceedAlways，批准所有请求的命令
      onConfirm(item, commands);
    }
  };

  // 选项配置
  const options: Array<RadioSelectItem<ToolConfirmationOutcome>> = [
    {
      label: 'Yes, allow once',
      value: ToolConfirmationOutcome.ProceedOnce,
    },
    {
      label: 'Yes, allow always for this session',
      value: ToolConfirmationOutcome.ProceedAlways,
    },
    {
      label: 'No (esc)',
      value: ToolConfirmationOutcome.Cancel,
    },
  ];

  return (
    <Box
      flexDirection="column"
      borderStyle="round"
      borderColor={Colors.AccentYellow}
      padding={1}
      width="100%"
      marginLeft={1}
    >
      {/* 标题和说明 */}
      <Box flexDirection="column" marginBottom={1}>
        <Text bold>Shell Command Execution</Text>
        <Text>A custom command wants to run the following shell commands:</Text>
        
        {/* 命令展示区域 */}
        <Box
          flexDirection="column"
          borderStyle="round"
          borderColor={Colors.Gray}
          paddingX={1}
          marginTop={1}
        >
          {commands.map((cmd) => (
            <Text key={cmd} color={Colors.AccentCyan}>
              {cmd}
            </Text>
          ))}
        </Box>
      </Box>

      {/* 确认问题 */}
      <Box marginBottom={1}>
        <Text>Do you want to proceed?</Text>
      </Box>

      {/* 选项选择器 */}
      <RadioButtonSelect items={options} onSelect={handleSelect} isFocused />
    </Box>
  );
};
```

## 组件层次结构

### 布局结构

```
ShellConfirmationDialog
├── 外层容器 (Box with yellow border)
│   ├── 标题区域 (Box with title and description)
│   │   ├── 标题文本 ("Shell Command Execution")
│   │   ├── 说明文本 (描述要执行的命令)
│   │   └── 命令展示区域 (Box with gray border)
│   │       └── 命令列表 (每行一个命令)
│   ├── 确认问题 (Box with question text)
│   └── 选项选择器 (RadioButtonSelect)
│       ├── "Yes, allow once" (ProceedOnce)
│       ├── "Yes, allow always for this session" (ProceedAlways) 
│       └── "No (esc)" (Cancel)
```

### 样式配置

```typescript
// 主容器样式
const mainContainerStyle = {
  flexDirection: "column" as const,
  borderStyle: "round" as const,
  borderColor: Colors.AccentYellow,    // 黄色边框警示
  padding: 1,
  width: "100%",
  marginLeft: 1,
};

// 命令展示区域样式
const commandDisplayStyle = {
  flexDirection: "column" as const,
  borderStyle: "round" as const,
  borderColor: Colors.Gray,            // 灰色边框区分
  paddingX: 1,
  marginTop: 1,
};

// 命令文本样式
const commandTextColor = Colors.AccentCyan;  // 青色高亮命令
```

## 使用示例

### 基本使用

```typescript
import { ShellConfirmationDialog } from './ui/components/ShellConfirmationDialog.js';
import { ToolConfirmationOutcome } from '@google/gemini-cli-core';

// 基本的 Shell 命令确认
const BasicShellConfirmation: React.FC = () => {
  const [request, setRequest] = useState<ShellConfirmationRequest | null>(null);

  const handleExecuteCommands = (commands: string[]) => {
    const confirmationRequest: ShellConfirmationRequest = {
      commands,
      onConfirm: (outcome, approvedCommands) => {
        switch (outcome) {
          case ToolConfirmationOutcome.ProceedOnce:
            console.log('User approved commands for once:', approvedCommands);
            executeCommands(approvedCommands || []);
            break;
            
          case ToolConfirmationOutcome.ProceedAlways:
            console.log('User approved commands for session:', approvedCommands);
            // 保存会话级批准状态
            setSessionApproval(commands, true);
            executeCommands(approvedCommands || []);
            break;
            
          case ToolConfirmationOutcome.Cancel:
            console.log('User cancelled command execution');
            break;
        }
        setRequest(null);
      }
    };
    
    setRequest(confirmationRequest);
  };

  const executeCommands = async (commands: string[]) => {
    for (const command of commands) {
      try {
        console.log(`Executing: ${command}`);
        // 实际的命令执行逻辑
        await shellService.execute(command);
      } catch (error) {
        console.error(`Failed to execute ${command}:`, error);
      }
    }
  };

  return (
    <Box flexDirection="column">
      {request && <ShellConfirmationDialog request={request} />}
      
      {/* 触发命令执行的按钮示例 */}
      <Button onClick={() => handleExecuteCommands(['npm install', 'npm run build'])}>
        Execute Build Commands
      </Button>
    </Box>
  );
};
```

### 与工具系统集成

```typescript
// 工具系统中的 Shell 命令确认
class ShellTool extends BaseTool {
  private sessionApprovals = new Set<string>();

  async execute(params: ShellToolParams): Promise<ToolResult> {
    const commands = this.parseCommands(params.command);
    
    // 检查会话级批准
    const needsConfirmation = commands.some(cmd => 
      !this.sessionApprovals.has(this.normalizeCommand(cmd))
    );

    if (needsConfirmation) {
      return this.requestConfirmation(commands);
    }

    return this.executeCommands(commands);
  }

  private requestConfirmation(commands: string[]): Promise<ToolResult> {
    return new Promise((resolve) => {
      const request: ShellConfirmationRequest = {
        commands,
        onConfirm: (outcome, approvedCommands) => {
          switch (outcome) {
            case ToolConfirmationOutcome.ProceedOnce:
              resolve(this.executeCommands(approvedCommands || []));
              break;
              
            case ToolConfirmationOutcome.ProceedAlways:
              // 保存会话级批准
              approvedCommands?.forEach(cmd => {
                this.sessionApprovals.add(this.normalizeCommand(cmd));
              });
              resolve(this.executeCommands(approvedCommands || []));
              break;
              
            case ToolConfirmationOutcome.Cancel:
              resolve({
                summary: 'Command execution cancelled by user',
                llmContent: 'The user chose not to execute the requested shell commands.',
                returnDisplay: 'Command execution cancelled.'
              });
              break;
          }
        }
      };

      // 通过应用状态或事件系统发送确认请求
      this.showConfirmationDialog(request);
    });
  }

  private async executeCommands(commands: string[]): Promise<ToolResult> {
    const results: string[] = [];
    
    for (const command of commands) {
      try {
        const result = await this.shellService.execute(command);
        results.push(`✓ ${command}\n${result.output}`);
      } catch (error) {
        results.push(`✗ ${command}\nError: ${error.message}`);
      }
    }

    return {
      summary: `Executed ${commands.length} shell command(s)`,
      llmContent: results.join('\n\n'),
      returnDisplay: results.join('\n\n')
    };
  }

  private normalizeCommand(command: string): string {
    // 标准化命令以进行批准状态检查
    return command.trim().toLowerCase();
  }
}
```

### 安全验证集成

```typescript
// 带安全验证的 Shell 确认对话框
class SecureShellConfirmationManager {
  private dangerousPatterns = [
    /rm\s+-rf/i,
    /sudo\s+rm/i,
    /dd\s+if=/i,
    /mkfs/i,
    /format/i,
    /> \/dev\//i,
    /curl.*\|\s*sh/i,
    /wget.*\|\s*sh/i,
  ];

  validateCommands(commands: string[]): ValidationResult {
    const warnings: string[] = [];
    const blocked: string[] = [];

    for (const command of commands) {
      const risk = this.assessCommandRisk(command);
      
      if (risk.level === 'blocked') {
        blocked.push(command);
      } else if (risk.level === 'high') {
        warnings.push(`High risk: ${command} - ${risk.reason}`);
      }
    }

    return { warnings, blocked, safe: blocked.length === 0 };
  }

  private assessCommandRisk(command: string): RiskAssessment {
    for (const pattern of this.dangerousPatterns) {
      if (pattern.test(command)) {
        return {
          level: 'blocked',
          reason: 'Contains potentially dangerous operation'
        };
      }
    }

    // 检查网络操作
    if (/curl|wget|nc|telnet/.test(command)) {
      return {
        level: 'high',
        reason: 'Performs network operation'
      };
    }

    // 检查系统修改
    if (/chmod|chown|mount|umount/.test(command)) {
      return {
        level: 'medium',
        reason: 'Modifies system permissions or mounts'
      };
    }

    return { level: 'low', reason: 'Standard operation' };
  }

  createSecureRequest(
    commands: string[],
    originalOnConfirm: ShellConfirmationRequest['onConfirm']
  ): ShellConfirmationRequest {
    const validation = this.validateCommands(commands);
    
    if (!validation.safe) {
      throw new Error(`Blocked dangerous commands: ${validation.blocked.join(', ')}`);
    }

    return {
      commands,
      onConfirm: (outcome, approvedCommands) => {
        if (outcome !== ToolConfirmationOutcome.Cancel) {
          // 最终安全检查
          const finalValidation = this.validateCommands(approvedCommands || []);
          if (!finalValidation.safe) {
            console.error('Security validation failed during execution');
            return;
          }
        }
        
        originalOnConfirm(outcome, approvedCommands);
      }
    };
  }
}

interface ValidationResult {
  warnings: string[];
  blocked: string[];
  safe: boolean;
}

interface RiskAssessment {
  level: 'low' | 'medium' | 'high' | 'blocked';
  reason: string;
}

// 使用示例
const securityManager = new SecureShellConfirmationManager();

const handleSecureShellExecution = (commands: string[]) => {
  try {
    const secureRequest = securityManager.createSecureRequest(
      commands,
      (outcome, approvedCommands) => {
        // 处理确认结果
        console.log('Secure execution approved:', outcome, approvedCommands);
      }
    );

    // 显示对话框
    setShellConfirmationRequest(secureRequest);
  } catch (error) {
    console.error('Security validation failed:', error.message);
    // 显示安全错误消息
  }
};
```

## 高级功能

### 命令分组和分类

```typescript
// 增强的 Shell 确认对话框，支持命令分组
interface EnhancedShellConfirmationRequest extends ShellConfirmationRequest {
  commandGroups?: CommandGroup[];
  riskLevel?: 'low' | 'medium' | 'high';
  estimatedDuration?: string;
  workingDirectory?: string;
}

interface CommandGroup {
  name: string;
  commands: string[];
  description?: string;
  riskLevel: 'low' | 'medium' | 'high';
}

const EnhancedShellConfirmationDialog: React.FC<{
  request: EnhancedShellConfirmationRequest;
}> = ({ request }) => {
  const { commands, commandGroups, riskLevel, estimatedDuration, workingDirectory } = request;

  const getRiskColor = (risk: string) => {
    switch (risk) {
      case 'low': return Colors.AccentGreen;
      case 'medium': return Colors.AccentYellow;
      case 'high': return Colors.AccentRed;
      default: return Colors.Foreground;
    }
  };

  return (
    <Box
      flexDirection="column"
      borderStyle="round"
      borderColor={getRiskColor(riskLevel || 'low')}
      padding={1}
      width="100%"
      marginLeft={1}
    >
      {/* 标题和元信息 */}
      <Box flexDirection="column" marginBottom={1}>
        <Text bold>Shell Command Execution Request</Text>
        
        {workingDirectory && (
          <Text color={Colors.Gray}>Working Directory: {workingDirectory}</Text>
        )}
        
        {estimatedDuration && (
          <Text color={Colors.Gray}>Estimated Duration: {estimatedDuration}</Text>
        )}
        
        {riskLevel && (
          <Text color={getRiskColor(riskLevel)}>
            Risk Level: {riskLevel.toUpperCase()}
          </Text>
        )}
      </Box>

      {/* 命令组展示 */}
      {commandGroups ? (
        <Box flexDirection="column" marginBottom={1}>
          {commandGroups.map((group, index) => (
            <Box key={index} flexDirection="column" marginBottom={1}>
              <Text bold color={getRiskColor(group.riskLevel)}>
                {group.name}
              </Text>
              {group.description && (
                <Text color={Colors.Gray}>{group.description}</Text>
              )}
              <Box
                flexDirection="column"
                borderStyle="round"
                borderColor={Colors.Gray}
                paddingX={1}
                marginTop={1}
              >
                {group.commands.map((cmd) => (
                  <Text key={cmd} color={Colors.AccentCyan}>
                    {cmd}
                  </Text>
                ))}
              </Box>
            </Box>
          ))}
        </Box>
      ) : (
        /* 标准命令展示 */
        <Box flexDirection="column" marginBottom={1}>
          <Text>The following commands will be executed:</Text>
          <Box
            flexDirection="column"
            borderStyle="round"
            borderColor={Colors.Gray}
            paddingX={1}
            marginTop={1}
          >
            {commands.map((cmd) => (
              <Text key={cmd} color={Colors.AccentCyan}>
                {cmd}
              </Text>
            ))}
          </Box>
        </Box>
      )}

      {/* 确认问题 */}
      <Box marginBottom={1}>
        <Text>Do you want to proceed with this execution?</Text>
      </Box>

      {/* 动态选项 */}
      <RadioButtonSelect 
        items={generateOptions(riskLevel)} 
        onSelect={request.onConfirm} 
        isFocused 
      />
    </Box>
  );
};

const generateOptions = (riskLevel?: string): Array<RadioSelectItem<ToolConfirmationOutcome>> => {
  const baseOptions = [
    {
      label: 'Yes, allow once',
      value: ToolConfirmationOutcome.ProceedOnce,
    },
    {
      label: 'No (esc)',
      value: ToolConfirmationOutcome.Cancel,
    },
  ];

  // 高风险命令不提供会话级批准选项
  if (riskLevel !== 'high') {
    baseOptions.splice(1, 0, {
      label: 'Yes, allow always for this session',
      value: ToolConfirmationOutcome.ProceedAlways,
    });
  }

  return baseOptions;
};
```

### 历史记录和审计

```typescript
// Shell 命令执行历史和审计功能
class ShellExecutionAuditor {
  private executionHistory: ShellExecutionRecord[] = [];
  private maxHistorySize = 1000;

  recordExecution(record: ShellExecutionRecord): void {
    this.executionHistory.unshift(record);
    
    // 限制历史记录大小
    if (this.executionHistory.length > this.maxHistorySize) {
      this.executionHistory = this.executionHistory.slice(0, this.maxHistorySize);
    }

    // 持久化到文件（可选）
    this.saveHistoryToFile();
  }

  getExecutionHistory(limit?: number): ShellExecutionRecord[] {
    return limit ? this.executionHistory.slice(0, limit) : [...this.executionHistory];
  }

  getCommandStats(): CommandStats {
    const stats: CommandStats = {
      totalExecutions: this.executionHistory.length,
      approvedOnce: 0,
      approvedAlways: 0,
      cancelled: 0,
      mostCommonCommands: new Map(),
      riskLevelDistribution: { low: 0, medium: 0, high: 0 },
    };

    for (const record of this.executionHistory) {
      // 统计批准类型
      switch (record.outcome) {
        case ToolConfirmationOutcome.ProceedOnce:
          stats.approvedOnce++;
          break;
        case ToolConfirmationOutcome.ProceedAlways:
          stats.approvedAlways++;
          break;
        case ToolConfirmationOutcome.Cancel:
          stats.cancelled++;
          break;
      }

      // 统计命令频率
      for (const command of record.commands) {
        const baseCommand = command.split(' ')[0];
        stats.mostCommonCommands.set(
          baseCommand,
          (stats.mostCommonCommands.get(baseCommand) || 0) + 1
        );
      }

      // 统计风险级别
      if (record.riskLevel) {
        stats.riskLevelDistribution[record.riskLevel]++;
      }
    }

    return stats;
  }

  private saveHistoryToFile(): void {
    try {
      const historyFile = path.join(os.homedir(), '.gemini-cli', 'shell-history.json');
      fs.writeFileSync(historyFile, JSON.stringify(this.executionHistory, null, 2));
    } catch (error) {
      console.warn('Failed to save shell execution history:', error);
    }
  }

  loadHistoryFromFile(): void {
    try {
      const historyFile = path.join(os.homedir(), '.gemini-cli', 'shell-history.json');
      if (fs.existsSync(historyFile)) {
        const data = fs.readFileSync(historyFile, 'utf-8');
        this.executionHistory = JSON.parse(data);
      }
    } catch (error) {
      console.warn('Failed to load shell execution history:', error);
    }
  }
}

interface ShellExecutionRecord {
  id: string;
  timestamp: Date;
  commands: string[];
  outcome: ToolConfirmationOutcome;
  approvedCommands?: string[];
  workingDirectory: string;
  riskLevel?: 'low' | 'medium' | 'high';
  duration?: number;
  success?: boolean;
  error?: string;
}

interface CommandStats {
  totalExecutions: number;
  approvedOnce: number;
  approvedAlways: number;
  cancelled: number;
  mostCommonCommands: Map<string, number>;
  riskLevelDistribution: { low: number; medium: number; high: number };
}

// 集成审计功能的确认对话框
const AuditedShellConfirmationDialog: React.FC<{
  request: ShellConfirmationRequest;
  auditor: ShellExecutionAuditor;
}> = ({ request, auditor }) => {
  const enhancedOnConfirm = useCallback((
    outcome: ToolConfirmationOutcome,
    approvedCommands?: string[]
  ) => {
    // 记录执行历史
    const record: ShellExecutionRecord = {
      id: `shell-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`,
      timestamp: new Date(),
      commands: request.commands,
      outcome,
      approvedCommands,
      workingDirectory: process.cwd(),
    };

    auditor.recordExecution(record);

    // 调用原始回调
    request.onConfirm(outcome, approvedCommands);
  }, [request, auditor]);

  const enhancedRequest = {
    ...request,
    onConfirm: enhancedOnConfirm,
  };

  return <ShellConfirmationDialog request={enhancedRequest} />;
};
```

## 错误处理

### 常见错误场景

```typescript
// Shell 确认对话框错误处理
enum ShellConfirmationError {
  EMPTY_COMMANDS = 'No commands provided for confirmation',
  INVALID_CALLBACK = 'Invalid confirmation callback provided',
  SECURITY_VIOLATION = 'Command blocked by security policy',
  TIMEOUT = 'User confirmation timeout',
  SYSTEM_ERROR = 'System error during command confirmation',
}

class ShellConfirmationErrorHandler {
  static handleError(error: Error, commands: string[]): void {
    console.error('Shell confirmation error:', error);

    // 根据错误类型进行不同处理
    if (error.message.includes('security')) {
      // 安全错误
      console.warn('Security policy blocked commands:', commands);
      // 可以显示安全警告对话框
    } else if (error.message.includes('timeout')) {
      // 超时错误
      console.warn('User confirmation timeout for commands:', commands);
      // 自动取消执行
    } else {
      // 其他系统错误
      console.error('Unexpected error during shell confirmation:', error);
    }
  }

  static validateRequest(request: ShellConfirmationRequest): void {
    if (!request.commands || request.commands.length === 0) {
      throw new Error(ShellConfirmationError.EMPTY_COMMANDS);
    }

    if (typeof request.onConfirm !== 'function') {
      throw new Error(ShellConfirmationError.INVALID_CALLBACK);
    }

    // 验证命令格式
    for (const command of request.commands) {
      if (typeof command !== 'string' || command.trim().length === 0) {
        throw new Error(`Invalid command format: ${command}`);
      }
    }
  }
}

// 错误恢复示例
const RobustShellConfirmationDialog: React.FC<{
  request: ShellConfirmationRequest;
}> = ({ request }) => {
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    try {
      ShellConfirmationErrorHandler.validateRequest(request);
      setError(null);
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Unknown error');
      ShellConfirmationErrorHandler.handleError(
        err instanceof Error ? err : new Error(String(err)),
        request.commands
      );
    }
  }, [request]);

  if (error) {
    return (
      <Box
        flexDirection="column"
        borderStyle="round"
        borderColor={Colors.AccentRed}
        padding={1}
        width="100%"
        marginLeft={1}
      >
        <Text bold color={Colors.AccentRed}>
          Shell Confirmation Error
        </Text>
        <Text color={Colors.AccentRed}>{error}</Text>
        <Box marginTop={1}>
          <Text>
            Please check the command request and try again.
          </Text>
        </Box>
      </Box>
    );
  }

  return <ShellConfirmationDialog request={request} />;
};
```

## 测试

### 单元测试示例

```typescript
// ShellConfirmationDialog 组件测试
import { render, screen } from '@testing-library/react';
import { ShellConfirmationDialog } from './ShellConfirmationDialog';
import { ToolConfirmationOutcome } from '@google/gemini-cli-core';

describe('ShellConfirmationDialog', () => {
  const mockOnConfirm = jest.fn();
  
  const defaultRequest = {
    commands: ['npm install', 'npm run build'],
    onConfirm: mockOnConfirm,
  };

  beforeEach(() => {
    jest.clearAllMocks();
  });

  it('renders command list correctly', () => {
    render(<ShellConfirmationDialog request={defaultRequest} />);
    
    expect(screen.getByText('Shell Command Execution')).toBeInTheDocument();
    expect(screen.getByText('npm install')).toBeInTheDocument();
    expect(screen.getByText('npm run build')).toBeInTheDocument();
  });

  it('shows confirmation options', () => {
    render(<ShellConfirmationDialog request={defaultRequest} />);
    
    expect(screen.getByText('Yes, allow once')).toBeInTheDocument();
    expect(screen.getByText('Yes, allow always for this session')).toBeInTheDocument();
    expect(screen.getByText('No (esc)')).toBeInTheDocument();
  });

  it('calls onConfirm with correct parameters for ProceedOnce', () => {
    render(<ShellConfirmationDialog request={defaultRequest} />);
    
    // 模拟选择 "Yes, allow once"
    // 这需要根据 RadioButtonSelect 的实际实现来调整
    // fireEvent.click(screen.getByText('Yes, allow once'));
    
    // expect(mockOnConfirm).toHaveBeenCalledWith(
    //   ToolConfirmationOutcome.ProceedOnce,
    //   ['npm install', 'npm run build']
    // );
  });

  it('handles empty command list', () => {
    const emptyRequest = { ...defaultRequest, commands: [] };
    render(<ShellConfirmationDialog request={emptyRequest} />);
    
    expect(screen.getByText('Shell Command Execution')).toBeInTheDocument();
    // 验证空命令列表的处理
  });

  it('handles long command strings', () => {
    const longCommandRequest = {
      ...defaultRequest,
      commands: ['very-long-command-that-might-overflow-the-display-area-and-cause-wrapping-issues'],
    };
    
    render(<ShellConfirmationDialog request={longCommandRequest} />);
    
    // 验证长命令的显示处理
    expect(screen.getByText(/very-long-command/)).toBeInTheDocument();
  });
});

// 集成测试示例
describe('ShellConfirmationDialog Integration', () => {
  it('integrates with shell execution service', async () => {
    const mockShellService = {
      execute: jest.fn().mockResolvedValue({ output: 'success', exitCode: 0 }),
    };

    const integrationRequest = {
      commands: ['echo "test"'],
      onConfirm: async (outcome: ToolConfirmationOutcome, commands?: string[]) => {
        if (outcome === ToolConfirmationOutcome.ProceedOnce && commands) {
          for (const command of commands) {
            await mockShellService.execute(command);
          }
        }
      },
    };

    render(<ShellConfirmationDialog request={integrationRequest} />);
    
    // 模拟用户确认
    // ... 测试用户交互和命令执行集成
  });
});
```

## 最佳实践

### 安全性

```typescript
// ✅ 推荐：命令安全验证
const validateShellCommands = (commands: string[]): boolean => {
  const dangerousPatterns = [
    /rm\s+-rf\s+\//,
    /sudo\s+rm/,
    /mkfs/,
    /format/,
  ];

  return commands.every(command => 
    !dangerousPatterns.some(pattern => pattern.test(command))
  );
};

// ✅ 推荐：用户体验优化
const createUserFriendlyRequest = (
  commands: string[],
  context: ExecutionContext
): ShellConfirmationRequest => {
  return {
    commands: commands.map(cmd => 
      cmd.length > 80 ? `${cmd.substring(0, 77)}...` : cmd
    ),
    onConfirm: (outcome, approvedCommands) => {
      // 提供清晰的反馈
      const action = outcome === ToolConfirmationOutcome.Cancel 
        ? 'cancelled' 
        : 'approved';
      console.log(`Shell execution ${action} by user`);
      
      context.handleConfirmation(outcome, approvedCommands);
    }
  };
};

// ✅ 推荐：响应式布局
const adaptToTerminalSize = (terminalWidth: number) => {
  const maxCommandWidth = Math.max(40, terminalWidth * 0.8);
  const borderStyle = terminalWidth < 80 ? 'single' : 'round';
  
  return { maxCommandWidth, borderStyle };
};
```

### 性能优化

```typescript
// ✅ 推荐：组件缓存优化
const MemoizedShellConfirmationDialog = React.memo(ShellConfirmationDialog, 
  (prevProps, nextProps) => {
    // 只有命令列表或回调改变时才重新渲染
    return (
      JSON.stringify(prevProps.request.commands) === 
      JSON.stringify(nextProps.request.commands) &&
      prevProps.request.onConfirm === nextProps.request.onConfirm
    );
  }
);

// ✅ 推荐：异步确认处理
const handleAsyncConfirmation = async (
  outcome: ToolConfirmationOutcome,
  commands?: string[]
) => {
  try {
    // 显示处理中状态
    setProcessing(true);
    
    if (outcome !== ToolConfirmationOutcome.Cancel) {
      await executeCommandsWithProgress(commands || []);
    }
  } catch (error) {
    console.error('Command execution failed:', error);
  } finally {
    setProcessing(false);
  }
};
```

### 可访问性

```typescript
// ✅ 推荐：键盘导航支持
const AccessibleShellConfirmationDialog: React.FC<ShellConfirmationDialogProps> = ({ 
  request 
}) => {
  useInput((input, key) => {
    // 支持数字键快速选择
    if (input >= '1' && input <= '3') {
      const index = parseInt(input) - 1;
      const options = [
        ToolConfirmationOutcome.ProceedOnce,
        ToolConfirmationOutcome.ProceedAlways,
        ToolConfirmationOutcome.Cancel,
      ];
      
      if (options[index]) {
        request.onConfirm(options[index], 
          options[index] !== ToolConfirmationOutcome.Cancel ? request.commands : undefined
        );
      }
    }
    
    // ESC 键取消
    if (key.escape) {
      request.onConfirm(ToolConfirmationOutcome.Cancel);
    }
  });

  // ... 组件渲染
};

// ✅ 推荐：屏幕阅读器支持
const ScreenReaderFriendlyDialog: React.FC<ShellConfirmationDialogProps> = ({ 
  request 
}) => {
  return (
    <Box
      flexDirection="column"
      borderStyle="round"
      borderColor={Colors.AccentYellow}
      padding={1}
      width="100%"
      marginLeft={1}
      // 添加语义化属性
      role="dialog"
      aria-labelledby="shell-confirmation-title"
      aria-describedby="shell-confirmation-description"
    >
      <Text id="shell-confirmation-title" bold>
        Shell Command Execution Confirmation
      </Text>
      <Text id="shell-confirmation-description">
        Review the commands below and choose how to proceed.
        Press 1 for once, 2 for always, 3 for cancel, or Escape to cancel.
      </Text>
      
      {/* 命令列表 */}
      <Box flexDirection="column" role="list">
        {request.commands.map((cmd, index) => (
          <Text key={cmd} role="listitem" color={Colors.AccentCyan}>
            Command {index + 1}: {cmd}
          </Text>
        ))}
      </Box>
      
      {/* ... 其他内容 */}
    </Box>
  );
};
```