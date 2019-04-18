---
layout: post
title: ! "[CSAW Quals 2017] BabyCrypt Writeup"
comments: true
excerpt_separator: <!--more-->
tags:
  - MyriaBreak
  - Write-up
  - CTF
  - Crypto
---

얼마전에 서울하이유스호스텔에서 있었던 `HackingCamp 18th`에 참여하였었는데요. 그 때 내부 CTF에서 나왔던 `Crypto`문제(Easy CAESC)가 인상깊어서 비슷한 문제가 없을까 하고 찾던 중에 알게된 문제입니다. `CSAW Quals 2017`에서 나왔던 `BabyCrypt`란 문제인데요 ㅎㅎ
Baby란 이름이 붙었음에도 350점이라는 높은 점수를 들고있기에 한번 풀어보도록 하겠습니다!

<!--more-->

```
baby_crypt The cookie is input + flag AES ECB encrypted with the sha256 of the flag as the key.

nc crypto.chal.csaw.io 1578
```

먼저 문제를 살펴보면 이 baby_crypt가 무엇을 하는지 알려줍니다. cookie를 뱉어내는데, 이것이 `input + flag`의 형태로 되어있고 `AES ECB` 모드로 flag의 sha256해시값을 키로하여 암호화되어나온다는 것을 알 수 있습니다.

여기서 AES의 키 값으로 flag의 sha256해시값을 사용했으니까 "AES의 키값을 알아내야하나?"하시는 분들이 있을 수도 있는데, 설령 키 값을 알아낸다고 해도 단방향 해시값이기 때문에 다시 이를 평문으로 되돌릴수는 없습니다. (물론 키 값을 알아내는 것부터가 안됩니다만 ㅎ)

그럼 어떻게 해야하느냐?
문제에서 암호화해서 주어지는 값! `input + flag`에서 뒤의 flag값을 취하면 되는 것입니다!!
그런데 문제를 풀기전에 이 문제의 서버는 이미 닫힌지 아~~주 오래되었기 때문에, 저 문제를 풀기위해 문제를 직접 구현할 필요가 있습니다.(...)

그래서 직접 구현하였습니다! 핫ㅋ

```python
#!/usr/bin/env python
from Crypto.Cipher import AES
import hashlib
import sys

flag = open("flag", "r").read().strip()

AES_KEY = hashlib.sha256(flag).hexdigest().decode("hex")[:16]
BLOCK_SIZE  = 16

pad = lambda s: s + (BLOCK_SIZE - len(s) % BLOCK_SIZE) * chr(BLOCK_SIZE - len(s) % BLOCK_SIZE)

def GetCookie(input):
	cipher = AES.new(AES_KEY, AES.MODE_ECB)
	input = pad(input + flag)
	print("Your Cookie is: " + cipher.encrypt(input).encode("hex"))

while(True):
	print("Enter you username (no whitespace) : "),
	sys.stdout.flush()
	input = raw_input()
	GetCookie(input)
```

이제 위 소스파일을 이용해 문제를 풀어봅시다!

먼저 대부분의 블록암호 문제에서 블록암호 모드를 사용하여 암호화된 암호문에 대한 공격은 아래와 같이 진행됩니다.

1. 암호모드 특정(Mode Detection: ECB, CBC, CTR 등)
2. 블록사이즈 구하기(Block size)
3. 공격 (1~2에 구한 것을 기반으로 하여)

하지만 이 문제에선 이미 1번에 해당하는 암호 모드를 알려주었기 때문에 저희는 2번부터 시작하면 됩니다.

이제 이 암호모드에서 사용하는 블록사이즈를 구해야합니다.
여기서 블록사이즈란, 블록암호에서는 데이터를 암호화할때 블록단위로 나누어 암호화를 하게게되는데
이 때 블록을 어느크기만큼 나누어 암호화할지 알려주는 것이 바로 블록사이즈입니다.

