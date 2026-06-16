
<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="https://placehold.co/800x200/0d1117/ffffff?text=Rigi+AI+Commons&font=Montserrat">
    <img alt="Rigi AI Commons" src="https://placehold.co/800x200/ffffff/5b5fe7?text=Rigi+AI+Commons&font=Montserrat" width="600">
  </picture>
</p>

<p align="center">
  <strong>AI 助手 × 专家人设 × 浏览器操控 = 能聊更能干的 AI 专家队友</strong><br>
  Plug-and-play AI expert skills. Each expert can browse, search, fill forms, and post — autonomously.
</p>

<p align="center">
  <a href="https://github.com/xiaoyaotsyx-dotcom/Rigi-AI-Commons/blob/main/LICENSE"><img src="https://img.shields.io/badge/license-AGPLv3-blue.svg" alt="License"></a>
  <a href="#"><img src="https://img.shields.io/github/stars/xiaoyaotsyx-dotcom/Rigi-AI-Commons" alt="Stars"></a>
  <a href="#"><img src="https://img.shields.io/github/last-commit/xiaoyaotsyx-dotcom/Rigi-AI-Commons" alt="Last Commit"></a>
</p>

---

## 🤔 这不是又一套 Prompt 合集

市面上有很多 "AI Prompt 大全"。它们是**话术**——告诉 AI 怎么说。

Rigi 做的是 **自动化专家**——给 AI 配人设 + 配浏览器操控能力，让它**真的动手干活**：

> "分析这只股票" → 开浏览器抓三源数据 → 交叉验证 → 出研报  
> "上架这个产品" → 1688 采集 → 店小秘填表 → 速卖通发布  
> "审这个网站" → 扒前端源码 → 扫敏感信息 → 出安全报告

---

## 🏗 架构

```
┌────────────────────────────────────────┐
│           Rigi Expert System            │
│                                        │
│  👤 投研专家    🛒 电商专家   🔒 安全专家  │  ← 5 个专家
│  🕵️ 竞品专家    📝 (自定义专家)         │
│                                        │
│  ────────────────────────────────────  │
│                                        │
│  🔌 CDP 连接    🌐 浏览器操控  📱 社媒发布 │  ← 基础设施
│                                        │
└────────────────────────────────────────┘
```

---

## ⚡ Quick Start

```bash
# 选择一个专家，告诉你的 AI 助手加载它

# 投研分析
"加载 investment-research skill，分析特斯拉"

# 安全审计
"加载 security-audit skill，看看这个网站有没有信息泄露"

# 速卖通上架
"加载 aliexpress-listing skill，上品"
```

> 💡 支持 **Hermes Agent / Claude Code / OpenAI Codex / Cursor**。任何能跑 Python + 操控浏览器的 AI 助手。

---

## 📦 产品

### 🧠 专家（experts/）

| 专家 | 能做什么 | 状态 |
|------|---------|------|
| 📈 **投研分析** | 三源数据→交叉验证→五 Skill 并行→四份研报 | ✅ v1.0 |
| 🛒 **速卖通上架** | 1688采集→店小秘ERP填表→速卖通待发布 | ✅ v1.0 |
| 🔒 **安全审计** | 扒前端源码→扫 API key/手机号/云 ID→出报告 | ✅ v1.0 |
| 🕵️ **竞品分析** | 逆向AI产品→提取提示词架构→分析工作流设计 | ✅ v1.0 |
| 🎨 **专家模板** | 3 分钟创建一个新专家（内嵌 CDP 能力） | ✅ 框架 |

### ⚙️ 基础设施（core/）

| 模块 | 说明 |
|------|------|
| 🔌 **CDP 快速连接** | Chrome 调试模式三步诊断连接 |
| 🌐 **通用浏览器操控** | 导航/点击/填表/上传/抓取/反检测 |
| 📱 **社媒全自动发布** | X/微博/知乎/头条/小红书/Facebook 六平台 |

### 🔧 工具（tools/）

| 工具 | 说明 |
|------|------|
| 📊 **数据验证** | 股价 PE 验算 + 三源交叉验证 |

---

## 🌐 支持的 AI 平台

| 平台 | 状态 |
|------|------|
| Hermes Agent | ✅ 主力平台 |
| Claude Code | ✅ |
| Cursor | ✅ |
| OpenAI Codex | ✅ |

---

## 📄 License

**AGPLv3 双许可**

| 使用场景 | License |
|----------|---------|
| 个人非商业 | AGPLv3 ✅ 免费 |
| 企业 ≤5 人 | AGPLv3 ✅ 免费 |
| 企业 6+ 人 / 商用 | 商业授权 required |

---

## 📬 Contact

- 📕 小红书: [@瑞吉AI人民公社](https://www.xiaohongshu.com/user/profile/42084313799)
- 📧 Email: xiaoyao@aibook.online

---

<p align="center">
  <sub>Reject AI echo chambers. Democratize practical AI for everyone.</sub><br>
  <sub>拒绝 AI 小圈子自嗨，推动实用 AI 全面普惠。</sub>
</p>
