---
title: 資料庫的 Index 重點筆記
subtitle: 
date: 2023-08-22 12:00:00
tags: ["Fundamentals of Database Engineering"]
---


# 重要名詞


- Table
    - 邏輯表

- Row_id
    - 內部系統維護
    - 在某些系統 mysql innoDB 是同 primary key，但在 Postgres 是有個系統欄位 rowid (tuple id)
    - PostgreSQL 的 rowid 正確名稱是叫 ctid，形式是 tuple

- Page
    - 多筆 row 會存在一個 Logical Page 中
    - 資料庫不是讀單列，而是讀一個或多個 page，在一次 IO 中取得多筆列資料
    - Postgres Page 的大小為 8KB，MySQL Page 的大小為 16KB


<!--more-->

- IO
    - IO 是一個 read request to disk
    - 我們想盡量減小 IO 次數
    - 一個 IO 可以讀一個 page 或多個，取決於 disk partitons 或其他原因
    - 有些 IO 是從 OS cache 取得資料並非從 disk

- Heap data structure
    - Heap 是一個資料結構用來存 table 和它相關的 pages
    - 掃 heap 相當耗時，因此我們需要 index 幫助我知道要到 heap 哪個部分讀資料

- Index
    - index 一個資料結構指出 heap 裡位置
    - 可以對一個欄位或多個欄位設定 index
    - Index 告訴我們正確需要讀哪一個 page，如此可以省去掃描 page 的時間
    - Index 也是存成 page，要讀取也是費 IO
    - index 常用的資料結構是 b-trees


{{< line_break >}}
# Clustered Index v.s. Non-clustered Index
{{< line_break >}}

在 SQL 中，Index 是用來加速查詢操作的數據結構

Clustered index 和 Non-clustered index 是兩種常見的索引類型

- Clustered Index
    - 一個資料表中只能有一個 Clustered Index
    - 資料表的資料要按照 Clustered Index 的排序方式存儲的
    - 查詢 Clsutered Index 時，查詢效率通常較高

{{< line_break >}}

- Non-Clustered Index
    - 一個資料表中可以有多個 Non-Clustered Index
    - Non-Clustered Index 不會重新排列數據表中的實際數據行，而是在索引中保留指向實際數據行的指針
    - Non-Clustered Index 根據索引鍵的值進行排序，但實際數據表中的數據行順序可能是不連續的
    - 查詢 Non-Clustered Index 時，需要先查詢索引，然後再通過指標找到實際資料，效率相對較低

{{< line_break >}}
# Primary Index v.s. Secondary Index
{{< line_break >}}

- Primary Index (Primary Key)
    - Primary Index 是一個唯一性索引，用於識別資料表中的每一筆紀錄
    - 一個資料表只能有一個 Primary Index
    - Primary Index 的值不能為 NULL，確保每一筆紀錄都有唯一的識別值
    - Primary Index 通常會自動建立 Clustered Index (MySQL)

{{< line_break >}}

- Secondary Index (Secondary Key)
    - Secondary Index 也稱為 Non-Primary Index 或 Non-Unique Index
    - 一個資料表可以有多個 Secondary Index
    - Secondary Index 的值可以重複，不一定是唯一的
    - Secondary Index 通常會自動建立 Non-Clustered Index
    - 總結來說，Secondary Index 就是一個用於查詢加速的非叢集索引，它與 Non-Clustered Index 的概念是相關的。當你需要根據不同的查詢需求來加速資料表的查詢操作時，可以考慮建立不同的 Secondary Index


{{< line_break >}}
# MySQL v.s. PostgreSQL
{{< line_break >}}

MySQL 預設自動將表中的 Primary Key 建立 Clustered Index

