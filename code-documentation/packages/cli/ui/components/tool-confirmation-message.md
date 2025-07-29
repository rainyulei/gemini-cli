# ToolConfirmationMessage 工具确认消息文档

## 概述

`ToolConfirmationMessage` 是 Gemini CLI 的工具执行确认组件，当 AI 模型请求使用工具时显示给用户进行确认。它支持多种工具类型的确认，包括文件编辑、命令执行、信息获取和 MCP 工具调用，提供灵活的确认选项和丰富的上下文信息展示。

## 主要功能

- 多种工具类型确认支持（edit、exec、info、mcp）
- 文件差异可视化显示（DiffRenderer 集成）
- 外部编辑器修改支持
- MCP 服务器工具权限管理
- 响应式布局和终端高度自适应
- 键盘快捷键和导航支持
- 动态选项生成和上下文感知

## 接口定义

### `ToolConfirmationMessageProps`

**功能**: 工具确认消息组件属性接口

```typescript
export interface ToolConfirmationMessageProps {
  /** 工具调用确认详情 */
  confirmationDetails: ToolCallConfirmationDetails;
  /** 配置对象（可选） */
  config?: Config;
  /** 是否聚焦（默认 true） */
  isFocused?: boolean;
  /** 可用的终端高度 */
  availableTerminalHeight?: number;
  /** 终端宽度 */
  terminalWidth: number;
}
```

### 工具确认类型

#### `ToolCallConfirmationDetails` (联合类型)

```typescript
type ToolCallConfirmationDetails =
  | ToolEditConfirmationDetails
  | ToolExecuteConfirmationDetails
  | ToolInfoConfirmationDetails
  | ToolMcpConfirmationDetails;
```

#### 编辑工具确认详情

```typescript
interface ToolEditConfirmationDetails {
  type: 'edit';
  fileName: string;
  fileDiff: string;
  isModifying: boolean;
  onConfirm: (outcome: ToolConfirmationOutcome) => void;
}
```

#### 执行工具确认详情

```typescript
interface ToolExecuteConfirmationDetails {
  type: 'exec';
  rootCommand: string;
  command: string;
  onConfirm: (outcome: ToolConfirmationOutcome) => void;
}
```

#### 信息工具确认详情

```typescript
interface ToolInfoConfirmationDetails {
  type: 'info';
  prompt: string;
  urls?: string[];
  onConfirm: (outcome: ToolConfirmationOutcome) => void;
}
```

#### MCP 工具确认详情

```typescript
interface ToolMcpConfirmationDetails {
  type: 'mcp';
  serverName: string;
  toolName: string;
  onConfirm: (outcome: ToolConfirmationOutcome) => void;
}
```

### 确认结果类型

```typescript
enum ToolConfirmationOutcome {
  ProceedOnce = 'proceed_once',              // 允许一次
  ProceedAlways = 'proceed_always',          // 始终允许
  ProceedAlwaysTool = 'proceed_always_tool', // 始终允许此工具
  ProceedAlwaysServer = 'proceed_always_server', // 始终允许此服务器
  ModifyWithEditor = 'modify_with_editor',   // 使用外部编辑器修改
  Cancel = 'cancel',                         // 取消
}
```

## 核心组件实现

### 主组件结构

```typescript
export const ToolConfirmationMessage: React.FC<ToolConfirmationMessageProps> = ({
  confirmationDetails,
  isFocused = true,
  availableTerminalHeight,
  terminalWidth,
}) => {
  const { onConfirm } = confirmationDetails;
  const childWidth = terminalWidth - 2; // 为填充预留空间

  // 键盘输入处理
  useInput((_, key) => {
    if (!isFocused) return;
    if (key.escape) {
      onConfirm(ToolConfirmationOutcome.Cancel);
    }
  });

  const handleSelect = (item: ToolConfirmationOutcome) => onConfirm(item);

  // 动态内容和选项生成
  let bodyContent: React.ReactNode | null = null;
  let question: string;
  const options: Array<RadioSelectItem<ToolConfirmationOutcome>> = [];

  // 根据确认类型生成内容和选项
  if (confirmationDetails.type === 'edit') {
    // 文件编辑确认逻辑
  } else if (confirmationDetails.type === 'exec') {
    // 命令执行确认逻辑
  } else if (confirmationDetails.type === 'info') {
    // 信息获取确认逻辑
  } else {
    // MCP 工具确认逻辑
  }

  return (
    <Box flexDirection="column" padding={1} width={childWidth}>
      {/* 主体内容区域 */}
      <Box flexGrow={1} flexShrink={1} overflow="hidden" marginBottom={1}>
        {bodyContent}
      </Box>

      {/* 确认问题 */}
      <Box marginBottom={1} flexShrink={0}>
        <Text wrap="truncate">{question}</Text>
      </Box>

      {/* 选项选择器 */}
      <Box flexShrink={0}>
        <RadioButtonSelect
          items={options}
          onSelect={handleSelect}
          isFocused={isFocused}
        />
      </Box>
    </Box>
  );
};
```

