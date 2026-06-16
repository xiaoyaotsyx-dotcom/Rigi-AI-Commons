---
name: cdp-quick-connect
description: CDP 快速连接工作流 — 每次换电脑/Chrome重启后的标准操作。三步确认 Chrome调试模式 → 中继 → 验证。加载本Skill后立即诊断并连接用户桌面Chrome。
domain: browser-automation
triggers:
  - 用户说"打开浏览器"、"看看我的页面"、"浏览器上开了XX"
  - 需要操作用户桌面Chrome的任何场景
  - Chrome重启/换电脑后的首次连接
---

# CDP 快速连接工作流

## 核心原则

1. **加载即诊断** — 不废话，直接测连通性
2. **三步修好** — Chrome调试模式 → 中继 → 验证
3. **不行就告诉用户** — 别反复试，直接给具体操作步骤

## 三步标准流程

### Step 0: 诊断

**先检查 Chrome 进程是否存在**（中继可能活着但 Chrome 已死，curl 会返回 `ConnectionResetError`）：

```bash
PWSH="/mnt/c/Windows/System32/WindowsPowerShell/v1.0/powershell.exe"
CHROME_COUNT=$($PWSH -Command "(Get-Process -Name chrome -ErrorAction SilentlyContinue).Count" 2>/dev/null)
echo "Chrome processes: $CHROME_COUNT"
```

**如果 Chrome 进程数 = 0** → Chrome 已死。**不要只重启中继**，必须走完整三步：杀中继→启 Chrome→启中继。

**如果 Chrome 进程数 > 0**，再测连通性：

```bash
GW=$(ip route show default | awk '{print $3}' | head -1)
curl -s --max-time 5 "http://$GW:19223/json/version" 2>/dev/null | python3 -c "import sys,json; d=json.load(sys.stdin); print(f'OK|{d.get(\"Browser\",\"?\")}')" 2>/dev/null || echo "FAIL"
```

- ✅ `OK|Chrome/...` → 中继已运行，直接跳到 Step 3
- ❌ `FAIL` → 继续 Step 1

### Step 1: 启动 Chrome 调试模式

**Chrome 必须先杀干净再重启**，否则 --remote-debugging-port 不生效。**启动时用默认 profile 路径保留登录态**，不用 `D:\chrome-debug-profile` 独立目录。正确路径：`--user-data-dir=C:\Users\<username>\AppData\Local\Google\Chrome\User Data`。

```bash
PWSH="/mnt/c/Windows/System32/WindowsPowerShell/v1.0/powershell.exe"

# 先杀所有 Chrome 进程
$PWSH -Command "Get-Process -Name chrome -ErrorAction SilentlyContinue | Stop-Process -Force"
sleep 3

# 写启动脚本（.ps1 文件方式最可靠，避免空格路径引号问题）
cat > /mnt/d/launch_chrome.ps1 << 'PSEOF'
$chrome = "C:\Program Files\Google\Chrome\Application\chrome.exe"
$args = @(
    '--remote-debugging-port=9222',
    '--remote-allow-origins=*',
    '--user-data-dir=C:\Users\60914\AppData\Local\Google\Chrome\User Data',
    '--restore-last-session'
)
Start-Process -FilePath $chrome -ArgumentList $args
PSEOF

$PWSH -ExecutionPolicy Bypass -File D:\launch_chrome.ps1
sleep 6

# 验证 Chrome 9222 端口
$PWSH -Command "netstat -ano | findstr ':9222 '" 
# 应显示 LISTENING
```

⚠️ **关键参数：**
- `--user-data-dir` 必须指向用户默认 profile（保留登录态），不要用独立目录
- `--remote-allow-origins=*` 缺失 → WSL 连接返回 403
- `--restore-last-session` 可能不恢复所有标签页（部分网站会重定向到登录页）  
⚠️ 从 WSL 启动 Chrome 时，`-Command` 内联引号极易被 shell 吃掉导致失败。**必须写 `.ps1` 脚本文件到 D:\\，再用 `-File` 执行**。  
⚠️ 杀 Chrome 进程前确认用户是否在浏览中：「Chrome 需要重启调试模式，关闭浏览器我帮你重新打开？」，等确认。  
⚠️ `--restore-last-session` 不一定能恢复所有标签页（取决于 Chrome 用户配置），做好丢失准备。

