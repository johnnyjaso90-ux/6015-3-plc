# MCP Toolkit 适配 InoProShop SP11 (CODESYS 3.5.11) 修复指南

## 环境

- InoProShop: 1.9.1.6 (基于 CODESYS 3.5.11.80, Python 2.7.7)
- MCP Toolkit: @codesys/mcp-toolkit 1.1.16
- MCP 配置文件: `.claude/mcp.json`

## 问题根因

`codesys-mcp-toolkit` 要求 CODESYS SP21+。在 SP11 上有三个兼容性问题：

1. **print() 输出不走 stdout** — 脚本引擎的 `print()` 输出到 CODESYS 内部消息窗口，不经过父进程 stdout/stderr
2. **`open(path, 'w', 0)` 无缓冲模式失效** — 文件被创建但内容无法写入（Python 2.7.7 bug）
3. **GUI 模式下进程不退出** — InoProShop GUI 保持运行，进程不会自然退出

## 需要修改的文件

所有文件位于: `C:\Users\<用户名>\AppData\Roaming\npm\node_modules\@codesys\mcp-toolkit\dist\`

### 1. codesys_interop.js

#### 1.1 添加 fileOutput 变量声明 (约第 91 行)

在 `let exitCode = null;` 之后添加一行:

```javascript
let fileOutput = ''; // For capturing output from redirected stdout file (older CODESYS compatibility)
```

#### 1.2 添加 outputFilePath 变量 (约第 89 行)

在 `const tempFilePath = ...` 之后添加:

```javascript
const outputFilePath = tempFilePath + '.out'; // Output file for capturing redirected stdout
```

#### 1.3 设置环境变量 MCP_OUTPUT_PATH (约第 125 行)

在 `spawnEnv.PATH = ...` 之后添加:

```javascript
spawnEnv.MCP_OUTPUT_PATH = outputFilePath;
process.stderr.write(`INTEROP ENV: MCP_OUTPUT_PATH = ${outputFilePath}\n`);
```

#### 1.4 添加输出文件轮询 (约第 155 行, timeoutId 之后)

将 resolveOnce 函数改为同时清理 interval:

```javascript
const resolveOnce = (result) => {
    if (!resolved) {
        resolved = true;
        clearTimeout(timeoutId);
        if (outputPollInterval) clearInterval(outputPollInterval);
        if (!childProcess.killed) {
            try { childProcess.kill('SIGTERM'); } catch (_) {}
        }
        resolve(result);
    }
};
```

在 timeoutId 之后添加文件轮询:

```javascript
let outputPollInterval = null;
let lastFileSize = 0;
outputPollInterval = setInterval(() => {
    try {
        if (fs.existsSync(outputFilePath)) {
            const stats = fs.statSync(outputFilePath);
            if (stats.size > lastFileSize) {
                lastFileSize = stats.size;
                const content = fs.readFileSync(outputFilePath, 'utf-8');
                if (content.includes('SCRIPT_SUCCESS')) {
                    process.stderr.write('INTEROP: Detected SCRIPT_SUCCESS in output file, resolving early.\n');
                    resolveOnce({ code: 0, stdout: stdoutData + '\n' + content, stderr: stderrData });
                } else if (content.includes('SCRIPT_ERROR')) {
                    process.stderr.write('INTEROP: Detected SCRIPT_ERROR in output file, resolving early.\n');
                    resolveOnce({ code: 1, stdout: stdoutData + '\n' + content, stderr: stderrData });
                }
            }
        }
    } catch (_) { /* file not ready yet */ }
}, 2000);
```

#### 1.5 清理 interval — 在 close, error, abort 事件中添加

在 `childProcess.on('close', ...)` 中添加: `if (outputPollInterval) clearInterval(outputPollInterval);`

在 `childProcess.on('error', ...)` 中添加: `if (outputPollInterval) clearInterval(outputPollInterval);`

在 `timeoutSignal.addEventListener('abort', ...)` 中添加: `if (outputPollInterval) clearInterval(outputPollInterval);`

#### 1.6 读取输出文件并合并到结果 (约第 218 行, exitCode 赋值后)

在 `exitCode = spawnResult.code;` 之后添加:

```javascript
let fileOutput = '';
try {
    if (fs.existsSync(outputFilePath)) {
        fileOutput = fs.readFileSync(outputFilePath, 'utf-8');
        process.stderr.write(`INTEROP: Read output file (${fileOutput.length} bytes)\n`);
    } else {
        process.stderr.write(`INTEROP: Output file not found: ${outputFilePath}\n`);
    }
} catch (fileErr) {
    process.stderr.write(`INTEROP: Failed to read output file: ${fileErr.message}\n`);
}
const combinedOutput = fileOutput + '\n' + output;
```

#### 1.7 更新 marker 检测使用 combinedOutput (约第 240 行)

```javascript
const hasSuccessMarker = combinedOutput.includes(SCRIPT_SUCCESS_MARKER) || stderrOutput.includes(SCRIPT_SUCCESS_MARKER);
const hasErrorMarker = combinedOutput.includes(SCRIPT_ERROR_MARKER) || stderrOutput.includes(SCRIPT_ERROR_MARKER);
```

#### 1.8 更新最终输出和文件清理 (约第 299 行)

在最终输出处理前添加:

```javascript
if (fileOutput) {
    output = combinedOutput;
}
```

在 return 之前添加输出文件清理:

```javascript
try {
    if (fs.existsSync(outputFilePath)) {
        yield (0, promises_1.unlink)(outputFilePath);
        process.stderr.write(`INTEROP: Output file deleted: ${outputFilePath}\n`);
    }
} catch (cleanupErr) {
    process.stderr.write(`INTEROP: Failed to delete output file: ${cleanupErr.message}\n`);
}
```

---

### 2. server.js

在三个 Python 脚本模板中，将 `sys.stdout` 重定向代码改为使用**自动刷新的包装类**。

#### 2.1 ENSURE_PROJECT_OPEN_PYTHON_SNIPPET (约第 134 行)

在 `import os` 之后, `import time` 之前插入:

```python
# --- Redirect stdout to file for older CODESYS compatibility ---
__mcp_out_ensure__ = os.environ.get('MCP_OUTPUT_PATH')
if __mcp_out_ensure__:
    try:
        class _AutoFlushFile:
            def __init__(self, f): self.f = f
            def write(self, s): self.f.write(s); self.f.flush()
            def flush(self): self.f.flush()
            def __getattr__(self, n): return getattr(self.f, n)
        _f_out_ensure = _AutoFlushFile(open(__mcp_out_ensure__, 'w'))
        sys.stdout = _f_out_ensure
    except: pass
