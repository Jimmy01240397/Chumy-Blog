---
title: "AIS3 EOF 2021 Writeup"
date: 2022-01-16T00:00:00+08:00
draft: false
github_link: "https://github.com/Jimmy01240397/CTF-writeup/tree/master/ais3-eof-2021-writeup"
author: "Chumy"
tags:
  - CTF
image: /images/CTF.png
description: ""
toc: 
---

## leetcall

### 題目
![image](https://user-images.githubusercontent.com/57281249/149655714-e6376ea0-d898-4f29-9de4-d3036977f841.png)

### 限制
![image](https://user-images.githubusercontent.com/57281249/149655738-eba68a84-87fb-4f74-9869-4241f4084258.png)

### payload
``` python
# Problem 1: Hello
print(getattr("Hello, {:}!", 'format')(getattr('!\nHello, ', 'join')(getattr(getattr(open(0),'read')(),'splitlines')())))

# Problem 2: Fibonacci
print(getattr('\n','join')(map(str,map(round,map(getattr(0.4472135954999579,'__mul__'),map(getattr(1.618033988749895,'__pow__'),map(int,getattr(open(0),'readlines')())))))))

# Problem 3: FizzBuzz
getattr('','format')(setattr(__builtins__,'a',range(1,10001)),setattr(__builtins__,'b',list(map(bool,map(getattr(3,'__rmod__'),a)))),setattr(__builtins__,'c',list(map(bool,map(getattr(5,'__rmod__'),a)))),print(getattr('\n','join')(map(getattr(str,'__add__'),map(getattr(str,'__add__'),map(getattr(str,'__mul__'),map(str,a),map(getattr(bool,'__and__'),b,c)),map(getattr('Fizz','__mul__'),map(getattr(0,'__eq__'),b))),map(getattr('Buzz','__mul__'),map(getattr(0,'__eq__'),c))))))
```

![image](https://user-images.githubusercontent.com/57281249/149656065-11129a7b-6f05-44c2-b292-965b80a5de0b.png)

### flag
FLAG{actually_you_can_also_solve_those_leetcode_challenges_in_this_way:D}

## SSRF Challenge or Not?

### 題目
![image](https://user-images.githubusercontent.com/57281249/149656247-59579593-98b6-4874-b6bf-14d99f59242f.png)

### 網址
https://ssrf.h4ck3r.quest/

### 解題
![image](https://user-images.githubusercontent.com/57281249/149656280-316252b6-c18f-4c9b-8319-541ebd1c23ce.png)
![image](https://user-images.githubusercontent.com/57281249/149656292-74ee449f-133e-4779-b74a-4810ac094811.png)
![image](https://user-images.githubusercontent.com/57281249/149656304-9e10382a-c94d-4af7-9f22-b9c08f7e894f.png)
![image](https://user-images.githubusercontent.com/57281249/149656319-995c05b0-df12-4e72-af3a-3f9a1bff4ec0.png)
![image](https://user-images.githubusercontent.com/57281249/149656323-ce056c33-43cb-49a2-ba7b-4d54fcfa6dbf.png)
![image](https://user-images.githubusercontent.com/57281249/149656358-cf01ee72-ca11-4ae9-9cce-ece7979edab4.png)
![image](https://user-images.githubusercontent.com/57281249/149656376-68d2fddc-2a70-41bb-a1f5-c6825d99d01e.png)
![image](https://user-images.githubusercontent.com/57281249/149656409-8eabc8d6-9889-4969-94c2-3ea239b927a4.png)
![image](https://user-images.githubusercontent.com/57281249/149656426-74cee109-ad40-4327-a4c7-ac389ce672ca.png)
![image](https://user-images.githubusercontent.com/57281249/149656443-7e0201fb-8bf0-44f0-976d-a54b74958bee.png)
![image](https://user-images.githubusercontent.com/57281249/149656461-15492549-47f7-418e-85a0-657c08f664d7.png)

### flag
FLAG{well, maybe not? XD}
