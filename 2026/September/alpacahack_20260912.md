# ■ sin function vs. fsin instruction
[sin function vs. fsin instruction - AlpacaHack](https://alpacahack.com/daily/challenges/sin-function-vs-fsin-instruction)

## 1. Writeup

```c:chal.c
double sine_using_instruction(double theta) {
    double result;
    __asm__(
        "fsin"
        : "=t"(result)
        : "0"(theta)
    );

    return result;
}
```

x86アーキテクチャの `fsin` 命令における入力範囲は$-2^{63}<N<2^{63}$. 範囲外の値を受け取るとC2フラグが1となり, 値 `theta` は変更されない.

```c:chal.c
42     if (fabs(sine_1 - sine_2) > 1337.0) {
43         const char *env_flag = getenv("FLAG");
44         printf("Unbelievable! FLAG: %s\n", env_flag ? env_flag : "Alpaca{DUMMY}");
45     }
46     else {
47         puts("There isn't much difference.");
48     }
```

これを踏まえて, `fsin - sin() > 1337` となればよいので, `fsin` の範囲より大きい値かつ, `isinf(theta) == true` とならないような範囲の値を渡せば良い. (といっても, 型 `double` の `INF` は十分に大きいので, `fsin` の入力範囲のみを意識するので満足できる.)

example:
```bash
$ ./chal
theta> 1e20
sine_1: -0.645251
sine_2: 1e+20
Unbelievable! FLAG: Alpaca{DUMMY}
```

ref(s):
- [FSIN: Sine (x86 Instruction Set Reference)](https://c9x.me/x86/html/file_module_x86_id_114.html)