### 文件编辑确认处理

```typescript
// 文件编辑确认的详细实现
if (confirmationDetails.type === 'edit') {
  if (confirmationDetails.isModifying) {
    // 正在修改状态
    return (
      <Box
        minWidth="90%"
        borderStyle="round"
        borderColor={Colors.Gray}
        justifyContent="space-around"
        padding={1}
        overflow="hidden"
      >
        <Text>Modify in progress: </Text>
        <Text color={Colors.AccentGreen}>
          Save and close external editor to continue
        </Text>
      </Box>
    );
  }

  question = `Apply this change?`;
  options.push(
    {
      label: 'Yes, allow once',
      value: ToolConfirmationOutcome.ProceedOnce,
    },
    {
      label: 'Yes, allow always',
      value: ToolConfirmationOutcome.ProceedAlways,
    },
    {
      label: 'Modify with external editor',
      value: ToolConfirmationOutcome.ModifyWithEditor,
    },
    { label: 'No (esc)', value: ToolConfirmationOutcome.Cancel },
  );

  bodyContent = (
    <DiffRenderer
      diffContent={confirmationDetails.fileDiff}
      filename={confirmationDetails.fileName}
      availableTerminalHeight={availableBodyContentHeight()}
      terminalWidth={childWidth}
    />
  );
}
```

### 命令执行确认处理

```typescript
// 命令执行确认的详细实现
else if (confirmationDetails.type === 'exec') {
  const executionProps = confirmationDetails as ToolExecuteConfirmationDetails;

  question = `Allow execution of: '${executionProps.rootCommand}'?`;
  options.push(
    {
      label: `Yes, allow once`,
      value: ToolConfirmationOutcome.ProceedOnce,
    },
    {
      label: `Yes, allow always ...`,
      value: ToolConfirmationOutcome.ProceedAlways,
    },
    { label: 'No (esc)', value: ToolConfirmationOutcome.Cancel }
  );

  let bodyContentHeight = availableBodyContentHeight();
  if (bodyContentHeight !== undefined) {
    bodyContentHeight -= 2; // 考虑填充
  }

  bodyContent = (
    <Box flexDirection="column">
      <Box paddingX={1} marginLeft={1}>
        <MaxSizedBox
          maxHeight={bodyContentHeight}
          maxWidth={Math.max(childWidth - 4, 1)}
        >
          <Box>
            <Text color={Colors.AccentCyan}>{executionProps.command}</Text>
          </Box>
        </MaxSizedBox>
      </Box>
    </Box>
  );
}
```

### MCP 工具确认处理

```typescript
// MCP 工具确认的详细实现
else {
  const mcpProps = confirmationDetails as ToolMcpConfirmationDetails;

  bodyContent = (
    <Box flexDirection="column" paddingX={1} marginLeft={1}>
      <Text color={Colors.AccentCyan}>MCP Server: {mcpProps.serverName}</Text>
      <Text color={Colors.AccentCyan}>Tool: {mcpProps.toolName}</Text>
    </Box>
  );

  question = `Allow execution of MCP tool "${mcpProps.toolName}" from server "${mcpProps.serverName}"?`;
  options.push(
    {
      label: 'Yes, allow once',
      value: ToolConfirmationOutcome.ProceedOnce,
    },
    {
      label: `Yes, always allow tool "${mcpProps.toolName}" from server "${mcpProps.serverName}"`,
      value: ToolConfirmationOutcome.ProceedAlwaysTool,
    },
    {
      label: `Yes, always allow all tools from server "${mcpProps.serverName}"`,
      value: ToolConfirmationOutcome.ProceedAlwaysServer,
    },
    { label: 'No (esc)', value: ToolConfirmationOutcome.Cancel },
  );
}
```

### 可用高度计算

```typescript
function availableBodyContentHeight(): number | undefined {
  if (options.length === 0) {
    throw new Error('Options not provided for confirmation message');
  }

  if (availableTerminalHeight === undefined) {
    return undefined;
  }

  // 计算周围 UI 元素占用的垂直空间（以行为单位）
  const PADDING_OUTER_Y = 2;      // 主容器填充（上下）
  const MARGIN_BODY_BOTTOM = 1;   // 主体容器边距
  const HEIGHT_QUESTION = 1;      // 问题文本一行
  const MARGIN_QUESTION_BOTTOM = 1; // 问题容器边距
  const HEIGHT_OPTIONS = options.length; // 每个选项占一行

  const surroundingElementsHeight =
    PADDING_OUTER_Y +
    MARGIN_BODY_BOTTOM +
    HEIGHT_QUESTION +
    MARGIN_QUESTION_BOTTOM +
    HEIGHT_OPTIONS;

  return Math.max(availableTerminalHeight - surroundingElementsHeight, 1);
}
```