### Step 2: 启动 CDP 中继

```bash
PWSH="/mnt/c/Windows/System32/WindowsPowerShell/v1.0/powershell.exe"

# 先杀旧中继
$PWSH -Command "Get-Process -Name python | Where-Object { (Get-Process -Id \$_.Id).MainWindowTitle -eq '' } | Stop-Process -Force -ErrorAction SilentlyContinue" 2>/dev/null
sleep 2

# 启动中继
$PWSH -Command "Start-Process -WindowStyle Hidden python.exe -ArgumentList 'D:\relay2.py'"
sleep 3
```

如果 `D:\relay2.py` 不存在，先写一份：

```bash
cat > /mnt/d/relay2.py << 'PYEOF'
import socket, threading, sys

def relay(src, dst):
    while True:
        try:
            data = src.recv(8192)
            if not data: break
            dst.sendall(data)
        except: break
    try: src.close()
    except: pass
    try: dst.close()
    except: pass

def handle(client):
    try:
        remote = socket.create_connection(('127.0.0.1', 9222), timeout=5)
        t1 = threading.Thread(target=relay, args=(client, remote), daemon=True)
        t2 = threading.Thread(target=relay, args=(remote, client), daemon=True)
        t1.start(); t2.start()
        t1.join(); t2.join()
    except: client.close()

server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
server.bind(('0.0.0.0', 19223))
server.listen(5)
sys.stdout.write("CDP RELAY READY\n")
sys.stdout.flush()
while True:
    client, _ = server.accept()
    threading.Thread(target=handle, args=(client,), daemon=True).start()
PYEOF
```

### Step 3: 验证 + 连接

```bash
GW=$(ip route show default | awk '{print $3}' | head -1)

# 验证
curl -s --max-time 5 "http://$GW:19223/json/version" | python3 -c "
import sys,json
d=json.load(sys.stdin)
print(f'✅ {d[\"Browser\"]} | WS: {d[\"webSocketDebuggerUrl\"][:50]}...')
" 2>/dev/null && echo "READY" || echo "FAIL"
```

验证成功后，用 Python Playwright 连接：

```python
from playwright.sync_api import sync_playwright
import urllib.request, json

GW = "172.29.48.1"
req = urllib.request.urlopen(f"http://{GW}:19223/json/version", timeout=5)
WS = json.loads(req.read())["webSocketDebuggerUrl"]

with sync_playwright() as p:
    browser = p.chromium.connect_over_cdp(WS)
    for ctx in browser.contexts:
        for i, pg in enumerate(ctx.pages):
            print(f"  [{i}] {pg.url[:100]}")
```

## 无中继直连模式（旧电脑/特殊配置）

如果用户说"之前不用中继" → Chrome 可能直接在 `0.0.0.0:9222` 监听。测试：

```bash
GW=$(ip route show default | awk '{print $3}' | head -1)
curl -s --max-time 3 "http://$GW:9222/json/version" 2>/dev/null && echo "DIRECT_OK" || echo "NEED_RELAY"
```

- ✅ `DIRECT_OK` → 连接时用 `http://$GW:9222` 代替 `http://$GW:19223`
- ❌ `NEED_RELAY` → 走标准中继流程

## Chrome 重启关键经验

### 换电脑/首次连接标准流程

```bash
# Step 0: 先全杀 Chrome（必须，否则 --remote-debugging-port 不生效）
PWSH="/mnt/c/Windows/System32/WindowsPowerShell/v1.0/powershell.exe"
$PWSH -Command "Get-Process -Name chrome -ErrorAction SilentlyContinue | Stop-Process -Force"
sleep 3

# Step 1: 写 .ps1 脚本（避免 WSL bash 吞引号）
cat > /mnt/d/launch_chrome.ps1 << 'PSEOF'
$chrome = "C:\Program Files\Google\Chrome\Application\chrome.exe"
$args = @(
    '--remote-debugging-port=9222',
    '--remote-allow-origins=*',
    '--user-data-dir=C:\Users\60914\AppData\Local\Google\Chrome\User Data',
    '--restore-last-session'
)
Start-Process -FilePath $chrome -ArgumentList $args
PSEOF

$PWSH -ExecutionPolicy Bypass -File D:\launch_chrome.ps1
sleep 6
```

