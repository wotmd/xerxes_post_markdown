---
layout: post
title: ! "[Crypto] CTR 모드 공격 Break fixed-nonce CTR"
comments: true
excerpt_separator: <!--more-->
tags:
  - MyriaBreak
  - Crypto
---

시간이 없어 암호 공부를 얼마간 못하다가 다시 시작하였습니다. 이번에는 CTR모드에 대해, 그리고 `고정된 nonce를 사용하는 CTR모드`를 공격하는 방법에 대해 알아보겠습니다.
참고로 공격대상이 되는 데이터는 영어평문문장입니다.

<!--more-->

먼저 CTR모드에 대해 알아보겠습니다.
#### CTR(Counter)
카운터(Counter, CTR) 방식은 `블록 암호`를 `스트림 암호`로 바꿔주는 구조를 가집니다. CTR모드에서는 각 블록마다 현재 블록이 몇 번째인지 값을 얻어, 그 숫자(count)와 nonce를 결합하여 블록 암호의 입력으로 사용합니다. 그렇게 각 암호화를 통해 연속적인 난수를 얻은 다음 암호화하려는 문자열과 XOR하는 방식입니다.

이렇게 되면 CTR모드는 각 블록의 암호화 및 복호화가 이전 블록에 의존하지 않으며, 따라서 병렬적으로 동작하는 것이 가능해집니다. 이것은 암호화 및 복호화의 속도가 타모드에 비해 매우 빠르다는 것을 의미합니다. 이전에 있던 모드들은(CBC, CFB, OFB와 같은) 블록의 암호화 및 복호화가 이전 블록에 의존되게 때문에 각 블록이 독립적으로 동작할 수 없고, 때문에 암호화와 복호화가 순차적으로 이루어져야했습니다.
또한 암호화된 문자열 중 원하는 부분만 골라 복호화하는 것이 불가능하였으나, CTF모드에서는 이것이 가능합니다.

![]({{ site.baseurl }}/images/myria/Break_fixed_nonce_CTR/break_fixednonce_ctr_01.png)
![]({{ site.baseurl }}/images/myria/Break_fixed_nonce_CTR/break_fixednonce_ctr_02.png)
#### * 정리

* 블록마다 1씩 증가하는 counter와 nonce를 결합하여 암호화하여 키 스트림을 만든다.
* 키 스트림과 평문 블록을 XOR한 결과가 암호문 블록이다.
* 블록 암호를 스트림 암호구조로 바꿔준다.
* 암호화와 복호화가 같은 구조다. (XOR암호)
* 병렬처리가 가능하다. (임의의 블록만 암/복호화가 가능하다)
* Error propagation : 각 블록이 독립적이므로, 에러가 다른 블록으로 전파되지않는다.


자. 이제 CTR모드에 대해 알아보았으니, 이 모드에서 고정된 nonce를 사용할 경우 어떻게 공격할 수 있는지에 대해 살펴보겠습니다.

먼저 위에서 살펴보았듯이 CTR모드를 사용하게되면 블록암호를 스트림암호방식으로 사용할 수 있습니다. 즉, 패딩이 불필요해지고, 키스트림을 생성하고 나면 XOR암호화 동일해집니다.

그럼 이제 남은 일은 이 `XOR암호`를 공격하는 것입니다. 고정된 nonce인 경우, 여러개의 데이터를 암호화하게 되면 `같은 counter값`을 가진 구간에 대해선 모두 같은 키스트림을 가지고 XOR암호화되게 됩니다.

![]({{ site.baseurl }}/images/myria/Break_fixed_nonce_CTR/break_fixednonce_ctr_03.png)

위와 같이 여러개의 데이터가 있을 때, 각 데이터의 첫번째 블록은 `고정된 nonce | counter`으로 만들어진 키스트림과 XOR을 통해 암호화되게 됩니다. 이 때 `nonce`가 고정되어있고, `counter`값은 블록의 인덱스를 나타내므로 각 데이터의 블록들은 모두 각각 같은 키스트림을 가지게됩니다.

그렇다면 각 데이터에서 같은 인덱스의 블록들을 모은다면, 같은 키스트림을 가지는 암호문 리스트를 얻을 수 있습니다.

![]({{ site.baseurl }}/images/myria/Break_fixed_nonce_CTR/break_fixednonce_ctr_04.png)

