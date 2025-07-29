# InputPrompt 组件文档

## 概述

`InputPrompt` 是 Gemini CLI 的核心输入组件，负责处理用户输入、自动完成、历史记录导航和命令建议。它集成了多种输入模式（普通模式和 Shell 模式），支持文件路径自动完成、斜杠命令建议和剪贴板图像处理。

## 主要功能

- 多行文本输入和编辑
- 智能自动完成（文件路径、斜杠命令）
- 输入历史记录导航
- Shell 模式集成
- 剪贴板图像处理
- Vim 模式支持
- 键盘快捷键处理

## 接口定义

### `InputPromptProps`

```typescript
interface InputPromptProps {
  buffer: TextBuffer;                    // 文本缓冲区对象
  onSubmit: (value: string) => void;     // 提交处理函数
  userMessages: readonly string[];       // 用户历史消息
  onClearScreen: () => void;             // 清屏函数
  config: Config;                        // 应用配置
  slashCommands: readonly SlashCommand[]; // 可用的斜杠命令
  commandContext: CommandContext;        // 命令上下文
  placeholder?: string;                  // 占位符文本
  focus?: boolean;                       // 是否获得焦点
  inputWidth: number;                    // 输入区域宽度
  suggestionsWidth: number;              // 建议区域宽度
  shellModeActive: boolean;              // Shell 模式是否激活
  setShellModeActive: (value: boolean) => void; // Shell 模式切换函数
  vimHandleInput?: (key: Key) => boolean; // Vim 输入处理函数
}
```

## 组件状态

### 本地状态

```typescript
const [justNavigatedHistory, setJustNavigatedHistory] = useState(false);
```

**功能**: 跟踪是否刚刚通过历史记录导航，用于控制自动完成的显示行为

## 核心 Hooks 集成

### `useCompletion`

**功能**: 提供智能自动完成功能

```typescript
const completion = useCompletion(
  buffer,
  config.getTargetDir(),
  slashCommands,
  commandContext,
  config,
);
```

**提供的功能**:

- 文件路径自动完成
- 斜杠命令建议
- 动态建议过滤
- 建议选择处理

### `useShellHistory`

**功能**: Shell 命令历史记录管理

```typescript
const shellHistory = useShellHistory(config.getProjectRoot());
```

### `useInputHistory`

**功能**: 用户输入历史记录导航

```typescript
const inputHistory = useInputHistory({
  userMessages,
  onSubmit: handleSubmitAndClear,
  isActive: (!completion.showSuggestions || completion.suggestions.length === 1) && !justNavigatedHistory,
  setText: customSetTextAndResetCompletionSignal,
  getText: () => buffer.text,
});
```

**激活条件**:

- 自动完成建议不显示或只有一个建议
- 未刚刚进行历史记录导航

### `useKeypress`

**功能**: 键盘事件处理

```typescript
useKeypress(
  (key: Key) => {
    // Vim 模式处理
    if (vimHandleInput) {
      const handled = vimHandleInput(key);
      if (handled) return;
    }
    
    // 标准键盘处理逻辑
    handleKeypress(key);
  },
  focus,
);
```

## 核心功能函数

### `handleSubmitAndClear`

**功能**: 处理输入提交和清理

**实现**:

```typescript
const handleSubmitAndClear = useCallback(
  (submittedValue: string) => {
    if (shellModeActive) {
      shellHistory.addCommandToHistory(submittedValue);
    }
    // 在调用 onSubmit 之前清空缓冲区，防止重复提交
    buffer.setText('');
    onSubmit(submittedValue);
    resetCompletionState();
  },
  [onSubmit, buffer, resetCompletionState, shellModeActive, shellHistory],
);
```

**执行步骤**:

1. 如果在 Shell 模式，将命令添加到 Shell 历史
2. 清空文本缓冲区
3. 调用提交处理函数
4. 重置自动完成状态

### `customSetTextAndResetCompletionSignal`

**功能**: 设置文本并标记历史导航状态

```typescript
const customSetTextAndResetCompletionSignal = useCallback(
  (newText: string) => {
    buffer.setText(newText);
    setJustNavigatedHistory(true);
  },
  [buffer, setJustNavigatedHistory],
);
```

### `handleKeypress`

**功能**: 处理键盘按键事件

**主要处理逻辑**:

#### Tab 键 - 自动完成

```typescript
if (key.name === 'tab') {
  if (completion.showSuggestions && completion.suggestions.length > 0) {
    if (completion.selectedIndex !== -1) {
      completion.acceptSuggestion();
    } else {
      completion.selectNext();
    }
    return;
  }
}
```

#### Enter 键 - 提交或换行

```typescript
if (key.name === 'return') {
  if (completion.showSuggestions && completion.selectedIndex !== -1) {
    completion.acceptSuggestion();
    return;
  }
  
  // Shift+Enter 添加换行
  if (key.shift) {
    buffer.insertAtCursor('\n');
    return;
  }
  
  // 普通 Enter 提交
  const text = buffer.text.trim();
  if (text.length > 0) {
    handleSubmitAndClear(text);
  }
  return;
}
```

#### 方向键 - 导航

```typescript
if (key.name === 'up' || key.name === 'down') {
  if (completion.showSuggestions) {
    // 在建议中导航
    if (key.name === 'up') {
      completion.selectPrevious();
    } else {
      completion.selectNext();
    }
    return;
  }
  
  // 输入历史导航
  if (key.name === 'up') {
    inputHistory.goToPrevious();
  } else {
    inputHistory.goToNext();
  }
  return;
}
```

#### Ctrl+V - 剪贴板处理