### ⚠️ 坑：PowerShell 多行语法在 WSL bash 中不可用

`@"...  "@` 和 `<com>` 块在通过 bash 传参时会被破坏。**唯一可靠方案：写 `.ps1` 文件再 `-File` 执行。**

### ⚠️ 坑：必须用用户默认 profile

`--user-data-dir` 必须指向默认 Chrome profile（如 `C:\Users\<name>\AppData\Local\Google\Chrome\User Data`），不能用独立目录。否则所有登录态丢失，用户需要重新登录所有网站。

### ⚠️ 坑：Chrome 已在运行时 --remote-debugging-port 不生效

如果 Chrome 进程已存在，即使加了参数启动新实例，也不会开启 9222 端口。**必须先 `Stop-Process -Force` 杀干净。**

## 故障速查

| 症状 | 诊断 | 修复 |
|------|------|------|
| Chrome 启动后 9222 无监听 | --remote-debugging-port 被忽略 | 先 `taskkill /F` 杀干净 Chrome，再重启 |
| 中继端口被占 | `netstat -ano \| findstr 19223` | `taskkill /F /PID <pid>` |
| Playwright 超时 | relay2.py 不是后台运行 | 确保 `Start-Process -WindowStyle Hidden` |
| curl 返回 403 | Chrome 缺少 `--remote-allow-origins=*` | 重启 Chrome 加上该参数 |
| curl 返回空 | WSL 安全策略拦截 | `powershell.exe -Command "curl http://127.0.0.1:9222/json/version"` 验证 |
| 标签页恢复失败 | --restore-last-session 不可靠 | 手动 `new_page()` 打开目标网站，用户重新登录 |
| contenteditable 点不动 | 网站用 fake-placeholder 覆盖 | `page.evaluate("document.querySelector('.fake-placeholder').remove()")` |
| keyboard.type 打字没反应 | 编辑器未聚焦 | 先点击或 `evaluate` 聚焦编辑器元素后再 type |
| curl 返回空 | WSL 安全策略拦截 | `powershell.exe -Command "curl http://127.0.0.1:9222/json/version"` 验证 |
| ConnectionResetError | **Chrome 进程已死**（长时间闲置后常见） | 重新走 Step 0 诊断，如果 FAIL 走完整三步流程 |
| 长时间操作后CDP断开 | Chrome标签页过多或内存耗尽 | **每次大型操作前先 curl 验证连通性**，不假设连接一直活着 |
| 🔴 **CDP 连接永远超时（20-30分钟）** | `config.yaml` 里 `browser.cdp_url` 写死了浏览器实例 ID。Chrome 每次重启生成新 ID，旧 URL 变废链，Agent 无限重试 | `sed -i '/cdp_url:/d' ~/.hermes/config.yaml` 删掉这行。Agent 应从 `/json/version` 动态获取 WebSocket URL |
| 🔴 **Agent 飞书喊不停** | `approvals.mode: false` + `busy_input_mode: interrupt`（非 cancel） | 改为 `mode: smart` + `busy_input_mode: cancel`。详见 `references/hermes-config-pitfalls.md` |

### 长会话防断策略

连续多步操作时，每 5-10 分钟或每次重要操作前，先跑一次：

```bash
GW=$(ip route show default | awk '{print $3}' | head -1)
curl -s --max-time 3 "http://$GW:19223/json/version" | python3 -c "import sys,json;d=json.load(sys.stdin);print('OK')" 2>/dev/null || echo "DEAD"
```

