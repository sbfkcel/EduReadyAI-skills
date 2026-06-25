# 活动主题定制系统 - 完整开发手册

一套主题 = 一个目录，包含活动页和购买页两个 HTML 文件。

```
用户访问流程：

  /campaign/{slug}                    /buy/{productId}?campaign={id}
  ┌─────────────┐                     ┌─────────────────┐
  │  index.html  │ ──── 点击购买 ────→ │    buy.html      │
  │  (活动着陆页) │                     │  (结账购买页)    │
  └─────────────┘                     └─────────────────┘
  展示产品、折扣、倒计时                  表单填写、优惠券、付款
```

---

## 目录结构

```
my-theme/                   ← 上传整个文件夹
  ├── index.html            ← 活动页入口（必需）
  ├── buy.html              ← 购买页入口（可选）
  ├── style.css             ← 共享样式
  └── assets/               ← 静态资源（图片、字体等）
      └── hero.jpg
```

- **活动页**（`index.html`）：营销着陆页，通过 postMessage 从父页面获取真实数据
- **购买页**（`buy.html`）：结账页，通过 SDK 与平台交互
- 两个页面共享同目录下的 CSS、JS、图片等资源
- 购买页是可选的，不配置则使用系统默认购买页

---

## 一、上传方式

在运营后台 → 活动详情 → 「主题」Tab，选择本地文件夹上传。

填写：
- **主题名称**：如 "春节特惠"
- **主题标识**（slug）：如 `spring-2026`，只能小写字母、数字、连字符
- **活动页入口**：默认 `index.html`
- **购买页入口**：如 `buy.html`，留空则使用系统默认购买页

第一个上传的主题自动设为默认。重复上传同 slug 主题会更新而非报错。

---

## 二、访问 URL

活动页和购买页的静态文件都通过同一个端点访问：

```
/api/campaign-files/{campaignSlug}/{templateSlug}/{filePath}
```

示例：
```
/api/campaign-files/spring-2026/spring-theme/index.html      ← 活动页
/api/campaign-files/spring-2026/spring-theme/buy.html         ← 购买页
/api/campaign-files/spring-2026/spring-theme/style.css        ← 共享样式
```

SDK 脚本：
```
/api/buy-template-files/sdk/eduready-buy.js
```

---

## 三、渲染逻辑

### 活动页 `/campaign/{slug}`

1. 查询活动的默认模板
2. 有自定义模板 → iframe 全屏渲染 `index.html`
3. 无自定义模板 → 渲染系统内置活动页（可通过 `pageConfig` 配置标题、背景等）

### 购买页 `/buy/{productId}?campaign={id}&template={slug}`

1. URL 中有 `template` 参数 → 使用指定模板的 `buy.html`
2. 无 `template` 参数 → 使用活动默认模板的 `buyEntryFile`
3. 都没有 → 渲染系统默认购买页

---

## 四、活动页开发详解

### 4.1 基本结构

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
  <!-- 页面内容 -->

  <!-- SDK：自动检测环境 -->
  <script>
    document.write('<script src="' + (
      (location.hostname === '127.0.0.1' || location.hostname === 'localhost')
        ? 'https://eduready.ai'
        : ''
    ) + '/api/campaign-files/sdk/eduready-campaign.js?v=3"><\/script>');
  </script>

  <script>
    // 业务逻辑...
  </script>
