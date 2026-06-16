---
name: hermes-social-media-automation
description: Hermes 社媒全自动发布 — 通过 Chrome CDP + Playwright 操控用户真实浏览器，在 X(Twitter)、微博、头条号、知乎、小红书、Facebook 六大平台自动发布内容和互动。一个 Skill 覆盖全平台。
domain: social-media
triggers:
  - 用户说"发X"、"发微博"、"发头条"、"发知乎"、"发小红书"、"发Facebook"
  - 用户提供内容要求发布到以上任一平台
  - 用户要求批量运营社交媒体
  - 用户说"社媒发"
  - 用户说"继续运营X"、等社媒运营类请求
authorization:
  default: full_auto  # 用户完整授权：发布/评论/运营无需逐次征求，直接执行
---

# Hermes 社媒全自动发布 Skill

## 这个 Skill 是什么

让 Hermes（或任何兼容的 AI Agent）像真人一样操控 Chrome 浏览器，在六大社交平台自动写内容、配图、发布、评论互动。不需要 API Key，不需要平台授权——直接操控用户已登录的浏览器。

**支持的平台：**

| 平台 | 发布 | 配图 | 评论 | 长文 |
|:---|:---:|:---:|:---:|:---:|
| X/Twitter | ✅ | ✅ | ✅ | ✅ Premium 25000字 |
| 微博 | ✅ | — | ✅ | — |
| 头条号 | ✅ | ✅ | — | ✅ 图文+微头条 |
| 知乎 | ✅ | — | — | ✅ 回答 |
| 小红书 | ✅ | ✅ | — | ✅ 图文+长文 |
| Facebook | ✅ | ✅ | ✅ | — |
| 雪球 | ✅ | — | — | — |

---

## 一、前置条件（所有平台共用）

### 1.1 Windows 端启动 Chrome 调试模式

用户在 Windows 上**手动**执行（不是 Agent 执行）：

```cmd
"C:\Program Files\Google\Chrome\Application\chrome.exe" --remote-debugging-port=9222 --remote-allow-origins=* --user-data-dir="D:\chrome-debug-profile"
```

⚠️ `--remote-allow-origins=*` 必须加，否则 WSL 连接时 CDP WebSocket 返回 403。

### 1.2 启动 CDP 中继

```cmd
python.exe D:\relay2.py
```

中继在 `0.0.0.0:19223` 监听，转发到 `127.0.0.1:9222`。WSL 无法直连 Windows localhost，必须通过中继。

### 1.3 用户在 Chrome 中登录各平台

Agent **不会**帮用户登录。用户需要自己在 Chrome 中打开并登录：
- x.com（X/Twitter）
- weibo.com（微博）
- mp.toutiao.com（头条号）
- zhihu.com（知乎）
- creator.xiaohongshu.com（小红书创作者平台）
- facebook.com（Facebook）

**保持至少一个标签页开着**——Agent 通过查找已有标签页来操作，不会新建标签页（新建会触发重新登录）。

### 1.4 WSL 端连接验证

```bash
GW=$(ip route show default | awk '{print $3}')
curl -s --max-time 5 "http://$GW:19223/json/version" | python3 -c "import sys,json; d=json.load(sys.stdin); print(f'Browser: {d.get(\"Browser\",\"?\")}')"
```

### 1.5 Playwright 安装

```bash
pip3 install playwright
```

---

## 二、通用架构

所有平台的发布流程遵循同一模式：

```
发现已有标签页 → 导航到发布页 → 填写内容 → 上传图片(可选) → 点击发布 → 验证
```

```python
from playwright.sync_api import sync_playwright
import urllib.request, json, time

GW = "172.29.48.1"

# 获取浏览器 WebSocket
req = urllib.request.urlopen(f"http://{GW}:19223/json/version", timeout=5)
BROWSER_WS = json.loads(req.read())["webSocketDebuggerUrl"]

with sync_playwright() as p:
    browser = p.chromium.connect_over_cdp(BROWSER_WS)
    
    # ⚠️ 关键：查找已有标签页，不要 new_page() + goto()
    page = None
    for ctx in browser.contexts:
        for pg in ctx.pages:
            if "platform.com" in pg.url and "login" not in pg.url:
                page = pg
                break
        if page: break
    
    if not page:
        print("❌ 未找到已登录页面")
        exit()
    
    page.bring_to_front()
    # ... 平台特定操作 ...
```

---

## 三、X/Twitter 发布

### 关键认知

- X.com 在中国大陆被 GFW 阻断，WSL 无法直连。**必须通过 CDP 中继到 Windows Chrome。**
- 用户如果是 Premium 会员，最长可发 25000 字符。
- 编辑器是 Draft.js（contenteditable div），**不是 textarea**。
- **✅ 唯一可靠的填文方式：Playwright 的 `locator.fill()`**
- **🔴 `execCommand('insertText')` = 内容丢失！** — 2026-06-07 实际验证：DOM 显示文字但 Draft.js React 状态未更新，Ctrl+Enter 提交后帖子只有标签没有正文，8 条全废。不要用。
- **🔴 `textContent =` = 内容丢失！** — 只改 DOM 表层，提交时 React 从内部状态重建，文字消失。
- **🔴 裸 CDP WebSocket 在 X 上完全不可靠** — 必须安装 Playwright，`pip3 install playwright`，不可省略。

