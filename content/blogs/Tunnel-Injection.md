---
title: "到處都有免費 VPN -- Tunnel Injection"
date: 2030-08-30T00:00:00+08:00
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

## TL;DR

這篇文章講述了大多數 stateless tunnel protocol 的缺陷以及我如何把他們變成一個僅僅需要 5 行 linux ip 指令就能夠設定好的免費 VPN

## 感謝

在進入正題前這邊感謝 [seadog007](https://seadog007.work/) 的 [STUIX](https://stuix.io/)，因為我的網路的 ASN 主要是在 STUIX 做交換的，並且也用到他們很多的資源做測試。

也感謝 [NCSE](https://ncse.tw/) 特別幫我在我測試的時候幫我加一點 IPT 的流量。

## 前言

這是一個講述我如何從研究一個協議到讓全世界都有我的 VPN 的小故事

事情是這樣發生的，大約在我大一的時候在 cloud native 聽到有關於 segment routing v6 這個酷東西，因此在大二的時候就有稍微研究一下同時也大概理解它的原理，至於這東西到底怎麼運作的，因為不是這篇文的重點所以就不談，可能之後會發其他的文章討論這個。

大三的時候因為接到 [CGGC](https://cggc.nchc.org.tw/) aka 一場台灣辦的 CTF 的出題邀請，因此就開始在想梗。

![image](https://github.com/user-attachments/assets/15d204e5-72b9-4827-97a4-9abbcb51f621)

![image](https://github.com/user-attachments/assets/21be27d8-b269-47ee-8c1b-48f3831bea3d)

這是 [seadog007](https://seadog007.work/) 幫我畫的攻擊手法的詳細圖

![image](https://github.com/user-attachments/assets/ac66ca21-eadc-4000-98d9-6133a3f8456d)

然而在某天洗澡的時候突然仔細回味了一下就突然想到了這個非常酷的利用手法，順帶一題這題只有 [seadog007](https://seadog007.work/) 有打出來

## What is Tunnel

大家最耳熟能詳的 Tunnel 應該就是 VPN 了，因為 VPN 基本上就是一種 Tunnel，用很多組加密或未加密的 Tunnel 組合成的虛擬網路就是 VPN 的核心概念。

那它的運作原理其實很簡單，基本上就是信封袋外面再加封一層信封袋的概念。

<img width="863" height="351" alt="image" src="https://github.com/user-attachments/assets/6e5c692c-183f-41a2-a4d4-e03cb3eb4202" />

<img width="1107" height="351" alt="image" src="https://github.com/user-attachments/assets/c71b10e9-f6fa-473d-a94a-77e99d0a3dd5" />

用網路的說法講就是，當 packet 從 tunnel 的網卡離開時，負責處理這個 tunnel 的程式就會把這個封包的內容封裝成這個 tunnel protocol 的格式以後作為 tcp 或 udp 又或者是某個 L3 的 protocol 的 payload 從實體網卡（或下一層 tunnel 網卡）送出。

接收時負責監聽這個 tunnel protocol 的程式從監聽的 port 接收到資料時把該 tunnel protocol 的 header 拆掉以後將內部的 raw data 直接作為 l3 或 l2 的 packet 送到 tunnel 網卡上。

<img width="1322" height="503" alt="image" src="https://github.com/user-attachments/assets/88cf1d26-7f70-485e-8089-6e39ec9e4bca" />

而 stateless tunnel protocol 就是指說這個 tunnel protocol 你只要給他甚麼 tunnel protocol 的 packet 他就直接拆開然後轉送，沒做任何的驗證或者是握手。

## Impact

那沒有驗證會怎樣，基本上我只要能夠偽造出一個符合這個 tunnel protocol 的封包，我就可以做到將惡意的 packet 透過這個裸奔的 tunnel 轉送到 victim 的內網內，然而過往的研究基本上都是認為這個攻擊是只有單向的，頂多可以做一些 DDoS 而已。

<img width="1675" height="886" alt="image" src="https://github.com/user-attachments/assets/9c57dd15-5db7-438b-9d39-d7871cd189aa" />

<img width="1675" height="697" alt="image" src="https://github.com/user-attachments/assets/21321374-4940-4520-9e9a-618ce5ccb06e" />

包含今年 1 月剛出來的一篇[研究](https://www.top10vpn.com/research/tunneling-protocol-vulnerability/)

<img width="1062" height="759" alt="image" src="https://github.com/user-attachments/assets/359e4baa-1152-4fe7-9445-976b6a74733b" />

這篇研究雖說有講到很多裸奔 stateless tunnel 的掃描方法，以及一些特定型號產品的 router 不會驗證 source ip 的問題，然而主要也都是 focus 在 DDoS 的攻擊面上。

然而事實上只需要把腦袋轉一下：我們其實並不需要來回都要過這個 tunnel 才對。

畢竟攻擊的核心是需要將流量「注入到 Tunnel 內」因此這邊就先取名叫做 Tunnel Injection。

### Tunnel Injection to Internal Network

第一個利用手法是我們可以做到 Interactive Internal Network Access

首先我們偽造出一個符合這個 tunnel protocol 的封包

外層的 source ip 是自己的 public ip 或者一些 tunnel protocol 因為會驗證 source ip 所以這邊也可能需要做 ip spoofing，destination ip 是 victim tunnel 的 public ip。

內層的 source ip 是自己的 ***public ip*** 這個就是這邊的核心了，而 destination ip 就是你想要扁的 victim 內網機器的 ***private ip***。

<img width="1512" height="888" alt="image" src="https://github.com/user-attachments/assets/7a95b8c2-cf6c-4d50-9439-1a7ff17d9f52" />

送出去以後會發生甚麼事呢。

首先 packet 抵達 victim 的 router 時會被解封裝，並把內層的封包依照這台 router 的 routing table 做 forwarding，同時 conntrack table 會留下一條轉發的紀錄。

<img width="1740" height="881" alt="image" src="https://github.com/user-attachments/assets/d0e5e518-0435-42b3-96c1-5529311bfd73" />

然後內網的機器就會看到一個 src = ***public ip*** dest = 自己的 ip 的封包，因此他 response 理所當然會 src = 自己的 ip dest = ***public ip***

<img width="1753" height="829" alt="image" src="https://github.com/user-attachments/assets/33f5e5d2-2954-457b-bae5-e43ba2b12040" />

這個 packet 來到 victim router 的時候，因為 dest = ***public ip*** 所以就會***依照 routing table 做 forwarding 直接走 default gateway 出去***。

這邊同時要補充一點，因為一般的 router 不會對奇怪類型的封包做 SNAT 比如 TCP SYN/ACK、ICMP type 0 aka pong 因此回來的 packet 離開 victim router 的時候是不會被 NAT 也就是說 ***source ip 是 private ip*** 這在某些場景是會有問題的。

那接著我們就可以從網路上收到這個來自 victim 內網的回應。

<img width="1749" height="787" alt="image" src="https://github.com/user-attachments/assets/2a483ec0-2a7d-445a-a8cb-403f24bcd8bc" />

我們就能夠 Arbitrary Interactive Internal Network Access

<iframe width="560" height="315" src="https://www.youtube.com/embed/KSNkpPdzw8o?si=pE8cLkI-OTlZILsh" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

### Tunnel Injection to External Network

在講攻擊手法之前我們先來看看 NAT 一般是怎麼實作的，這邊以 Linux 為例

在 Linux kernel base 的 router 上，一般我們毀使用 `netfilter` 與 `conntrack` 來處理 NAT，而最常見的指令介面會是 `iptables` 與 `nftables`，這邊以 `iptables` 為例

一般情況下要啟動 NAT 我們會下這條 `iptables` 指令

```bash
iptables -t nat -A POSTROUTING -o wan -s 192.168.0.0/16 -j MASQUERADE
```

這條指令的意思是在確定出口網卡時當封包從 ***wan*** 網卡離開時，如果 source ip 符合 192.168.0.0/16 就做 NAT 並使用 wan 網卡的 public ip 作為 source ip。

但是事實上其實很多 router 的廠商都會偷懶，只這樣寫

```bash
iptables -t nat -A POSTROUTING -o wan -j MASQUERADE
```

這條指令的意思是在確定出口網卡時當封包從 ***wan*** 網卡離開時，就做 NAT 並使用 wan 網卡的 public ip 作為 source ip。

可以發現它少了 source ip 的檢查，所以事實上如果 source ip 已經是 public 他也會做 NAT，畢竟 RFC 1918 只是定義而已，沒說你不能拿 public ip 當 private ip 用。

然而如果今天這台 router 同時有一個裸奔的 tunnel 再跑會怎樣呢？

答案是我們會得到一個***免費的跳板***

事實上我們只需要將前面 [Tunnel Injection to Internal Network](#tunnel-injection-to-internal-network) 所說的內層的 destination ip 更改成你要 access 的外網 target public ip 就好。

一樣我們偽造出一個符合這個 tunnel protocol 的封包

外層的 source ip 是自己的 public ip 或 ip spoofing 的 IP，destination ip 是 victim tunnel 的 public ip。

內層的 source ip 是自己的 ***public ip*** ，而 destination ip 這次就改成用***你要 access 的外網 target public ip***。

<img width="1577" height="892" alt="image" src="https://github.com/user-attachments/assets/ffbfa772-3f79-4701-ad49-b968441ba406" />

packet 抵達 victim 的 router 時會被解封裝，並把內層的封包依照這台 router 的 routing table 做 forwarding，同時依照上面 `iptables` 做 NAT 的 source ip 轉換，同時 conntrack table 會留下一條 NAT 的轉換紀錄。

<img width="1547" height="1035" alt="image" src="https://github.com/user-attachments/assets/b120f257-d8b2-4392-a619-80133643455c" />

接著你要連的 target 就會收到你的 packet 並且可能會做 response，因為 source ip 已經因為上面所說的做 NAT 所以 target 看到的會是

src = ***victim public ip*** dest = ***target public ip***

因此 response 會是 src = ***target public ip*** dest = ***victim public ip***

<img width="1548" height="896" alt="image" src="https://github.com/user-attachments/assets/92a6f6f4-293e-42ff-80be-e659a6bd32cd" />

接著 victim router 收到 packet 後會依照 conntrack 的紀錄作 destination ip 的復原，並且依照這台 router 的 routing table 做 forwarding

<img width="1550" height="916" alt="image" src="https://github.com/user-attachments/assets/9ab444c2-9e01-4723-996e-e006786c340c" />

然後你就收到 response 了

<img width="1539" height="920" alt="image" src="https://github.com/user-attachments/assets/9c24ec6f-6eae-4fe0-a7fd-df7588e38fd2" />

我們就能夠 Arbitrary Interactive Pivoting Attack via Tunnel 

<img width="2079" height="1269" alt="image" src="https://github.com/user-attachments/assets/6dfa50fa-0f75-46f5-acf2-b4d52f2a313c" />

<iframe width="560" height="315" src="https://www.youtube.com/embed/_GcIFKyjGmE?si=z7rLzEjtBHx35nQl" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

## RPF or source ip verify bypass

當你以為這樣就結束時現實往往會來一巴掌

還記得前面講 [Tunnel Injection to Internal Network](#tunnel-injection-to-internal-network) 時所說的 

```
一般的 router 不會對奇怪類型的封包做 SNAT 比如 TCP SYN/ACK、ICMP type 0 aka pong 因此回來的 packet 離開 victim router 的時候是不會被 NAT 也就是說 source ip 是 private ip 這在某些場景是會有問題的。
```

而這邊所說的場景就是指 Reverse path forwarding (RPF) 或者 source ip verify

<img width="874" height="875" alt="image" src="https://github.com/user-attachments/assets/e80bb09f-a306-4308-944b-938e24e01da0" />

首先絕大多數的 ISP 為了要防止客戶對外做 IP Spoofing 的攻擊，一般都會針對 client 方向的網卡開啟 RPF

RPF 就是指說，當網卡收到 packet 時，會拿 source ip 對一次 routing tables，如果這個 source ip 並不符合 routing tables 上送往這張網卡的任何規則的話，這個封包就會被 DROP 掉。

因此如果遇到 victim 的 ISP 有啟用這類防護的情況的話你是收不到 response 的，那該怎麼辦

### Bypass for Internal Network Access

這邊我們就來談談 P2P 網路是怎麼做 NAT hole punching 的

首先當雙方的 source port 提前知道，或者可預測的情況（以 p2p 應用程式的情況，會有一個 stun server 協助探測兩邊的 client 使用的 source port 是多少）

雙方可以同時朝對方的 public ip 與 port 發送一個 packet，這邊以 udp 為例

<img width="974" height="292" alt="hownatholepunchwork1" src="https://github.com/user-attachments/assets/590fc3b4-e990-4ebb-8f8d-e44c20f9f30f" />

packet 經過 router 以後會做 NAT 並且同時 conntrack table 會留下一條 NAT 的轉換紀錄。

<img width="1077" height="432" alt="hownatholepunchwork2" src="https://github.com/user-attachments/assets/1a9e7d65-5fc1-44c0-bbed-92361eaed76d" />

兩個封包都各自抵達對方的 router，這邊可以發現抵達的時候 conntrack table 已經有符合條件的 NAT 轉換紀錄。

<img width="1078" height="422" alt="hownatholepunchwork3" src="https://github.com/user-attachments/assets/4e531fdf-9045-44aa-bb86-2b1b7189c7e2" />

因此接著就會依照 conntrack 的紀錄作 destination ip 的復原

<img width="1078" height="422" alt="hownatholepunchwork4" src="https://github.com/user-attachments/assets/368274e7-f142-4270-becf-f3589dfd908a" />

以上就是整個 NAT hole punching 的大概流程，如果說是 tcp 的話會更複雜一點，因為 tcp 是 statefull，有興趣可以去查 TCP Simultaneous Open，剛好今年的 [HITCON CTF 2025 有類似的題目](https://ctf2025.hitcon.org/dashboard/#19) 畢竟 TCP Self Connect 本身也是一種 TCP Simultaneous Open 的特殊案例。

順帶一題我會清楚整個 P2P 傳輸的流程也歸功於我高中時曾經[手刻過整個 P2P 做 NAT hole punching 的流程，也手刻過 STUN server](https://github.com/Jimmy01240397/NetworkServicePackage) ~~雖說現在看他的 code 醜醜的就是了~~，之前有些人總是問我為甚麼要自己造輪子，但是我認為自己手刻輪子也是一種樂趣，並且這樣才能夠真正的學到東西，畢竟使用自己寫出來的工具也是一種樂趣吧？

總之我們可以理解為從內部發 packet 做完 NAT 以後，其時就變相地做完了一個有限制的 port forwarding

再加上我們可以利用 Tunnel Injection 做到單方向任意存取內網或者從內網發 packet，那我們是否可以手動做 NAT hole punching 以後直接從外網做存取呢？

事實上是可以的。以下用 TCP 舉例。

我們利用 Tunnel Injection 在 victim 的內網發送 TCP SYN 來做 hole punching

source ip 是你想要扁的 victim 內網機器的 ***private ip***，而 destination ip 是自己的 ***public ip***。

<img width="1797" height="864" alt="image" src="https://github.com/user-attachments/assets/34fa43d2-f10d-4c30-bf6b-9bba2537b88d" />

packet 抵達 victim 的 router 時會依照這台 router 的 routing table 做 forwarding，接著做 NAT 的 source ip 轉換，同時 conntrack table 會留下一條 NAT 的轉換紀錄。

這樣 hole punching 就完成了

<img width="1894" height="769" alt="image" src="https://github.com/user-attachments/assets/68ca12ef-176c-42c8-9306-406007a1eaa2" />

接著朝著這個打好的洞發 tcp SYN 

source ip 是自己的 ***public ip***，而 destination ip 是你想要扁的 victim 內網機器的 ***private ip***。

<img width="1894" height="750" alt="image" src="https://github.com/user-attachments/assets/6a05f51d-89e9-4b54-a0d8-a1af4b66f30e" />

packet 會依照 conntrack 的紀錄作 destination ip 的復原，並且依照這台 router 的 routing table 做 forwarding 轉到 victim 的內網機器

<img width="1896" height="853" alt="image" src="https://github.com/user-attachments/assets/9da739d2-8185-4c74-a558-1268905a25b5" />

victim 的內網機器收到後會回 TCP SYN/ACK

<img width="1880" height="737" alt="image" src="https://github.com/user-attachments/assets/dd125103-65b5-4ff8-a28d-106f12453ba6" />

並且 source ip 會很正常的 NAT 成 victim 的 public ip

<img width="1906" height="712" alt="image" src="https://github.com/user-attachments/assets/dfc538ec-f8bd-4f92-ad21-d53ad41c4f77" />

接著 attacker 就回 ACK

<img width="1876" height="797" alt="image" src="https://github.com/user-attachments/assets/a2f3b7a5-61fd-45f8-a810-3076ed283495" />

然後 connection 就建立起來了，可以快樂傳資料到內網了

<img width="1922" height="808" alt="image" src="https://github.com/user-attachments/assets/30dec908-b538-4163-8151-7adc197584d2" />

然而上面的流程其實只有對 UDP 完全成立。

TCP 只有部份的 router 或較舊的 Linux kernel 成立而已。

主要是因為現今的 Linux kernel 的 conntrack 做 TCP Simultaneous Open 會嚴格檢查封包傳輸的行為是否符合 TCP Simultaneous Open 的行為，如果有一邊的 SYN/ACK 沒收到，你會發現他的 state 會永遠停留在 `SYN_SENT2`，這時候內網對外發 TCP PUSH 的時候***依然不會被 NAT 成 victim 的 public ip***

因為上面的流程會少 attacker 發向 victim 內網機器的 SYN/ACK 因此永遠不會 ESTABLISHED

那如果我補發 attacker 發向 victim 內網機器的 SYN/ACK 是否就能解決呢？

由於 conntrack 只要看到任何 TCP RST 就會直接把這條紀錄轉成 close state 並且在幾秒後 free 掉，並且一般的系統在收到來源不明，或者是不符合握手流程的 TCP 包會直接回 TCP RST（因為 TCP SYN 是我們手動利用 Tunnel Injection 打出去的，因此 victim 內網機器壓根不認識這個 SYN/ACK）

雖說這種作法在一些會 DROP 掉 TCP RST 的內網環境可能是可以成功的，但是這邊還是以預設狀況為主比較好。

那該怎麼解決問題呢

仔細看 sysctl 的參數我們會發現 conntrack 的 tcp close 紀錄的壽命只有 10 秒，之後紀錄就會被 free 掉

<img width="666" height="49" alt="image" src="https://github.com/user-attachments/assets/fcab59ab-27a7-4a7f-a1e5-10c82ae13728" />

然後還有一個有趣的參數是 net.netfilter.nf_conntrack_tcp_loose 

<img width="587" height="48" alt="image" src="https://github.com/user-attachments/assets/84632096-8e77-4e61-bb89-35a1617bfbc3" />

這個參數預設是 1

那他有甚麼意思呢？

當 net.netfilter.nf_conntrack_tcp_loose 為 1 時，允許在 conntrack 沒有任何符合條件的紀錄時收到 PUSH 或 ACK 做 NAT

也就是說當我 conntrack 沒有任何符合條件的紀錄時，內網向外發一個 TCP PUSH 或 ACK 他會直接生出一個 ESTABLISHED state 的 NAT 紀錄

那這就非常好玩了，因為 close state 時間夠短，並且 TCP 是會重傳的，我們可以手動發 TCP RST 來 free 掉 conntrack record 然後內網發 PUSH 出來就直接變成 ESTABLISHED state

所以這邊在講一次流程：

我們利用 Tunnel Injection 在 victim 的內網發送 TCP SYN 來做 hole punching

source ip 是你想要扁的 victim 內網機器的 ***private ip***，而 destination ip 是自己的 ***public ip***。

<img width="1797" height="864" alt="image" src="https://github.com/user-attachments/assets/34fa43d2-f10d-4c30-bf6b-9bba2537b88d" />

packet 抵達 victim 的 router 時會依照這台 router 的 routing table 做 forwarding，接著做 NAT 的 source ip 轉換，同時 conntrack table 會留下一條 NAT 的轉換紀錄。

<img width="1894" height="769" alt="image" src="https://github.com/user-attachments/assets/68ca12ef-176c-42c8-9306-406007a1eaa2" />

接著朝著這個打好的洞發 tcp SYN 

source ip 是自己的 ***public ip***，而 destination ip 是你想要扁的 victim 內網機器的 ***private ip***。

<img width="1894" height="750" alt="image" src="https://github.com/user-attachments/assets/6a05f51d-89e9-4b54-a0d8-a1af4b66f30e" />

packet 會依照 conntrack 的紀錄作 destination ip 的復原，並且依照這台 router 的 routing table 做 forwarding 轉到 victim 的內網機器

<img width="1896" height="853" alt="image" src="https://github.com/user-attachments/assets/9da739d2-8185-4c74-a558-1268905a25b5" />

victim 的內網機器收到後會回 TCP SYN/ACK

<img width="1880" height="737" alt="image" src="https://github.com/user-attachments/assets/dd125103-65b5-4ff8-a28d-106f12453ba6" />

並且 source ip 會很正常的 NAT 成 victim 的 public ip

<img width="1906" height="712" alt="image" src="https://github.com/user-attachments/assets/dfc538ec-f8bd-4f92-ad21-d53ad41c4f77" />

接著 attacker 就回 ACK

<img width="1860" height="796" alt="image" src="https://github.com/user-attachments/assets/a0f2a086-4755-4169-9085-daf7b523b6fe" />

我們這邊發一個 PUSH 進去，可能是 HTTP 的 request 之類的

<img width="1890" height="808" alt="image" src="https://github.com/user-attachments/assets/3f55be71-accc-4b0e-8aa1-878ac136cc92" />

因為上面講的機制，內網出來的 PUSH 會被 DROP，attacker 沒收到資料自然不會回 ACK，victim 內網機器沒收到 ACK 就會一直重傳 TCP PUSH

<img width="1848" height="790" alt="image" src="https://github.com/user-attachments/assets/ef676fa9-b7d5-492b-86bd-ffd606b8c0bb" />

接著我們利用 Tunnel Injection 從內網打一個 TCP RST 出來，手動 free 掉這條 conntrack record

<img width="1928" height="762" alt="image" src="https://github.com/user-attachments/assets/7671e6c4-8041-4034-b9da-93fd323e0b1d" />

Free 掉以後當內網重傳 TCP PUSH

<img width="1744" height="820" alt="image" src="https://github.com/user-attachments/assets/115257d7-8133-4b60-9dd9-3177cbb7583d" />

就會成功被 NAT 並且 conntrack table 會留下一條 NAT ***ESTABLISHED state*** 的轉換紀錄。

<img width="1940" height="827" alt="image" src="https://github.com/user-attachments/assets/b04c2350-c5b4-428f-b55d-7f665be6a19e" />

然後 connection 就建立起來了，可以快樂傳資料到內網了

<img width="1922" height="808" alt="image" src="https://github.com/user-attachments/assets/30dec908-b538-4163-8151-7adc197584d2" />

以上我們就成功做到 RPF bypass 的 Internal Network Access 了。


