---
title: "MPLS 運作原理"
date: 2030-07-15T00:00:00+08:00
draft: false
github_link: ""
author: "Chumy"
tags:
  - Networking
  - Infra
image: ""
description: ""
toc: 
---

# MPLS 運作原理
多重通訊協定標籤交換傳輸（Multi-Protocol Label Switching）是一種網路封包的路由技術，本意是要優化傳統 IP 路由所消耗的效能。

## 傳統 IP 路由方式
傳統的 IP 路由通常是依照 dst IP address 去比對 router 的 routing tables，並且找到最優的 rule 再來像指定的 gateway 進行傳遞。

比如說假設 A 有一個 packet 要送到 G，A 會先對本機的 routing tables 找送到 G 的最優路徑，比如說 [G-X] -> Z [G-H] -> B 之類的，因為 [G-H] -> B 的搜索範圍比較小，所以就會把這個包送給 B。

![image](https://github.com/user-attachments/assets/014e5dcc-8706-473f-979e-425b47501d54)

接著 B 收到以後一樣會先對本機的 routing tables 找送到 G 的最優路徑，依照 rule 對應到 D，然後就會丟給 D。

![image](https://github.com/user-attachments/assets/9fae6115-475f-43ef-9c86-3fa4e7270f3d)

然後依此類推直到送到目的地。

![image](https://github.com/user-attachments/assets/bd446113-b142-48be-a7fd-f0df8e2161d5)

## 標籤與標籤交換
但是這種傳輸方式會有一個問題，就是當你的 routing tables 過於複雜，同個 IP 可能符合多個規則需要篩選最優解時，會消耗過多的效能。因此有一種比較簡單的方式就是利用標籤交換（Label Switching）。



