---
title: redis-py 源碼分析筆記
subtitle: 
date: 2023-09-10 12:00:00
tags: ["Python"]
---


這次來研究 [redis-py](https://github.com/redis/redis-py) 這個 Python 庫，redis-py 主要功用是提供 Redis 客戶端的介面讓使用者操作 Redis



{{< line_break >}}
# 範例程式碼
{{< line_break >}}

先來看看教學文件裡的範例程式碼

```python
>>> import redis
>>> r = redis.Redis(host='localhost', port=6379, db=0)
>>> r.set('foo', 'bar')
True
>>> r.get('foo')
b'bar'
```

使用者使用 redis.Redis 物件來對 Redis 服務下指令

<!-- 針對 redis-py 我個人比較好奇連線管理的作法，因此先來看看這部分 -->


<!--more-->

{{< line_break >}}
# Redis class 與 Command class 的關係
{{< line_break >}}

我們先來研究一下範例程式碼的這行 r.get('foo') 的呼叫路徑吧


```python
# Redis 類繼承了 RedisModuleCommands, CoreCommands, SentinelCommands 這三個類
class Redis(RedisModuleCommands, CoreCommands, SentinelCommands):
    def execute_command(self, *args, **options):
        """Execute a command and return a parsed response"""


# 來看一下 CoreCommands 類
class CoreCommands(
    ACLCommands,
    ClusterCommands,
    DataAccessCommands,
    ManagementCommands,
    ModuleCommands,
    PubSubCommands,
    ScriptCommands,
    FunctionCommands,
    GearsCommands,
):
    ...


# 當中的 DataAccessCommands 再繼承了其他類，而其中的 BasicKeyCommands 正定義了 get(name: KeyT) 方法
class DataAccessCommands(
    BasicKeyCommands,
    HyperlogCommands,
    HashCommands,
    GeoCommands,
    ListCommands,
    ScanCommands,
    SetCommands,
    StreamCommands,
    SortedSetCommands,
):
    ...


class BasicKeyCommands(CommandsProtocol):
    def get(self, name: KeyT) -> ResponseT:
        """
        Return the value at key ``name``, or None if the key doesn't exist

        For more information see https://redis.io/commands/get
        """
        return self.execute_command("GET", name)  # Redis class 定義了 execute_command 函式

```

這裡的類關係是 Mixin，Mixin 本身不是 "is-a" 的關係，它的設計是要讓一個類別添加上 Mixin 類的特性，在 Python 裡可以用多重繼承來實現

{{< line_break >}}
## 關於 Mixin
{{< line_break >}}

用下面的例子來看 Mixin 的做法

```python
class PrintableMixin:
    def print_info(self):
        print(f"Name: {self.name}")


class ComparableMixin:
    def __eq__(self, other):
        return self.name == other.name


# 我們可以使用這兩個 Mixin 類別來擴展 Person 類別的功能
class Person(PrintableMixin, ComparableMixin):
    def __init__(self, name):
        self.name = name
```

總的來說，繼承是一種建立類別之間階層結構的方式，允許子類別重用父類別的行為，並且可以進一步擴展，Mixin 則是一種向類別中動態添加功能的機制，通常用於支援類別之間的共享和重用程式碼，並提供了一種彈性的方式來添加新的功能

{{< line_break >}}
## typing.Protocol 用途
{{< line_break >}}

BasicKeyCommands 繼承 CommandsProtocol，然後 CommandsProtocol 繼承 typing 的 Protocol

```python
from typing import Protocol

class CommandsProtocol(Protocol):
    connection_pool: Union["AsyncConnectionPool", "ConnectionPool"]

    def execute_command(self, *args, **options):
        ...


class BasicKeyCommands(CommandsProtocol):
```

{{< line_break >}}

繼承 typing 的 Protocol 類有點類似在建立一個 abstract class

有點難描述功用，我們直接看用法

```python
from typing import Protocol, runtime_checkable

@runtime_checkable
class Runnable(Protocol):
    def run(self):
        pass

def someone_run(someone: Runnable):
    someone.run()

class Employee:
    def run(self):
        print("emplyee run")

p = Employee()
print(isinstance(p, Runnable))   # True，有掛 runtime_checkable 的話，才可以在執行時 runtime 的使用 isinstance 檢查是否同為 Runnable 類
someone_run(p)
```

注意到 Employee 不需明確的繼承 Runnable，只要定義 Runnable 的方法和屬性就可以算是 Runnable 類，這就是 typing Protocol 的功用



{{< line_break >}}
# ConnectionPool 管理連線
{{< line_break >}}

每個 Redis 物件內有含一個 ConnectionPool 物件，ConnectionPool 功用是負責管理和重複利用多筆連線


ConnectionPool 初始化要提供連線的類型、連線上限和連線的設定參數
```python
def __init__(
        self, connection_class=Connection, max_connections=None, **connection_kwargs
    ):
```

{{< line_break >}}

ConnectionPool 有幾個跟管理連線比較相關的屬性需要知道：
- self._created_connection：int，現在生成的連線數
- self._available_connections：list，存有現在可用的連線
- self._in_use_connections：set，存有現在正在使用的連線
- self.pid = os.getpid()：process id
- self._fork_lock：threading.Lock()
- self._lock：threading.Lock()

{{< line_break >}}

每次執行 Redis execute_command 函數時會先從 ConnectionPool get_connection，如果 self._available_connections 有可用連線就會拿裡面的連線來用，沒有可用的連線時，則會呼叫 make_connection 函數產生一個新的 Connection 類 (會使用 ConnectionPool 初始化時給的 connection_kwargs 來初始化 Connection 類)


{{< line_break >}}

ConnectionPool 在多線程下管理連線要注意 Race Condition，因此在 get_connection 時會要用 self._lock 保護，防止同時改動造成非預期結果


self._lock 是在 get_connection() 時 check_pid 後會鎖住，self._in_use_connections 加上生成的 Connection 後在釋放鎖，另外也會在 release_connection() 和disconnect() 使用 self._lock 鎖住 (ConnectionPool 的 release 函式是連線轉為 idle 狀態但仍存在 pool 的 self._available_connections 中，disconnect 函式則是會將 pool 裡的 in-use 和 available connection 做斷線)，簡單的說就是在加減連線數的程式碼都需要限制同時只有一個線程在跑這段程式碼



以下是 ConnectionPool 的 get_connection 函數程式碼
```python
def get_connection(self, command_name, *keys, **options):
    "Get a connection from the pool"
    self._checkpid()
    with self._lock:
        try:
            connection = self._available_connections.pop()
        except IndexError:
            connection = self.make_connection()
        self._in_use_connections.add(connection)

    try:
        # ensure this connection is connected to Redis
        connection.connect()
        # connections that the pool provides should be ready to send
        # a command. if not, the connection was either returned to the
        # pool before all data has been read or the socket has been
        # closed. either way, reconnect and verify everything is good.
        try:
            if connection.can_read():
                raise ConnectionError("Connection has data")
        except (ConnectionError, OSError):
            connection.disconnect()
            connection.connect()
            if connection.can_read():
                raise ConnectionError("Connection not ready")
    except BaseException:
        # release the connection back to the pool so that we don't
        # leak it
        self.release(connection)
        raise

    return connection
```

{{< line_break >}}

self._fork_lock 則是在 get_connection 時都會檢查一下目前的 pid 是否與當初存的 self.pid 相同 (即 check_pid 函數)，如果不同會判斷現在是在 fork 出來的子進程，會鎖住 self._fork_lock 進行 reset()，reset 完後會釋放 self._fork_lock，disconnect 時也會 check_pid

reset() 會將 self._created_connection=0、self._available_connections=[]、self._in_use_connections = set()、self.pid = os.getpid()


```python
def _checkpid(self):
    if self.pid != os.getpid():
        acquired = self._fork_lock.acquire(timeout=5)
        if not acquired:
            raise ChildDeadlockedError
        try:
            if self.pid != os.getpid():
                self.reset()
        finally:
            self._fork_lock.release()
```


{{< line_break >}}
# class Connection 建立連線
{{< line_break >}}

這裡我們來看看 redis-py 如何建立連線

最底層的 socket TCP 連線是寫在 Connection 類的 _connect() 函數

```python
def _connect(self):
    "Create a TCP socket connection"
    # we want to mimic what socket.create_connection does to support
    # ipv4/ipv6, but we want to set options prior to calling
    # socket.connect()
    err = None
    for res in socket.getaddrinfo(
        self.host, self.port, self.socket_type, socket.SOCK_STREAM
    ):
        family, socktype, proto, canonname, socket_address = res
        sock = None
        try:
            sock = socket.socket(family, socktype, proto)
            # TCP_NODELAY
            sock.setsockopt(socket.IPPROTO_TCP, socket.TCP_NODELAY, 1)

            # TCP_KEEPALIVE
            if self.socket_keepalive:
                sock.setsockopt(socket.SOL_SOCKET, socket.SO_KEEPALIVE, 1)
                for k, v in self.socket_keepalive_options.items():
                    sock.setsockopt(socket.IPPROTO_TCP, k, v)

            # set the socket_connect_timeout before we connect
            sock.settimeout(self.socket_connect_timeout)

            # connect
            sock.connect(socket_address)

            # set the socket_timeout now that we're connected
            sock.settimeout(self.socket_timeout)
            return sock

        except OSError as _:
            err = _
            if sock is not None:
                sock.close()

    if err is not None:
        raise err
    raise OSError("socket.getaddrinfo returned an empty list")

```

{{< line_break >}}
# class Retry 與 class Backoff
{{< line_break >}}

redis-py 將 retry 機制包裝成一個 Retry 類


Retry 的初始化要定義 backoff 重試間隔和 retries 重試上限次數
```python
class Retry:
    """Retry a specific number of times after a failure"""

    def __init__(
        self,
        backoff,
        retries,
        supported_errors=(ConnectionError, TimeoutError, socket.timeout),
    )

    def call_with_retry(self, do, fail):
        """
        Execute an operation that might fail and returns its result, or
        raise the exception that was thrown depending on the `Backoff` object.
        `do`: the operation to call. Expects no argument.
        `fail`: the failure handler, expects the last error that was thrown
        """
        self._backoff.reset()
        failures = 0
        while True:
            try:
                return do()
            except self._supported_errors as error:
                failures += 1
                fail(error)
                if self._retries >= 0 and failures > self._retries:
                    raise error
                backoff = self._backoff.compute(failures)
                if backoff > 0:
                    sleep(backoff)
```

{{< line_break >}}

Backoff 類定義，ExponentialBackoff 會隨失敗次數增加也增加重試的間隔時間

```python
class AbstractBackoff(ABC):
    """Backoff interface"""

    def reset(self):
        """
        Reset internal state before an operation.
        `reset` is called once at the beginning of
        every call to `Retry.call_with_retry`
        """
        pass

    @abstractmethod
    def compute(self, failures):
        """Compute backoff in seconds upon failure"""
        pass


class ExponentialBackoff(AbstractBackoff):
    """Exponential backoff upon failure"""

    def __init__(self, cap=DEFAULT_CAP, base=DEFAULT_BASE):
        """
        `cap`: maximum backoff time in seconds
        `base`: base backoff time in seconds
        """
        self._cap = cap
        self._base = base

    def compute(self, failures):
        return min(self._cap, self._base * 2**failures)

```


{{< line_break >}}
# 結論
{{< line_break >}}

從 redis-py 源碼，我們學習到
- ConnectionPool 連線管理
- Mixin
- typing.Protocol
- socket 連線
- 自定義 retry


{{< line_break >}}


