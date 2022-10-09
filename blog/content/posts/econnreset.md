---
title: "Node.js における ECONNRESET について"
date: 2021-10-17T14:30:00+09:00
draft: false
categories:
- 開発
tags:
- JavaScript
- TypeScript
toc: true
---

# 要約

- `ECONNRESET` は接続相手（とくにサーバー側）から接続を強制的に切断された場合に発生する
- `ECONNRESET` が発生した場合はリトライすることで回復できる可能性がある
  - [axios](https://github.com/axios/axios) を使っている場合は [axios-retry](https://github.com/softonic/axios-retry) でリトライできる

# 前提

以下の環境で確認しました。

- Node.js v14.18.0
- axios v0.22.0
- axios-retry v3.2.0

また、以下に記載する `ECONNRESET` の再現例とリトライ方法については [リポジトリ](https://github.com/Aminevsky/econnreset-reproduce) も参照してください。

# 事象

AWS Lambda 上で Nuxt.js を動かして SSR （サーバーサイドレンダリング）を行っています。各ページでは axios を使って JSON を取得しています。

この Lambda で、ときどき、以下のようなエラーが発生しています。

```
Error: read ECONNRESET
```

# 再現

どのような場合に `ECONNRESET` は発生するのでしょうか。

## 再現例1

Node.js の Issue で `ECONNRESET` の再現に成功したという [コメント](https://github.com/nodejs/node/issues/20256#issuecomment-473170418) を見つけました。これを参考に再現してみます。

このコメントでは Keep-Alive のタイムアウトに着目しています。Keep-Alive は、1回接続したら、それをできるだけ使い回すことで効率化を図る手法です。しかし、追加のリクエストが無いのに繋ぎっぱなしにするわけにもいきません。そこで、一定期間が経つと、サーバー側から接続を切断するようになっています。Node.js ではデフォルトで 5000 ミリ秒（＝5秒）待ってもリクエストが無ければ切断することになっています（[公式ドキュメント](https://nodejs.org/docs/latest-v14.x/api/http.html#http_server_keepalivetimeout)）。

例えば、クライアント側で 5000 ミリ秒おきに 10 回リクエストを送ってみます。

```typescript
const httpAgent = new http.Agent({ keepAlive: true });
const httpsAgent = new https.Agent({ keepAlive: true });

for (let i = 1; i <= 10; i++) {
  await new Promise(async (resolve) => {
    setTimeout(async () => {
      try {
        await axios.get(`http://localhost:8001/${i}`, {
          httpAgent,
          httpsAgent,
        });
        console.log(`Success: ${i}`);
        resolve('Success');
      } catch (err) {
        console.error(`Error: ${i}, ${err}`);
        resolve('Error');
      }
    }, 5000);
  });
}
```

すると、以下のようになります。実行するたびに結果は変わりますが、おおむね、10 回中 4 回ほど、エラーが発生します。 `socket hang up` も混じっていますが、 `ECONNRESET` が発生していることがわかります。

```
Success: 1
Success: 2
Error: 3, Error: read ECONNRESET
Success: 4
Error: 5, Error: read ECONNRESET
Success: 6
Error: 7, Error: read ECONNRESET
Success: 8
Error: 9, Error: socket hang up
Success: 10
```

なお、リクエストを送る間隔を早めてみると、失敗回数は少なくなります。[^1]

| 間隔 | 成功回数 | 失敗回数 |
| ---- | -------- | -------- |
| 4900 | 10       | 0        |
| 4950 | 10       | 0        |
| 4980 | 10       | 0        |
| 4985 | 9        | 1        |
| 4990 | 9        | 1        |
| 4995 | 8        | 2        |
| 5000 | 6        | 4        |

[^1]: もとのコメントでは `maxSockets` を `1` にしていますが、これは結果にさほど影響しませんでした。

以上から、Keep-Alive でタイムアウトする確率が高いほど `ECONNRESET` も発生しやすいことがわかりました。

## 再現例2

サーバー側が許容する最大接続数を上回るリクエストがあった場合にも `ECONNRESET` は発生します。

例えば、サーバー側の [maxConnections](https://nodejs.org/docs/latest-v14.x/api/net.html#net_server_maxconnections) を `1` にしてみます。

```typescript
const server = http.createServer((_, res) => {
  res.writeHead(200, { 'Content-Type': 'application/json' });
  res.end(JSON.stringify({ message: 'Success' }));
});
server.maxConnections = 1;
server.listen(8001);
```

この状態で、クライアントから同時に 2 つリクエストが送ってみます。

```typescript
const httpAgent = new http.Agent({
  keepAlive: true,
});
const httpsAgent = new https.Agent({
  keepAlive: true,
});

