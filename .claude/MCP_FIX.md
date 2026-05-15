# InoProShop MCP Spawn Failed 修复指南

## 问题

MCP 工具每次调用都报 `SCRIPT_ERROR: Spawn failed: The operation was aborted`，但实际脚本已送达 IDE（日志显示 `AddedToUI` 和 `OnPrimaryProjectSwitched`）。

## 根因

1. `codesys_interop.js` 中成功/失败判断逻辑顺序错误：spawn error 检查先于 `SCRIPT_SUCCESS` marker 检查。Windows 上 `child_process.spawn` 配合 `shell: true` + `AbortSignal` 会产生虚假的 error 事件。
2. 之前尝试去掉 `--noUI` 来绕过 Inovance 插件崩溃，导致每次 MCP 调用都弹出 IDE 窗口。

## 修复步骤

编辑文件（全局 npm 包路径）：
```
%APPDATA%\npm\node_modules\@codesys\mcp-toolkit\dist\codesys_interop.js
```

### 修复 1：SCRIPT_SUCCESS marker 优先级 (~第197行)

将 `// --- Success Determination Logic ---` 部分改为 marker 优先：

**修改前**：spawn error → shell error → ... → markers
**修改后**：markers 优先 → spawn error → shell error → ... → exit code

核心逻辑：
```javascript
const hasSuccessMarker = output.includes('SCRIPT_SUCCESS') || stderrOutput.includes('SCRIPT_SUCCESS');
const hasErrorMarker = output.includes('SCRIPT_ERROR') || stderrOutput.includes('SCRIPT_ERROR');

if (hasSuccessMarker) {
    success = true;  // 忽略 spawn/shell errors
} else if (hasErrorMarker) {
    success = false;
} else if (spawnResult.error) {
    // ... 只在无 marker 时才判断 spawn error
}
```

### 修复 2：恢复 --noUI 标志 (~第113行)

```javascript
// 修改前（会弹出 IDE 窗口）
const fullCommandString = `${quotedExePath} ${profileArg} ${scriptArg}`;

// 修改后（无头模式，不弹窗）
const fullCommandString = `${quotedExePath} ${profileArg} --noUI ${scriptArg}`;
```

### 修复 3：如果 --noUI 模式下 Inovance 插件崩溃

若 `--noUI` 模式下 `projects.open()` 报 NullReferenceException（InoObjectManager.OnProjectLoaded 崩溃），可以去掉 `--noUI`，改为在 IDE 已打开项目的情况下使用 MCP。但每次操作仍会启动新进程，可能触发 profile 选择弹窗。

## MCP 配置

```json
{
  "mcpServers": {
    "inoproshop": {
      "type": "stdio",
      "command": "codesys-mcp-tool.cmd",
      "args": [
        "--codesys-path", "C:\\Inovance Control\\InoProShop\\CODESYS\\Common\\InoProShop.exe",
        "--codesys-profile", "InoProShop(V1.9.1.6)",
        "--workspace", "<项目目录>"
      ]
    }
  }
}
```

修改后需重启 Claude Code 会话使改动生效。
