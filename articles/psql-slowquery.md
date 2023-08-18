---
title: "MySQL/PlanetScaleの読み取り行数を節約しスロークエリを改善した話"
emoji: "🪐"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [mysql,planetscale,db,prisma]
published: true
---

## 対象読者

- PlanetScaleの読み取り行数を節約したい人
- データベースのスロークエリを改善したい人
- Prismaでの照合順序を含めたINDEXの貼り方を知りたい人
- PlanetScaleの無料枠を超えたらどうなるか知りたい人

## 事の始まり

こちらの記事でWebアプリをサーバーレスアーキテクチャに移行した際に、データベースはMySQLベースのサーバーレスDB、PlanetScaleを採用しました。

https://zenn.dev/bmth/articles/comike-oshinagaki

先日制作を終えた後すぐPlanetScaleの無料枠範囲内である10億行Readを超えてしまいました。
Next.jsの開発サーバーで再読み込みするたびに万単位でRow readsが増えていることには気づいていたのですが、期日内にリリースすることを最優先に考えていたため制作中は後回しにしていたのです。
PlanetScaleの無料プランでは制限を超えて利用した場合もお金はかからず、「サービスが低下する可能性がある」ということのようでした。
本番環境ではキャッシュが効くとはいえ、開発サーバーでページを読み込むたびに数万行読むのは不快です。
初回アクセスのページでは読込み時間もかかっていたし、いつ止められるのかわからない状態で使い続けるのも怖いので直したいところです。
![psql.png](/images/psql-slowquery/psql.png)

## 調査

まずはどのクエリが重いのかを調べます。

ORMにログを出力させ、重そうなクエリに当たりを付けそのクエリにEXPLAINを付けて実行計画を出力させることで重いクエリを確認します。

```SQL:SQL
EXPLAIN
SELECT
    *
FROM
    oshinakai.space_view
WHERE
    event_id = 'C102'
```

Viewに対して実行するSQL文がスロークエリとなっていました。
space表の内容と、そのspaceに紐づく投稿日時が最新のtweet表の内容を取得するクエリです。
実際のクエリは62行もある複雑なクエリで話の本質ではないため省略します。

```table:出力結果
id | select_type | table           | type   | possible_keys              | key                | key_len | ref                   | rows   | filtered | Extra
-----------------------------------------------------------------------------------------------------------------------------------------
1  | PRIMARY     | j1              | const  | PRIMARY                    | PRIMARY            | 766     | const                 | 1      | 100      | Using index; Using temporary; Using filesort
1  | PRIMARY     | t1              | ref    | PRIMARY,day_event_id_idx   | day_event_id_idx   | 766     | const                 | 2      | 100      | Using where; Using index; Start temporary
1  | PRIMARY     | j0              | eq_ref | PRIMARY                    | PRIMARY            | 4       | oshinagaki.t1.id      | 1      | 16.67    | Using where
1  | PRIMARY     | s               | ref    | PRIMARY,space_day_id_idx   | space_day_id_idx   | 5       | oshinagaki.t1.id      | 8718   | 100      | Using index
1  | PRIMARY     | s               | eq_ref | PRIMARY                    | PRIMARY            | 4       | oshinagaki.s.id       | 1      | 11.11    | Using where
1  | PRIMARY     | orderby_1_Block | eq_ref | PRIMARY                    | PRIMARY            | 4       | oshinagaki.s.block_id | 1      | 100      |
1  | PRIMARY     | s               | eq_ref | PRIMARY,space_block_id_idx | PRIMARY            | 4       | oshinagaki.s.id       | 1      | 100      | Using where
1  | PRIMARY     | j0              | eq_ref | PRIMARY                    | PRIMARY            | 4       | oshinagaki.s.block_id | 1      | 10       | Using where
1  | PRIMARY     | <derived6>      | ref    | <auto_key1>                | <auto_key1>        | 5       | oshinagaki.s.id       | 16     | 100      |
1  | PRIMARY     | t               | ref    | tweet_space_id_idx         | tweet_space_id_idx | 5       | oshinagaki.s.id       | 2      | 100      | Using where
1  | PRIMARY     | <derived8>      | ref    | <auto_key1>                | <auto_key1>        | 5       | oshinagaki.s.id       | 16     | 100      |
1  | PRIMARY     | t               | ref    | tweet_space_id_idx         | tweet_space_id_idx | 5       | oshinagaki.s.id       | 2      | 100      | Using where
1  | PRIMARY     | <derived10>     | ref    | <auto_key1>                | <auto_key1>        | 5       | oshinagaki.s.id       | 16     | 100      | Using where
1  | PRIMARY     | t               | ref    | tweet_space_id_idx         | tweet_space_id_idx | 5       | oshinagaki.s.id       | 2      | 100      | Using where; End temporary
6  | DERIVED     | tweet           | index  | tweet_space_id_idx         | tweet_space_id_idx | 5       |                       | 62318  | 100      |
10 | DERIVED     | tweet           | index  | tweet_space_id_idx         | tweet_space_id_idx | 5       |                       | 62318  | 100      |
8  | DERIVED     | tweet           | index  | tweet_space_id_idx         | tweet_space_id_idx | 5       |                       | 62318  | 100      |
```

