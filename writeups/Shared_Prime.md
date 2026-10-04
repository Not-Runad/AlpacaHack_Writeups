# ■ Shared Prime
[Shared Prime - AlpacaHack](https://alpacahack.com/daily/challenges/shared-prime)
# 1. Writeup
## 1.0. `chall.py`

`chall.py`:
```python
 1	import os
 2	
 3	from Crypto.Util.number import bytes_to_long, getPrime
 4	
 5	
 6	FLAG = os.environ.get("FLAG", "Alpaca{DUMMY}").encode()
 7	e = 65537
 8	
 9	p = getPrime(1024)
10	q1 = getPrime(1024)
11	q2 = getPrime(1024)
12	
13	n1 = p * q1
14	n2 = p * q2
15	m = bytes_to_long(FLAG)
16	assert m < min(n1, n2)
17	
18	c1 = pow(m, e, n1)
19	c2 = pow(m, e, n2)
20	
21	print(f"{n1 = }")
22	print(f"{n2 = }")
23	print(f"{c1 = }")
24	print(f"{c2 = }")
```

## 1.1. 素数$p$
$n_1 = pq_1$, $n_2 = pq_2$ より,

$$\text{gcd}(n_1,n_2) = \text{gcd}(pq_1,pq_2) = p$$

が成り立つ. なぜなら,

$$\text{gcd}(pq_1,pq_2) = p \cdot \text{gcd}(q_1,q_2)$$

ここで, $q_1$ と $q_2$ は互いに素な素数( $\because n_1\neq n_2$ )より $\text{gcd}(q_1,q_2) = 1$. したがって

$$p \cdot \text{gcd}(q_1,q_2) = p \cdot 1=p$$

## 1.2. $q_k$
$p$ がわかったので, $q_k$ がわかる.

$$p \cdot q_k = n_k \iff q_k = n_k \div p$$

## 1.3. RSAに基づいて復号
$$\phi(n_k) = (p - 1)(q_k - 1)$$

より秘密鍵 $d_k$ は

$$d_k = e^{-1}\mod\phi(n_k)$$

したがって平文 $m$ は

$$m=c_{k}^{d_{k}}\mod n_k$$

で求まる.

## 1.4. 回答
以上の考察をスクリプトに落とし込む.

`solve.py`:
```python
from math import gcd
from Crypto.Util.number import long_to_bytes

e = 65537
with open('./output.txt', 'r') as f:
	declare = f.read()
	exec(declare)

p = gcd(n1, n2)
print(f'[*] {p = }')

q1 = n1 // p
q2 = n2 // p
print(f'[*] {q1 = }')
print(f'[*] {q2 = }')

phi1 = (p - 1) * (q1 - 1)
phi2 = (p - 1) * (q2 - 1)

d1 = pow(e, -1, phi1)
d2 = pow(e, -1, phi2)

flag1 = long_to_bytes(pow(c1, d1, n1)).decode()
flag2 = long_to_bytes(pow(c2, d2, n2)).decode()
assert flag1 == flag2

print(f'[*] FLAG: {flag1}')
```

実行例:
```bash
$ python ./solve.py
[*] p = ...
[*] q1 = ...
[*] q2 = ...
[*] FLAG: Alpaca{DUMMY}
```
