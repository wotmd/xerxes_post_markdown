---
layout: post
title: "[BSidesSF 2019] mixxer"
comments: true
excerpt_separator: <!--more-->
tags:
  - Write-up
  - CTF
  - Web
  - Crypto
  - MyriaBreak
---

몇 일전에 있었던 `BSidesSF CTF 2019`에 나왔던 문제를 풀어보겠다. `mixxer`라는 문제로 `Web`과 `Crypto`분야의 문제이다.

<!--more-->

![]({{ site.baseurl }}/images/myria/mixxer-writeup/mixxer_01.PNG)

```
Log in as administrator!

(Check out the user cookie)

Location - https://mixer-f3834380.challenges.bsidessf.net/
```

주어진 `url`을 통해 웹사이트에 들어가게 되면 로그인을 할 수 있는 페이지가 나온다. `권한을 높여라!`라고 크게 적혀있고, 로그인할 수 있는 폼이 있다.

![]({{ site.baseurl }}/images/myria/mixxer-writeup/mixxer_02.PNG)

일단 활성화되어있는 칸이 2개 있으므로, `admin`을 입력하면 아래와 같은 내용이 나온다.

![]({{ site.baseurl }}/images/myria/mixxer-writeup/mixxer_03.PNG)

`is_admin`의 값이 1로 설정되어야하는 것 같다. 하지만 `is_admin`은 칸은 비활성화되어있어 값을 수정할 수 없다. 그래서 제일 먼저 생각나는 크롬 개발자 도구를 이용해보았다.

![]({{ site.baseurl }}/images/myria/mixxer-writeup/mixxer_04.PNG)

값을 1로 바꾸는데 성공하였다. 그러나 저 상태로 아무리 로그인을 시도하여도 아래 메세지는 변함이 없었다...

```
Welcome back, admin admin!

It looks like you aren't admin, though!
Better work on that! Remember, is_admin must bet set to 1 (integer)!
And you can safely ignore the rack.session cookie. Like actually. But that other cookie, however....
```

웹페이지에 걸려있는 `Note`와 로그인 시도시 나오는 문구를 살펴보면 `rack.session` 쿠키는 무시하고, 다른 쿠키값이 문제를 풀기 위한 키포인트일 것 같다.

그래서 페이지의 쿠키를 살펴본 결과 `user`라고하는 수상한 쿠키값을 발견할 수 있었다.

![]({{ site.baseurl }}/images/myria/mixxer-writeup/mixxer_05.PNG)

그러나 아직 이 값이 무엇인지 모르겠다...
그래서 `Burp suite`를 사용하여 값이 어떻게 넘어가는지 살펴보았다.

![]({{ site.baseurl }}/images/myria/mixxer-writeup/mixxer_06.PNG)

`!` `?`
`is_admin`의 값은 넘어가지않고, `action`, `first_name`, `last_name`의 값만 파라미터로 넘어가는것을 알 수 있었다.
왠지 `is_admin`은 아무리 바꾸어도 `user`쿠키나 그 무엇에도 영향이 없더라 ...

![]({{ site.baseurl }}/images/myria/mixxer-writeup/mixxer_07.PNG)

그리고 소스코드를 살펴보면 `is_admin`은 name이 지정되어있지않은것을 알 수 있다.

뭐 아무튼 그렇다면 이제 남은것은 `user`라는 쿠키값이다.
`first_name`과 `last_name`을 여러번 넣어보면 이 `user`라는 쿠키값이 어떻게 나오는지 유추할 수 있다.

![]({{ site.baseurl }}/images/myria/mixxer-writeup/mixxer_08.PNG)

![]({{ site.baseurl }}/images/myria/mixxer-writeup/mixxer_09.PNG)

`Fisrt name`  : aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa

`Last name`   : b

`user` Cookie : f75f9acf55c0f1efbfedd5509e2cb55fbd3fc0da723d226f5d2dd82478531b24bd3fc0da723d226f5d2dd82478531b245c36e6b0b2e6ef806cad8c1dce32c2f4f72de03131106d5a3f8384d2aadf9d2c

`Fisrt name`  : aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa

`Last name`   : bb

