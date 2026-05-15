# InoProShop MCP 修复指南

## 问题

1. MCP 工具报 `SCRIPT_ERROR: Spawn failed: The operation was aborted`
2. `--noUI` 模式下 `projects.open()` 触发 Inovance 插件崩溃（NullReferenceException）
3. 去掉 `--noUI` 后 GUI 进程不退，导致 MCP 操作超时（120s）

## 根因

1. **marker 优先级**：spawn error 检查先于 `SCRIPT_SUCCESS` marker，Windows 上 `spawn` + `shell: true` + `AbortSignal` 产生虚假 error
2. **Inovance 插件**：`--noUI` 模式下 `InoObjectManager.OnProjectLoaded` 缺少 UI 组件崩溃，`projects.open()` 彻底失败
3. **GUI 进程不退**：去掉 `--noUI` 后 InoProShop.exe 作为 GUI 进程不会自动退出，导致一直等到超时

## 修复策略

**去掉 `--noUI`，在 stdout 检测到 marker 后立即结束进程**，不再等待进程自然退出。

## 修复步骤

编辑以下文件（换电脑后按此步骤操作）：

### 文件 1：codesys_interop.js

路径：`%APPDATA%\npm\node_modules\@codesys\mcp-toolkit\dist\codesys_interop.js`

#### 修复 A：去掉 --noUI（~第113行）

```javascript
// 去掉 --noUI，让脚本进程连接到运行中的 IDE
const fullCommandString = `${quotedExePath} ${profileArg} ${scriptArg}`;
```

#### 修复 B：marker 检测后立即结束进程（~第141行）

在 `timeoutId` 之前添加 `resolveOnce`，在 stdout data 事件中检测 marker 并立即 resolve：

```javascript
let resolved = false;
const resolveOnce = (result) => {
    if (!resolved) {
        resolved = true;
        clearTimeout(timeoutId);
        if (!childProcess.killed) {
            try { childProcess.kill('SIGTERM'); } catch (_) {}
        }
        resolve(result);
    }
};
// ...
childProcess.stdout.on('data', (data) => {
    // ... 收集数据 ...
    if (stdoutData.includes('SCRIPT_SUCCESS') || stderrData.includes('SCRIPT_SUCCESS')) {
        resolveOnce({ code: 0, stdout: stdoutData, stderr: stderrData });
    } else if (stdoutData.includes('SCRIPT_ERROR') || stderrData.includes('SCRIPT_ERROR')) {
        resolveOnce({ code: 1, stdout: stdoutData, stderr: stderrData });
    }
});
```

#### 修复 C：移除 AbortSignal + 增加超时（~第131行）

```javascript
const timeoutDuration = 120000;
const childProcess = spawn(fullCommandString, [], {
    windowsHide: true,
    cwd: codesysDir,
    env: spawnEnv,
    shell: true
    // 注意: 不传 signal 参数（Windows shell:true 兼容性）
});
```

#### 修复 D：marker 优先判断（~第197行）

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

### 文件 2：ensure_project_open.py

路径：`%APPDATA%\npm\node_modules\@codesys\mcp-toolkit\dist\scripts\ensure_project_open.py`

#### 修复 E：open() 异常后检查项目是否已加载（~第146行）

```python
except Exception as open_err:
    print("ERROR: Exception during projects.open call: %s" % open_err)
    traceback.print_exc()
    # Inovance 插件崩溃可能只是回调副作用，项目可能已加载
    try:
        post_err_primary = script_engine.projects.primary
        if post_err_primary:
            post_err_path = os.path.normcase(os.path.abspath(post_err_primary.path))
            if post_err_path == normalized_target_path:
                _ = len(post_err_primary.get_children(False))
                return post_err_primary
    except:
        pass
```

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

## 使用流程

1. 启动 InoProShop IDE，手动打开项目
2. 在 Claude Code 中使用 MCP 工具（save、compile、create_pou 等）
3. MCP 会启动一个短暂的 InoProShop 进程连接到 IDE，完成操作后立即退出

修改后需重启 Claude Code 会话使改动生效。