## 使用示例

### 文件编辑确认

```typescript
import { ToolConfirmationMessage } from './ui/components/messages/ToolConfirmationMessage.js';
import { ToolConfirmationOutcome } from '@google/gemini-cli-core';

// 文件编辑确认示例
const FileEditConfirmation: React.FC = () => {
  const [confirmationDetails, setConfirmationDetails] = useState<ToolCallConfirmationDetails | null>(null);

  const handleFileEdit = (fileName: string, fileDiff: string) => {
    const editConfirmation: ToolEditConfirmationDetails = {
      type: 'edit',
      fileName,
      fileDiff,
      isModifying: false,
      onConfirm: (outcome) => {
        switch (outcome) {
          case ToolConfirmationOutcome.ProceedOnce:
            console.log('User approved file edit once');
            applyFileChanges(fileName, fileDiff);
            break;

          case ToolConfirmationOutcome.ProceedAlways:
            console.log('User approved file edits always');
            setAlwaysAllowFileEdits(true);
            applyFileChanges(fileName, fileDiff);
            break;

          case ToolConfirmationOutcome.ModifyWithEditor:
            console.log('User wants to modify with external editor');
            openExternalEditor(fileName, fileDiff);
            break;

          case ToolConfirmationOutcome.Cancel:
            console.log('User cancelled file edit');
            break;
        }
        setConfirmationDetails(null);
      }
    };

    setConfirmationDetails(editConfirmation);
  };

  const applyFileChanges = async (fileName: string, diff: string) => {
    try {
      await fileService.applyDiff(fileName, diff);
      console.log(`Successfully applied changes to ${fileName}`);
    } catch (error) {
      console.error(`Failed to apply changes to ${fileName}:`, error);
    }
  };

  const openExternalEditor = async (fileName: string, diff: string) => {
    try {
      // 设置修改状态
      setConfirmationDetails({
        ...confirmationDetails!,
        isModifying: true,
      } as ToolEditConfirmationDetails);

      const modifiedContent = await editorService.openForModification(fileName, diff);
      
      // 应用修改后的内容
      await fileService.writeFile(fileName, modifiedContent);
      console.log(`Successfully modified ${fileName} with external editor`);
      
      setConfirmationDetails(null);
    } catch (error) {
      console.error('External editor modification failed:', error);
      setConfirmationDetails(null);
    }
  };

  return (
    <Box flexDirection="column">
      {confirmationDetails && (
        <ToolConfirmationMessage
          confirmationDetails={confirmationDetails}
          terminalWidth={80}
          availableTerminalHeight={20}
          isFocused={true}
        />
      )}
    </Box>
  );
};
```

### 命令执行确认

```typescript
// 命令执行确认示例
const CommandExecutionConfirmation: React.FC = () => {
  const handleCommandExecution = (rootCommand: string, fullCommand: string) => {
    const execConfirmation: ToolExecuteConfirmationDetails = {
      type: 'exec',
      rootCommand,
      command: fullCommand,
      onConfirm: async (outcome) => {
        switch (outcome) {
          case ToolConfirmationOutcome.ProceedOnce:
            console.log('User approved command execution once');
            await executeCommand(fullCommand);
            break;

          case ToolConfirmationOutcome.ProceedAlways:
            console.log('User approved similar commands always');
            await addToAlwaysAllowedCommands(rootCommand);
            await executeCommand(fullCommand);
            break;

          case ToolConfirmationOutcome.Cancel:
            console.log('User cancelled command execution');
            break;
        }
      }
    };

    setConfirmationDetails(execConfirmation);
  };

  const executeCommand = async (command: string) => {
    try {
      const result = await shellService.execute(command);
      console.log('Command executed successfully:', result.output);
    } catch (error) {
      console.error('Command execution failed:', error);
    }
  };

  const addToAlwaysAllowedCommands = async (rootCommand: string) => {
    // 将根命令添加到总是允许的列表
    const allowedCommands = await storage.getAllowedCommands();
    allowedCommands.add(rootCommand);
    await storage.setAllowedCommands(allowedCommands);
  };

  return (
    <Box>
      <Button onClick={() => handleCommandExecution('npm', 'npm install --save express')}>
        Request npm install
      </Button>
    </Box>
  );
};
```

### MCP 工具确认

