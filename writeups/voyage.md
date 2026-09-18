# ■ voyage
[voyage - AlpacaHack](https://alpacahack.com/daily/challenges/voyage)

## 1. Writeup
アーカイブファイル`initramfs.cpio`をcpioコマンドを用いて展開することでフラグファイル`flag.txt`を得られる. `print_flag.py`内の処理で`flag.txt`のパスを指定して実行することでフラグを取得できる.

```bash
$ cpio -i < initramfs.cpio

$ ls
... flag.txt ...

$ ./print_flag.py
Alpaca{DUMMY}
```
