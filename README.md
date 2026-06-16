
<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="https://placehold.co/800x200/0d1117/ffffff?text=Rigi+AI+Commons&font=Montserrat">
    <img alt="Rigi AI Commons" src="https://placehold.co/800x200/ffffff/5b5fe7?text=Rigi+AI+Commons&font=Montserrat" width="600">
  </picture>
</p>

<p align="center">
  <strong>Turn your AI assistant into an expert teammate.</strong><br>
  Plug-and-play agent skills for real business. No coding. No hype.
</p>

<p align="center">
  <a href="https://github.com/xiaoyaotsyx-dotcom/Rigi-AI-Commons/blob/main/LICENSE"><img src="https://img.shields.io/badge/license-AGPLv3-blue.svg" alt="License"></a>
  <a href="#"><img src="https://img.shields.io/github/stars/xiaoyaotsyx-dotcom/Rigi-AI-Commons" alt="Stars"></a>
  <a href="#"><img src="https://img.shields.io/github/last-commit/xiaoyaotsyx-dotcom/Rigi-AI-Commons" alt="Last Commit"></a>
  <a href="https://xiaoyaotsyx-dotcom.github.io/Rigi-AI-Commons/"><img src="https://img.shields.io/badge/docs-📖_Read-5b5fe7" alt="Docs"></a>
  <br>
  <sub>中文 ｜ English</sub>
</p>

---

<p align="center">
  <strong>AI 助手即专家队友。</strong><br>
  即装即用的 AI Agent 技能包，不需编程，没有噱头。<br>
  数据不出你的电脑，工作流实战验证。
</p>

---

## 🤔 What Problem Does This Solve?

You have an AI assistant (Hermes / Claude Code / Cursor). It can answer questions — but you want it to **do real work**:

- "Translate these 200 AliExpress listings into English — but format them for the ERP too."
- "Pull 1688 product data, fill in Dianxiaomi, and publish to AliExpress."  
- "Analyze this stock and write a research report."
- "Check this legal document for risks."

That's not a chatbot. That's an **agent with a workflow**.

**Rigi AI Commons gives your AI assistant those workflows** — battle-tested, step-by-step, ready to run.

---

## 🧩 这解决什么问题？

你的 AI 助手会聊天，但你想要它**真的干活**：

- 「把 200 个速卖通标题翻译成英文，顺便填好 ERP 格式」
- 「1688 采集 → 店小秘填表 → 速卖通上架」
- 「分析这只股票，写研报」
- 「审这份合同，标出风险点」

聊天机器人做不了这些。**有工作流的 Agent 才行。**

Rigi AI Commons 就是给 AI 助手配**工作流**——经过实战、一步步跑通、拿来就用。

---

## ⚡ Quick Start

```bash
# No install. No pip. No npm.
# Just download the skill file and tell your AI assistant to load it.

# Example: AliExpress Listing Automation
# 1. Download the folder
# 2. Tell your Hermes/Claude: "Load aliexpress-listing-automation skill"
# 3. Say: "上品" (list product)
```

> 💡 Works with **Hermes Agent**, **Claude Code**, **OpenAI Codex**, **Cursor**, **Open Interpreter**. Any AI that can run Python + control a browser via CDP.

---

## 📦 Products

| Skill | What It Does | Platform | Status |
|-------|-------------|----------|--------|
| 🛒 **AliExpress Listing Automation** | 1688采集 → 店小秘ERP填表 → 速卖通待发布 | AliExpress | ✅ v1.0 |
| 📈 Stock Research | Multi-source data → Cross-validation → Research report | A-Share/US/HK | 🚧 Coming |
| ⚖️ Legal Review | Document scanning → Risk flagging → Compliance check | PRC Law | 🚧 Coming |

---

## 🏗 Architecture

```
┌─────────────────────────────────────┐
│          Your AI Assistant           │
│  (Hermes / Claude Code / Cursor)     │
└──────────────┬──────────────────────┘
               │  "Load skill: aliexpress-listing"
               ▼
┌─────────────────────────────────────┐
│         Rigi AI Commons Skill        │
│  ┌──────────┐  ┌──────────┐        │
│  │ Workflow │  │  Prompts │        │
│  │  Steps   │  │  Rules   │        │
│  └──────────┘  └──────────┘        │
│  ┌──────────┐  ┌──────────┐        │
│  │  Python  │  │  CDP     │        │
│  │  Scripts │  │ Browser  │        │
│  └──────────┘  └──────────┘        │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│          Your Computer               │
│  Chrome (your login) + Your Data    │
│  Nothing leaves your machine        │
└─────────────────────────────────────┘
```

> 🔒 **Privacy-first.** Your browser, your logins, your data. Nothing uploaded to any cloud.

---

## 📖 Documentation

- [📘 Full Documentation](https://xiaoyaotsyx-dotcom.github.io/Rigi-AI-Commons/)
- [🚀 Getting Started Guide](https://xiaoyaotsyx-dotcom.github.io/Rigi-AI-Commons/getting-started)
- [📦 All Skills Catalog](https://xiaoyaotsyx-dotcom.github.io/Rigi-AI-Commons/skills)

---

## 🌐 Supported Platforms

| Platform | Status | Notes |
|----------|--------|-------|
| Hermes Agent | ✅ Full | Primary target. CDP + Python. |
| Claude Code | ✅ Compatible | Load as skill file. |
| Cursor | ✅ Compatible | Load as `.cursorrules` skill. |
| OpenAI Codex | ✅ Compatible | GitHub-native. |
| Open Interpreter | ✅ Compatible | Python execution. |

---

## 📄 License

**Dual License — AGPLv3 + Commercial**

- 🏠 **Personal, non-commercial use:** Free. Forever. [AGPLv3](LICENSE)
- 🏢 **Business commercial use:** [Commercial license required](mailto:xiaoyao@aibook.online)

| Use Case | License |
|----------|---------|
| Individual learning / personal projects | AGPLv3 ✅ Free |
| Company internal use (≤5 seats) | AGPLv3 ✅ Free |
| Company internal use (6+ seats) | Commercial required |
| Building paid services / SaaS | Commercial required |
| Embedding in commercial products | Commercial required |

---

## 🤝 Contributing

We welcome contributions! See [CONTRIBUTING.md](CONTRIBUTING.md).

> 🎯 **Good First Issues** tagged for newcomers.

---

## 📬 Contact

- 📕 RedNote (小红书): [@瑞吉AI人民公社](https://www.xiaohongshu.com/user/profile/42084313799)
- 🐦 X/Twitter: [@xiaoy17305382](https://x.com/xiaoy17305382)
- 📧 Email: xiaoyao@aibook.online

---

<p align="center">
  <sub>Reject AI echo chambers. Democratize practical AI for everyone.</sub><br>
  <sub>拒绝 AI 小圈子自嗨，推动实用 AI 全面普惠。</sub>
</p>
