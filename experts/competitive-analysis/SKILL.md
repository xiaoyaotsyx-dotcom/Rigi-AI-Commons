---
name: competitive-ai-product-analysis
description: Reverse-engineer a competitor's AI product to extract prompt architectures, workflow designs, and system design patterns — using CDP interception, Vue store access, static JS analysis, and browser-based API capture.
category: research
---

# Competitive AI Product Analysis

Systematically analyze a competitor's AI product to understand their prompt engineering, system architecture, and UX patterns. Used for competitive research and inspiration — to learn what works, not to copy directly.

## What You CAN Extract

### 1. Frontend JS — static analysis

Download all JS bundles (curl the main .js files) and grep for:

API endpoints - TWO patterns depending on framework:
- React/Vite (relative paths): grep -oP '/api/[a-zA-Z0-9_/_-]+' bundle.js | sort -u
- Vue/Next.js (full URLs): use the https:// pattern below

Single-line minified bundles: split with tr before grepping:
  cat bundle.js | tr ',' '\n' | grep -iE '(auth|login|payment|model)' | head -30

Also scan for: route definitions, feature flags, model names, prompt categories, third-party services, hardcoded credentials. See references/server-side-static-recon.md for the full server-side workflow.

### 2. API Endpoint Map — via CDP request interception

Connect to the user's logged-in Chrome via CDP relay, then intercept `fetch()`:
```javascript
const origFetch = window.fetch;
window.fetch = function(...args) {
  return origFetch.apply(this, args).then(async resp => {
    const clone = resp.clone();
    if (typeof args[0] === 'string' && args[0].includes('/api/')) {
      const body = await clone.text();
      window.__CAPTURED.push({url: args[0], method: options.method, reqBody: options.body, respBody: body});
    }
    return resp;
  });
};
```

Key endpoints to identify:
- `/api/v1/expand/chapter` — AI expansion
- `/api/v1/shortcuts/collection?type=N` — prompt library per category
- `/api/v1/script-tools/workflows` — workflow templates
- `/api/v1/model-state/latest` — active AI model selection
- `/api/v1/glossary` — character/world-building context injection
- `/api/v1/characters` — character profiles fed as context

### 3. Vue Store Data — read decoded state directly

When API responses are encrypted but the app decodes them into Vue/Pinia stores, read the store directly:
```javascript
const pinia = document.querySelector('#app').__vue_app__.config.globalProperties.$pinia;
const store = pinia._s.get('shortcut-store');  // or any store name
const decodedData = store.$state.list;  // already decrypted!
```

**Store names can be discovered:**
```javascript
Array.from(pinia._s.keys())  // → ['app-store', 'try-url-store', 'auth-store', 'user-store', ...]
```

### 4. Page Structure & UX Flow

Navigate the site and screenshot key pages:
- Writing/creation interface: what parameters does the user see?
- Prompt selector: what categories, how are prompts organized?
- AI model selector: what models are available?
- Context injection fields: characters, glossary, plot notes, style presets

### 5. Product Ecosystem Extraction

AI product JS bundles often expose the full product matrix. Key patterns to grep:

```bash
# Firebase auth domains → reveals staging/test/preview environments
grep -oP 'authDomain\s*:\s*"[^"]+"' bundle.js

# AI model names
grep -oP '(?:openai|claude|gemini|deepseek|gpt-|sonnet|haiku|flash)[^"\x27\s]{0,40}' bundle.js -i

# Domain lists in multi-lang strings
grep -oP 'https?://[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}' bundle.js | sort -u

# Supabase URL (common in SvelteKit/Next.js apps)
grep -oP 'https?://[a-z]{20}\.supabase\.co' bundle.js
grep -oP 'eyJ[A-Za-z0-9_\-]{50,}\.[A-Za-z0-9_\-]{20,}\.[A-Za-z0-9_\-]{10,}' bundle.js

# WeChat OAuth AppID
grep -oP 'open\.weixin\.qq\.com/connect/qrconnect\?appid=[a-zA-Z0-9]+' bundle.js

# R2 / Cloudflare storage
grep -oP 'r2\.[a-zA-Z0-9.-]+' bundle.js

# Competitor importers (reveals product positioning)
grep -oP '(?:celtx|scrivener|final.draft|writerduet|fade.?in|arc.studio)[^"\x27\s]{0,30}' bundle.js -i
```

