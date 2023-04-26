---
title: 如何使用Hugo在GitHub架設自己的網站
subtitle: 
date: 2023-04-24
tags: ["Hugo"]
---


紀念性的第一篇文章，就來說說是如何建立這個網站的吧！


建立網站前後我用了以下這幾個工具：
- GitHub Page
    - GitHub Page 是 GitHub 上提供的免費服務，可以讓每個帳號在 GitHub 上建立一個靜態的網站
    - 專案名稱統一要冠上 {GitHub帳號名}.github.io，這也將會是網站的 URL
- Hugo
    - Hugo 是用 Go 寫成的一個快速架站 Framework ，能快速建立一個靜態網站
- beautifulhugo
    - beatifulhugo 是 Hugo 的一個外觀主題，至於為什麼會選這個？其實沒什麼特別的理由，只是因為這主題在搜尋的第一頁，然後我比較喜歡它的 Tag 頁能直接展開列出文章

<!--more-->
{{< line_break >}}

# 流程

{{< line_break >}}
具體的流程如下：

{{< line_break >}}

1. 安裝 Hugo

{{< line_break >}}

首先到官網下載 (https://gohugo.io/installation/) Hugo

我是使用 Windows，所以選擇 Windows ，進到 Windows 安裝說明頁時一路下滑到 Prebuilt binaries 下載最新的release (我是載 hugo_0.111.3_windows-amd64.zip 版本)

之後再將檔案解壓縮到 C:\Hugo\bin 路徑內，並新增此路徑值到系統環境變數 PATH 裡

最後試著在 CMD 下 hogo version 指令，應該能看到安裝的 Hugo 版本資訊，證明 Hugo 有安裝與設定成功


{{< line_break >}}

2. Local 建立網站

{{< line_break >}}

先在自己電腦的環境，建立網站的資料夾

到自己想要建立資料夾的路徑後，執行以下指令來初始化 Hugo 網站的資料夾 (wovex 是我 GitHub 帳號名稱，請依自己的帳號名代換)
```
hugo new site wovex.github.io
```

執行後，會產生 wovex.github.io 資料夾，進到資料夾裡後，再執行下面指令將 beautifulhugo 主題帶入

```
git init
git submodule add https://github.com/halogenica/beautifulhugo.git themes/beautifulhugo
```

然後將 wovex.github.io/themes/beautifulhugo/exampleSite 資料夾底下的資料夾與檔案 (content, layouts, static, config.coml ) 複製貼到 wovex.github.io 路徑，有重複的檔案就直接覆蓋


最後我們就可以試著在 Local 端也就是我們自己的電腦上 build 網站，只要執行

```
hugo serve
```

從 Console 應該可以看到類似這樣的訊息：Web Server is available at http://localhost:1313/ (bind address 127.0.0.1)

到瀏覽器開啟 http://localhost:1313/ 就可以看到網站了，內容會是 beautifulhugo 的 sample 網站

hugo serve 很方便的是它會偵測資料夾變動，能及時反應任何變動，在編輯 markdown 文章時，馬上就能在瀏覽器看到結果


{{< line_break >}}

3. Deploy 到 GitHub

{{< line_break >}}

到自己的 GitHub 建立新的 Repository ，這裡一定要將 Repository 命名為 {GitHub帳號名}.github.io

{{< figure src="/img/hugo_2.png" >}}

把剛剛在 Local 建好的 wovex.github.io 資料夾照以下步驟 git push 到這新建的 Repository

```
git remote add origin git@github.com:wovex/wovex.github.io.git

git add .
git commit -m "init"

git branch -M main
git push -u origin main
```

之後點選在頁面上面的菜單 Settings -> Pages，在右邊的 Build and depolyment 地方將 Source 改成 GitHub Actions，這裡 GitHub 似乎很神的偵測到你是要建立 Hugo 網站，按下 Hugo 的 Cofigure，GitHub 就把你寫好 GitHub Actions 的 workflows yaml 檔，也就是幫你寫好 CI/CD 的程式碼了，只要按下 Commit 就好

{{< figure src="/img/hugo_4.png" >}}


之後每次你 git push 程式碼，GitHub Actions 就會自動執行 Hugo build 和 deploy


{{< figure src="/img/hugo_1.png" >}}

網站的位置會是在 https://{GitHub帳號名}.github.io


{{< line_break >}}

4. 最後的調整

{{< line_break >}}

現在的網站內容是 Sample 的文章資料，到 content/post 資料夾刪掉範例文章，新增自己的 markdown 文章

config.toml 設定檔隨自己的需求做些修改

新增方便用的 layouts/shortcodes html檔

(以上的修改詳細請直接查看https://github.com/wovex/wovex.github.io)


*剩下的就是快樂的編輯與產出文章了^^ (有空閒可能在做些美化工作)*


{{< line_break >}}
