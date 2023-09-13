---
title: Pytube 源碼分析筆記
subtitle: 
date: 2023-09-09 12:00:00
tags: ["Python"]
---

源碼分析筆記系列的文章我想找一些程式碼不算多且有名的 Github Repo，研究其程式碼架構和設計，希望能從中學到一些設計想法，提升自己的能力

不過我不可能記下全部細節，所以只會寫下我覺得有趣的地方

這次找的是 [Pytube](https://github.com/pytube/pytube) 這個 Python 庫，Pytube 的功用是用來下載 Youtube 影片

<!--more-->
{{< line_break >}}
# 範例程式碼
{{< line_break >}}

要學習使用一個新的 Python 庫，我想第一步就是先看教學文件裡的範例程式碼，這能讓我們大概了解這個程式庫的功用以及基本用法

```python
>>> from pytube import YouTube


>>> YouTube('https://youtu.be/9bZkp7q19f0').streams.first().download()


>>> yt = YouTube('http://youtube.com/watch?v=9bZkp7q19f0')
>>> yt.streams
... .filter(progressive=True, file_extension='mp4')
... .order_by('resolution')
... .desc()
... .first()
... .download()
```

Pytube 的功用相當單純，我們從上面的程式碼知道基本上使用者會透過 Youtube 物件來做影片下載操作


{{< line_break >}}
# 關於流式接口 (Fluent Interface)
{{< line_break >}}

範例程式碼的方法呼叫可以看到是類似這樣 boo().foo()，不像一般的呼叫方式是一個包一個 foo(boo())

其實這樣的呼叫方法有個稱呼，叫做流式接口，流式接口寫法主要的好處是程式碼讀起來簡潔

下面是Python範例：
```python
class Msg:
    def __init__(self, msg):
        self.msg = msg
        self.receiver = ""
    
    def to(self, receiver):
        self.receiver = receiver
        return self

    def send(self):
        print(f"Send {self.msg} to {self.receiver}")
        return self

Msg("hello").to("Mary").send()
```

Msg("hello").to("Mary").send() 這段看起來真的整潔好懂

實作的關鍵就是return self，回傳物件自己本身

另外在這有讀到建議是回傳一個新的物件而非self (https://stackoverflow.com/questions/37827808/fluent-interface-with-python)，用意是防止從別的變數改動到原物件

{{< line_break >}}
# class StreamQuery(Sequence)
{{< line_break >}}

Sequence 是 collections.abc 的抽象類，序列是一個有順序的，可以按位置獲取的元素的集合

```python
class C(Sequence):                      # Direct inheritance
    def __init__(self): ...             # Extra method not required by the ABC
    def __getitem__(self, index):  ...  # Required abstract method
    def __len__(self):  ...             # Required abstract method
    def count(self, value): ...         # Optionally override a mixin method
```

{{< line_break >}}

Youtube("url").streams 返回的是 StreamQuery 類型

StreamQuery 是用來 query media stream 用的介面，它將 fmt_streams 封裝起來，並使用流式接口作法讓我們能方便操作 fmt_streams，它提供像是 filter、sort 等方法讓我們能過濾不需要的 fmt_streams 和進行排序等

fmt_streams 是 Stream 物件串列，一個 Stream 物件代表一種型式的影像流，這裡的型式像是編碼不同或解析度不同等

```python
>>> yt.streams
[<Stream: itag="18" mime_type="video/mp4" res="360p" fps="30fps" vcodec="avc1.42001E" acodec="mp4a.40.2" progressive="True" type="video">,
<Stream: itag="22" mime_type="video/mp4" res="720p" fps="30fps" vcodec="avc1.64001F" acodec="mp4a.40.2" progressive="True" type="video">,
<Stream: itag="137" mime_type="video/mp4" res="1080p" fps="30fps" vcodec="avc1.640028" progressive="False" type="video">,
...
<Stream: itag="250" mime_type="audio/webm" abr="70kbps" acodec="opus" progressive="False" type="audio">,
<Stream: itag="251" mime_type="audio/webm" abr="160kbps" acodec="opus" progressive="False" type="audio">]
```

{{< line_break >}}

我們重新看一下範例程式碼
```python
>>> yt = YouTube('http://youtube.com/watch?v=9bZkp7q19f0')
>>> yt.streams
... .filter(progressive=True, file_extension='mp4')
... .order_by('resolution')
... .desc()
... .first()
... .download()
```

這裡說詳細點的話，我們透過 Youtube 物件分析影片資訊取得多個型式的影像流，使用 filter 過濾不需要的影像流，再進行解析度排序，使用 first() 取得 fmt_streams[0] 第一筆影像流，並執行該影像流的下載動作

{{< line_break >}}
# class Stream
{{< line_break >}}

Stream 初始化：
```python

class Stream:
    """Container for stream manifest data."""

    def __init__(
        self, stream: Dict, monostate: Monostate
    ):
        """Construct a :class:`Stream <Stream>`.

        :param dict stream:
            The unscrambled data extracted from YouTube.
        :param dict monostate:
            Dictionary of data shared across all instances of
            :class:`Stream <Stream>`.
        """
```
{{< line_break >}}

Stream 的 download 方法

```python
def download(
        self,
        output_path: Optional[str] = None,
        filename: Optional[str] = None,
        filename_prefix: Optional[str] = None,
        skip_existing: bool = True,    # 若檔案已存在是否跳過
        timeout: Optional[int] = None,
        max_retries: Optional[int] = 0
    ) -> str:
```

底層會打 HTTP GET API 帶 Range=bytes={start}-{stop_pos} header 取得影片部分資料，分段下載並輸出成檔案


{{< line_break >}}
# 關於 Monostate Pattern (Borg pattern)
{{< line_break >}}

我們想要在多個物件裡分享相同的屬性資料時，可以使用 Monostate Pattern，用途有點類似 Singleton 單例模式

Pytube 的 Monostate class
```python
class Monostate:
    def __init__(
        self,
        on_progress: Optional[Callable[[Any, bytes, int], None]],
        on_complete: Optional[Callable[[Any, Optional[str]], None]],
        title: Optional[str] = None,
        duration: Optional[int] = None,
    ):
        self.on_progress = on_progress
        self.on_complete = on_complete
        self.title = title
        self.duration = duration
```

Monostate 裡存有我們想要多個 Stream 物件共用的方法和屬性，在生成 Stream 物件時我們都以相同的 Monostate 物件進行初始化就可達成

{{< line_break >}}
# CLI 命令
{{< line_break >}}

許多程式庫會提供 Command Line 能讓使用者直接在 shell 下命令使用方法，而不需使用者自己在寫程式引用程式庫呼叫方法，Pytube CLI：https://pytube.io/en/latest/user/cli.html


Pytube CLI 實作上是建立一個獨立的 cli.py 檔案，有自己的 main() 函式，main 裡使用 argparse 分析指令參數，在一指令呼叫對應的方法

```python
def main():
    """Command line application to download youtube videos."""
    parser = argparse.ArgumentParser(description=main.__doc__)
    args = _parse_args(parser)
    ...

if __name__ == "__main__":
    main()
```

CLI 安裝 Pytube 是透過 setuptool build，在 setup.py 我們可以看到這樣的宣告：

```python
setup(
    name="pytube",
    version=__version__,  # noqa: F821
    ...
    entry_points={
        "console_scripts": [
            "pytube = pytube.cli:main"],},
    ...
)
```

{{< line_break >}}
# 自定義 Exception
{{< line_break >}}

Pytube 自建一個 Base pytube exception class 稱作 PytubeError，所有 Pytube 自定義的例外都繼承自此 Exception
```python
class PytubeError(Exception):
    """Base pytube exception that all others inherit.

    This is done to not pollute the built-in exceptions, which *could* result
    in unintended errors being unexpectedly and incorrectly handled within
    implementers code.
    """
```

Pytube 例外類範例：
```python
class VideoUnavailable(PytubeError):
    """Base video unavailable error."""
    def __init__(self, video_id: str):
        """
        :param str video_id:
            A YouTube video identifier.
        """
        self.video_id = video_id
        super().__init__(self.error_string)

    @property
    def error_string(self):
        return f'{self.video_id} is unavailable'
```

{{< line_break >}}

使用者可以透過引用 Pytube 這些例外類，識別對應的錯誤狀況並處理 Error Handling

```python
>>> from pytube import Playlist, YouTube
>>> from pytube.exceptions import VideoUnavailable
>>> playlist_url = 'https://youtube.com/playlist?list=special_playlist_id'
>>> p = Playlist(playlist_url)
>>> for url in p.video_urls:
...     try:
...         yt = YouTube(url)
...     except VideoUnavailable:
...         print(f'Video {url} is unavaialable, skipping.')
...     else:
...         print(f'Downloading video: {url}')
...         yt.streams.first().download()
```


{{< line_break >}}
# 結論
{{< line_break >}}

從 Pytube 源碼，我們學習到
- 流式接口封裝
- Monostate Pattern
- 分段下載檔案
- CLI 工具實作與安裝
- 自定義 Exception


{{< line_break >}}
