
<p align="center">
  <img src="https://placehold.co/600x150/ffffff/5b5fe7?text=Rigi+AI+Commons&font=Montserrat" width="500">
</p>

<p align="center">
  <strong>Your AI can chat. We make it work.</strong><br>
  <strong>给你的 AI 装上手脚——不再只会聊天，真的能干活。</strong>
</p>

<p align="center">
  <a href="https://github.com/xiaoyaotsyx-dotcom/Rigi-AI-Commons/blob/main/LICENSE"><img src="https://img.shields.io/badge/license-AGPLv3-blue"></a>
  <a href="#"><img src="https://img.shields.io/github/stars/xiaoyaotsyx-dotcom/Rigi-AI-Commons"></a>
</p>

---

## What is Rigi AI Commons? · 这是什么？

**A collection of AI agent skills that turn your assistant from a chatbot into a doer.** Each skill gives your AI the ability to control a browser — search the web, fill forms, upload files, post on social media, extract data. No API keys. No coding. Everything runs on your own computer, using your own Chrome, your own logins.

**一套 AI Agent 技能包，让你的 AI 助手从"陪聊"变成"干活"。** 每个技能赋予 AI 浏览器操控能力——搜索网页、填写表单、上传文件、发布社媒、提取数据。不需要 API，不需要编程。一切在你自己的电脑上运行，用你自己的 Chrome、你自己的登录态。

---

## Who Is This For? · 谁需要这个？

| You are... · 你是... | You probably... · 你可能... |
|------|------|
| 🛒 跨境卖家 | 每天手动填店小秘 2-3 小时，想自动化 |
| 📈 个人投资者 | 想系统化分析股票，但没时间手动拉数据 |
| 📱 内容创作者 | 想一键发 X/微博/知乎/小红书，不用挨个平台贴 |
| 🔒 开发者/安全从业者 | 想快速审计竞品网站前端安全 |
| 🧠 AI 深度用户 | 想给 AI 定制专业人设 + 实操能力，不止聊天 |

> **If you've ever thought "I wish my AI could just open a browser and do this for me" — this is for you.**
> **如果你曾经想过"要是 AI 能帮我打开浏览器把这事干了就好了"——就是你。**

---

## Our Products · 产品

**Two products. One engine. Each independently starred.**
**两个产品。一套引擎。各自独立打星。**

| Product · 产品 | What it does · 做什么 | ⭐ |
|------|------|:--:|
| 🛒 [**AliExpress Auto Listing**](https://github.com/xiaoyaotsyx-dotcom/aliexpress-listing)<br>🛒 [**速卖通自动上架**](https://github.com/xiaoyaotsyx-dotcom/aliexpress-listing) | 1688 采集 → 店小秘 ERP 填表 → 速卖通发布，全自动<br>1688 sourcing → Dianxiaomi ERP → AliExpress, fully automated | [⭐](https://github.com/xiaoyaotsyx-dotcom/aliexpress-listing) |
| 🧠 [**Auto Experts**](https://github.com/xiaoyaotsyx-dotcom/auto-experts)<br>🧠 [**自动化 AI 专家**](https://github.com/xiaoyaotsyx-dotcom/auto-experts) | Not just another chatbot — a persona-driven AI expert that controls your browser, fills forms, analyzes data, and works while you sleep.<br>不只是聊天——人设驱动的 AI 专家，操控你的浏览器、填表单、分析数据、在你睡觉时替你干活 | [⭐](https://github.com/xiaoyaotsyx-dotcom/auto-experts) |

---

## How It Works · 怎么做到的？

```
  你的 AI 助手                     Rigi 技能包                    真实世界
  Your AI Assistant              Rigi Skill Pack               Real World

  ┌──────────┐              ┌────────────────────┐         ┌──────────────┐
  │ "上品"    │──────────────▶│ CDP 连接            │─────────▶│ 店小秘 ERP    │
  │ "分析特斯拉"│              │ 浏览器操控           │         │ X / 微博     │
  │ "审计这个站"│              │ 社媒发布             │         │ 知乎 / 小红书 │
  └──────────┘              └────────────────────┘         └──────────────┘
                                    │
                            ┌───────▼───────┐
                            │ 你的 Chrome    │
                            │ 你的登录态     │
                            │ 数据不出电脑   │
                            └───────────────┘
```

> 🔒 **Privacy-first.** Nothing uploaded to any cloud. Your browser. Your logins. Your data.
> 🔒 **隐私优先。** 不上传任何云端。你的浏览器。你的登录态。你的数据。

---

## What's in This Repo? · 本仓库有什么？

| Directory · 目录 | Purpose · 用途 |
|------|------|
| `shared/` | Infrastructure shared by all products — CDP connection, browser automation, social media posting.<br>所有产品共用的基础设施：CDP 连接、浏览器操控、社媒发布 |
| `tools/` | Utility tools — data verification, etc.<br>辅助工具：数据验证等 |

> 💡 Both product repos reference `shared/` from here. You don't need to clone this repo to use the products — go directly to each product repo.
> 💡 两个产品仓库各自引用这里的 `shared/` 基础设施。不需要克隆这个仓库——直接去各自仓库下载即可。

---

## Quick Start · 上手（30 秒）

```bash
# Pick a product, tell your AI to load it. · 选一个产品，告诉你的 AI 加载它。

# AliExpress auto listing · 速卖通上架
"Load aliexpress-listing skill. Let's publish a product."
"加载 aliexpress-listing skill。上品。"

# Stock analysis · 股票分析
"Load investment-research skill. Analyze Tesla."
"加载 investment-research skill。分析特斯拉。"
```

> 💡 支持 **Hermes Agent / Claude Code / Cursor**——能跑 Python + 操控浏览器的 AI 助手都行。

---

## License · 许可

| Use Case · 使用场景 | License · 许可 |
|------|---------|
| Personal, non-commercial · 个人非商业 | AGPLv3 ✅ Free · 免费 |
| Any company or commercial use · 任何企业/商用 | [Contact us · 联系我们](mailto:Walter.x@qq.com) |

---

## Contact · 联系

- 📕 RedNote · 小红书: [@瑞吉AI人民公社](https://www.xiaohongshu.com/user/profile/42084313799)
- 📧 Walter.x@qq.com

---

<p align="center">
  <sub>Reject AI echo chambers. Democratize practical AI for everyone.</sub><br>
  <sub>拒绝 AI 小圈子自嗨，推动实用 AI 全面普惠。</sub>
</p>
