---
title: 資料庫的 ACID
subtitle: 
date: 2023-04-27 12:00:00
tags: ["Fundamentals of Database Engineering"]
---

{{< line_break >}}

一開始前想先介紹一個 Youtube 上一個不錯的頻道：

https://www.youtube.com/@hnasr

裡面有許多軟工尤其是後端相關的知識，初階到進階都有，而且作者更新還滿頻繁的，相當值得訂閱

作者也有經營 Medium 和 Podcast，還有在 Udemy 開設課程，而我就買了 Udemy 的這門課 [Fundamentals of Database Engineering](https://www.udemy.com/course/database-engines-crash-course/)

資料庫的文章可能都會以這門課程學習到的內容展開，有興趣的話也可以看看這門課 (從 Youtube 頻道上的課程連結都固定會有優惠價格)


{{< line_break >}}

# ACID

{{< line_break >}}

維基百科是這樣說：
> ACID 是資料庫管理系統 (DBMS) 在寫入或更新資料的過程中，為保證交易（transaction）是正確可靠的，所必須具備的四個特性

而這四個特性分別是：

- Atomicity 不可分割性
- Consistency 一致性
- Isolation 隔離性
- Duration 持久性

<!--more-->

{{< line_break >}}

# Transaction

{{< line_break >}}

Transaction 交易是

- 一系列的資料庫 query 指令
- 被視為一單位的動作，邏輯上指令間不可分割，不是全部完成，就是全部失敗
    - 舉例：轉帳，A 轉給 B 100 元，步驟一 A 先扣 100 元，步驟二 B 加 100 元，若只有步驟二失敗，就白白蒸發 100 元了

{{< line_break >}}

Transaction 的生命週期
- Transaction BEGIN：交易開始
- Transaction COMMIT：交易結束，真正將資料寫入
- Transaction ROLLBACK：交易倒回，當交易中發生不預期的錯誤，就會觸發 ROLLBACK 返回之前所做的更改

{{< line_break >}}



{{< notice tip >}}
只有 Read query 時有必要使用 Transaction 嗎？

只有 Read 使用 Transaction 包起來是很正常的做法，當你想要確保每個 Read 都是讀到同時間的 snapshot 是很有用的
{{< /notice >}}


{{< line_break >}}