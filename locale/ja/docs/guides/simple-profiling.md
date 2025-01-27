---
title: Node.js アプリケーションの簡単なプロファイリング
layout: docs.hbs
---

# Node.js アプリケーションの簡単なプロファイリング

Node.js アプリケーションのプロファイリングに使用できるサードパーティ製ツールは多数ありますが、
多くの場合、最も簡単な方法はプロファイラに組み込まれている Node.js を使用することです。
組み込みプロファイラは、プログラムの実行中に定期的にスタックをサンプリングする
[V8 内部のプロファイラ][] を使用します。
これらのサンプルの結果と、jit コンパイルなどの重要な最適化イベントを
一連のティックとして記録します。

```
code-creation,LazyCompile,0,0x2d5000a337a0,396,"bp native array.js:1153:16",0x289f644df68,~
code-creation,LazyCompile,0,0x2d5000a33940,716,"hasOwnProperty native v8natives.js:198:30",0x289f64438d0,~
code-creation,LazyCompile,0,0x2d5000a33c20,284,"ToName native runtime.js:549:16",0x289f643bb28,~
code-creation,Stub,2,0x2d5000a33d40,182,"DoubleToIStub"
code-creation,Stub,2,0x2d5000a33e00,507,"NumberToStringStub"
```

以前は、ティックを解釈するために V8 ソースコードが必要でした。
幸いなことに、Node.js 4.4.0 以降、ソースとは別に V8 を構築することなく
容易にこの情報を利用できるツールが導入されました。
組み込みのプロファイラーがアプリケーションのパフォーマンスに関する
洞察を提供するのに役立つ方法を見てみましょう。

ティックプロファイラーの使い方を説明するために、
簡単な Express アプリケーションを使います。
アプリケーションには、システムに新しいユーザを追加する2つのハンドラがあります。

```javascript
app.get('/newUser', (req, res) => {
  let username = req.query.username || '';
  const password = req.query.password || '';

  username = username.replace(/[!@#$%^&*]/g, '');

  if (!username || !password || users.username) {
    return res.sendStatus(400);
  }

  const salt = crypto.randomBytes(128).toString('base64');
  const hash = crypto.pbkdf2Sync(password, salt, 10000, 512, 'sha512');

  users[username] = { salt, hash };

  res.sendStatus(200);
});
```

ユーザ認証の試行を検証するためのものもあります。

```javascript
app.get('/auth', (req, res) => {
  let username = req.query.username || '';
  const password = req.query.password || '';

  username = username.replace(/[!@#$%^&*]/g, '');

  if (!username || !password || !users[username]) {
    return res.sendStatus(400);
  }

  const { salt, hash } = users[username];
  const encryptHash = crypto.pbkdf2Sync(password, salt, 10000, 512, 'sha512');

  if (crypto.timingSafeEqual(hash, encryptHash)) {
    res.sendStatus(200);
  } else {
    res.sendStatus(401);
  }
});
```

*これらは、Node.js アプリケーションでユーザを認証する推奨のハンドラではなく、
純粋に例を示す目的で使用されていることに注意してください。
自身の暗号認証メカニズムを一般的に設計しようとしてはいけません。
既存の実証済みの認証ソリューションを使用することをお勧めします。*

ここで、アプリケーションをデプロイし、ユーザがリクエストの待ち時間が長いことについて不満を言っているとします。
内蔵のプロファイラーでアプリを簡単に実行できます。

```
NODE_ENV=production node --prof app.js
```

`ab` (ApacheBench) を使用してサーバに負荷をかけます。

```
curl -X GET "http://localhost:8080/newUser?username=matt&password=password"
ab -k -c 20 -n 250 "http://localhost:8080/auth?username=matt&password=password"
```

そして、次の ab 出力を取得します。

```
Concurrency Level:      20
Time taken for tests:   46.932 seconds
Complete requests:      250
Failed requests:        0
Keep-Alive requests:    250
Total transferred:      50250 bytes
HTML transferred:       500 bytes
Requests per second:    5.33 [#/sec] (mean)
Time per request:       3754.556 [ms] (mean)
Time per request:       187.728 [ms] (mean, across all concurrent requests)
Transfer rate:          1.05 [Kbytes/sec] received

...

Percentage of the requests served within a certain time (ms)
  50%   3755
  66%   3804
  75%   3818
  80%   3825
  90%   3845
  95%   3858
  98%   3874
  99%   3875
 100%   4225 (longest request)
```

この出力から、1秒あたり約5つのリクエストを処理することに成功しているだけで、
平均リクエストはラウンドトリップで4秒弱かかります。
実際の例では、ユーザのリクエストに代わって
多くの機能で多くの作業を行っている可能性があります。
しかし簡単な例でさえ、正規表現のコンパイル、ランダムソルトの生成、ユーザパスワードからの一意のハッシュの生成、
または Express フレームワーク自体の内部では、時間が無駄になる可能性があります。

`--prof` オプションを使用してアプリケーションを実行したため、
ティックファイルはローカルで実行しているアプリケーションと同じディレクトリに生成されました。
形式は `isolate-0xnnnnnnnnnnnn-v8.log` (`n` は数字) です。

このファイルを理解するには、Node.js バイナリにバンドルされている tick プロセッサを使用する必要があります。
プロセッサを実行するには `--prof-process` フラグを使用します。

```
node --prof-process isolate-0xnnnnnnnnnnnn-v8.log > processed.txt
```

お気に入りのテキストエディタで processed.txt を開くと、情報が何種類か表示されます。
ファイルはセクションに分割されており、セクションは言語ごとに分割されています。
まず、要約セクションを見ると、このようになっています：

