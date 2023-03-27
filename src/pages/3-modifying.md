```scala mdoc:invisible
import slick.jdbc.H2Profile.api._

case class Message(
  sender:  String,
  content: String,
  id:      Long = 0L)

class MessageTable(tag: Tag) extends Table[Message](tag, "message") {

  def id      = column[Long]("id", O.PrimaryKey, O.AutoInc)
  def sender  = column[String]("sender")
  def content = column[String]("content")

  def * = (sender, content, id).mapTo[Message]
}

lazy val messages = TableQuery[MessageTable]

import scala.concurrent.{Await,Future}
import scala.concurrent.duration._

val db = Database.forConfig("chapter03")

def exec[T](action: DBIO[T]): T = Await.result(db.run(action), 2.seconds)

def freshTestData = Seq(
  Message("Dave", "Hello, HAL. Do you read me, HAL?"),
  Message("HAL",  "Affirmative, Dave. I read you."),
  Message("Dave", "Open the pod bay doors, HAL."),
  Message("HAL",  "I'm sorry, Dave. I'm afraid I can't do that.")
)

exec(messages.schema.create andThen (messages ++= freshTestData))
```

# Creating and Modifying Data {#Modifying}

前章では、selectクエリを使用してデータベースからデータを取得する方法について説明しました。この章では、insert、update、deleteクエリを使用して、保存されたデータを変更する方法を説明します。