いろいろ項目はありますが、右から3番めのrows列が実際に取得している行数です。(横長ですみませんがスクロールしてください。)
74行だけ取得するためのクエリのはずなのに、なんと1回のSELECT文で17ステップ約20万行を読み取っていました…！
tweetテーブルに関しては6万行を3回も読んでいます。
地味に上から4つ目の8718行も大きいです。こちらはspaceの検索にINDEXされていないblock.nameを使用している影響と推察されます。

ちなみにPlanetScaleではどのクエリが重くなっているのかを自動で分析してくれており、管理画面で確認することが可能です。
なんと実行には1秒前後もかかってしまっていたことがわかりました。
![スロークエリ](/images/psql-slowquery/slow-query.png)

## INDEXの作成

tweetテーブルとblockテーブルに問題があることがわかったため改善方法を考えます。
tweetテーブルを取得する際にはそのスペースのツイートの中で最新のツイートを取得させる処理を行っていました。

```SQL
SELECT
     * 
FROM 
    `oshinagaki`.`space` s
LEFT JOIN 
    (SELECT 
        `tweet`.`space_id` AS `space_id`,
        MAX(`tweet`.`created_at`) AS `max_created_at`
    FROM
        `tweet`
    GROUP BY 
        `tweet`.`space_id`) `latest_tweet` 
ON 
    ((`s`.`id` = `latest_tweet`.`space_id`));
```

安直にいけばspace_idと降順に並び替えたcreated_atにINDEXを貼ればいけるはず…！

```diff js:schema.prisma
model Tweet {
  id         Int        @id @default(autoincrement())
  text       String?  @DATABASE_URL.VarChar(400)
  spaceId    Int?       @map("space_id")
  url        String?
  authorName String?    @map("author_name")
  authorId   String?    @map("author_id")
  createdAt  DateTime?  @map("created_at")
  favs       Int
  retweets   Int
  error      Boolean @default(false)
  spaces     Space[]
  spaceView  SpaceView?

+  @@index([spaceId,createdAt(sort:Desc)])
-  @@index([spaceId])
  @@map("tweet")
}
```

DB定義はPrismaのSchemaで管理していたので、Prismaでそのままdb pushできるように公式ドキュメントを読んでやり方を確認してきました。
わざわざSQL文を書かなくてもテーブル定義を簡単に変更でき、試行錯誤がしやすかったです。
https://www.prisma.io/docs/reference/api-reference/prisma-schema-reference#index
CREATE INDEX文でも良いと思います。

WHERE句を使って検索する列にはINDEXを貼ったほうが良いというのが調べてわかったため、同様にblockテーブルのnameにもINDEXを貼りました。
その結果…