```typescript
// MCP 工具确认示例
const McpToolConfirmation: React.FC = () => {
  const handleMcpToolExecution = (serverName: string, toolName: string, toolArgs: any) => {
    const mcpConfirmation: ToolMcpConfirmationDetails = {
      type: 'mcp',
      serverName,
      toolName,
      onConfirm: async (outcome) => {
        switch (outcome) {
          case ToolConfirmationOutcome.ProceedOnce:
            console.log('User approved MCP tool execution once');
            await executeMcpTool(serverName, toolName, toolArgs);
            break;

          case ToolConfirmationOutcome.ProceedAlwaysTool:
            console.log('User approved this MCP tool always');
            await addToAlwaysAllowedTools(serverName, toolName);
            await executeMcpTool(serverName, toolName, toolArgs);
            break;

          case ToolConfirmationOutcome.ProceedAlwaysServer:
            console.log('User approved all tools from this server');
            await addToAlwaysAllowedServers(serverName);
            await executeMcpTool(serverName, toolName, toolArgs);
            break;

          case ToolConfirmationOutcome.Cancel:
            console.log('User cancelled MCP tool execution');
            break;
        }
      }
    };

    setConfirmationDetails(mcpConfirmation);
  };

  const executeMcpTool = async (serverName: string, toolName: string, args: any) => {
    try {
      const result = await mcpService.executeTool(serverName, toolName, args);
      console.log('MCP tool executed successfully:', result);
    } catch (error) {
      console.error('MCP tool execution failed:', error);
    }
  };

  const addToAlwaysAllowedTools = async (serverName: string, toolName: string) => {
    const allowedTools = await storage.getAllowedMcpTools();
    allowedTools.add(`${serverName}:${toolName}`);
    await storage.setAllowedMcpTools(allowedTools);
  };

  const addToAlwaysAllowedServers = async (serverName: string) => {
    const allowedServers = await storage.getAllowedMcpServers();
    allowedServers.add(serverName);
    await storage.setAllowedMcpServers(allowedServers);
  };

  return (
    <Box>
      <Button onClick={() => handleMcpToolExecution('filesystem', 'read_file', { path: '/tmp/test.txt' })}>
        Request file read via MCP
      </Button>
    </Box>
  );
};
```

## 高级功能

### 权限管理系统

```typescript
// 工具权限管理器
class ToolPermissionManager {
  private alwaysAllowedTools = new Set<string>();
  private alwaysAllowedServers = new Set<string>();
  private alwaysAllowedCommands = new Set<string>();
  private alwaysAllowedFileEdits = false;

  constructor() {
    this.loadPermissions();
  }

  // 检查是否需要确认
  needsConfirmation(details: ToolCallConfirmationDetails): boolean {
    switch (details.type) {
      case 'edit':
        return !this.alwaysAllowedFileEdits;

      case 'exec':
        return !this.alwaysAllowedCommands.has(details.rootCommand);

      case 'mcp':
        return !this.alwaysAllowedServers.has(details.serverName) &&
               !this.alwaysAllowedTools.has(`${details.serverName}:${details.toolName}`);

      case 'info':
        return true; // 信息工具总是需要确认

      default:
        return true;
    }
  }

  // 处理权限更新
  updatePermissions(details: ToolCallConfirmationDetails, outcome: ToolConfirmationOutcome): void {
    switch (outcome) {
      case ToolConfirmationOutcome.ProceedAlways:
        if (details.type === 'edit') {
          this.alwaysAllowedFileEdits = true;
        } else if (details.type === 'exec') {
          this.alwaysAllowedCommands.add(details.rootCommand);
        }
        break;

      case ToolConfirmationOutcome.ProceedAlwaysTool:
        if (details.type === 'mcp') {
          this.alwaysAllowedTools.add(`${details.serverName}:${details.toolName}`);
        }
        break;

      case ToolConfirmationOutcome.ProceedAlwaysServer:
        if (details.type === 'mcp') {
          this.alwaysAllowedServers.add(details.serverName);
        }
        break;
    }

    this.savePermissions();
  }

  // 获取权限状态
  getPermissionStatus(): PermissionStatus {
    return {
      fileEditsAlwaysAllowed: this.alwaysAllowedFileEdits,
      allowedCommands: Array.from(this.alwaysAllowedCommands),
      allowedMcpTools: Array.from(this.alwaysAllowedTools),
      allowedMcpServers: Array.from(this.alwaysAllowedServers),
    };
  }

  // 重置权限
  resetPermissions(): void {
    this.alwaysAllowedTools.clear();
    this.alwaysAllowedServers.clear();
    this.alwaysAllowedCommands.clear();
    this.alwaysAllowedFileEdits = false;
    this.savePermissions();
  }

  private loadPermissions(): void {
    try {
      const permissionsFile = path.join(os.homedir(), '.gemini-cli', 'permissions.json');
      if (fs.existsSync(permissionsFile)) {
        const data = JSON.parse(fs.readFileSync(permissionsFile, 'utf-8'));
        
        this.alwaysAllowedTools = new Set(data.allowedTools || []);
        this.alwaysAllowedServers = new Set(data.allowedServers || []);
        this.alwaysAllowedCommands = new Set(data.allowedCommands || []);
        this.alwaysAllowedFileEdits = data.fileEditsAlwaysAllowed || false;
      }
    } catch (error) {
      console.warn('Failed to load tool permissions:', error);
    }
  }

  private savePermissions(): void {
    try {
      const permissionsFile = path.join(os.homedir(), '.gemini-cli', 'permissions.json');
      const dir = path.dirname(permissionsFile);
      
      if (!fs.existsSync(dir)) {
        fs.mkdirSync(dir, { recursive: true });
      }

      const data = {
        allowedTools: Array.from(this.alwaysAllowedTools),
        allowedServers: Array.from(this.alwaysAllowedServers),
        allowedCommands: Array.from(this.alwaysAllowedCommands),
        fileEditsAlwaysAllowed: this.alwaysAllowedFileEdits,
        lastUpdated: new Date().toISOString(),
      };

      fs.writeFileSync(permissionsFile, JSON.stringify(data, null, 2));
    } catch (error) {
      console.error('Failed to save tool permissions:', error);
    }
  }
}

interface PermissionStatus {
  fileEditsAlwaysAllowed: boolean;
  allowedCommands: string[];
  allowedMcpTools: string[];
  allowedMcpServers: string[];
}

// 权限管理集成的确认组件
const PermissionAwareConfirmationMessage: React.FC<{
  confirmationDetails: ToolCallConfirmationDetails;
  permissionManager: ToolPermissionManager;
  terminalWidth: number;
}> = ({ confirmationDetails, permissionManager, terminalWidth }) => {
  // 检查是否需要确认
  if (!permissionManager.needsConfirmation(confirmationDetails)) {
    // 自动执行已授权的操作
    useEffect(() => {
      confirmationDetails.onConfirm(ToolConfirmationOutcome.ProceedOnce);
    }, []);

    return (
      <Box padding={1}>
        <Text color={Colors.AccentGreen}>
          ✓ Auto-approved based on previous permissions
        </Text>
      </Box>
    );
  }

  // 增强的确认处理
  const enhancedOnConfirm = (outcome: ToolConfirmationOutcome) => {
    // 更新权限
    permissionManager.updatePermissions(confirmationDetails, outcome);
    
    // 调用原始确认处理
    confirmationDetails.onConfirm(outcome);
  };

  const enhancedDetails = {
    ...confirmationDetails,
    onConfirm: enhancedOnConfirm,
  };

  return (
    <ToolConfirmationMessage
      confirmationDetails={enhancedDetails}
      terminalWidth={terminalWidth}
    />
  );
};
```