**Laper.ai case study (2026-06-09):** SvelteKit + Supabase stack. 1.98MB Vite bundle revealed: Supabase URL + anon key, WeChat OAuth AppID (`wxf0378082c1a0c2d3`), R2 bucket (`r2.laper.ai`), Stripe integration, competitor importers for Celtx/Scrivener/Final Draft/WriterDuet. Model selection (Claude + DeepSeek) was server-side — only `anthropic.com/claude-code` appeared in bundle. See `references/case-study-laer-ai-2026-06-09.md` in `frontend-security-audit`.

**Fengyue.ai case study:** The 5.2MB main bundle revealed:
- 4 Firebase auth domains (testaf.aiero.cc, staging.aiero.cc, preview.aiero.cc, aiporn.tw)
- 6+ related product domains (aifun.sale, catai.wiki, thegena.xyz, usegena.xyz, aify.pages.dev)
- AI model matrix: OpenAI, DeepSeek, Gemini, Claude
- DMM Japan integration (accounts.dmm.co.jp, api.chara-chat.dmm.co.jp)
- Telegram WebApp full integration
- BYOK model — users bring their own API keys; no hardcoded credentials

### 6. Payment Infrastructure Analysis

**The payUrl tells everything.** Register a disposable test account (see Phase 1.5), create a real order (don't pay), and analyze the returned `payUrl` to identify the actual payment gateway. The `provider` field in the order response names the backend, and the payUrl domain reveals whether it's direct Alipay, a payment aggregator, or a fourth-party personal payment platform.

Full workflow and diagnostic table: `references/payment-infrastructure-analysis.md`

Key intel from payment analysis:
- Is the company a legitimate business (direct merchant) or individual (personal payment)?
- Which payment channels are active (Alipay only, or WeChat too)?
- What signature algorithm — MD5 is a red flag for security maturity
- Does the merchant ID timestamp reveal when they started monetizing?

### 7. Guest/Trial Mode Discovery

Navigate the SPA as an unauthenticated user to discover free access paths:
- Check for "Guest", "游客", "匿名" labels
- Try clicking category tabs ("纯聊", "免费", "热门") 
- Navigate directly to `/explore/installed/{uuid}` URLs
- Check if chat textarea appears without login prompt
- Note the model selector (reveals which AI models are available)

Even if guest can't chat, the model names and guest point balance are valuable competitive intel.

### Supabase anon key extraction (SvelteKit pattern)

SvelteKit apps using Supabase often hardcode URL + anon key in the JS bundle as `i$="https://xxx.supabase.co",a$="eyJ..."`. This is a Vite minification artifact — grep for the Supabase project ref (random 22-char string before `.supabase.co`) to locate it. The anon key alone rarely leaks data (RLS usually blocks it), but it reveals the project ID, database provider, and allows probing for misconfigured RLS policies.

**Laper.ai case study:** Anon key exposed, RLS locked tight — all tables returned 401 "Auth required". Severity downgraded from 🔴 to 🟡. Full case study in `frontend-security-audit/references/case-study-laper-ai-2026-06-09.md`.
If the backend uses middleware-level encryption (e.g., Laravel/Hyperf `CryptoMiddleware`), both request AND response bodies are AES-encrypted. Even with CDP intercepting `fetch()`, you only get `{"data":"base64-ciphertext..."}`. The decryption key is either:
- In the JS bundle (searchable — `grep 'encrypt\|decrypt\|CryptoJS\|AES\|app_key' bundle.js`)
- Server-side only (not extractable)

### Backend prompt assembly
The actual system prompt is assembled server-side from multiple sources (prompt template + user context + characters + glossary + writing style). Even if you get the prompt template from the shortcuts API, the full assembled prompt is never sent to the frontend.

### Model parameters
Temperature, top_p, max_tokens, and other generation parameters are server-side. You can infer them from output quality but can't extract exact values.

## Workflow

### Phase 1: Passive Recon (no login needed)
1. `curl` the main page, extract all JS bundle URLs
2. Download all bundles, scan for API endpoints, routes, secrets
3. Check for public config files: `robots.txt`, `server_urls.json`, `hosts-prod.json`, `manifest.json`
4. Map the tech stack (Vue/React, backend framework, CDN, captcha, analytics)
5. Probe public API endpoints systematically — config, payment packages, auth/me — to map product architecture (see `references/server-side-static-recon.md` for full curl-based recon workflow)
6. DNS/WHOIS/hosting analysis: dig + whois + ipapi for infrastructure intel
7. **JS bundle Chinese feature extraction** — for Chinese-language products, use unicode-aware grep:
   ```bash
   # Routes (Vue Router)
   grep -oP 'path:"[^"]*"' bundle.js | sort -u
   # Chinese feature names
   grep -oP '"[^"]{3,30}(生成|写作|扩写|续写|拆书|润色|提示词|工作流|风格)[^"]{0,30}"' bundle.js | sort -u
   # Pricing
   grep -oP '"[^"]{3,60}"' bundle.js | grep -iE '(会员|VIP|套餐|价格|白银|黄金|钻石)' | sort -u
   ```
   See `references/spa-deep-scraping.md` for full grep recipe kit.

### Phase 1.5: Disposable Account Recon (no user account needed)

For sites with email-based registration, create a disposable account to access authenticated endpoints — no need for the user to have an existing account.

1. Create temp email via **mail.tm API** (free, no phone, Python-native): see `references/payment-infrastructure-analysis.md` for full recipe
2. Request verification code via `/api/auth/send-code` with `scene="register"`
3. Poll mail.tm inbox, extract 4-6 digit code with regex
4. Register via `/api/auth/verify-code` — capture the auth cookie from response headers
5. With the auth cookie, probe authenticated endpoints: wallet, payment packages, order creation

**Critical for payment analysis**: create a real order via `/api/payment/create` to extract the `payUrl` and `provider` fields — this reveals the actual payment infrastructure that's invisible in passive recon.

**Pitfall**: Some sites require phone verification or CAPTCHA before sending email codes. If `/api/auth/send-code` returns an error about phone or captcha, the temp email approach won't work — fall back to Phase 2 (user's own account via CDP).

