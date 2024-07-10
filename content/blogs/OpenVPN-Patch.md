---
title: "從 Wireshark 到 patch OpenVPN driver"
date: 2023-04-08T00:00:00+08:00
draft: false
github_link: "https://github.com/OpenVPN/tap-windows6/issues/158"
author: "Chumy"
tags:
  - Networking
  - Infra
  - Open Source Contribution
image: /images/ovpn.png
description: ""
toc: 
---

## 緣由
當初會這樣是因為接到一個很有趣的 case 

![image](https://github.com/Jimmy01240397/Chumy-Blog/assets/57281249/c20e43c3-3ed9-46ad-8c94-657afc66cd3c)
![image](https://github.com/Jimmy01240397/Chumy-Blog/assets/57281249/4d1df898-0b4a-4a95-ac08-82b658b97a2c)
![image](https://github.com/Jimmy01240397/Chumy-Blog/assets/57281249/acc1825f-0a69-4ec4-8a2e-fa84166df57c)

簡單來說就是想要 client 可以打 VPN 到另一邊，並且讓 PPPoE 可以透過 VPN 拿到中華電信的 public IP address。

![未命名绘图 drawio (63)](https://github.com/Jimmy01240397/Chumy-Blog/assets/57281249/d1df4baf-2101-4aee-87d8-555316cc8d23)

至於他們原本的架構，也是很有趣，首先會找人重架的原因是因為他們說接 VPN 後網路很不穩定，speedtest 也無法測速的那種。

於是我問了架構發現他們架構很酷。

首先他們是用那種小電腦做 server，如下圖。

![image](https://github.com/Jimmy01240397/Chumy-Blog/assets/57281249/9f9de987-f4fd-499c-9ca5-f1d345b64b80)

而裡面長這樣。

![image](https://github.com/Jimmy01240397/Chumy-Blog/assets/57281249/339e3dde-62a9-4679-9cb5-5b12bcad7151)

就是他小電腦裝 windows 10，裡面只有裝 VMware，然後開兩台 windows 的 VM，一台是 windows 7 做 VPN server，裡面用 softether vpn，另一台 windows 10 我不知道用途，他們說測試用。

然後我就想這一台小電腦塞那麼多台 windows 不當才怪，再加上 softether vpn 沒架過並沒有很熟，也不確定這個效能好不好。

於是我就推薦他們 host 機可以用 Proxmox VE 做虛擬化環境，然後 VPN 的部分由於 PPPoE 是 L2 的協定，需要 ethernet header，因此 VPN 的選擇上就需要用 L2 Tunnel。關於甚麼是 L2 Tunnel 可以去看[這篇文章](https://www.comparitech.com/blog/vpn-privacy/tun-tap-adapters/)與[這篇文章](https://ipwithease.com/layer-2-vs-layer-3-vpn/)。

這邊我選擇用 OpenVPN tap 作為 L2 Tunnel。

因此架構就改成 Proxmox VE 裡面裝 Debian 跟 Windows 10，Debian 用 OpenVPN tap。

## 過程
於是乎我開始架設 VPN 架設的 config 可以參考[這裡](https://github.com/Jimmy01240397/Chumy_Note/tree/master/Linux/openvpn/L2TunnelWithBridge)。

![image](https://github.com/Jimmy01240397/Chumy-Blog/assets/57281249/99f456b3-72b5-47cc-ab8b-52506897f9a7)

![image](https://github.com/Jimmy01240397/Chumy-Blog/assets/57281249/ceb4cee3-e651-4ba9-8ad8-778a0d787fc0)

![image](https://github.com/Jimmy01240397/Chumy-Blog/assets/57281249/38bdd06e-8d04-4f55-98f8-f81f9f19241e)

架好的我開開心心的用 windows 測試

![image](https://github.com/Jimmy01240397/Chumy-Blog/assets/57281249/42470eb1-e58b-45f9-9900-f51110c465f5)

![image](https://github.com/Jimmy01240397/Chumy-Blog/assets/57281249/605861b4-ef47-4c3f-9a78-2ee755807b28)

![image](https://github.com/Jimmy01240397/Chumy-Blog/assets/57281249/953af183-6d55-4bb7-9b4c-d15b6d8d74df)

![image](https://github.com/Jimmy01240397/Chumy-Blog/assets/57281249/d792d5ad-cf42-46a0-be71-2add2822aaeb)

到這邊我想說拿到 IP 了應該能動了，結果 ping 一下發現出不去。

![image](https://github.com/Jimmy01240397/Chumy-Blog/assets/57281249/5e72324b-897f-4e73-85ea-2206df3859e2)

這就很奇怪了，於是乎我 ping 看看 ipv6 看看是不是一樣的問題，結果可以通。

![image](https://github.com/Jimmy01240397/Chumy-Blog/assets/57281249/bdfcfaf2-9c57-42cb-beee-27744ba0375d)

這下有趣了，V4 沒通 V6 有通，怎麼想都不正常，再加上我兩個都有拿到 IP。

![image](https://github.com/Jimmy01240397/Chumy-Blog/assets/57281249/aaece719-a68f-43b4-b3f0-15a025e243eb)

而作為一個遇到問題會死命 debug 的我直接就開始抓包測試。

可以發現 ICMP ping 確實有封裝上 PPPoE header 並且從 VPN 網卡送出，且 VPN server 是有收到並轉發出去的，但是遲遲沒收到 response，因此可能是上游因為某些原因，可能不認可我們持有的 V4 IP 然後 drop 掉，但是認可 V6 的 IP。

![image](https://github.com/Jimmy01240397/Chumy-Blog/assets/57281249/93f7dc97-4f4d-435e-bf5e-75bef15981f7)

![image](https://github.com/Jimmy01240397/Chumy-Blog/assets/57281249/564b1f61-a22b-4786-be6d-7c45d0959b59)

因此就要來想一下為何上游不認可這個 V4 IP，在思考很久後得出一個可能是因為有些包在中途掉包而導致 PPPoE server 沒收到某個 ACK 導致這個問題的發生。

所以就要拿 client 跟 server 的 pppoe pcap 分析來比對

![image](https://github.com/Jimmy01240397/Chumy-Blog/assets/57281249/ed8ef034-30f5-4240-829e-12145c9ade6b)

然後就可以發現有一個 ACK 送到 server 就不見了，這就有趣了，如此有規律的有特定包掉包，這基本上可以排除因為碰撞或資料有誤導致的自然掉包的可能。

這邊就有一個問題了，linux 作為 client 會不會有這種情況發生。

所以這邊就來測試。

![image](https://github.com/Jimmy01240397/Chumy-Blog/assets/57281249/1fa5490a-519e-45d9-8bcb-547445ed3d80)

![image](https://github.com/Jimmy01240397/Chumy-Blog/assets/57281249/c198b678-bfbc-4799-b742-6419dfa23791)

![image](https://github.com/Jimmy01240397/Chumy-Blog/assets/57281249/76a02d48-b81b-43e0-946c-cc315a25481d)

![image](https://github.com/Jimmy01240397/Chumy-Blog/assets/57281249/7d768f82-ef58-4886-b34d-4e95717b8772)

可以發現 linux 是完全正常的 PPPoE 也是通的，沒有發生 windows 的那種情況，那這代表是 windows 的 OpenVPN 網卡 driver 有問題。

那接下來就要想問題點了。

![image](https://github.com/Jimmy01240397/Chumy-Blog/assets/57281249/6a8c7594-6938-43cc-935c-fef504480ba1)

為何他如此與眾不同，這邊就可以聯想到這個 packet 是所有 PPPoE packet 中最小的，長度只有 32 byte。

![image](https://github.com/Jimmy01240397/Chumy-Blog/assets/57281249/1823efca-2831-4673-8974-282440bb2857)

因此我猜測是因為長度問題，可能 driver 某個地方有一個如果發現 packet 長度過短就 drop 掉的 code。

所以我就用 scapy 寫了一個可以發送很小的 ethernet frame 的測試程式來做測試。

```python
from scapy.all import *
import sys

IFACES.show() # let’s see what interfaces are available. Windows only
iface = IFACES.dev_from_index(35)
socket = conf.L2socket(iface=iface)
payload = Ether(dst = sys.argv[2], src = sys.argv[1])/Raw(load=sys.argv[3].encode())
print(bytes(payload))
print(len(bytes(payload)))
socket.send(payload)
```

先發一個長度 40 的包。

![image](https://github.com/Jimmy01240397/Chumy-Blog/assets/57281249/a78c31c2-c4b2-4a5c-aded-d4603d2d5cd1)

![image](https://github.com/Jimmy01240397/Chumy-Blog/assets/57281249/a45f523d-0441-405e-913a-9226f05e576d)

發現 server 有收到，改發長度 20 的包。

![image](https://github.com/Jimmy01240397/Chumy-Blog/assets/57281249/3cdaa8b3-4ab1-46dd-bef2-aad0d04d0442)

![image](https://github.com/Jimmy01240397/Chumy-Blog/assets/57281249/2d31fe5b-2fdc-43ef-8049-08e3dc9ade77)

會發現 server 沒有收到，改發長度 30 的包。

![image](https://github.com/Jimmy01240397/Chumy-Blog/assets/57281249/52d60a30-0e3c-4986-b0fe-d3c7f93b45c1)

![image](https://github.com/Jimmy01240397/Chumy-Blog/assets/57281249/43f34a8d-d2ac-4a54-8d4c-99338786cab8)

會發現 server 還是沒有收到，改發長度 35 的包。

![image](https://github.com/Jimmy01240397/Chumy-Blog/assets/57281249/21a980cb-cb73-4865-a4bb-00eb0f4fc4f2)

![image](https://github.com/Jimmy01240397/Chumy-Blog/assets/57281249/1b87c1b9-2f26-439d-8ced-5e1e7254109d)

會發現 server 收到了，改發長度 33 的包。

![image](https://github.com/Jimmy01240397/Chumy-Blog/assets/57281249/84416f87-3642-49f0-b2ef-280bd083c2eb)

![image](https://github.com/Jimmy01240397/Chumy-Blog/assets/57281249/8791b7f9-6e40-4f23-935e-fee3a50987d0)

會發現 server 又沒有收到，改發長度 34 的包。

![image](https://github.com/Jimmy01240397/Chumy-Blog/assets/57281249/92fe6f69-1d75-423e-87e8-e8031bdf5d4c)

![image](https://github.com/Jimmy01240397/Chumy-Blog/assets/57281249/34559391-8bc9-4f39-b652-f974a5ce1ff8)

會發現 server 又收到了。

因此我們可以確定最小可以接受的 packet length 是 34，由於 IPv4 的 configuration ACK 大小是 32，因此再送往 server 前就被 windows 的 driver 給 drop 掉了，也就是。

```python
if len(packet) < 34:
    drop(packet)
```

因此我就去搜 [tap-windows6](https://github.com/OpenVPN/tap-windows6) 這個 repo 裡有沒有地方是 34 的。

![image](https://github.com/Jimmy01240397/Chumy-Blog/assets/57281249/2a3cfae7-2ae8-4b9d-99fc-063b858228d5)

![image](https://github.com/Jimmy01240397/Chumy-Blog/assets/57281249/f946dad1-1054-4cf5-8c91-285c42a09c12)

但是沒發現任何東西，於是慢慢減少數字看看有沒有甚麼線索，直到找到 20 我看到一個有趣的東西。

![image](https://github.com/Jimmy01240397/Chumy-Blog/assets/57281249/3e9f9647-44ac-4872-a42b-b65b514cf0ff)

![image](https://github.com/Jimmy01240397/Chumy-Blog/assets/57281249/405c801d-7ab0-437b-a9f4-71b009815683)

這時我就聯想到 34 這個數字剛好是 ethernet header + ipv4 header 的長度，因此我馬上對整個專案搜尋 `IP_HEADER_SIZE`，會發現馬上就出現了一些很有趣的內容。

![image](https://github.com/Jimmy01240397/Chumy-Blog/assets/57281249/37c75a9a-1e02-476c-884a-92e77a976b1a)

其中 `txpath.c` 上面的 `< (ETHERNET_HEADER_SIZE + IP_HEADER_SIZE)` 只是做分類而已，因此不影響重點是下面兩個。

![image](https://github.com/Jimmy01240397/Chumy-Blog/assets/57281249/01ae29e9-2c36-4766-aa47-6aba8549500e)

首先我們可以很確定這個 function 是對 tap adapter 做處理，閱讀 function name `tapNetBufferListNetBufferLengthsValid` 可以猜測這是驗證 packet 長度的 function，那問題來了，理論上 tap adapter 只要有 ethernet header 都可以傳的，那為何需要驗證 `IP_HEADER_SIZE`。

於是我把它改成這樣。

![image](https://github.com/Jimmy01240397/Chumy-Blog/assets/57281249/706e5025-4c14-43c8-940f-41f1a2df6963)

並且發了 [PR](https://github.com/OpenVPN/tap-windows6/pull/159)。

一個很有趣的小插曲是，在我還沒發現 code 有問題前我就有發 [issue](https://github.com/OpenVPN/tap-windows6/issues/158) 了，說 Packet lost for pppoe over openvpn tap，但是他們並沒有採信。

![image](https://github.com/Jimmy01240397/Chumy-Blog/assets/57281249/1c28dab1-55a5-409c-ad49-8ada7e385cfe)

![image](https://github.com/Jimmy01240397/Chumy-Blog/assets/57281249/5286b5c0-5c94-4ffa-9b94-52974d33bf0e)

![image](https://github.com/Jimmy01240397/Chumy-Blog/assets/57281249/2d9604df-c9f1-417f-a993-9fe032709a2e)

最後我找到問題並發了 [PR](https://github.com/OpenVPN/tap-windows6/pull/159) 他們才發覺真的有這問題。 

![image](https://github.com/Jimmy01240397/Chumy-Blog/assets/57281249/5cab908e-56c8-4a5e-a3e6-a56aaf6b5aaa)

![image](https://github.com/Jimmy01240397/Chumy-Blog/assets/57281249/7b0bb026-6499-4629-b1b0-cfae6d982335)

並且經過了一寫討論與測試並發布後，我測試完 PPPoE 就正常了。

![image](https://github.com/Jimmy01240397/Chumy-Blog/assets/57281249/4378dfb8-8ab5-4f2f-8839-3f7741b12955)

![image](https://github.com/Jimmy01240397/Chumy-Blog/assets/57281249/da3d5e33-18e8-4b23-b50f-a7c6ca1bc926)

![image](https://github.com/Jimmy01240397/Chumy-Blog/assets/57281249/88ea778f-b45f-48c2-a0fb-993c473a8607)

