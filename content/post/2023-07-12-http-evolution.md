---
title: 簡述 HTTP/1 到 HTTP/3 的差異
subtitle: 
date: 2023-07-12 12:00:00
tags: ["網路"]
---


HTTP 的演化史：

HTTP/1 -> HTTP/1.1 -> HTTP/2 -> HTTP/3

{{< line_break >}}
# HTTP/1 v.s. HTTP/1.1
{{< line_break >}}

HTTP/1 在每一次的請求都必須重新建立 TCP 連線，既費時且耗頻寬，HTTP/1.1 則默認使用 Keep-alive 持久連線，可重複使用同一個 TCP 連線發送 HTTP 請求


HTTP/1 主要使用標頭中的 If-Modified-Since、Expires 來做為緩存的判斷標準，HTTP/1.1 則引入更多緩存控制機制，例如：Etag、If-Unmodified-Since、If-Match、If-None-Match，透過這些可以更優化緩存的實現


HTTP/1 客戶端發送一個請求給伺服器端，需等待伺服器回應後，才可發送第二筆請求，HTTP/1.1 新增 HTTP pipeline 機制，允許讓客戶端傳送多個 HTTP 請求，不用等待之前的請求回覆，如此能讓網頁載入速度變快，但伺服器端必須按照客戶端傳送的請求順序來回覆請求，要維持先進先出的規則，所以有可能發生隊頭阻塞(HOL blocking)的狀況，即因前一筆請求回應耗時，後面的請求都必須延遲回應，另外因為先進先出的規則嚴格難實作，許多瀏覽器都是預設關閉此功能的 (瀏覽器實踐上，為了維持一定的載入速度，會創建多個 TCP 連線來達到平行載入，瀏覽器通常會限制對一個域名同時只能有一定數量的 TCP 連線)

<!--more-->

{{< line_break >}}
# HTTP/1.1 v.s. HTTP/2
{{< line_break >}}

HTTP/2 引入 HTTP Stream 能在單一 TCP 連線傳送多個 frame，每個 frame 有一個識別碼來識別是屬於哪個 stream，frame 彼此獨立且因為有識別碼因此能重新組合資料，不需按照順序傳送或接收，如此可以達到並行發送多個請求且請求之間互不影響，解決 HTTP/1.1 pipeline 的 HOL blocking 問題


HTTP/2 加入 Push 推送機制，能讓伺服器端主動推送資源到客戶端，此種傳輸省去客端請求的時間，因此相當快速


HTTP/2 使用一種稱為 HPACK 的更進階壓縮方法，可以消除 HTTP 標頭封包中的多餘資訊


{{< line_break >}}
# HTTP/3
{{< line_break >}}

HTTP/2 的多路複用解決應用層的 HOL blocking 問題，但並沒有解決 TCP 層的 HOL blocking 問題，TCP 是有序字節流，若發生有丟失，TCP 會卡住等待重傳，當網絡繁忙時丟包機率高，多路複用會受到很大限制。HTTP3 改採用 UDP 作爲傳輸層協議，重新實現了無序連接，並在此基礎上通過有序的 QUIC Stream 提供了多路複用


{{< line_break >}}
# 參考
{{< line_break >}}

- [HTTP/1、HTTP/1.1 和 HTTP/2 的區別](https://www.explainthis.io/zh-hant/interview-guides/browser/http1.0-http1.1-http2.0-difference)

- [深入剖析 HTTP3 協議](https://www.readfog.com/a/1661736376533618688)

{{< line_break >}}