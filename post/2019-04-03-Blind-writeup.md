---
layout: post
title: "[VolgaCTF 2019] Blind"
comments: true
excerpt_separator: <!--more-->
tags:
  - Write-up
  - CTF
  - Crypto
  - MyriaBreak
---

이번 `VolgaCTF 2019 Qual`에서 `Crypto`로 나온 `Blind`라는 문제를 풀어보겠습니다.

<!--more-->

```
Blind
Pull the flag...if you can.

nc blind.q.2019.volgactf.ru 7070
```

문제 설명은 위와 같고, `server.py`라는 파이썬 스크립트가 하나 주어집니다.
주어진 파이썬 스크립트는 아래와 같습니다.

```python
#!/usr/bin/env python
from __future__ import print_function
import os
import sys
import shlex
import subprocess

from Crypto.PublicKey import RSA
from Crypto.Util.number import long_to_bytes, bytes_to_long

privkey = RSA.generate(1024)
pubkey = privkey.publickey()


"""
    Utils
"""


def run_cmd(cmd):
    try:
        args = shlex.split(cmd)
        return subprocess.check_output(args)
    except Exception as ex:
        return str(ex)


"""
    Signature
"""

class RSA:
    def __init__(self, e, d, n):
        self.e = e
        self.d = d
        self.n = n

    def sign(self, message):
        message = int(message.encode('hex'), 16)
        return pow(message, self.d, self.n)

    def verify(self, message, signature):
        message = int(message.encode('hex'), 16)
        verify = pow(signature, self.e, self.n)
        return message == verify


"""
	Keys
"""


n = privkey.n
d = privkey.d
e = 65537
print("n : "+str(n))
print("d : "+str(d))
print("e : "+str(e))


"""
    Communication utils
"""

def read_message():
    return sys.stdin.readline()


def send_message(message):
    sys.stdout.write('{0}\r\n'.format(message))
    sys.stdout.flush()


def eprint(*args, **kwargs):
    print(*args, file=sys.stderr, **kwargs)


"""
    Main
"""

def check_cmd_signatures(signature):
    cmd1 = 'exit'
    cmd2 = 'leave'
    assert (signature.verify(cmd1, signature.sign(cmd1)))
    assert (signature.verify(cmd2, signature.sign(cmd2)))


class SignatureException(Exception):
    pass


if __name__ == '__main__':
    signature = RSA(e, d, n)
    check_cmd_signatures(signature)
    try:
        while True:
            send_message('Enter your command:')
            message = read_message().strip()
            (sgn, cmd_exp) = message.split(' ', 1)
            eprint('Accepting command {0}'.format(cmd_exp))
            eprint('Accepting command signature: {0}'.format(sgn))

            cmd_l = shlex.split(cmd_exp)
            cmd = cmd_l[0]
            if cmd == 'ls' or cmd == 'dir':
                ret_str = run_cmd(cmd_exp)
                send_message(ret_str)

            elif cmd == 'cd':
                try:
                    sgn = int(sgn)
                    if not signature.verify(cmd_exp, sgn):
                        raise SignatureException('Signature verification check failed')
                    os.chdir(cmd_l[1])
                    send_message('')
                except Exception as ex:
                    send_message(str(ex))

            elif cmd == 'cat':
                try:
                    sgn = int(sgn)
                    if not signature.verify(cmd_exp, sgn):
                        raise SignatureException('Signature verification check failed')
                    if len(cmd_l) == 1:
                        raise Exception('Nothing to cat')
                    ret_str = run_cmd(cmd_exp)
                    send_message(ret_str)
                except Exception as ex:
                    send_message(str(ex))

            elif cmd == 'sign':
                try:
                    send_message('Enter your command to sign:')
                    message = read_message().strip()
                    message = message.decode('base64')
                    cmd_l = shlex.split(message)
                    sign_cmd = cmd_l[0]
                    if sign_cmd not in ['cat', 'cd']:
                        sgn = signature.sign(sign_cmd)
                        send_message(str(sgn))
                    else:
                        send_message('Invalid command')
                except Exception as ex:
                    send_message(str(ex))

            elif cmd == 'exit' or cmd == 'leave':
                sgn = int(sgn)
                if not signature.verify(cmd_exp, sgn):
                    raise SignatureException('Signature verification check failed')
                break

            else:
                send_message('Unknown command {0}'.format(cmd))
                break

    except SignatureException as ex:
        send_message(str(ex))
        eprint(str(ex))

    except Exception as ex:
        send_message('Something must have gone very, very wrong...')
        eprint(str(ex))

    finally:
        pass
```



