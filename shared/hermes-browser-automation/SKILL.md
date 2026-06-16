---
name: hermes-browser-automation
description: Hermes 通用浏览器操控 Skill — 通过 Chrome CDP + Playwright 让 AI Agent 像真人一样操控浏览器。覆盖导航、点击、填表、文件上传、数据抓取、登录态保持、验证码绕过等全部浏览器交互能力。任何需要操控网页的 Agent 都可以加载此 Skill 获得浏览器操作能力。
domain: browser-automation
triggers:
  - 用户说"操控浏览器"、"操作网页"、"浏览器自动化"、"打开网页"、"帮我登录"
  - 用户需要让 Agent 替自己在网页上执行操作
  - 用户需要批量抓取网页数据
  - 用户需要绕过反自动化检测
---

# Hermes 通用浏览器操控 Skill

## 这是什么

让你的 AI Agent 获得**真人的浏览器操作能力**——导航、点击、打字、上传文件、填表、截图、抓数据，全部通过操控用户真实 Chrome 浏览器完成。

**不是 Selenium。不是 Puppeteer。不是无头浏览器。** 是直接操控用户桌面上正在运行的真实 Chrome——登录态、Cookie、浏览器指纹全是真实的。网站看到的就是一个真人在操作。

### 能做什么

| 能力 | 说明 |
|------|------|
| 🌐 页面导航 | 打开 URL、前进后退、刷新、等待加载 |
| 🖱️ 鼠标点击 | 按文本/选择器/坐标点击任意元素 |
| ⌨️ 键盘输入 | 逐字打字、粘贴、快捷键组合 |
| 📝 表单填写 | input/textarea/contenteditable/Draft.js/ProseMirror 全兼容 |
| 📎 文件上传 | 原生 file input 注入，突破网站对自动化工具的检测 |
| 📸 截图 | 全页/元素/区域截图，用于视觉验证或 CAPTCHA |
| 🔍 数据抓取 | DOM 查询、innerText 提取、表格解析、列表遍历 |
| 🔐 登录态保持 | 用户手动登录一次，Agent 永久复用 |
| 🤖 反检测 | 用真实浏览器避开绝大多数反自动化机制 |
| 🧩 编辑器兼容 | Draft.js (X) / ProseMirror (头条) / Vue (微博) / React (知乎) 全适配 |

---

## 一、架构原理

```
┌────────────────────────────────────────────────┐
│                   WSL (Linux)                   │
│  ┌──────────┐     ┌──────────────────────┐     │
│  │ AI Agent │────▶│  Playwright via CDP  │     │
│  │ (Hermes) │     │  (Python)            │     │
│  └──────────┘     └──────────┬───────────┘     │
│                              │                  │
└──────────────────────────────┼──────────────────┘
                               │ HTTP WebSocket
                               │ 172.29.48.1:19223
┌──────────────────────────────┼──────────────────┐
│                  Windows     │                  │
│  ┌───────────────────────────▼──────────────┐   │
│  │         CDP 中继 (relay2.py)             │   │
│  │         0.0.0.0:19223 → 127.0.0.1:9222  │   │
│  └───────────────────────────┬──────────────┘   │
│                              │                  │
│  ┌───────────────────────────▼──────────────┐   │
│  │     Google Chrome (--remote-debugging)    │   │
│  │     用户已登录各网站的标签页               │   │
│  └──────────────────────────────────────────┘   │
└──────────────────────────────────────────────────┘
```

**核心优势：网站看到的是用户真实 Chrome，不是自动化工具。** 无头浏览器（headless）有数百个可检测指纹——Chrome CDP 模式下这些指纹全部是真实的。

---

## 二、前置条件

### 2.1 Windows 端启动 Chrome 调试模式

用户在 Windows 上**手动**执行：

```cmd
"C:\Program Files\Google\Chrome\Application\chrome.exe" --remote-debugging-port=9222 --remote-allow-origins=* --user-data-dir="D:\chrome-debug-profile"
```