키스트림의 길이는 블록길이와 같으므로, `XOR암호`에서 키 사이즈는 `BLOCKSIZE`입니다.
여기서는 블록암호로 `AES-128`을 사용하였으므로, 키 사이즈는 16입니다.
이제 `Muliple XOR Cipher`을 풀 차례입니다. 그러나 키 길이를 알 고 있으므로 저희는 이것을 `Single XOR Cipher`로 바꿀 수 있습니다. 키길이가 16이면, XOR수행시 같은 바이트값, 즉 `SingleXorByte`가 16byte마다 나타나는 것을 알 수 있습니다. 그러므로 이번에는 바이트단위로 암호문을 나누어 다시 새롭게 배열합니다.

아래는 만약 KEY가 `XOR`인 keySize가 3일 때를 나타내는 예시입니다.

![]({{ site.baseurl }}/images/myria/Break_fixed_nonce_CTR/break_fixednonce_ctr_05.png)

위의 같은 색으로 칠해진 부분은 같은 `단일 KEYBYTE`에 의해 암호화됩니다. 그러므로 위 `Ciphertext`를 아래와 같이 새롭게 정렬해줄 수 있습니다.

![]({{ site.baseurl }}/images/myria/Break_fixed_nonce_CTR/break_fixednonce_ctr_06.png)

이제 새롭게 정렬된 `Ciphertext`를 `Single XOR Cipher`공격으로 풀면 됩니다. 공격은 기본 `BruteForce`로 정확도를 높이기 위해 `AlphabetScoring`방식으로 진행합니다.

![]({{ site.baseurl }}/images/myria/Break_fixed_nonce_CTR/break_fixednonce_ctr_07.png)
![]({{ site.baseurl }}/images/myria/Break_fixed_nonce_CTR/break_fixednonce_ctr_08.png)

이제 각각에 대해 `AlphabetScoring`을 통해 KEYSTEAM을 알아낼 수 있습니다.

![]({{ site.baseurl }}/images/myria/Break_fixed_nonce_CTR/break_fixednonce_ctr_09.png)

그럼 이제 실전으로 들어가보겠습니다.
먼저 `AES-CTR` 암호화에 사용할 데이터를 40개 준비합니다.

```
SSBoYXZlIG1ldCB0aGVtIGF0IGNsb3NlIG9mIGRheQ==
Q29taW5nIHdpdGggdml2aWQgZmFjZXM=
RnJvbSBjb3VudGVyIG9yIGRlc2sgYW1vbmcgZ3JleQ==
RWlnaHRlZW50aC1jZW50dXJ5IGhvdXNlcy4=
SSBoYXZlIHBhc3NlZCB3aXRoIGEgbm9kIG9mIHRoZSBoZWFk
T3IgcG9saXRlIG1lYW5pbmdsZXNzIHdvcmRzLA==
T3IgaGF2ZSBsaW5nZXJlZCBhd2hpbGUgYW5kIHNhaWQ=
UG9saXRlIG1lYW5pbmdsZXNzIHdvcmRzLA==
QW5kIHRob3VnaHQgYmVmb3JlIEkgaGFkIGRvbmU=
T2YgYSBtb2NraW5nIHRhbGUgb3IgYSBnaWJl
VG8gcGxlYXNlIGEgY29tcGFuaW9u
QXJvdW5kIHRoZSBmaXJlIGF0IHRoZSBjbHViLA==
QmVpbmcgY2VydGFpbiB0aGF0IHRoZXkgYW5kIEk=
QnV0IGxpdmVkIHdoZXJlIG1vdGxleSBpcyB3b3JuOg==
QWxsIGNoYW5nZWQsIGNoYW5nZWQgdXR0ZXJseTo=
QSB0ZXJyaWJsZSBiZWF1dHkgaXMgYm9ybi4=
VGhhdCB3b21hbidzIGRheXMgd2VyZSBzcGVudA==
SW4gaWdub3JhbnQgZ29vZCB3aWxsLA==
SGVyIG5pZ2h0cyBpbiBhcmd1bWVudA==
VW50aWwgaGVyIHZvaWNlIGdyZXcgc2hyaWxsLg==
V2hhdCB2b2ljZSBtb3JlIHN3ZWV0IHRoYW4gaGVycw==
V2hlbiB5b3VuZyBhbmQgYmVhdXRpZnVsLA==
U2hlIHJvZGUgdG8gaGFycmllcnM/
VGhpcyBtYW4gaGFkIGtlcHQgYSBzY2hvb2w=
QW5kIHJvZGUgb3VyIHdpbmdlZCBob3JzZS4=
VGhpcyBvdGhlciBoaXMgaGVscGVyIGFuZCBmcmllbmQ=
V2FzIGNvbWluZyBpbnRvIGhpcyBmb3JjZTs=
SGUgbWlnaHQgaGF2ZSB3b24gZmFtZSBpbiB0aGUgZW5kLA==
U28gc2Vuc2l0aXZlIGhpcyBuYXR1cmUgc2VlbWVkLA==
U28gZGFyaW5nIGFuZCBzd2VldCBoaXMgdGhvdWdodC4=
VGhpcyBvdGhlciBtYW4gSSBoYWQgZHJlYW1lZA==
QSBkcnVua2VuLCB2YWluLWdsb3Jpb3VzIGxvdXQu
SGUgaGFkIGRvbmUgbW9zdCBiaXR0ZXIgd3Jvbmc=
VG8gc29tZSB3aG8gYXJlIG5lYXIgbXkgaGVhcnQs
WWV0IEkgbnVtYmVyIGhpbSBpbiB0aGUgc29uZzs=
SGUsIHRvbywgaGFzIHJlc2lnbmVkIGhpcyBwYXJ0
SW4gdGhlIGNhc3VhbCBjb21lZHk7
SGUsIHRvbywgaGFzIGJlZW4gY2hhbmdlZCBpbiBoaXMgdHVybiw=
VHJhbnNmb3JtZWQgdXR0ZXJseTo=
QSB0ZXJyaWJsZSBiZWF1dHkgaXMgYm9ybi4=
```

