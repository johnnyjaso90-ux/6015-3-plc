# MCP Toolkit 适配 InoProShop SP11 (CODESYS 3.5.11) 完整修复指南

## 环境

- InoProShop: 1.9.1.6 (基于 CODESYS 3.5.11.80, Python 2.7.7)
- MCP Toolkit: @codesys/mcp-toolkit 1.1.16
- MCP 配置文件: `.claude/mcp.json`

## 架构改进

原方案每次 MCP 调用启动新 InoProShop 进程 → 打开项目 → 执行 → 关闭。
**新方案使用常驻 Daemon 进程**：首次调用启动后台 InoProShop，后续操作复用以进程，通过文件传递命令。

```
首次 open_project:  启动 daemon InoProShop（~30s）
后续 save/compile/set_code: 复用同一进程（秒级响应）
```

## 问题根因 (共 4 个)

`codesys-mcp-toolkit` 要求 CODESYS SP21+。在 SP11 上有四个兼容性问题：

1. **print() 输出不走 stdout** — 脚本引擎的 `print()` 输出到 CODESYS 内部消息窗口，不经过父进程 stdout/stderr
2. **`open(path, 'w', 0)` 无缓冲模式失效** — 文件被创建但内容无法写入（Python 2.7.7 bug）
3. **GUI 模式下进程不退出** — InoProShop GUI 保持运行，进程不会自然退出
4. **ASCII 编码导致中文字符报错** — 项目中有中文 POU 名称时，`open(path, 'w')` 默认 ASCII 编码会触发 `UnicodeEncodeError`，需用 `codecs.open(path, 'w', 'utf-8')`

## 需要修改的文件

所有文件位于: `C:\Users\<用户名>\AppData\Roaming\npm\node_modules\@codesys\mcp-toolkit\dist\`

---

### 1. codesys_interop.js — 添加 Daemon 管理系统 + 输出文件轮询

#### 1.1 更新导出声明 (约第 57 行)

在 `exports.executeCodesysScript = executeCodesysScript;` 之后添加:

```javascript
exports.startDaemon = startDaemon;
exports.sendDaemonCommand = sendDaemonCommand;
exports.shutdownDaemon = shutdownDaemon;
exports.isDaemonRunning = isDaemonRunning;
```

#### 1.2 在 SCRIPT_SUCCESS/ERROR_MARKER 常量后添加 Daemon 管理代码 (约第 69 行)

在 `const SCRIPT_ERROR_MARKER = 'SCRIPT_ERROR';` 之后, `function executeCodesysScript` 之前，插入完整 Daemon 模块:

```javascript
// --- Persistent Daemon Management ---
let daemonProcess = null;
let daemonCmdFile = null;
let daemonRespFile = null;
let daemonStartupPromise = null;
let daemonSeq = 0;

function isDaemonRunning() {
    return daemonProcess !== null && !daemonProcess.killed;
}

function getDaemonFilePaths() {
    const tmpDir = os.tmpdir();
    const base = path.join(tmpDir, 'mcp_codesys_daemon');
    return {
        cmd: base + '_cmd.txt',
        resp: base + '_resp.txt',
        output: base + '_output.txt',
    };
}

