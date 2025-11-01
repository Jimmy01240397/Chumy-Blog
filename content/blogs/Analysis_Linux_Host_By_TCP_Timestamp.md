---
title: "利用 TCP 的 Timestamps 做到精準分類來源 linux machine"
date: 2025-02-20T14:00:00+08:00
draft: false
author: "Chumy"
tags:
  - Networking
  - Research
image: ""
description: ""
toc: 
---

# 利用 TCP 的 Timestamps 做到分類來源 linux machine

## 緣由

會做這個的原因是因為要打今年的 [AIS3 EOF](/blogs/ais3-eof-2025/) 要做封包的分析工具，然而按照 [HITCON CTF](/blogs/hitconctf-2024-final/) 與往年 AIS3 EOF 的經驗，我們是沒辦法取得每個隊伍的 SRC IP 的，也就是說無法辨識出這個 request 是屬於哪一個隊伍的。

這會造成甚麼問題，假設有一個 HTTP Post Auth 的 RCE 好了，可能會需要這些流程。

1. 註冊
2. 登入
3. 前置設定
4. RCE

如果是走 HTTP/1.0 這樣至少會有 4 個 connection，沒辦法辦識的話在 source port 都不同的情況下要光看這個 pcap 幾乎是很難直接看出這個漏洞怎觸發的，因為我們可能可以用 flag 去 filter 出 RCE 的 connection 但是沒辦法知道前面有哪些動作（註冊、登入、前置動作）。

為了解決這個問題開始研究 TCP header 與 IP header 的一些 field 有沒有可以利用的，而這邊又有一些限制，因為拿到的 pcap 的 TCP 有可能不是完整的 connection，可能會沒有 SYN 包，如果用只有 SYN 才有的 field 去過濾會不太精準。

## 正題

原本會去看 Timestamp 是因為想說有沒有可能用 TCP 的 RTT 來分類隊伍，但是意外發現了很有趣的特性，首先來看看這個封包

![image](https://github.com/user-attachments/assets/1bf4cb92-14d1-4c37-b8dc-837f16cbf4f8)

我們有 TSVal: 871787126 如果用 unix timestamp 轉成時間的話會發現

![image](https://github.com/user-attachments/assets/0cea336b-3efc-4121-ad2e-d733b65e610c)

這個時間好像怪怪的，可能是有某種 offset 在，因此去稍微看一下 [linux kernel](https://github.com/torvalds/linux) 是如何實作這個 TCP timestamp 的部分。

慢慢地來追一下 code

[__tcp_transmit_skb](https://github.com/torvalds/linux/blob/87a132e73910e8689902aed7f2fc229d6908383b/net/ipv4/tcp_output.c#L1333)

[tcp_syn_options](https://github.com/torvalds/linux/blob/87a132e73910e8689902aed7f2fc229d6908383b/net/ipv4/tcp_output.c#L853)

[tcp_established_options](https://github.com/torvalds/linux/blob/87a132e73910e8689902aed7f2fc229d6908383b/net/ipv4/tcp_output.c#L999)

我們可以發現他使用了 `tp->tsoffset` 作為他的 offset，那這個 `tp->tsoffset` 是怎算出來的呢？

[tcp_v4_connect](https://github.com/torvalds/linux/blob/87a132e73910e8689902aed7f2fc229d6908383b/net/ipv4/tcp_ipv4.c#L331)

[secure_tcp_ts_off](https://github.com/torvalds/linux/blob/87a132e73910e8689902aed7f2fc229d6908383b/net/core/secure_seq.c#L121)

這邊會發現它會使用 `ts_secret` 這個 `ts_secret` 基本上就是這台機器開機後第一次打 tcp connection 的時候就會隨機生成。

然後他會拿這個 `ts_secret` 與 `source ip` 跟 `destination ip` 算一個 hash 出來做為 offset

```C
ts_secret_init();
return siphash_2u32((__force u32)saddr, (__force u32)daddr,
		    &ts_secret);
```

這下有趣了，每台機器的 `ts_secret` 基本上都是不同的，加上 source ip 不一定會相同，而且我們隊伍同一個 challenge 的 destination ip 絕對是相同的，這代表如果可以忽略掉時間，**我們可以利用這個 offset 分類出這台機器到我們服務的所有 connection**

這邊做一下驗證

![image](https://github.com/user-attachments/assets/fb81c5df-620b-4fcb-8209-61520af3171c)

![image](https://github.com/user-attachments/assets/124e1eb9-ff40-4738-9a89-962fe966399d)

![image](https://github.com/user-attachments/assets/26e2f2e6-9c3e-426c-9a2b-993f92721345)

![image](https://github.com/user-attachments/assets/5d257381-5153-4686-8934-d84f8d43746b)

![image](https://github.com/user-attachments/assets/8af7c009-e5d0-4ae2-9f65-5c4e2c0219d6)

![image](https://github.com/user-attachments/assets/de697614-0ba5-4a98-8fd4-186a04715fdd)

![image](https://github.com/user-attachments/assets/e160edbf-1427-4786-8f2e-5b86b0e5ae93)

![image](https://github.com/user-attachments/assets/e20673eb-0fb8-4ef9-9fbb-50c10974069b)

因為一 round 大約 5 分鐘，所以我取前 3 位 hex 作為 user id

![image](https://github.com/user-attachments/assets/fdfd99eb-3d89-4af1-97fc-c46041e92079)

這樣一來就可以完美的分類出同一台來源 linux 機器的所有 tcp connection，比如 NAS 題的登入後 LFI 我完全只看 PCAP 就看出來了

![image](https://github.com/user-attachments/assets/f74d9de9-6c0d-45c6-bf16-2c2e763479ea)

![image](https://github.com/user-attachments/assets/ef1d75a3-4562-466b-8ed7-9d72187eb868)

甚至於 passive mode ftp 的 control port 跟 transfer port 的 connection 也能完美對應出來。

這導致我們這隊看封包方面上幾乎跟開外掛一樣XXD

~~希望明年比賽不會把 tcp header 的 field 抹掉~~

