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
  - Sections 5.1-5.10: Buy page development
  - Section 6: SDK API reference
  - Section 8: CSS styling reference

- **`references/index.html`** — Example campaign page
- **`references/buy.html`** — Example checkout page
- **`references/style.css`** — Example responsive styles

Read the guide for complete details. Use the example files as templates when creating new themes.

## Campaign Page (index.html)

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
    ) + '/api/campaign-files/sdk/eduready-campaign.js?v=2"><\/script>');
  </script>
  <script>
    // Request data from parent page
    window.parent.postMessage({ type: 'eduready:campaign:requestData' }, '*');

    // Listen for response
    window.addEventListener('message', function(e) {
      if (e.data?.type === 'eduready:campaign:data') {
        var campaign = e.data.payload.campaign;
        var products = e.data.payload.products;
        var config = e.data.payload.config || {};
        var isActive = e.data.payload.isActive;
        // Render your page here
      }
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
    name: '课程名称',
    description: '课程描述',
    coverUrl: 'https://...',  // 产品封面图，比例 16:9；可能为空（未上传时）
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

function buildBuyUrl(productId) {
  var qs = [];
  if (campaignSlug) qs.push('campaign=' + campaignSlug);
  if (templateSlug) qs.push('template=' + templateSlug);
  if (refCode) qs.push('ref=' + refCode);
  return BASE_URL + '/buy/' + productId + (qs.length ? '?' + qs.join('&') : '');
}

function goToBuy(url) {
  var win = window;
  try { while (win !== win.parent) { win = win.parent; } } catch (e) {}
  win.location.href = url;
}
```

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
    ) + '/api/buy-template-files/sdk/eduready-buy.js?v=2"><\/script>');
  </script>
  <script>
    EduReady.ready(function(data) {
      // data.product, data.pricing, data.formGroups, data.postPayFormGroups
      // data.refUser, data.coupon, data.previousSubmissions, data.userEmail
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
| `EduReady.verifyCoupon(code, productId)` | `{code, type, value, discountAmount}` |
| `EduReady.checkout({email, couponCode?, formSubmissions?})` | `{url, sessionId, orderId}` |
| `EduReady.stepChange(step, total)` | Notify step change |
| `EduReady.resize(height)` | Adjust iframe height |

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

### Payment Flow

```javascript
async function pay() {
  var email = document.getElementById('email').value.trim();
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