function startDaemon(daemonScriptContent, codesysExePath, codesysProfileName, projectPath) {
    return __awaiter(this, void 0, void 0, function* () {
        if (isDaemonRunning()) {
            process.stderr.write('DAEMON: Already running.\n');
            return true;
        }
        const paths = getDaemonFilePaths();
        daemonCmdFile = paths.cmd;
        daemonRespFile = paths.resp;
        const outputFile = paths.output;

        try { fs.unlinkSync(daemonCmdFile); } catch (_) {}
        try { fs.unlinkSync(daemonRespFile); } catch (_) {}
        try { fs.unlinkSync(outputFile); } catch (_) {}

        if (!codesysExePath) throw new Error('CODESYS path required');
        if (!fs.existsSync(codesysExePath)) throw new Error('CODESYS exe not found');

        const codesysDir = path.dirname(codesysExePath);
        const scriptPath = path.join(os.tmpdir(), 'mcp_codesys_daemon.py');

        const normalizedContent = daemonScriptContent.replace(/\r\n/g, '\n');
        yield (0, promises_1.writeFile)(scriptPath, normalizedContent, 'latin1');

        const spawnEnv = Object.assign({}, process.env);
        spawnEnv.PATH = `${codesysDir};${spawnEnv.PATH || ''}`;
        spawnEnv.MCP_CMD_FILE = daemonCmdFile;
        spawnEnv.MCP_RESP_FILE = daemonRespFile;
        spawnEnv.MCP_PROJECT_PATH = projectPath;

        const fullCmd = `"${codesysExePath}" --profile="${codesysProfileName}" --runscript="${scriptPath}"`;

        daemonProcess = (0, child_process_1.spawn)(fullCmd, [], {
            windowsHide: true, cwd: codesysDir, env: spawnEnv,
            shell: true, stdio: ['ignore', 'pipe', 'pipe']
        });

        daemonProcess.on('close', (code) => { daemonProcess = null; });
        daemonProcess.on('error', () => { daemonProcess = null; });

        // Wait for DAEMON_READY in output file
        daemonStartupPromise = new Promise((resolve) => {
            const start = Date.now();
            const check = setInterval(() => {
                try {
                    if (fs.existsSync(outputFile)) {
                        const content = fs.readFileSync(outputFile, 'utf-8');
                        if (content.includes('DAEMON_READY')) {
                            clearInterval(check); resolve(true);
                        }
                        if (content.includes('DAEMON_ERROR')) {
                            clearInterval(check); resolve(false);
                        }
                    }
                } catch (_) {}
                if (Date.now() - start > 300000) { clearInterval(check); resolve(false); }
            }, 2000);
        });
        return yield daemonStartupPromise;
    });
}

function sendDaemonCommand(commandText) {
    return __awaiter(this, void 0, void 0, function* () {
        if (!isDaemonRunning()) throw new Error('Daemon not running');
        daemonSeq++;
        const seq = daemonSeq;

        try { fs.unlinkSync(daemonRespFile); } catch (_) {}
        yield (0, promises_1.writeFile)(daemonCmdFile, commandText, 'utf-8');

        return yield new Promise((resolve) => {
            const start = Date.now();
            const check = setInterval(() => {
                try {
                    if (fs.existsSync(daemonRespFile)) {
                        const content = fs.readFileSync(daemonRespFile, 'utf-8');
                        if (content.includes('DAEMON_SUCCESS') || content.includes('DAEMON_ERROR')) {
                            clearInterval(check);
                            const success = content.includes('DAEMON_SUCCESS');
                            resolve({ success, output: content });
                        }
                    }
                } catch (_) {}
                if (Date.now() - start > 120000) {
                    clearInterval(check);
                    resolve({ success: false, output: 'DAEMON_ERROR: Command timeout' });
                }
            }, 500);
        });
    });
}

function shutdownDaemon() {
    return __awaiter(this, void 0, void 0, function* () {
        if (!isDaemonRunning()) return;
        try { yield sendDaemonCommand('CMD:EXIT\n'); } catch (_) {}
        try { daemonProcess.kill('SIGTERM'); } catch (_) {}
        setTimeout(() => { try { if (daemonProcess && !daemonProcess.killed) daemonProcess.kill('SIGKILL'); } catch (_) {} }, 5000);
        daemonProcess = null;
    });
}
// --- End Daemon Management ---
```

#### 1.3 添加 fileOutput 变量声明 (在 executeCodesysScript 函数内, `let exitCode = null;` 之后)

```javascript
let fileOutput = '';
```

#### 1.4 添加 outputFilePath (在 `const tempFilePath = ...` 之后)

```javascript
const outputFilePath = tempFilePath + '.out';
```

#### 1.5 设置环境变量 MCP_OUTPUT_PATH (在 `spawnEnv.PATH = ...` 之后)

```javascript
spawnEnv.MCP_OUTPUT_PATH = outputFilePath;
process.stderr.write(`INTEROP ENV: MCP_OUTPUT_PATH = ${outputFilePath}\n`);
```

#### 1.6 添加输出文件轮询 (在 resolveOnce 中增加 interval 清理, timeoutId 后添加轮询)

resolveOnce 改为:

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

timeoutId 后添加:

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
                    resolveOnce({ code: 0, stdout: stdoutData + '\n' + content, stderr: stderrData });
                } else if (content.includes('SCRIPT_ERROR')) {
                    resolveOnce({ code: 1, stdout: stdoutData + '\n' + content, stderr: stderrData });
                }
            }
        }
    } catch (_) {}
}, 2000);
```

