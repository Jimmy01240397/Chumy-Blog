---
title: "AIS3 EOF 2025 Final 心得"
date: 2025-02-20T12:00:00+08:00
draft: false
github_link: "https://github.com/Jimmy01240397/CTF-writeup"
author: "Chumy"
tags:
  - CTF
image: /images/CTF.png
description: ""
toc: 
---

# HITCON CTF 2024 Final 心得

這波 EOF 自幹全套 A&D 工具兼首次實戰測試大成功，只差 ELK 沒整併進來了，但能夠把上次 HITCON CTF 的一些想法實現出來也是很不錯。

round alert 實在方便甚至跳通知比視覺化快，讓 葉東逸 跟 vincent55 可以用最快速度登入 game 完成任務。

![image](https://github.com/user-attachments/assets/1f733fb5-aa4b-4630-995e-7e9bb8c43b47)

attack manager 第一天運行的很完美，舒服的顯示哪一隊這輪有打穿，這樣就能迅速應對更新 exploit 。

![image](https://github.com/user-attachments/assets/ef8e4bbe-0b84-4e10-bfba-f58b546e0663)

pcap analysis 也是完美的將所有的流量整理好，也完美的分類每台電腦的流量，只可惜還是需要導入 elk 才能看。

![image](https://github.com/user-attachments/assets/a481b619-c2cb-4ce9-8e4e-a8e8705be99e)

patch manager 第一天壞掉所以一直修，但是好險第一天沒有太多人在 patch，當天晚上將他修好測好以後第二天就完美運行，讓 三角蛇 可以快樂追 patch。但是最可惜的是賽前 WeakGod Chiu 準備的 binary diff 沒有用到。

![image](https://github.com/user-attachments/assets/35687ed3-9e9a-4d8f-816e-a28733fddd84)

![image](https://github.com/user-attachments/assets/352f3f21-1ace-4ea8-8f9d-7ddb8b704fe3)

另外我們的超讚 crypto jeopardy 隊友 李彥佑 也是第一天就解完了超讚的。

另外第一次試著戴一台小 server 來充當 router 兼 NAS 也是一個蠻成功的，畢竟能夠內網傳檔案也不用在自己電腦接 vpn 去比賽環境還是比較方便的。

最後有些人問我為啥不用現成的，要客製化自己想要的功能直接自幹肯定是最快的對ㄅ（X

最後感謝 楊竣鴻 幫我寫前端♥️

![image](https://github.com/user-attachments/assets/95614bac-cf9e-4a92-84a7-fd04502ce0f0)

![image](https://github.com/user-attachments/assets/3075e49b-1a67-43e2-95a3-b84c8fce7684)