base64로 인코딩된 평문을 40개 준비하였습니다.
이제 base64로 디코딩하여 평문을 40개 리스트에 넣고, 이를 `AES_CTR`로 암호화시켜 암호문 배열을 만들겠습니다.

```python
plainlist = open("data.txt", "r").readlines()
plainlist = [b64decode(x) for x in plainlist]

cipherlist = [AES_CTR(x, KEY, nonce) for x in plainlist]
```

이제 암호문을 `Coulumn`으로 나눠서 다시 정렬하여, 위에서 설명했던 대로 `AlphabetScoring`을 통해 키를 구합니다.

```python

def get_keyString(ciphers):
	keyString = ""
	for nColumnCipher in ciphers:
		Max_score = 0.0
		key = 0
		for sbyte in range(0,256):
			nomal_string = XOR_singleByte(nColumnCipher, sbyte)
			score = alphabet_score(nomal_string)

			if score > Max_score:
				Max_score = score
				key = sbyte
		keyString += chr(key)
	return keyString

maxLen = max([len(x) for x in cipherlist])
print("maxLen : %d " % maxLen)

nColumnCipher = []
for n in range(0, maxLen):
  nColumn = ""
  for c in cipherlist:
  	if len(c) > n:
  		nColumn += c[n]
  nColumnCipher.append(nColumn)
keyString = get_keyString(nColumnCipher)
```

결과를 한번 확인해보겠습니다.

```python
decrypt_list = [XOR_with_Key(ciph, keyString) for ciph in cipherlist]

	for i, decrpyt in enumerate(decrypt_list):
		print("%d : %s" %(i, decrpyt))
```

![]({{ site.baseurl }}/images/myria/Break_fixed_nonce_CTR/break_fixednonce_ctr_10.png)

대체로 성공적으로 복호화 완료된 것을 볼 수 있습니다. 뒷 부분은 평문이 제대로 복호화가 되지않는 결과가 조금씩 보이는데, 이것은 각 `Coulumn`으로 정렬하는데 있어 데이터가 부족하여 생긴 결과입니다.
이 부분은 어쩔 수 없이 수동으로 조금 씩 맞춰주거나, 아니면 단어를 가지고 `Scoring`을 수행하던지 해야하는 부분입니다.
저는 이 부분을 `Fix_keyString`이란 함수를 만들어 처리하였지만, 여기서는 설명하지않도록 하겠습니다.

이렇게 `CTR`모드의 공격에 대한 설명을 마치겠습니다.
아래는 `CTR`모드의 공격을 테스트하는 전체코드입니다.