#### 1.7 在 close, error, abort 事件中都添加 `if (outputPollInterval) clearInterval(outputPollInterval);`

#### 1.8 读取输出文件并合并 (在 `exitCode = spawnResult.code;` 之后)

```javascript
fileOutput = '';
try {
    if (fs.existsSync(outputFilePath)) {
        fileOutput = fs.readFileSync(outputFilePath, 'utf-8');
    }
} catch (_) {}
const combinedOutput = fileOutput + '\n' + output;
```

#### 1.9 更新 marker 检测使用 combinedOutput

```javascript
const hasSuccessMarker = combinedOutput.includes(SCRIPT_SUCCESS_MARKER) || stderrOutput.includes(SCRIPT_SUCCESS_MARKER);
const hasErrorMarker = combinedOutput.includes(SCRIPT_ERROR_MARKER) || stderrOutput.includes(SCRIPT_ERROR_MARKER);
```

#### 1.10 最终输出使用 combinedOutput + 清理输出文件

```javascript
if (fileOutput) { output = combinedOutput; }
// ... 在 return 前添加输出文件清理:
try {
    if (fs.existsSync(outputFilePath)) {
        yield (0, promises_1.unlink)(outputFilePath);
    }
} catch (_) {}
```

---

### 2. server.js — 添加 Daemon 脚本 + 改写工具处理器

#### 2.1 在脚本模板区添加 CODESYS_DAEMON_SCRIPT (约第 133 行, ENSURE_PROJECT_OPEN 之前)

在 `// --- Python Script Templates (Imported from v1.6.9) ---` 之后添加完整 daemon 脚本:

```javascript
        const CODESYS_DAEMON_SCRIPT = `
import sys, os, codecs, time, traceback
import scriptengine as script_engine

_output_file = os.environ.get('MCP_OUTPUT_PATH', r'C:\\Users\\<用户名>\\AppData\\Local\\Temp\\mcp_codesys_daemon_output.txt')
if _output_file:
    try:
        class _AF:
            def __init__(self, f): self.f = f
            def write(self, s): self.f.write(s); self.f.flush()
            def flush(self): self.f.flush()
            def __getattr__(self, n): return getattr(self.f, n)
        sys.stdout = _AF(codecs.open(_output_file, 'w', 'utf-8'))
    except: pass

_cmd_file = os.environ.get('MCP_CMD_FILE')
_resp_file = os.environ.get('MCP_RESP_FILE')
_project_path = os.environ.get('MCP_PROJECT_PATH', '')

_primary = None

def _ensure_open(target):
    global _primary
    import os as _os
    norm = _os.path.normcase(_os.path.abspath(target))
    for attempt in range(3):
        try: _primary = script_engine.projects.primary
        except: _primary = None
        if _primary is not None:
            try:
                if _os.path.normcase(_os.path.abspath(_primary.path)) == norm:
                    return True
            except: pass
        try:
            mode = script_engine.VersionUpdateFlags.NoUpdates | script_engine.VersionUpdateFlags.SilentMode
            _primary = script_engine.projects.open(target, update_flags=mode)
            if _primary is not None: return True
        except Exception as e:
            print("DAEMON_DEBUG: Open error: " + str(e))
        time.sleep(2.0)
    return False

def _find_pou(path_str):
    path_str = path_str.replace('\\\\', '/').strip('/')
    parts = path_str.split('/')
    obj = _primary
    for part in parts:
        found = obj.find(part, False)
        if not found: found = obj.find(part, True)
        if found: obj = found[0]
        else: return None
    return obj

# Startup
if _project_path:
    if _ensure_open(_project_path):
        print("DAEMON_READY")
    else:
        print("DAEMON_ERROR: Failed to open project")
        sys.exit(1)
else:
    print("DAEMON_READY")