위 파이썬 스크립트는 서버에 sign값과 cmd값을 보내면 특정 명령어를 실행할 수 있습니다.
사용가능한 명령어는 `ls, dir, cd, cat`으로 여기서 `ls, dir`은 `sign`값 없이도 실행할 수 있지만 `cd, cat`은 `sign`값이 필요합니다.

```python
def run_cmd(cmd):
    try:
        args = shlex.split(cmd)
        return subprocess.check_output(args)
    except Exception as ex:
        return str(ex)
```
```python
while True:
    send_message('Enter your command:')
    message = read_message().strip()
    (sgn, cmd_exp) = message.split(' ', 1)
    eprint('Accepting command {0}'.format(cmd_exp))
    eprint('Accepting command signature: {0}'.format(sgn))

    cmd_l = shlex.split(cmd_exp)
    cmd = cmd_l[0]
    if cmd == 'ls' or cmd == 'dir':
        ret_str = run_cmd(cmd_exp)
        send_message(ret_str)

    elif cmd == 'cd':
        try:
            sgn = int(sgn)
            if not signature.verify(cmd_exp, sgn):
                raise SignatureException('Signature verification check failed')
            os.chdir(cmd_l[1])
            send_message('')
        except Exception as ex:
            send_message(str(ex))

    elif cmd == 'cat':
        try:
            sgn = int(sgn)
            if not signature.verify(cmd_exp, sgn):
                raise SignatureException('Signature verification check failed')
            if len(cmd_l) == 1:
                raise Exception('Nothing to cat')
            ret_str = run_cmd(cmd_exp)
            send_message(ret_str)
        except Exception as ex:
            send_message(str(ex))
```

그리고 `cat, cd`를 제외한 모든 문자열에 대해서 서버로 부터 `sign`값을 받아낼 수 있습니다.

```python
elif cmd == 'sign':
    try:
        send_message('Enter your command to sign:')
        message = read_message().strip()
        message = message.decode('base64')
        cmd_l = shlex.split(message)
        sign_cmd = cmd_l[0]
        if sign_cmd not in ['cat', 'cd']:
            sgn = signature.sign(sign_cmd)
            send_message(str(sgn))
        else:
            send_message('Invalid command')
    except Exception as ex:
        send_message(str(ex))
```

먼저 `ls` 명령을 통해 파일 목록을 보면 `flag`가 있는 것을 볼 수 있습니다.
그러므로 `cat flag`의 `sign`값을 알아내기만 하면 `flag`를 얻을 수 있습니다.

![]({{ site.baseurl }}/images/myria/Blind-writeup/Blind_01.PNG)

공격법으로는 RSA 암호의 특징을 이해하고, mod 연산의 특성을 알면 쉽게 생각해낼 수 있는 방법이 있습니다. 이에 대한 증명은 위키피디아 등에 찾아보면 아주 자세히 증명해놓았기 때문에 여기서 설명하진 않겠습니다.

1. 먼저 서명할 메세지(m / "cat flag")를 정수로 변환하여 약수를 구합니다.
  m = 2 * 3 * ....
2. 구한 약수 중 하나(r)를 임의로 선택합니다.
  r = 2
3. m/r을 서명합니다.
  S1 = (m/r)^d mod N
4. r을 서명합니다.
  S2 = (r)^d mod N
5. S1과 S2를 곱합니다.
  S1 * S2 = (r)^d mod N  *  (m/r)^d mod N = (m)^d mod N = S'
6. S'를 서명으로 하여 m을 전송합니다.
  S'^e mod N = m^ed mod N = m

위와 같이 되어 `sign` 필터링을 우회하여 `cat flag`를 서명할 수 있습니다.
위를 바탕으로 `exploit`을 짜면 아래와 같습니다.