### 上下文感知确认

```typescript
// 上下文感知的工具确认
class ContextAwareConfirmationProvider {
  private context: ExecutionContext;
  private riskAssessment: RiskAssessmentService;

  constructor(context: ExecutionContext) {
    this.context = context;
    this.riskAssessment = new RiskAssessmentService();
  }

  enhanceConfirmationDetails(
    details: ToolCallConfirmationDetails
  ): EnhancedToolConfirmationDetails {
    const risk = this.riskAssessment.assessRisk(details);
    const contextInfo = this.context.getCurrentContext();

    return {
      ...details,
      riskLevel: risk.level,
      riskReasons: risk.reasons,
      executionContext: contextInfo,
      relatedFiles: this.getRelatedFiles(details),
      potentialImpact: this.assessPotentialImpact(details),
    };
  }

  private getRelatedFiles(details: ToolCallConfirmationDetails): string[] {
    switch (details.type) {
      case 'edit':
        return [details.fileName];
      case 'exec':
        return this.context.getWorkingDirectoryFiles();
      case 'mcp':
        return this.context.getMcpRelatedFiles(details.serverName, details.toolName);
      default:
        return [];
    }
  }

  private assessPotentialImpact(details: ToolCallConfirmationDetails): ImpactAssessment {
    return {
      scope: this.calculateScope(details),
      reversibility: this.calculateReversibility(details),
      securityImplications: this.calculateSecurityRisk(details),
    };
  }

  private calculateScope(details: ToolCallConfirmationDetails): 'file' | 'directory' | 'system' {
    switch (details.type) {
      case 'edit':
        return 'file';
      case 'exec':
        return details.rootCommand.includes('sudo') ? 'system' : 'directory';
      case 'mcp':
        return 'directory'; // MCP 工具通常影响目录级别
      default:
        return 'file';
    }
  }

  private calculateReversibility(details: ToolCallConfirmationDetails): 'easy' | 'difficult' | 'impossible' {
    switch (details.type) {
      case 'edit':
        return 'easy'; // 文件编辑通常可以撤销
      case 'exec':
        return details.command.includes('rm') ? 'impossible' : 'difficult';
      case 'mcp':
        return 'difficult'; // MCP 工具的可逆性取决于具体实现
      default:
        return 'easy';
    }
  }

  private calculateSecurityRisk(details: ToolCallConfirmationDetails): 'low' | 'medium' | 'high' {
    switch (details.type) {
      case 'edit':
        return 'low';
      case 'exec':
        return details.command.includes('curl') || details.command.includes('wget') ? 'high' : 'medium';
      case 'mcp':
        return 'medium'; // MCP 工具的安全风险中等
      default:
        return 'low';
    }
  }
}

interface EnhancedToolConfirmationDetails extends ToolCallConfirmationDetails {
  riskLevel: 'low' | 'medium' | 'high';
  riskReasons: string[];
  executionContext: ExecutionContext;
  relatedFiles: string[];
  potentialImpact: ImpactAssessment;
}

interface ImpactAssessment {
  scope: 'file' | 'directory' | 'system';
  reversibility: 'easy' | 'difficult' | 'impossible';
  securityImplications: 'low' | 'medium' | 'high';
}

// 增强的确认消息组件
const EnhancedToolConfirmationMessage: React.FC<{
  details: EnhancedToolConfirmationDetails;
  terminalWidth: number;
}> = ({ details, terminalWidth }) => {
  const getRiskColor = (risk: string) => {
    switch (risk) {
      case 'low': return Colors.AccentGreen;
      case 'medium': return Colors.AccentYellow;
      case 'high': return Colors.AccentRed;
      default: return Colors.Foreground;
    }
  };

  return (
    <Box flexDirection="column" padding={1} width={terminalWidth - 2}>
      {/* 风险指示器 */}
      <Box marginBottom={1}>
        <Text color={getRiskColor(details.riskLevel)}>
          Risk Level: {details.riskLevel.toUpperCase()}
        </Text>
        {details.riskReasons.length > 0 && (
          <Box flexDirection="column" marginLeft={2}>
            {details.riskReasons.map((reason, index) => (
              <Text key={index} color={Colors.Gray}>
                • {reason}
              </Text>
            ))}
          </Box>
        )}
      </Box>

      {/* 影响评估 */}
      <Box flexDirection="column" marginBottom={1}>
        <Text bold>Impact Assessment:</Text>
        <Text color={Colors.Gray}>
          Scope: {details.potentialImpact.scope} | 
          Reversibility: {details.potentialImpact.reversibility} | 
          Security: {details.potentialImpact.securityImplications}
        </Text>
      </Box>

      {/* 相关文件 */}
      {details.relatedFiles.length > 0 && (
        <Box flexDirection="column" marginBottom={1}>
          <Text bold>Related Files:</Text>
          {details.relatedFiles.slice(0, 5).map((file, index) => (
            <Text key={index} color={Colors.AccentCyan}>
              • {file}
            </Text>
          ))}
          {details.relatedFiles.length > 5 && (
            <Text color={Colors.Gray}>
              ... and {details.relatedFiles.length - 5} more files
            </Text>
          )}
        </Box>
      )}

      {/* 原始确认组件 */}
      <ToolConfirmationMessage
        confirmationDetails={details}
        terminalWidth={terminalWidth}
      />
    </Box>
  );
};
```