# Command loop
while True:
    try:
        if os.path.exists(_cmd_file):
            with codecs.open(_cmd_file, 'r', 'utf-8') as f:
                cmd_text = f.read()
            try: os.remove(_cmd_file)
            except: pass

            lines = cmd_text.strip().split('\\n')
            cmd = lines[0].strip() if lines else ''
            resp_lines = []

            try:
                if cmd == 'CMD:OPEN':
                    target = lines[1].strip() if len(lines) > 1 else _project_path
                    ok = _ensure_open(target) if target else False
                    resp_lines.append('DAEMON_SUCCESS' if ok else 'DAEMON_ERROR: open failed')

                elif cmd == 'CMD:SAVE':
                    if _primary:
                        _primary.save()
                        resp_lines.append('DAEMON_SUCCESS: saved')
                    else:
                        resp_lines.append('DAEMON_ERROR: no project open')

                elif cmd == 'CMD:COMPILE':
                    if not _primary:
                        resp_lines.append('DAEMON_ERROR: no project open')
                    else:
                        app = None
                        try: app = _primary.active_application
                        except: pass
                        if app is None:
                            for c in _primary.get_children(True):
                                if hasattr(c, 'is_application') and c.is_application and hasattr(c, 'build'):
                                    app = c; break
                        if app: app.build(); resp_lines.append('DAEMON_SUCCESS: build started')
                        else: resp_lines.append('DAEMON_ERROR: no application found')

                elif cmd == 'CMD:SET_CODE':
                    pou_path = lines[1].strip() if len(lines) > 1 else ''
                    pou = _find_pou(pou_path)
                    if pou is None:
                        resp_lines.append('DAEMON_ERROR: POU not found: ' + pou_path)
                    else:
                        decl_code = ''; impl_code = ''; section = None
                        for i in range(2, len(lines)):
                            ln = lines[i].strip()
                            if ln == 'SECTION:DECL': section = 'decl'; continue
                            elif ln == 'SECTION:IMPL': section = 'impl'; continue
                            elif ln == 'SECTION:END': section = None; continue
                            if section == 'decl': decl_code += lines[i] + '\\n'
                            elif section == 'impl': impl_code += lines[i] + '\\n'
                        try:
                            if decl_code: pou.textual_declaration.replace(decl_code)
                            if impl_code: pou.textual_implementation.replace(impl_code)
                            _primary.save()
                            resp_lines.append('DAEMON_SUCCESS: code set for ' + pou_path)
                        except Exception as e2:
                            resp_lines.append('DAEMON_ERROR: ' + str(e2))

                elif cmd == 'CMD:GET_STATUS':
                    s_ok = True; s_open = _primary is not None
                    s_name = 'No project'; s_path = 'N/A'
                    if _primary:
                        try: s_name = _primary.get_name() or 'Unnamed'
                        except: pass
                        try: s_path = _primary.path
                        except: pass
                    resp_lines.append('Scripting OK: ' + str(s_ok))
                    resp_lines.append('Project Open: ' + str(s_open))
                    resp_lines.append('Project Name: ' + s_name)
                    resp_lines.append('Project Path: ' + s_path)
                    resp_lines.append('DAEMON_SUCCESS')

                elif cmd == 'CMD:EXIT':
                    with codecs.open(_resp_file, 'w', 'utf-8') as rf:
                        rf.write('DAEMON_SUCCESS: exiting')
                    sys.exit(0)

                else:
                    resp_lines.append('DAEMON_ERROR: unknown command ' + cmd)

            except Exception as ex:
                resp_lines.append('DAEMON_ERROR: ' + str(ex))
                resp_lines.append(traceback.format_exc())

            with codecs.open(_resp_file, 'w', 'utf-8') as rf:
                rf.write('\\n'.join(resp_lines))

        time.sleep(0.5)
    except Exception as loop_err:
        print("DAEMON_LOOP_ERROR: " + str(loop_err))
        time.sleep(1.0)
`;
```

#### 2.2 添加 daemonSend 辅助函数 (在 fileExists 函数之后, MCP Server 初始化之前)

