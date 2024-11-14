---
title: "HITCON CTF 2024 Final 心得"
date: 2024-11-11T00:00:00+08:00
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

承接 [Quals](/blogs/hitconctf-2024-writeup) 來聊聊決賽吧！

## 前言

這次參加 HITCON CTF Final 也算是我第一次參加這種比較大型的 CTF 決賽比賽，而我基本上負責規劃與設定整個隊伍的基礎建設，並提供機器給隊友架設並設定對內的自動化攻擊與防禦系統如 attack manager 或封包工具與攻擊重送工具。由於考慮到這場有提供 game box 且不會用到會場內網然後賽中我可能要顧 infra 加上 Ching 通靈到可能有 live CTF 所以我就在場外的飯店房間總部打比賽。

## 賽前準備

賽前一個月先把整個主要架構架好上好 ldap、file server、VPN 等等測試完寫好 document 然後交給 Caleb 架 attack manager 與 elk、Curious 架抓封包與重送工具，然後 Ching 架模擬環境給隊員學工具使用方式。

## 賽前一天

首先比賽前一天因為剛好去取貨撿到的二手 L3 switch 想說飯店如果有網路孔會很方便還能直接打 sitetosite 就扛了一台 7 公斤的東西過去，結果房間沒網路孔只有 wifi

晚上十點多規則放出來就要馬上工作當水管工把 tunnel 打通路由設通 MTU 調好然後測試，然後再應 Vincent55 與 kazma 需求把內網也串到社團內網上（外加幫隊友修網路（X）中途設到一半突然 jumpbox 斷線差點嚇死，結果看一下 DC 才發現是大家都斷掉，全部檢查完以後就等隔天。

## 第一天

比賽第一天剛開賽的時候發現連 scoreboard 會 403 但是 live 組連的上，因為我前一天沒有想到（或者規則沒看到） scoreboard 會鎖 ip，因為我只有設定 10.102.0.0/16，當時意識到在開賽一分鐘內光速把路由調好然後發公告要條 wireguard 的 allowips 以後基本上現場組跟飯店總部都能順利連線開始打了，然後接著幫竹區那邊 sync vpn 設定（他們似乎沒注意到公告）。

等整體穩定後 ShallowFeather 跟 Pwn2ooown 大佬們直接把第二題打穿了，但是 Pwn2ooown 對 attack manager 沒有很熟所以我接下幫他上 attack manager 的任務，而聽 Pwn2ooown 說似乎 scoreboard 跟題目更新 flag 好像會有 delay 所以我設定讓他一分鐘打一次，但是有發現 Caleb 給的 attack manager 腳本如果兩次拿到的 flag 是一樣的第二次會直接不送，因此花了點時間修他。

下午開始我接第五題的解題，首先因為這題他用 python 寫了一個 VM 殼，會先讀圖片然後用圖片生成 op code 而且是中文的 op code 長的雞巴噁心，花了一些時間看懂他的 op code後接下寫 decompiler 的任務，然後就寫到晚上了，所以這題基本上第一天我們只有做黑箱，晚上把 decompiler 優化到可以除了變數名稱完美生成 python code，讓第二天可以方便白箱。

## 第二天

第二天考慮到可能會需要 patch 所以把白箱的工作交給 Ching 然後叫 Vincent55 寫 opcode 轉圖片然後我寫 compiler ，到中午的時候 compiler 基本上完成了，但是 Vincent55 的 opcode 轉圖片還沒寫完就跑掉了，所以我開始看 opcode 轉圖片的部分，後面越看約頭痛，考慮到時間不多了因此忍痛放棄開始看別隊的 patch 看看能否打下那幾個第五題還活著的隊伍順便找個好 patch 偷，結果打到最後才意識到既然一張圖片代表一個 function，484 可以把每一隊的 patch 收集起來然後把 patch 好且沒有藏後門的 function 挑出來組合成新 patch 丟上去，於是最後五輪就這樣做，把 MMM 隊的 patch 拿下來然後拆掉後門丟上去還成功通過 SLA。

## 總結

Infra 整體上除了開賽 scoreboard 的問題還有教一些狀況外連有 file server 然後能登入 LDAP 以及比賽當天還在用模擬用 vpn config 的隊員，以及開賽時 Curious 寫的封包與重送工具直接炸開開始救火以外基本上還算穩定。

但感覺還有很多需要改進的地方，因為其實有很多原本規劃要架但時間關係沒架的東西比如 CI/CD 與 monitor，然後感覺我投資在 compiler 的時間太久了。

然後這幾天官方的 infra 似乎也在燒所以也有出現不知道到底是我的問題、官方的問題、還是隊友自己的問題的情況，比如淺宇就跟我抱怨他有一題要送大圖片但送到一半直接 EOF，但是我從 jumpbox 錄封包就看到是圖片送到一半賽場題目那邊直接送 TCP FIN 的問題。

雖說只拿到世界第十名，但是至少台灣第一沒被隔壁四人組拿走。

附個第五題 [decompile 結果](https://github.com/Jimmy01240397/CTF-writeup/blob/master/hitconctf-2024-final-writeup/ad-fulu/decompiler/allresult.py)

