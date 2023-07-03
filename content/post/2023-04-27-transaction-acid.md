---
title: 資料庫的 Trasaction Isolation Level 與 ACID
subtitle: 
date: 2023-04-27 12:00:00
tags: ["Fundamentals of Database Engineering"]
---

{{< line_break >}}

開始前想先介紹一個 Youtube 上一個不錯的頻道：

https://www.youtube.com/@hnasr

裡面有許多軟工尤其是後端相關的知識，初階到進階都有，而且作者更新還滿頻繁的，相當值得訂閱

作者也有經營 Medium 和 Podcast，還有在 Udemy 開設課程，而我就買了 Udemy 的這門課 [Fundamentals of Database Engineering](https://www.udemy.com/course/database-engines-crash-course/)

資料庫的文章可能都會以這門課程學習到的內容展開，有興趣的話也可以看看這門課 (從 Youtube 頻道上的課程連結都固定會有優惠價格)


{{< line_break >}}

# ACID

{{< line_break >}}

維基百科是這樣說：
> ACID 是資料庫管理系統 (DBMS) 在寫入或更新資料的過程中，為保證交易（transaction）是正確可靠的，所必須具備的四個特性

ACID 四個特性分別是：

- Atomicity 不可分割性
- Isolation 隔離性
- Consistency 一致性
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

# Atomicity 不可分割性

{{< line_break >}}


- transaction 內的 queries 必須都成功，中間有失敗應做 rollback



{{< line_break >}}

# Isolation 隔離性

{{< line_break >}}


- 在交易中時，我們是否想要看到其他交易的 DB 改動？看你想不想看到，transaction Isolation 提供各種等級的交易隔離，協助你在多個交易同時運行時，不要得到非預期的結果 (race condititon)



{{< line_break >}}

### Read Phenomena

{{< line_break >}}

1. `Dirty Reads`

讀到還沒有 commit 的值

範例：

DB： X = 0

- Transaction A：寫入 X = 10

- Transaction B：讀取 X 值，X = 10

- Transaction A：rollback X，X 還原回 X = 0

B 讀到 X=0，但 X = 10 從來沒真的寫入 DB 裡


{{< line_break >}}

2. `Non-repeatable reads`

重複讀相同位置的值卻得到不同結果

範例：

DB： X = 0

- Transaction A：讀取 X 值，X = 10

- Transaction B：更新 X 值，X = 1，並 COMMIT

- Transaction A：讀取 X 值，X = 1

A 兩次讀取 X 值，但卻得到不同值


{{< line_break >}}

3. `Phantom reads`

重複讀，但得到不同的結果集

範例：

- Transaction A：讀取 age 10 ~ 20 範圍人的值

- Transaction B：新增一筆值高中生，並 COMMIT

- Transaction A：再次讀取 age 10 ~ 20 範圍的人，多了一筆值


{{< line_break >}}

4. `Lost updates`

更新值，但最後存的不是預期的值

範例：

X = 10

- Transaction A：X = X + 10

- Transaction B：X = X + 5，並 COMMIT

- Transaction A：COMMIT

最後 X = 15，A 預期應該要是 20

{{< line_break >}}

### Isolation Level

{{< line_break >}}

1. `Read uncommitted`

沒有 Isolation，沒有 commit 的變動都可見

{{< line_break >}}

2. `Read committed`

只有 commit 的變動才可見，許多 DB 預設 Read committed Isolation level

{{< line_break >}}

3. `Repeatable Read`

確保 query read a row，該 row 在交易中都不會有變動

{{< line_break >}}

4. `Snapshot`

交易中看到的值，都是在該交易開始時當下的 DB 狀態

{{< line_break >}}

5. `Serializable`

確保多個交易同時讀寫得到的結果會跟交易依次序執行的結果一致，犧牲 concurrency

{{< line_break >}}


### Isolation Level vs read phenomena

{{< line_break >}}

{{< figure src="/img/aws/db-isolation.png" >}}

{{< line_break >}}

- 每個 DB 實作 Isolation Level 的方法會有不同，像是 lost update 可能有下面兩種實作
    - Pessimistic 消極：使用 row level locks, table lock, page locks 避免 lost updates
    - Optimistic 樂觀：沒有鎖，但偵測交易期間值是否有變動，有則交易 rollback 視作失敗
- Postgres 實作 Repeatable Read isoaltion level 是使用 snapshot 方法，在 postgres 使用 Repeatable Read level 不會有 phantom reads
- Serializable 常以 optimistic 方法實作，但你也可以使用 "SELECT FOR UPDATE" 來達到 Serializable



{{< line_break >}}

# Consistency 一致性

{{< line_break >}}

- 由使用者定義，table schema、referential integrity (foreign keys)，資料庫修改數據必須滿足定義好的規則
- atomicity, isolation 特性確保 consistency 成立
- 如果一個 transaction commit 修改，一個新的 trasaction 是否能馬上看到該變動？
    - NoSql 多是 Eventual consistency


{{< line_break >}}

# Durability 持久性

{{< line_break >}}

- 保證提交的資料內容寫入 non-voliatile storage，即使系統 crash 重啟仍保存
- 達成 Duratbility 的技術
    - WAL - Write ahead log：所有的修改在生效之前都要先寫入log文件中
    - Asynchronous snapshot：資料寫在 RAM，之後在非同步的一次存到 disk
    - AOF：類似WAL
- OS Cache
    - OS 寫入請求經常會先存在 OS cache，再累積 batch 寫入 disk，若在還沒真的寫入硬碟系統掛掉，就會造成資料遺失
    - fsync 命令強制寫入一定要存到 disk 才算成功，但也因此降低 commit 速度

{{< line_break >}}