```python
from pwn import *
from base64 import b64encode
import shlex

conn = remote("blind.q.2019.volgactf.ru", 7070)

conn.recvuntil("Enter your command:")

# sign1
payload = "1 sign"
conn.sendline(payload)

conn.recvuntil("Enter your command to sign:")
m = int("cat flag".encode('hex'), 16)
m_1 = m/408479
m_1 = ("0"+(hex(m_1)[2:])).decode("hex")

payload = b64encode(m_1)
conn.sendline(payload)
conn.recvline()

sign1 = int(conn.recvline().strip())
log.info("sign1 : " + str(sign1))

# sign2
conn.recvuntil("Enter your command:")
payload = "1 sign"
conn.sendline(payload)

conn.recvuntil("Enter your command to sign:")
payload = b64encode(p32(408479)[::-1][1:]) # 408479
conn.sendline(payload)
conn.recvline()

sign2 = int(conn.recvline().strip())
log.info("sign2 : " + str(sign2))

## mix!
sign = sign1*sign2
log.info("sign : " + str(sign))

conn.recvuntil("Enter your command:")
payload = str(sign) + " "
payload += "cat flag"
conn.sendline(payload)

conn.interactive()
```

![]({{ site.baseurl }}/images/myria/Blind-writeup/Blind_02.PNG)


대회가 끝나고 나서 알았는데, `Blind RSA signatures Attack`이라는게 있었습니다.
문제명도 `Blind`인 것을 보니... 제가 한 공격이 아니라 이 공격이 원래 의도한 문제풀이였나봅니다. `Blind RSA attack`도 간단해서 한번 정리해봅니다.

1. 먼저 임의의 수 r을 선택합니다. (이때 r은 n과 서로수),
  gcd(r, n)==1
2. 메세지(m)을 서명한 r과 곱합니다. 그리고 r은 r^-1를 구합니다.
  m' ≡ m*r^e (mod n),  r^{-1} (mod n)
3.  m'를 서명합니다.
  s' ≡ (m')^d (mod n)
4. s'에 r^-1를 곱하게 되면 m^d mode N을 구할 수 있습니다.
  s ≡ s'r' ≡ m^d (mod n)



관련 사이트 :
위키피디 https://en.wikipedia.org/wiki/Blind_signature#Blind_RSA_signatures
Blinding Attack on RSA Digital Signatures https://masterpessimistaa.wordpress.com/2017/07/10/blinding-attack-on-rsa-digital-signatures/

`Blind RSA attack`을 이용한 `exploit`입니다.

```python
from pwn import *
import gmpy
from gmpy2 import gcd

n = 26507591511689883990023896389022361811173033984051016489514421457013639621509962613332324662222154683066173937658495362448733162728817642341239457485221865493926211958117034923747221236176204216845182311004742474549095130306550623190917480615151093941494688906907516349433681015204941620716162038586590895058816430264415335805881575305773073358135217732591500750773744464142282514963376379623449776844046465746330691788777566563856886778143019387464133144867446731438967247646981498812182658347753229511846953659235528803754112114516623201792727787856347729085966824435377279429992530935232902223909659507613583396967
e = 65537

m = int('cat flag'.encode('hex'), 16)
r = 2
"""
while True:
	if gcd(r,n)!=1:
		r+=1
		continue
	m1 = (m*r**e)%n
	m1 = hex(m1)[2:-1] # cut leading '0x'
	if (len(m1)%2 == 1): m1 = '0' + m1 # adjust padding
	m1 = m1.decode('hex')
	print('r = ' + str(r))
	try:
		res = shlex.split(m1)[0]
	except:
		r+=1
		continue
	if (res == m1):
		print('r = ' + str(r))
		break
	r += 1
"""
r = 6631

# connect to ctf server
conn = remote('blind.q.2019.volgactf.ru', 7070)
conn.recvuntil('Enter your command')

# sign modified message m1
conn.sendline('1 sign')
conn.recvuntil('Enter your command to sign:')
conn.sendline(m1)

# receive signature s1
conn.recvline()
resp = conn.recvline()
s1 = int(resp)

# calculate signature s from s1 and r
s = s1*int(gmpy.invert(r,n))%n

# send command 'cat flag' with appropriate signature
conn.sendline(str(s) + ' cat flag')
conn.interactive()
```

![]({{ site.baseurl }}/images/myria/Blind-writeup/Blind_03.PNG)