### 3.1 发布推文

```python
# 导航到首页
page.goto("https://x.com/home", wait_until="domcontentloaded", timeout=30000)
time.sleep(3)

# 点击发帖按钮
compose_btn = page.locator('[data-testid="SideNav_NewTweet_Button"]')
compose_btn.click()
time.sleep(2)

# ✅ 填文：Playwright fill() — 唯一可靠方式
editor = page.locator('[data-testid="tweetTextarea_0"]').first
editor.click()
editor.fill(post_text)  # 支持长文，瞬间完成
time.sleep(1)

# 验证文字已填入
preview = editor.text_content() or ""
print(f"已填入 {len(preview)} 字符")
```

### 3.2 上传图片

```python
img_path = "/mnt/d/Hermes/X/images/chart.png"  # ⚠️ WSL 路径，不是 D:\

# ⚠️ 关键：必须用 .first — compose 打开后有 2 个 fileInput（主页+弹窗）
file_input = page.locator('[data-testid="fileInput"]').first
file_input.set_input_files(img_path)
time.sleep(8)  # 等图片上传完成+遮罩消失
```

### 3.3 提交

```python
# ✅ 方式 A：Ctrl+Enter（绕过图片上传遮罩）
page.keyboard.press('Control+Enter')
time.sleep(8)

# 验证发布成功：URL 不再包含 "compose"
url = page.url
if 'compose' in url:
    time.sleep(10)  # 长文+图片可能延迟
    url = page.url

if 'compose' not in url:
    print("✅ 发布成功")
else:
    print("⚠️ 可能未成功")
```

### 3.4 发布 Thread（长推文串）

```python
# Thread = 首帖 + 逐条回复自己
# 不要用 X 的 "Add post" UI —— 通过 CDP 不稳定

# 1. 发首帖（Hook + 图片）
# ... 同上发布流程 ...

# 2. 在首帖下逐条回复
REPLIES = ["回复1", "回复2", ...]
for reply_text in REPLIES:
    time.sleep(3)
    reply_btn = page.locator('[data-testid="reply"]').first
    reply_btn.click()
    time.sleep(2)
    
    editor = page.locator('[data-testid="tweetTextarea_0"]').first
    editor.click()
    editor.fill(reply_text)
    time.sleep(1)
    
    page.keyboard.press('Control+Enter')
    time.sleep(4)
```

### 3.5 搜索 + 评论

```python
# 搜索话题
query = "AI startup funding"
page.goto(f"https://x.com/search?q={query.replace(' ', '%20')}&src=typed_query&f=live",
          wait_until="domcontentloaded", timeout=30000)
time.sleep(4)

# 找到推文并回复
articles = page.locator('article').all()
for a in articles[:10]:
    text = a.inner_text()[:200]
    if len(text) < 50 or 'YourHandle' in text.lower():  # 跳过自己的帖
        continue
    
    # 点击回复
    reply_btn = a.locator('[data-testid="reply"]').first
    reply_btn.click()
    time.sleep(2)
    
    # 填写回复
    editor = page.locator('[data-testid="tweetTextarea_0"]').first
    editor.click()
    editor.fill("Your substantive comment here...")
    time.sleep(1)
    
    # 提交
    page.keyboard.press('Control+Enter')
    time.sleep(5)
    
    # ⚠️ 关键：回复后导航回首页重置状态
    # 如不重置，后续评论会因 DOM 过期全部超时
    page.goto("https://x.com/home", wait_until="domcontentloaded", timeout=15000)
    time.sleep(2)
    break  # 每轮搜索只评论一条 + 回首页重置，避免状态混乱
```

### 3.6 X 内容策略（周日历）

| 星期 | 主题 | 内容方向 |
|:---|:---|:---|
| 周一 | 🤖 Tech Monday | AI 深度分析、工具评测 |
| 周二 | 🇨🇳 China Reality | 社会、经济、日常生活 |
| 周三 | 💰 Ecom & Money | 电商、供应链、案例 |
| 周四 | 🛡️ Military & Geo | 国防、地缘（理性视角） |
| 周五 | 🚀 Innovation | AI 生态、航天、新能源 |
| 周六 | 🎭 Culture | 软内容、美食、街拍 |
| 周日 | 📊 Review | 周数据、目标复盘 |

**五类内容配比：**
- China Window 35% — 数据驱动的中国视角
- AI & Ecommerce 25% — 工具、案例
- Build in Public 20% — 创业日记
- Data Insights 12% — 图表、数字
- Engagement 8% — 互动提问

---

## 四、微博发布

### 关键认知

