---
layout: post
title: "[glibc] malloc - fastchunk size check 분석"
comments: true
excerpt_separator: <!--more-->
tags:
  - Write-up
  - pwnable
  - defcon27
  - MyriaBreak
---

[HITCON-Training lab12 sercretgarden](https://xerxes-break.tistory.com/441) 문제를 풀다가 `fastbin chunk size check`루틴에 의문을 가지게 되어서 알아보게 되었다. 

<!--more-->

이 문제 exploit시 아래와 같은 영역을 사용하게 된다. 문제는 아래 chunk의 size부분인데. 사이즈가 0xe168000000000060이다.



그런데 malloc(0x50)로 할당시 아래와 같은 영역이 `fastbin`에 존재한다면 이것을 오류없이 할당할 수 있을까?

```
0x601ffa:    0x1e28000000000000    0xe168000000000060
0x60200a:    0x0000414141414141    0x2390000000000000
```

할당할 수 있다가 정답이다. `tcache`가 trriger되지 않은 상태에서 아래와 같은 영역을 `fastbin chunk`로 사용하여 `double free` 버그에서 할당 시 사용할 수 있다. 그런데 사이즈를 보면 fastbin의 사이즈가 아니니까 사이즈에러아닌가?

```
0x601ffa (size error (0xe168000000000060))
```

그래서 [malloc.c](https://code.woboq.org/userspace/glibc/malloc/malloc.c.html#3602)의 소스코드를 살펴보았다.
여기서 3602번째줄 부터 보면 `fastchunk size check` 루틴이 있다.

 `malloc(): memory corruption (fast)` 오류는 malloc 요청 처리시 fastbin에서 첫 번째 청크를 제거할 때, 첫번재 청크의 크기가 fastbin 청크 범위에 속하지 않으면 발생한다.

https://heap-exploitation.dhavalkapil.com/diving_into_glibc_heap/security_checks.html

```c
  unsigned int idx;                 /* associated bin index */
```

```c
          if (__glibc_likely (victim != NULL))
            {
              size_t victim_idx = fastbin_index (chunksize (victim));
              if (__builtin_expect (victim_idx != idx, 0))
                malloc_printerr ("malloc(): memory corruption (fast)");
              check_remalloced_chunk (av, victim, nb);
```

이 첫번째 victim청크의 fastbin_index를 구하여 해당 fastbin의 index가 맞는지 확인을 하게 되는데, 이 fastbin_index 매크로는 아래와 같다.

```c
((((unsigned int) ((((victim)->mchunk_size) & ~((0x1 | 0x2 | 0x4))))) >> (SIZE_SZ == 8 ? 4 : 3)) - 2)
```

즉, 사이즈가 unsigned int이다. 이 말은 즉  `fastchunk size check`에는 4byte만 사용된다는 것이고  0xe1ffffff00000060같은 값도 check를 통과한다는 것이다.

정말 그런지 확인해보기 위해 `peda`를 사용해 디버깅해보았다.

```c
gdb-peda$ $ set *(0x601ffa+12)=0xe122ffff
gdb-peda$ $ heapinfo
(0x20)     fastbin[0]: 0x0
(0x30)     fastbin[1]: 0x0
(0x40)     fastbin[2]: 0x0
(0x50)     fastbin[3]: 0x0
(0x60)     fastbin[4]: 0x601ffa (size error (0xe122ffff00000060)) --> 0xeee000007ffff7ff (invaild memory)
(0x70)     fastbin[5]: 0x6040d0 --> 0x0
(0x80)     fastbin[6]: 0x0
(0x90)     fastbin[7]: 0x0
(0xa0)     fastbin[8]: 0x0
(0xb0)     fastbin[9]: 0x0
                  top: 0x604290 (size : 0x1fd70) 
       last_remainder: 0x0 (size : 0x0) 
            unsortbin: 0x0
gdb-peda$ $ c
```

위와 같이 0x60에 해당되는 `fastbin`의 첫번째 청크 크기를 0xe122ffff00000060로 바꾸고 할당을 시켜보았다.

```c
gdb-peda$ $ x/4gx 0x601ffa
0x601ffa:    0x1e28000000000000    0xe122ffff00000060
0x60200a:    0x4141414141414141    0x14f000007ffff70a
gdb-peda$ $ 
```

오... 잘되는 것을 알 수 있다.
이걸 몰라서 항상 어렵게 익스를 구성했엇는데,  이러면 `tcache`가 비활성화된 libc에서 `double free bug`를 이용할 때 조금 더 편하게 익스를 짤 수 있을 것같다.