그러므로 평문을 암호화하기 위해서는 평문의 길이를 `Block_SIZE`로 맞추어줄 필요가 있고, 이를 위해 패딩을 하게 됩니다. AES블록암호화에서는 보통 `PKCS#7` 패딩방식을 이용하는데, 이 문제에서 패딩이 어떻게 이뤄지는냐는 중요하지않으므로 그림 하나로 설명을 대체하겠습니다. 자세한 설명은 [위키피디아](https://en.wikipedia.org/wiki/Padding_(cryptography))에서 보실수 있습니다.

![]({{ site.baseurl }}/images/myria/BabyCrypt-writeup/babycrypt_03.PNG)

자 이제 블록암호에서 암호화를 할 때 평문크기를 블록크기에 맞게 패딩을 한다는 사실을 알았으니, 문제 소스를 실행하여 한글자씩 문자를 보내봅시다.

![]({{ site.baseurl }}/images/myria/BabyCrypt-writeup/babycrypt_04.PNG)

`abcdefghijklmo`를 보냈을 때와 `abcdefghijklmop`를 보냈을 때 받은 암호문의 길이가 갑자기 증가하게 되고 우리는 이를 통해 `Block_SIZE`를 유추할 수 있습니다.
아래 `python`코드를 통해 실제 블록크기가 얼마가 되는지 구할 수 있습니다.
(여기서 encryption_oracle(plain)함수가 평문에 대한 암호문을 받아오는 함수입니다.)

```python
def find_block_size(encryption_oracle):
	pre_cp = encryption_oracle("")
	p = "A"
	while(True):
		cp = encryption_oracle(p)
		size = len(cp)-len(pre_cp)
		if size != 0:
			return size
		p+="A"
```

![]({{ site.baseurl }}/images/myria/BabyCrypt-writeup/babycrypt_05.PNG)

Block_SIZE : 16

자 이제 블록크기를 구했으니, 실제 공격만 하면 됩니다.
하지만 그 전에 `ECB모드`가 어떻게 동작되는지 알아보도록 하겠습니다.

>> ECB Mode
* 가장 단순한 모드로 평문을 블록단위로 나누어 순차적으로 암호화 하는 구조
* 같은 평문 -> 같은 암호문

![]({{ site.baseurl }}/images/myria/BabyCrypt-writeup/babycrypt_01.PNG)

암호화과정은 위와 같습니다. 여기서 보시면 아시겠지만, ECB모드는 `동일 평문이 항상 동일 암호문으로 암호화되는 것을 알 수 있는데`, 대부분이 사람들이 ECB모드를 설명할 때 나오는 펭귄 사진을 가져와보도록하겠습니다.

![]({{ site.baseurl }}/images/myria/BabyCrypt-writeup/babycrypt_02.PNG)

이렇게 `ECB Mode`로 암호화할 경우, 평문이 항상 같은 암호문으로 1:1 대칭되기때문에 그림의 윤곽이 드러나는 것을 알 수 있습니다.
이제 이 `ECB Mode`의 특성을 이용해서 문제를 풀어보겠습니다.

문제에서 주어진 암호화는 아래와 같이 이루어집니다.

![]({{ site.baseurl }}/images/myria/BabyCrypt-writeup/babycrypt_08.PNG)

저희가 `username`을 입력하면 그 값뒤에 FLAG가 붙어 암호화되는 방식입니다.
즉, 최종적으로는 아래와 같은 데이터가 암호화 오라클에 들어가게됩니다.

![]({{ site.baseurl }}/images/myria/BabyCrypt-writeup/babycrypt_06.png)

예를 들어 `FLAG`의 길이가 20이고, `username`으로 "AAAA..."로 10글자가 들어갔다면 블록사이즈인 16에 맞춰줘야하기때문에 `PADDING`이 2개붙어서 data가 들어가게 되고 암호화되어 출력되게됩니다.

![]({{ site.baseurl }}/images/myria/BabyCrypt-writeup/babycrypt_07.png)

여기서 저희가 제어할 수 있는 부분은 `username`부분입니다.
data : 16bytes(1) | 16bytes(2) | 16 bytes(3)

여기서 저희가 `Block_SIZE`인 16보다 작은 15바이트를 "AAAAAAAAAAAAAAA"로 입력하게 된다면 `FLAG`의 마지막 한바이트를 가져올 수 있게 되고, 저희는 (1)블록의 처음 15바이트를 알고있으므로 (1)블록의 16번째 바이트에 대하여 255가지 암호문을 만들어 비교함으로써 마지막 한바이트를 알 수 있습니다.

계획은 이제 아래와 같습니다.

1. 블록크기보다 1바이트 작은 입력블록을 만든다.
>(예: 블록크기가 8인 경우 "AAAAAAA"로 지정).

2. 마지막 바이트를 바꿔가며 마지막 바이트에 대한 블록암호의 사전을 만든다.
>(예: “AAAAAAAA”, “AAAAAAAB”, “AAAAAAAC” 의 암호문 블록 사전)

![]({{ site.baseurl }}/images/myria/BabyCrypt-writeup/babycrypt_09.PNG)

3. 1번에서 만든 입력블록(1바이트작은)을 input_str에 넣어 나온 암호문을 사전에서 찾는다. 이게 `FLAG`의 첫 번째 바이트

![]({{ site.baseurl }}/images/myria/BabyCrypt-writeup/babycrypt_10.PNG)

4. 위 과정을 반복하여 `FLAG`의 바이트를 하나씩 찾는다.

![]({{ site.baseurl }}/images/myria/BabyCrypt-writeup/babycrypt_11.PNG)


```python
#!/usr/bin/env python
from pwn import *
import string

conn = process(["python", "BabyCrypto.py"])

def encryption_oracle(plain):
	conn.recvuntil("Enter you username (no whitespace) : ")
	conn.sendline(plain)
	conn.recvuntil("Your Cookie is: ")
	enc = conn.recvline().strip()
	return enc.decode("hex")

def find_block_size(encryption_oracle):
	pre_cp = encryption_oracle("")
	p = "A"
	while(True):
		cp = encryption_oracle(p)
		size = len(cp)-len(pre_cp)
		if size != 0:
			return size
		p+="A"

def get_next_byte(encryption_oracle, known_suffix, block_size):
	dic = {}
	feed = "\x00"*(block_size-1-(len(known_suffix)%block_size))

	pt = ''
	#for i in range(0,256):
	for i in range(0x20,0x80):
		pt += feed + known_suffix + chr(i)
	ct = encryption_oracle(pt)[:len(pt)]
	for i in range(0x20,0x80):
		idx = (i-0x20)*(len(feed + known_suffix)+1)
		ct_one = ct[idx:idx+len(feed + known_suffix)+1]
		dic[ct_one]=chr(i)

	ct = encryption_oracle(feed)[:len(feed + known_suffix)+1]
	if ct in dic:
		return dic[ct]
	else:
		return ""

BLOCK_SIZE = find_block_size(encryption_oracle)
secret = ""

print("BLOCK_SIZE : %d "  %  BLOCK_SIZE)

while(True):
	one_byte = get_next_byte(encryption_oracle, secret, BLOCK_SIZE)
	if one_byte == "":
		break
	secret += one_byte
	print(secret)
print("result : "+secret)


```

서버가 이미 닫혔으므로, python pwntools를 사용하여 로컬에서 공격하는 코드이다. 위 파이썬 익스코드를 실행하면 로컬환경에 있는 flag를 획득할 수 있다.

![]({{ site.baseurl }}/images/myria/BabyCrypt-writeup/babycrypt_12.PNG)

### *후기

한번 풀었던 유형의 문제였지만, Writeup을 써보고, 다시 한번 정리해보고 싶어서 작성해보았다. 문제는 AES-ECB모드에 대한 이해와 BLOCK_SIZE로 암호화되는 블록암호의 특성을 대한 지식을 필요로 하기때문에 블록암호에 대해 공부하고 있다면 풀어보면 좋은 문제라고 생각한다.
