---
type: Archive
title: 「初めてのGraphQL」を読んだ(前半)
date: 2021-04-15
description: 読書感想文
titleWrap: wrap
tags: 
- 本

image: images/summary/graphql.jpeg
---

ACMのオライリーサブスクリプションが6月末で終了のお知らせを受けて、慌てて手に取った一冊。
オライリーの本を気軽に読めなくなるのはかなり惜しいが、7月以降にOREIILYに直接支払う形で年間数万円をかけるまでの熱意はない。
{{< blogcard url="https://www.oreilly.co.jp/books/9784873118932/" >}}  


# 前半部分の感想
- SWAPI（https://graphql.org/swapi-graphql/）というものがあり、スターウォーズ情報を提供するAPIサーバがあるらしく大変興味深い。
  ![](/2022-04-15-23-02-31.png)
- グラフ理論の概念がGraphQLの裏側には潜んでいて、このグラフ理論の考え方が、アプリケーションのエンティティと相性がいい。このあたりのグラフ理論とは？グラフ理論のどこがGraphQLと交差するのか？というあたりが丁寧に書かれていて、読み物として面白かった。

# 読書中のメモ
詳細はぜひ本著を、ということで導入部分のさわりだけメモ。
書いてる内容は1～3章くらいの内容で5・6章では実際にGraphQLサーバとクライアントそれぞれの実装を、多少では認可やセキュリティなど実案件で利用する際に気になる内容が記載されている。


## GraphQL入門
- GraphQLはSQLと同じ問い合わせ言語の一つ。SQLとの明確な違いは問い合わせ先がDBか、APIサーバか。
- GraphQLではクエリが入れ子になっている場合でも1回のHTTPリクエストで異なるデータを取得することができる。
 そのため、Restでよくありがちな「あるAPIコールのレスポンスから引っこ抜いた情報を使って他のAPIコール」がなくなる。
- クエリのレスポンスはJsonで、成功すると``data``キーに結果が、失敗すると``errors``キーにエラーの内容が記録されて返却される

## スキーマの設計
- GraphQLはRestAPIとは異なり、デザインプロセスを変える。まずAPIを作成する際に、APIで扱うデータ型を定義する必要がある。この定義のことをスキーマと呼ぶ。
- スキーマ定義言語(SDL)を使って型定義を行う。アプリの言語に依存せずにSDLは使用可能。(そりゃそう)
- 型はオブジェクト指向言語で言うクスと同じ感覚で、フィールドを持つenumなんかもあったりする。  
例: フィールドを保持するPhoto型
  ````.graphql
  type Photo {
    id: ID!
    name: String!  # 組み込み型であるスカラー型
    url: String!
    description: String # スカラー型のNull許容属性
  }
  ````

- フィールドに別の固有型を持たせるとも可能（グラフ理論におけるエッジ⇔ッジの接続）
- 一対多の接続が頻繁に出てくるのは下のようなクエリを定義する場合。

  ````.graphql
  type Query {
    totalPhotos: Int!
    allPhotos: [Photo!]!
    totalUsers: Int!
    allUsers: [User!]!
  }
  schema {
    query: Query
  }
  ````
  以下のように呼び出しが可能になる

  ````.graphql
  query {
    totalPhotos
    allPhotos {
      name
      url
    }
  }
  ````

- フィールドに引数を持たせたり、デタのフィルタリングも可能
- データページングにより取得するデータ量の制御が可能

  ````.graphql
  type Query {
    ……
    allUsers(first: Int=50 start: Int=0): [User!]!
    allPhotos(first: Int=25 start: Int=0): [Photo!]!
  }
  ````

- アプリの状態を更新する「ミューテーション」（詳細割愛）
  