實際表資料會依 Primary Key 進行排序，因此一般建議 Primary Key 應使用增長的數值，若是像 UUID 這樣的隨機值會需要隨機存取 Page，會沒能善用緩存與增加寫入磁碟次數，造成插入資料緩慢 ([UUIDs are Bad for Performance in MySQL - Is Postgres better? Let us Discuss](https://www.youtube.com/watch?v=Y5mWz4vK10A)) ( 可以使用UUID_TO_BIN()緩解此問題 )

MySQL 的 Non-Clustered Index 設計是存 Clustered Index 指針，先取得對應 Clustered Index 才能對應到真正的資料列，因此使用 Non-Clustered Index 搜尋會有需要查詢兩次 Index tree：先查詢 Non-Clustered Index -> Clustered Index -> Data Page


PostgreSQL 則沒有 Clustered Index，全部都是 Non-Clustered Index，Non-Clustered Index 存有 row id (ctid tuple) 代表實體資料位置，不需要像 MySQL 的 Non-Clustered Index 多次存取 Index

但 PostgreSQL 的 Non-Clustered Index 更新寫入比較耗時，可以看一下這篇文章： [Why Uber Engineering Switched from Postgres to MySQL](https://www.uber.com/en-TW/blog/postgres-to-mysql-migration/)

簡單說的話，因為 PostgreSQL 在更新 row 資料時，ctid 也會更新，因此若是使用者更新了其中一個 Non-Clustered Index 的欄位值，這表的所有的 Index 都必須去更新該 row 的 ctid，不像 MySQL 只要更新該 Non-Clustered Index 就好，該篇還有提到多個為什麼從 PostgreSQL 改用 MySQL 理由，推薦看看


這裡不是要說 MySQL 效率一定優於 PostgreSQL，還是要考量使用的需求狀況，以下是個人看了一些文章的心得：若是要處理複雜的查詢這部份是 PostgreSQL 提供比較多的查詢和分析方法，像是針對 JSON 的支持 PostgreSQL 就比較豐富，複雜的查詢表現也比較好，另外 PostgreSQL 也提供更多的數據類型和更強大的事務控制和約束功能，確保數據的一致性，而 MySQL 是在寫入密集型的負載表現較好

我個人是頃向優先使用 PostgreSQL，理由是因為我工作較常用 PostgreSQL 對它比較熟 XD


也推薦可以看這篇文章，了解 PostgreSQL 的一些弱項：[The Part of PostgreSQL We Hate the Most](https://ottertune.com/blog/the-part-of-postgresql-we-hate-the-most/)


{{< line_break >}}
# 資料庫的 Query Plan (說明以 PostgreSQL 為主)
{{< line_break >}}


我們可以使用 EXPLAIN + SQL 指令來查看該 SQL 指令在 PostgreSQL 是如何執行找到資料

假設我們現在有個 grades 表
內含欄位
- id (Primary Key)
- name
- g (有設定 Index：g_idx)


{{< line_break >}}

使用 EXPLAIN 會看到類似這樣的顯示：Seq Scan on grades (cost=0.00..289025.15 rows=12141215 width=31)

- cost：
  - 0.00：花多少 postgres unit 去取得第一個 page
  - 289025.15：預想會花的 unit 時間

- rows：預估抓多少 rows

- width：row 的大小 (byte)

{{< line_break >}}

除了 EXPLAIN 還有 EXPLAIN ANALYZE 命令可以查看 Query Plan，兩者的不同在於 EXPLAIN ANALYZE 會真的執行 SQL 指令，因此並不是估計值，而是真的運行狀況


{{< line_break >}}
## Seq Scan 全表掃描
{{< line_break >}}

- SELECT * FROM grades

{{< line_break >}}

撈取 grades 表所有資料，需要執行全表的掃瞄

PostgreSQL 能跑多個 Worker 加快掃描執行速度 

{{< line_break >}}
## Index Scan 索引掃描
{{< line_break >}}

- SELECT * FROM grades WHERE id = 10

{{< line_break >}}

Primary Key 會自動創建 index

這裡會使用 Idex Scan，在 Index Tree 取得 id = 10 資料位置，再到 Heap 去撈資料 Page，不用掃表


要注意不是 WHERE 條件設定 Index 欄位就一定會執行 Index Scan
在有些狀況資料庫會判斷條件符合的 page 過多，不如直接掃表，這麼做可以省去每次 random access table 的時間
像是：SELECT * FROM grades where id > 100 大範圍的搜索

{{< line_break >}}
## Bitmap Index Scan 位圖掃描
{{< line_break >}}

如果只選擇少量記錄，PostgreSQL 會使用 Index Scan，如果選擇大範圍記錄，PostgreSQL 將決定讀取表，但是如果讀取的記錄太多，Index Scan 效率不高，對於 Seq Scan 來說又太少，該怎麽辦呢？

PostgreSQL 會使用 Bitmap Index Scan


{{< line_break >}}

- SELECT name FROM grades WHERE g > 95

{{< line_break >}}

PostgreSQL 會將掃描 Index Tree 的結果存在一個數據塊，在依照此數據塊資料來撈取表資料
像是這樣：
0 0 0 1 0 0 1 1：如此意思可以表示要撈取 Page 0, 1, 4

{{< line_break >}}

使用 Bitmap 在使用多個索引來掃描一個表時很有幫助

{{< line_break >}}

- SELECT name FROM grades WHERE g > 95 and id < 10000

{{< line_break >}}

針對 g > 95 產生一個 bitmap，和 id < 10000 也產生一個 bitmap
兩個 bitmap 做 BitmpaAnd 計算，就可以組出新的 bitmap，而這 bitmap 就是要撈取的 Page 資料

{{< line_break >}}
## Index Only Scan
{{< line_break >}}

想搜尋的資料在 Index Tree 就有，省去存取 heap 的時間

{{< line_break >}}
- SELECT id FROM grades WHERE id = 10

{{< line_break >}}
### Non-Key Column
{{< line_break >}}

PostgreSQL 可以在創建 index 也加進一些 Non key 的欄位到 index tree 中

就假設你創建了 g 欄位的 index，然後含進 id 欄位，如此你在以分數進行搜尋取 id 時，就在 index 裡掃描就可，不需要到 table heap 拉資料

```sql
create index g_idx on grades(g) include (id); 
```

{{< line_break >}}
### 可見性


[Index-Only Scans and Covering Indexes](https://docs.postgresql.tw/the-sql-language/index/index-only-scans-and-covering-indexes)

因為 PostgreSQL MVCC 機制的實現，需要對掃描的 index 進行可見性判斷，即檢查 visibility map，可見性訊息不儲存在索引項目中，僅儲存在 heap 中，如果資料列最近被修改，就算是 Index Only Scan 還是需要存取 heap，因此與標準的索引掃描相比，不見得能得到效能優勢，但是對於很少變化的資料，使用 Index Only Scan 還是有優勢

{{< line_break >}}