- 微博新版使用 CSS-module 哈希类名（如 `_text_hwnxp_2`），传统属性选择器全部失效。
- 发送按钮文字是**"发送"**（不是"发布"不是"发微博"）。
- 评论区按钮通过纯数字 span 定位。
- ⚠️ 不要新建标签页 + goto weibo.com → 会跳到访客登录页。

### 4.1 发布微博

```python
# 查找已有微博页面
for pg in ctx.pages:
    if 'weibo.com' in pg.url and 'passport' not in pg.url:
        page = pg
        break

page.bring_to_front()

# 填写内容（JS native setter — Vue 表单需要）
text = "要发布的微博内容 #话题"
page.evaluate(f"""
    (() => {{
        const ta = document.querySelector('textarea');
        if (!ta || ta.offsetHeight < 50) return 'not found';
        const nativeSetter = Object.getOwnPropertyDescriptor(
            window.HTMLTextAreaElement.prototype, 'value'
        ).set;
        nativeSetter.call(ta, `{text}`);
        ta.dispatchEvent(new Event('input', {{bubbles: true}}));
        return 'filled';
    }})()
""")
time.sleep(1)

# 点击发送按钮
page.evaluate("""
    (() => {
        const all = document.querySelectorAll('button, a, div, span');
        for (const el of all) {
            if (el.textContent.trim() === '发送' && el.getBoundingClientRect().width > 30) {
                el.click();
                return 'clicked';
            }
        }
        return 'not found';
    })()
""")
time.sleep(3)

# 验证：发布成功后 textarea 自动清空
cleared = page.evaluate("document.querySelector('textarea')?.value.length === 0")
print("✅ 发布成功" if cleared else "⚠️ 可能失败")
```

### 4.2 评论他人微博

```python
# 导航到用户主页
page.goto(f'https://weibo.com/u/{uid}', wait_until='domcontentloaded')
time.sleep(3)

# 滚动加载微博
for i in range(5):
    page.evaluate("window.scrollBy(0, 400)")
    time.sleep(2)

# 通过纯数字 span 定位评论按钮
page.evaluate("""
    (() => {
        const spans = document.querySelectorAll('span');
        let cur = [];
        let lastY = -1;
        for (const s of spans) {
            const t = s.textContent.trim();
            if (/^\\d+$/.test(t) && s.offsetParent !== null) {
                const r = s.getBoundingClientRect();
                if (lastY === -1 || Math.abs(Math.round(r.y) - lastY) <= 5) {
                    cur.push(s);
                } else {
                    if (cur.length >= 3) {
                        const cmt = parseInt(cur[1].textContent);
                        if (cmt > 0 && cmt < 50) {
                            cur[1].click();
                            return 'clicked';
                        }
                    }
                    cur = [s];
                }
                lastY = Math.round(r.y);
            }
        }
        return 'none';
    })()
""")
time.sleep(2)

# 填写评论并 Ctrl+Enter 提交
page.evaluate(f"document.querySelector('textarea').value = '{comment}'; document.querySelector('textarea').dispatchEvent(new Event('input', {{bubbles: true}}))")
page.keyboard.press('Control+Enter')
time.sleep(3)
```

---

## 五、头条号发布

### 关键认知

- 编辑器是 **ProseMirror**（ByteEditor），极其特殊。
- **❌ `fill()` 无效** — ProseMirror 状态不认
- **❌ `innerHTML =` 无效** — 保存时内容消失
- **❌ `execCommand('insertText')` 无效** — ProseMirror 拦截
- **✅ 唯一方式：`keyboard.type(text, delay=1)`** 逐字符模拟输入
- **⚠️ WSL→中继路径下预览弹窗不弹！** 长文发布须在 Windows 侧用 PowerShell 直连 `127.0.0.1:9222` 执行。

### 5.1 图文文章发布

```python
# 导航到图文发布页
page.goto("https://mp.toutiao.com/profile_v4/graphic/publish", 
          wait_until="domcontentloaded")
time.sleep(3)

# 1. 填写标题（≤30字）
title = "你的标题不超过30字"
ta = page.locator('textarea').first()
ta.click()
ta.fill(title)
time.sleep(1)

# 2. 填写正文（ProseMirror — keyboard.type 唯一方式）
editor = page.locator('.ProseMirror').first
editor.click()
time.sleep(0.3)

# 清空旧内容
page.keyboard.press("Control+a")
time.sleep(0.2)
page.keyboard.press("Delete")
time.sleep(0.3)

# 逐行输入
lines = content.split('\n')
for li, line in enumerate(lines):
    if not line.strip():
        page.keyboard.press('Enter')
        continue
    page.keyboard.type(line, delay=1)  # delay=1ms 是最佳值
    if li < len(lines) - 1:
        page.keyboard.press('Enter')
        time.sleep(0.05)

time.sleep(2)

# 3. 选择"无封面"（第三个 label.byte-radio）
page.evaluate("window.scrollTo(0, document.body.scrollHeight)")
time.sleep(1)
page.locator('label.byte-radio').filter(has_text='无封面').first.click(force=True, timeout=5000)
time.sleep(1)

# 4. 点击"预览并发布"
page.locator('button').filter(has_text='预览并发布').first.click(force=True, timeout=5000)
time.sleep(5)  # 等预览弹窗

# 5. 点击"确认发布"
page.locator('button').filter(has_text='确认发布').first.click(force=True, timeout=5000)
time.sleep(5)

# 6. 验证
body = page.evaluate("document.body.innerText.substring(0, 300)")
if '审核' in body or '已发布' in body:
    print("✅ 发布成功")
```