如果返回 `DEAD` → 重新走三步。不跳过诊断。
| **Chrome 端口显示 LISTENING 但 curl 不通** | Chrome 只绑 127.0.0.1 | 正常！中继在 19223 转发，走中继而非直连 |
| **`--restore-last-session` 没恢复标签** | Chrome 被杀后 session 不可靠 | 别指望恢复，直接 `curl -X PUT json/new` 开新标签 |
| **PowerShell 多行 `@""@` 语法报错** | WSL bash 吃掉引号 | 写 `.ps1` 文件再用 `-File` 执行 |
| 🔴 会话未恢复 | Chrome 被 kill 后重启，`--restore-last-session` 不生效 | 使用默认 profile 路径（见下方「获取用户路径」）；接受标签丢失，直接用 CDP `curl -X PUT json/new` 开新标签 |
| 🔴 重复中继进程 | 多个 relay 同时监听 19223，WebSocket 超时 | `netstat -ano | findstr ':19223 '` 查所有 PID，`taskkill /F /PID ...` 全杀后再干净启动一个 |
| 🔴 PS 脚本中文乱码 | 文件编码不是 UTF-8 | 通过 `cat > /mnt/d/file.ps1` 写入而非 PowerShell 创建 |
| 🔴 换电脑后 user-data-dir 路径不对 | 不同 Windows 用户名导致路径不同 | 动态获取（见「获取用户 Chrome Profile 路径」） |
| 🔴 ConnectionResetError 连续出现 | Chrome 进程已死或 CDP 端口关闭（长闲置后常见） | 回到 Step 0 完整诊断，不要绕步骤 |

### 获取用户 Chrome Profile 路径（换电脑时必须）

不同电脑的用户名不同，路径不能硬编码。先检测：

```bash
# 方法：从已运行的 Chrome 进程找到 user-data-dir
PWSH="/mnt/c/Windows/System32/WindowsPowerShell/v1.0/powershell.exe"
$PWSH -Command "Get-Process -Name chrome -ErrorAction SilentlyContinue | Select-Object -First 1 | ForEach-Object { Get-WmiObject Win32_Process -Filter \\"ProcessId=\$(\$_.Id)\\" } | Select-Object -ExpandProperty CommandLine" 2>/dev/null | grep -oP 'user-data-dir=\K[^ ]+'

# 如果上面失败，列出可能的路径
$PWSH -Command "Get-ChildItem 'C:\\Users' -Directory | ForEach-Object { \\$p = \\"\\\$(\$_.FullName)\\AppData\\Local\\Google\\Chrome\\User Data\\"; if (Test-Path \\$p) { Write-Output \\$p } }"
```

## ⚠️ 坑：Windows/WSL 双实例冲突

如果同一台电脑上同时装过 Windows 原生 Hermes 和 WSL Hermes：

- **Windows 版 Hermes 卸载后，gateway 进程可能残留在后台**，占用端口
- **自检脚本必须在 WSL 终端中运行**，不要在 Windows PowerShell/CMD 中执行——Windows 侧看不到 WSL 里的 hermes CLI、Python 包和 skill 目录
- 自检报告出现 `WSL: 否` + `hermes 命令未找到` + `Gateway: ✅ 运行中` → 说明脚本跑在 Windows 侧，抓到了 Windows 残骸的 gateway

**清理 Windows 残留：**
```powershell
# 在 Windows PowerShell (管理员) 中：
Get-Process -Name "hermes*","node" -ErrorAction SilentlyContinue | Stop-Process -Force
# 然后确认 WSL 内的 Hermes 端口正常
```

**验证没有冲突：**
```bash
# 在 WSL 终端中：
ss -tlnp | grep -E '9222|19223'    # CDP 端口
ps aux | grep hermes | grep -v grep  # Hermes 进程（应该只有一个）
```

## 其他平台兼容

本 Skill 为通用 CDP 连接。具体平台操作（X/微博/头条/知乎/小红书/Facebook）参考 `hermes-browser-automation` skill 的编辑器兼容性矩阵和反检测策略。雪球、知识星球等非标准平台，连接后用 `page.evaluate()` 和 `keyboard.type()` 操作。

## 延伸: GitHub 上架工作流

发布 AI Skill 到 GitHub 的完整流程（中英双语 README、AGPLv3 双许可、隐私扫描、SkillsMD 提交），见 `references/github-publishing-workflow.md`。