| 参数 | 说明 |
|------|------|
| `--remote-debugging-port=9222` | 开启 CDP 调试端口 |
| `--remote-allow-origins=*` | ⚠️ 必须！否则 WSL 连接时返回 403 |
| `--user-data-dir` | 使用独立 profile，保留登录态 |

### 2.2 启动 CDP 中继

```cmd
python.exe D:\relay2.py
```

WSL 无法直连 Windows `127.0.0.1`。中继在 `0.0.0.0:19223` 监听，转发到 `127.0.0.1:9222`。

中继脚本（`D:\relay2.py`）：

```python
import socket, threading, sys

def relay(src, dst):
    while True:
        try:
            data = src.recv(8192)
            if not data: break
            dst.sendall(data)
        except: break
    src.close()
    dst.close()

def handle(client):
    try:
        remote = socket.create_connection(('127.0.0.1', 9222), timeout=5)
        t1 = threading.Thread(target=relay, args=(client, remote), daemon=True)
        t2 = threading.Thread(target=relay, args=(remote, client), daemon=True)
        t1.start(); t2.start()
        t1.join(); t2.join()
    except: client.close()

def main():
    server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    server.bind(('0.0.0.0', 19223))
    server.listen(5)
    print(f"CDP relay: 0.0.0.0:19223 → 127.0.0.1:9222")
    while True:
        client, addr = server.accept()
        threading.Thread(target=handle, args=(client,), daemon=True).start()

if __name__ == '__main__':
    main()
```

### 2.3 WSL 端安装 Playwright

```bash
pip3 install playwright
```

### 2.4 连接验证

```bash
GW=$(ip route show default | awk '{print $3}')
curl -s --max-time 5 "http://$GW:19223/json/version" | python3 -c "
import sys, json
d = json.load(sys.stdin)
print(f'Browser: {d.get(\"Browser\",\"?\")}')
print(f'User-Agent: {d.get(\"User-Agent\",\"?\")[:60]}')
"
```

---

## 三、连接浏览器

### 3.1 基础连接

```python
from playwright.sync_api import sync_playwright
import urllib.request, json

GW = "172.29.48.1"

# 获取浏览器级 WebSocket
req = urllib.request.urlopen(f"http://{GW}:19223/json/version", timeout=5)
BROWSER_WS = json.loads(req.read())["webSocketDebuggerUrl"]

# 连接
with sync_playwright() as p:
    browser = p.chromium.connect_over_cdp(BROWSER_WS)
    # browser 对象现在可以操作所有已打开的标签页
```

### 3.2 查找标签页

```python
# 列出所有标签页
for ctx in browser.contexts:
    for i, pg in enumerate(ctx.pages):
        print(f"  [{i}] {pg.url[:80]}")

# 按 URL 查找特定标签页
def find_page(browser, url_fragment, exclude=None):
    """找到 URL 包含 url_fragment 的标签页，排除包含 exclude 的"""
    for ctx in browser.contexts:
        for pg in ctx.pages:
            url = pg.url
            if url_fragment in url:
                if exclude and exclude in url:
                    continue
                return pg
    return None

# 示例：找 X 已登录页面
page = find_page(browser, "x.com", exclude="login")
```

### 3.3 创建新标签页

```python
# ⚠️ 注意：新标签页 goto 某些网站会触发重新登录
# 优先使用已有标签页，只在必要时新建

ctx = browser.contexts[0]
new_page = ctx.new_page()
new_page.goto("https://example.com", wait_until="domcontentloaded")
```

### 3.4 检查登录状态

```python
def check_login(page, logged_in_selector):
    """通用登录状态检查"""
    try:
        el = page.locator(logged_in_selector).first
        return el.is_visible(timeout=3000)
    except:
        return False

# 各平台登录标志
LOGIN_INDICATORS = {
    "x.com": '[data-testid="SideNav_NewTweet_Button"]',
    "weibo.com": 'textarea',
    "zhihu.com": '[aria-label="写回答"]',
    "facebook.com": '[aria-label="分享新鲜事"]',
}
```

---

## 四、核心操作

