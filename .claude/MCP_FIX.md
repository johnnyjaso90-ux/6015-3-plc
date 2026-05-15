# InoProShop MCP 修复指南

## 问题

1. MCP 工具每次调用都报 `SCRIPT_ERROR: Spawn failed`，但脚本已送达 IDE
2. `--noUI` 模式下 `projects.open()` 触发 Inovance 插件崩溃（NullReferenceException）
3. 去掉 `--noUI` 会导致每次 MCP 调用都弹出新的 IDE 窗口

## 根因

1. **marker 优先级**：`codesys_interop.js` 中 spawn error 检查先于 `SCRIPT_SUCCESS` marker
2. **AbortSignal + shell:true**：Node.js 的 `child_process.spawn` 在 Windows 上配合 `shell: true` + `AbortSignal` 会产生虚假的 "aborted" 错误
3. **Inovance 插件**：`--noUI` 模式下 `InoObjectManager.OnProjectLoaded` 因缺少 UI 组件崩溃，但崩溃发生在项目加载**之后**（非致命回调副作用）

## 修复步骤

编辑以下两个文件：

### 文件 1：codesys_interop.js
路径：`%APPDATA%\npm\node_modules\@codesys\mcp-toolkit\dist\codesys_interop.js`

#### 修复 A：保持 --noUI（~第113行）
```javascript
// 保持 --noUI，避免弹出 IDE 窗口
const fullCommandString = `${quotedExePath} ${profileArg} --noUI ${scriptArg}`;
```

#### 修复 B：移除 AbortSignal + 增加超时（~第131行）
```javascript
// 不要传 signal 给 spawn（Windows shell:true 兼容性问题）
// 超时改为 120 秒，给项目加载留足时间
const timeoutDuration = 120000;
const childProcess = spawn(fullCommandString, [], {
    windowsHide: true,
    cwd: codesysDir,
    env: spawnEnv,
    shell: true
    // 注意: 不传 signal 参数
});
```

#### 修复 C：marker 优先判断（~第197行）
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

#### 修复 D：open() 异常后检查项目是否已加载（~第146行）
```python
except Exception as open_err:
    print("ERROR: Exception during projects.open call: %s" % open_err)
    traceback.print_exc()
    # 插件崩溃可能发生在项目加载之后的回调中，
    # 检查项目是否实际上已变为 primary
    try:
        post_err_primary = script_engine.projects.primary
        if post_err_primary:
            post_err_path = os.path.normcase(os.path.abspath(post_err_primary.path))
            if post_err_path == normalized_target_path:
                print("DEBUG: Project became primary despite open() exception.")
                try:
                    _ = len(post_err_primary.get_children(False))
                    print("DEBUG: Project accessible. Proceeding.")
                    return post_err_primary
                except Exception as access_err:
                    print("WARN: Not accessible: %s" % access_err)
    except Exception as post_check_err:
        print("WARN: %s" % post_check_err)
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

1. 确保 InoProShop IDE **未运行**（MCP 会通过 --noUI 自己启动 headless 实例）
2. 直接使用 MCP 工具（open_project、compile、create_pou 等）
3. 不需要手动打开 IDE

修改后需重启 Claude Code 会话使改动生效。
