# Joins and Aggregates {#joins}

結合（joins）と集計（aggregates）によるデータの操作は、しばしば複雑さを伴います。
この章では、以下の内容を探索することで、その複雑さを緩和しようとします。

* 異なるスタイルの結合（モナディックスタイルとアプリカティブスタイル）
* 異なる結合方法（内部結合、外部結合、zip結合）
* 集約関数とグループ化

## Two Kinds of Join

Slickには2つの結合スタイルがあります。
1つは、明示的な`join`メソッドを基にしたアプリカティブスタイルです。
これはSQLの`JOIN` ... `ON`構文に非常に似ています。

2番目の結合スタイルは、モナディックスタイルです。
これは`flatMap`を結合する方法として使用します。

これら2つの結合スタイルは互いに排他的ではありません。
クエリ内でこれらを組み合わせて使用することができます。
アプリカティブ結合を作成してモナディック結合で使用することもよくあります。

## Chapter Schema 

結合を示すために、少なくとも2つのテーブルが必要です。
1つはユーザーを格納するテーブルで、もう1つは別のテーブルでメッセージを格納します。
これらのテーブルを横断して結合して、誰がメッセージを送信したかを特定します。

まずは`User`を定義します...

```scala mdoc:silent
import slick.jdbc.H2Profile.api._
import scala.concurrent.ExecutionContext.Implicits.global
```
```scala mdoc
case class User(name: String, id: Long = 0L)

class UserTable(tag: Tag) extends Table[User](tag, "user") {
  def id    = column[Long]("id", O.PrimaryKey, O.AutoInc)
  def name  = column[String]("name")

  def * = (name, id).mapTo[User]
}

lazy val users = TableQuery[UserTable]
lazy val insertUser = users returning users.map(_.id)
```

...そして`Message`を追加します：

```scala mdoc:silent
// 注意：メッセージには送信者が存在し、これはユーザーへの参照となります
case class Message(
  senderId : Long,
  content  : String,
  id       : Long = 0L)

class MessageTable(tag: Tag) extends Table[Message](tag, "message") {
  def id       = column[Long]("id", O.PrimaryKey, O.AutoInc)
  def senderId = column[Long]("senderId")
  def content  = column[String]("content")

  def sender = foreignKey("sender_fk", senderId, users)(_.id)
  def * = (senderId, content, id).mapTo[Message]
}

lazy val messages = TableQuery[MessageTable]
lazy val insertMessages = messages returning messages.map(_.id)
```

```scala mdoc:invisible
import scala.concurrent.{Await,Future}
import scala.concurrent.duration._
val db = Database.forConfig("chapter06")
def exec[T](action: DBIO[T]): T = Await.result(db.run(action), 2.seconds)
```

通常の映画の台本でデータベースを埋めます：

```scala mdoc
def freshTestData(daveId: Long, halId: Long) = Seq(
  Message(daveId, "Hello, HAL. Do you read me, HAL?"),
  Message(halId,  "Affirmative, Dave. I read you."),
  Message(daveId, "Open the pod bay doors, HAL."),
  Message(halId,  "I'm sorry, Dave. I'm afraid I can't do that.")
)

val setup = for {
  _         <- (users.schema ++ messages.schema).create
  daveId    <- insertUser += User("Dave")
  halId     <- insertUser += User("HAL")
  rowsAdded <- messages ++= freshTestData(daveId, halId)
} yield rowsAdded

exec(setup)
```

この章の後半では、より複雑な結合を実現するために、さらにテーブルを追加します。

## Monadic Joins 

前の章でモナディック結合の例を見てきました：

```scala mdoc
val monadicFor = for {
  msg <- messages
  usr <- msg.sender
} yield (usr.name, msg.content)
```

