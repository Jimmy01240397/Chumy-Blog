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

那沒有驗證會怎樣，基本上我只要能夠偽造出一個符合這個 tunnel protocol 的封包，我就可以做到將惡意的 packet 透過這個裸奔的 tunnel 轉送到對方的內網內，然而過往的研究基本上都是認為這個攻擊是只有單向的，頂多可以做一些 DDoS 而已。

<img width="1675" height="886" alt="image" src="https://github.com/user-attachments/assets/9c57dd15-5db7-438b-9d39-d7871cd189aa" />

<img width="1675" height="697" alt="image" src="https://github.com/user-attachments/assets/21321374-4940-4520-9e9a-618ce5ccb06e" />

包含今年 1 月剛出來的一篇[研究](https://www.top10vpn.com/research/tunneling-protocol-vulnerability/)

<img width="1062" height="759" alt="image" src="https://github.com/user-attachments/assets/359e4baa-1152-4fe7-9445-976b6a74733b" />

這篇研究雖說有講到很多裸奔 stateless tunnel 的掃描方法，以及一些特定型號產品的 router 不會驗證 source ip 的問題，然而主要也都是 focus 在 DDoS 的攻擊面上。

然而事實上只需要把腦袋轉一下：我們其實並不需要來回都要過這個 tunnel 才對。

畢竟攻擊的核心是需要將流量「注入到 Tunnel 內」因此這邊就先取名叫做 Tunnel Injection。

## Tunnel Injection to Internal Network

第一個利用手法是我們可以做到 Interactive Internal Network Access

首先我們偽造出一個符合這個 tunnel protocol 的封包

外層的 source ip 是自己的 public ip 或者一些 tunnel protocol 因為會驗證 source ip 所以這邊也可能需要做 ip spoofing，destination ip 是對方 tunnel 的 public ip。

內層的 source ip 是自己的 ***public ip*** 這個就是這邊的核心了，而 destination ip 就是你想要扁的對方內網機器的 ***private ip***。

<img width="1512" height="888" alt="image" src="https://github.com/user-attachments/assets/7a95b8c2-cf6c-4d50-9439-1a7ff17d9f52" />

送出去以後會發生甚麼事呢。

首先 packet 抵達對方的 router 時會被解封裝，並把內層的封包依照這台 router 的 routing table 做 forwarding，同時 conntrack table 會留下一條轉發的紀錄。

<img width="1740" height="881" alt="image" src="https://github.com/user-attachments/assets/d0e5e518-0435-42b3-96c1-5529311bfd73" />

然後內網的機器就會看到一個 src = ***public ip*** dest = 自己的 ip 的封包，因此他回信理所當然會 src = 自己的 ip dest = ***public ip***

<img width="1753" height="829" alt="image" src="https://github.com/user-attachments/assets/33f5e5d2-2954-457b-bae5-e43ba2b12040" />

這個 packet 來到 router 的時候，因為 dest = ***public ip*** 所以就會***依照 routing table 做 forwarding 直接走 default gateway 出去***。

這邊同時要補充一點，因為一般的 router 不會對奇怪類型的封包做 SNAT 比如 TCP SYN/ACK、ICMP type 0 aka pong 因此回來的 packet 離開 router 的時候是不會被 NAT 也就是說 ***source ip 是 private ip*** 這在某些場景是會有問題的。

那接著我們就可以從網路上收到這個來自對方內網的回應。

<img width="1749" height="787" alt="image" src="https://github.com/user-attachments/assets/2a483ec0-2a7d-445a-a8cb-403f24bcd8bc" />

我們就能夠 Arbitrary Interactive Internal Network Access

<iframe width="560" height="315" src="https://www.youtube.com/embed/KSNkpPdzw8o?si=pE8cLkI-OTlZILsh" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

## Tunnel Injection to External Network