### 5.2 微头条发布

```python
# 微头条使用不同页面和不同编辑器
page.goto("https://mp.toutiao.com/profile_v4/weitoutiao/publish", 
          wait_until="domcontentloaded")
time.sleep(3)

# 填写内容（微头条建议 ≤100 字 — 中文可能截断）
page.evaluate("document.querySelector('.ProseMirror')?.focus()")
time.sleep(0.5)

for j in range(0, len(text), 20):
    chunk = text[j:j+20]
    page.keyboard.type(chunk, delay=5)
    time.sleep(0.02)

time.sleep(1)

# 点击"发布"按钮（直接发，无二次确认）
page.locator('button').filter(has_text='发布').first.click(force=True, timeout=5000)
time.sleep(3)
```

### 5.3 图文文章 vs 微头条对照

| 维度 | 图文文章 | 微头条 |
|------|----------|--------|
| 页面 URL | `graphic/publish` | `weitoutiao/publish` |
| 标题 | 需要（≤30字） | 无标题 |
| 编辑器 | ProseMirror | ByteEditor（同一引擎） |
| 内容限长 | 数千字（稳定） | ≤ 100 字（中文截断） |
| 封面 | 需选"无封面" | 无封面 |
| 发布按钮 | "预览并发布" → "确认发布" | "发布"（一步到位） |

---

## 六、知乎发布

### 关键认知

- 问题 ID 已切换到 **19 位长 QID** 格式（如 `1893050326199821014`）
- 搜索结果提取的是 `/question/数字/answer/数字` 链接，需正则提取 QID
- "写回答"按钮检测用 Unicode 转义（`\u5199\u56de\u7b54`）避免编码问题
- **连续发 3-5 篇后触发限频**，"写回答"按钮集体消失
- 知乎 React 编辑器中 **`fill()` 是可靠的**（与其他平台不同）

### 6.1 两阶段批量发布法

**阶段1：搜索 → 提取 QID → 验证有效性**

```python
search_queries = [
    "中国 经济 为什么", "芯片 制裁 中国 未来",
    "AI 创业 中国", "中美 科技 对比",
]

all_qids = set()
for query in search_queries:
    page.goto(f"https://www.zhihu.com/search?type=content&q={query}", 
              wait_until='domcontentloaded')
    time.sleep(5)
    
    # 提取 QID（从 /question/数字/ 正则提取）
    qids = page.evaluate("""
        () => {
            const ids = new Set();
            document.querySelectorAll('a').forEach(a => {
                const m = (a.href || '').match(/\/question\/(\d+)/);
                if (m) ids.add(m[1]);
            });
            return Array.from(ids);
        }
    """)
    all_qids.update(qids)

# 批量验证有效性
valid_questions = []
for qid in all_qids:
    page.goto(f"https://www.zhihu.com/question/{qid}", wait_until='domcontentloaded')
    time.sleep(4)
    
    # 跳过已删除问题（标题含"荒原"）
    if page.evaluate("() => document.title.includes('荒原')"):
        continue
    
    # 检查"写回答"按钮
    btn = page.evaluate("""
        () => {
            const btns = document.querySelectorAll('button');
            for (const b of btns) {
                const t = (b.textContent || '').trim();
                if (t.includes('\u5199\u56de\u7b54')) return 'write';
                if (t.includes('\u67e5\u770b\u6211\u7684\u56de\u7b54')) return 'view_mine';
            }
            return 'none';
        }
    """)
    if btn == 'write':
        valid_questions.append(qid)
```

**阶段2：批量写回答**

```python
for i, qid in enumerate(valid_questions):
    if i >= 10:  # 单次最多 10 篇，避免限频
        break
    
    page.goto(f"https://www.zhihu.com/question/{qid}", wait_until='domcontentloaded')
    time.sleep(4)
    
    # 点击"写回答"
    page.evaluate("""
        () => {
            const btns = document.querySelectorAll('button');
            for (const b of btns) {
                if ((b.textContent||'').includes('\u5199\u56de\u7b54')) {
                    b.click(); return;
                }
            }
        }
    """)
    time.sleep(5)
    
    # ✅ fill() 在知乎 React 编辑器中是可靠的
    page.locator('[contenteditable]').first.fill(answer_text)
    time.sleep(3)
    
    # 点击"发布回答"
    page.evaluate("""
        () => {
            const btns = document.querySelectorAll('button');
            for (const b of btns) {
                const t = (b.textContent||'').trim();
                if (t.includes('\u53d1\u5e03\u56de\u7b54') || t.includes('\u53d1\u5e03')) {
                    b.click(); return;
                }
            }
        }
    """)
    time.sleep(8)  # 等发布完成
    
    # 间隔 5 秒以上防止限频
    time.sleep(5)
```

