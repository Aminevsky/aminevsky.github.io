---
title: "p-queue を使ってみた"
date: 2021-10-09T18:50:00+09:00
draft: false
categories:
- 開発
tags:
- JavaScript
- TypeScript
toc: true
---

仕事で、外部の API に対してリクエストを大量に送りたいということがあった。ただし、件数は約 3000 件。一度に大量にリクエストするわけにはいかないので[^1]、同時に実行できる数に上限を設けたい。

自作してもいいなと思ったが、とりあえずライブラリを使うことにした。いろいろ調べたら [p-queue](https://github.com/sindresorhus/p-queue) がおもしろそうだったので、これで。

[^1]: API が同時にいくつまでリクエストを受け入れるか知らないが、過去にリクエストしすぎてエラーが出たことがあるらしい。

# 前提

- p-queue v7.1.0
- Node.js v14.18.0
- TypeScript v4.4.3
- esbuild v0.13.4

# 概要

[p-queue](https://github.com/sindresorhus/p-queue) は、その名のとおり、Promise のキューを作ってくれる。

サンプルプロジェクトは [こちら](https://github.com/Aminevsky/pqueue-sample) 。

キューを作るときに同時実行数を指定する。

```typescript
const queue = new PQueue({ concurrency: 2 });
```

`add()` でキューに追加していく。最後に `Promise.all()` で全ての Promise が完了するのを待つ。

```typescript
(async () => {
  const promises = [];
  for (let i = 1; i <= 30; i++) {
    promises.push(
      queue.add(async () => {
        const { data } = await axios.post(API_URL, { id: i });
        console.log(data);
      }),
    );
  }
  await Promise.all(promises);
})();
```

実際に動かしてみると、最初に

```
{ message: 'Success: 1' }
{ message: 'Success: 2' }
```

が出て、少し経ってから

```
{ message: 'Success: 3' }
{ message: 'Success: 4' }
```

が出る。このように同時に 2 つまでしか実行されていないことが分かる。

# 備考

`p-queue` は ES Modules (ESM) で作られている。一方、TypeScript はデフォルトで CommonJS (CJS) で出力する。

この不一致のせいで、実行時に以下のエラーが出力されてしまう。

```
internal/modules/cjs/loader.js:1102
      throw new ERR_REQUIRE_ESM(filename, parentPath, packageJsonPath);
      ^

Error [ERR_REQUIRE_ESM]: Must use import to load ES Module: /Users/xxxx/pqueue-sample/node_modules/p-queue/dist/index.js
require() of ES modules is not supported.
require() of /Users/xxxx/pqueue-sample/node_modules/p-queue/dist/index.js from /Users/xxxx/pqueue-sample/dist/client.js is an ES module file as it is a .js file whose nearest parent package.json contains "type": "module" which defines all .js files in that package scope as ES modules.
Instead rename index.js to end in .cjs, change the requiring code to use import(), or remove "type": "module" from /Users/xxxx/pqueue-sample/node_modules/p-queue/package.json.

    at new NodeError (internal/errors.js:322:7)
    at Object.Module._extensions..js (internal/modules/cjs/loader.js:1102:13)
    at Module.load (internal/modules/cjs/loader.js:950:32)
    at Function.Module._load (internal/modules/cjs/loader.js:790:12)
    at Module.require (internal/modules/cjs/loader.js:974:19)
    at require (internal/modules/cjs/helpers.js:93:18)
    at Object.<anonymous> (/Users/xxxx/pqueue-sample/dist/client.js:7:35)
    at Module._compile (internal/modules/cjs/loader.js:1085:14)
    at Object.Module._extensions..js (internal/modules/cjs/loader.js:1114:10)
    at Module.load (internal/modules/cjs/loader.js:950:32) {
  code: 'ERR_REQUIRE_ESM'
}
````

`p-queue` の作者は、この問題のために懇切丁寧な [ドキュメント](https://gist.github.com/sindresorhus/a39789f98801d908bbc7ff3ecc99d99c) を用意してくれている。TypeScript で使う方法は [How can I make my TypeScript project output ESM?](https://gist.github.com/sindresorhus/a39789f98801d908bbc7ff3ecc99d99c#how-can-i-make-my-typescript-project-output-esm) に書かれている。

問題は、この長い手順をやりたいかどうかである。私には ESM はわからぬ。私は、一介のエンジニアである。仕事で渋々 TypeScript を書いているにすぎない。けれども面倒くささに対しては、人一倍に敏感であった。[^2]

[^2]: `p-queue` は v6 まで CJS だった。それが v7 で ESM に変わったことで、多くのユーザが混乱しているようだ。[ここ](https://github.com/sindresorhus/p-queue/issues/134) とか [ここ](https://github.com/sindresorhus/p-queue/issues/144) とか [ここ](https://github.com/sindresorhus/p-queue/issues/152) 。OSS なので作者の決断は尊重されるべきだと思うけど、ESM がまだ十分普及していない状況はもうちょっと考慮してもよかったんじゃないだろうか。「使い方わからないから v6 へ戻す」というユーザもどうかと思うけど...

ということで、esbuild でどうにかすることにした。esbuild のビルドスクリプトに、以下のように書けば ESM で出力してくれる。

```javascript
const esbuild = require('esbuild');

esbuild.buildSync({
  entryPoints: ['src/client.ts'],
  outdir: 'dist',
  outExtension: { '.js': '.mjs' },
  format: 'esm',
});
```

[outExtension](https://esbuild.github.io/api/#out-extension) は、 `.js` を出力するときに拡張子を自動的に ESM の `.mjs` にする。これだけでも ESM で出力してくれるみたいだが、念の為、 [format](https://esbuild.github.io/api/#format) でも `esm` を指定している。
