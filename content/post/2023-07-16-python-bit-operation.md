---
title: Python 的 bit 級操作
subtitle: 
date: 2023-07-16 12:00:00
tags: ["Python"]
---


在研究一些網路協定時，不乏有許多封包包裝定義，要實作勢必要使用到 bit 級的操作

雖然我工作上多有熟成的工具直接套用，所以都沒什機會用到這類操作，但總覺得還是要熟練一下 bit 操作，然後能自組封包內容，因此有了這篇筆記

{{< line_break >}}
# Big-Endian v.s. Little-Endian
{{< line_break >}}


位元組順序 (Endianness) 分成兩種：
- Big-Endian 大端序
- Little-Endian 小端序

{{< line_break >}}

大端序最高位元組會存在最低的記憶體位址處，以 0x0A0B0C0D 的 32-bit int 值來說，0x0A byte 會存在記憶體較低的位置，0x0D byte 則會存在較高位置

<!--more-->

{{< figure src="/img/bit_op/big_endian.png" >}}

{{< line_break >}}

小端序則相反，最高位元組會存在最高的記憶體位址處，以 0x0A0B0C0D 的 32-bit int 值來說，0x0D byte 會存在記憶體較低的位置，0x0A byte 則會存在較高位置

{{< figure src="/img/bit_op/little_endian.png" >}}

{{< line_break >}}

依處理器不同，可能採用大端序或小端序
- x86、MOS Technology 6502、Z80、VAX、PDP-11等處理器為小端序
- Motorola 6800、Motorola 68000、PowerPC 970、System/370、SPARC（除V9外）等處理器為大端序
- ARM、PowerPC（除PowerPC 970外）、DEC Alpha、SPARC V9、MIPS、PA-RISC及IA64的位元組序是可調整的

{{< line_break >}}
網路傳輸一般採用大端序，也被稱之為網路位元組序，或網路序


{{< line_break >}}
# 如何表示 byte 值
{{< line_break >}}

## 1. 使用 b
```python
b''   # 空 byte
b'\x10'  # 1 byte，二進制表示的話是 0001 0000
b'\x10\x00'  # 2 byte，二進制表示的話是 0001 0000 0000 0000

b'\x10' + b'\x10\x00'  # 3 byte
```

{{< line_break >}}

## 2. 使用 struct.pack(format, v1, v2, ...)

```python
import struct

struct.pack("B", 12)   # 1 byte，0000 1100 = \x0c
struct.pack(">i", 12)   # 4 byte，big-endian，\x00\x00\x00\x0c
struct.pack("<i", 12)   # 4 byte，little-endian，\x0c\x00\x00\x00

struct.pack(">Bi", 12, 12)   # 5 byte, big-endian, \x0c\x00\x00\x00\x0c'

struct.pack("B", 12) + struct.pack(">i", 12)  # \x0c\x00\x00\x00\x0c
```

{{< line_break >}}
### format 格式字符
{{< line_break >}}

|  格式   | C 類型  | Python 類型 | 標準大小 |
|  ----  |  ----  |  ----  | ----  |
| c | char | 長度為1的字節串 | 1 |
| b | signed char | 整數 | 1 |
| B | unsigned char | 整數 | 1 |
| ? | _Bool | bool | 1 |
| h | short | 整數 | 2 |
| H | unsigned short | 整數 | 2 |
| i | int | 整數 | 4 |
| I | unsigned int | 整數 | 4 |
| l | long | 整數 | 4 |
| L | unsigned long | 整數 | 4 |
| q | long long | 整數 | 8 |
| Q | unsigned long long | 整數 | 8 |
| f | float | float | 4 |
| d | double | float | 8 |

{{< line_break >}}
### 字節順序
{{< line_break >}}

|  字符   | 字節順序  | 大小 | 對齊方式 |
|  ----  |  ----  |  ----  | ----  |
| < | Little-Endian | 標準 | 無 |
| > | Big-Endian | 標準 | 無 |
| ! | 網路 = Big-Endian | 標準 | 無 |


{{< line_break >}}
## 3. 使用 bytearray
{{< line_break >}}

```python
bytearray([1, 2, 3])   # \x01\x02\x03

bytearray(3)   # \x00\x00\x00

x = bytearray(b'\x01')
x.append(2)
x.append(3)  # \x01\x02\x03

bytearray(b'\x01') + bytearray(b'\x02')  # bytearray(b'\x01\x02')
```

{{< line_break >}}
# 從 byte 到數值
{{< line_break >}}

## 使用 struct.unpack(format, buffer)
{{< line_break >}}

```python
struct.unpack('B', b'\x0c')   # tuple (12,)

a, b = struct.unpack('>Bi', b'\x0c\x00\x00\x00\x0c')  # a=12, b=12
```



{{< line_break >}}
# 參考
{{< line_break >}}

- [wiki-位元組順序](https://zh.wikipedia.org/zh-tw/%E5%AD%97%E8%8A%82%E5%BA%8F)

{{< line_break >}}