## 错误处理

### 确认流程错误处理

```typescript
// 工具确认错误处理器
class ToolConfirmationErrorHandler {
  static handleConfirmationError(
    error: Error,
    details: ToolCallConfirmationDetails,
    fallbackAction?: () => void
  ): void {
    console.error('Tool confirmation error:', error);

    switch (error.name) {
      case 'ValidationError':
        console.error('Invalid confirmation details:', details);
        this.showValidationError(error.message);
        break;

      case 'PermissionError':
        console.error('Permission denied for tool execution');
        this.showPermissionError();
        break;

      case 'TimeoutError':
        console.error('User confirmation timeout');
        this.handleTimeout(fallbackAction);
        break;

      case 'NetworkError':
        console.error('Network error during tool confirmation');
        this.showNetworkError();
        break;

      default:
        console.error('Unexpected tool confirmation error:', error);
        this.showGenericError(error.message);
    }
  }

  private static showValidationError(message: string): void {
    // 显示验证错误消息
    console.warn(`Validation Error: ${message}`);
  }

  private static showPermissionError(): void {
    // 显示权限错误消息
    console.warn('Permission denied. Please check your authorization settings.');
  }

  private static handleTimeout(fallbackAction?: () => void): void {
    console.warn('User confirmation timeout. Cancelling operation.');
    if (fallbackAction) {
      fallbackAction();
    }
  }

  private static showNetworkError(): void {
    console.warn('Network error occurred. Please check your connection and try again.');
  }

  private static showGenericError(message: string): void {
    console.error(`Tool confirmation failed: ${message}`);
  }
}

// 带错误处理的确认组件
const RobustToolConfirmationMessage: React.FC<ToolConfirmationMessageProps> = (props) => {
  const [error, setError] = useState<string | null>(null);
  const [isProcessing, setIsProcessing] = useState(false);

  const handleConfirmWithErrorHandling = useCallback((outcome: ToolConfirmationOutcome) => {
    setIsProcessing(true);
    setError(null);

    try {
      // 验证确认选择
      if (!Object.values(ToolConfirmationOutcome).includes(outcome)) {
        throw new Error(`Invalid confirmation outcome: ${outcome}`);
      }

      // 调用原始确认处理
      props.confirmationDetails.onConfirm(outcome);
    } catch (error) {
      const errorMessage = error instanceof Error ? error.message : 'Unknown error';
      setError(errorMessage);
      
      ToolConfirmationErrorHandler.handleConfirmationError(
        error instanceof Error ? error : new Error(String(error)),
        props.confirmationDetails,
        () => {
          // 超时或错误时的回退操作
          props.confirmationDetails.onConfirm(ToolConfirmationOutcome.Cancel);
        }
      );
    } finally {
      setIsProcessing(false);
    }
  }, [props.confirmationDetails]);

  // 创建增强的确认详情
  const enhancedDetails = useMemo(() => ({
    ...props.confirmationDetails,
    onConfirm: handleConfirmWithErrorHandling,
  }), [props.confirmationDetails, handleConfirmWithErrorHandling]);

  const enhancedProps = {
    ...props,
    confirmationDetails: enhancedDetails,
  };

  if (error) {
    return (
      <Box
        flexDirection="column"
        borderStyle="round"
        borderColor={Colors.AccentRed}
        padding={1}
        width="100%"
      >
        <Text bold color={Colors.AccentRed}>
          Confirmation Error
        </Text>
        <Text color={Colors.AccentRed}>{error}</Text>
        <Box marginTop={1}>
          <Text>
            The operation has been cancelled. Please try again.
          </Text>
        </Box>
      </Box>
    );
  }

  if (isProcessing) {
    return (
      <Box
        flexDirection="column"
        borderStyle="round"
        borderColor={Colors.AccentYellow}
        padding={1}
        width="100%"
      >
        <Text color={Colors.AccentYellow}>
          Processing confirmation...
        </Text>
      </Box>
    );
  }

  return <ToolConfirmationMessage {...enhancedProps} />;
};
```

