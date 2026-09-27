# ■ typhoon-pythoon
[typhoon-pythoon - AlpacaHack](https://alpacahack.com/daily/challenges/typhoon-pythoon)
## 1. Writeup

python3.10でコンパイルされた `server.pyc` を解析する.

```bash
$ file ./server.pyc
./server.pyc: Byte-compiled Python module for CPython 3.10 (magic: 3439), timestamp-based, .py timestamp: Mon Aug  3 12:55:39 2026 UTC, .py size: 1590 bytes
```

とりあえず実行してみると, 以下のような表示が出現して終了してしまう.

```bash
$ pwd
/home/user/alpacahack/typhoon-pythoon

$ python3.10 ./server.pyc
Hint: Move this file into the eye of the typhoon.
```

`server.pyc` の属性やメソッドを確認してみる.

```python
$ python3.10
>>> import server
>>> dir(server)
['EYE_DATA', 'Path', '__builtins__', '__cached__', '__doc__', '__file__', '__loader__', '__name__', '__package__', '__spec__', 'is_inside_the_typhoon_eye', 'main', 'reverse_the_wind']
```

`server.main` をディスアセンブルしてみる.

```bash
>>> import dis
>>> dis.dis(server.main)
 36           0 LOAD_GLOBAL              0 (is_inside_the_typhoon_eye)
              2 CALL_FUNCTION            0
              4 POP_JUMP_IF_TRUE         9 (to 18)

 37           6 LOAD_GLOBAL              1 (print)
              8 LOAD_CONST               1 ('Hint: Move this file into the eye of the typhoon.')
             10 CALL_FUNCTION            1
             12 POP_TOP

 38          14 LOAD_CONST               0 (None)
             16 RETURN_VALUE

 40     >>   18 LOAD_GLOBAL              2 (input)
             20 LOAD_CONST               2 ('What was inside the eye? > ')
             22 CALL_FUNCTION            1
             24 STORE_FAST               0 (answer)

 42          26 LOAD_FAST                0 (answer)
             28 LOAD_GLOBAL              3 (reverse_the_wind)
             30 LOAD_GLOBAL              4 (EYE_DATA)
             32 CALL_FUNCTION            1
             34 COMPARE_OP               2 (==)
             36 POP_JUMP_IF_FALSE       25 (to 50)

 43          38 LOAD_GLOBAL              1 (print)
             40 LOAD_CONST               3 ('The typhoon is gone. Correct!')
             42 CALL_FUNCTION            1
             44 POP_TOP
             46 LOAD_CONST               0 (None)
             48 RETURN_VALUE

 45     >>   50 LOAD_GLOBAL              1 (print)
             52 LOAD_CONST               4 ('The wind is still too strong...')
             54 CALL_FUNCTION            1
             56 POP_TOP
             58 LOAD_CONST               0 (None)
             60 RETURN_VALUE
```

`main`関数の初めに `is_inside_the_typhoon_eye` が実行されていることがわかる.

次に, この `server.is_inside_the_typhoon_eye` をディスアセンブルしてみる.

```bash
>>> dis.dis(server.is_inside_the_typhoon_eye)
 19           0 LOAD_GLOBAL              0 (Path)
              2 LOAD_GLOBAL              1 (__file__)
              4 CALL_FUNCTION            1
              6 LOAD_METHOD              2 (resolve)
              8 CALL_METHOD              0
             10 LOAD_ATTR                3 (parent)
             12 LOAD_ATTR                4 (name)
             14 STORE_FAST               0 (current_place)

 20          16 LOAD_GLOBAL              5 (bytes)
             18 BUILD_LIST               0
             20 LOAD_CONST               1 ((101, 121, 101))
             22 LIST_EXTEND              1
             24 CALL_FUNCTION            1
             26 LOAD_METHOD              6 (decode)
             28 CALL_METHOD              0
             30 STORE_FAST               1 (required_place)

 21          32 LOAD_FAST                0 (current_place)
             34 LOAD_FAST                1 (required_place)
             36 COMPARE_OP               2 (==)
             38 RETURN_VALUE
```

`(Path)`, `(current_place)`, `(current_place)`, `(required_place)`, `(==)`などが見えることから, ディレクトリパスを取得して比較していると推測できる.

20行目(`20`のブロック)を見ると, `(101, 121, 101)`をデコード(=`'eye'`)して`required_place`に格納している事がわかる.

これを踏まえて, ディレクトリ`eye/` 内で `server.pyc` を実行させてみる.

```bash
$ pwd
/home/user/alpacahack/typhoon-pythoon/eye

$ python3.10 ./server.pyc
What was inside the eye? > AAAAAAAAAAAAA
The wind is still too strong...
```

すると, 出力される内容が変わり, 入力を促される. `server.main` のディスアセンブル結果を見ると,

```python
40     >>   18 LOAD_GLOBAL              2 (input)
             20 LOAD_CONST               2 ('What was inside the eye? > ')
             22 CALL_FUNCTION            1
             24 STORE_FAST               0 (answer)

 42          26 LOAD_FAST                0 (answer)
             28 LOAD_GLOBAL              3 (reverse_the_wind)
             30 LOAD_GLOBAL              4 (EYE_DATA)
             32 CALL_FUNCTION            1
             34 COMPARE_OP               2 (==)
             36 POP_JUMP_IF_FALSE       25 (to 50)

 43          38 LOAD_GLOBAL              1 (print)
             40 LOAD_CONST               3 ('The typhoon is gone. Correct!')
             42 CALL_FUNCTION            1
             44 POP_TOP
             46 LOAD_CONST               0 (None)
             48 RETURN_VALUE

 45     >>   50 LOAD_GLOBAL              1 (print)
             52 LOAD_CONST               4 ('The wind is still too strong...')
```

入力`answer` と `reverse_the_wind(EYE_DATA)` を比較している事がわかる.

`reverse_the_wind(EYE_DATA)` の出力結果を確認してみる.

```python
>>> print(server.reverse_the_wind(server.EYE_DATA))
Alpaca{REDACTED}
```

得られた結果を入力に渡してみると, 正しい事がわかる.

```bash
$ python3.10 ./server.pyc
What was inside the eye? > Alpaca{REDACTED}
The typhoon is gone. Correct!
```