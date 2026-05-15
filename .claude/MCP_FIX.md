# InoProShop MCP 修复指南

## 问题

1. MCP 工具每次调用都报 `SCRIPT_ERROR: Spawn failed`，但脚本已送达 IDE
2. `--noUI` 模式下无法检测 IDE 中已打开的项目，且 `projects.open()` 被 Inovance 插件崩溃和设备预约冲突阻塞

## 根因

1. **marker 优先级**：`codesys_interop.js` 中 spawn error 检查先于 `SCRIPT_SUCCESS` marker，Windows 上 `spawn` + `shell: true` + `AbortSignal` 会产生虚假 error
2. **--noUI 进程隔离**：`--noUI` 启动独立 CODESYS 进程，`projects.primary` 为 None，无法看到 IDE 中已打开的项目。尝试 `projects.open()` 又与 IDE 的设备预约冲突
3. **Inovance 插件**：`--noUI` 模式下 `InoObjectManager.OnProjectLoaded` 因缺少 UI 组件崩溃（NullReferenceException）

## 修复步骤

编辑文件：
```
%APPDATA%\npm\node_modules\@codesys\mcp-toolkit\dist\codesys_interop.js
```

### 修复 1：SCRIPT_SUCCESS marker 优先级 (~第197行)

将 `// --- Success Determination Logic ---` 部分改为 marker 优先：

```javascript
const hasSuccessMarker = output.includes('SCRIPT_SUCCESS') || stderrOutput.includes('SCRIPT_SUCCESS');
const hasErrorMarker = output.includes('SCRIPT_ERROR') || stderrOutput.includes('SCRIPT_ERROR');

if (hasSuccessMarker) {
    success = true;  // 忽略 spawn/shell errors
} else if (hasErrorMarker) {
    success = false;
} else if (spawnResult.error) {
    // 只在无 marker 时才判断 spawn error
}
```

### 修复 2：去掉 --noUI（~第113行）

`--noUI` 启动独立进程，无法检测 IDE 中已打开的项目。去掉它以连接到运行中的 IDE：

```javascript
// 去掉 --noUI（让脚本进程连接到运行中的 IDE）
const fullCommandString = `${quotedExePath} ${profileArg} ${scriptArg}`;
```

### 使用流程

1. 启动 InoProShop IDE
2. 在 IDE 中手动打开项目
3. 通过 MCP 工具（save、compile、create_pou 等）操作项目

## MCP 配置 (.mcp.json)

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