```javascript
        let _daemonProjectPath = null;
        function daemonSend(projectPath, command) {
            return __awaiter(this, void 0, void 0, function* () {
                if (!(0, codesys_interop_1.isDaemonRunning)()) {
                    const ok = yield (0, codesys_interop_1.startDaemon)(CODESYS_DAEMON_SCRIPT, codesysExePath, codesysProfileName, projectPath);
                    if (!ok) throw new Error('Daemon failed to start');
                    _daemonProjectPath = projectPath;
                } else if (projectPath && projectPath !== _daemonProjectPath) {
                    yield (0, codesys_interop_1.sendDaemonCommand)('CMD:OPEN\n' + projectPath);
                    _daemonProjectPath = projectPath;
                }
                return yield (0, codesys_interop_1.sendDaemonCommand)(command);
            });
        }
```

#### 2.3 在三个 Python 脚本模板中添加 stdout 重定向 (保留旧的 fallback 机制)

三个脚本模板: `ENSURE_PROJECT_OPEN_PYTHON_SNIPPET`, `CHECK_STATUS_SCRIPT`, `CREATE_PROJECT_SCRIPT_TEMPLATE`

每个在 import 后添加:

```python
import codecs  # (加入已有 import 行)
# --- Redirect stdout to file for older CODESYS compatibility ---
__mcp_out__ = os.environ.get('MCP_OUTPUT_PATH')
if __mcp_out__:
    try:
        class _AutoFlushFile:
            def __init__(self, f): self.f = f
            def write(self, s): self.f.write(s); self.f.flush()
            def flush(self): self.f.flush()
            def __getattr__(self, n): return getattr(self.f, n)
        _f_out = _AutoFlushFile(codecs.open(__mcp_out__, 'w', 'utf-8'))
        sys.stdout = _f_out
    except: pass
# --- End stdout redirect ---
```

#### 2.4 改写工具处理器使用 daemonSend

四个核心工具的 try 块改为调用 daemonSend:

**open_project** (约第 1460 行):
```javascript
const result = yield daemonSend(absPath, 'CMD:OPEN\n' + absPath);
const success = result.success;
```

**save_project** (约第 1540 行):
```javascript
const result = yield daemonSend(absPath, 'CMD:SAVE');
const success = result.success;
```

**compile_project** (约第 1700 行):
```javascript
const result = yield daemonSend(absPath, 'CMD:COMPILE');
const success = result.success;
```

**set_pou_code** (约第 1595 行):
```javascript
let cmd = 'CMD:SET_CODE\n' + sanPouPath + '\n';
if (declarationCode !== null && declarationCode !== void 0) {
    cmd += 'SECTION:DECL\n' + declarationCode + '\n';
}
if (implementationCode !== null && implementationCode !== void 0) {
    cmd += 'SECTION:IMPL\n' + implementationCode + '\n';
}
cmd += 'SECTION:END';
const result = yield daemonSend(absPath, cmd);
const success = result.success;
```

**project-status resource** (约第 1318 行):
```javascript
const result = yield daemonSend(null, 'CMD:GET_STATUS');
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
3. 替换 daemon 脚本中的 `<用户名>` 为实际 Windows 用户名
4. 配置 `.claude/mcp.json`（替换用户名和项目路径）
5. **重新加载 VS Code 窗口** (Ctrl+Shift+P → Reload Window)
6. 测试: `open_project` 启动 daemon (~30s)，后续操作秒级响应

## 已验证的 MCP 工具

| 工具 | 模式 | 状态 |
|------|------|------|
| open_project | Daemon | 正常（首次启动 daemon ~30s）|
| save_project | Daemon | 正常（复用进程，秒级）|
| compile_project | Daemon | 正常（复用进程，秒级）|
| set_pou_code | Daemon | 正常（复用进程，秒级）|
| project-status (resource) | Daemon | 正常（复用进程，秒级）|
| create_pou | Fallback (旧) | 正常 |
| create_project | Fallback (旧) | 正常 |

## 注意事项

- **Daemon 进程在首次 `open_project` 时启动**，后续操作自动复用
- **不再需要关 IDE** — daemon 是独立进程，不会和 IDE 冲突（但不要两个同时写入项目）
- `--noUI` 不可用（InoProShop SP11 中会导致项目子系统未初始化）
- 中文 POU 名称需要 `codecs.open(path, 'w', 'utf-8')` 编码
- 若 daemon 进程意外退出，下次调用会自动重启