### Phase 2: Authenticated Recon (user must log in)

1. User starts Chrome with `--remote-debugging-port=9222 --user-data-dir="D:\chrome-debug-profile"`
2. User starts TCP relay: `python D:\relay2.py`
3. Connect via Playwright: `chromium.connect_over_cdp(WS_URL)`
4. User navigates to the target site and logs in
5. Find the logged-in tab: filter pages by URL
6. Install `fetch()` interceptor
7. Navigate to all key pages — the interceptor captures every API call
8. Read Vue/Pinia stores for decoded data
9. Have the user trigger one AI generation while intercepting

### Phase 2b: SPA Content Scraping (fallback when API is encrypted)

When API responses are fully encrypted (CryptoJS AES middleware), skip network interception entirely. Instead, **scrape the rendered DOM** — extract every page the logged-in user can see.

1. Navigate the SPA section by section using text-content-based click fallback (see `references/spa-deep-scraping.md`)
2. Extract `document.body.innerText` from each page
3. Map: navigation structure → feature inventory → prompt categories → pricing → creator ecosystem
4. This yields a complete product architecture map without touching any API

This approach captured xingyuexiezuo.com's full 10-module product structure, 30+ prompt categories, 15+ workflow templates, and 4-tier pricing — all through rendered DOM despite complete API encryption.

### Phase 3: Analysis
1. Map the prompt architecture from what was captured
2. Identify the category/type system (each prompt type has an ID)
3. Map the workflow system (if any)
4. Extract the writing style templates
5. Analyze the context injection pattern (characters, glossary, plot notes, style presets)
6. Document the UI/UX decisions (parameter layout, progress indicators, cost display)

## Real-World Finding Patterns

### API Response Encryption
Many Chinese AI writing platforms use **Hyperf (PHP) + CryptoMiddleware**:
- All responses wrapped in `{"code":200,"data":{"encoded":"base64..."}}`
- Decryption happens in the JS bundle using CryptoJS AES
- The middleware error traces expose the server path structure: `App\Controller\V1\...`, `App\Middleware\CryptoMiddleware`

### Prompt System Architecture (星月写作 example)
```
Categories (type parameter):
  type=1  → AI写作/扩写
  type=26 → 剧本工具
  
Prompt sources:
  /api/v1/my/shortcuts/collection?type=N    → user's prompts
  /api/v1/shortcuts/recommended?type=N      → system recommended
  /api/v1/shortcuts/tag/page?type=N&tag=1   → by tag

Context injection:
  /api/v1/characters?id=BOOK_ID             → character profiles
  /api/v1/glossary?id=BOOK_ID               → world-building terms
  /api/v1/book/memos?id=BOOK_ID             → plot memos/notes
  /api/v1/chapter/summary?id=CHAPTER_ID     → chapter summaries (for continuity)

Model selection:
  /api/v1/model-state/latest               → available and active models
```

### Infrastructure Exposure Pattern
See `references/infrastructure-exposure-patterns.md` for examples of what competitors expose in their frontend JS (DeepSider case study).

### Supabase Exposure Pattern (SvelteKit/Vite)

SvelteKit apps using Supabase often inline credentials in the client bundle. The pattern is:

```javascript
// Look for this in minified SvelteKit/Vite bundles:
i$="https://{ref}.supabase.co",a$="eyJ..."
```