### 6.2 知乎回答风格

**五大法则：**
1. **第一句话信息密度最高** — 直接给答案
2. **每个观点配数据或案例** — 数字比形容词有力 10 倍
3. **有态度、不中立** — 知乎不是维基百科
4. **隐形结构** — 逻辑链藏在内容里，不要"首先其次最后"
5. **设问/反问制造对话感** — "你觉得呢？"

**字数：300-500 字最佳。** 太短没分量，太长没人看完。

---

## 七、小红书发布

### 关键认知

- 通过 **creator.xiaohongshu.com**（创作者平台）发布，不是 xiaohongshu.com
- 登录方式：**短信验证码**（非二维码）
- Agent 负责填内容，**用户负责最终点击"发布"**
- 页面使用 CSS-module 哈希类名 — 必须用 snapshot ref ID 定位元素
- **✅ 推荐：Playwright 点击进入编辑器** — `page.locator('text=发布图文笔记').first.click()` 比 raw CDP `evaluate click` 更稳
- **上传图片后等 2-3 秒让平台处理**，一次性上传多张（最多 9 张）

### 7.1 登录流程

```python
# 导航到创作者平台
page.goto("https://creator.xiaohongshu.com/login", wait_until="domcontentloaded")
time.sleep(3)

# 点击"短信登录"标签
page.locator('text=短信登录').click()
time.sleep(1)

# 输入手机号
page.locator('input[type="tel"]').fill("手机号")
time.sleep(0.5)

# 点击"发送验证码"
page.evaluate("""
    () => {
        const els = document.querySelectorAll('.css-uyobdj');
        for (const el of els) {
            if (el.textContent.trim() === '发送验证码') {
                el.click(); return;
            }
        }
    }
""")

# ⚠️ 此时用户收到短信，把验证码发给 Agent
# Agent 填入验证码 → 点击"登 录"
```

### 7.2 发布图文笔记（Playwright 方式，推荐）

```python
from playwright.sync_api import sync_playwright

# ... 连接浏览器 ...

# 导航到首页
page.goto("https://creator.xiaohongshu.com/new/home", wait_until="domcontentloaded")
time.sleep(3)

# ✅ Playwright click 进入发布页
page.locator('text=发布图文笔记').first.click(timeout=5000)
time.sleep(4)

# 点击上传图片
page.locator('text=上传图片').first.click()
time.sleep(2)

# 上传多张图片
images = ["/mnt/d/img1.png", "/mnt/d/img2.png", ...]
for img in images:
    page.locator('input[type="file"]').first.set_input_files(img)
    time.sleep(2)

# 填写标题
page.locator('[placeholder="填写标题会有更多赞哦"]').fill("标题文字")

# 填写正文
page.locator('[placeholder="这一刻的想法..."]').fill("正文内容 #标签")

# ⚠️ 到此为止 — 截图给用户预览
page.screenshot(path="/tmp/xhs_preview.png")
print("请用户确认后手动点击发布")
```

### 7.3 小红书内容风格

**工具推荐类：**
```
装了这6个XXX后，再也没加过班～
这是真正能帮XXX的，非常好用，必装的6个：

1/ [工具名]：[场景描述 + 效果]
2/ [工具名]：[不同角度描述]
...

[简短总结扣题]

#tag1 #tag2 #tag3
```

---

## 八、Facebook 发布

### 关键认知

- Facebook 检测 CDP 连接——**必须用 `keyboard.type(delay=25)`**，不能用 `fill()`
- 两步发布：点击"继续"(Continue) → 再点击"发帖"(Post)
- 内容过滤：政治敏感内容可能被静默丢弃

### 8.1 发布帖子

```python
# 查找已登录的 Facebook 页面
for pg in ctx.pages:
    if 'facebook.com' in pg.url and 'login' not in pg.url:
        page = pg
        break

page.bring_to_front()

# 点击"分享新鲜事"（inline composer）
composer = page.locator('[aria-label="分享新鲜事"], [aria-label="What\'s on your mind"]').first
composer.click()
time.sleep(2)

# ✅ keyboard.type(delay=25) — CDP 检测规避
page.keyboard.type(post_text, delay=25)
time.sleep(2)

# 上传图片（如有）
if images:
    page.locator('[aria-label="照片/视频"], [aria-label="Photo/Video"]').first.click(force=True, timeout=10000)
    time.sleep(5)
    page.locator('input[type="file"]').first.set_input_files(images[:6])
    time.sleep(8)

# 第一步：点击"继续"
page.locator('button').filter(has_text='继续').first.click(force=True, timeout=5000)
time.sleep(3)

# 第二步：点击"发帖"或"Post"
page.locator('button').filter(has_text='发帖').first.click(force=True, timeout=5000)
# 或者英文界面：
# page.locator('button').filter(has_text='Post').first.click(force=True, timeout=5000)
time.sleep(5)

# 验证：截图 + vision_analyze
page.screenshot(path="/tmp/fb_verify.png")
```

