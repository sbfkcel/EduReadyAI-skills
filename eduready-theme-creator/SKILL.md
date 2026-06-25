---
name: eduready-theme-creator
description: >
  Create custom HTML themes for EduReady campaign landing pages and buy/checkout pages.
  Use this skill when the user wants to create a campaign theme, design a landing page for
  an educational product, build a custom checkout flow, or create HTML templates that integrate
  with the EduReady platform. Triggers on requests like "create a theme", "design a campaign page",
  "build a buy page", "make a landing page for our course", or any mention of EduReady themes/templates.
---

# EduReady Theme Creator

Generate custom HTML themes for the EduReady educational platform's campaign and buy pages.

## Quick Start

A theme is a directory with HTML files that render inside iframes on the EduReady platform:

```
my-theme/
  ├── index.html      ← Campaign landing page (required)
  ├── buy.html        ← Checkout page (optional)
  ├── style.css       ← Shared styles
  └── assets/         ← Images, fonts, etc.
```

## Reference Files

This skill bundles reference files in the `references/` directory:

- **`references/campaign-theme-guide.md`** — Complete development guide
  - Sections 4.1-4.7: Campaign page development
  - Sections 5.1-5.12: Buy page development (5.12 = reusing campaign common info)
  - Section 6: SDK API reference
  - Section 8: CSS styling reference

- **`references/index.html`** — Example campaign page
- **`references/buy.html`** — Example checkout page
- **`references/style.css`** — Example responsive styles

Read the guide for complete details. Use the example files as templates when creating new themes.

## Campaign Page (index.html)

> ### ⚠️ 头号铁律：避免活动页卡在"加载中…"
> 这是活动主题最常见、也最隐蔽的 bug。**用户从购买页点浏览器后退回到活动页，会永远停在"加载中…"。**
> 它只在**真实用户**身上出现——开发时若开了 DevTools 的 "Disable cache" 会顺带禁用 bfcache，
> 反而看不到问题，极易误判为"已修好"。务必关掉 Disable cache 用浏览器后退键验证。
>
> 两条必须同时满足，缺一即复现：
> 1. **请求数据要带重试 + bfcache 重发**——不能只 `requestData()` 一次。活动页常从 bfcache 整页冻结恢复，
>    父页面 effect 不再执行、不主动推数据，且 iframe 常先于父页面初始化；一次性请求一旦丢失就死等。
>    必须循环重试直到拿到数据，并在 `pageshow`（`e.persisted`）时重新请求。
> 2. **购买跳转用软导航**——`goToBuy` 必须发 `eduready:campaign:navigate` 让父页面 SPA 内 `router.push`，
>    **不要** `win.location.href` 顶层硬跳转（硬跳转后再后退，父页面从 bfcache 恢复、effect 不重跑 → 卡死）。
>
> 下面 Required Structure 的握手代码即正确范式；`references/index.html` 也是正确范式，照抄即可。

### Required Structure

```html
<!DOCTYPE html>
<html lang="zh">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>活动标题</title>
  <link rel="stylesheet" href="style.css">
</head>
<body>
  <!-- Hero section with countdown -->
  <!-- Product grid -->
  <!-- Rules section (optional) -->

  <script>
    document.write('<script src="' + (
      (location.hostname === '127.0.0.1' || location.hostname === 'localhost')
        ? 'https://eduready.ai'
        : ''
    ) + '/api/campaign-files/sdk/eduready-campaign.js?v=3"><\/script>');
  </script>
  <script>
    var _gotData = false, _reqTries = 0;

    // Request data from parent. The parent listener may not be ready yet (first
    // render data not in, or iframe re-inits before parent on back/bfcache restore),
    // so RETRY until data arrives — a one-shot request gets stuck on "加载中…".
    function requestData() {
      if (_gotData) return;
      try { window.parent.postMessage({ type: 'eduready:campaign:requestData' }, '*'); } catch (e) {}
      if (++_reqTries < 40) setTimeout(requestData, 300); // ~12s cap
    }

    // Listen for response
    window.addEventListener('message', function(e) {
      if (e.data?.type === 'eduready:campaign:data') {
        _gotData = true;
        var campaign = e.data.payload.campaign;
        var products = e.data.payload.products;
        var config = e.data.payload.config || {};
        var isActive = e.data.payload.isActive;
        // Render your page here
      }
    });

    requestData();

    // Re-request on back/forward bfcache restore
    window.addEventListener('pageshow', function(e) {
      if (e.persisted && !_gotData) { _reqTries = 0; requestData(); }
    });
  </script>
</body>
</html>
```