### 4.1 页面导航

```python
# 基础导航
page.goto("https://example.com", wait_until="domcontentloaded", timeout=30000)

# 等待选项
# "load" — 等所有资源加载完（慢）
# "domcontentloaded" — DOM 就绪即可（推荐）
# "networkidle" — 网络空闲（最慢，动态页面用）

# 后退
page.go_back()

# 刷新
page.reload()

# 等待指定时间
import time
time.sleep(3)  # 简单粗暴但有时必要

# 等待元素出现
page.locator('.result-list').first.wait_for(state='visible', timeout=10000)
```

### 4.2 点击元素

```python
# 按文本点击
page.locator('button').filter(has_text='提交').click()
page.locator('text=登录').click()

# 按选择器点击
page.locator('#submit-btn').click()
page.locator('[data-testid="confirm"]').click()

# 按角色点击
page.get_by_role('button', name='发送').click()

# 强制点击（绕过遮罩层）
page.locator('button').filter(has_text='发布').click(force=True, timeout=5000)

# 坐标点击
page.mouse.click(500, 300)

# JS 注入点击（React 页面有时需要）
page.evaluate("document.querySelector('button.submit').click()")
```

### 4.3 键盘输入

```python
# 基础输入到 input/textarea
page.locator('input[name="username"]').fill("my_username")

# 逐字打字（contenteditable / 编辑器）
page.keyboard.type("要输入的内容", delay=25)  # delay 毫秒

# 快捷键
page.keyboard.press('Control+Enter')  # 提交
page.keyboard.press('Control+a')      # 全选
page.keyboard.press('Escape')         # 关闭弹窗

# 组合键
page.keyboard.press('Control+Shift+I')
```

### 4.4 表单填写

```python
# 普通 input
page.locator('input[type="email"]').fill("user@example.com")
page.locator('input[placeholder="手机号"]').fill("13800138000")

# textarea
page.locator('textarea').fill("长文本内容...")

# contenteditable（富文本编辑器）
# 方式 A：fill() — 适用于 Draft.js、React 编辑器
page.locator('[contenteditable]').first.fill("内容")

# 方式 B：keyboard.type() — 适用于 ProseMirror、Facebook
page.locator('.ProseMirror').first.click()
page.keyboard.type("内容", delay=1)

# 方式 C：evaluate 原生 setter — 适用于 Vue/React 表单
page.evaluate("""
    (text) => {
        const ta = document.querySelector('textarea');
        const setter = Object.getOwnPropertyDescriptor(
            HTMLTextAreaElement.prototype, 'value'
        ).set;
        setter.call(ta, text);
        ta.dispatchEvent(new Event('input', {bubbles: true}));
    }
""", "内容")

# 选择下拉框
page.locator('select').select_option('value')

# 勾选复选框
page.locator('input[type="checkbox"]').check()
```

### 4.5 文件上传

```python
# ⚠️ WSL 环境：路径必须用 Linux 格式 /mnt/d/... 而非 D:\
image_path = "/mnt/d/Hermes/images/photo.jpg"

# 单文件
page.locator('input[type="file"]').set_input_files(image_path)

# 多文件
page.locator('input[type="file"]').set_input_files([
    "/mnt/d/img1.jpg", "/mnt/d/img2.jpg"
])

# 隐藏 file input（如 X/Twitter 的 fileInput）
page.locator('[data-testid="fileInput"]').first.set_input_files(image_path)

# 上传后等待处理
time.sleep(8)  # 等上传进度条完成
```

### 4.6 滚动页面

```python
# 滚动到指定位置
page.evaluate("window.scrollBy(0, 500)")

# 滚动到底部
page.evaluate("window.scrollTo(0, document.body.scrollHeight)")

# 滚动到元素可见
page.locator('#target').scroll_into_view_if_needed()

# 懒加载页面连续滚动
for i in range(10):
    page.evaluate("window.scrollBy(0, 800)")
    time.sleep(1.5)
```

### 4.7 截图

