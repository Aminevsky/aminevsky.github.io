---
title: "Cognito の JWT をテストするには"
date: 2021-06-13T17:00:00+09:00
draft: false
categories:
- 開発
tags:
- AWS
- TypeScript
toc: true
---

# 要約

AWS Cognito が提供する `jwks.json` を模擬的に再現するプログラムを作りました。

- GitHub
  - https://github.com/Aminevsky/create-jwks

# AWS Cognito

AWS Cognito にログインすると、3 種類の JWT が返却されます。

- アクセストークン
- ID トークン
- リフレッシュトークン

ログイン済みかどうかをチェックするためには、まずクライアントから JWT （アクセストークン or ID トークン）を送ってもらいます。例えば、Cookie や Authorization ヘッダーに JWT を入れて送ってもらいます。

そして、サーバー側では送られてきた JWT が妥当かどうかを検証します。API Gateway ならば [Cognito オーソライザー](https://dev.classmethod.jp/articles/api-gateway-cognito-authorizer/) に検証を任せることができますが、[^1] Lambda@Edge のように自前で検証しなければならない場合も多いでしょう。

[^1]: ところで、Cognito オーソライザーで「アクセストークン」を検証することはできないんですかね？ 試してみたかぎり「ID トークン」でしか検証できない。

この記事では、後者のように自前で検証しなければならず、それゆえテストも自前で行われなければならない場合を想定しています。

## 公開鍵

Cognito で発行された JWT の妥当性を検証するためには、まず、AWS から各ユーザープールの公開鍵を取得する必要があります。公開鍵は、以下からダウンロードできます。

```
https://cognito-idp.{リージョン}.amazonaws.com/{ユーザープールID}/.well-known/jwks.json
```

例えば、AWS の [記事](https://aws.amazon.com/jp/premiumsupport/knowledge-center/decode-verify-cognito-json-token/) から引用すると、以下のようになります。

```json
{
    "keys": [{
        "alg": "RS256",
        "e": "AQAB",
        "kid": "abcdefghijklmnopqrsexample=",
        "kty": "RSA",
        "n": "lsjhglskjhgslkjgh43lj5h34lkjh34lkjht3example",
        "use": "sig"
    }, {
        "alg": "RS256",
        "e": "AQAB",
        "kid": "fgjhlkhjlkhexample=",
        "kty": "RSA",
        "n": "sgjhlk6jp98ugp98up34hpexample",
        "use": "sig"
    }]
}
```

`keys` は配列になっていて、要素が 2 つ格納されています。これは、それぞれ「アクセストークン」と「ID トークン」の公開鍵に対応しています。

送られてくる JWT は「アクセストークン」かもしれないですし、「ID トークン」かもしれません。 `jwks.json` にある公開鍵のうち、どちらを使うべきかは、JWT のヘッダーにある `kid` （キー ID）で判断します。 `kid` が一致すれば、それが使うべき公開鍵ということになります。

## 検証フロー

以下が、JWT の検証フローです。

- `jwks.json` の中の公開鍵をそれぞれ JWK 形式から PEM 形式へ変換する
- JWT をデコードする
- ペイロードの `iss` （発行者）が `https://cognito-idp.{リージョン}.amazonaws.com/{ユーザープールID}` と一致しているかをチェックする
- ペイロードの `token_use` が、「ID トークン」で認証するならば `id` に、「アクセストークン」で認証するならば `access` になっているかをチェックする
- ヘッダーの `kid` と一致する `kid` を持つ公開鍵があるかをチェックする
- JWT の署名などを検証する

詳しくは、[公式ドキュメント](https://docs.aws.amazon.com/cognito/latest/developerguide/amazon-cognito-user-pools-using-tokens-verifying-a-jwt.html) や [Lambda@Edge のサンプル](https://aws.amazon.com/jp/blogs/networking-and-content-delivery/authorizationedge-how-to-use-lambdaedge-and-json-web-tokens-to-enhance-web-application-security/) を参照してください。

どの処理もライブラリを呼び出すだけだったり、簡単な条件分岐を書くだけなのですが、いずれにしても自分でコードを書かなければなりません。

そして、コードを書いたらテストをしなければなりません。

## テスト

テストでは、ペイロードやヘッダーの中身が想定と異なるような JWT を作る必要があります。

ところで、Cognito が発行する JWT は、秘密鍵を使って署名されています。当然のことながら、この秘密鍵は AWS の奥底に隠されていて、私たちには窺い知ることができません。

JWT の仕様上、ペイロードやヘッダーの中身を変更したら署名し直さなければなりません。しかし、Cognito の秘密鍵は分かりません。

そこで、検証フローが正しいかどうかをテストするためには、以下のような作業が必要になります。

- テスト用の秘密鍵と公開鍵を作る
- ペイロードやヘッダーの中身を変える
- テスト用の秘密鍵で署名して JWT を作る
- テスト用の公開鍵で JWT を検証する

このプログラムでは、上記の作業のうち、テスト用の秘密鍵と公開鍵を作り、Cognito が提供しているような `jwks.json` も作ります。

# プログラム解説
## 概略

`yarn run:create` を実行すると、 `keys` ディレクトリの下に、以下のファイルを作ります。

- 秘密鍵
- 公開鍵（PEM 形式）
- `jwks.json`

なお、秘密鍵と公開鍵を作るときに OpenSSL を呼び出しているので、Windows では動かないかもしれません。

また、おまけとして、 `yarn run:example` を実行すると、JWT の作成と検証を行うサンプルも添付しています。

## 実装メモ

すでに似たようなことをしている人がいました。

- [OpenID ConnectのJWTとJWKを手軽につくりたい](https://qiita.com/shu-yusa/items/36855cf1e9b4ec2adf28)

上記記事ではシェルスクリプトを書いていますが、Node.js で全て実行するようにしました。

OpenSSL も Node.js の [execSync()](https://nodejs.org/docs/latest-v14.x/api/child_process.html#child_process_child_process_execsync_command_options) で呼び出しています。

```typescript
execSync(`openssl genrsa 2048 > ${privatePemPath}`);
execSync(`openssl rsa -pubout < ${privatePemPath} > ${publicPemPath}`);
```

また、ヘッダーやペイロードを署名する部分は [jsonwebtoken](https://github.com/auth0/node-jsonwebtoken) の `sign()` を使っています。

上記記事では JWK で `"use":"sig"` を追加する場合などに `jq` コマンドを使っていますが、実は [pem-jwk](https://github.com/dannycoates/pem-jwk) の [pem2jwk()](https://github.com/dannycoates/pem-jwk/blob/master/index.js#L150) には、第2引数として `extras` というのがあります。

```javascript
function pem2jwk(pem, extras) {
  var text = pem.toString().split(/(\r\n|\r|\n)+/g)
  text = text.filter(function(line) {
    return line.trim().length !== 0
  });
  var decoder = getDecoder(text[0])

  text = text.slice(1, -1).join('')
  return decoder(Buffer.from(text.replace(/[^\w\d\+\/=]+/g, ''), 'base64'), extras)
}
```

これを辿っていくと...

```javascript
function decodeRsaPublic(buffer, extras) {
  var key = RSAPublicKey.decode(buffer, 'der')
  var e = pad(key.e.toString(16))
  var jwk = {
    kty: 'RSA',
    n: bn2base64url(key.n),
    e: hex2b64url(e)
  }
  return addExtras(jwk, extras)
}
```

JWK オブジェクトに、引数で指定したプロパティをそのまま追加していることが分かります。

```javascript
function addExtras(obj, extras) {
  extras = extras || {}
  Object.keys(extras).forEach(
    function (key) {
      obj[key] = extras[key]
    }
  )
  return obj
}
```

README にも記載されていない **裏機能** ぽいですが、もうほとんど更新されていないライブラリなので今更削除されることも無いだろうと踏んで、ありがたく使わせてもらうことにしました。

# 補足

[1週間前](https://aminevsky.github.io/blog/posts/20210605/) にも書きましたが、Cognito の公開鍵は「将来ローテートするように変更するかもしれない」です。

- [AWS Developer Forums: JWT signing key rotation ...](https://forums.aws.amazon.com/thread.jspa?threadID=241570)

> Cognito UserPool currently does not rotate keys but this behavior can be changed in future. For example, we may periodically rotate keys or allow a developer to replace the keys. We recommend that you cache each key in jwks uri against kid. Now when you process Id or Access token in your APIs, you should check the kid in JWT header and retrieve the key from cache. When your API sees token with different kid, you should query jwks uri again to check if keys have been changed and update your cache accordingly with new keys.

なので、これを信じるならば、 `jwks.json` はキャッシュしておいて定期的に取得するように実装するのが安全でしょう。[^2]

[^2]: でも、これって、具体的にはどういう実装にしたらいいんでしょうね？ とくに Lambda@Edge の場合は「関数タイムアウト」が 5 秒（ビューワーリクエスト）といった感じに制約が厳しいので、あまり重たい処理はできないと思います。最近作ったときは Lambda ハンドラーの外にグローバル変数を作って、変数の中にデータが無ければ axios で取得するという超簡易的キャッシュを実装しましたが、あれで良かった自信が全く無いですね...