`user` Cookie : f75f9acf55c0f1efbfedd5509e2cb55fbd3fc0da723d226f5d2dd82478531b24bd3fc0da723d226f5d2dd82478531b245c36e6b0b2e6ef806cad8c1dce32c2f4543f1ee77054c119fdfa2343152015ece379f6cd6b130380dd363f9d48a409ea

위 입력을 인자로 주었을 때, 두 쿠키값이 비슷함을 볼 수 있다. 즉, 이 쿠키값은 적어도 해시값이 아닌 일정한 암호화과정이 있다는 것이다. 또한 `Last name`의 길이가 1늘어남에 따라 `user`의 길이는 32만큼 증가하였다.

즉, 블록크기만큼 끓어 암호화하는 `블록암호`일 가능성이 생겼다. 그러므로 `user` Cookie를 32문자씩 끓어비교하면 아래와 같다.
```
f75f9acf55c0f1efbfedd5509e2cb55f
bd3fc0da723d226f5d2dd82478531b24
bd3fc0da723d226f5d2dd82478531b24
5c36e6b0b2e6ef806cad8c1dce32c2f4
f72de03131106d5a3f8384d2aadf9d2c
```

2번째 블록과 3번째 블록이 같음을 알 수 있다.
이는 아마 aaaaaaaaaaaaaaaa가 암호화된 결과일 것이다.

자. 그렇다면 이제 위와 같은 결과들을 통해 조심스럽게 이 `user`라는 쿠키는 `AES-ECB` mode를 통해 암호화되었다고 유추해볼 수 있다.

그렇다면 이제 할 일은 간단한데, 이전에 필자가 올린 글 중 [CSAW Quals 2017 BabyCrypt Writeup](https://go-madhat.github.io/BabyCrypt-writeup/)에서 사용했던던 `Byte_at_a_time_ECB_decryption`기법을 이용해 공격해보는 것이다.

```python
import requests
import string

alpha = string.ascii_letters+string.digits

def encryption_oracle(plain):
	print(plain)
	url = "https://mixer-f3834380.challenges.bsidessf.net/"

	session = requests.Session()

	parameter = "?action=login&first_name="+plain+"&last_name="
	new_url = url + parameter
	cookies = {'rack.session': 'BAh7B0kiD3Nlc3Npb25faWQGOgZFVEkiRThmMTYzMzAwM2Q5NjgyNmUwN2Rh%0AOWU5MzY2MzFkNzBjMmI0OWY2ZjYxMzRkYTIyNzhlY2NlNWU2NmI5ODZlZmIG%0AOwBGSSIMYWVzX2tleQY7AEYiJVzOrjIKbvpJ9eq5eel4KQ4hCry4b4wQeVGT%0AZzmWrYHk%0A--97592bb99a0aea52091f7361fa4238deac2f4df3'}
	response = session.get(new_url, cookies=cookies)
	user = session.cookies.get_dict()['user']

	return user.decode("hex")



def find_block_size(encryption_oracle):
	pre_cp = encryption_oracle("")
	p = "A"
	while(True):
		cp = encryption_oracle(p)
		size = len(cp)-len(pre_cp)
		if size != 0:
			return size
		p+="A"

def get_next_byte(encryption_oracle, known_suffix, block_size, prefix_size):
	dic = {}
	feed = "A"*(block_size-(prefix_size%block_size))
	feed += "A"*(block_size-1-(len(known_suffix)%block_size))

	for i in range(0x00,0x7F):
		pt = feed + known_suffix + "%"+hex(i)[2:].rjust(2, "0")
		ct = encryption_oracle(pt)[:len(pt)+prefix_size]
		dic[ct]=chr(i)
	ct = encryption_oracle(feed)[:len(feed + known_suffix)+1+prefix_size]

	if ct in dic:
		return dic[ct]
	else:
		return ""

BLOCK_SIZE = 16
PREFIX_SIZE = 15
print("BLOCK_SIZE  : %d" % BLOCK_SIZE)
print("PREFIX_SIZE : %d" % PREFIX_SIZE)
secret = ""

while(True):
	one_byte = get_next_byte(encryption_oracle, secret, BLOCK_SIZE, PREFIX_SIZE)
	if one_byte == "":
		break
	secret += one_byte
	print(secret)
print("result : "+secret)

```

이제 위 코드를 돌리면 `user`라는 쿠키값의 원래 값을 알아낼 수 있을 것으로 예상되었으나... 실패하였다;;

![]({{ site.baseurl }}/images/myria/mixxer-writeup/mixxer_10.PNG)

대신 다른 재미있는 결과를 얻을 수 있었는데, `\x80`인 아스키범위를 넘어가는 값이 들어갔을 경우이다.

![]({{ site.baseurl }}/images/myria/mixxer-writeup/mixxer_11.PNG)

JSON::GeneratorError를 볼 수 있는데, 파라미터가  json 형식으로 전달되어 `user`값으로 암호화되는 것을 알 수 있다.

그렇다면 `user` 쿠키값의 뒷 부분만 조금 변경하면 `is_admin`값에 영향을 줄 수 있을것이라 생각되어 조금 변경해보았다.

![]({{ site.baseurl }}/images/myria/mixxer-writeup/mixxer_12.PNG)

와우.. 새로운 오류메세지를 발견함과 동시에 암호화되기전의 `user`값을 유추할 수 있다.

#### {"first_name":"admin","last_name":"bb","is_admin":0}

그렇다면 이제 간단해진다. 우리는 `First_name`과 `Last_name`을 마음대로 쓸 수 있으므로 원하는 평문값을 `AES-ECB`로 암호화하여 바꿔쓰기할 수 있다.
`AES-ECB`의 블록크기는 16bytes이므로 아래와 같이 payload를 구성하여 암호화된 `user`쿠키값에서 2번째 블록의 내용을 5번째 블록에 바꿔넣어준다면, `is_amdin`값은 1로 설정될 것이다.

`Fisrt name`  : X1.0000000000000}
`Last name`   : XXXX
`user` Cookie : 97333dd079886bf10452d25f119e24ec316eefd0b1d1734f116488a927fca3f7ccad1e8a1ed41ef310a377abe5c651d903c772d4cd5279ec078ead4300c3f294006c43bbbb599339783cac770c7371b7

