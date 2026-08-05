# Happy Birthday
[Happy Birthday - AlpacaHack](https://alpacahack.com/daily/challenges/happy-birthday)

```python:prob.py
from hashlib import sha256
from os import getenv

FLAG = getenv("FLAG", "Alpaca{REDACTED}")


def H(m):
    return sha256(m).digest()[:5]


a = bytes.fromhex(input("user hex > "))
b = bytes.fromhex(input("admin hex > "))

ok = (
    a != b
    and a.startswith(b"user=")
    and b.startswith(b"admin=")
    and H(a) == H(b)
)

print(FLAG if ok else "nope")
```

関数`H(m)`はsha256によって得られるバイト列の先頭5バイト(40ビット)を返す.

40 bitの値は $2^{40} \neq 1.1 \times 10^{12}$ 通りある. $2^{20}$通りのハッシュ値を2セット用意し, 両者の衝突期待値は

$$
\dfrac{nm}{2^{40}} = \dfrac{2^{20} \cdot 2^{20}}{2^{40}} = 1
$$

なので, 期待値1回程度の衝突が得られる. よってこれを解くスクリプトを作成し, 衝突するまで実行すれば良い.

```python:solve.py
from hashlib import sha256
import os

# return output the first 5 bytes
def H(x):
    return sha256(x).digest()[:5]

table = {}

while len(table) >= 1 << 20:
    # generate random 8 bytes
    m = b"user=" + os.urandom(8)
    table[H(m)] = m

while True:
    n = b"admin=" + os.urandom(8)
    h = H(n)
        # check if there is the bytes where the first 5 bytes match
    if h in table:
        print(table[h].hex())
        print(n.hex())
        break
```

実行結果例:

```bash
$ python3 solve.py
757365723d31e5135563b42e1d
61646d696e3d5fe7c03c81ce8699

$ python3 solve.py
757365723d4940a69a47180209
61646d696e3d36cc2ce81bff4b55

...
```