> **browser_vision 滚动截图失效？** 当内置截图工具忽略滚动位置时，使用拆分页面方案：将 HTML 按 section 拆分为独立文件，分别截图。详见 `references/html-section-screenshot-workaround.md`。

```python
# 全页截图
page.screenshot(path="/tmp/full_page.png")

# 元素截图
page.locator('.result-card').screenshot(path="/tmp/card.png")

# 全页截图（包含滚动区域）
page.screenshot(path="/tmp/full.png", full_page=True)

# 用于视觉验证
from hermes_tools import vision_analyze
vision_analyze("/tmp/captcha.png", question="验证码中的文字是什么？")
```

### 4.8 弹窗处理

```python
# 监听 dialog（alert/confirm/prompt）
page.on("dialog", lambda dialog: dialog.accept())  # 自动确认
page.on("dialog", lambda dialog: dialog.dismiss()) # 自动取消

# 关闭残留弹窗
page.keyboard.press('Escape')

# 等待弹窗消失
page.locator('.modal-overlay').wait_for(state='hidden', timeout=10000)
```

---

## 五、数据抓取

### 5.1 DOM 文本提取

```python
# 提取元素文本
title = page.locator('h1').first.inner_text()

# 提取所有匹配元素的文本
items = page.locator('.product-name').all()
names = [item.inner_text() for item in items]

# 提取表格数据
rows = page.locator('table tr').all()
data = []
for row in rows:
    cells = row.locator('td, th').all()
    data.append([c.inner_text() for c in cells])
```

### 5.2 JS evaluate 批量提取

```python
# 复杂提取用 evaluate（一次往返，比逐元素查询快 100 倍）
results = page.evaluate("""
    () => {
        const cards = document.querySelectorAll('.card');
        return Array.from(cards).slice(0, 50).map(card => ({
            title: card.querySelector('h2')?.textContent?.trim() || '',
            price: card.querySelector('.price')?.textContent?.trim() || '',
            link: card.querySelector('a')?.href || '',
            image: card.querySelector('img')?.src || '',
        }));
    }
""")

print(f"提取了 {len(results)} 条数据")
```

### 5.3 链接和属性提取

```python
# 所有链接
links = page.evaluate("""
    () => Array.from(document.querySelectorAll('a[href]'))
        .map(a => ({text: a.textContent.trim().slice(0,50), href: a.href}))
""")

# 图片 URL
images = page.evaluate("""
    () => Array.from(document.querySelectorAll('img[src]'))
        .map(img => img.src)
        .filter(src => src.startsWith('http'))
""")

# 特定属性
data_ids = page.evaluate("""
    () => Array.from(document.querySelectorAll('[data-id]'))
        .map(el => el.getAttribute('data-id'))
""")
```

### 5.4 分页抓取

```python
all_data = []

for page_num in range(1, 11):
    url = f"https://example.com/list?page={page_num}"
    page.goto(url, wait_until="domcontentloaded")
    time.sleep(2)
    
    data = page.evaluate("""
        () => Array.from(document.querySelectorAll('.item'))
            .map(item => item.innerText.slice(0, 200))
    """)
    
    all_data.extend(data)
    print(f"第 {page_num} 页: {len(data)} 条")
    
    time.sleep(3)  # 反爬间隔

print(f"共抓取 {len(all_data)} 条")
```

---

## 六、编辑器兼容性矩阵

不同网站使用不同的富文本编辑器，填文方式各不相同。**选错方式 = 内容丢失。**

| 编辑器 | 代表网站 | ✅ 正确方式 | ❌ 无效方式 |
|--------|---------|------------|------------|
| Draft.js | X/Twitter | `fill()` | execCommand, textContent= |
| ProseMirror | 头条号 | `keyboard.type(delay=1)` | fill(), innerHTML= |
| MediumEditor | 雪球 | `keyboard.type(delay=5)` + 前置移除 `.fake-placeholder` | JS注入innerHTML（发布按钮不会激活） |
| React ContentEditable | 知乎 | `fill()` | textContent= |
| Vue textarea | 微博 | JS native setter | fill() |
| Facebook Composer | Facebook | `keyboard.type(delay=25)` | fill() (CDP检测) |
| Tiptap | 小红书 | evaluate + dispatchEvent | fill() |