SQLの経験者なら、updateやdeleteクエリがselectクエリと多くの類似点があることを知っているでしょう。Slickでも同じで、`Query`モナドとコンビネータを使って様々な種類のクエリを構築しています。[第2章](#selecting)の内容をよく理解していることを確認した上で、次に進んでください。

## Inserting Rows

[第1章](#基本編)で見たように、新しいデータを追加することは、ミュータブルコレクションに対するアペンド操作のように見えます。テーブルに1つの行を挿入するには `+=` メソッドを、複数の行を挿入するには `++=` メソッドを使用することができます。この2つの操作については後述します。

### Inserting Single Rows

テーブルに1行を挿入するには、`+=`メソッドを使用します。これまで見てきた select クエリとは異なり、このメソッドでは中間の `Query` を使わずにすぐに `DBIOAction` を作成できることに注意してください。

```scala mdoc
val insertAction =
  messages += Message("HAL", "No. Seriously, Dave, I can't let you in.")

exec(insertAction)
```

`action`の `DBIO[Int]` 型アノテーションを外したので、Slickが使用している特定の型を確認することができます。
この議論では重要ではありませんが、Slickは内部で多くの異なる種類の`DBIOAction`クラスを使用していることを知っておくのは価値があります。

アクションの結果は、挿入された行の数です。しかし、新しい行に対して生成された主キーのような他のものを返すと便利なことがよくあります。この情報は `returning` と呼ばれるメソッドで取得することができます。その前に、まず主キーがどこから来るのかを理解する必要があります。

### Primary Key Allocation

データを挿入する際、新しい行に主キーを割り当てるかどうかをデータベースに指示する必要があります。自動インクリメントの主キーを宣言するのが一般的で、SQLで手動で指定しなくても、データベースが自動的に値を割り当てるようにします。

Slickでは、カラム定義のオプションで自動インクリメントの主キーを割り当てることができます。第1章に登場した`MessageTable`の定義を思い出してください。


```scala
class MessageTable(tag: Tag) extends Table[Message](tag, "message") {

  def id      = column[Long]("id", O.PrimaryKey, O.AutoInc)
  def sender  = column[String]("sender")
  def content = column[String]("content")

  def * = (sender, content, id).mapTo[Message]
}
```

`O.AutoInc`オプションは、`id`カラムがオートインクリメントであることを指定するもので、Slickは対応するSQLでこのカラムを省略できることを意味します。

```scala mdoc
insertAction.statements.head
```

便宜上、このサンプルコードでは `id` フィールドを case クラスの最後に置き、デフォルト値として `0L` を与えています。
これにより、`Message`型の新しいオブジェクトを作成する際に、このフィールドを省略することができます。

```scala
case class Message(
  sender:  String,
  content: String,
  id:      Long = 0L
)
```

```scala mdoc
Message("Dave", "You're off my Christmas card list.")
```
デフォルトの値である`0L`は、特に何も意味することはないのです。
`O.AutoInc`オプションは、`+=`の挙動を決定するものです。

データベースのデフォルトの自動インクリメント動作をオーバーライドして、独自の主キーを指定したい場合があります。Slickはこれを実現するために `forceInsert` メソッドを提供しています。


```scala mdoc:silent
val forceInsertAction = messages forceInsert Message(
   "HAL",
   "I'm a computer, what would I do with a Christmas card anyway?",
   1000L)
```

このアクションのために生成されたSQLには手動で指定したIDが含まれていること,そしてアクションを実行するとそのIDを持つレコードが挿入されることに注意してください。

```scala mdoc
forceInsertAction.statements.head

exec(forceInsertAction)

exec(messages.filter(_.id === 1000L).result)
```

### Retrieving Primary Keys on Insert

データベースが主キーを割り当てた場合、挿入後にキーを取り戻したいと思うことがよくあります。
Slickは `returning` メソッドでこれをサポートしています。

```scala mdoc
val insertDave: DBIO[Long] =
  messages returning messages.map(_.id) += Message("Dave", "Point taken.")

val pk: Long = exec(insertDave)
```

```scala mdoc:invisible
assert(pk == 1001L, "Text below expects PK of 1001L")
```

`messages returning` の引数は同じテーブルに対する `Query` であり、これが `messages.map(_.id)` がここで意味を持つ理由です。
クエリには、挿入が完了した後にデータベースが返すべきデータを指定します。

先ほど挿入したレコードを調べることで、戻り値がプライマリキーであることを証明することができます。

```scala mdoc
exec(messages.filter(_.id === 1001L).result.headOption)
```

便利なことに、いくつかのキー操作を節約して、常に主キーを返す挿入クエリを定義することができます。

```scala mdoc
lazy val messagesReturningId = messages returning messages.map(_.id)

exec(messagesReturningId += Message("HAL", "Humans, eh."))
```

`messagesReturningId`を使用すると、挿入された行数のカウントではなく、`id`値を返します。

### Retrieving Rows on Insert {#retrievingRowsOnInsert}

データベースによっては、主キーだけでなく、挿入されたレコード全体を取得することができるものもあります。
例えば、`Message`全体を返してもらうことができます。

```scala
exec(messages returning messages +=
  Message("Dave", "So... what do we do now?"))
```

すべてのデータベースが `returning` メソッドを完全にサポートしているわけではありません。
H2 では、インサートから主キーを取得することだけが可能です。

H2でこれを試すと、ランタイムエラーが発生します。

```scala mdoc:crash
exec(messages returning messages +=
  Message("Dave", "So... what do we do now?"))
```

これは残念なことですが、主キーを取得することで、必要なものがすべて揃うことが多いです。

<div class="callout callout-info">

**Profile Capabilities**

Slickのマニュアルには、[各データベースプロファイルのケイパビリティ][link-ref-dbs]の総合表が掲載されています。挿入クエリから完全なレコードを返す機能は、`jdbc.returnInsertOther`ケイパビリティとして参照されます。

各プロファイルのAPIドキュメントには、そのプロファイルが *持っていない* ケイパビリティも記載されています。例えば、[H2 Profile Scaladoc][link-ref-h2driver] ページのトップには、その欠点がいくつか指摘されています。

</div>

`jdbc.returnInsertOther`をサポートしていないデータベースから、完全に入力された `Message` を取得したい場合は、主キーを取得し、挿入したレコードに手動で追加します。Slickでは、別のメソッドである `into` でこれを簡略化しています。


```scala mdoc
val messagesReturningRow =
  messages returning messages.map(_.id) into { (message, id) =>
    message.copy(id = id)
  }

val insertMessage: DBIO[Message] =
  messagesReturningRow += Message("Dave", "You're such a jerk.")

exec(insertMessage)
```

`into`メソッドでは、レコードと新しい主キーを結合するための関数を指定することができます。これは、`jdbc.returnInsertOther`機能をエミュレートするのに最適ですが、挿入されたデータに対するあらゆる後処理を想像して使用することができます。

### Inserting Specific Columns {#insertingSpecificColumns}

データベーステーブルにデフォルト値を持つカラムが多く含まれている場合、挿入クエリでカラムのサブセットを指定することが有用な場合があります。
これは、`insert`を呼び出す前にクエリを `mapping` することで実現できます。

```scala mdoc
messages.map(_.sender).insertStatement
```
`+=`メソッドのパラメータ型は、クエリの *unpacked* 型と一致します。

```scala mdoc
messages.map(_.sender)
```

そこで、`sender`に`String`を渡して、このクエリを実行します。

```scala mdoc:silent:crash
exec(messages.map(_.sender) += "HAL")
```

`content`カラムが我々のスキーマではnullableでないため、クエリは実行時に失敗します。
問題ありません。第5章](#Modelling)でスキーマについて説明するときに、NULL可能なカラムを取り上げることにします。

### Inserting Multiple Rows

例えば、複数の「メッセージ」を同時に挿入したいとします。このとき、`+=`を使って順番に挿入していくこともできます。しかし、この場合、各レコードに対して別々のクエリがデータベースに発行されることになり、大量の挿入には時間がかかることがあります。

その代わり、Slickは *batch inserts* をサポートしており、すべての挿入が一度にデータベースに送信されます。これは第1章ですでに見たとおりです。

```scala mdoc
val testMessages = Seq(
  Message("Dave", "Hello, HAL. Do you read me, HAL?"),
  Message("HAL",  "Affirmative, Dave. I read you."),
  Message("Dave", "Open the pod bay doors, HAL."),
  Message("HAL",  "I'm sorry, Dave. I'm afraid I can't do that.")
)

exec(messages ++= testMessages)
```

このコードでは、1つのSQL文を用意し、`Seq`の各行に対してそれを使用します。
原理的には、Slick はデータベース固有の機能を使ってこの挿入をさらに最適化することができます。
これにより、多数のレコードを挿入する際のパフォーマンスを大幅に向上させることができます。

この章の前半で見たように、単一のインサートのデフォルトの戻り値は、挿入された行の数です。上の複数行の挿入も行数を返しますが、今回は型が `Option[Int]` であることを除いては、行数です。この理由は、JDBCの仕様では、基礎となるデータベースドライバが、挿入された行数が不明であることを示すことを許可しているからです。

また、Slickは `into` メソッドを含む `messages returning...` のバッチ版も提供しています。前節で定義した `messagesReturningRow` クエリを使用して、次のように記述することができます。

```scala mdoc
exec(messagesReturningRow ++= testMessages)
```

### More Control over Inserts {#moreControlOverInserts}

この時点では、データベースに固定データを挿入しています。
別のクエリに基づいてデータを挿入するなど、より柔軟な対応が必要な場合もあります。
Slick は `forceInsertQuery` を使ってこれをサポートします。


`forceInsertQuery`の引数はクエリです。 つまり、形式は以下のようになります。

```scala
 insertExpression.forceInsertQuery(selectExpression)
```

`selectExpression`はほとんど何でも良いのですが、`insertExpression`が必要とするカラムと一致する必要があります。

例えば、クエリは特定の行のデータが既に存在するかどうかをチェックし、存在しない場合はそれを挿入することができます。
つまり、"insert if doesn't exist "関数である。

例えば、監督が「カット！」と言えるのは1回だけだとしましょう。SQLは次のようになります。

~~~ sql
insert into "messages" ("sender", "content")
  select 'Stanley', 'Cut!'
where
  not exists(
    select
      "id", "sender", "content"
    from
      "messages" where "sender" = 'Stanley'
                 and   "content" = 'Cut!')
~~~

かなり複雑に見えますが、徐々に構築していけばいいのです。

この中で厄介なのは、`select 'Stanley', 'Cut!'` の部分で、ここには `FROM` 句がありません。
[第2章](#constantQueries)で、`Query.apply`を使ってそれを作る例を見ました。この状況なら、こうなります。


```scala mdoc
val data = Query(("Stanley", "Cut!"))
```

`data`は、固定値（2つのカラムのタプル）を返す定数クエリです。これは、データベースに対して `SELECT 'Stanley', 'Cut!';` を実行するのと同じことで、私たちが必要とするクエリの一部分でもあります。

また、データがすでに存在するかどうかをテストすることも必要です。これは簡単なことです。

```scala mdoc:silent
val exists =
  messages.
   filter(m => m.sender === "Stanley" && m.content === "Cut!").
   exists
```
行が _存在しない_ ときに `data` を使いたいので、`data` と `exists` を組み合わせて、`filter` ではなく `filterNot` とします。

```scala mdoc:silent
val selectExpression = data.filterNot(_ => exists)
```

最後に、このクエリを `forceInsertQuery` で適用する必要があります。
しかし、insert と select のカラムタイプは一致させる必要があることを忘れないでください。
そこで、`messages`に`map`して、それが正しいことを確認します。

```scala mdoc
val forceAction =
  messages.
    map(m => m.sender -> m.content).
    forceInsertQuery(selectExpression)

exec(forceAction)

exec(forceAction)
```

クエリを最初に実行したときは、メッセージが挿入されます。
2回目は、行に影響はありません。

要約すると、`forceInsertQuery` は、より複雑な挿入を構築するための方法を提供します。
もし、このメソッドで対応できない状況の場合は[7章](#PlainSQL)で説明するPlain SQLの挿入を常に使用することができます。


## Deleting Rows

Slickでは、[第2章](#selecting)で見たのと同じ `Query` オブジェクトを使って行を削除することができます。
つまり、`filter`メソッドで削除する行を指定し、`delete`を呼び出します。

```scala mdoc
val removeHal: DBIO[Int] =
  messages.filter(_.sender === "HAL").delete

exec(removeHal)
```

戻り値は、影響を受けた行の数です。

このアクションのために生成されたSQLは、`delete.statements` を呼び出すことで見ることができます。


```scala mdoc
messages.filter(_.sender === "HAL").delete.statements.head
```

なお、`delete`を`map`と組み合わせて使用するのは誤りです。`delete`は `TableQuery` に対してのみ呼び出すことができます。

```scala mdoc:fail
messages.map(_.content).delete
```


## Updating Rows {#UpdatingRows}

ここまでは、新しいデータの挿入と既存のデータの削除についてだけ見てきました。しかし、既存のデータを削除せずに更新したい場合はどうすればよいのでしょうか。Slickでは、これまで行の選択と削除に使用してきた `Query` 値を使用して、SQLの `UPDATE` アクションを作成できます。


<div class="callout callout-info">

**Restoring Data**

前節では、HAL用の行をすべて削除しました。行の更新を続ける前に、行を元に戻しておく必要があります。

```scala mdoc
exec(messages.delete andThen (messages ++= freshTestData) andThen messages.result)
```

`andThen` のような _Action combinators_ は次の章の主題である。

</div>

### Updating a Single Field

これまで作成した `Messages` の中で、*2001年宇宙の旅* に登場するコンピュータを「`HAL`」と呼んでいましたが、正しくは「`HAL 9000`」です。
これを修正しましょう。

まず、修正する行と変更する列を選択するクエリを作成することから始めます。


```scala mdoc
val updateQuery =
  messages.filter(_.sender === "HAL").map(_.sender)
```

これを実行するアクションに変えるには、`update`を使用します。
updateには、変更したいカラムの新しい値が必要です。

```scala mdoc
exec(updateQuery.update("HAL 9000"))
```

このクエリのSQLは、`update`の代わりに`updateStatment`を呼び出すことで取得することができます。


```scala mdoc
updateQuery.updateStatement
```

Scalaの表現でコードを分解してみましょう。
`messages` `TableQuery` から更新クエリを構築することで、データベースの `message` テーブルのレコードを更新したいことを明示します。


```scala mdoc
val messagesByHal = messages.filter(_.sender === "HAL")
```

更新したいのは `sender` カラムだけなので、`map` を使ってクエリをそのカラムだけに絞り込んでいます。

```scala mdoc
val halSenderCol = messagesByHal.map(_.sender)
```

最後に `update` メソッドを呼び出します。このメソッドは *unpacked* 型のパラメータ (この場合は `String`) を受け取ります。

```scala mdoc
val action: DBIO[Int] = halSenderCol.update("HAL 9000")
```

そのアクションを実行すると、変更された行の数が返されます。

### Updating Multiple Fields

クエリを気になるカラムのタプルにマッピングすることで、複数のフィールドを同時に更新することができます。

```scala mdoc:invisible
val assurance_10167 = exec(messages.filter(_.content like "I'm sorry, Dave%").result)  
assert(assurance_10167.map(_.id) == Seq(1016L), s"Text below assumes ID 1016 exists: found $assurance_10167")
```

```scala mdoc
// 1016 is "I'm sorry, Dave...."
val query = messages.
    filter(_.id === 1016L).
    map(message => (message.sender, message.content))
```

そして、更新に使用したいタプルの値をメソッドに渡します。

```scala mdoc
val updateAction: DBIO[Int] =
  query.update(("HAL 9000", "Sure, Dave. Come right in."))

exec(updateAction)

exec(messages.filter(_.sender === "HAL 9000").result)
```

ここでも、`updateStatement`メソッドを使って実行しているSQLを見ることができます。返されたSQLには、予想通り、各フィールドに1つずつ、2つの `?` プレースホルダーが含まれています。

```scala mdoc
messages.
  filter(_.id === 1016L).
  map(message => (message.sender, message.content)).
  updateStatement
```

また、`mapTo`を使って、`update`のパラメータとしてケースクラスを使うこともできます。

```scala mdoc
case class NameText(name: String, text: String)

val newValue = NameText("Dave", "Now I totally don't trust you.")

messages.
  filter(_.id === 1016L).
  map(m => (m.sender, m.content).mapTo[NameText]).
  update(newValue)
```

### Updating with a Computed Value

それでは、もっと面白いアップデートに目を向けましょう。すべてのメッセージを大文字にするのはどうでしょう？あるいは、各メッセージの末尾に感嘆符をつけるのはどうでしょう。これらのクエリはどちらも、データベース内の現在の値という観点から、希望する結果を表現するものです。SQLでは、次のように書くことができます。

~~~ sql
update "message" set "content" = CONCAT("content", '!')
~~~

これは現在Slickの`update`ではサポートされていませんが、同じ結果を得るための方法はあります。
そのような方法の1つは、[第7章](#PlainSQL)で説明するPlain SQLクエリを使用することです。
もうひとつは、各行の変更を取得するScala関数を定義して、*client-side update* を実行する方法です。

```scala mdoc
def exclaim(msg: Message): Message =
  msg.copy(content = msg.content + "!")
```

データベースから該当するデータを選択し、この関数を適用して、結果を個別に書き戻すことで、行を更新することができます。この方法は大規模なデータセットでは非常に非効率的であることに注意してください。
つまり`N`個の結果に更新を適用するために`N + 1`個のクエリを必要とします。

このような書き方をしたくなるかもしれません。

```scala mdoc
def modify(msg: Message): DBIO[Int] =
  messages.filter(_.id === msg.id).update(exclaim(msg))

// Don't do it this way:
for {
  msg <- exec(messages.result)
} yield exec(modify(msg))
```

これは望ましい効果をもたらしますが、多少の犠牲を伴います。
今回行ったのは、結果を待つための独自の `exec` メソッドを使用することです。
このメソッドを使用してすべての行を取得し、各行に対してこのメソッドを使用して行を変更します。
これは非常に多くの待ち時間が発生します。
また、トランザクションもサポートされていないので、各アクションを個別に `db.run` します。

より良いアプローチは、_action combinators_ を使用して、ロジックを1つの`DBIO`アクションにすることです。
これはトランザクションと一緒に次の章のトピックになります。

しかし、この特定の例では、クライアントサイドの更新ではなく、プレーンSQL（[7章](#PlainSQL)）を使用することをお勧めします。


## Take Home Points

データベースの行の変更については、以下の通りです。

- インサートは、テーブルの`+=`または`++=`呼び出しによって行われます。

- 更新はクエリの`update`呼び出しで行いますが、既存の行の値を使って更新する必要がある場合は、やや制限されます。

- 削除は、クエリへの`delete`呼び出しによって行われます。

自動インクリメントの値は、強制されない限り、Slickによって挿入されます。自動インクリメントされた値は、`returning`を使用することで挿入から戻すことができます。

データベースにはさまざまな機能があります。各プロファイルの制限事項は、そのプロファイルのScala Docページに記載されています。




## Exercises

本章のコードは、[GitHub repository][link-example]の _chapter-03_ フォルダーにあります。1章、2章と同様に、SBTの`run`コマンドを使用して、H2データベースに対してコードを実行することができます。


<div class="callout callout-info">

**Where Did My Data Go?**

本章のいくつかの演習では、データベースからコンテンツを削除したり更新したりする必要があります。上記ではデータを復元する方法を紹介しましたが、スキーマを調査して変更したい場合は、スキーマを完全にリセットすることをお勧めします。

サンプルコードでは、使用可能な`populate`メソッドを提供しています：


``` scala
exec(populate)
```

これにより、`messages`テーブルがドロップされ、作成され、既知の値で入力されます。

Populateとは、以下のように定義されています：

```scala mdoc
import scala.concurrent.ExecutionContext.Implicits.global

def populate: DBIOAction[Option[Int], NoStream, Effect.All] =
  for {    
    // Drop table if it already exists, then create the table:
    _  <- messages.schema.drop.asTry andThen messages.schema.create
    // Add some data:
    count <- messages ++= freshTestData
  } yield count
```

次章で`asTry`と`andThen`について学びます。
</div>


### Get to the Specifics

[Inserting Specific Columns](#insertingSpecificColumns)では、senderカラムの挿入のみ検討しました。 

```scala mdoc:silent
messages.map(_.sender) += "HAL"
```

これは, `message`テーブルスキーマの要件を満たしていないため、使用しようとすると失敗します。これを成功させるためには、`sender`だけでなく`content`も含める必要があります。

上記のクエリを、`content`カラムを含むように書き換えてください。

<div class="solution">
`messages`テーブルの要件は、`sender`と`content`がnullであってはならないことです。これを踏まえると、クエリを修正することができます

```scala mdoc
val senderAndContent = messages.map { m => (m.sender, m.content) }
val insertSenderContent = senderAndContent += ( ("HAL","Helllllo Dave") )
exec(insertSenderContent)
```

`map`を使って、気になる2つのカラムに対して動作するクエリを作成しました。そのクエリを使用して挿入するために、2つのフィールド値を供給します。

念のため、列の値を囲む余分な括弧を削除して、2つの値のタプルが1つの値であることを明確にしています。

</div>

### Bulk All the Inserts

アリスとボブの間で以下の会話を挿入し、`id`を入力したメッセージを返すようにしてください。

```scala mdoc:silent
val conversation = List(
  Message("Bob",  "Hi Alice"),
  Message("Alice","Hi Bob"),
  Message("Bob",  "Are you sure this is secure?"),
  Message("Alice","Totally, why do you ask?"),
  Message("Bob",  "Oh, nothing, just wondering."),
  Message("Alice","Ten was too many messages"),
  Message("Bob",  "I could do with a sleep"),
  Message("Alice","Let's just get to the point"),
  Message("Bob",  "Okay okay, no need to be tetchy."),
  Message("Alice","Humph!"))
```

<div class="solution">
batch insert（`++=`）と`into`を使う必要がありま

```scala mdoc
val messageRows =
  messages returning messages.map(_.id) into { (message, id) =>
    message.copy(id = id)
  }

exec(messageRows ++= conversation).foreach(println)
```
</div>

### No Apologies

"sorry"を含むメッセージを削除するクエリを作成してください。

<div class="solution">
データを選択するクエリを定義し、deleteを使用することで実現できます。

```scala mdoc
messages.filter(_.content like "%sorry%").delete
```
</div>


### Update Using a For Comprehension

以下のupdate文を、for式を使用するように書き直してください。

```scala mdoc
val rebootLoop = messages.
  filter(_.sender === "HAL").
  map(msg => (msg.sender, msg.content)).
  update(("HAL 9000", "Rebooting, please wait..."))
```

どのスタイルが好みですか？

<div class="solution">
`query`と`update`に分けました。

```scala mdoc
val halMessages = for {
  message <- messages if message.sender === "HAL"
} yield (message.sender, message.content)

val rebootLoopUpdate = halMessages.update(("HAL 9000", "Rebooting, please wait..."))
```
</div>

### Selective Memory

HALの最初の2つのメッセージを削除してください。これはより難しい練習です。

メッセージのIDも、内容もわかりません。しかし、あなたはIDが増えることを知っています。

ヒント：

- まず、2つのメッセージを選択するクエリを書きます。次に、サブクエリとして使用する方法を見つけられるかどうかを確認します。

- クエリで`in`を使用すると、ある値がクエリから返される値の集合の中にあるかどうかを確認することができます。

<div class="solution">
HALのメッセージIDを選択し、IDでソートし、このクエリーをフィルター内で使用しています：

```scala mdoc
val selectiveMemory =
  messages.filter{
   _.id in messages.
      filter { _.sender === "HAL" }.
      sortBy { _.id.asc }.
      map    {_.id}.
      take(2)
  }.delete

selectiveMemory.statements.head
```

</div>
