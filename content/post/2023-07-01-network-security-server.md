---
title: 網路安全筆記 (1) 服務器端的漏洞
subtitle: 
date: 2023-07-01 12:00:00
tags: ["網路"]
---

最近發現不錯的網路安全學習資源：

https://portswigger.net/web-security/learning-path

在這個網站介紹許多網路安全漏洞以及預防方法

如果有登入帳號的話，還有各種 Lab 讓你體驗嘗試攻擊者是如何利用這些漏洞來進行網路攻擊

以下是我基於此網站的內容作的筆記整理

{{< line_break >}}

# SQL Injection (SQL 注入)

{{< line_break >}}

SQL injection 發生在使用者能影響改動資料庫的 query 時

<!--more-->

範例：

預期正常的 Query：

- SELECT * FROM products WHERE category = 'Gifts' AND released = 1

{{< line_break >}}

若有 SQL Injection 的狀況，可能讓攻擊者有機會產生這樣的 Query：

- SELECT * FROM products WHERE category = 'Gifts'--' AND released = 1

released = 1 的部分被注解掉，這樣攻擊者就會取得非 release = 1 的資料

{{< line_break >}}

- SELECT * FROM products WHERE category = 'Gifts' OR 1=1--' AND released = 1

就會取得所有 Gifts products 資料


{{< line_break >}}

## 預防方法

{{< line_break >}}

使用 parameterized queries，不要直接將字串接起來

{{< line_break >}}

# Authentication 認證漏洞

{{< line_break >}}

三種認證要素：

1. 你知道的事 (knowledge factors)，如密碼或安全問題

2. 你擁有的東西 (possession factors)，如你的手機

3. 你是什麼 (inherence factors)，你的指紋


{{< line_break >}}

Authentication：你是誰

Authorization：允許做什麼事

認證相關的漏洞主要跟認證機制弱和實作邏輯缺陷等原因造成


{{< line_break >}}

# Directory Traversal 目錄遍歷

{{< line_break >}}

目錄遍歷漏洞讓攻擊者能讀取服務器上的任意文件


{{< line_break >}}

## 預防方法

{{< line_break >}}

避免文件系統API能讓用戶提供輸入

如果無可避免，須對輸入做驗證，跟允許的白名單比較，或驗證輸入只包含允許的字符，並對路徑規範化，驗證規範化的路徑以預期的基本目錄開始

{{< line_break >}}

# OS Command Injection

{{< line_break >}}

系統命令注入漏洞允許攻擊者在運行應用程序的服務器上執行任意OS系統命令


{{< line_break >}}

## 預防方法

{{< line_break >}}

別在應用程式調用系統指令

如不可避免，需做輸入驗證，驗證白名單允許值或驗證輸入是數字、字母、沒空白之類的


{{< line_break >}}

# Business logic vulnerabilities

{{< line_break >}}

業務邏輯漏洞是應用程序設計和實作中的缺陷，允許攻擊者引起非預期的行為


{{< line_break >}}

## 預防方法

{{< line_break >}}

確保開發人員和測試人員了解應用程序所服務的領域

避免對用戶行為或應用程序的行為作出隱含的假設，應該確定對服務端狀態做了那些假設，並實現必要的邏輯來驗證這些假設是否得到滿足


{{< line_break >}}

# Information Disclosure vulnerabilities

{{< line_break >}}

訊息洩漏是指網站無意中透漏敏感資訊，例如其他用戶的數據

一些基本例子：

- robots.txt 文件透露目錄列表
- 錯誤訊息中提到資料庫表或列的名稱
- 源代碼中應編碼 API key 或 IP 地址等


{{< line_break >}}

## 預防方法

{{< line_break >}}

確保參與製作者都充分了解那些是敏感資訊

盡可能使用通用錯誤訊息

Prod 環境需禁用任何調試或診斷功能

確保理解使用的任何第三方技術的配置和安全影響，禁用不需要的功能和設置


{{< line_break >}}

# Access Control

{{< line_break >}}

訪問控制 (或授權 Authorization) 是誰對什麼可以執行的行動或訪問的約束

在網路背景下，訪問控制依賴於認證和會話管理

- 認證可以識別用戶是誰

- 會話管理確認了那些後續的 HTTP 請求是由同個用戶提出

- 訪問控制決定了用戶是否被允許執行他們試圖執行的動作

{{< line_break >}}

從使用者觀點來看，訪問控制能分成下列幾種

- Virtual access controls
    - 限制使用者能否使用該功能

- Horizontal access controls
    - 限制使用者能否存取該資源，像是用戶能存取自己的資料但不能看到其他人的資料

- Context-dependent access controls
    - 與環境相關的訪問控制，根據應用程序狀態或用戶與他的互動來限制功能和資源的訪問，例如網站可能會阻止用戶在付款後修改其購物車的內容


{{< line_break >}}

## Insecure direct object reference (IDOR)

{{< line_break >}}

不安全的直接對象引用是訪問控制漏洞的一個子類別

當應用程序使用用戶提供的輸入來直接訪問對象，而攻擊者可修改輸入以獲得未經授權的訪問時，就會出現 IDOR

{{< line_break >}}

## 預防方法

{{< line_break >}}

除非一個資源打算被公開訪問，否則默認拒絕訪問

在可能情況下，使用單一個應用程序範圍內的機制來執行訪問控制

徹底審計和測試訪問控制，以確保他們按設計工作


{{< line_break >}}

# File upload vulnerabilities

{{< line_break >}}

文件上傳漏洞是指服務器允許用戶上傳文件，但沒對其進行充分的驗證，可能被用來上傳任意的、有潛在危險的文件

例如：

- 在最壞的情況下，文件類型沒有被正確的驗證，而服務器允許某些類型的文件作為代碼執行，攻擊者有可能上傳控制代碼文件從而完全控制服務器
- 如果文件名沒有得到正確驗證，攻擊者也可能上傳一個相同名稱的文件覆蓋關鍵文件
- 如果不能確保文件的大小在預期範圍內，也可能實現某種形式的 DoS 攻擊，即攻擊者填滿硬碟空間

{{< line_break >}}

## 預防方法

{{< line_break >}}

根據白名單限制附檔名

確保文件名不包含任何可能被解釋為目錄或遍歷序列的字符串 (…)

重命名上傳的文件，避免與現有文件覆蓋碰撞

在文件被完全驗證前，不要將文件上傳到服務器的永久文件系統

盡可能地使用一個既有的框架來預處理文件上傳，而不是試圖編寫你自己的驗證機制


{{< line_break >}}

# Server-side request forgery (SSRF)

{{< line_break >}}


服務器端請求偽造 (SSRF) 是一個網路安全漏洞，他允許攻擊者誘導服務器端應用程序向一個非預期的位置發出請求

在一個典型的 SSRF 攻擊中，攻擊者可能會使服務器連接到組織的基礎設施中僅有的內部服務，其他情況或也可能迫使服務器連接到任意的外部系統洩漏敏感資訊


{{< line_break >}}

# XML external entity (XXE) injection

{{< line_break >}}

XML 外部實體注入是一個網路安全漏洞，他允許攻擊者干擾應用程序對 XML 數據的處理

{{< line_break >}}

## 預防方法

{{< line_break >}}

幾乎所有的 XXE 漏洞都是因為應用程式的 XML 解析庫支持潛在的危險的 XML 特性，而應用程式並不需要這些特性，防止 XXE 攻擊最簡單有效的方法是禁用這些功能

{{< line_break >}}