```typescript
if (key.ctrl && key.name === 'v') {
  handlePaste();
  return;
}
```

### `handlePaste`

**功能**: 处理剪贴板粘贴操作

**实现流程**:

1. **检查剪贴板内容类型**:
   ```typescript
   const hasImage = await clipboardHasImage();
   ```

2. **图像处理**:
   ```typescript
   if (hasImage) {
     const imageId = Date.now().toString();
     const imagePath = path.join(config.getTargetDir(), `.gemini-images/clipboard-${imageId}.png`);
     
     const success = await saveClipboardImage(imagePath);
     if (success) {
       buffer.insertAtCursor(`@${imagePath} `);
       await cleanupOldClipboardImages(config.getTargetDir());
     }
   }
   ```

3. **文本处理**: 如果不是图像，使用标准文本粘贴

## 渲染逻辑

### 主要结构

```typescript
return (
  <Box flexDirection="column">
    {/* 输入提示区域 */}
    <Box marginBottom={showSuggestions ? 0 : 1}>
      <Text color={promptColor}>
        {promptPrefix}
      </Text>
      <Text>{displayText}</Text>
      {showCursor && <Text color={cursorColor}>█</Text>}
    </Box>
    
    {/* 自动完成建议 */}
    {showSuggestions && (
      <SuggestionsDisplay
        suggestions={completion.suggestions}
        selectedIndex={completion.selectedIndex}
        width={suggestionsWidth}
        maxSuggestions={maxSuggestions}
      />
    )}
  </Box>
);
```

### 显示状态计算

#### 提示符颜色和前缀

```typescript
const promptColor = shellModeActive ? Colors.AccentBlue : Colors.AccentGreen;
const promptPrefix = shellModeActive ? '$ ' : '> ';
```

#### 文本显示处理

```typescript
const displayText = buffer.text || placeholder;
const showCursor = focus && !completion.showSuggestions;
```

#### 建议显示条件

```typescript
const showSuggestions = completion.showSuggestions && 
                       completion.suggestions.length > 0 && 
                       !justNavigatedHistory;
```

## 特殊功能

### Shell 模式集成

- **模式切换**: 通过 `shellModeActive` 属性控制
- **历史记录**: Shell 命令单独管理历史记录
- **视觉反馈**: 不同的提示符颜色和前缀

### 文件路径自动完成

- **实时建议**: 基于当前输入的文件路径建议
- **@ 符号触发**: 自动检测 @ 符号并提供文件路径建议
- **相对路径**: 基于项目根目录的相对路径建议

### 斜杠命令建议

- **命令过滤**: 基于输入前缀过滤可用命令
- **上下文感知**: 根据当前命令上下文提供相关建议
- **描述显示**: 显示命令描述帮助用户选择

### 剪贴板图像处理

- **图像检测**: 自动检测剪贴板中的图像内容
- **图像保存**: 将图像保存到项目的 `.gemini-images` 目录
- **路径插入**: 自动插入图像路径到输入中
- **清理机制**: 定期清理旧的剪贴板图像

### Vim 模式支持

- **模式集成**: 通过 `vimHandleInput` 函数集成 Vim 模式
- **优先处理**: Vim 模式键盘事件优先处理
- **模式切换**: 支持 INSERT 和 NORMAL 模式切换

## 性能优化

### useCallback 优化

```typescript
const handleSubmitAndClear = useCallback(/* ... */, [dependencies]);
const customSetTextAndResetCompletionSignal = useCallback(/* ... */, [dependencies]);
const handleKeypress = useCallback(/* ... */, [dependencies]);
```

### 状态管理优化

- **最小化重新渲染**: 合理使用 useCallback 和状态依赖
- **异步操作**: 剪贴板操作使用异步处理避免阻塞 UI
- **内存清理**: 及时清理不需要的资源

## 使用示例

### 基本用法

```typescript
<InputPrompt
  buffer={textBuffer}
  onSubmit={handleUserInput}
  userMessages={previousMessages}
  onClearScreen={clearScreen}
  config={appConfig}
  slashCommands={availableCommands}
  commandContext={currentContext}
  inputWidth={80}
  suggestionsWidth={60}
  shellModeActive={false}
  setShellModeActive={setShellMode}
  placeholder="Type your message..."
  focus={true}
/>
```

### Shell 模式用法

```typescript
<InputPrompt
  // ... 其他属性
  shellModeActive={true}
  setShellModeActive={setShellMode}
  placeholder="Enter shell command..."
/>
```

### Vim 模式集成

```typescript
<InputPrompt
  // ... 其他属性
  vimHandleInput={vimKeyHandler}
/>
```

## 依赖关系

### 外部依赖

- `react` - React hooks 和组件
- `ink` - 终端 UI 组件
- `chalk` - 终端颜色处理
- `string-width` - 字符串宽度计算

### 内部依赖

- `./SuggestionsDisplay.js` - 建议显示组件
- `../hooks/useInputHistory.js` - 输入历史 hook
- `../hooks/useCompletion.js` - 自动完成 hook
- `../hooks/useKeypress.js` - 按键处理 hook
- `./shared/text-buffer.js` - 文本缓冲区
- `../utils/clipboardUtils.js` - 剪贴板工具
- `../utils/textUtils.js` - 文本处理工具

## 错误处理

### 剪贴板错误

- **图像保存失败**: 静默失败，不影响用户输入
- **权限错误**: 优雅降级到文本粘贴

### 文件系统错误

- **路径访问错误**: 自动完成功能降级
- **图像目录创建失败**: 跳过图像保存功能

### 输入验证

- **空输入**: 不执行提交操作
- **无效命令**: 由上层组件处理验证