### Data Payload Structure

```javascript
{
  campaign: {
    id: 1,
    name: '活动名称',
    endTime: '2026-02-15T00:00:00Z',
    status: 'active',
    pageConfig: { title, subtitle, buttonText, rules }
  },
  products: [{
    id: 1,
    publicId: '38e54a5b-b10b-4b32-9cf6-feb66eb4fce1',
    name: '课程名称',
    description: '课程描述',
    coverUrl: 'https://...',  // 产品封面图，比例 16:9；为空时可使用平台默认图 /cover.jpg
    basePrice: 9999,
    discountPrice: 7999,
    currency: 'CNY',  // 平台系统配置的默认货币
    soldOut: false
  }],
  config: {
    title: '页面标题',
    subtitle: '副标题',
    bgType: 'gradient',      // 'gradient' | 'image' | 'custom'
    bgValue: 'linear-gradient(...)',
    bgImage: 'https://...',
    buttonText: '立即抢购',
    rules: '活动规则文本'
  },
  isActive: true,
  isPaused: false
}
```

### Buy URL Construction

```javascript
var urlParams = new URLSearchParams(window.location.search);
var campaignSlug = urlParams.get('campaignSlug');
var templateSlug = urlParams.get('template');
var refCode = urlParams.get('ref');

var BASE_URL = (location.hostname === '127.0.0.1' || location.hostname === 'localhost')
  ? 'https://eduready.ai' : '';

// 构建购买链接；productId 优先使用 data.products[i].publicId
function buildBuyUrl(productId) {
  var qs = [];
  if (campaignSlug) qs.push('campaign=' + campaignSlug);
  if (templateSlug) qs.push('template=' + templateSlug);
  if (refCode) qs.push('ref=' + refCode);
  return BASE_URL + '/buy/' + productId + (qs.length ? '?' + qs.join('&') : '');
}

function goToBuy(url) {
  // 请求父页面在其 SPA 内软导航（router.push），避免顶层硬跳转——
  // 硬跳转会让浏览器后退回活动页时卡在“加载中…”（父页面 effect 不再执行）。
  try {
    if (window.parent !== window) {
      window.parent.postMessage({ type: 'eduready:campaign:navigate', url: url }, '*');
      return;
    }
  } catch (e) {}
  // 兜底：非 iframe 或 postMessage 失败时，顶层窗口硬跳转
  var win = window;
  try { while (win !== win.parent) { win = win.parent; } } catch (e) {}
  win.location.href = url;
}
```

### Cover Image Handling

`product.coverUrl` is stored at 16:9 and **may be empty** (not uploaded). Always resolve it
through a helper so empty covers fall back to the platform default and relative paths get the
`BASE_URL` prefix — using a bare `/cover.jpg` breaks in local dev (it resolves to `127.0.0.1`).

```javascript
// 为空→平台默认图 /cover.jpg；绝对 URL 原样；相对路径补 BASE_URL
function resolveCoverUrl(url) {
  if (!url) return BASE_URL + '/cover.jpg';
  if (/^https?:\/\//.test(url)) return url;
  return BASE_URL + url;
}

// var imgHtml = '<img src="' + resolveCoverUrl(p.coverUrl) +
//   '" alt="' + p.name + '" style="aspect-ratio:16/9;object-fit:cover;width:100%">';
```

