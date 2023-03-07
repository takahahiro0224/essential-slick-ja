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

Slick lets us delete rows using the same `Query` objects we saw in [Chapter 2](#selecting).
That is, we specify which rows to delete using the `filter` method, and then call `delete`:

```scala mdoc
val removeHal: DBIO[Int] =
  messages.filter(_.sender === "HAL").delete

exec(removeHal)
```

The return value is the number of rows affected.

The SQL generated for the action can be seen by calling `delete.statements`:

```scala mdoc
messages.filter(_.sender === "HAL").delete.statements.head
```

Note that it is an error to use `delete` in combination with `map`. We can only call `delete` on a `TableQuery`:

```scala mdoc:fail
messages.map(_.content).delete
```


## Updating Rows {#UpdatingRows}

So far we've only looked at inserting new data and deleting existing data. But what if we want to update existing data without deleting it first? Slick lets us create SQL `UPDATE` actions via the kinds of `Query` values we've been using for selecting and deleting rows.


<div class="callout callout-info">
**Restoring Data**

In the last section we removed all the rows for HAL. Before continuing with updating rows, we should put them back:

```scala mdoc
exec(messages.delete andThen (messages ++= freshTestData) andThen messages.result)
```

_Action combinators_, such as `andThen`, are the subject of the next chapter.
</div>

### Updating a Single Field

In the `Messages` we've created so far we've referred to the computer from *2001: A Space Odyssey* as "`HAL`", but the correct name is "`HAL 9000`". 
Let's fix that.

We start by creating a query to select the rows to modify, and the columns to change:

```scala mdoc
val updateQuery =
  messages.filter(_.sender === "HAL").map(_.sender)
```

We can use `update` to turn this into an action to run.
Update requires the new values for the column we want to change:

```scala mdoc
exec(updateQuery.update("HAL 9000"))
```

We can retrieve the SQL for this query by calling `updateStatment` instead of `update`:

```scala mdoc
updateQuery.updateStatement
```

Let's break down the code in the Scala expression.
By building our update query from the `messages` `TableQuery`, we specify that we want to update records in the `message` table in the database:

```scala mdoc
val messagesByHal = messages.filter(_.sender === "HAL")
```

We only want to update the `sender` column, so we use `map` to reduce the query to just that column:

```scala mdoc
val halSenderCol = messagesByHal.map(_.sender)
```

Finally we call the `update` method, which takes a parameter of the *unpacked* type (in this case `String`):

```scala mdoc
val action: DBIO[Int] = halSenderCol.update("HAL 9000")
```

Running that action would return the number of rows changed.

### Updating Multiple Fields

We can update more than one field at the same time by mapping the query to a tuple of the columns we care about...

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

...and then supplying the tuple values we want to used in the update:

```scala mdoc
val updateAction: DBIO[Int] =
  query.update(("HAL 9000", "Sure, Dave. Come right in."))

exec(updateAction)

exec(messages.filter(_.sender === "HAL 9000").result)
```

Again, we can see the SQL we're running using the `updateStatement` method. The returned SQL contains two `?` placeholders, one for each field as expected:

```scala mdoc
messages.
  filter(_.id === 1016L).
  map(message => (message.sender, message.content)).
  updateStatement
```

We can even use `mapTo` to use case classes as the parameter to `update`:

```scala mdoc
case class NameText(name: String, text: String)

val newValue = NameText("Dave", "Now I totally don't trust you.")

messages.
  filter(_.id === 1016L).
  map(m => (m.sender, m.content).mapTo[NameText]).
  update(newValue)
```

### Updating with a Computed Value

Let's now turn to more interesting updates. How about converting every message to be all capitals? Or adding an exclamation mark to the end of each message? Both of these queries involve expressing the desired result in terms of the current value in the database. In SQL we might write something like:

~~~ sql
update "message" set "content" = CONCAT("content", '!')
~~~

This is not currently supported by `update` in Slick, but there are ways to achieve the same result.
One such way is to use Plain SQL queries, which we cover in [Chapter 7](#PlainSQL).
Another is to perform a *client-side update* by defining a Scala function to capture the change to each row:

```scala mdoc
def exclaim(msg: Message): Message =
  msg.copy(content = msg.content + "!")
```

We can update rows by selecting the relevant data from the database, applying this function, and writing the results back individually. Note that approach can be quite inefficient for large datasets---it takes `N + 1` queries to apply an update to `N` results.

You may be tempted to write something like this:

```scala mdoc
def modify(msg: Message): DBIO[Int] =
  messages.filter(_.id === msg.id).update(exclaim(msg))

// Don't do it this way:
for {
  msg <- exec(messages.result)
} yield exec(modify(msg))
```

This will have the desired effect, but at some cost.
What we have done there is use our own `exec` method which will wait for results.
We use it to fetch all rows, and then we use it on each row to modify the row.
That's a lot of waiting.
There is also no support for transactions as we `db.run` each action separately.

A better approach is to turn our logic into a single `DBIO` action using _action combinators_.
This, together with transactions, is the topic of the next chapter.

However, for this particular example, we recommend using Plain SQL ([Chapter 7](#PlainSQL)) instead of client-side updates.


## Take Home Points

For modifying the rows in the database we have seen that:

* inserts are via a  `+=` or `++=` call on a table;

* updates are via an `update` call on a query, but are somewhat limited when you need to update using the existing row value; and

* deletes are via a  `delete` call to a query.

Auto-incrementing values are inserted by Slick, unless forced. The auto-incremented values can be returned from the insert by using `returning`.

Databases have different capabilities. The limitations of each profile is listed in the profile's Scala Doc page.


## Exercises

The code for this chapter is in the [GitHub repository][link-example] in the _chapter-03_ folder.  As with chapter 1 and 2, you can use the `run` command in SBT to execute the code against an H2 database.


<div class="callout callout-info">
**Where Did My Data Go?**

Several of the exercises in this chapter require you to delete or update  content from the database.
We've shown you above how to restore you data,
but if you want to explore and change the schema you might want to completely reset the schema.

In the example code we provide a `populate` method you can use:

``` scala
exec(populate)
```

This will drop, create, and populate the `messages` table with known values.

Populate is defined as:

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

We'll meet `asTry` and `andThen` in the next chapter.
</div>


### Get to the Specifics

In [Inserting Specific Columns](#insertingSpecificColumns) we looked at only inserting the sender column:

```scala mdoc:silent
messages.map(_.sender) += "HAL"
```

This failed when we tried to use it as we didn't meet the requirements of the `message` table schema.
For this to succeed we need to include `content` as well as `sender`.

Rewrite the above query to include the `content` column.

<div class="solution">
The requirements of the `messages` table is `sender` and `content` can not be null.
Given this, we can correct our query:

```scala mdoc
val senderAndContent = messages.map { m => (m.sender, m.content) }
val insertSenderContent = senderAndContent += ( ("HAL","Helllllo Dave") )
exec(insertSenderContent)
```

We have used `map` to create a query that works on the two columns we care about.
To insert using that query, we supply the two field values.

In case you're wondering, we've out the extra parentheses around the column values
to be clear it is a single value which is a tuple of two values.
</div>

### Bulk All the Inserts

Insert the conversation below between Alice and Bob, returning the messages populated with `id`s.

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
For this we need to use a batch insert (`++=`) and `into`:

```scala mdoc
val messageRows =
  messages returning messages.map(_.id) into { (message, id) =>
    message.copy(id = id)
  }

exec(messageRows ++= conversation).foreach(println)
```
</div>

### No Apologies

Write a query to delete messages that contain "sorry".

<div class="solution">
The pattern is to define a query to select the data, and then use it with `delete`:

```scala mdoc
messages.filter(_.content like "%sorry%").delete
```
</div>


### Update Using a For Comprehension

Rewrite the update statement below to use a for comprehension.

```scala mdoc
val rebootLoop = messages.
  filter(_.sender === "HAL").
  map(msg => (msg.sender, msg.content)).
  update(("HAL 9000", "Rebooting, please wait..."))
```

Which style do you prefer?

<div class="solution">
We've split this into a `query` and then an `update`:

```scala mdoc
val halMessages = for {
  message <- messages if message.sender === "HAL"
} yield (message.sender, message.content)

val rebootLoopUpdate = halMessages.update(("HAL 9000", "Rebooting, please wait..."))
```
</div>

### Selective Memory

Delete `HAL`s first two messages. This is a more difficult exercise.

You don't know the IDs of the messages, or the content of them.
But you do know the IDs increase. 

Hints: 

- First write a query to select the two messages. Then see if you can find a way to use it as a subquery.

- You can use `in` in a query to see if a value is in a set of values returned from a query.

<div class="solution">
We've selected HAL's message IDs, sorted by the ID, and used this query inside a filter:

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