```
 [Summary]:
   ticks  total  nonlib   name
     79    0.2%    0.2%  JavaScript
  36703   97.2%   99.2%  C++
      7    0.0%    0.0%  GC
    767    2.0%          Shared libraries
    215    0.6%          Unaccounted
```

これは、収集されたすべてのサンプルの 97％ が C++ コードで発生しており、
処理された出力の他のセクションを見るときは
(JavaScript ではなく) C++ で行われている作業に最も注意する必要があることを示しています。
これを念頭に置いて、次にどの C++ 関数が最も CPU 時間を消費しているかについての情報を含む
[C++] セクションを見てみます。

```
 [C++]:
   ticks  total  nonlib   name
  19557   51.8%   52.9%  node::crypto::PBKDF2(v8::FunctionCallbackInfo<v8::Value> const&)
   4510   11.9%   12.2%  _sha1_block_data_order
   3165    8.4%    8.6%  _malloc_zone_malloc
```

上位 3 つのエントリが、プログラムで使用された CPU 時間の 72.1％ を占めていることがわかります。
この出力から、すぐにユーザのパスワードからのハッシュ生成に対応する PBKDF2 と呼ばれる機能によって
CPU 時間の少なくとも 51.8％ が占められていることが分かります。
ただし、下位2つのエントリがどのように私たちのアプリケーションに組み込まれるのか
(またはそうである場合は例のために別のふりをすることになる)、
すぐには明らかにならないかもしれません。
これらの関数間の関係をよりよく理解するために、
次に各関数の主な呼び出し元に関する情報を提供する [Bottom up (heavy) profile] セクションを見ていきます。
このセクションを調べると、次のことがわかります。

```
   ticks parent  name
  19557   51.8%  node::crypto::PBKDF2(v8::FunctionCallbackInfo<v8::Value> const&)
  19557  100.0%    v8::internal::Builtins::~Builtins()
  19557  100.0%      LazyCompile: ~pbkdf2 crypto.js:557:16

   4510   11.9%  _sha1_block_data_order
   4510  100.0%    LazyCompile: *pbkdf2 crypto.js:557:16
   4510  100.0%      LazyCompile: *exports.pbkdf2Sync crypto.js:552:30

   3165    8.4%  _malloc_zone_malloc
   3161   99.9%    LazyCompile: *pbkdf2 crypto.js:557:16
   3161  100.0%      LazyCompile: *exports.pbkdf2Sync crypto.js:552:30
```

このセクションを解析することは、
上の生のティックカウントよりも少し多くの作業を要します。
上記の各「コールスタック」内で、親列のパーセントは、
上の行の関数が現在の行の関数によって呼び出されたサンプルの割合を示しています。
たとえば、_sha1_block_data_order の上の中央の "call stack" では、
`_sha1_block_data_order` がサンプルの 11.9％ で発生していることがわかります。
これは、上記の raw カウントからわかりました。
ただし、ここでは、Node.js 暗号モジュール内の pbkdf2 関数によって常に呼び出されたこともわかります。
同様に、`_malloc_zone_malloc` はほとんど同じ pbkdf2 関数によって呼び出されています。
したがって、このビューの情報を使用して、次のことがわかります。
`_sha1_block_data_order` と `_malloc_zone_malloc` の呼び出しは pbkdf2 関数に代わって行われたため、
ユーザのパスワードからのハッシュ計算は、上記の 51.8％ だけでなく、
上位 3 つのサンプル関数のすべての CPU 時間も占めています。

この時点で、パスワードベースのハッシュ生成が最適化の対象になるはずです。
幸い、[非同期プログラミングの利点][] を完全に内部化したので、
ユーザのパスワードからハッシュを生成する作業は同期的に行われ、
イベントループが抑制されていることがわかりました。
これにより、ハッシュを計算している間に
他の受信したリクエストを処理することができなくなります。

この問題を解決するには、非同期バージョンの pbkdf2 関数を使用するように
上記のハンドラを少し変更します。

```javascript
app.get('/auth', (req, res) => {
  let username = req.query.username || '';
  const password = req.query.password || '';

  username = username.replace(/[!@#$%^&*]/g, '');

  if (!username || !password || !users[username]) {
    return res.sendStatus(400);
  }

  crypto.pbkdf2(password, users[username].salt, 10000, 512, 'sha512', (err, hash) => {
    if (users[username].hash.toString() === hash.toString()) {
      res.sendStatus(200);
    } else {
      res.sendStatus(401);
    }
  });
});
```

非同期バージョンのアプリを使って新しい上記の ab ベンチマークを実行すると、
次のようになります。

```
Concurrency Level:      20
Time taken for tests:   12.846 seconds
Complete requests:      250
Failed requests:        0
Keep-Alive requests:    250
Total transferred:      50250 bytes
HTML transferred:       500 bytes
Requests per second:    19.46 [#/sec] (mean)
Time per request:       1027.689 [ms] (mean)
Time per request:       51.384 [ms] (mean, across all concurrent requests)
Transfer rate:          3.82 [Kbytes/sec] received

...

Percentage of the requests served within a certain time (ms)
  50%   1018
  66%   1035
  75%   1041
  80%   1043
  90%   1049
  95%   1063
  98%   1070
  99%   1071
 100%   1079 (longest request)
```

やりました！ アプリは現在、毎秒約20リクエストを処理しています。
これは同期ハッシュ生成の場合の約4倍です。
さらに、平均待ち時間は4秒弱から1秒強に減少しています。

うまくいけば、この (明らかに考案された) 例のパフォーマンス調査を通じて、
V8 ティックプロセッサが Node.js アプリケーションのパフォーマンスの理解を深めるのに
役立つ方法を見ることがあるでしょう。

[V8 内部のプロファイラ]: https://v8.dev/docs/profile
[非同期プログラミングの利点]: https://nodesource.com/blog/why-asynchronous