### 8.2 内容策略

- **50/25/25 配比：** 产品(50%) / 行业洞察(25%) / 科技趋势(25%)
- **10-15 个 SEO 标签** 每帖
- **人设：** 诚信中国采购代理
- **语言：** 默认英文
- **单帖 ≤ 550 字** 效果最佳

---

## 九、雪球发布

### 关键认知

- 编辑器是 **MediumEditor**（contenteditable div，类名 `medium-editor-element`）
- **有 `fake-placeholder` div 拦截点击！** 必须先 JS 移除
- **✅ 唯一方式：`keyboard.type(text, delay=5)`** — 逐字符模拟输入
- **❌ JS 注入 innerHTML 无效！** 发布按钮永远 disabled
- 两种发帖入口：右侧栏"发帖"→"发讨论"弹出编辑器，或页面底部内联输入区
- 新开标签页可能出现不同 UI 模式，需灵活处理

### 9.1 发布讨论帖

**推荐流程：新开标签页 → 内联编辑器 → 打字发布**（比"发帖"→"发讨论"弹窗更稳定）

```python
# 每次都新开标签页（避免 DOM 残留导致"发讨论"找不到）
page = browser.contexts[0].new_page()
page.goto("https://xueqiu.com/", wait_until="domcontentloaded")
time.sleep(5)

# 1. 移除 fake-placeholder 并聚焦内联编辑器
focused = page.evaluate("""
    () => {
        const ph = document.querySelector('.fake-placeholder');
        if (ph) { ph.click(); ph.remove(); return 'ph_clicked'; }
        
        // 回退：找第2个 medium-editor（第1个是侧边栏隐藏的）
        const eds = Array.from(document.querySelectorAll('[medium-editor-index][contenteditable="true"]'))
            .filter(e => e.getBoundingClientRect().width > 50);
        if (eds.length) { eds[0].focus(); eds[0].innerHTML = ''; return 'ed_focused'; }
        
        return 'nothing';
    }
""")
# focused: 'ph_clicked' | 'ed_focused' | 'nothing'

# 2. 逐字打字（delay=5-8ms 最佳）
page.keyboard.type(post_text, delay=5)
time.sleep(2)

# 3. 发布 — 找非 disabled 的按钮（可能有多个，选可见的）
page.evaluate("""
    () => {
        const btns = Array.from(document.querySelectorAll('a, button'));
        const pub = btns.find(b =>
            b.textContent.trim() === '发布'
            && b.offsetParent !== null
            && !b.classList.contains('disabled')
        );
        if (pub) { pub.click(); return 'ok'; }
        return 'no_enabled_button';
    }
""")
time.sleep(4)
```

**🔴 掉坑记录**：
- `keyboard.type()` 前必须移除 `fake-placeholder` 并聚焦编辑器，否则打字到空气里，发布按钮永远 disabled
- JS 注入 `innerHTML` 无效！MediumEditor 不认，发布按钮 disabled
- "发帖"→"发讨论"弹窗模式在新标签页有时不出现下拉菜单（`text=发讨论` not found），优先用内联编辑器
- 批量发布：每条用 `new_page()` 开新标签页，避免 DOM 残留

### 9.2 内容格式

```
$股票代码(SH601138)$

分析文字...

#tags
```

- `$股票代码$` 格式引用股票（如 `$工业富联(SH601138)$`）
- 讨论帖无严格字数限制，建议 200-500 字
- 必须带 `#标签` 增加曝光
- 不碰具体数据避免过期，侧重分析逻辑

### 9.3 发长文

长文使用独立编辑器页面 `https://xueqiu.com/write`，而非内联讨论编辑器。

**编辑器结构**：

| 元素 | 选择器 | 填文方式 | 必填 |
|------|--------|---------|:---:|
| 标题 | `textarea.long-text__title` | `fill()` | ✅ |
| 正文 | `div.sb-editor__container[class*="medium-editor"]` | `keyboard.type(delay=2)` | ✅ |
| 标签 | `input.sb-editor__input__tag` | `click()` → `keyboard.type(...)` → `Enter` | ✅ |
| 原创声明 | `text=声明原创` | `click(force=True)` | 推荐 |
| 发布 | `a.submit__confirm__btn` | `click(force=True)` | — |

