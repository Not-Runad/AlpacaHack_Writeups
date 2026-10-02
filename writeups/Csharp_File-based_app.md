# ■ C# File-based app
[C# File-based app - AlpacaHack](https://alpacahack.com/daily/challenges/csharp-file-based-app)

# 1. Writeup
## 1.0. `chal.cs`

`chal.cs`:
```cs
...
14	bool IsCorrect()
15	{
16	    ReadOnlySpan<byte> encoded = "GzMA+MVtbG/XtFYgGNGLSpSiwoAdaiYvxraqBw=="u8;
17	    Span<byte> temp1 = stackalloc byte[256];
18	    Base64.DecodeFromUtf8(encoded, temp1, out _, out int bytesWritten1);
19	
20	    Span<byte> temp2 = stackalloc byte[256];
21	    using var decoder = new BrotliDecoder();
22	    decoder.Decompress(temp1[..bytesWritten1], temp2, out _, out int bytesWritten2);
23	
24	    Span<byte> temp3 = stackalloc byte[Encoding.UTF8.GetMaxByteCount(input.Length)];
25	    int bytesWritten3 = Encoding.UTF8.GetBytes(input, temp3);
26	    return temp2[..bytesWritten2].SequenceEqual(temp3[..bytesWritten3]);
27	}
```

## 1.1. base64

```cs
16	    ReadOnlySpan<byte> encoded = "GzMA+MVtbG/XtFYgGNGLSpSiwoAdaiYvxraqBw=="u8;
17	    Span<byte> temp1 = stackalloc byte[256];
18	    Base64.DecodeFromUtf8(encoded, temp1, out _, out int bytesWritten1);
```

`encoded`(`"GzMA+MVtbG/XtFYgGNGLSpSiwoAdaiYvxraqBw=="`)をbase64でdecodeして `temp1` に保存.

## 1.2. Brotli

```cs
20	    Span<byte> temp2 = stackalloc byte[256];
21	    using var decoder = new BrotliDecoder();
22	    decoder.Decompress(temp1[..bytesWritten1], temp2, out _, out int bytesWritten2);
```

`BrotliDecoder().Decompress()` でBrotli形式の文字列(`temp1`)をdecodeして `temp2` に保存.

## 1.3. 判定

```cs
24	    Span<byte> temp3 = stackalloc byte[Encoding.UTF8.GetMaxByteCount(input.Length)];
25	    int bytesWritten3 = Encoding.UTF8.GetBytes(input, temp3);
26	    return temp2[..bytesWritten2].SequenceEqual(temp3[..bytesWritten3]);
```

`temp2` と `input`(`temp3`) を比較.

## 1.4. 回答

`solve.py`:
```python
import brotli
import base64

raw = 'GzMA+MVtbG/XtFYgGNGLSpSiwoAdaiYvxraqBw=='
base64_decoded = base64.b64decode(raw)
brotli_decode = brotli.decompress(base64_decoded)
print(f'[+] base64_decoded: {base64_decoded}')
print(f'[+] brotli_decode: {brotli_decode}')
```

実行例:
```bash
$ python ./solve.py
[+] base64_decoded: b'\x??\x??\x??\x??\x??\x??\x??\x??\x??\x??\x??'
[+] brotli_decode: b'Alpaca{REDACTED}'

$ ./run-using-docker.sh
flag> Alpaca{REDACTED}
Correct! The flag is Alpaca{REDACTED}
```