### 决策流程

```
编辑器是什么？
├── textarea / input[type=text]
│   └── ✅ page.locator().fill()
│
├── contenteditable div
│   ├── Draft.js (data-contents="true")?
│   │   └── ✅ page.locator().fill()
│   ├── ProseMirror (.ProseMirror 类)?
│   │   └── ✅ keyboard.type(delay=1)
│   ├── MediumEditor (.medium-editor-element 类)?
│   │   └── ✅ keyboard.type(delay=5) + 先移除 .fake-placeholder
│   ├── Facebook?
│   │   └── ✅ keyboard.type(delay=25)
│   └── 其他?
│       └── 先试 fill()，不行换 keyboard.type()
│
└── 未知
    └── 尝试顺序: fill() → keyboard.type() → evaluate + dispatchEvent
```

### MediumEditor 专用工作流（雪球 Xueqiu）

雪球使用 MediumEditor，有独特的 `fake-placeholder` 陷阱：

```python
# 1. 移除 placeholder（否则拦截所有点击和聚焦）
page.evaluate("""
    () => {
        const ph = document.querySelector('.fake-placeholder');
        if (ph) ph.remove();
        const ed = document.querySelector('[medium-editor-index][contenteditable="true"]');
        if (ed) { ed.focus(); ed.innerHTML = ''; }
    }
""")

# 2. 逐字打字（delay=5ms 是 MediumEditor 最佳值）
page.keyboard.type(text, delay=5)

# 3. 发布按钮是 <a class="lite-editor__submit"> — 必须找非 disabled 的
page.evaluate("""
    () => {
        const btns = Array.from(document.querySelectorAll('a, button'));
        const pub = btns.find(b =>
            b.textContent.trim() === '发布'
            && b.offsetParent !== null
            && !b.classList.contains('disabled')
        );
        if (pub) { pub.click(); return 'ok'; }
    }
""")
```

⚠️ **JS 注入 innerHTML 无效！** MediumEditor 有自己的内部状态跟踪，`el.innerHTML = text` 或 `el.textContent = text` 不会触发内部校验，发布按钮保持 disabled。

---

## 七、反检测策略

### 7.1 为什么真实 Chrome 比无头浏览器强

| 检测维度 | Headless Chrome | CDP 真实 Chrome |
|----------|:---:|:---:|
| `navigator.webdriver` | `true` ❌ | `false` ✅ |
| WebGL 指纹 | 异于正常 | 与用户一致 ✅ |
| 字体列表 | 异常 | 用户系统字体 ✅ |
| Canvas 指纹 | 可检测 | 与用户一致 ✅ |
| CDP 运行时检测 | 可暴露 | 部分暴露 ⚠️ |
| 浏览器扩展 | 无 | 用户真实扩展 ✅ |

**Facebook、Cloudflare 等高级反爬可以检测 CDP 连接本身。** 对于这些网站，用 `keyboard.type(delay=25)` 代替 `fill()`。

### 7.2 降低检测风险的技巧

```python
# 1. 不要瞬间填完大段文字 — 逐字打字
page.keyboard.type(text, delay=25)  # 25ms 间隔 ≈ 人类打字速度

# 2. 随机延迟
import random
time.sleep(random.uniform(1.5, 3.0))

# 3. 鼠标移动轨迹
page.mouse.move(x, y, steps=10)  # 10 步移动到目标

# 4. 不要在非活跃标签页操作
page.bring_to_front()

# 5. 控制操作频率
# X/Twitter: 间隔 3-5秒
# 微博: 间隔 2-3秒
# Facebook: 间隔 5-10秒
```

### 7.3 被检测时的应对

