---
title: 網路安全筆記 (2) 用戶端的漏洞
subtitle: 
date: 2023-07-03 12:00:00
tags: ["網路"]
---

{{< line_break >}}
# Corss-Site scripting (XSS)
{{< line_break >}}

跨站腳本漏洞通常允許攻擊者偽裝成受害用戶，執行用戶能夠執行的行動並訪問用戶的任何數據

跨站腳本的工作原理是操控一個有漏洞的網站，使其向用戶返回惡意的 JS，當惡意代碼在受害者的瀏覽器中執行時，攻擊者可以操控破壞他與應用程序的互動

有三種 XSS 攻擊

- `Reflected XSS`：惡意腳本來自當前的 HTTP 請求
- `Stored XSS`：惡意腳本來自網站的資料庫
- `DOM-based XSS`：漏洞存在於客戶端程式碼非服務器端程式碼

<!--more-->

{{< line_break >}}
## Content Security Policy (CSP)
{{< line_break >}}

CSP 是一種瀏覽器的安全機制，目的在緩減 XSS 攻擊

CSP 通過限制一個頁面可加載的資源 (如腳本和圖像) 以及限制一個頁面是否可在其他頁面作用

為了啟用 CSP，一個響應需要包含 Content-Security-Policy header，值帶指定的策略 policy

{{< line_break >}}
## 預防方法
{{< line_break >}}

Filter input on arrival：嚴格檢查與過濾輸入

Encode data on output：對 HTTP 回應中用戶可控制的資料進行編碼，以防止其被解釋為要執行的內容

使用適當的 Content-Type 和 X-Content-Type-Options Header，確保瀏覽器以預期的方式解釋響應

使用 CSP 來降低 XSS 漏洞的嚴重性

{{< line_break >}}
# Corss-site request forgery (CSRF)
{{< line_break >}}

跨站請求偽造 (CSRF) 漏洞允許攻擊者誘導用戶執行他們不打算執行的操作

CSRF 攻擊要成立必須要具備以下三條件

- A relevant action：在應用程式中，有一個攻擊者有理由誘發的行動，例如對用戶特定數據進行修改
- Cookie-based session handling：該動作依賴會話 cookie 來識別發出請求的用戶，沒有其他機制來跟蹤會話或驗證用戶請求
- No unpredictable request parameters：執行的動作請求不含任何攻擊者無法確定或猜測其值的參數，例如當使用者改變他們密碼時，如果攻擊者需要知道現有密碼值時，該功能沒有 CSRF 漏洞

{{< line_break >}}
## 預防方法
{{< line_break >}}

CSRF token：CSRF token 是一個祕密的不可預期的值，由服務器端產生與客戶端共享，當試圖執行一個敏感的動作時，客戶端必須在請求中包含正確的 CSRF token

SameSite cookie：SameSite 是一種瀏覽器安全機制，用於限制一個網站的 cookie 是否能被包含在其他網站的請求中，SameSite 限制能阻止攻擊者跨網站觸發 CSRF 攻擊


{{< line_break >}}
# Cross-origin resource sharing (CORS)
{{< line_break >}}

跨資源共享 (CORS) 是一種瀏覽器機制，可以限制對於特定域外的資源的訪問

{{< line_break >}}
## Same-origin policy (SOP)
{{< line_break >}}

同源策略 (Same-origin policy) 是一種限制性的跨源規範，他限制了一個網站與源域之外的資源進行互動

同源：url 要同 scheme、host、port

{{< line_break >}}
## Access-Control-Allow-Origin
{{< line_break >}}

Access-Control-Allow-Origin header 含在一個網站對來自另一個網站的請求的響應中，瀏覽器會將 Access-Control-Allow-Origin 與請求網站的來源進行比較，如果匹配則允許能訪問該響應

{{< line_break >}}
## Access-Control-Allow-Credentials
{{< line_break >}}

跨源請求的默認行為是沒有傳遞 Cookie 這樣的憑證

在跨源請求帶 Cookie 的情況，響應必須帶 Access-Control-Allow-Credentials: true，瀏覽器才允許讀取響應

{{< line_break >}}
## Pre-flight checks 預檢請求
{{< line_break >}}

當一個跨域請求包括一個非標準的 HTTP 方法或 headers 時，跨源請求前會有一個使用 OPTIONS 方法的請求，CORS 協議要在允許跨源請求前初步檢查那些方法和 header 是允許的，這稱為 Pre-flight check

```bash
OPTIONS /data HTTP/1.1
Host: <some website>
...
Origin: https://normal-website.com
Access-Control-Request-Method: PUT
Access-Control-Request-Headers: Special-Request-Header
```

回應：

```bash
HTTP/1.1 204 No Content
...
Access-Control-Allow-Origin: https://normal-website.com
Access-Control-Allow-Methods: PUT, POST, OPTIONS
Access-Control-Allow-Headers: Special-Request-Header
Access-Control-Allow-Credentials: true
Access-Control-Max-Age: 240
```

都有在 Allow 裡的話，就可發跨源請求

{{< line_break >}}
## 預防方法
{{< line_break >}}

CORS 漏洞主要是因錯誤的配置出現的，因此預防是一個配置問題

Access-Control-Allow-Origin 只允許信任的網站

避免使用 wildcard 或 null


{{< line_break >}}
# Clickjacking (UI redressing)
{{< line_break >}}

點擊劫持是一種基於界面的攻擊，即通過點即釣魚網站中的一些其他內容來誘使用戶點擊隱藏網站中的可操作內容

{{< line_break >}}
## 預防方法
{{< line_break >}}

X-Frame-Options header 提供網站所有者對使用 iframes 的控制

X-Frame-Options: deny 禁止在 iframe 中包含網頁

X-Frame-Options: sameorigin 限制同源才可

X-Frame-Options: allow-from https://normal-website.com 限制指定網站可

{{< line_break >}}