## 测试

### 组件测试示例

```typescript
// ToolConfirmationMessage 组件测试
import { render, screen, fireEvent } from '@testing-library/react';
import { ToolConfirmationMessage } from './ToolConfirmationMessage';
import { ToolConfirmationOutcome } from '@google/gemini-cli-core';

describe('ToolConfirmationMessage', () => {
  const mockOnConfirm = jest.fn();

  beforeEach(() => {
    jest.clearAllMocks();
  });

  describe('Edit confirmation', () => {
    const editDetails = {
      type: 'edit' as const,
      fileName: 'test.ts',
      fileDiff: '@@ -1,3 +1,3 @@\n-old line\n+new line',
      isModifying: false,
      onConfirm: mockOnConfirm,
    };

    it('renders edit confirmation correctly', () => {
      render(
        <ToolConfirmationMessage
          confirmationDetails={editDetails}
          terminalWidth={80}
        />
      );

      expect(screen.getByText('Apply this change?')).toBeInTheDocument();
      expect(screen.getByText('Modify with external editor')).toBeInTheDocument();
    });

    it('shows modifying state correctly', () => {
      const modifyingDetails = { ...editDetails, isModifying: true };
      
      render(
        <ToolConfirmationMessage
          confirmationDetails={modifyingDetails}
          terminalWidth={80}
        />
      );

      expect(screen.getByText('Modify in progress:')).toBeInTheDocument();
      expect(screen.getByText('Save and close external editor to continue')).toBeInTheDocument();
    });
  });

  describe('Execution confirmation', () => {
    const execDetails = {
      type: 'exec' as const,
      rootCommand: 'npm',
      command: 'npm install express',
      onConfirm: mockOnConfirm,
    };

    it('renders execution confirmation correctly', () => {
      render(
        <ToolConfirmationMessage
          confirmationDetails={execDetails}
          terminalWidth={80}
        />
      );

      expect(screen.getByText("Allow execution of: 'npm'?")).toBeInTheDocument();
      expect(screen.getByText('npm install express')).toBeInTheDocument();
    });
  });

  describe('MCP confirmation', () => {
    const mcpDetails = {
      type: 'mcp' as const,
      serverName: 'filesystem',
      toolName: 'read_file',
      onConfirm: mockOnConfirm,
    };

    it('renders MCP confirmation correctly', () => {
      render(
        <ToolConfirmationMessage
          confirmationDetails={mcpDetails}
          terminalWidth={80}
        />
      );

      expect(screen.getByText('MCP Server: filesystem')).toBeInTheDocument();
      expect(screen.getByText('Tool: read_file')).toBeInTheDocument();
      expect(screen.getByText(/Always allow all tools from server/)).toBeInTheDocument();
    });
  });

  describe('Keyboard interaction', () => {
    it('handles escape key correctly', () => {
      const details = {
        type: 'info' as const,
        prompt: 'Test prompt',
        onConfirm: mockOnConfirm,
      };

      render(
        <ToolConfirmationMessage
          confirmationDetails={details}
          terminalWidth={80}
          isFocused={true}
        />
      );

      // 模拟 ESC 键按下
      fireEvent.keyDown(document, { key: 'Escape' });
      
      expect(mockOnConfirm).toHaveBeenCalledWith(ToolConfirmationOutcome.Cancel);
    });
  });

  describe('Height calculation', () => {
    it('calculates available height correctly', () => {
      const details = {
        type: 'edit' as const,
        fileName: 'test.ts',
        fileDiff: 'test diff',
        isModifying: false,
        onConfirm: mockOnConfirm,
      };

      render(
        <ToolConfirmationMessage
          confirmationDetails={details}
          terminalWidth={80}
          availableTerminalHeight={20}
        />
      );

      // 验证高度计算逻辑
      // 这需要根据实际的高度计算实现来调整测试
    });
  });
});
```