```python
# 症状：发布按钮点击无反应、fill() 内容消失
# → 网站检测到 CDP，切换策略

# 应对 1: 改 fill() 为 keyboard.type()
page.locator('editor').click()
page.keyboard.type(text, delay=25)

# 应对 2: 用原生事件代替 Playwright 事件
page.evaluate("""
    (text) => {
        const el = document.querySelector('editor');
        el.focus();
        document.execCommand('insertText', false, text);
        el.dispatchEvent(new Event('input', {bubbles: true}));
    }
""", text)
```

---

## 八、登录态管理

### 8.1 原理

用户在自己 Chrome 中登录一次 → Cookie/Token 保存在 `user-data-dir` → Agent 每次复用。

**Agent 永远不需要知道密码。** 用户手动登录，Agent 只操作已登录的页面。

### 8.2 Cookie 检查

```python
def has_auth(page, domain, cookie_name="auth_token"):
    """检查指定域名是否有登录 Cookie"""
    cookies = page.context.cookies([f"https://{domain}"])
    return any(c["name"] == cookie_name for c in cookies)
```

### 8.3 登录态丢失恢复

```python
# 如果 goto 后被重定向到登录页 → 登录态过期
# 不要尝试让 Agent 登录 → 通知用户手动登录

page.goto("https://x.com/home", wait_until="domcontentloaded")
time.sleep(3)

if "login" in page.url or "signin" in page.url.lower():
    print("⚠️ 登录态失效，需要用户重新登录")
    # 用户手动登录后继续
```

---

## 九、CAPTCHA 处理

> **平台专项参考**: `references/discuz-captcha-bypass.md` — Discuz! X3.4 论坛验证码绕过全流程（含 formhash/seccodehash 提取、GD 验证码 OCR、WAF 行为、登录锁、Discuz! 响应格式解析）。Discuz! 专用，其他平台只需 9.1-9.3 章节。

### 9.1 截图 + AI 识别

```python
# 遇到 CAPTCHA → 截图 → 发给 AI 视觉识别
page.locator('.captcha-image').screenshot(path="/tmp/captcha.png")

# AI 识别（需要 vision 工具）
from hermes_tools import vision_analyze
answer = vision_analyze("/tmp/captcha.png", 
    question="What characters or numbers are shown in this CAPTCHA image?")

# 填入答案
page.locator('input[name="captcha"]').fill(answer)
```

### 9.2 SMS 验证码

```python
# 用户手机收验证码 → 用户发给 Agent → Agent 填入
# 不拦截短信，不改用户手机
print("请将手机收到的验证码发给我")
code = input("验证码: ")
page.locator('input[placeholder*="验证码"]').fill(code)
```

### 9.3 滑动验证

```python
# 滑动验证（如阿里系滑块）
# 方式：拖动鼠标模拟
slider = page.locator('.slider-button')
box = slider.bounding_box()

page.mouse.move(box['x'] + box['width']/2, box['y'] + box['height']/2)
page.mouse.down()
page.mouse.move(box['x'] + 300, box['y'] + box['height']/2, steps=50)  # 慢慢拖
page.mouse.up()
```

---

## 十、常见操作模板

### 10.1 通用表单提交

```python
def fill_and_submit(page, fields, submit_text="提交"):
    """fields = {'选择器': '值'}"""
    for selector, value in fields.items():
        el = page.locator(selector).first
        el.click()
        el.fill(str(value))
        time.sleep(0.3)
    
    page.locator('button').filter(has_text=submit_text).click()
    time.sleep(3)
```

### 10.2 列表页批量操作

```python
def process_list(page, item_selector, action_fn, max_items=50):
    """遍历列表项并执行操作"""
    items = page.locator(item_selector).all()
    for i, item in enumerate(items[:max_items]):
        try:
            action_fn(item)
            print(f"处理 {i+1}/{min(len(items), max_items)}")
        except Exception as e:
            print(f"跳过 #{i}: {e}")
            continue
```

### 10.3 等待条件满足

```python
# 等待文本出现
page.locator('text=加载完成').wait_for(state='visible', timeout=30000)

# 等待元素数量
page.locator('.result').nth(9).wait_for(state='attached', timeout=30000)

# 等待网络空闲
page.wait_for_load_state('networkidle', timeout=30000)

# 等待 URL 变化
page.wait_for_url('**/success**', timeout=30000)
```