# --- End stdout redirect ---
```

#### 2.2 CHECK_STATUS_SCRIPT (约第 328 行)

在 `import sys, scriptengine ... traceback` 之后插入:

```python
# --- Redirect stdout to file for older CODESYS compatibility ---
__mcp_out__ = os.environ.get('MCP_OUTPUT_PATH')
if __mcp_out__:
    try:
        class _AutoFlushFile2:
            def __init__(self, f): self.f = f
            def write(self, s): self.f.write(s); self.f.flush()
            def flush(self): self.f.flush()
            def __getattr__(self, n): return getattr(self.f, n)
        _f_out = _AutoFlushFile2(open(__mcp_out__, 'w'))
        sys.stdout = _f_out
    except: pass
# --- End stdout redirect ---
```

#### 2.3 CREATE_PROJECT_SCRIPT_TEMPLATE (约第 384 行)

在 `import sys, scriptengine ... traceback` 之后插入:

```python
# --- Redirect stdout to file for older CODESYS compatibility ---
__mcp_out_create__ = os.environ.get('MCP_OUTPUT_PATH')
if __mcp_out_create__:
    try:
        class _AutoFlushFile3:
            def __init__(self, f): self.f = f
            def write(self, s): self.f.write(s); self.f.flush()
            def flush(self): self.f.flush()
            def __getattr__(self, n): return getattr(self.f, n)
        _f_out_create = _AutoFlushFile3(open(__mcp_out_create__, 'w'))
        sys.stdout = _f_out_create
    except: pass
# --- End stdout redirect ---
```

---

## MCP 配置文件

`.claude/mcp.json`:
```json
{
  "mcpServers": {
    "inoproshop": {
      "type": "stdio",
      "command": "C:\\Users\\<用户名>\\AppData\\Roaming\\npm\\codesys-mcp-tool.cmd",
      "args": [
        "--codesys-path",
        "C:\\Inovance Control\\InoProShop\\CODESYS\\Common\\InoProShop.exe",
        "--codesys-profile",
        "InoProShop(V1.9.1.6)",
        "--workspace",
        "<项目路径>"
      ]
    }
  }
}
```

## 部署步骤

1. 定位 npm 全局包路径: `%APPDATA%\npm\node_modules\@codesys\mcp-toolkit\dist\`
2. 按上述修改编辑 `codesys_interop.js` 和 `server.js`
3. 配置 `.claude/mcp.json`（替换用户名和项目路径）
4. **重新加载 VS Code 窗口** (Ctrl+Shift+P → Reload Window) 使 MCP 服务器加载新代码
5. 测试: 调用 `save_project` 验证，应在大约 30-60 秒内返回成功

## 已验证的 MCP 工具

| 工具 | 状态 |
|------|------|
| save_project | 正常 |
| open_project | 正常 |
| compile_project | 正常 |
| project-status (resource) | 正常 |

## 注意事项

- 每次 MCP 工具调用都会启动一个新的 InoProShop 实例，执行完脚本后关闭
- 连续调用时需确保前一个 InoProShop 进程已退出，否则可能冲突
- 如果修改后 MCP 工具仍超时，请确认已 Reload Window