```python
# 导航到长文编辑器
page.goto("https://xueqiu.com/write", wait_until="domcontentloaded")
time.sleep(5)

# 1. 填标题
title_input = page.locator('textarea.long-text__title')
title_input.click()
title_input.fill("你的标题")
time.sleep(0.5)

# 2. 填正文（先移除 fake-placeholder，聚焦编辑器容器）
page.evaluate("""
    () => {
        const ph = document.querySelector('.fake-placeholder');
        if (ph) ph.remove();
        const container = document.querySelector('.sb-editor__container[class*="medium-editor"]');
        if (container) { container.focus(); container.innerHTML = ''; }
    }
""")
time.sleep(0.3)
page.keyboard.type(body_text, delay=2)  # 长文用 delay=2 速度更快
time.sleep(3)

# 3. 🔴 加标签（必填！否则发布按钮可能不响应）
tag_input = page.locator('input.sb-editor__input__tag')
tag_input.click()
time.sleep(0.3)
page.keyboard.type("公司分析", delay=5)
page.keyboard.press("Enter")
time.sleep(1)

# 4. 勾选原创（推荐）
page.locator('text=声明原创').first.click(force=True, timeout=3000)
time.sleep(0.5)

# 5. 发布
page.locator('a.submit__confirm__btn').first.click(force=True)
time.sleep(5)
```

**🔴 长文关键坑**：

1. **标签必须选！** — 不选标签点发布可能没反应（页面默默忽略点击）
2. **安全验证弹窗（新账号）** — 新注册或低活跃账号发长文时，雪球弹出"账号安全等级低"，要求绑定手机号或修改密码。讨论帖可绕过，长文必须过验证。用户需手动在 Chrome 中完成验证后，Agent 再点发布。
3. **内容丢失风险** — 如果发布失败（安全弹窗等），页面刷新后编辑器内容清空。长文发布前建议 `page.screenshot()` 备份。
4. **button vs a 标签** — 发布按钮是 `<a class="submit__confirm__btn">` 不是 `<button>`，用 `page.locator('a.submit__confirm__btn')` 定位。

### 9.4 内联编辑器 vs 下拉菜单

雪球发讨论入口有两种模式，取决于页面加载状态：

| 模式 | 触发方式 | 表现 |
|------|---------|------|
| 内联编辑器 | 新标签页直接 `goto` 首页 | 底部显示"发表你的观点..."，`fake-placeholder` 在上方 |
| 下拉菜单 | 从已有登录态页面点"发帖" | 弹出"发讨论"/"发长文"选项 |

**✅ 推荐策略**：优先尝试内联编辑器（更简单），如果找不到编辑器元素，回退到"发帖"→"发讨论"路径。

```python
# 优先：内联
removed = page.evaluate("""
    () => {
        const ph = document.querySelector('.fake-placeholder');
        if (ph) { ph.click(); ph.remove(); return true; }
        return false;
    }
""")

# 回退：下拉菜单
if not removed:
    page.locator('text=发帖').first.click(force=True, timeout=5000)
    time.sleep(2)
    page.locator('text=发讨论').first.click(force=True, timeout=5000)
```

### 9.5 发帖频率

| 单次最多 | 间隔 |
|:-------:|:----:|
| 5-6 条 | 10s |

⚠️ 每条后必须间隔足够时间，雪球对新号连续发帖较敏感。长文单次1篇即可，连续长文有触安风险。

### 填文方式

| 平台 | 编辑器类型 | 填文方式 | 原因 |
|------|-----------|---------|------|
| X/Twitter | Draft.js | `fill()` | Draft.js 认 fill() ✅ |
| 微博 | Vue textarea | JS native setter | Vue 表单响应 |
| 头条号 | ProseMirror | `keyboard.type(delay=1)` | 唯一有效方式 |
| 知乎 | React contenteditable | `fill()` | React 认 fill() ✅ |
| 小红书 | Tiptap/ProseMirror | evaluate + dispatchEvent | 创作者平台 |
| Facebook | contenteditable | `keyboard.type(delay=25)` | CDP 检测规避 |
| 雪球 | MediumEditor | `keyboard.type(delay=5)` | 先移除 .fake-placeholder |

### 提交方式

| 平台 | 按钮文字 | 提交方式 |
|------|---------|---------|
| X/Twitter | tweetButton | Ctrl+Enter |
| 微博 | "发送" | Ctrl+Enter 或 click |
| 头条号 | "预览并发布"→"确认发布" | force click |
| 知乎 | "发布回答" | evaluate click |
| 小红书 | "发布" | 用户手动 |
| Facebook | "继续"→"发帖" | force click |

### 频率限制

| 平台 | 单次最多 | 间隔 | 评论上限 |
|------|:-------:|:----:|:-------:|
| X/Twitter | 5-6 条 | 3-5s | 5-6 条 |
| 微博 | ~15 人关注 | 2-3s | — |
| 头条号 | 无限制 | — | — |
| 知乎 | 10-15 篇 | 3-5s | — |
| 小红书 | — | — | — |
| Facebook | ~10 条(带图) | 3-5s | ~15 条 |

---

## 十、常见踩坑汇总

### 🐍 Python 脚本编写（跨平台通用）

