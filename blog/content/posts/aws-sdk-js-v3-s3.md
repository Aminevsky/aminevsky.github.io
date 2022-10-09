---
title: "AWS SDK for JavaScript v3 で S3 からオブジェクトを取得するには"
date: 2021-03-21T18:30:30+09:00
draft: false
categories:
- 開発
tags:
- AWS
- TypeScript
toc: true
---

昨年 12 月に **AWS SDK for JavaScript v3** が [GA](https://aws.amazon.com/jp/about-aws/whats-new/2020/12/aws-sdk-javascript-version-3-generally-available/) された。[^1] [^2]

[^1]: ただし、GA されたとは思えないほど、日本語でも英語でも情報が少ない。一番情報があるのは SDK の [Issues](https://github.com/aws/aws-sdk-js-v3/issues) だと思う。

[^2]: 現時点では Lambda の [実行環境](https://docs.aws.amazon.com/lambda/latest/dg/lambda-runtimes.html) は v2 のままである。Lambda で v3 を使うには webpack などでバンドルするか、Lambda Layers を使うことになると思う。正直、そこまでして使いたくなるほどの魅力的な新機能があるわけでもない...

少し使ってみたところ、S3 からオブジェクトを取得する際に v2 と異なる書き方をしなければならなかったのでメモしておく。[^3]

[^3]: 他にも DynamoDB の `DocumentClient` が無くなり `marshall()` や `unmarshall()` を使わなければならなかった。これはいずれ書くかもしれないし、書かないかもしれない。

なお、AWS が TypeScript で SDK を実装したと [アピール](https://aws.amazon.com/jp/blogs/developer/first-class-typescript-support-in-modular-aws-sdk-for-javascript/) しているので、この記事も TypeScript で書くことにする。

# 前提

- SDK v3
  - `@aws-sdk/client-s3`
    - `3.8.0`
- SDK v2
  - `2.858.0`
- node.js
  - `14.15.5`
- TypeScript
  - `4.2.3`

# インストール

まずは、インストールから。

## v2

v2 では、SDK 全体をインストールする必要があった。

```shell
npm i aws-sdk
```

```typescript
import * as AWS from 'aws-sdk';

const s3 = new AWS.S3({
  apiVersion: '2006-03-01',
});
```

`import` 文は、以下のようにも書くことができる。こうすると、webpack で SDK の必要な部分のみをバンドルすることができ、バンドルサイズを削減できるらしい。

```typescript
import S3 from 'aws-sdk/clients/S3';

const s3 = new S3({
  apiVersion: '2006-03-01',
});
```

## v3

一方、v3 では、インストールの時点から、必要な部分だけをインストールできるようになった。

```shell
npm i @aws-sdk/client-s3
```

```typescript
import { S3Client } from '@aws-sdk/client-s3';

const s3 = new S3Client({
  apiVersion: '2006-03-01',
});
```

# S3

S3 からオブジェクト（例：テキストファイル）を取得し、その中身を出力するプログラムを書いてみる。

なお、 [LocalStack](https://github.com/localstack/localstack) 上に作った `example-bucket` の中に `test.txt` が保存されている前提である。

## v2

v2 で書くと、こうなる。

```typescript
import S3 from 'aws-sdk/clients/S3';

const s3 = new S3({
  apiVersion: '2006-03-01',
  endpoint: 'http://localhost:4566',
  s3ForcePathStyle: true,
});

handler()
  .then((result) => {
    console.log(result);
    console.log('Completed');
  })
  .catch((error) => console.error(error));

async function handler(): Promise<string> {
  const { Body } = await s3
    .getObject({
      Bucket: 'example-bucket',
      Key: 'test.txt',
    })
    .promise();

  if (Body === undefined) {
    throw new Error('Invalid Body');
  }

  return Body.toString();
}
```

実行結果は、以下のようになる。

```shell
# test.txt の中身
TEST FILE

Completed
```

## v3

続いて、v3 で同様の処理を書いてみる。

### NG

あとで書くように、v3 ではメソッドの呼び出し方法が変わっているが、v2 の呼び出し方法もサポートしている。一旦、ここでは v2 スタイルで書いてみる。

```typescript
// import 文が変わった。
import { S3 } from '@aws-sdk/client-s3';

const s3 = new S3({
  apiVersion: '2006-03-01',
  endpoint: 'http://localhost:4566',
  forcePathStyle: true,
});

handler()
  .then((result) => {
    console.log(result);
    console.log('Completed');
  })
  .catch((error) => console.error(error));

async function handler(): Promise<string> {
  // promise() は要らない。
  const { Body } = await s3.getObject({
    Bucket: 'example-bucket',
    Key: 'test.txt',
  });

  if (Body === undefined) {
    throw new Error('Invalid Body');
  }

  return Body.toString();
}
```

実行結果は、以下のようになる。

```shell
# test.txt の中身が出力されない！
[object Object]
Completed
```

これは一体どうしたことだろうか？

### OK (ver.1)

以下のように書くと、v3 でも、S3 からオブジェクトを取得し、その中身を出力することができる。

なお、この方法は SDK の [Issue](https://github.com/aws/aws-sdk-js-v3/issues/1877) に掲載されている。

```typescript
import { S3 } from '@aws-sdk/client-s3';
import { Readable } from 'stream';

const s3 = new S3({
  apiVersion: '2006-03-01',
  endpoint: 'http://localhost:4566',
  forcePathStyle: true,
});

handler()
  .then((result) => {
    console.log(result);
    console.log('Completed');
  })
  .catch((error) => {
    console.error(error);
  });

async function handler(): Promise<string> {
  const { Body } = await s3.getObject({
    Bucket: 'example-bucket',
    Key: 'test.txt',
  });

  if (!(Body instanceof Readable)) {
    throw new Error('Invalid S3 Body');
  }

  // ???
  return new Promise((resolve, reject) => {
    const chunks: Buffer[] = [];

    Body.on('data', (chunk) => chunks.push(chunk));
    Body.on('error', (err) => reject(err));
    Body.on('end', () => resolve(Buffer.concat(chunks).toString()));
  });
}
```

問題は、以下の部分である。

```typescript
return new Promise((resolve, reject) => {
  const chunks: Buffer[] = [];

  Body.on('data', (chunk) => chunks.push(chunk));
  Body.on('error', (err) => reject(err));
  Body.on('end', () => resolve(Buffer.concat(chunks).toString()));
});
```

ここでは Node.js の [Stream](https://nodejs.org/api/stream.html) を使っている。 `getObject()` が返す `Body` は、 [Readable Stream](https://nodejs.org/api/stream.html#stream_class_stream_readable) である。

Readable Stream では、データは以下のように読み込まれる。

1. 送信元からデータを取得する
2. データを内部バッファに貯める
3. 内部バッファから一部を取り出す
4. データの一部が利用できるようになったので `data` イベントが発生する
5. 全てのデータが利用できるようになるまで `data` イベントが発生し続ける
6. 全てのデータが利用できるようになると `end` イベントが発生する
7. エラーが発生した場合は `error` イベントが発生する。

上記をもとに、処理を振り返ると、こうなる。

```typescript
Body.on('data', (chunk) => chunks.push(chunk));
```

まず、データの一部が利用できるようになるたびに、 `data` イベントが発生する。その際、コールバック関数の引数として、利用できるようになった部分のデータが渡されるので、配列に貯めていく。

なお、データは [Buffer](https://nodejs.org/api/buffer.html) オブジェクトとして渡される。 `Buffer` クラスは [Uint8Array](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Uint8Array) のサブクラスで、バイナリを扱うためのクラスである。

```typescript
Body.on('end', () => resolve(Buffer.concat(chunks).toString()));
```

全てのデータが利用できるようになると、 `end` イベントが発生する。 `Buffer` オブジェクトの配列を `Buffer.concat()` でまとめて、1 つの `Buffer` オブジェクトを生成する。そして、 `toString()` で文字列へ変換している。

### OK (ver.2)

先程の例では Node.js の Stream を使った。

しかし、イベントハンドラを 3 つ書いたり、全てのデータが利用できるようになるのを待つために Promise を使ったりするのは、かなり面倒である。

そこで、 [for await...of](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Statements/for-await...of) 文を使って、書き直してみる。

```typescript
import { GetObjectCommand, S3Client } from '@aws-sdk/client-s3';
import { Readable } from 'stream';

const s3 = new S3Client({
  apiVersion: '2006-03-01',
  endpoint: 'http://localhost:4566',
  forcePathStyle: true,
});

handler()
  .then((result) => {
    console.log(result);
    console.log('Completed');
  })
  .catch((error) => console.error(error));

async function handler(): Promise<string> {
  const { Body } = await s3.send(
    new GetObjectCommand({
      Bucket: 'example-bucket',
      Key: 'test.txt',
    }),
  );

  if (!(Body instanceof Readable)) {
    throw new Error('Invalid S3 Body');
  }

  const chunks: Buffer[] = [];

  for await (const chunk of Body) {
    chunks.push(chunk);
  }

  return Buffer.concat(chunks).toString();
}
```

`for await...of` 文を使うと、Stream の反復処理を行うことができるので、イベントハンドラも Promise も書く必要が無くなる。

また、v2 スタイルで以下のように書いてきたが、

```typescript
const { Body } = await s3.getObject({
  Bucket: 'example-bucket',
  Key: 'test.txt',
});
```

v3 ではコマンドを送るスタイルになった。

```typescript
const { Body } = await s3.send(
  new GetObjectCommand({
    Bucket: 'example-bucket',
    Key: 'test.txt',
  }),
);
```

# 参考
- [さよならStream](https://qiita.com/koh110/items/0fba3acbce38916928f1)
- [nodejs v12(LTS)におけるasync, awaitを用いたstream処理](https://qiita.com/kaz2ngt/items/ef2617aab3ae3209d81e)
