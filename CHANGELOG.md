# Changelog

## [1.1.0] — 2026-06-16

### Added
- 🧠 **Rigi Expert System** — 专家人设 × CDP 浏览器操控架构
  - `experts/expert-panel-template/` — 3 分钟创建自动化专家（内嵌 CDP 能力）
  - 📈 `experts/investment-research/` — 投研分析专家（三源数据→交叉验证→四份研报）
  - 🔒 `experts/security-audit/` — 前端安全审计专家（扫 API key/手机号/云 ID）
  - 🕵️ `experts/competitive-analysis/` — 竞品 AI 产品逆向分析专家
- ⚙️ **基础设施层 `core/`** — 所有专家共享
  - 🔌 `core/cdp-quick-connect/` — CDP 三步标准连接
  - 🌐 `core/hermes-browser-automation/` — 通用浏览器操控
  - 📱 `core/hermes-social-media-automation/` — 六平台社媒发布
- 🔧 `tools/price-data-verification/` — 股价数据验证铁律

### Changed
- 架构从「单个 skill 文件夹」升级为「core/ experts/ tools/」三层
- 每个专家天生具备浏览器操控能力（内嵌 CDP 依赖）
- 旧 `aliexpress-listing-automation/` → `experts/aliexpress-listing/`
- README 更新为专家系统架构图

---

## [1.0.0] — 2026-06-16

### Added
- 🛒 **AliExpress Listing Automation v1.0** — 1688采集→店小秘ERP填表→速卖通待发布
- README, CONTRIBUTING, SECURITY, LICENSE
- AGPLv3 dual-license structure