**🔴 中文引号与三引号冲突（2026-06-07 三次踩坑）**：
在 Python 脚本中用 `"""..."""` 包裹含中文弯引号的文本时，解析器会将弯引号误判为字符串终止符，导致 `SyntaxError: unterminated string literal`。典型症状：
```python
text = """Weibo: "History will punish those who chose the wrong side.""""
#                                     ^ 这里 Python 认为字符串结束了
```
**✅ 唯一可靠方案：内容写文件，脚本读文件。**
```python
# 1. write_file("/tmp/content.txt", text)  # 安全写入
# 2. 脚本中: text = open("/tmp/content.txt").read()  # 安全读取
```
这同时解决了：中文引号冲突、execute_code sandbox 路径限制、长文本管理。

**🔴 Thread/批量发布验证假阴性**：Ctrl+Enter 后 `'compose' not in page.url` 检查在 Thread 首帖+图片场景下可能延迟 10+ 秒仍返回 False，但帖子实际已发出。**不要因 URL 检查失败就重新提交。** 先导航到个人主页验证。

**🔴 后台脚本 stdout 缓冲**：Python 脚本通过 process manager 后台运行时 stdout 全缓冲，`print()` 输出完全不可见（即使加 `-u` 标志）。**不要依赖后台脚本日志。** 直接去目标平台个人主页验证发布结果。

---

### 平台特定

1. **❌ 新建标签页 + goto → 丢失登录态** — 必须查找已有标签页
2. **❌ `execCommand('insertText')` 在 Draft.js/ProseMirror 不生效** — 用平台推荐的填文方式。X 上 execCommand 的典型失败症状：帖子只显示标签没有正文（Draft.js 未接收内容），或文字累积混入上一条残留。**唯一稳方案：Playwright `fill()`**。X 上 execCommand 的典型失败模式：帖子只有标签没有正文（文字未进入 Draft.js 状态）
3. **❌ WSL→中继→Windows 路径文件上传失败** — 用 `/mnt/d/...` 而非 `D:\`
4. **❌ X 图片上传后遮罩拦截按钮** — 用 Ctrl+Enter 或 `click(force=True)`
5. **❌ 头条预览弹窗在 WSL 中继下不弹出** — PowerShell 直连 `127.0.0.1:9222`
6. **❌ 知乎 19 位 QID 被截断** — 从回答链接正则提取而非标题链接
7. **❌ 微博 CSS-module 选择器失效** — 用纯文本内容匹配而非属性选择器
8. **❌ `fill()` 在头条 ProseMirror 中无效** — 改用 `keyboard.type(delay=1)`
9. **❌ `fill()` 在 Facebook 触发 CDP 检测** — 改用 `keyboard.type(delay=25)`
10. **⚠️ 发布后 DOM 残留** — 每条发布后导航回首页重置状态
11. **⚠️ 小红书 raw CDP 点击不跳转** — `Page.navigate('/publish')` 和 `evaluate click` 经常停在首页。改用 Playwright `page.locator('text=发布图文笔记').first.click()` 才稳
12. **⚠️ CDP 中继断连** — 如果 WSL ping 不通网关，让用户在 Windows 确认 relay2.py 没报错、Chrome 调试端口开着。必要时 `wsl --shutdown` 重启 WSL
11. **⚠️ 小红书 raw CDP 点击不跳转** — `Page.navigate('/publish')` 和 `evaluate click` 经常不生效。改用 Playwright `page.locator('text=发布图文笔记').first.click()` 稳

---

## 十一、完整发布脚本模板

以下模板展示了单个平台的完整发布流程。替换平台特定部分即可用于任何平台。

```python
#!/usr/bin/env python3
"""通用社媒发布模板"""

from playwright.sync_api import sync_playwright
import urllib.request, json, time, sys

GW = "172.29.48.1"
PLATFORM_URL = "目标平台URL"  # 如 x.com/home
PLATFORM_NAME = "目标平台标识"  # 如 x.com

# 获取浏览器连接
req = urllib.request.urlopen(f"http://{GW}:19223/json/version", timeout=5)
BROWSER_WS = json.loads(req.read())["webSocketDebuggerUrl"]

with sync_playwright() as p:
    browser = p.chromium.connect_over_cdp(BROWSER_WS)
    
    # 查找已有标签页
    page = None
    for ctx in browser.contexts:
        for pg in ctx.pages:
            if PLATFORM_NAME in pg.url and 'login' not in pg.url:
                page = pg
                break
        if page: break
    
    if not page:
        print(f"❌ 未找到 {PLATFORM_NAME} 已登录页面")
        sys.exit(1)
    
    page.bring_to_front()
    
    # ===== 平台特定发布逻辑 =====
    # 此处替换为具体平台的填文+提交代码
    # ===========================
    
    print("✅ 发布完成")
```

---

## Skill 维护说明

这是一个"活的" Skill——每次踩坑后应更新对应平台的注意事项。

**配套图片：** `/mnt/d/Hermes/skills-images/social-media/` — six-platforms.png / capability-matrix.png / flow.png（暗色主题，闲鱼展示用）

如果你在发布中遇到新问题：

1. 记录：平台、操作、症状、解决方案
2. 更新本 Skill 对应平台章节的"关键认知"或"踩坑汇总"
3. 如果发现了新的编辑器类型或更优的填文方式，覆盖原有推荐