</body>
</html>
```

### 4.2 postMessage 协议

活动页在 iframe 中运行，通过 postMessage 与父页面（CampaignContent）通信获取真实数据。

#### 模板 → 父页面（请求）

```javascript
window.parent.postMessage({ type: 'eduready:campaign:requestData' }, '*');
```

#### 父页面 → 模板（响应）

```javascript
{
  type: 'eduready:campaign:data',
  payload: {
    campaign: {
      id: 1,
      name: '春节特惠',
      endTime: '2026-02-15T00:00:00Z',
      status: 'active',
      pageConfig: { title, subtitle, buttonText, rules }
    },
    products: [{
      id: 1,
      publicId: '38e54a5b-b10b-4b32-9cf6-feb66eb4fce1',
      name: '课程名称',
      description: '课程描述',
      coverUrl: 'https://...',   // 产品封面图，平台存储比例为 16:9；可能为空（未上传时）
      basePrice: 9999,
      discountPrice: 7999,
      currency: 'CNY',
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
}
```

#### 完整用法

```javascript
// 请求数据
window.parent.postMessage({ type: 'eduready:campaign:requestData' }, '*');

// 接收数据
window.addEventListener('message', function(e) {
  if (e.data?.type === 'eduready:campaign:data') {
    var campaign = e.data.payload.campaign;
    var products = e.data.payload.products;
    var config = e.data.payload.config || {};
    var isActive = e.data.payload.isActive;

    // 渲染页面...
  }
});
```

### 4.3 购买链接构建

活动页中的购买按钮应指向平台购买页：

```javascript
// 从 URL 读取参数
var urlParams = new URLSearchParams(window.location.search);
var campaignId = urlParams.get('campaign');
var campaignSlug = urlParams.get('campaignSlug');
var templateSlug = urlParams.get('template');
var refCode = urlParams.get('ref');

// 构建购买页 URL
var BASE_URL = (location.hostname === '127.0.0.1' || location.hostname === 'localhost')
  ? 'https://eduready.ai' : '';

function buildBuyUrl(productId) {
  var qs = [];
  if (campaignSlug) qs.push('campaign=' + campaignSlug);
  if (templateSlug) qs.push('template=' + templateSlug);
  if (refCode) qs.push('ref=' + refCode);
  return BASE_URL + '/buy/' + productId + (qs.length ? '?' + qs.join('&') : '');
}

// 在顶层窗口跳转（避免在 iframe 中打开）
function goToBuy(url) {
  var win = window;
  try {
    while (win !== win.parent) {
      win = win.parent;
    }
  } catch (e) {}
  win.location.href = url;
}
```

### 4.4 倒计时实现

```javascript
if (campaign.endTime && isActive) {
  var endTime = new Date(campaign.endTime);

  function pad(n) { return n < 10 ? '0' + n : '' + n; }

  function updateCountdown() {
    var diff = endTime - new Date();
    if (diff <= 0) return;
    document.getElementById('cd-days').textContent = pad(Math.floor(diff / 86400000));
    document.getElementById('cd-hours').textContent = pad(Math.floor((diff % 86400000) / 3600000));
    document.getElementById('cd-mins').textContent = pad(Math.floor((diff % 3600000) / 60000));
    document.getElementById('cd-secs').textContent = pad(Math.floor((diff % 60000) / 1000));
  }

  updateCountdown();
  setInterval(updateCountdown, 1000);
}
```

### 4.5 产品列表渲染

```javascript
// 解析封面图：为空→平台默认图 /cover.jpg；绝对 URL 原样；相对路径补 BASE_URL
// 注意：务必用 BASE_URL 前缀，否则本地调试时 /cover.jpg 会指向 127.0.0.1 而裂图
function resolveCoverUrl(url) {
  if (!url) return BASE_URL + '/cover.jpg';
  if (/^https?:\/\//.test(url)) return url;
  return BASE_URL + url;
}

function renderProducts(products, isActive, config) {
  var grid = document.getElementById('product-grid');
  grid.innerHTML = '';

  if (!products || products.length === 0) {
    grid.innerHTML = '<p style="text-align:center;color:#999">暂无产品</p>';
    return;
  }

  products.forEach(function(p) {
    var buyUrl = buildBuyUrl(p.publicId || p.id);
    var card = document.createElement('div');
    card.className = 'product-card';

    // coverUrl 可能为空，resolveCoverUrl 会回退到平台默认图 /cover.jpg
    var imgSrc = resolveCoverUrl(p.coverUrl);
    var imgHtml = '<img src="' + imgSrc + '" alt="' + p.name + '" style="aspect-ratio:16/9;object-fit:cover;width:100%;">';
    var priceHtml = formatPrice(p.discountPrice || p.basePrice, p.currency);
    var origHtml = (p.basePrice !== p.discountPrice)
      ? '<span class="price-original">' + formatPrice(p.basePrice, p.currency) + '</span>'
      : '';

    var btnHtml;
    if (p.soldOut) {
      btnHtml = '<button class="btn btn-primary" disabled>已售罄</button>';
    } else if (isActive) {
      btnHtml = '<a href="javascript:void(0)" onclick="goToBuy(\'' + buyUrl.replace(/'/g, "\\'") + '\')" class="btn btn-primary">'
        + (config.buttonText || '立即抢购') + '</a>';
    } else {
      btnHtml = '<button class="btn btn-primary" disabled>已结束</button>';
    }

    card.innerHTML = imgHtml +
      '<div class="info">' +
        '<div class="name">' + p.name + '</div>' +
        (p.description ? '<div class="desc">' + p.description + '</div>' : '') +
        '<div class="price-row">' +
          '<span class="price-current">' + priceHtml + '</span>' +
          origHtml +
        '</div>' +
        btnHtml +
      '</div>';

    grid.appendChild(card);
  });
}
```

### 4.6 货币格式化

`currency` 由平台系统配置决定（通过 API 返回，如 `data.currency` 或 `product.currency`），主题内不要硬编码货币符号。

```javascript
function formatPrice(amount, currency) {
  var symbols = { CNY: '¥', USD: '$', EUR: '€', GBP: '£', JPY: '¥' };
  var sym = symbols[currency] || currency;
  return sym + ' ' + Number(amount).toLocaleString('en-US', {
    minimumFractionDigits: 0,
    maximumFractionDigits: 2
  });
}
```

### 4.7 通知父页面调整高度

```javascript
// 在内容变化后调用
try {
  window.parent.postMessage({
    type: 'eduready:campaign:resize',
    height: document.body.scrollHeight
  }, '*');
} catch (e) {}
```

---

## 五、购买页开发详解

### 5.1 基本结构

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
      <!-- Steps sidebar (动态生成) -->
      <div class="steps" id="steps"></div>
      <!-- Step containers (动态生成) -->
      <div class="step-content" id="step-container" style="flex:1;min-width:0"></div>
    </div>
  </div>

  <!-- SDK -->
  <script>
    document.write('<script src="' + (
      (location.hostname === '127.0.0.1' || location.hostname === 'localhost')
        ? 'https://eduready.ai'
        : ''
    ) + '/api/buy-template-files/sdk/eduready-buy.js?v=3"><\/script>');
  </script>

  <script>
    // 业务逻辑...
  </script>
</body>
</html>
```

### 5.2 SDK 初始化

```javascript
EduReady.ready(function(data) {
  // data.product           — 产品信息
  // data.pricing           — 价格信息
  // data.formGroups        — 预付表单组
  // data.postPayFormGroups — 付后表单组
  // data.refUser           — 推荐人
  // data.coupon            — 优惠券
  // data.previousSubmissions — 历史提交数据（已登录用户）
  // data.userEmail         — 当前用户邮箱（已登录用户）
});
```

### 5.3 数据结构详解

#### product（产品信息）

```javascript
{
  id: 1,
  publicId: '38e54a5b-b10b-4b32-9cf6-feb66eb4fce1',
  name: '课程名称',
  type: 'course',
  description: '课程描述',
  coverUrl: 'https://...',
  basePrice: 9999,
  currency: 'CNY',
  stages: [                    // 交付阶段
    {
      id: 1,
      name: '第一阶段',
      withdrawableAmount: null,
      withdrawableRate: null,
      intervalDays: 30
    }
  ]
}
```

#### pricing（价格信息）

```javascript
{
  originalPrice: 9999,         // 原价
  finalPrice: 7999,            // 最终价格（含活动折扣）
  discountAmount: 2000,        // 折扣金额
  cashbackInfo: {              // 返现信息（如果有）
    triggerType: 'manual',
    delayDays: 0,
    amount: 500
  }
}
```

#### formGroups（表单组）

```javascript
[
  {
    productFormGroupId: 1,     // 关联 ID（提交时使用）
    groupId: 1,
    name: '学生信息',
    description: '请填写学生基本信息',
    submitTiming: 'pre_pay',   // 'pre_pay' | 'post_pay'
    minSubmissions: 1,
    maxSubmissions: 3,
    sortOrder: 0,
    templates: [               // 表单模板
      {
        id: 1,
        name: '基本信息',
        description: '请填写以下信息',
        sortOrder: 0,
        fields: [              // 表单字段
          {
            id: 1,
            fieldKey: 'student_name',
            label: '学生姓名',
            fieldType: 'text',     // text | email | tel | number | date
            isRequired: true,
            placeholder: '请输入姓名',
            defaultValue: null,
            hint: '填写真实姓名',
            options: null,         // select/multiselect 时有值
            validation: null,
            showWhen: null,
            sortOrder: 0
          }
        ]
      }
    ]
  }
]
```

#### 字段类型

| fieldType | 渲染为 | 说明 |
|-----------|--------|------|
| `text` | `<input type="text">` | 单行文本 |
| `email` | `<input type="email">` | 邮箱 |
| `tel` | `<input type="tel">` | 电话 |
| `number` | `<input type="number">` | 数字 |
| `date` | `<input type="date">` | 日期 |
| `textarea` | `<textarea>` | 多行文本 |
| `select` | `<select>` | 单选下拉 |
| `multiselect` | checkbox group | 多选 |

### 5.4 表单渲染

```javascript
function renderFormField(field, templateId, groupId, submitTiming) {
  var fieldType = field.fieldType || field.type;
  var html = '<div class="form-group">';
  html += '<label>' + field.label;
  if (field.isRequired) html += ' <span class="required">*</span>';
  html += '</label>';

  var reqAttr = field.isRequired ? ' data-required="true"' : '';

  // 从历史提交中获取默认值
  var defaultValue = '';
  var prevData = previousSubmissions[templateId];
  if (prevData && prevData.length > 0 && prevData[0].data) {
    defaultValue = prevData[0].data[field.fieldKey] || '';
  }

  if (fieldType === 'select' && field.options) {
    html += '<select data-field-key="' + field.fieldKey
      + '" data-template-id="' + templateId
      + '" data-group-id="' + groupId
      + '" data-submit-timing="' + submitTiming + '"' + reqAttr + '>';
    html += '<option value="">请选择</option>';
    (Array.isArray(field.options) ? field.options : []).forEach(function(opt) {
      var optVal = opt.value || opt;
      var selected = defaultValue == optVal ? ' selected' : '';
      html += '<option value="' + optVal + '"' + selected + '>' + (opt.label || opt) + '</option>';
    });
    html += '</select>';
  } else if (fieldType === 'multiselect' && field.options) {
    html += '<div class="checkbox-group" data-field-key="' + field.fieldKey
      + '" data-template-id="' + templateId
      + '" data-group-id="' + groupId
      + '" data-submit-timing="' + submitTiming + '"' + reqAttr + '>';
    var selectedValues = Array.isArray(defaultValue) ? defaultValue : [];
    (Array.isArray(field.options) ? field.options : []).forEach(function(opt) {
      var optVal = opt.value || opt;
      var checked = selectedValues.includes(optVal) ? ' checked' : '';
      html += '<label class="checkbox-item">'
        + '<input type="checkbox" value="' + optVal + '"' + checked + '>'
        + ' ' + (opt.label || opt)
        + '</label>';
    });
    html += '</div>';
  } else if (fieldType === 'textarea') {
    html += '<textarea rows="3" data-field-key="' + field.fieldKey
      + '" data-template-id="' + templateId
      + '" data-group-id="' + groupId
      + '" data-submit-timing="' + submitTiming + '"'
      + (field.placeholder ? ' placeholder="' + field.placeholder + '"' : '')
      + reqAttr + '>'
      + (defaultValue || '')
      + '</textarea>';
  } else {
    html += '<input type="' + (fieldType || 'text') + '" data-field-key="' + field.fieldKey
      + '" data-template-id="' + templateId
      + '" data-group-id="' + groupId
      + '" data-submit-timing="' + submitTiming + '"'
      + (defaultValue ? ' value="' + defaultValue + '"' : '')
      + (field.placeholder ? ' placeholder="' + field.placeholder + '"' : '')
      + reqAttr + '>';
  }

  if (field.hint) {
    html += '<div class="hint">' + field.hint + '</div>';
  }
  html += '</div>';
  return html;
}
```

### 5.5 表单验证

```javascript
function validateStep(stepNum) {
  var stepEl = document.getElementById('step' + stepNum);
  if (!stepEl) return true;

  // 清除之前的错误提示
  stepEl.querySelectorAll('.field-error').forEach(function(el) { el.remove(); });
  stepEl.querySelectorAll('.has-error').forEach(function(el) { el.classList.remove('has-error'); });

  var valid = true;
  var fields = stepEl.querySelectorAll('[data-field-key]');

  fields.forEach(function(el) {
    var isRequired = el.dataset.required === 'true';
    if (!isRequired) return;

    var value;
    if (el.classList.contains('checkbox-group')) {
      var checked = el.querySelectorAll('input[type="checkbox"]:checked');
      value = checked.length > 0 ? 'has-value' : '';
    } else {
      value = el.value ? el.value.trim() : '';
    }

    if (!value) {
      valid = false;
      var group = el.closest('.form-group');
      if (group) {
        group.classList.add('has-error');
        var errDiv = document.createElement('div');
        errDiv.className = 'field-error';
        errDiv.textContent = '此项为必填';
        group.appendChild(errDiv);
      }
    }
  });

  return valid;
}
```

### 5.6 步骤切换

```javascript
var currentStep = 1;
var totalSteps = 1;

function goStep(n) {
  if (n < 1 || n > totalSteps) return;

  // 向前切换时验证当前步骤
  if (n > currentStep && !validateStep(currentStep)) {
    return;
  }

  currentStep = n;

  // 切换步骤内容
  document.querySelectorAll('.step').forEach(function(el) { el.classList.remove('active'); });
  var stepEl = document.getElementById('step' + n);
  if (stepEl) stepEl.classList.add('active');

  // 更新步骤指示器
  document.querySelectorAll('.step-indicator').forEach(function(el) {
    var s = parseInt(el.dataset.step);
    el.classList.remove('active', 'done');
    if (s === n) el.classList.add('active');
    else if (s < n) el.classList.add('done');
  });

  EduReady.stepChange(n, totalSteps);
  setTimeout(function() { EduReady.resize(document.body.scrollHeight); }, 50);
}
```

### 5.7 表单数据收集

```javascript
function collectFormSubmissions() {
  var grouped = {};

  document.querySelectorAll('[data-field-key]').forEach(function(el) {
    var key = el.dataset.groupId + '_' + el.dataset.templateId;
    if (!grouped[key]) {
      grouped[key] = {
        productFormGroupId: parseInt(el.dataset.groupId),
        templateId: parseInt(el.dataset.templateId),
        submitTiming: el.dataset.submitTiming,
        data: {}
      };
    }

    var value;
    if (el.classList.contains('checkbox-group')) {
      var checked = [];
      el.querySelectorAll('input[type="checkbox"]:checked').forEach(function(cb) {
        checked.push(cb.value);
      });
      value = checked.length > 0 ? checked : '';
    } else {
      value = el.value ? el.value.trim() : '';
    }
    if (value) {
      grouped[key].data[el.dataset.fieldKey] = value;
    }
  });

  // 只返回有数据的表单组
  return Object.values(grouped).filter(function(g) {
    return Object.keys(g.data).length > 0;
  });
}
```

### 5.8 优惠券验证

```javascript
var couponInfo = null;

async function verifyCoupon() {
  var code = document.getElementById('coupon').value.trim();
  if (!code) return;
  var msg = document.getElementById('coupon-msg');

  try {
    var result = await EduReady.verifyCoupon(code, productData.product.publicId || productData.product.id);
    couponInfo = result;
    showCouponSuccess(result);
  } catch (e) {
    msg.className = 'coupon-error';
    msg.textContent = e.message || '优惠券无效';
    couponInfo = null;
  }
}

function showCouponSuccess(coupon) {
  var msg = document.getElementById('coupon-msg');
  if (!msg) return;
  var cur = productData.product.currency;
  msg.className = 'coupon-success';
  msg.textContent = '优惠券已生效，减免 ' + formatPrice(coupon.discountAmount, cur);

  // 更新价格显示
  var orig = productData.pricing.originalPrice;
  var final_ = orig - coupon.discountAmount;
  var discountRow = document.getElementById('sum-discount-row');
  if (discountRow) discountRow.style.display = 'flex';
  var discountEl = document.getElementById('sum-discount');
  if (discountEl) discountEl.textContent = '-' + formatPrice(coupon.discountAmount, cur);
  var totalEl = document.getElementById('sum-total');
  if (totalEl) totalEl.textContent = formatPrice(final_, cur);
}
```

### 5.9 登录门控（重要）

购买流程需要登录后才能填写表单与支付。门控原则：

- **不要一进购买页就跳登录**。Step 1（产品信息）应对所有访客可见，便于浏览。
- **登录门控放在「进入表单步骤之前」**：从第一个表单步骤起，未登录时用登录提示替代表单内容；支付步骤同理。除非需求特别指定「访问即登录」，否则不要在 `EduReady.ready` 里直接跳转。
- 登录信号是 `data.userEmail`（已登录用户才有值）。
- 兜底：`pay()` 提交前再判一次，无 email 则 `redirectToLogin()`。

```javascript
// 跳转登录：优先用 SDK，回退到平台 /login（携带当前页作为回跳地址）
function redirectToLogin() {
  if (window.EduReady && window.EduReady.redirectToLogin) {
    window.EduReady.redirectToLogin();
    return;
  }
  var returnUrl = window.location.pathname + window.location.search;
  window.location.href = BASE_URL + '/login?redirect=' + encodeURIComponent(returnUrl);
}

// 未登录时，用它替代表单/支付步骤的内容
function renderLoginPrompt() {
  return '<div style="text-align:center;padding:40px 20px">'
    + '<h2>请先登录</h2>'
    + '<p class="step-desc">登录后即可填写信息并提交购买</p>'
    + '<button class="btn btn-primary" onclick="redirectToLogin()">登录</button>'
    + '</div>';
}

// 渲染表单/支付步骤时：
// var inner = data.userEmail ? renderFormFields(...) : renderLoginPrompt();
```

### 5.10 支付提交

```javascript
async function pay() {
  // 登录兜底：未登录直接跳登录，避免无谓的 checkout 失败
  var email = productData.userEmail || document.getElementById('email').value.trim();
  if (!email) {
    redirectToLogin();
    return;
  }

  var btn = document.getElementById('pay-btn');
  btn.disabled = true;
  btn.textContent = '处理中...';
  document.getElementById('pay-error').style.display = 'none';

  var submissions = collectFormSubmissions();

  try {
    var result = await EduReady.checkout({
      email: email,
      couponCode: couponInfo ? (couponInfo.code || document.getElementById('coupon').value) : undefined,
      formSubmissions: submissions
    });

    // 跳转到 Stripe 支付页面 - 向上遍历找到最顶层窗口
    var win = window;
    try {
      while (win !== win.parent) {
        win = win.parent;
      }
    } catch (e) {}
    win.location.href = result.url;
  } catch (e) {
    var errEl = document.getElementById('pay-error');
    errEl.style.display = 'block';
    errEl.textContent = e.message || '支付失败，请稍后重试';
    btn.disabled = false;
    btn.textContent = '立即支付';
  }
}
```

### 5.11 步骤指示器（侧边栏）

```javascript
function renderStepIndicators(data) {
  var stepsEl = document.getElementById('steps');
  stepsEl.innerHTML = '';

  // Step 1: 选择课程
  stepsEl.appendChild(createStepIndicator(1, '选择课程'));

  // Steps 2-N: 每个模板
  allTemplates.forEach(function(item, i) {
    stepsEl.appendChild(createStepConnector());
    stepsEl.appendChild(createStepIndicator(i + 2, item.template.name || '信息 ' + (i + 1)));
  });

  // Last step: 确认支付
  stepsEl.appendChild(createStepConnector());
  stepsEl.appendChild(createStepIndicator(totalSteps, '确认支付'));
}

function createStepIndicator(stepNum, label) {
  var div = document.createElement('div');
  div.className = 'step-indicator' + (stepNum === 1 ? ' active' : '');
  div.dataset.step = stepNum;
  div.innerHTML = '<div class="dot">' + stepNum + '</div><span>' + label + '</span>';
  div.onclick = function() { goStep(stepNum); };
  return div;
}

function createStepConnector() {
  var div = document.createElement('div');
  div.className = 'step-connector';
  return div;
}
```

---

## 六、SDK API 参考

### EduReady 对象

| 方法 | 参数 | 返回值 | 说明 |
|------|------|--------|------|
| `EduReady.ready(cb)` | callback | — | 收到初始数据时回调 |
| `EduReady.getProductInfo()` | — | `{product, pricing}` | 获取产品信息 |
| `EduReady.getFormGroups()` | — | `{formGroups, postPayFormGroups}` | 获取表单组 |
| `EduReady.verifyCoupon(code, productId)` | 优惠券码, 产品ID | `{code, type, value, discountAmount}` | 验证优惠券 |
| `EduReady.checkout(opts)` | `{email, couponCode?, formSubmissions?}` | `{url, sessionId, orderId}` | 创建支付并跳转 |
| `EduReady.submitForms(orderId, submissions)` | 订单ID, 表单数组 | — | 提交付款后表单 |
| `EduReady.stepChange(step, total)` | 当前步, 总步数 | — | 通知步骤变化 |
| `EduReady.resize(height)` | 像素值 | — | 调整 iframe 高度 |
| `EduReady.redirectToCheckout(url)` | URL | — | 在顶层窗口跳转 |
| `EduReady.redirectToLogin()` | — | — | 跳转平台登录页（购买前未登录时调用） |
| `EduReady.getParam(name)` | 参数名 | `string\|null` | 读取 URL 参数 |
| `EduReady.getParams()` | — | `object` | 读取所有 URL 参数 |
| `EduReady.isInIframe` | — | `boolean` | 是否在 iframe 中 |

所有异步方法返回 Promise，可用 `async/await`。

### SDK 运行模式

| 模式 | 条件 | 数据来源 | 通信方式 |
|------|------|---------|---------|
| iframe 模式 | `window.self !== window.top` | 父页面通过 postMessage 发送 | postMessage |
| 独立模式 | 直接在浏览器打开 | 直接调用平台 API | fetch |

独立模式需要 URL 中包含 `productId` 参数，如 `buy.html?productId=1`。

---

## 七、本地开发调试

### 核心原理

```
浏览器 → eduready.ai（平台）→ iframe 加载本地文件
                                    ↓
                              postMessage 通信
                                    ↓
                           平台提供真实数据（产品、价格、表单）
```

### 步骤

1. 在本地创建模板目录：

```
my-theme/
  ├── index.html      ← 活动页
  ├── buy.html        ← 购买页
  └── style.css
```

2. 启动本地静态服务器：

```bash
# VS Code Live Server（推荐）
# 右键 index.html → Open with Live Server

# 或用 npx serve
npx serve . -p 5500

# 或用 Python
python3 -m http.server 5500
```

3. 通过平台访问，加 `?dev=` 参数：

```bash
# 活动页
https://eduready.ai/campaign/{slug}?dev=http://127.0.0.1:5500/index.html

# 购买页（通常从活动页点击购买自动跳转）
https://eduready.ai/buy/{productId}?campaign={id}&dev=http://127.0.0.1:5500/buy.html
```

### 完整调试流程

```
1. 打开活动页：
   https://eduready.ai/campaign/q91yrw?dev=http://127.0.0.1:5500/index.html
   ↓
   平台获取活动数据 → iframe 加载本地 index.html → 模板通过 postMessage 获取真实数据

2. 点击购买按钮 → 跳转到：
   https://eduready.ai/buy/1?campaign=1&dev=http://127.0.0.1:5500/buy.html
   ↓
   平台获取产品数据 → iframe 加载本地 buy.html → SDK 通过 postMessage 获取真实数据

3. 编辑本地 HTML → 刷新浏览器 → 看到效果
```

---

## 八、CSS 样式参考

### 颜色变量建议

```css
:root {
  --primary: #667eea;
  --primary-dark: #764ba2;
  --danger: #e74c3c;
  --success: #27ae60;
  --text: #1a1a2e;
  --text-light: #666;
  --text-muted: #999;
  --bg: #f8f9fa;
  --border: #e9ecef;
  --white: #ffffff;
}
```

### 响应式断点

```css
/* 移动端 */
@media (max-width: 640px) {
  .hero h1 { font-size: 28px; }
  .product-grid { grid-template-columns: 1fr; }
  .buy-card { flex-direction: column; }
  .steps { flex-direction: row; overflow-x: auto; }
}
```

---

## 九、技术细节

### 数据库

```sql
-- 统一模板表
campaign_templates (
  id, campaign_id, name, slug,
  is_default,
  storage_path,        -- campaigns/{campaignSlug}/{templateSlug}
  entry_file,          -- 活动页入口（默认 index.html）
  buy_entry_file,      -- 购买页入口（可选，如 buy.html）
  file_manifest,       -- JSON 文件清单
  status, created_at, updated_at, deleted_at
)

-- campaigns 表关联
campaigns.default_template_id → campaign_templates.id
```

### API 端点

| 端点 | 方法 | 用途 |
|------|------|------|
| `/api/ops/campaigns/:id/templates` | GET/POST | 主题 CRUD |
| `/api/campaign-files/:slug/:tpl/:file` | GET | 静态文件服务 |
| `/api/buy-template-files/sdk/eduready-buy.js` | GET | SDK 脚本 |
| `/api/campaigns/:slug` | GET | 活动数据（含模板列表） |
| `/api/checkout/buy-template-data` | GET | 购买页数据 |
| `/api/checkout/session` | POST | 创建 Stripe 支付 |
| `/api/checkout/submit-form` | POST | 提交表单 |
| `/api/checkout/verify-coupon` | GET | 验证优惠券 |

### 无自定义主题时的回退

| 页面 | 回退方案 |
|------|---------|
| 活动页 | 系统内置页面，通过 `campaign.pageConfig` 配置标题、背景、规则 |
| 购买页 | 系统默认 React 组件，支持产品展示、表单、优惠券、Stripe 支付 |
