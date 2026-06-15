# SITE/2
[SITE/2 - AlpacaHack](https://alpacahack.com/daily/challenges/site2)
## Writeup
```Node.js:server.js
import http2 from "node:http2";

const server = http2.createServer();

server.on("stream", (stream) => {
    stream.end("🦙 Welcome to my website 🦙\nFlag: Alpaca{REDACTED}");
});

server.listen(3000, () => { console.log("http://localhost:3000"); });
```

1. Node.jsでは標準の`node:http2`モジュールを使用してHTTP/2サーバを立ち上げられるが, h2c(HTTP/2 Cleartext)での平文通信はPrior Knowledge方式でのみ接続できる. 
2. メジャーブラウザ(Edge, Chromeなど)はh2cに対応しておらず, 検証できない.

→ `curl`で検証.

```bash
$ curl --http2 http://12.345.678.901:12345/
curl: (1) Received HTTP/0.9 when not allowed

$ curl --http2-prior-knowledge http://12.345.678.901:12345/
🦙 Welcome to my website 🦙
Flag: Alpaca{REDACTED}
```
## Reference(s)
[h2c (HTTP/2 平文) で通信してみた 【その1 〜 バックエンド(Go/Node.js/Python)サービス編】 #http2 - Qiita](https://qiita.com/ssc-ksaitou/items/a1afcd4433c5e21654fa)