### * 전체코드

```python
from pwn import p64
from base64 import b64decode
from Crypto.Cipher import AES
import os

#Letter Distribution - Exterior Memory  http://www.macfreek.nl/memory/Letter_Distribution
freq = {
' ' : 18.28846265,
'E' : 10.26665037,
'T' : 7.51699827,
'A' : 6.53216702,
'O' : 6.15957725,
'N' : 5.71201113,
'I' : 5.66844326,
'S' : 5.31700534,
'R' : 4.98790855,
'H' : 4.97856396,
'L' : 3.31754796,
'D' : 3.28292310,
'U' : 2.27579536,
'C' : 2.23367596,
'M' : 2.02656783,
'F' : 1.98306716,
'W' : 1.70389377,
'G' : 1.62490441,
'P' : 1.50432428,
'Y' : 1.42766662,
'B' : 1.25888074,
'V' : 0.79611644,
'K' : 0.56096272,
'X' : 0.14092016,
'J' : 0.09752181,
'Q' : 0.08367550,
'Z' : 0.05128469,
}

def alphabet_score(stringA):
	score = 0.0
	for c in stringA:
		c=c.upper()
		if c in freq:
			score+=freq[c]
	return score

def XOR_singleByte(str1, sbyte):
	str1 = str1.encode("hex")
	result = ""
	for i in range(0,len(str1),2):
		h1 = int(str1[i:i+2],16)
		h2 = sbyte
		result+= '{:02x}'.format(h1^h2)
	return result.decode("hex")



KEY = os.urandom(16)
nonce = p64(0)

BLOCK_SIZE = 16



def XOR_with_Key(str1, Key):
	str1 = str1.encode("hex")
	result = ""
	for i in range(0,len(str1),2):
		h1 = int(str1[i:i+2],16)
		h2 = ord(Key[(i/2)%len(Key)])
		result+= '{:02x}'.format(h1^h2)
	return result.decode("hex")

def AES_ecb_encrypt(plain, key):
	obj = AES.new(key, AES.MODE_ECB)
	cipher = obj.encrypt(plain)
	return cipher

def AES_CTR(data, key, nonce):
	xor_data = ""
	cnt = 0
	for i in range(0, len(data), BLOCK_SIZE):
		input = nonce + p64(cnt)
		xor_data += AES_ecb_encrypt(input, key)
		cnt+=1
	processing_data = XOR_with_Key(data, xor_data)
	return processing_data


def get_keyString(ciphers):
	keyString = ""
	for nColumnCipher in ciphers:
		Max_score = 0.0
		key = 0
		for sbyte in range(0,256):
			nomal_string = XOR_singleByte(nColumnCipher, sbyte)
			score = alphabet_score(nomal_string)

			if score > Max_score:
				Max_score = score
				key = sbyte
		keyString += chr(key)
	return keyString

def Fix_keyString(keyString, cipher, fix_plain):
	decrypt = XOR_with_Key(cipher, keyString)
	fix = XOR_with_Key(decrypt, fix_plain)
	fix_keyString = XOR_with_Key(keyString, fix)
	return fix_keyString


def main():
	plainlist = open("data.txt", "r").readlines()
	plainlist = [b64decode(x) for x in plainlist]

	cipherlist = [AES_CTR(x, KEY, nonce) for x in plainlist]
	maxLen = max([len(x) for x in cipherlist])
	print("maxLen : %d " % maxLen)

	nColumnCipher = []
	for n in range(0, maxLen):
		nColumn = ""
		for c in cipherlist:
			if len(c) > n:
				nColumn += c[n]
		nColumnCipher.append(nColumn)

	keyString = get_keyString(nColumnCipher)
	keyString = Fix_keyString(keyString, cipherlist[0], "I have met them at close of day")
	keyString = Fix_keyString(keyString, cipherlist[25], "This other his helper and friend")
	decrypt_list = [XOR_with_Key(ciph, keyString) for ciph in cipherlist]

	for i, decrpyt in enumerate(decrypt_list):
		print("%d : %s" %(i, decrpyt))

	# compare to original_plain & decrypt_plain
	for original, decrpyt in zip(plainlist, decrypt_list):
		print("o : " + original)
		print("d : " + decrpyt)

if __name__ == '__main__':
    main()
```