const promises = [];
for (let i = 1; i <= 2; i++) {
  promises.push(
    new Promise((resolve, reject) => {
      axios
        .get(`http://localhost:8001/${i}`, {
          httpAgent,
          httpsAgent,
        })
        .then(() => {
          console.log(`success ${i}`);
          resolve('success');
        })
        .catch((err) => {
          console.log(`error ${i} ${err}`);
          console.log(`response: ${err.response}`);
          console.log(`code: ${err.code}`);
          reject('error');
        });
    }),
  );
}

await Promise.all(promises);
```

すると、以下のようになります。1個は成功していますが、もう1個のリクエストでは `ECONNRESET` が発生していることがわかります。

```
error 2 Error: read ECONNRESET
response: undefined
code: ECONNRESET
(node:67293) UnhandledPromiseRejectionWarning: error
(Use `node --trace-warnings ...` to show where the warning was created)
(node:67293) UnhandledPromiseRejectionWarning: Unhandled promise rejection. This error originated either by throwing inside of an async function without a catch block, or by rejecting a promise which was not handled with .catch(). To terminate the node process on unhandled promise rejection, use the CLI flag `--unhandled-rejections=strict` (see https://nodejs.org/api/cli.html#cli_unhandled_rejections_mode). (rejection id: 2)
(node:67293) [DEP0018] DeprecationWarning: Unhandled promise rejections are deprecated. In the future, promise rejections that are not handled will terminate the Node.js process with a non-zero exit code.

success 1
```

以上から、サーバー側が許容する最大接続数を上回るリクエストがあった場合に `ECONNRESET` が発生することがわかりました。[^2]

[^2]: ちなみに、 Node.js の [テストコード](https://github.com/nodejs/node/blob/e46c680bf2b211bbd52cf959ca17ee98c7f657f5/test/parallel/test-http-set-timeout-server.js#L169) には `ECONNRESET could happen on a heavily-loaded server.` というコメントが記載されているものがあり、サーバーに負荷が掛かっている場合に `ECONNRESET` が発生しやすいことが示唆されています。

# 観察

Wireshark で観察すると、TCP ヘッダーで `RST` フラグが `1` になっているときに `ECONNRESET` が発生します。

{{< figure src="/blog/img/20211017_econnreset_tcp_rst.png">}}

では、どのような場合に `RST` フラグは `1` になるのでしょうか。[マスタリングTCP/IP 入門編 第5版](https://amzn.to/3n0unXD) の p253 には、次のように書かれています。[^3]

[^3]: 最新は [第6版](https://amzn.to/3j9EacI) ですが、手元に無いので第5版から引用しています。

> このビットが「1」の場合には、コネクションが強制的に切断されます。これは、何らかの異常を検出した場合に送信されます。たとえば、使われていない TCP ポート番号に接続要求が来ても通信はできません。この場合には、RST が「1」に設定されたパケットが返送されます。また、プログラムの暴走や電源断などによって、コンピュータが再起動されると TCP の通信は継続できなくなります。コネクションの情報がすべて初期化されてしまうからです。このような場合に通信相手からパケットが送られてくると、RST が「1」に設定されたパケットを返送して通信を強制的に中断させます。

これは教科書的な記述ですが、もう少し実用的な [日経ITエンジニアスクール TCP/IP最強の指南書](https://amzn.to/3jazPWH) の p130 には、次のように書かれています。

> なお，コネクション確立時と違って，切断手順はいくつかのパターンがある。例えば，RSTフラグを使うと，問答無用で切ることができる。

この書き方からすると、何らかの事情でサーバー側から切断したいときは `RST` フラグを `1` にして送るようですね。[^4]

[^4]: ちなみに、もともと `ECONNRESET` はシステムコール [send](https://linuxjm.osdn.jp/html/LDP_man-pages/man2/send.2.html) で「接続が接続相手によりリセットされた」ときに返るエラーコードとして定義されています。Node.js の実装を確認していないので確かなことは言えませんがメモとして記載しておきます。

# 対策

上記の再現例のように Keep-Alive でタイムアウトしたり最大接続数を上回った場合に `ECONNRESET` が発生しているならば、リトライで回復できそうです。

axios 自体にはリトライ機能はありませんが、 [axios-retry](https://github.com/softonic/axios-retry) というプラグインを使うことができます。

もともと axios には [interceptors](https://axios-http.com/docs/interceptors) という仕組みが備わっています。「途中で捕らえる」「傍受する」といった [意味](https://ejje.weblio.jp/content/intercept) のとおり、interceptors を使うと、リクエストやレスポンスをしたときに任意の処理を行うことができます。axios-retry はその仕組みを使って、エラーレスポンスが返ってきたときにリトライ処理を行います。

Nuxt.js を使っている場合は、[axios モジュール](https://github.com/nuxt-community/axios-module) に axios-retry があらかじめ組み込まれているので `nuxt.config.js` でそれを有効にするだけです（[Nuxt.jsのaxiosにリトライを設定する](https://www.blog.danishi.net/2021/07/31/post-5317/)）。

```javascript
axios: {
  retry: true
},
```

Nuxt.js 以外では、axios-retry を追加でインストールすることで使うことができます。

```
yarn add axios-retry
```

インストールしたら、axios-retry の `import` 文と `axiosRetry()` を追加します。 `axiosRetry()` の第 1 引数には axios インスタンスを渡します。第 2 引数にはリトライのオプションを指定します。

```typescript
import axiosRetry from 'axios-retry';

axiosRetry(axios, { retries: 3 });
```

`axiosRetry()` は引数で渡された axios インスタンスに interceptor を追加します。[es/index.mjs](https://github.com/softonic/axios-retry/blob/v3.2.2/es/index.mjs#L196) を見てみましょう。

```javascript
export default function axiosRetry(axios, defaultOptions) {
  axios.interceptors.request.use((config) => {
    // リクエストするときの interceptor
  });

  axios.interceptors.response.use(null, async (error) => {
    // エラーレスポンスが返ってきたときの interceptor

    const {
      retries = 3,
      retryCondition = isNetworkOrIdempotentRequestError,
      retryDelay = noDelay,
      shouldResetTimeout = false
    } = getRequestOptions(config, defaultOptions);

    const currentState = getCurrentState(config);

    if (await shouldRetry(retries, retryCondition, currentState, error)) {
      currentState.retryCount += 1;
      const delay = retryDelay(currentState.retryCount, error);

      // ...

      return new Promise((resolve) => setTimeout(() => resolve(axios(config)), delay));
    }

    return Promise.reject(error);
  });
}
```

どういったエラーが返ってきたときにリトライするかを判定しているのは、変数 `retryCondition` に代入されている関数です。デフォルトでは `isNetworkOrIdempotentRequestError()` が使われます。

`isNetworkOrIdempotentRequestError()` は、以下のような [実装](https://github.com/softonic/axios-retry/blob/v3.2.2/es/index.mjs#L62) になっています。

```javascript
export function isNetworkOrIdempotentRequestError(error) {
  return isNetworkError(error) || isIdempotentRequestError(error);
}
```

また、 `isNetworkError()` は、以下のような [実装](https://github.com/softonic/axios-retry/blob/v3.2.2/es/index.mjs#L9) になっています。

```javascript
export function isNetworkError(error) {
  return (
    !error.response &&
    Boolean(error.code) && // Prevents retrying cancelled requests
    error.code !== 'ECONNABORTED' && // Prevents retrying timed out requests
    isRetryAllowed(error)
  ); // Prevents retrying unsafe errors
}
```

ここで呼ばれている `isRetryAllowed()` は axios-retry が依存している [is-retry-allowed](https://github.com/sindresorhus/is-retry-allowed/tree/v2.2.0) の関数です。[^5]

[^5]: 偶然にも前回の [記事]({{< ref "posts/pqueue-sample" >}}) で取り上げた [p-queue](https://github.com/sindresorhus/p-queue) と同じ作者のライブラリです。axios-retry に組み込んでしまってもよさそうなコード量ですが...

ところで、axios で `ECONNRESET` が発生したときのエラーオブジェクトの中身は、以下のようになっています。

```javascript
{
  response: undefined,
  code: 'ECONNRESET',
}
```

これを上記の `isNetworkError()` で判定すると `true` になります。axios-retry のデフォルト設定で `ECONNRESET` がリトライされることがわかりました。

# 結語

`ECONNRESET` は、何らかの理由により接続相手（とくにサーバー側）から切断された場合に発生することがわかりました。

偶発的に起こるエラーなのでローカルでの再現は難しいと思っていましたが、Keep-Alive のタイムアウトや最大接続数を制限することで再現できました。

また、axios を使っている場合は axios-retry によってリトライされることも確認できました。

ちなみに、この対応を本番環境に行ってから、いまのところ、 `ECONNRESET` エラーは再発していません。
