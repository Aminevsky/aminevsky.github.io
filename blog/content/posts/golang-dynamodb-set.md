---
title: "Go言語でDynamoDBのセット型を扱う"
date: 2022-10-15T13:35:00+09:00 
draft: false
categories:
- 開発 
tags:
- golang 
toc: true
---

# 要約

- Go 言語で DynamoDB のセット型を扱うことはできる
- ただし、要素の追加や削除が面倒
- リスト型で十分な場合はリスト型を使うべき

# 前提

各ライブラリのバージョンは以下のとおりです。

```
github.com/aws/aws-sdk-go-v2 v1.16.16
github.com/aws/aws-sdk-go-v2/feature/dynamodb/attributevalue v1.10.0
github.com/aws/aws-sdk-go-v2/feature/dynamodb/expression v1.4.26
github.com/aws/aws-sdk-go-v2/service/dynamodb v1.17.1
```

また、サンプルコードは以下に置いています。

https://github.com/Aminevsky/go-dynamodb-set-sample

# セット型とは

[AWS ドキュメント](https://docs.aws.amazon.com/ja_jp/amazondynamodb/latest/developerguide/HowItWorks.NamingRulesDataTypes.html)
によると、 DynamoDB のセット型には以下のような特徴があります。

- 数値セット、文字列セット、バイナリセットの 3 種類のみ
- 要素は全て同じ型でなければならない
- 要素は一意でなければならない（重複不可）
- 要素の順序は保持されない

たとえば、リスト型は重複を認めるので、次のように格納することができます。

```
[ 1, 1, 2, 3, 4, 5]
```

一方、セット型では、必ず、次のように格納されます。

```
[ 1, 2, 3, 4, 5]
```

# テーブル例

ここでは、次のようなテーブルを想定します。

| 属性名 | キー     | データ型  |
| --- |--------|-------|
| id | パーティションキー | 文字列   |
| team_name |        | 文字列   |
| batting_order          |        | リスト   |
| reserve          |        | 数値セット |

# 型定義

`feature/dynamodb/attributevalue` の `Marshal()` は、デフォルトでスライスをリスト型へ変換します。

https://pkg.go.dev/github.com/aws/aws-sdk-go-v2/feature/dynamodb/attributevalue#Marshal
> Marshaling slices to AttributeValue will default to a List for all types except for []byte and [][]byte.

`Marshal()` でリスト型ではなくセット型へ変換するためには、 構造体タグで明示する必要があります。以下では、フィールド `Reserve` に `numberset` を指定することで、数値セットであることを明示しています。

```go
type BaseballTeam struct {
    ID           string `dynamodbav:"id"`
    TeamName     string `dynamodbav:"team_name"`
    BattingOrder []int  `dynamodbav:"batting_order"`
    Reserve      []int  `dynamodbav:"reserve,numberset"`
}
```

# 要素追加

DynamoDB では、セット型に要素を追加するときは、更新式 `UPDATE` で [ADD](https://docs.aws.amazon.com/ja_jp/amazondynamodb/latest/developerguide/Expressions.UpdateExpressions.html#Expressions.UpdateExpressions.ADD) アクションを使います。

Go 言語で書いた場合は以下のようになります。

```go
func (r *BaseballTeamRepository) AddReserve(ctx context.Context, id string, addNumbers []int) error {
	key, err := attributevalue.MarshalMap(map[string]string{"id": id})
	if err != nil {
		return fmt.Errorf("marshal error: %w", err)
	}

	// each element must be string
	params := make([]string, 0, len(addNumbers))
	for _, n := range addNumbers {
		params = append(params, strconv.Itoa(n))
	}

	// value is number set, not number list
	update := expression.Add(expression.Name("reserve"), expression.Value(types.AttributeValueMemberNS{Value: params}))
	builder, err := expression.NewBuilder().WithUpdate(update).Build()
	if err != nil {
		return fmt.Errorf("builder error: %w", err)
	}

	_, err = r.client.UpdateItem(ctx, &dynamodb.UpdateItemInput{
		Key:                       key,
		TableName:                 aws.String(r.tableName),
		ExpressionAttributeNames:  builder.Names(),
		ExpressionAttributeValues: builder.Values(),
		UpdateExpression:          builder.Update(),
	})
	if err != nil {
		return fmt.Errorf("add to reserve failed: %w", err)
	}

	return nil
}
```

`expression.Value()` の引数としてスライスを直接与えることはできません。デフォルトでスライスはリスト型へ変換されるからです。 `types.AttributeValueMemberNS` を使って明示する必要があります。

```go
update := expression.Add(expression.Name("reserve"), expression.Value(types.AttributeValueMemberNS{Value: params}))
```

また、 `types.AttributeValueMemberNS` のフィールド `Value` の型は `[]string` なので、それに合わせる必要があります。そのため、以下のように `[]int` を `[]string` へ変換しています。

```go
params := make([]string, 0, len(addNumbers))
for _, n := range addNumbers {
    params = append(params, strconv.Itoa(n))
}
```

# 要素削除

セット型から要素を削除するときは、更新式 `UPDATE` で [DELETE](https://docs.aws.amazon.com/ja_jp/amazondynamodb/latest/developerguide/Expressions.UpdateExpressions.html#Expressions.UpdateExpressions.DELETE) を使います。

Go 言語で書いた場合は以下のようになります。`ADD` のときと同様に、あらかじめ `[]string` へ変換しておき、 `types.AttributeValueMemberNS` を使う必要があります。

```go
func (r *BaseballTeamRepository) DeleteReserve(ctx context.Context, id string, deleteNumbers []int) error {
	key, err := attributevalue.MarshalMap(map[string]string{"id": id})
	if err != nil {
		return fmt.Errorf("marshal error: %w", err)
	}

	// each element must be string
	params := make([]string, 0, len(deleteNumbers))
	for _, n := range deleteNumbers {
		params = append(params, strconv.Itoa(n))
	}

	update := expression.Delete(expression.Name("reserve"), expression.Value(types.AttributeValueMemberNS{Value: params}))
	builder, err := expression.NewBuilder().WithUpdate(update).Build()
	if err != nil {
		return fmt.Errorf("builder error: %w", err)
	}

	_, err = r.client.UpdateItem(ctx, &dynamodb.UpdateItemInput{
		Key:                       key,
		TableName:                 aws.String(r.tableName),
		ExpressionAttributeNames:  builder.Names(),
		ExpressionAttributeValues: builder.Values(),
		UpdateExpression:          builder.Update(),
	})
	if err != nil {
		return fmt.Errorf("delete from reserve failed: %s", err)
	}

	return nil
}
```

# まとめ

AWS SDK for Go v2 のデフォルトでは、スライスはリスト型として扱われます。 そのため、セット型を使うためには、構造体タグで明示的に指定したり、 `[]string` へ変換する必要があります。

セット型をどうしても使わなければならない場合以外は、リスト型を使うべきでしょう。