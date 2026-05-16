# MCP Toolkit 适配 InoProShop SP11 (CODESYS 3.5.11) 修复指南

## 环境

- InoProShop: 1.9.1.6 (基于 CODESYS 3.5.11.80, Python 2.7.7)
- MCP Toolkit: @codesys/mcp-toolkit 1.1.16
- MCP 配置文件: `.claude/mcp.json`

## 问题根因

`codesys-mcp-toolkit` 要求 CODESYS SP21+。在 SP11 上有四个兼容性问题：

1. **print() 输出不走 stdout** — 脚本引擎的 `print()` 输出到 CODESYS 内部消息窗口，不经过父进程 stdout/stderr
2. **`open(path, 'w', 0)` 无缓冲模式失效** — 文件被创建但内容无法写入（Python 2.7.7 bug）
3. **GUI 模式下进程不退出** — InoProShop GUI 保持运行，进程不会自然退出
4. **ASCII 编码导致中文字符报错** — 项目中有中文 POU 名称时，`open(path, 'w')` 默认 ASCII 编码会触发 `UnicodeEncodeError`，需用 `codecs.open(path, 'w', 'utf-8')`

## 架构说明

每次 MCP 工具调用启动独立的 InoProShop 进程（`--runscript`），执行完脚本后进程被 kill。约需 30 秒/次（InoProShop 启动 + 项目加载 + 脚本执行）。

## Daemon 常驻进程方案（已废弃）

**尝试过**：首次 open_project 启动后台 InoProShop 常驻，Python 脚本进入 `while True` 循环，通过文件读写接收命令（CMD:SAVE / CMD:COMPILE / CMD:SET_CODE 等），后续操作复用同一进程，免去启动开销。

**废弃原因**：CODESYS 的 Python 脚本引擎运行在主线程，`while True: time.sleep(N)` 循环会持续占用主线程，导致 GUI 消息泵无法处理窗口消息。表现为：
- 鼠标移到 InoProShop 窗口上时转圈圈
- 窗口内任何按钮/菜单都无法点击
- 无法通过 X 按钮关闭窗口，只能 taskkill 强杀

即使将 sleep 从 0.5s 加大到 3s，情况没有改善——只要循环在主线程运行，GUI 就得不到处理时间。

**ProcessMessageLoop 方案也失败**：在循环中调用 `scriptengine.system.process_messageloop()` 手动泵消息，InoProShop 窗口依然无响应。SP11 的 process_messageloop() 无法在脚本主循环中有效工作。

**结论**：在 InoProShop 1.9.1.6 / CODESYS 3.5.11.80 上，无法实现不阻塞 GUI 的常驻脚本循环。只能用单次脚本模式。

## 需要修改的文件

所有文件位于: `C:\Users\<用户名>\AppData\Roaming\npm\node_modules\@codesys\mcp-toolkit\dist\`

---

### 1. codesys_interop.js — 输出文件轮询 + marker 检测

#### 1.1 添加 fileOutput 变量声明 (约第 91 行)

在 `let exitCode = null;` 之后添加:

```javascript
let fileOutput = ''; // For capturing output from redirected stdout file
```

#### 1.2 添加 outputFilePath 变量 (约第 89 行)

在 `const tempFilePath = ...` 之后添加:

```javascript
const outputFilePath = tempFilePath + '.out';
```

#### 1.3 设置环境变量 MCP_OUTPUT_PATH (约第 125 行)

在 `spawnEnv.PATH = ...` 之后添加:

```javascript
spawnEnv.MCP_OUTPUT_PATH = outputFilePath;
process.stderr.write(`INTEROP ENV: MCP_OUTPUT_PATH = ${outputFilePath}\n`);
```

#### 1.4 修改 resolveOnce + 添加输出文件轮询 (约第 145-181 行)

修改 resolveOnce 清理 interval:

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

#### 1.5 在 close, error, abort 事件中清理 interval

在 `childProcess.on('close', ...)`, `childProcess.on('error', ...)`, `timeoutSignal.addEventListener('abort', ...)` 中都添加:

```javascript
if (outputPollInterval) clearInterval(outputPollInterval);
```

#### 1.6 读取输出文件 (约第 248 行, exitCode 赋值后)

```javascript
fileOutput = '';
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

