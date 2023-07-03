---
title: asycio 工作原理淺談
subtitle: 
date: 2023-07-02 12:00:00
tags: ["Python"]
---



asyncio 是 Python 內建的module，在 Python 3.4 時加入

是一種單 thread 的設計，它靠著 cooperative multitasking (協同運作式多工) 讓我們能多個工作併發處理 (concurrent)

協同運作式多工相對於搶佔式多工（Preemptive multitasking），協作式多工要求每一個運行中的程式，定時放棄自己的執行權利，告知作業系統可讓下一個程式執行

多線程採用的是搶佔式多工，多線程是由作業系統做排程，線程執行任務途中會被外力(作業系統)中斷改排其他線程執行，而協同運作式多工不由作業系統排程，在任務執行時遇到需要等待回應的狀況，會放棄執行權，改執行別的任務，而原任務在等到回應後再繼續執行

這篇文章主要是大概介紹 asyncio是如何做到 cooperative multitasking

<!--more-->

在介紹 asyncio 是怎麼做到 cooperative multitasking，首先需要知道什麼是Coroutine與Event Loop


{{< line_break >}}
# 什麼是Coroutine (協程)
{{< line_break >}}

在介紹 Corotines 之前，如果先知道 Python 的 Generator 或許比較好理解，因為兩者的概念其實相當類似

認識 Python Generator 首先要知道 yield 這個語法用意

直接看範例比較好理解用途


```python
def genNumber():
    x = 0
    for _ in range(3):
        yield x
        x = x + 1
    return 5

for n in genNumber():
    print(n)   # 會印出 0  1  2

z = genNumber()
print(z)         # 印出 generator object genNumber at 0x01CCDB18
print(next(z))   # 印出 0
print(next(z))   # 印出 1
print(next(z))   # 印出 2

try:
    print(next(z))   # 會 raise StopIteration
except StopIteration as e:
    print(e.value)  # 印出 5
```


上面的範例，有一點需要先注意的是 z = genNumer() 這行其實沒有運行我們寫的程式碼，它只是實例化一個 generator object (所以 print(z) 是印出 <generator object ...>)

genNumber() 是 generator object，當跑這物件的__next__() 時才會真的執行我們寫的程式碼  ( next(obj)會跑obj物件的__next__() 函式 )

而每一次的執行如果遇到 yield，會停在 yield 這行，並回傳 yield 後帶的值，且下次的__next__() 會又接續上次的斷點繼續執行，直到程式碼跑完這時就會 raise StopIteration



那為什麼說 Generator 跟 Coroutine 有關呢？

看看下面的例子

```python
async def func():
    return 1

coro = func()
print(coro)  # 印出 coroutine object func at 0x01C9E6E8
try:
    coro.send(None)
except StopIteration as e:
    print(e.value)  # 印出 1
```

不覺得整體結構還滿像的嗎？

Coroutine 的行為其實可以類比 Generator


總結來說Corotine同Generator有這幾個特性

- 函式可以暫停，並且保存當前運行狀態，恢復時能從保存狀態的地方執行

- 可以向暫停的地方傳入值，如此可以做到多個任務間的傳遞



另外，要跟 Coroutine 互動，除了使用 send() 方法還有 throw()，這是讓 Coroutine 執行噴錯


```python
import asyncio

async def f():
    try:
        while True: await asyncio.sleep(0)
    except asyncio.CancelledError:
        print('cancel')
    else:
        return 111

coro = f()
coro.send(None)
coro.throw(asyncio.CancelledError)  # 印出cancel後raise StopIteration
```

{{< line_break >}}
# async 與 await 語法
{{< line_break >}}

async 關鍵字是用來宣告函式為 coroutine 用

await 後必須接 awaitable 的物件，awaitable 的物件為 Coroutine 或有實作__await__()方法的物件，當執行到 await 會將控制權還回 event loop，並等待到回傳值後再繼續向下執行


{{< line_break >}}
# Event Loop
{{< line_break >}}

Event loop 包裝了一些方法讓我們更方便跟 Coroutine 互動，在要處理多個 Coroutine 運行也較為簡單



Event loop 有兩種

- SelectorEventLoop：使用selectors module

- ProcatorEventLoop：for Windows， 使用 I/O Completion Ports (IOCP)


```python
async def f():
    return 1

loop = asyncio.get_event_loop()
coro = f()
loop.run_until_complete(coro)  # 會有印出 1
```


像上例，在run_until_complete()其實內部就幫我們處理了coroutine send()與catch StopIteration


{{< line_break >}}
# Task 與 Future
{{< line_break >}}

asyncio 模組裡的 Task 物件封裝 Coroutine，便於我們控制執行



Future 物件則是包裝執行狀態與結果，從 Future 物件方法來看應該能大致理解 Future 的應用

{{< line_break >}}

- `set_result(result)`：使 Future done，並設定 result 值，若Future已經done還 set_result 會有 InvalidStateError


- `set_exception(exception)`：使 Future done，並設定exception，若Future已經done還 set_exception會有 InvalidStateError

- `cancel(msg=None)`：cancel Future

- `result()`：若Future done，回傳Future result值，但若是因為set_cxception(exception)才 done的話會 raise exception，若Future被cancelled，則會有 CancelledError，若Future還沒done的話(沒set_result)則有InvalidStateError

- `done()`：若Future done 回傳True  (cancel狀態也會是True)

- `cancelled()`：若Future cancelled 回傳 True

- `add_done_callback()`：設定當Future done時要跑的callback

- `get_loop()`：回傳 Funture 綁定的 event loop



另外 Task 是繼承 Future 的，Task 就是 Coroutine 加上 Future 物件方法


{{< line_break >}}
# asyncio 運作
{{< line_break >}}


有了上面的 Coroutine 與 Event loop 的認知後，再來看 asyncio 的運作是如何做到cooperative multitasking (協同運作式多工)


asyncio 名字是指 async I/O (異步I/O)


async 意指不會阻塞當前的程式執行，在 asyncio 模組裡除了 Event loop 與 Coroutine 還有包裝 async I/O 方法 (https://docs.python.org/zh-tw/3/library/asyncio-stream.html#asyncio-streams)，Event loop 裡則使用了 selectors module 來做到能在 async I/O 執行完時做喚醒的動作


然後協程的特性是可以保存執行斷點，恢復時能繼續執行，加上協程這設計，當協程跑到 async I/O 可以先斷點改跑其他的協程，到所有協程處於 waiting，Event loop 會 call select 函式並等待 async I/O 完成的通知，當 I/O 有資料可以讀取 select 會回傳對應的 socket，asyncio 會將綁定這 I/O socket 的 Future 設定成 done，之前斷點的協程在這之前有使用 add_done_callback() 加到這個 Future 裡，當這 Future 狀態變成 done 就會 call 之前的協程，這就相當於喚醒之前執行到中途的協程繼續跑


如此搭配下，Event Loop 單線程就可以在多個協程下交錯運行，且不會被耗時的 I/O 給阻塞住了


(附註，asyncio 裡內建的異步 I/O 方法較底層，使用上較為困難，這時候可以使用其他第三方 async 函式庫，像是 aiohttp (https://docs.aiohttp.org/en/stable/) 就將asyncio的異步方法包裝讓我們更簡單做到 async 的 http 操作)


{{< line_break >}}
# 參考資料 / 推薦閱讀
{{< line_break >}}

1. https://stackoverflow.com/questions/49005651/how-does-asyncio-actually-work

{{< line_break >}}