Plain(`json`) : `{"first_name":"X` `1.0000000000000}` `","last_name":"X` `XXX","is_admin":` `0}`

Cipher(`user` Cookie) :
`97333dd079886bf10452d25f119e24ec`
`316eefd0b1d1734f116488a927fca3f7`
`ccad1e8a1ed41ef310a377abe5c651d9`
`03c772d4cd5279ec078ead4300c3f294`
`006c43bbbb599339783cac770c7371b7`

이제 쿠키값의 5번째 블록을 2번째 블록과 같은 값으로 바꿔주면 아래와 같이 될 것이다.

Plain(`json`) : `{"first_name":"X` `1.0000000000000}` `","last_name":"X` `XXX","is_admin":` `1}`

Cipher(`user` Cookie) :
`97333dd079886bf10452d25f119e24ec`
`316eefd0b1d1734f116488a927fca3f7`
`ccad1e8a1ed41ef310a377abe5c651d9`
`03c772d4cd5279ec078ead4300c3f294`
`316eefd0b1d1734f116488a927fca3f7`

`user` Cookie : 97333dd079886bf10452d25f119e24ec316eefd0b1d1734f116488a927fca3f7ccad1e8a1ed41ef310a377abe5c651d903c772d4cd5279ec078ead4300c3f294316eefd0b1d1734f116488a927fca3f7

이제 웹페이지에 `user`쿠키값을 위의 변조된 쿠키값으로 바꾸고 새로고침을 누르면 `is_admin`의 값이 1로 되어 `flag`를 얻을 수 있다.

![]({{ site.baseurl }}/images/myria/mixxer-writeup/mixxer_13.PNG)


### *후기

왜 처음에 시도한 `Byte_at_a_time_ECB_decryption`이 성공하지 못했는지 생각해보니 `"`라는 값을 넣게되면 `\"`로 자동으로 바뀌기때문에... 성공할 수 없었던 것이였다. 덕분에 아쉽게 대회중에는 풀지 못했지만, 그래도 `Web`과 `Crypto`를 같이 붙여놓은 문제를 풀어볼 수 있는 좋은 기회였던 것 같다.