#### 1.7 更新 marker 检测使用 combinedOutput (约第 265 行)

```javascript
const hasSuccessMarker = combinedOutput.includes(SCRIPT_SUCCESS_MARKER) || stderrOutput.includes(SCRIPT_SUCCESS_MARKER);
const hasErrorMarker = combinedOutput.includes(SCRIPT_ERROR_MARKER) || stderrOutput.includes(SCRIPT_ERROR_MARKER);
```

#### 1.8 最终输出 + 清理输出文件 (约第 327 行)

```javascript
if (fileOutput) { output = combinedOutput; }
```

在 return 之前添加:

```javascript
try {
    if (fs.existsSync(outputFilePath)) {
        yield (0, promises_1.unlink)(outputFilePath);
    }
} catch (cleanupErr) { /* ignore cleanup errors */ }
```

---

### 2. server.js — stdout 重定向 + UTF-8 编码

在三个 Python 脚本模板中添加 stdout 重定向。每个脚本需添加 `import codecs` 和自动刷新包装类。

#### 2.1 ENSURE_PROJECT_OPEN_PYTHON_SNIPPET (约第 134 行)

在 `import os` 之后, `import time` 之前插入:

```python
import codecs
# --- Redirect stdout to file for older CODESYS compatibility ---
__mcp_out_ensure__ = os.environ.get('MCP_OUTPUT_PATH')
if __mcp_out_ensure__:
    try:
        class _AutoFlushFile:
            def __init__(self, f): self.f = f
            def write(self, s): self.f.write(s); self.f.flush()
            def flush(self): self.f.flush()
            def __getattr__(self, n): return getattr(self.f, n)
        _f_out_ensure = _AutoFlushFile(codecs.open(__mcp_out_ensure__, 'w', 'utf-8'))
        sys.stdout = _f_out_ensure
    except: pass
# --- End stdout redirect ---
```

#### 2.2 CHECK_STATUS_SCRIPT (约第 328 行)

将 import 行改为 `import sys, scriptengine as script_engine, os, codecs, traceback`，然后插入:

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
        _f_out = _AutoFlushFile2(codecs.open(__mcp_out__, 'w', 'utf-8'))
        sys.stdout = _f_out
    except: pass
# --- End stdout redirect ---
```

#### 2.3 CREATE_PROJECT_SCRIPT_TEMPLATE (约第 384 行)

将 import 行改为 `import sys, scriptengine as script_engine, os, shutil, time, codecs, traceback`，然后插入:

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
        _f_out_create = _AutoFlushFile3(codecs.open(__mcp_out_create__, 'w', 'utf-8'))
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
4. **重新加载 VS Code 窗口** (Ctrl+Shift+P → Reload Window)
5. 测试: 调用 `save_project` 验证，约 30-60 秒内返回成功

## 已验证的 MCP 工具

| 工具 | 状态 | 备注 |
|------|------|------|
| save_project | 正常 | ~30s/次 |
| open_project | 正常 | ~30s/次 |
| compile_project | 正常 | ~30s/次 |
| set_pou_code | 正常 | ~30s/次 |
| create_pou | 正常 | ~30s/次 |
| project-status (resource) | 正常 | ~15s/次 |

## 注意事项

- **每次 MCP 调用启动独立 InoProShop 进程**，约 30 秒（含 InoProShop 启动 + 项目加载 + 脚本执行）
- **IDE 和 MCP 不能同时打开同一项目** — 文件锁冲突
- **连续调用间需要间隔** — 确保前一个进程已退出
- `--noUI` 在 InoProShop SP11 中不可用（项目子系统未初始化）
- 中文 POU 名称需用 `codecs.open(path, 'w', 'utf-8')`，不可以用 `open(path, 'w')`
- Daemon 常驻进程方案不可行（CODESYS Python 主线程阻塞导致 GUI 冻结）
- 如果工具超时，确认已 Reload Window 使 MCP 加载新代码