```sql:INDEX作成後のEXPLAIN
id | select_type | table           | type   | possible_keys                                    | key                           | key_len | ref                                         | rows   | filtered | Extra
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
1  | PRIMARY     | j1              | const  | PRIMARY,event_id_idx                             | PRIMARY                       | 766     | const                                       | 1      | 100      | Using index; Using temporary; Using filesort
1  | PRIMARY     | t1              | ref    | PRIMARY,day_event_id_idx                         | day_event_id_idx              | 766     | const                                       | 2      | 100      | Using where; Using index; Start temporary
1  | PRIMARY     | j0              | eq_ref | PRIMARY                                          | PRIMARY                       | 4       | oshinagaki.t1.id                            | 1      | 16.67    | Using where
1  | PRIMARY     | j0              | ref    | PRIMARY,block_name_idx                           | block_name_idx                | 766     | const                                       | 3      | 100      | Using index
1  | PRIMARY     | s               | ref    | PRIMARY,space_block_id_idx                       | space_block_id_idx            | 5       | oshinagaki.j0.id                            | 162    | 100      | Using index
1  | PRIMARY     | s               | eq_ref | PRIMARY                                          | PRIMARY                       | 4       | oshinagaki.s.id                             | 1      | 11.11    | Using where
1  | PRIMARY     | orderby_1_Block | eq_ref | PRIMARY                                          | PRIMARY                       | 4       | oshinagaki.s.block_id                       | 1      | 100      |
1  | PRIMARY     | s               | eq_ref | PRIMARY,space_day_id_idx                         | PRIMARY                       | 4       | oshinagaki.s.id                             | 1      | 16.67    | Using where
1  | PRIMARY     | <derived6>      | ref    | <auto_key1>                                      | <auto_key1>                   | 5       | oshinagaki.s.id                             | 16     | 100      |
1  | PRIMARY     | t               | ref    | tweet_space_id_idx,tweet_space_id_created_at_idx | tweet_space_id_created_at_idx | 13      | oshinagaki.s.id,latest_tweet.max_created_at | 1      | 100      | Using index
1  | PRIMARY     | <derived8>      | ref    | <auto_key1>                                      | <auto_key1>                   | 5       | oshinagaki.s.id                             | 16     | 100      | Using where
1  | PRIMARY     | t               | ref    | tweet_space_id_idx,tweet_space_id_created_at_idx | tweet_space_id_created_at_idx | 13      | oshinagaki.s.id,latest_tweet.max_created_at | 1      | 100      | Using where; Using index
1  | PRIMARY     | <derived10>     | ref    | <auto_key1>                                      | <auto_key1>                   | 5       | oshinagaki.s.id                             | 16     | 100      |
1  | PRIMARY     | t               | ref    | tweet_space_id_idx,tweet_space_id_created_at_idx | tweet_space_id_created_at_idx | 13      | oshinagaki.s.id,latest_tweet.max_created_at | 1      | 100      | Using index; End temporary
6  | DERIVED     | tweet           | range  | tweet_space_id_idx,tweet_space_id_created_at_idx | tweet_space_id_created_at_idx | 5       |                                             | 27368  | 100      | Using index for group-by
10 | DERIVED     | tweet           | range  | tweet_space_id_idx,tweet_space_id_created_at_idx | tweet_space_id_created_at_idx | 5       |                                             | 27368  | 100      | Using index for group-by
8  | DERIVED     | tweet           | range  | tweet_space_id_idx,tweet_space_id_created_at_idx | tweet_space_id_created_at_idx | 5       |                                             | 27368  | 100      | Using index for group-by

```

読み取り行を半分以下に減らすことができました！
しかし、まだまだ8万行以上行数があり多いです。

## クエリ自体の最適化

もともと、コード側だけでクエリを実現したかったのですが、使用しているフレームワークのPrismaではリレーション先のテーブルを加工した結果を含めたクエリを1回で取得することができません。
2回の実行を許容するのであればPrismaだけで可能なのですが、それだと遅延を許すことになるためViewを使って予めtweetテーブルの最新のtweet.idをspace表に結合したViewを作成していました。
このアプローチ自体は問題ないはずだったので、View定義をあらためてよくよく確認したところ、View側にTweetテーブルの全てのカラムを含めていたことに気づきました。
Tweetの実体はPrismaでincludeさせるため、View側に持たせてしまうと冗長になってしまいます。
更に、Viewのクエリ実行時にINDEX対象行(この場合Tweet表の主キー)のみのSQL文になっていればINDEXの対象にもなります。
また、集約関数は対象表が大きいとコストが大きくパフォーマンスが悪化することもわかりました。
それなら、とGROUP BYを使用せずにJOINだけで最新のtweet.idを取得するようにViewを定義し直しました。

```sql
SELECT 
    *
FROM
    `space` `s`
    LEFT JOIN (SELECT 
        `t1`.`id` AS `id`, `t1`.`space_id` AS `space_id`
    FROM
        (`tweet` `t1`
    LEFT JOIN `tweet` `t2` ON (((`t1`.`space_id` = `t2`.`space_id`)
        AND (`t1`.`created_at` < `t2`.`created_at`))))
    WHERE
        (`t2`.`space_id` IS NULL)) `t` ON ((`s`.`id` = `t`.`space_id`));
```

## 最終的な実行計画

最後に新たに作成したINDEXと被る、外部キー制約相当のINDEXを削除しました。
(PlanetScaleでは外部キー制約は存在しないので手動で貼る必要があります。)