The `i$` variable is `supabaseUrl`, `a$` is `supabaseKey` (anon key). This is a code-hygiene issue — Supabase recommends env vars, but SvelteKit SSR inlines `$env/static/public` values into the client bundle.

**What to extract:**
1. Supabase project URL → reveals `{ref}.supabase.co`  
2. Anon key JWT → decode with `jwt.decode()` to get project ref, role, issued/expiry
3. Probe REST API with `apikey` header ONLY (NOT `Authorization: Bearer` — anon key is not a JWT)

**Key diagnostic:**
- `rest/v1/{table}?select=*&limit=1` with `apikey: <anon_key>` → response tells you everything
  - `401 "Auth required"` = RLS properly configured, data safe (🟡 cosmetic leak)
  - `200 + data rows` = RLS not configured, data exposed (🔴 actual leak)
- `auth/v1/signup` with `apikey: <anon_key>` → `"Invalid API key"` = signups disabled (normal)
- Note: auth endpoints are MORE restrictive than REST endpoints — anon key is valid for REST but rejected by auth endpoints when signup is disabled

## Pitfalls

1. **Don't use headless browser for authenticated recon** — it creates a separate session. ALWAYS connect to the user's desktop Chrome via CDP.
2. **Verify CDP is running before trying to connect**: `curl http://GATEWAY_IP:19223/json/version`
3. **Vue store data may be a Proxy** — `JSON.stringify()` may fail. Access specific fields instead.
4. **SPA navigation clears fetch hooks** — re-install hooks after each page navigation.
5. **`JSON.stringify()` on large objects may hang** — use targeted field access.
6. **The user's login session expires** — if APIs start returning auth errors, the token in `localStorage.SECRET_TOKEN` may have expired. User needs to re-login.
7. **CryptoMiddleware defeats full prompt extraction** — this is a fundamental limitation. Accept it and focus on what IS extractable: prompt categories, context injection patterns, UI parameter design. Fall back to SPA content scraping (Phase 2b) for product architecture analysis.
8. **SPA navigation: page.url doesn't track hash routes** — Vue `#/route` won't show in `page.url`. Don't use URL to identify which section you're on; use extracted text content instead.
9. **Vue SPA click fallback is fragile** — `querySelector('[class*="sidebar"] span')` may miss nav items if the component tree is deep. Always include the `querySelectorAll('*')` fallback pattern (see `references/spa-deep-scraping.md`).
10. **SvelteKit apps have no `__vue_app__` or React devtools** — Vue store access (`__vue_app__.config.globalProperties.$pinia`) won't work on SvelteKit sites. SvelteKit uses file-based routing (routes embedded in compiled components, not `path:` patterns) and SSR environment variables (`$env/static/public`) that get inlined into the client bundle. For SvelteKit sites, focus on static JS analysis — Supabase credentials and API endpoints are typically fully exposed in the bundle. If you need runtime state, Svelte stores are attached to the DOM via `__svelte` internals, but extraction is fragile and bundle-dependent. See `frontend-security-audit` references for Laper.ai SvelteKit case study.
11. **React minified bundles need `tr ',' '\n'` before grepping** — a 2MB single-line JS file defeats line-based grep. Split on commas first to reveal keywords buried mid-line. Without this, `grep -i 'auth'` returns zero results even though the bundle has 20+ auth-related function calls.
12. **react.profiler false positives** — broad patterns like `grep -i 'auth'` will match React internal symbols (e.g. `react.profiler` containing 'auth' substring). Use more specific patterns: `grep -iE '(auth/login|auth/me|auth/send)'`.
13. **React/Vite bundles use relative API paths** — they won't match full-URL grep patterns. Use `grep -oP '/api/[a-zA-Z0-9_/_-]+'` instead of `https?://.../api/...`.
14. **Payment order creation is safe — you don't need to pay** — the `/api/payment/create` endpoint returns the `payUrl` and full order details immediately. You don't need to follow the payUrl or complete payment. The order will sit in `pending` status and can be ignored. The `provider` field and decoded `payUrl` parameters are the intel you want.
15. **mail.tm inbox has ~3-5 second delivery delay** — after requesting a verification code, wait before polling. The first poll may return empty. Also: mail.tm accounts auto-expire after inactivity; create a fresh one per session.
16. **Some sites use phone-only registration** — if `/api/auth/send-code` returns "请输入有效手机号" or similar, temp email won't work. Check the JS bundle first: `grep -E '(email|phone|短信|邮箱)' bundle.js` to confirm the auth method before attempting.

## Related Skills
- `chrome-cdp-relay` — CDP connection setup
- `frontend-security-audit` — companion skill for security findings during recon
