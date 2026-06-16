# Xored PNG
[Xored PNG - AlpacaHack](https://alpacahack.com/daily/challenges/xored-png)
## Writeup

```Python:chal.py
import os

key_hex = os.getenv("KEY_HEX", "00112233445566778899aabbccddeeff")
key = bytes.fromhex(key_hex)
assert len(key) == 16

with open("flag.png", "rb") as f_in:
    png = bytearray(f_in.read())

for i in range(len(png)):
    png[i] ^= key[i % len(key)]

with open("flag.png.xored", "wb") as f_out:
    f_out.write(png)

```

注目する点は以下の2点.

```Python
assert len(key) == 16

...

for i in range(len(png)):
    png[i] ^= key[i % len(key)]
```

1. `assert len(key) == 16`から, key長は16であること
2. `key[i % len(key)]`から, 配列`key`の値を1つずつずらしながら暗号化している. 終端のあとは最初に戻る.

>[!NOTE] PNGのファイル構造
> ## PNGファイルシグネチャ
> 先頭8 byteにはPNGファイルシグネチャ(`89 50 4e 47 0d 0a 1a 0a`)が存在する.
> ## 必須チャンク: IHDRチャンク
> IHDRチャンクはPNGファイルシグネチャの直後に必ずある必須チャンク. データサイズ4 byte + チャンク名4 byte + ...で構成されている.
> データサイズは13 byte(`00 00 00 0d`)で固定であり, チャンク名はIHDR(`49 48 44 52`)であるため, `00 00 00 0d 49 48 44 52`で固定.

`flag.png.xored`の先頭16 byte(`b0 72 10 bf dc a2 91 24  4c ea a6 98 d8 69 ee 33`)を正しいPNGの先頭16 byte(`89 50 4e 47 0d 0a 1a 0a 00 00 00 0d 49 48 44 52`)に直すkeyを求めれば良い.

```bash: head of flag.png.xored
$ hexdump -C flag.png.xored | head -n 1
00000000  b0 72 10 bf dc a2 91 24  4c ea a6 98 d8 69 ee 33  |.r.....$L....i.3|
```

```Python:solve.py
key = []
xored = [0xb0, 0x72, 0x10, 0xbf, 0xdc, 0xa2, 0x91, 0x24, 0x4c, 0xea, 0xa6, 0x98, 0xd8, 0x69, 0xee, 0x33]
correct = [0x89, 0x50, 0x4e, 0x47, 0x0d, 0x0a, 0x1a, 0x0a, 0x00, 0x00, 0x00, 0x0d, 0x49, 0x48, 0x44, 0x52]
for i in range(len(xored)):
        for k in range(0x00, 0xff + 1):
                if arr[i] ^ k == correct[i]:
                        key.append(k)
                        break
```

得られたkeyをもとに`flag.png.xored`ファイルをすべて復号すれば良い. 以下はソルバ例.

```Python:solve.py
#!/usr/bin/python3

key = []

xored = [0xb0, 0x72, 0x10, 0xbf, 0xdc, 0xa2, 0x91, 0x24, 0x4c, 0xea, 0xa6, 0x98, 0xd8, 0x69, 0xee, 0x33]
correct = [0x89, 0x50, 0x4e, 0x47, 0x0d, 0x0a, 0x1a, 0x0a, 0x00, 0x00, 0x00, 0x0d, 0x49, 0x48, 0x44, 0x52]
for i in range(len(xored)):
        for k in range(0x00, 0xff + 1):
                if arr[i] ^ k == correct[i]:
                        key.append(k)
                        break

with open("flag.png.xored", "rb") as f_in:
        png = bytearray(f_in.read())

for i in range(len(png)):
        png[i] ^= key[i % len(key)]

with open("flag.png", "wb") as f_out:
        f_out.write(png)
```

得られた`flag.png`をpngビューアで開くことでフラグを取得できる. 🦙
