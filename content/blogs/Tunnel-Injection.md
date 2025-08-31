---
title: "到處都有免費 VPN -- Tunnel Injection"
date: 2025-08-28T13:50:00+08:00
draft: false
github_link: ""
author: "Chumy"
tags:
  - Cyber Security
  - Networking
image: ""
description: ""
toc: 
---

# Tunnel Injection

[English](/blogs/tunnel-injection-english)

## TL;DR

這篇文章講述了大多數 stateless tunnel protocol 的缺陷以及我如何把他們變成一個僅僅需要 5 行 linux ip 指令就能夠設定好的免費 VPN

## 感謝

在進入正題前這邊感謝 [seadog007](https://seadog007.work/) 的 [STUIX](https://stuix.io/)，因為我的網路的 ASN 主要是在 STUIX 做交換的，並且也用到他們很多的資源做測試。

也感謝 [NCSE](https://ncse.tw/) 特別幫我在我測試的時候幫我加一點 IPT 的流量。

## 前言

這是一個講述我如何從研究一個協議到讓全世界都有我的 VPN 的小故事

事情是這樣發生的，大約在我大一的時候在 cloud native 聽到有關於 segment routing v6 這個酷東西，因此在大二的時候就有稍微研究一下同時也大概理解它的原理，至於這東西到底怎麼運作的，因為不是這篇文的重點所以就不談，可能之後會發其他的文章討論這個。

大三的時候因為接到 [2024 CGGC](https://cggc.nchc.org.tw/) aka 一場台灣辦的 CTF 的出題邀請，因此就開始在想梗。

![image](https://github.com/user-attachments/assets/15d204e5-72b9-4827-97a4-9abbcb51f621)

![image](https://github.com/user-attachments/assets/21be27d8-b269-47ee-8c1b-48f3831bea3d)

然而在某天洗澡的時候突然仔細回味了一下就突然想到了這個非常酷的利用手法，順帶一題這題只有 [seadog007](https://seadog007.work/) 有打出來

這是 [seadog007](https://seadog007.work/) 幫我畫的攻擊手法的詳細圖

![image](https://github.com/user-attachments/assets/ac66ca21-eadc-4000-98d9-6133a3f8456d)

## What is Tunnel

大家最耳熟能詳的 Tunnel 應該就是 VPN 了，因為 VPN 基本上就是一種 Tunnel，用很多組加密或未加密的 Tunnel 組合成的虛擬網路就是 VPN 的核心概念。

那它的運作原理其實很簡單，基本上就是信封袋外面再加封一層信封袋的概念。

![image](https://github.com/user-attachments/assets/6e5c692c-183f-41a2-a4d4-e03cb3eb4202)

![image](https://github.com/user-attachments/assets/c71b10e9-f6fa-473d-a94a-77e99d0a3dd5)

用網路的說法講就是，當 packet 從 tunnel 的網卡離開時，負責處理這個 tunnel 的程式就會把這個封包的內容封裝成這個 tunnel protocol 的格式以後作為 tcp 或 udp 又或者是某個 L3 的 protocol 的 payload 從實體網卡（或下一層 tunnel 網卡）送出。

接收時負責監聽這個 tunnel protocol 的程式從監聽的 port 接收到資料時把該 tunnel protocol 的 header 拆掉以後將內部的 raw data 直接作為 l3 或 l2 的 packet 送到 tunnel 網卡上。

![image](https://github.com/user-attachments/assets/88cf1d26-7f70-485e-8089-6e39ec9e4bca)

而 stateless tunnel protocol 就是指說這個 tunnel protocol 你只要給他甚麼 tunnel protocol 的 packet 他就直接拆開然後轉送，沒做任何的驗證或者是握手。

## Impact

那沒有驗證會怎樣，基本上我只要能夠偽造出一個符合這個 tunnel protocol 的封包，我就可以做到將惡意的 packet 透過這個裸奔的 tunnel 轉送到 victim 的內網內，然而過往的研究基本上都是認為這個攻擊是只有單向的，頂多可以做一些 DDoS 而已。

![image](https://github.com/user-attachments/assets/9c57dd15-5db7-438b-9d39-d7871cd189aa)

![image](https://github.com/user-attachments/assets/21321374-4940-4520-9e9a-618ce5ccb06e)

包含今年 1 月剛出來的一篇[研究](https://www.top10vpn.com/research/tunneling-protocol-vulnerability/)

![image](https://github.com/user-attachments/assets/359e4baa-1152-4fe7-9445-976b6a74733b)

這篇研究雖說有講到很多裸奔 stateless tunnel 的掃描方法，以及一些特定型號產品的 router 不會驗證 source ip 的問題，然而主要也都是 focus 在 DDoS 的攻擊面上。

然而事實上只需要把腦袋轉一下：我們其實並不需要來回都要過這個 tunnel 才對。

畢竟攻擊的核心是需要將流量「注入到 Tunnel 內」因此這邊就先取名叫做 Tunnel Injection。

### Tunnel Injection to Internal Network

第一個利用手法是我們可以做到 Interactive Internal Network Access

首先我們偽造出一個符合這個 tunnel protocol 的封包

外層的 source ip 是 attacker 的 public ip 或者一些 tunnel protocol 因為會驗證 source ip 所以這邊也可能需要做 ip spoofing，destination ip 是 victim tunnel 的 public ip。

內層的 source ip 是 attacker 的 <font color="red">public ip</font> 這個就是這邊的核心了，而 destination ip 就是你想要扁的 victim 內網機器的 <font color="red">private ip</font>。

![image](https://github.com/user-attachments/assets/7a95b8c2-cf6c-4d50-9439-1a7ff17d9f52)

送出去以後會發生甚麼事呢。

首先 packet 抵達 victim 的 router 時會被解封裝，並把內層的封包依照這台 router 的 routing table 做 forwarding，同時 conntrack table 會留下一條轉發的紀錄。

![image](https://github.com/user-attachments/assets/d0e5e518-0435-42b3-96c1-5529311bfd73)

然後內網的機器就會看到一個 src = <font color="red">public ip</font> dest = 自己 ip 的封包，因此他 response 理所當然會 src = 自己 ip dest = <font color="red">public ip</font>

![image](https://github.com/user-attachments/assets/33f5e5d2-2954-457b-bae5-e43ba2b12040)

這個 packet 來到 victim router 的時候，因為 dest = <font color="red">public ip</font> 所以就會<font color="red">依照 routing table 做 forwarding 直接走 default gateway 出去</font>。

這邊同時要補充一點，因為一般的 router 不會對奇怪類型的封包做 SNAT 比如 TCP SYN/ACK、ICMP type 0 aka pong 因此回來的 packet 離開 victim router 的時候是不會被 NAT 也就是說 <font color="red">source ip 是 private ip</font> 這在某些場景是會有問題的。

那接著我們就可以從網路上收到這個來自 victim 內網的回應。

![image](https://github.com/user-attachments/assets/2a483ec0-2a7d-445a-a8cb-403f24bcd8bc)

我們就能夠 Arbitrary Interactive Internal Network Access

<p><iframe class="youtube" width="560" height="315" src="https://www.youtube.com/embed/KSNkpPdzw8o?si=pE8cLkI-OTlZILsh" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe></p>

### Tunnel Injection to External Network

在講攻擊手法之前我們先來看看 NAT 一般是怎麼實作的，這邊以 Linux 為例

在 Linux kernel base 的 router 上，一般我們毀使用 `netfilter` 與 `conntrack` 來處理 NAT，而最常見的指令介面會是 `iptables` 與 `nftables`，這邊以 `iptables` 為例

一般情況下要啟動 NAT 我們會下這條 `iptables` 指令

```bash
iptables -t nat -A POSTROUTING -o wan -s 192.168.0.0/16 -j MASQUERADE
```

這條指令的意思是在確定出口網卡時當封包從 <font color="red">wan</font> 網卡離開時，如果 source ip 符合 192.168.0.0/16 就做 NAT 並使用 wan 網卡的 public ip 作為 source ip。

但是事實上其實很多 router 的廠商都會偷懶，只這樣寫

```bash
iptables -t nat -A POSTROUTING -o wan -j MASQUERADE
```

這條指令的意思是在確定出口網卡時當封包從 <font color="red">wan</font> 網卡離開時，就做 NAT 並使用 wan 網卡的 public ip 作為 source ip。

可以發現它少了 source ip 的檢查，所以事實上如果 source ip 已經是 public 他也會做 NAT，畢竟 [RFC 1918](https://datatracker.ietf.org/doc/html/rfc1918) 只是定義而已，沒說你不能拿 public ip 當 private ip 用。

然而如果今天這台 router 同時有一個裸奔的 tunnel 再跑會怎樣呢？

答案是我們會得到一個<font color="red">免費的跳板</font>

事實上我們只需要將前面 [Tunnel Injection to Internal Network](#tunnel-injection-to-internal-network) 所說的內層的 destination ip 更改成你要 access 的外網 target public ip 就好。

一樣我們偽造出一個符合這個 tunnel protocol 的封包

外層的 source ip 是 attacker 的 public ip 或 ip spoofing 的 IP，destination ip 是 victim tunnel 的 public ip。

內層的 source ip 是 attacker 的 <font color="red">public ip</font> ，而 destination ip 這次就改成用<font color="red">你要 access 的外網 target public ip</font>。

![image](https://github.com/user-attachments/assets/ffbfa772-3f79-4701-ad49-b968441ba406)

packet 抵達 victim 的 router 時會被解封裝，並把內層的封包依照這台 router 的 routing table 做 forwarding，同時依照上面 `iptables` 做 NAT 的 source ip 轉換，同時 conntrack table 會留下一條 NAT 的轉換紀錄。

![image](https://github.com/user-attachments/assets/b120f257-d8b2-4392-a619-80133643455c)

接著你要連的 target 就會收到你的 packet 並且可能會做 response，因為 source ip 已經因為上面所說的做 NAT 所以 target 看到的會是

src = <font color="red">victim public ip</font> dest = <font color="red">target public ip</font>

因此 response 會是 src = <font color="red">target public ip</font> dest = <font color="red">victim public ip</font>

![image](https://github.com/user-attachments/assets/92a6f6f4-293e-42ff-80be-e659a6bd32cd)

接著 victim router 收到 packet 後會依照 conntrack 的紀錄作 destination ip 的復原，並且依照這台 router 的 routing table 做 forwarding

![image](https://github.com/user-attachments/assets/9ab444c2-9e01-4723-996e-e006786c340c)

然後你就收到 response 了

![image](https://github.com/user-attachments/assets/9c24ec6f-6eae-4fe0-a7fd-df7588e38fd2)

我們就能夠 Arbitrary Interactive Pivoting Attack via Tunnel 

![image](https://github.com/user-attachments/assets/6dfa50fa-0f75-46f5-acf2-b4d52f2a313c)

<p><iframe class="youtube" width="560" height="315" src="https://www.youtube.com/embed/EC9qySzo6J4?si=mF9rBsuFrUwao2-k" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe></p>

## RPF or source ip verify bypass

當你以為這樣就結束時現實往往會來一巴掌

還記得前面講 [Tunnel Injection to Internal Network](#tunnel-injection-to-internal-network) 時所說的 

```
一般的 router 不會對奇怪類型的封包做 SNAT 比如 TCP SYN/ACK、ICMP type 0 aka pong 因此回來的 packet 離開 victim router 的時候是不會被 NAT 也就是說 source ip 是 private ip 這在某些場景是會有問題的。
```

而這邊所說的場景就是指 Reverse path forwarding (RPF) 或者 source ip verify

![image](https://github.com/user-attachments/assets/e80bb09f-a306-4308-944b-938e24e01da0)

首先絕大多數的 ISP 為了要防止客戶對外做 IP Spoofing 的攻擊，一般都會針對 client 方向的網卡開啟 RPF

RPF 就是指說，當網卡收到 packet 時，會拿 source ip 對一次 routing tables，如果這個 source ip 並不符合 routing tables 上送往這張網卡的任何規則的話，這個封包就會被 DROP 掉。

因此如果遇到 victim 的 ISP 有啟用這類防護的情況的話你是收不到 response 的，那該怎麼辦

### Bypass for Internal Network Access

這邊我們就來談談 P2P 網路是怎麼做 NAT hole punching 的

首先當雙方的 source port 提前知道，或者可預測的情況（以 p2p 應用程式的情況，會有一個 stun server 協助探測兩邊的 client 使用的 source port 是多少）

雙方可以同時朝對方的 public ip 與 port 發送一個 packet，這邊以 udp 為例

![image](https://github.com/user-attachments/assets/590fc3b4-e990-4ebb-8f8d-e44c20f9f30f)

packet 經過 router 以後會做 NAT 並且同時 conntrack table 會留下一條 NAT 的轉換紀錄。

![image](https://github.com/user-attachments/assets/1a9e7d65-5fc1-44c0-bbed-92361eaed76d)

兩個封包都各自抵達對方的 router，這邊可以發現抵達的時候 conntrack table 已經有符合條件的 NAT 轉換紀錄。

![image](https://github.com/user-attachments/assets/4e531fdf-9045-44aa-bb86-2b1b7189c7e2)

因此接著就會依照 conntrack 的紀錄作 destination ip 的復原

![image](https://github.com/user-attachments/assets/368274e7-f142-4270-becf-f3589dfd908a)

以上就是整個 NAT hole punching 的大概流程，如果說是 tcp 的話會更複雜一點，因為 tcp 是 statefull，有興趣可以去查 TCP Simultaneous Open，剛好今年的 [HITCON CTF 2025 有類似的題目](https://ctf2025.hitcon.org/dashboard/#19) 畢竟 TCP Self Connect 本身也是一種 TCP Simultaneous Open 的特殊案例。

順帶一題我會清楚整個 P2P 傳輸的流程也歸功於我高中時曾經[手刻過整個 P2P 做 NAT hole punching 的流程，也手刻過 STUN server](https://github.com/Jimmy01240397/NetworkServicePackage) ~~雖說現在看他的 code 醜醜的就是了~~，之前有些人總是問我為甚麼要自己造輪子，但是我認為自己手刻輪子也是一種樂趣，並且這樣才能夠真正的學到東西，畢竟使用自己寫出來的工具也是一種樂趣吧？

總之我們可以理解為從內部發 packet 做完 NAT 以後，其時就變相地做完了一個有限制的 port forwarding

再加上我們可以利用 Tunnel Injection 做到單方向任意存取內網或者從內網發 packet，那我們是否可以手動做 NAT hole punching 以後直接從外網做存取呢？

事實上是可以的。以下用 TCP 舉例。

我們利用 Tunnel Injection 在 victim 的內網發送 TCP SYN 來做 hole punching

source ip 是你想要扁的 victim 內網機器的 <font color="red">private ip</font>，而 destination ip 是 attacker 的 <font color="red">public ip</font>。

![image](https://github.com/user-attachments/assets/34fa43d2-f10d-4c30-bf6b-9bba2537b88d)

packet 抵達 victim 的 router 時會依照這台 router 的 routing table 做 forwarding，接著做 NAT 的 source ip 轉換，同時 conntrack table 會留下一條 NAT 的轉換紀錄。

這樣 hole punching 就完成了

![image](https://github.com/user-attachments/assets/68ca12ef-176c-42c8-9306-406007a1eaa2)

接著朝著這個打好的洞發 tcp SYN 

source ip 是 attacker 的 <font color="red">public ip</font>，而 destination ip 是你想要扁的 victim 內網機器的 <font color="red">private ip</font>。

![image](https://github.com/user-attachments/assets/6a05f51d-89e9-4b54-a0d8-a1af4b66f30e)

packet 會依照 conntrack 的紀錄作 destination ip 的復原，並且依照這台 router 的 routing table 做 forwarding 轉到 victim 的內網機器

![image](https://github.com/user-attachments/assets/9da739d2-8185-4c74-a558-1268905a25b5)

victim 的內網機器收到後會回 TCP SYN/ACK

![image](https://github.com/user-attachments/assets/dd125103-65b5-4ff8-a28d-106f12453ba6)

並且 source ip 會很正常的 NAT 成 victim 的 public ip

![image](https://github.com/user-attachments/assets/dfc538ec-f8bd-4f92-ad21-d53ad41c4f77)

接著 attacker 就回 ACK

![image](https://github.com/user-attachments/assets/a2f3b7a5-61fd-45f8-a810-3076ed283495)

然後 connection 就建立起來了，可以快樂傳資料到內網了

![image](https://github.com/user-attachments/assets/30dec908-b538-4163-8151-7adc197584d2)

然而上面的流程其實只有對 UDP 完全成立。

TCP 只有部份的 router 或較舊的 Linux kernel 成立而已。

主要是因為現今的 Linux kernel 的 conntrack 做 TCP Simultaneous Open 會嚴格檢查封包傳輸的行為是否符合 TCP Simultaneous Open 的行為，如果有一邊的 SYN/ACK 沒收到，你會發現他的 state 會永遠停留在 `SYN_SENT2`，這時候內網對外發 TCP PUSH 的時候<font color="red">依然不會被 NAT 成 victim 的 public ip</font>

因為上面的流程會少 attacker 發向 victim 內網機器的 SYN/ACK 因此永遠不會 ESTABLISHED

那如果我補發 attacker 發向 victim 內網機器的 SYN/ACK 是否就能解決呢？

由於 conntrack 只要看到任何 TCP RST 就會直接把這條紀錄轉成 close state 並且在幾秒後 free 掉，並且一般的系統在收到來源不明，或者是不符合握手流程的 TCP 包會直接回 TCP RST（因為 TCP SYN 是我們手動利用 Tunnel Injection 打出去的，因此 victim 內網機器壓根不認識這個 SYN/ACK）

雖說這種作法在一些會 DROP 掉 TCP RST 的內網環境可能是可以成功的，但是這邊還是以預設狀況為主比較好。

那該怎麼解決問題呢

仔細看 sysctl 的參數我們會發現 conntrack 的 tcp close 紀錄的壽命只有 10 秒，之後紀錄就會被 free 掉

![image](https://github.com/user-attachments/assets/fcab59ab-27a7-4a7f-a1e5-10c82ae13728)

然後還有一個有趣的參數是 net.netfilter.nf_conntrack_tcp_loose 

![image](https://github.com/user-attachments/assets/84632096-8e77-4e61-bb89-35a1617bfbc3)

這個參數預設是 1

那他有甚麼意思呢？

當 net.netfilter.nf_conntrack_tcp_loose 為 1 時，允許在 conntrack 沒有任何符合條件的紀錄時收到 PUSH 或 ACK 做 NAT

也就是說當我 conntrack 沒有任何符合條件的紀錄時，內網向外發一個 TCP PUSH 或 ACK 他會直接生出一個 ESTABLISHED state 的 NAT 紀錄

那這就非常好玩了，因為 close state 時間夠短，並且 TCP 是會重傳的，我們可以手動發 TCP RST 來 free 掉 conntrack record 然後內網發 PUSH 出來就直接變成 ESTABLISHED state

所以這邊在講一次流程：

我們利用 Tunnel Injection 在 victim 的內網發送 TCP SYN 來做 hole punching

source ip 是你想要扁的 victim 內網機器的 <font color="red">private ip</font>，而 destination ip 是 attacker 的 <font color="red">public ip</font>。

![image](https://github.com/user-attachments/assets/34fa43d2-f10d-4c30-bf6b-9bba2537b88d)

packet 抵達 victim 的 router 時會依照這台 router 的 routing table 做 forwarding，接著做 NAT 的 source ip 轉換，同時 conntrack table 會留下一條 NAT 的轉換紀錄。

![image](https://github.com/user-attachments/assets/68ca12ef-176c-42c8-9306-406007a1eaa2)

接著朝著這個打好的洞發 tcp SYN 

source ip 是 attacker 的 <font color="red">public ip</font>，而 destination ip 是你想要扁的 victim 內網機器的 <font color="red">private ip</font>。

![image](https://github.com/user-attachments/assets/6a05f51d-89e9-4b54-a0d8-a1af4b66f30e)

packet 會依照 conntrack 的紀錄作 destination ip 的復原，並且依照這台 router 的 routing table 做 forwarding 轉到 victim 的內網機器

![image](https://github.com/user-attachments/assets/9da739d2-8185-4c74-a558-1268905a25b5)

victim 的內網機器收到後會回 TCP SYN/ACK

![image](https://github.com/user-attachments/assets/dd125103-65b5-4ff8-a28d-106f12453ba6)

並且 source ip 會很正常的 NAT 成 victim 的 public ip

![image](https://github.com/user-attachments/assets/dfc538ec-f8bd-4f92-ad21-d53ad41c4f77)

接著 attacker 就回 ACK

![image](https://github.com/user-attachments/assets/a0f2a086-4755-4169-9085-daf7b523b6fe)

我們這邊發一個 PUSH 進去，可能是 HTTP 的 request 之類的

![image](https://github.com/user-attachments/assets/3f55be71-accc-4b0e-8aa1-878ac136cc92)

因為上面講的機制，內網出來的 PUSH 會被 DROP，attacker 沒收到資料自然不會回 ACK，victim 內網機器沒收到 ACK 就會一直重傳 TCP PUSH

![image](https://github.com/user-attachments/assets/ef676fa9-b7d5-492b-86bd-ffd606b8c0bb)

接著我們利用 Tunnel Injection 從內網打一個 TCP RST 出來，手動 free 掉這條 conntrack record

![image](https://github.com/user-attachments/assets/7671e6c4-8041-4034-b9da-93fd323e0b1d)

Free 掉以後當內網重傳 TCP PUSH

![image](https://github.com/user-attachments/assets/115257d7-8133-4b60-9dd9-3177cbb7583d)

就會成功被 NAT 並且 conntrack table 會留下一條 NAT <font color="red">ESTABLISHED state</font> 的轉換紀錄。

![image](https://github.com/user-attachments/assets/b04c2350-c5b4-428f-b55d-7f665be6a19e)

然後 connection 就建立起來了，可以快樂傳資料到內網了

![image](https://github.com/user-attachments/assets/30dec908-b538-4163-8151-7adc197584d2)

以上我們就成功做到 RPF bypass 的 Internal Network Access 了。

<p><iframe class="youtube" width="560" height="315" src="https://www.youtube.com/embed/1YkltH1gCz4?si=V0iYDhGsGvKnd70O" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe></p>

### Bypass for External Network Access

前面會比較簡單是因為整個 Bypass 過程中只用到一次的 NAT，但是對於 External Network Access 我們如果要 Bypass 就需要做兩次 NAT。

1. 把去程的 source ip 換成 victim 的 public ip
2. 把回程的 source ip 換成 victim 的 public ip

那該怎麼辦呢？

這時候如果 victim 內網還有一層 tunnel 跟 NAT 的組合，我們就可以<font color="red">串一個 NAT Chain 出來</font>

![image](https://github.com/user-attachments/assets/6e0361aa-f2ed-48ba-8bfa-b97eae5e9a5a)

我們利用 Tunnel Injection 在 victim 的第一層內網發送 TCP SYN 來做 hole punching

source ip 是 <font color="red">target 的 public ip</font>，而 destination ip 是 <font color="red">attacker 的 public ip</font>。

![image](https://github.com/user-attachments/assets/eccd387e-42e7-44eb-8962-faa95ec0ffb2)

packet 抵達 victim 的第一層 router 時會依照這台 router 的 routing table 做 forwarding，接著做 NAT 的 source ip 轉換，同時 conntrack table 會留下一條 NAT 的轉換紀錄。

![image](https://github.com/user-attachments/assets/9a6223c9-c748-41be-81c2-9ae450bdc4e7)

然後我們必須故意打一個 TCP SYN 來讓第一層 router 的 conntrack record 轉成 SYN_SENT2 state

![image](https://github.com/user-attachments/assets/674aa743-e4b6-4bad-b05b-ea7763fb6f79)

這個封包轉往 target 時就會被 RPF 給 DROP 掉

![image](https://github.com/user-attachments/assets/e8b0399a-96c1-4b5a-98be-a9f29b356cba)

我們利用 Tunnel Injection 在 victim 從第二層內網發送 TCP SYN 到 target

source ip 是 <font color="red">attacker 的 public ip</font>，而 destination ip 是 <font color="red">target 的 public ip</font>。

![image](https://github.com/user-attachments/assets/59105cb3-cdd6-4808-b561-499c8b839fcf)

![image](https://github.com/user-attachments/assets/0689a0c6-40e9-4ee9-b5f2-43aa9fed3f1a)

TCP SYN 會被第二層 router 做 NAT 的 source ip 轉換，同時 conntrack table 會留下一條 NAT 的轉換紀錄。

![image](https://github.com/user-attachments/assets/00729446-faa7-401c-b282-030aae85a543)

TCP SYN 會被第一層 router 做 NAT 的 source ip 轉換，同時 conntrack table 又會留下一條 NAT 的轉換紀錄。

![image](https://github.com/user-attachments/assets/7be48293-0ae1-4f67-b4c1-bad4cae8c9d0)

target 會回 SYN/ACK 並且依照 conntrack 的紀錄作 destination ip 的復原

![image](https://github.com/user-attachments/assets/61f1d773-6e80-4f93-a80f-eec9a253db2e)

destination ip 變成第二層 router 的 wan IP，並傳到第二層 router，一樣依照 conntrack 的紀錄作 destination ip 的復原

![image](https://github.com/user-attachments/assets/0b813ca9-8604-48ad-9eb9-8f18c31d0416)

destination ip 變成 attacker 的 public ip，並傳到第一層 router

![image](https://github.com/user-attachments/assets/a84a2fcb-a332-4680-8f34-2d07c5cfc5b3)

這時候很有趣的是 source ip 是 target 的 public ip 當好對應到之前在第一層 route 打好的 conntrack record 的 NAT 轉換紀錄。

因此他會一樣依照 conntrack 的紀錄作 source ip 的 NAT

![image](https://github.com/user-attachments/assets/48ecb285-68ef-4a9f-82aa-030b7e9d910a)

attacker 收到的就是 source ip 為 <font color="red">victim 的第一層 router 的 public ip</font>

![image](https://github.com/user-attachments/assets/511aede6-4882-4676-a2a8-2204684283d7)

接下來的流程就跟前面講 [Bypass for Internal Network Access](#bypass-for-internal-network-access) 的流程一樣了

回 ACK

![image](https://github.com/user-attachments/assets/ac76ca3c-6eae-40fb-9aba-fa0a0864c138)

發 push

![image](https://github.com/user-attachments/assets/32ab2ce8-6caa-482a-add0-144899e0d438)

發 RST 來 free 掉 conntrack record

![image](https://github.com/user-attachments/assets/5f543174-edbe-4aa2-8aef-bf269000150d)

target 持續重傳

![image](https://github.com/user-attachments/assets/94a00362-e077-46f6-8c43-0cc7cda68ab3)

打回來成功 NAT 並且 conntrack record ESTABLISHED

![image](https://github.com/user-attachments/assets/6c0dbb36-6cf7-4b89-b55d-56acf98acaa6)

以上我們就成功做到 RPF bypass 的 External Network Access 了。

<p><iframe class="youtube" width="560" height="315" src="https://www.youtube.com/embed/_GcIFKyjGmE?si=qWA0cVCgFotki8nm" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe></p>

而且事實上 NAT Chain 的利用除了上面用來繞 RPF 外，也可以用來在 Internal Network Access 時繞一些 server 對 source 的檢查，畢竟有些 server 也是有自己的 firewall 的。

## L2 Tunnel Abuse

前面主要都是在談 L3 Tunnel 的利用，而 L2 Tunnel 因為多了一層 Ethernet header 要封，所以比較麻煩一些，因為我們需要 router 或者內往機器的 mac address 才可以達成利用，又或者可以做 broadcast 或 multicast。

那我們有沒有可能利用 broadcast 或 multicast 來 leak 一些有用的資訊或者 mac address 呢？

是有可能的

當內網除了有 IPv4 以外還啟用了 IPv6 並且有 unicast address 時，由於預設情況下 ICMPv6 request 是允許 multicast 的，因此我們可以用 multicast 的 IPv6 ping 來 leak IPv6 address

![image](https://github.com/user-attachments/assets/abc2edae-bf6e-46f9-ae66-001080bdffb2)

首先我們用 multicast mac address 以及 attacker 的 public IPv6 address 在 victim 內網 multicast IPv6 ping

![image](https://github.com/user-attachments/assets/ea7fd3e3-1d59-403d-a125-cbc0243b043e)

Switch 會把這個 ping multicast 給所有有訂閱這個 multicast mac address 的設備

![image](https://github.com/user-attachments/assets/5465ba53-bc01-486a-806f-8472c1c52097)

victim 內網設備會回 ICMP pong 給 attacker

![image](https://github.com/user-attachments/assets/950c89a1-933d-4d69-a962-b042104c807c)

我們便可拿到所有內網設備的 IPv6 address

![image](https://github.com/user-attachments/assets/4ed1c850-90c0-4e26-824f-82a9a49e0f22)

<p><iframe class="youtube" width="560" height="315" src="https://www.youtube.com/embed/Eqb1dv2bPzk?si=gX9q73c6ubSatqVy" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe></p>

那有 IPv6 address 可以幹嘛呢？

根據 [RFC 4291](https://datatracker.ietf.org/doc/html/rfc4291) 所記載的 SLAAC IPv6 address 的動態生成方式中，有一種方式是用 mac address 算出來的

![image](https://github.com/user-attachments/assets/39dad163-bb73-420f-bced-ca58c684cb1c)

這個算出的 IPv6 address 我們是可以逆推回 mac address 的，並且 linux 預設便是使用此方法生成 IPv6

![image](https://github.com/user-attachments/assets/9e120a86-764d-4558-992c-07732265b8e0)

此外，另外一名來自趨勢科技的研究員 [123ojp](https://github.com/123ojp) 也剛好跟我一樣再研究這個方面的攻擊手法，並且有發現針對 VXLAN 這個 L2 Tunnel 更為高效的利用手法，有興趣可以去看他在 [DEFCON 33 的議程](https://defcon.org/html/defcon-33/dc-33-speakers.html#content_60316)

## IPSec

或許大家以為只要啟用 IPSec 就可以防護這類的攻擊，但是事實上只要你的 IPSec 有配置不當，即使開 IPSec 也可能會遭受攻擊。

理論上 IPSec 開啟後，所有指定 protocol 的 packet 離開網卡前會被 xfrm 攔截並且做 IPSec 的加密，同時所有指定 protocol 的 raw packet 進入網卡時應該要被 DROP。

然而 xfrm 上有一個特殊的參數叫做 level，他允許你加密離開網卡的流量的同時 allow 進入網卡的 raw packet，雖說這項設定預設不會開，但你難保你家 IT 設定了一整天 IPSec 還沒設起來整個很躁的時候會做甚麼事情對吧！

![image](https://github.com/user-attachments/assets/58e4082d-c02c-4299-a859-05bf197ea631)

<iframe width="560" height="315" src="https://www.youtube.com/embed/y1ZlsGSu-RY?si=9klBHYJLEvnkw1aR" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

順帶一題，edgeos 你即使 level 屬性是預設值 require 他也會 allow raw packet，這是我的朋友 aka 現任[成大 CCNS](https://www.ccns.io/) 的網管 [zen](https://zenwen.tw/) 發現的，但基本上這件事還在待確認中。

## 掃描與 Real World

最後來談談掃描的部分吧！

雖說[這篇研究](https://www.top10vpn.com/research/tunneling-protocol-vulnerability/)已經講過具體的掃描方式了。

但我這邊也分享一下

### Internal access able

我們可以透過 victim 的 tunnel 在 victim 內網彈一個 ICMP request ping 出來，source ip 是 private IP destination ip 是 scanner 的 IP，並且在 ICMP ping 的 payload 內留一些標記以及寫下 tunnel 外層 src ip 與 dst ip，當我們可以從 wan 收到一模一樣的 ICMP ping 時，就可以確定對方的 router 是否存在該 protocol 的 tunnel，接著我們可以觀察傳過來的的 source ip 是否有變成 public 我們就可以知道 victim router 有沒有 NAT。

![image](https://github.com/user-attachments/assets/5cf3582e-5570-4f27-aa24-9fa9d02f16b4)

![image](https://github.com/user-attachments/assets/e3347bbf-949f-4682-aca2-051b501daa9d)

![image](https://github.com/user-attachments/assets/10ca7e2d-3a10-493e-92f3-8574b1fa36e8)

接著檢查是否有 RPF，可以透過 victim 的 tunnel 在 victim 內網彈一個 ICMP reply pong 出來，source ip 是 private IP destination ip 是 scanner 的 IP，由於[前面講過](#tunnel-injection-to-internal-network) "一般的 router 不會對奇怪類型的封包做 SNAT 比如 TCP SYN/ACK、ICMP type 0 aka pong" 因此就能確定從 victim router 出來絕對會是 private ip，接著只要看 scanner 有沒有收到就知道 victim 的 ISP 有沒有 RPF 了

![image](https://github.com/user-attachments/assets/6f3f4e5e-dee9-4929-9a93-edfb430a7fe2)

![image](https://github.com/user-attachments/assets/b4ba60ec-3f38-4086-979d-af123ce393e3)

### External access able

由於 External access able 的 list 必定為 Internal access able 且有開 NAT 的 list 的 sub set，因此我們可以直接拿這個 IP list 再掃一次

一樣的架構但這次多一個可控的 target 機

透過 victim 的 tunnel 在 victim 內網彈一個 ICMP request ping 出來，source ip 是 scanner 的 IP destination ip 是 target 的 ip

![image](https://github.com/user-attachments/assets/11f911c5-5654-4810-a88a-9a79596a6c3b)

如果是 External access able 的話這邊應該會對 public IP 做 NAT 之後送到 target 機，這邊 target 機要記得做 log

![image](https://github.com/user-attachments/assets/14d6f543-e252-49c7-ac93-ed754cd525ec)

target 機收到後會回 ICMP reply pong 到 victim router

![image](https://github.com/user-attachments/assets/aed43a57-e31d-4eda-bde2-7c56ba9c588b)

victim router 復原 destination ip 後會送回 scanner，這時候只需要檢查與比對 scanner 的 log 跟 target 的 log，我們就可以知道哪些是 External access able 的了。

![image](https://github.com/user-attachments/assets/dda1ce57-eae0-46a9-88bc-31241d4de968)

### 掃描結果

這邊針對不須 ip spoofing 的 tunnel 做掃描的結果

總共有：
- GRE: 230035 台
  - 有開 NAT: 191961 台
    - External access able: 22875 台
- IPIP: 76209 台
  - 有開 NAT: 23851
    - External access able: 5438 台

由於有些 ISP 配的是 dynamic ip 所以可能會有誤差

這邊也針對這些 IP 的 ASN 做分析可以看到有一坨中國的 IP

GRE: 

![image](https://github.com/user-attachments/assets/efb0aa63-43e5-45a5-bf00-0dbd50c8e38a)

IPIP:

![image](https://github.com/user-attachments/assets/a69480e1-7fdc-4045-9b6d-f33a749b5151)

## 結語

最後只能說很可惜，因為剛剛好 [123ojp](https://github.com/123ojp) 跟我撞到同一個研究且早了一點點，所以最後沒有投上 [HITCON 2025](https://hitcon.org/2025/zh-TW/)，但是最後也有成功上去 HITCON Lightning Talk 講了一下這個有趣的小東西並補充了 [123ojp](https://github.com/123ojp) 沒有講到也沒有想到的 [Tunnel Injection To External Network](#tunnel-injection-to-external-network) 利用手法，且最後其時也聊得蠻開心的，之後我可能會試著拿這個研究去投一些其他的研討會看看。

順帶一題，雖說上面的 Demo 都是用我自己手刻的工具時做的，但是理論上再不需要做 [RPF Bypass](#rpf-or-source-ip-verify-bypass) 的情況下，其實是可以用 <font color="red">5 行 linux ip 指令</font>就能夠把 internal 跟 external 的攻擊打出來，這部分就留給各位當作業了 XXD