```sql
id | select_type | table           | type   | possible_keys                           | key                | key_len | ref                       | rows   | filtered | Extra
-----------------------------------------------------------------------------------------------------------------------------------------------
1  | SIMPLE      | j1              | const  | PRIMARY,event_id_idx                    | PRIMARY            | 766     | const                     | 1      | 100      | Using index; Using temporary; Using filesort
1  | SIMPLE      | t1              | ref    | PRIMARY,day_event_id_idx                | day_event_id_idx   | 766     | const                     | 2      | 100      | Using where; Using index; Start temporary
1  | SIMPLE      | j0              | eq_ref | PRIMARY                                 | PRIMARY            | 4       | oshinagaki.t1.id          | 1      | 16.67    | Using where
1  | SIMPLE      | j0              | ref    | PRIMARY,block_name_idx                  | block_name_idx     | 766     | const                     | 3      | 100      | Using index
1  | SIMPLE      | s               | ref    | PRIMARY,space_block_id_idx              | space_block_id_idx | 5       | oshinagaki.j0.id          | 162    | 100      | Using index
1  | SIMPLE      | s               | eq_ref | PRIMARY                                 | PRIMARY            | 4       | oshinagaki.s.id           | 1      | 11.11    | Using where
1  | SIMPLE      | orderby_1_Block | eq_ref | PRIMARY                                 | PRIMARY            | 4       | oshinagaki.s.block_id     | 1      | 100      |
1  | SIMPLE      | s               | eq_ref | PRIMARY,space_day_id_idx                | PRIMARY            | 4       | oshinagaki.s.id           | 1      | 16.67    | Using where
1  | SIMPLE      | t1              | ref    | tweet_space_id_idx                      | tweet_space_id_idx | 5       | oshinagaki.s.id           | 2      | 100      |
1  | SIMPLE      | t2              | ref    | tweet_space_id_idx,tweet_created_at_idx | tweet_space_id_idx | 5       | oshinagaki.t1.space_id    | 2      | 100      | Using where
1  | SIMPLE      | t1              | ref    | tweet_space_id_idx                      | tweet_space_id_idx | 5       | oshinagaki.s.id           | 2      | 100      | Using where
1  | SIMPLE      | t2              | ref    | tweet_space_id_idx,tweet_created_at_idx | tweet_space_id_idx | 5       | oshinagaki.t1.space_id    | 2      | 100      | Using where
1  | SIMPLE      | t1              | ref    | tweet_space_id_idx                      | tweet_space_id_idx | 5       | oshinagaki.s.id           | 2      | 100      |
1  | SIMPLE      | t2              | ref    | tweet_space_id_idx,tweet_created_at_idx | tweet_space_id_idx | 5       | oshinagaki.t1.space_id    | 2      | 100      | Using where; End temporary
```

14ステップ184行にまで減らすことに成功しました！！！
せっかく作ったcreated_atのINDEXは使われていないようですが、GROUP BYを使用しなくなったことで行数を絞り込んだあとにtweet表を取得するようになり、取得行数が大幅に減りました。
主な目的は読み取り行数を削減することでしたが、副産物として当初のクエリでは約750msかかっていたところを約20ms前後にまで短縮することができました！
読み取り行数の多さとパフォーマンスの悪さは比例するという仮定の元、ステップ数は妥協しとにかく読み取り行数を減らすためのチューニングのみを考えていましたが、仮定の通りパフォーマンスが良くなってくれました。

## 終わりに

今回、いろいろ調べてみて思ったのはChatGPTはSQLのパフォーマンスチューニングには役に立たないですね。
いろいろな理論を調べながら試行錯誤することで、やっとここまでたどり着けました。
もともとは自宅サーバーでMySQLを動かしていたためパフォーマンスを気にせずクエリ取り放題でしたが、無料枠という制限があったからこそ強くなれました。
実際にチューニングしてみた記事が少なかったのでお困りの方のお力に慣れれば幸いです。

![](/images/psql-slowquery/psql2.png)
いろいろなINDEXを貼ったり剥がしたりして試行錯誤していたのですが、いろいろ試してたら無料枠の4倍使ってましたw
ちなみにまだ止まってません。
対策を打ったので来月からは無料枠に収まると思いますが、このまま何ヶ月も放置していたらおそらく止まるのでしょうか。


## 参考

こちらの記事を参考にさせていただきました。
https://zenn.dev/yuulab/articles/f415920220acf9

https://zenn.dev/canalun/articles/all_about_mysql_index

https://planetscale.com/docs/concepts/billing#usage-and-usage-limits