`msg.sender`を使用していることに注目してください。これは`MessageTable`の定義で外部キーとして定義されています。
（このトピックの復習として、[Foreign Keys](#fks)をChapter 5で確認してください。）

同じクエリをfor内包表記を使用せずに表現することもできます：

```scala mdoc
val monadicDesugar =
  messages flatMap { msg =>
    msg.sender.map { usr =>
      (usr.name, msg.content)
    }
  }
```

どちらの方法でも、クエリを実行するとSlickが次のようなSQLを生成します：

```sql
select
  u."name", m."content"
from
  "message" m, "user" u
where
  u."id" = m."sender"
```

これが、モナディックスタイルのクエリで、外部キーの関係を使用して結合しています。

<div class="callout callout-info">

**コードの実行**

このセクションの例クエリは、関連するGitHubリポジトリの`joins.sql`ファイルにあります。

`chapter-06`フォルダでSBTを起動し、SBTの`>`プロンプトで次のコマンドを実行します：

~~~
run

Main JoinsExample
~~~
</div>

外部キーが存在しない場合でも、同じスタイルを使用して結合し、結合を制御することができます：

```scala mdoc
val monadicFilter = for {
  msg <- messages
  usr <- users if usr.id === msg.senderId
} yield (usr.name, msg.content)
```

ここでは`msg.senderId`を使用しており、外部キーの`sender`ではないことに注意してください。
これにより、`sender`を使用した場合と同じクエリが生成されます。

このスタイルの結合の例はたくさんあります。
読みやすく、自然に記述することができます。
ただし、Slickはモナディックな式をSQLが実行可能な形式に変換する必要があるため、そのコストがかかります。

## Applicative Joins 

アプリカティブ結合は、結合を明示的にコードで記述する方法です。
SQLでは、`JOIN`と`ON`キーワードを使用しますが、
Slickでは次のメソッドを使用してこれらを反映させます。

- `join` --- インナージョイン

- `joinLeft` --- 左外部結合

- `joinRight` --- 右外部結合

- `joinFull` --- フル外部結合

それぞれのメソッドの例を紹介しますが、
まずは`messages`テーブルを`users`テーブルと`senderId`で結合する方法をご紹介します。

```scala mdoc
val applicative1: Query[(MessageTable, UserTable), (Message, User), Seq] =
  messages join users on (_.senderId === _.id)
```

このコードでは、`(MessageTable, UserTable)`のクエリが生成されます。
`on`の部分で使用する値をより明示的にすることもできます。

```scala mdoc
val applicative2: Query[(MessageTable, UserTable), (Message, User), Seq] =
  messages join users on ( (m: MessageTable, u: UserTable) =>
    m.senderId === u.id
   )
```

パターンマッチを使用して結合条件を記述することもできます。

```scala mdoc
val applicative3: Query[(MessageTable, UserTable), (Message, User), Seq] =
  messages join users on { case (m, u) =>  m.senderId === u.id }
```

このような結合は、通常の方法でアクションに変換することができます。

```scala mdoc
val action: DBIO[Seq[(Message, User)]] = applicative3.result

exec(action)
```

最終的な結果は、`(Message, User)`のシーケンスで、各メッセージが対応するユーザーとペアになっています。

### More Tables, Longer Joins

```scala mdoc:reset:invisible
import slick.jdbc.H2Profile.api._
import scala.concurrent.ExecutionContext.Implicits.global

case class User(name: String, id: Long = 0L)

class UserTable(tag: Tag) extends Table[User](tag, "user") {
  def id    = column[Long]("id", O.PrimaryKey, O.AutoInc)
  def name  = column[String]("name")

  def * = (name, id).mapTo[User]
}

lazy val users = TableQuery[UserTable]
lazy val insertUser = users returning users.map(_.id)

import scala.concurrent.{Await,Future}
import scala.concurrent.duration._
val db = Database.forConfig("chapter06")
def exec[T](action: DBIO[T]): T = Await.result(db.run(action), 2.seconds)
```


このセクションの残りの部分では、さまざまな結合を詳しく見ていきます。
この章で使用しているスキーマについては、図6.1を参照すると役立ちます。


![
The database schema for this chapter.
Find this code in the _chat-schema.scala_ file of the example project on GitHub.
A _message_ can have a _sender_, which is a join to the _user_ table.
Also, a _message_ can be in a _room_, which is a join to the _room_ table.
](src/img/Schema.png)

まず、さらに1つのテーブルを追加します。
これは、`User`が所属できる`Room`で、チャットの会話用のチャネルを提供します。

```scala mdoc:silent


case class Room(title: String, id: Long = 0L)

class RoomTable(tag: Tag) extends Table[Room](tag, "room") {
  def id    = column[Long]("id", O.PrimaryKey, O.AutoInc)
  def title = column[String]("title")
  def * = (title, id).mapTo[Room]
}

lazy val rooms = TableQuery[RoomTable]
lazy val insertRoom = rooms returning rooms.map(_.id)
```

そして、メッセージをオプションでルームに関連付けるようにメッセージを変更します。

```scala mdoc:silent
case class Message(
  senderId : Long,
  content  : String,
  roomId   : Option[Long] = None,
  id       : Long = 0L)

class MessageTable(tag: Tag) extends Table[Message](tag, "message") {
  def id       = column[Long]("id", O.PrimaryKey, O.AutoInc)
  def senderId = column[Long]("senderId")
  def content  = column[String]("content")
  def roomId   = column[Option[Long]]("roomId")

  def sender = foreignKey("sender_fk", senderId, users)(_.id)
  def * = (senderId, content, roomId, id).mapTo[Message]
}

lazy val messages = TableQuery[MessageTable]
lazy val insertMessages = messages returning messages.map(_.id)
```

データベースをリセットし、"Air Lock"ルームで発生するいくつかのメッセージを追加します。

```scala mdoc
exec(messages.schema.drop)

val daveId = 1L
val halId  = 2L

val setup = for {

  // 変更されたテーブルと新しいテーブルを作成します。
  _ <- (messages.schema ++ rooms.schema).create

  // 1つのルームを作成します。
  airLockId <- insertRoom += Room("Air Lock")

  // メッセージの半分はAir Lockルームにあります...
  _ <- insertMessages += Message(daveId, "Hello, HAL. Do you read me, HAL?", Some(airLockId))
  _ <- insertMessages += Message(halId, "Affirmative, Dave. I read you.",   Some(airLockId))

  // ...そして、半分はルームに含まれていません。
  _ <- insertMessages += Message(daveId, "Open the pod bay doors, HAL.")
  _ <- insertMessages += Message(halId, "I'm sorry, Dave. I'm afraid I can't do that.")

  // 最終結果を確認します。
  msgs <- messages.result
} yield (msgs)

exec(setup).foreach(println)
```

さて、これらのテーブルを横断して結合してみましょう。

### Inner Join

インナージョインは、複数のテーブルからデータを選択し、各テーブルの行がある方法で一致する場合に使用されます。
通常、一致は主キーを比較することで行われます。
一致しない行がある場合、結合結果には表示されません。

ユーザーテーブルに送信者があり、ルームテーブルにルームがあるメッセージを検索してみましょう。

```scala mdoc
val usersAndRooms =
  messages.
  join(users).on(_.senderId === _.id).
  join(rooms).on{ case ((msg,user), room) => msg.roomId === room.id }
```

`messages`テーブルと`users`テーブル、そして`messages`テーブルと`rooms`テーブルを結合します。
最初の`on`の呼び出しでは2項関数を使用し、2番目の呼び出しではパターンマッチング関数を使用して、2つのスタイルを説明します。

各結合はタプルのクエリを生成するため、連続した結合ではネストされたタプルが生成されます。
タプルを展開するための私たちの推奨される構文は、パターンマッチングです。
これにより、クエリの構造が明示的に明確になります。
ただし、このようなスタイルは、両方の結合に対して2項関数としてより簡潔に表現される場合もあります。

```scala mdoc
val usersAndRoomsBinaryFunction =
  messages.
  join(users).on(_.senderId  === _.id).
  join(rooms).on(_._1.roomId === _.id)
```

結果はどちらの方法でも同じです。

#### Mapping Joins

このクエリを以下のようにしてアクションにすることができます。

```scala mdoc
val usersAndRoomQuery: DBIO[Seq[((Message, User), Room)]] =
  usersAndRooms.result
```

結果にはネストされたタプルが含まれますが、通常はクエリを `map` して結果をフラットにし、必要な列を選択します。

テーブルのクラスではなく、メッセージ、送信者の名前、およびルームのタイトルなどの情報を取り出すことができます。

```scala mdoc
val usersAndRoomTitles =
  messages.
  join(users).on(_.senderId  === _.id).
  join(rooms).on { case ((msg,user), room) => msg.roomId === room.id }.
  map { case ((msg, user), room) => (msg.content, user.name, room.title) }

val action: DBIO[Seq[(String, String, String)]] = usersAndRoomTitles.result

exec(action).foreach(println)
```

#### Filter with Joins

結合はクエリなので、以前の章で学んだコンビネータを使用して変換することができます。すでに `map` コンビネータの例を見ました。別の例として `filter` メソッドがあります。

例えば、`usersAndRooms` クエリを使用して特定のルームに焦点を当てることができます。たとえば、Air Lock ルームでの結合を使用したい場合です。

```scala
// すでに見たクエリ...
val usersAndRooms =
  messages.
  join(users).on(_.senderId === _.id).
  join(rooms).on { case ((msg,user), room) => msg.roomId === room.id }
```

```scala mdoc
// ...修正して特定のルームに焦点を当てる
val airLockMsgs =
  usersAndRooms.
  filter { case (_, room) => room.title === "Air Lock" }
```

他のクエリと同様に、`filter` は SQL の `WHERE` 句となります。以下のような形式になります。

~~~ SQL
SELECT
  "message"."content", "user"."name", "room"."title"
FROM
  "message"
  INNER JOIN "user" ON "message"."sender" = "user"."id"
  INNER JOIN "room" ON "message"."room"   = "room"."id"
WHERE
  "room"."title" = 'Air Lock';
~~~

### Left Join

左結合（または左外部結合）は、さらに少し複雑です。
ここでは、1つのテーブルのすべてのレコードを選択し、もう1つのテーブルのレコードとのマッチングを行いますが、マッチングするレコードが存在する場合のみです。
左側にマッチングするレコードが見つからない場合、結果には `NULL` の値が含まれます。

チャットのスキーマを例にすると、メッセージはオプションでルームに属している場合があります。
すべてのメッセージとそれらが送信されたルームのリストを取得したいとしましょう。視覚的には、左外部結合は以下のようになります。

![
A visualization of the left outer join example. Selecting messages and associated rooms. For similar diagrams, see [A Visual Explanation of SQL Joins][link-visual-joins], _Coding Horror_, 11 Oct 2007.
](src/img/left-outer.png)

つまり、メッセージテーブルのすべてのデータを選択し、ルームに属しているメッセージの場合にはルームテーブルからのデータも取得します。

結合は次のようになります。

```scala mdoc
val left = messages.joinLeft(rooms).on(_.roomId === _.id)
```

このクエリ `left` は、メッセージを取得し、対応するルームをルームテーブルから参照します。すべてのメッセージがルームに属しているわけではないため、その場合は `roomId` 列は `NULL` になります。

Slick は、可能性のある `NULL` の値をより扱いやすい `Option` に変換します。`left` の完全な型は次のようになります。

```scala
Query[
  (MessageTable, Rep[Option[RoomTable]]),
  (MessageTable#TableElementType, Option[Room]),
  Seq]
```

このクエリの結果の型は `(Message, Option[Room])` であり、Slick が自動的に `Room` の側をオプショナルにしました。

もしメッセージの内容とルームのタイトルだけを抽出したい場合は、クエリに対して `map` を使用します。

```scala mdoc
val leftMapped =
  messages.
  joinLeft(rooms).on(_.roomId === _.id).
  map { case (msg, room) => (msg.content, room.map(_.title)) }
```

`room` 要素がオプションであるため、`Option.map` を使用して自然に `title` 要素を抽出します：`room.map(_.title)`。

このクエリの型は次のようになります。

```scala
Query[
  (Rep[String], Rep[Option[String]]),
  (String, Option[String]),
  Seq]
```

`String` と `Option[String]` の型は、メッセージの内容とルームのタイトルに対応しています。

```scala mdoc
exec(leftMapped.result).foreach(println)
```

### Right Join

前のセクションで、左結合は結合の左側からすべてのレコードを選択し、右側からは可能性のある `NULL` の値を持っています。

右結合（または右外部結合）はその逆の状況で、結合の右側からすべてのレコードを選択し、左側からは可能性のある `NULL` の値を持っています。

これを示すために、左結合の例を逆にしてみましょう。
すべてのルームと、それらが受信したプライベートメッセージを取得します。
バリエーションとして、今回は for 内包表記の構文を使用します。

```scala mdoc
val right = for {
  (msg, room) <- messages joinRight (rooms) on (_.roomId === _.id)
} yield (room.title, msg.map(_.content))
```

別のルームを作成して、クエリの結果を確認してみましょう。

```scala mdoc
exec(rooms += Room("Pod Bay"))

exec(right.result).foreach(println)
```

### Full Outer Join {#fullouterjoin}

完全外部結合（full outer join）では、どちらの側も `NULL` になる可能性があります。

スキーマからの例として、すべてのルームのタイトルとそのルーム内のメッセージが挙げられます。
メッセージはルームに属さなくてもよいし、ルームにメッセージがなくてもよいので、どちらの側も `NULL` になる可能性があります。

```scala mdoc
val outer = for {
  (room, msg) <- rooms joinFull messages on (_.id === _.roomId)
} yield (room.map(_.title), msg.map(_.content))
```

このクエリの型は両側にオプションを持っています。

```scala
Query[
  (Rep[Option[String]], Rep[Option[String]]),
  (Option[String], Option[String]),
  Seq]
```

結果からわかるように...

```scala mdoc
exec(outer.result).foreach(println)
```

...一部のルームには多くのメッセージがあり、一部にはメッセージがありません。
また、一部のメッセージにはルームがあり、一部にはありません。

<div class="callout callout-info">

執筆時点では、H2は完全外部結合をサポートしていません。
Slickの以前のバージョンではランタイムエクセプションが発生しましたが、Slick 3ではクエリを実行可能な形式にコンパイルし、完全外部結合をエミュレーションして実行します。

</div>


### Cross Joins

上記の例では、`join` を使用する場合には常に `on` 条件も使用してきましたが、これはオプションです。

もし、いずれかの `join`、`joinLeft`、`joinRight` の `on` 条件を省略すると、*クロスジョイン* が行われます。

クロスジョインは、左側のテーブルのすべての行を右側のテーブルのすべての行と結合します。
もし、最初のテーブルに10行、2番目のテーブルに5行あれば、クロスジョインは50行を生成します。

例:

```scala mdoc
val cross = messages joinLeft users
```

## Zip Joins

Zipジョインは、Scalaのコレクションにおける`zip`操作と同等です。
コレクションライブラリの`zip`は、2つのリストを操作し、ペアのリストを返します。

```scala mdoc
val xs = List(1, 2, 3)
xs zip xs.drop(1)
```

Slickでは、クエリに対して同等の`zip`メソッドと2つのバリエーションを提供しています。
隣接するメッセージをペアにするための例を見てみましょう。これを「会話」と呼びましょう。

```scala mdoc
// idでソートされたメッセージのコンテンツを選択します
val msgs = messages.sortBy(_.id.asc).map(_.content)

// 隣接するメッセージをペアにします
val conversations = msgs zip msgs.drop(1)
```

これはインナージョインに変換され、次のような出力を生成します。

```scala mdoc
exec(conversations.result).foreach(println)
```

2番目のバリエーションである`zipWith`では、ジョインとともにマッピング関数を提供することができます。ペアになった要素に対して変換を適用することができます。

```scala mdoc
def combiner(c1: Rep[String], c2: Rep[String]) =
  (c1.toUpperCase, c2.toLowerCase)

val query = msgs.zipWith(msgs.drop(1), combiner)

exec(query.result).foreach(println)
```

最後のバリエーションである`zipWithIndex`は、Scalaコレクションの同名のメソッドと同様です。メッセージに番号を付けましょう。

```scala mdoc
val withIndexQuery = messages.map(_.content).zipWithIndex

val withIndexAction: DBIO[Seq[(String, Long)]] =
  withIndexQuery.result
```

H2では、SQLの`ROWNUM()`関数が番号を生成するために使用されます。このクエリの結果は次のようになります。

```scala mdoc
exec(withIndexAction).foreach(println)
```

すべてのデータベースがzipジョインをサポートしているわけではありません。選択したデータベースプロファイルの`capabilities`フィールドで、`relational.zip`のサポートを確認してください。

```scala mdoc
// H2はzipをサポートしています
slick.jdbc.H2Profile.capabilities.
  map(_.toString).
  contains("relational.zip")
```

```scala mdoc
// SQLiteはzipをサポートしていません
slick.jdbc.SQLiteProfile.capabilities.
  map(_.toString).
  contains("relational.zip")
```

```scala mdoc:invisible
assert(false == slick.jdbc.SQLiteProfile.capabilities.map(_.toString).contains("relational.zip"), "SQLLite now supports ZIP!")
assert(true  == slick.jdbc.H2Profile.capabilities.map(_.toString).contains("relational.zip"), "H2 no longer supports ZIP?!")
```


## Joins Summary

In this chapter we've seen examples of the two different styles of join:
applicative and monadic.
We've also mixed and matched these styles.

We've seen how to construct the arguments to `on` methods,
either with a binary join condition
or by deconstructing a tuple with pattern matching.

Each join step produces a tuple.
Using pattern matching in `map` and `filter` allows us to clearly name each part of the tuple,
especially when the tuple is deeply nested.

We've also explored inner and outer joins, zip joins, and cross joins.
We saw that each type of join is a query,
making it compatible with combinators such as `map` and `filter`
from earlier chapters.


## Seen Any Strange Queries? {#scary}

If you've been following along and running the example joins,
you may have noticed large or unusual queries being generated.
Or you may not have. Since Slick 3.1, the SQL generated by Slick has improved greatly.

However, you may find the SQL generated a little strange or involved.
If Slick generates verbose queries are they are going to be slow?

Here's the key concept: the SQL generated by Slick is fed to the database optimizer.
That optimizer has far better knowledge
about your database, indexes, query paths, than anything else.
It will optimize the SQL from Slick into something that works well.

Unfortunately, some optimizers don't manage this very well.
Postgres does a good job. MySQL is, at the time of writing, pretty bad at this.
The trick here is to watch for slow queries,
and use your database's `EXPLAIN` command to examine and debug the query plan.

Optimisations can often be achieved by rewriting monadic joins in applicative style
and judiciously adding indices to the columns involved in joins.
However, a full discussion of query optimisation is out of the scope of this book.
See your database's documentation for more information.

If all else fails, we can rewrite queries for ultimate control
using Slick's _Plain SQL_ feature.
We will look at this in [Chapter 7](#PlainSQL).


## Aggregation

Aggregate functions are all about computing a single value from some set of rows.
A simple example is `count`.
This section looks at aggregation, and also at grouping rows, and computing values on those groups.

### Functions

Slick provides a few aggregate functions, as listed in the table below.

--------------------------------------------------------------------
Method           SQL
---------------  ---------------------------------------------------
 `length`        `COUNT(1)`

 `min`           `MIN(column)`

 `max`           `MAX(column)`

 `sum`           `SUM(column)`

 `avg`           `AVG(column)` --- mean of the column values
-----------      --------------------------------------------------

: A Selection of Aggregate Functions


Using them causes no great surprises, as shown in the following examples:

```scala mdoc
val numRows: DBIO[Int] = messages.length.result

val numDifferentSenders: DBIO[Int] =
  messages.map(_.senderId).distinct.length.result

val firstSent: DBIO[Option[Long]] =
  messages.map(_.id).min.result
```

While `length` returns an `Int`, the other functions return an `Option`.
This is because there may be no rows returned by the query, meaning there is no minimum, no maximum and so on.


### Grouping

Aggregate functions are often used with column grouping.
For example, how many messages has each user sent?
That's a grouping (by user) of an aggregate (count).

#### `groupBy`

Slick provides `groupBy` which will group rows by some expression. Here's an example:

```scala mdoc
val msgPerUser: DBIO[Seq[(Long, Int)]] =
  messages.groupBy(_.senderId).
  map { case (senderId, msgs) => senderId -> msgs.length }.
  result
```

A `groupBy` must be followed by a `map`.
The input to the `map` will be the grouping key (`senderId`) and a query for the group.

When we run the query, it'll work, but it will be in terms of a user's primary key:

```scala mdoc
exec(msgPerUser)
```

#### Groups and Joins

It'd be nicer to see the user's name. We can do that using our join skills:

```scala mdoc
val msgsPerUser =
   messages.join(users).on(_.senderId === _.id).
   groupBy { case (msg, user)   => user.name }.
   map     { case (name, group) => name -> group.length }.
   result
```

The results would be:

```scala mdoc
exec(msgsPerUser).foreach(println)
```

So what's happened here?
What `groupBy` has given us is a way to place rows into groups according to some function we supply.
In this example the function is to group rows based on the user's name.
It doesn't have to be a `String`, it could be any type in the table.

When it comes to mapping, we now have the key to the group (the user's name in our case),
and the corresponding group rows _as a query_.

Because we've joined messages and users, our group is a query of those two tables.
In this example we don't care what the query is because we're just counting the number of rows.
But sometimes we will need to know more about the query.


#### More Complicated Grouping

Let's look at a more involved example by collecting some statistics about our messages.
We want to find, for each user, how many messages they sent, and the id of their first message.
We want a result something like this:

```scala
Vector(
  (HAL,   2, Some(2)),
  (Dave,  2, Some(1)))
```

We have all the aggregate functions we need to do this:

```scala mdoc
val stats =
   messages.join(users).on(_.senderId === _.id).
   groupBy { case (msg, user) => user.name }.
   map     {
    case (name, group) =>
      (name, group.length, group.map{ case (msg, user) => msg.id}.min)
   }
```

We've now started to create a bit of a monster query.
We can simplify this, but before doing so, it may help to clarify that this query is equivalent to the following SQL:

```sql
select
  user.name, count(1), min(message.id)
from
  message inner join user on message.sender = user.id
group by
  user.name
```

Convince yourself the Slick and SQL queries are equivalent, by comparing:

* the `map` expression in the Slick query to the `SELECT` clause in the SQL;

* the `join` to the SQL `INNER JOIN`; and

* the `groupBy` to the SQL `GROUP` expression.

If you do that you'll see the Slick expression makes sense.
But when seeing these kinds of queries in code it may help to simplify by introducing intermediate functions with meaningful names.

There are a few ways to go at simplifying this,
but the lowest hanging fruit is that `min` expression inside the `map`.
The issue here is that the `group` pattern is a `Query` of `(MessageTable, UserTable)` as that's our join.
That leads to us having to split it further to access the message's ID field.

Let's pull that part out as a method:

```scala mdoc
import scala.language.higherKinds

def idOf[S[_]](group: Query[(MessageTable,UserTable), (Message,User), S]) =
    group.map { case (msg, user) => msg.id }
```

What we've done here is introduced a method to work on the group query,
using the knowledge of the `Query` type introduced in [The Query and TableQuery Types](#queryTypes) section of Chapter 2.

The query (`group`) is parameterized by thee things: the join, the unpacked values, and the container for the results.
By container we mean something like `Seq[T]`.
We don't really care what our results go into, but we do care we're working with messages and users.

With this little piece of domain specific language in place, the query becomes:

```scala mdoc
val nicerStats =
   messages.join(users).on(_.senderId === _.id).
   groupBy { case (msg, user)   => user.name }.
   map     { case (name, group) => (name, group.length, idOf(group).min) }

exec(nicerStats.result).foreach(println)
```

We think these small changes make code more maintainable and, quite frankly, less scary.
It may be marginal in this case, but real world queries can become large.
Your team mileage may vary, but if you see Slick queries that are hard to understand,
try pulling the query apart into named methods.


<div class="callout callout-info">
**Group By True**

There's a `groupBy { _ => true}` trick you can use where you want to select more than one aggregate from a query.

As an example, have a go at translating this SQL into a Slick query:

```sql
select min(id), max(id) from message where content like '%read%'
```

It's pretty easy to get either `min` or `max`:

```scala mdoc
messages.filter(_.content like "%read%").map(_.id).min
```

But you want both `min` and `max` in one query. This is where `groupBy { _ => true}` comes into play:

```scala mdoc
messages.
 filter(_.content like "%read%").
 groupBy(_ => true).
 map {
  case (_, msgs) => (msgs.map(_.id).min, msgs.map(_.id).max)
}
```

The effect of `_ => true` here is to group all rows into the same group!
This allows us to reuse the `msgs` query, and obtain the result we want.
</div>

#### Grouping by Multiple Columns

The result of `groupBy` doesn't need to be a single value: it can be a tuple.  This gives us access to grouping by multiple columns.

We can look at the number of messages per user per room.  Something like this:

```scala
Vector(
  (Air Lock, HAL,   1),
  (Air Lock, Dave,  1),
  (Kitchen,  Frank, 3) )
```

...assuming we add a message from Frank:

```scala mdoc
val addFrank = for {
  kitchenId <- insertRoom += Room("Kitchen")
  frankId   <- insertUser += User("Frank")
  rowsAdded <- messages ++= Seq(
    Message(frankId, "Hello?",    Some(kitchenId)),
    Message(frankId, "Helloooo?", Some(kitchenId)),
    Message(frankId, "HELLO!?",   Some(kitchenId))
  )
} yield rowsAdded

exec(addFrank)
```

To run the report we're going to need to group by room and then by user, and finally count the number of rows in each group:

```scala mdoc
val msgsPerRoomPerUser =
   rooms.
   join(messages).on(_.id === _.roomId).
   join(users).on{ case ((room,msg), user) => user.id === msg.senderId }.
   groupBy { case ((room,msg), user)   => (room.title, user.name) }.
   map     { case ((room,user), group) => (room, user, group.length) }.
   sortBy  { case (room, user, group)  => room }
```

Hopefully you're now in a position where you can unpick this:

* We join on messages, room and user to be able to display the room title and user name.

* The value passed into the `groupBy` will be determined by the join.

* The result of the `groupBy` is the columns for the grouping, which is a tuple of the room title and the user's name.

* We select (`map`) just the columns we want: room, user and the number of rows.

* For fun we've thrown in a `sortBy` to get the results in room order.

Running the action produces our expected report:

```scala mdoc
exec(msgsPerRoomPerUser.result).foreach(println)
```


## Take Home Points

Slick supports `join`, `joinLeft`, `joinRight`, `joinOuter` and a `zip` join. You can map and filter over these queries as you would other queries with Slick.  Using pattern matching on the query tuples can be more readable than accessing tuples via `._1`, `._2` and so on.

Aggregation methods, such as `length` and `sum`, produce a value from a set of rows.

Rows can be grouped based on an expression supplied to `groupBy`. The result of a grouping expression is a group key and a query defining the group. Use `map`, `filter`, `sortBy` as you would with any query in Slick.

The SQL produced by Slick might not be the SQL you would write.
Slick expects the database query engine to perform optimisation. If you find slow queries, take a look at _Plain SQL_, discussed in the next chapter.



## Exercises

Because these exercises are all about multiple tables, take a moment to remind yourself of the schema.
You'll find this in the example code, `chatper-06`, in the source file `chat_schema.scala`.

### Name of the Sender

Each message is sent by someone.
That is, the `messages.senderId` will have a matching row via `users.id`.

Please...

- Write a monadic join to return all `Message` rows and the associated `User` record for each of them.

- Change your answer to return only the content of a message and the name of the sender.

- Modify the query to return the results in name order.

- Re-write the query as an applicative join.

These exercises will get your fingers familiar with writing joins.

<div class="solution">
These queries are all items we've covered in the text:

```scala mdoc
val ex1 = for {
  m <- messages
  u <- users
  if u.id === m.senderId
} yield (m, u)

val ex2 = for {
  m <- messages
  u <- users
  if u.id === m.senderId
} yield (m.content, u.name)

val ex3 = ex2.sortBy{ case (content, name) => name }

val ex4 =
  messages.
   join(users).on(_.senderId === _.id).
   map    { case (msg, usr)     => (msg.content, usr.name) }.
   sortBy { case (content,name) => name }
```
</div>

### Messages of the Sender

Write a method to fetch all the message sent by a particular user.
The signature is:

```scala
def findByName(name: String): Query[Rep[Message], Message, Seq] = ???
```

<div class="solution">
This is a filter, a join, and a map:

```scala mdoc
def findByNameMonadic(name: String): Query[Rep[Message], Message, Seq] = for {
  u <- users    if u.name === name
  m <- messages if m.senderId === u.id
} yield m
```

...or...

```scala mdoc
def findByNameApplicative(name: String): Query[Rep[Message], Message, Seq] =
  users.filter(_.name === name).
  join(messages).on(_.id === _.senderId).
  map{ case (user, msg) => msg }
```
</div>


### Having Many Messages

Modify the `msgsPerUser` query...

```scala
val msgsPerUser =
   messages.join(users).on(_.senderId === _.id).
   groupBy { case (msg, user)  => user.name }.
   map     { case (name, group) => name -> group.length }
```

...to return the counts for just those users with more than 2 messages.

<div class="solution">
SQL distinguishes between `WHERE` and `HAVING`. In Slick you use `filter` for both:

```scala mdoc
val modifiedMsgsPerUser =
   messages.join(users).on(_.senderId === _.id).
   groupBy { case (msg, user)  => user.name }.
   map     { case (name, group) => name -> group.length }.
   filter  { case (name, count) => count > 2 }
```

At this point in the book, only Frank has more than two messages:

```scala mdoc
exec(modifiedMsgsPerUser.result)
```

```scala mdoc
// Let's check:
val frankMsgs = 
  messages.join(users).on {
    case (msg,user) => msg.senderId === user.id && user.name === "Frank" 
  }

exec(frankMsgs.result).foreach(println)
```

...although if you've been experimenting with the database, your results could be different.
</div>

### Collecting Results

A join on messages and senders will produce a row for every message.
Each row will be a tuple of the user and message:

```scala
users.join(messages).on(_.id === _.senderId)
// res1: slick.lifted.Query[
//  (UserTable, MessageTable),
//  (UserTable#TableElementType, MessageTable#TableElementType),
//  Seq] = Rep(Join Inner)
```

The return type is effectively `Seq[(User, Message)]`.

Sometimes you'll really want something like a `Map[User, Seq[Message]]`.

There's no built-in way to do that in Slick, but you can do it in Scala using the collections `groupBy` method.

```scala mdoc
val almost = Seq(
  ("HAL"  -> "Hello"),
  ("Dave" -> "How are you?"),
  ("HAL"  -> "I have terrible pain in all the diodes")
  ).groupBy{ case (name, message) => name }
```

That's close, but the values in the map are still a tuple of the name and the message.
We can go further and reduce this to:

```scala mdoc:silent
val correct = almost.view.mapValues { values =>
  values.map{ case (name, msg) => msg }
}
```
```scala mdoc
correct.foreach(println)
```

The `.view` call is required in Scala 2.13 to convert the lazy evaluated map into a strict map.
A future version of Scala will remove the need for the `.view` call.

Go ahead and write a method to encapsulate this for a join:

```scala
def userMessages: DBIO[Map[User, Seq[Message]]] = ???
```

<div class="solution">
You need all the code in the question and also what you know about action combinators:

```scala mdoc
def userMessages: DBIO[Map[User,Seq[Message]]] =
  users.join(messages).on(_.id === _.senderId).result.
  map { rows => rows
    .groupBy{ case (user, message) => user }
    .view
    .mapValues(values => values.map{ case (name, msg) => msg })
    .toMap
  }

exec(userMessages).foreach(println)
```

You may have been tripped up on the call to `toMap` at the end.
We didn't need this in the examples in the text because we were not being explicit that we wanted a `Map[User,Seq[Message]]`.
However, `userMessages` does define the result type, and as such we need to explicitly covert the sequence of tuples into a `Map`.

</div>