### 10.4 iframe 操作

```python
# 进入 iframe
frame = page.frame_locator('#payment-iframe')
frame.locator('input[name="card"]').fill("4111111111111111")

# 退出 iframe
page.locator('#outside-button').click()
```

---

## 十一、完整示例：自动登录+填写+提交

```python
#!/usr/bin/env python3
"""通用浏览器操作完整示例"""

from playwright.sync_api import sync_playwright
import urllib.request, json, time

GW = "172.29.48.1"
req = urllib.request.urlopen(f"http://{GW}:19223/json/version", timeout=5)
BROWSER_WS = json.loads(req.read())["webSocketDebuggerUrl"]

with sync_playwright() as p:
    browser = p.chromium.connect_over_cdp(BROWSER_WS)
    
    # 查找目标页面
    page = None
    for ctx in browser.contexts:
        for pg in ctx.pages:
            if "target-site.com" in pg.url:
                page = pg
                break
        if page: break
    
    if not page:
        print("未找到目标页面，创建新标签页")
        page = browser.contexts[0].new_page()
    
    page.bring_to_front()
    
    # 1. 导航
    page.goto("https://target-site.com/form", wait_until="domcontentloaded")
    time.sleep(3)
    
    # 2. 检查登录
    if "login" in page.url:
        print("需要登录，请手动登录后继续...")
        time.sleep(30)  # 给用户时间登录
    
    # 3. 填表
    page.locator('input[name="title"]').fill("我的标题")
    page.locator('textarea').fill("正文内容...")
    time.sleep(1)
    
    # 4. 上传文件
    page.locator('input[type="file"]').set_input_files("/mnt/d/document.pdf")
    time.sleep(5)
    
    # 5. 提交
    page.locator('button').filter(has_text='提交').click()
    time.sleep(5)
    
    # 6. 验证
    if "success" in page.url or "成功" in page.content():
        print("✅ 操作成功")
    else:
        print("⚠️ 请检查结果")
```

---

## 十二、故障排查

| 症状 | 可能原因 | 解决 |
|------|---------|------|
| CDP 连接超时 | relay2.py 未启动 | 在 Windows 运行 `python D:\relay2.py` |
| 403 Forbidden | 缺少 `--remote-allow-origins=*` | 重启 Chrome 加上参数 |
| `fill()` 内容消失 | 网站检测到 CDP | 换 `keyboard.type(delay=25)` |
| click 无反应 | 元素被遮罩覆盖 | `click(force=True)` 或 Ctrl+Enter |
| 文件上传失败 | Windows 路径不兼容 | 用 `/mnt/d/...` 不用 `D:\` |
| 新标签页跳登录 | 部分网站限制 | 必须用用户已打开的标签页 |
| `keyboard.type` 丢字 | delay 太小 | 增大 delay 到 5-25ms |
| 连续操作被限频 | 操作太密集 | 每条间隔 3-10 秒 |

---

## 十三、与社媒发布 Skill 的关系

本 Skill 是**通用浏览器操控能力层**。`hermes-social-media-automation` 是基于本 Skill 的**社交平台专用封装**。

**配套图片：** `/mnt/d/Hermes/skills-images/browser/` — architecture.png / cdp-vs-selenium.png / editor-compat.png（暗色主题，闲鱼展示用）

```
hermes-browser-automation        ← 本 Skill（通用层）
    ↓ 被引用
hermes-social-media-automation   ← 社媒专用（X/微博/头条/知乎/小红书/Facebook）
    ↓ 被调用
用户对话："帮我在 X 发一条推"
```

加载社媒 Skill 的 Agent 应同时加载本 Skill 以获得完整的浏览器操控基础能力。

---

## Skill 维护

每次遇到新的编辑器类型、新的反爬机制、新的坑，更新本 Skill 对应章节。这是一个不断演化的"浏览器操作宝典"。