## 最佳实践

### 用户体验优化

```typescript
// ✅ 推荐：提供清晰的上下文信息
const createInformativeConfirmation = (
  type: string,
  details: any
): ToolCallConfirmationDetails => {
  switch (type) {
    case 'edit':
      return {
        type: 'edit',
        fileName: details.fileName,
        fileDiff: details.diff,
        isModifying: false,
        onConfirm: (outcome) => {
          console.log(`File edit ${outcome} for ${details.fileName}`);
          handleConfirmation(outcome, details);
        }
      };
    
    // 其他类型...
  }
};

// ✅ 推荐：渐进式权限管理
const useProgressivePermissions = () => {
  const [trustLevel, setTrustLevel] = useState<'none' | 'partial' | 'full'>('none');

  const adjustPermissionsBasedOnUsage = (outcome: ToolConfirmationOutcome) => {
    if (outcome === ToolConfirmationOutcome.ProceedAlways) {
      // 增加信任级别
      if (trustLevel === 'none') setTrustLevel('partial');
      else if (trustLevel === 'partial') setTrustLevel('full');
    }
  };

  return { trustLevel, adjustPermissionsBasedOnUsage };
};

// ✅ 推荐：响应式布局适应
const useResponsiveLayout = (terminalWidth: number) => {
  const layoutConfig = useMemo(() => {
    if (terminalWidth < 60) {
      return {
        showFullCommands: false,
        maxQuestionLength: 40,
        compactMode: true,
      };
    } else if (terminalWidth < 100) {
      return {
        showFullCommands: true,
        maxQuestionLength: 70,
        compactMode: false,
      };
    } else {
      return {
        showFullCommands: true,
        maxQuestionLength: 100,
        compactMode: false,
      };
    }
  }, [terminalWidth]);

  return layoutConfig;
};
```

### 性能优化

```typescript
// ✅ 推荐：组件记忆化
const MemoizedToolConfirmationMessage = React.memo(
  ToolConfirmationMessage,
  (prevProps, nextProps) => {
    return (
      prevProps.confirmationDetails.type === nextProps.confirmationDetails.type &&
      prevProps.terminalWidth === nextProps.terminalWidth &&
      prevProps.availableTerminalHeight === nextProps.availableTerminalHeight &&
      prevProps.isFocused === nextProps.isFocused
    );
  }
);

// ✅ 推荐：异步处理优化
const useAsyncConfirmation = () => {
  const [isProcessing, setIsProcessing] = useState(false);

  const handleAsyncConfirmation = useCallback(async (
    outcome: ToolConfirmationOutcome,
    asyncHandler: (outcome: ToolConfirmationOutcome) => Promise<void>
  ) => {
    setIsProcessing(true);
    try {
      await asyncHandler(outcome);
    } catch (error) {
      console.error('Async confirmation handling failed:', error);
    } finally {
      setIsProcessing(false);
    }
  }, []);

  return { isProcessing, handleAsyncConfirmation };
};

// ✅ 推荐：智能缓存策略
const useConfirmationCache = () => {
  const cache = useRef(new Map<string, ToolConfirmationOutcome>());

  const getCachedConfirmation = (details: ToolCallConfirmationDetails): ToolConfirmationOutcome | null => {
    const key = generateCacheKey(details);
    return cache.current.get(key) || null;
  };

  const setCachedConfirmation = (details: ToolCallConfirmationDetails, outcome: ToolConfirmationOutcome) => {
    if (outcome === ToolConfirmationOutcome.ProceedAlways) {
      const key = generateCacheKey(details);
      cache.current.set(key, outcome);
    }
  };

  const generateCacheKey = (details: ToolCallConfirmationDetails): string => {
    switch (details.type) {
      case 'edit':
        return `edit:${details.fileName}`;
      case 'exec':
        return `exec:${details.rootCommand}`;
      case 'mcp':
        return `mcp:${details.serverName}:${details.toolName}`;
      default:
        return `${details.type}:general`;
    }
  };

  return { getCachedConfirmation, setCachedConfirmation };
};
```