Show the cover on both the **campaign product grid** and the **buy page Step 1** (the product
has `coverUrl` on the buy flow too — don't leave checkout without a product visual).

## Buy Page (buy.html)

### Required Structure

```html
<!DOCTYPE html>
<html lang="zh">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>购买课程</title>
  <link rel="stylesheet" href="style.css">
  <style>
    .step { display: none; }
    .step.active { display: block; }
  </style>
</head>
<body>
  <div class="buy-page">
    <div class="buy-card">
      <div class="steps" id="steps"></div>
      <div class="step-content" id="step-container" style="flex:1;min-width:0"></div>
    </div>
  </div>

  <script>
    document.write('<script src="' + (
      (location.hostname === '127.0.0.1' || location.hostname === 'localhost')
        ? 'https://eduready.ai'
        : ''
    ) + '/api/buy-template-files/sdk/eduready-buy.js?v=3"><\/script>');
  </script>
  <script>
    EduReady.ready(function(data) {
      // Product & checkout:
      //   data.product, data.pricing, data.formGroups, data.postPayFormGroups
      //   data.refUser, data.coupon, data.previousSubmissions, data.userEmail
      // Campaign common info (same source as the campaign home page — see below):
      //   data.campaign, data.campaignProducts,
      //   data.campaignIsActive, data.campaignIsPaused, data.campaignSlug
      // Build your step wizard here
    });
  </script>
</body>
</html>
```

### SDK Methods

| Method | Returns |
|--------|---------|
| `EduReady.ready(cb)` | Initial data callback |
| `EduReady.verifyCoupon(code, productPublicId)` | `{code, type, value, discountAmount}` |
| `EduReady.checkout({email, couponCode?, formSubmissions?})` | `{url, sessionId, orderId}` |
| `EduReady.stepChange(step, total)` | Notify step change |
| `EduReady.resize(height)` | Adjust iframe height |
| `EduReady.redirectToLogin()` | Redirect to platform login (call when buying while logged out) |

### Campaign Common Info (on the buy page)

When the buy URL carries `?campaign={id|slug}` (it does when reached by clicking buy on a campaign page), `EduReady.ready(data)` also delivers the **same campaign info the campaign home page has** — so `buy.html` can render the campaign theme/subtitle, description, countdown, rules, and a cross-sell grid of other campaign products, without any extra request.

| Field | Description |
|-------|-------------|
| `data.campaign` | Campaign object (`null` if no campaign context). Key fields: `name`, `slug`, `status`, `endTime`, `pageConfig.{title, subtitle, description, rules, buttonText}` |
| `data.campaignProducts` | Other products in the campaign — `{publicId, name, basePrice, discountPrice, coverUrl, currency, soldOut}` for a "more courses" grid |
| `data.campaignIsActive` | Campaign is `active` (discounts apply) |
| `data.campaignIsPaused` | Campaign is paused — show a notice and disable checkout |
| `data.campaignSlug` | Slug for building other products' buy links |

```javascript
EduReady.ready(function(data) {
  var c = data.campaign;
  if (data.campaignIsPaused) showPausedBanner();        // disable checkout
  if (c) {
    var pc = c.pageConfig || {};
    if (pc.title)    setHeroTitle(pc.title);            // theme / 主题
    if (pc.subtitle) setHeroSubtitle(pc.subtitle);      // 副标题
    if (pc.rules)    renderRules(pc.rules);             // 活动规则
    if (c.endTime)   startCountdown(new Date(c.endTime).getTime());  // 倒计时
  }
  (data.campaignProducts || []).forEach(function (p) {  // 活动更多产品
    var url = '/buy/' + p.publicId + '?campaign=' + (c ? (c.slug || c.id) : '');
    renderCrossSellCard(p, url, p.soldOut);
  });
});
```

Rules: use `data.campaign.endTime` for the countdown (don't hardcode); cross-sell links **must** keep `?campaign=` so the discount/context survives; use each product's own `currency`. See guide §5.12 for full field shapes.

### Form Field Types

| fieldType | Render As |
|-----------|-----------|
| `text`, `email`, `tel`, `number`, `date` | `<input type="...">` |
| `textarea` | `<textarea>` |
| `select` | `<select>` |
| `multiselect` | Checkbox group |

### Required Data Attributes on Form Elements

Every form element must include these attributes for data collection:

```javascript
'data-field-key="' + field.fieldKey + '"'
'data-template-id="' + templateId + '"'
'data-group-id="' + groupId + '"'
'data-submit-timing="' + submitTiming + '"'  // 'pre_pay' | 'post_pay'
```

### Login Gating

The buy flow requires the user to be logged in before filling forms or paying. Follow these rules:

- **Do NOT redirect to login on page load.** Step 1 (product info) must be visible to everyone so they can browse. Only gate from the first form step onward — unless the requirement explicitly asks for "login on entry".
- **Login signal is `data.userEmail`** (only set for logged-in users). When rendering a form step or the pay step, swap the content for a login prompt if `data.userEmail` is missing.
- **Safety net:** in `pay()`, re-check the email and call `redirectToLogin()` if it's empty.

```javascript
// Redirect to login: prefer the SDK, fall back to platform /login with a return URL
function redirectToLogin() {
  if (window.EduReady && window.EduReady.redirectToLogin) {
    window.EduReady.redirectToLogin();
    return;
  }
  var returnUrl = window.location.pathname + window.location.search;
  window.location.href = BASE_URL + '/login?redirect=' + encodeURIComponent(returnUrl);
}

function renderLoginPrompt() {
  return '<div style="text-align:center;padding:40px 20px">'
    + '<h2>请先登录</h2><p>登录后即可填写信息并提交购买</p>'
    + '<button class="btn btn-primary" onclick="redirectToLogin()">登录</button></div>';
}

// When rendering a form/pay step:
//   var inner = data.userEmail ? renderFormFields(...) : renderLoginPrompt();
```

### Payment Flow

```javascript
async function pay() {
  // Login safety net: gate before checkout instead of letting it fail server-side
  var email = productData.userEmail || document.getElementById('email').value.trim();
  if (!email) {
    redirectToLogin();
    return;
  }

  var submissions = collectFormSubmissions();

  var result = await EduReady.checkout({
    email: email,
    couponCode: couponCode,
    formSubmissions: submissions
  });

  // Redirect to Stripe (traverse to top window)
  var win = window;
  try { while (win !== win.parent) { win = win.parent; } } catch (e) {}
  win.location.href = result.url;
}
```

## CSS Guidelines

- Use CSS variables for theming: `:root { --primary: #667eea; --primary-dark: #764ba2; }`
- Include responsive breakpoints: `@media (max-width: 640px) { ... }`
- Use system fonts: `font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif`
- Support both light and dark aesthetics if needed

## Local Development

1. Start local server: `npx serve . -p 5500`
2. Access via platform with `?dev=` parameter:
   - Campaign: `https://eduready.ai/campaign/{slug}?dev=http://127.0.0.1:5500/index.html`
   - Buy: `https://eduready.ai/buy/{productId}?campaign={id}&dev=http://127.0.0.1:5500/buy.html`

## Creating a Theme

When asked to create a theme:

1. **Ask for requirements**: color scheme, layout style, specific features needed
2. **Read the example files** from `references/` directory for working code patterns
3. **Generate style.css**: CSS variables, responsive grid, component styles
4. **Generate index.html**: Hero section with countdown, product grid, buy buttons
5. **Generate buy.html**: Step wizard, form rendering, coupon input, payment integration
6. **Ensure all SDK integration works**: postMessage, EduReady.ready, checkout flow

Use `references/index.html`, `references/buy.html`, and `references/style.css` as templates.
