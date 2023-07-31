```scala mdoc:invisible
import slick.jdbc.H2Profile.api._
import scala.concurrent.{Await,Future}
import scala.concurrent.duration._
import scala.concurrent.ExecutionContext.Implicits.global

val db = Database.forConfig("chapter07")

def exec[T](action: DBIO[T]): T = Await.result(db.run(action), 4.seconds)
```
# Plain SQL {#PlainSQL}

Slickは、これまで見てきたリフテッド埋め込みスタイルに加えて、プレーンSQLクエリもサポートしています。プレーンクエリはリフテッドクエリほどきれいに合成されず、同じような型の安全性も提供しませんが、必要に応じて本質的に任意のSQLを実行できるようにします。もしSlickによって生成された特定のクエリに不満がある場合は、プレーンSQLに移行することが適しています。

このセクションでは、次のことを見ていきます：

- [補間子][link-scala-interpolation] `sql` (選択のためのもの) と `sqlu` (更新のためのもの) を使用して、プレーンSQLクエリを作成する方法

- `${expresson}` の構文を使って、クエリに値を安全に挿入する方法

- カスタム型をプレーンSQLで使用する方法（ただし、スコープ内にコンバータがある場合）

- データベース上でコンパイル時にクエリの構文と型をチェックするために `tsql` 補間子を使用する方法

<div class="callout callout-info">
**作業用のテーブル**

以下に続く例のために、Rooms用のテーブルを設定します。
今のところ、これをリフテッド埋め込みスタイルを使用して他の章と同様に行います：

```scala mdoc
case class Room(title: String, id: Long = 0L)

class RoomTable(tag: Tag) extends Table[Room](tag, "room") {
 def id    = column[Long]("id", O.PrimaryKey, O.AutoInc)
 def title = column[String]("title")
 def * = (title, id).mapTo[Room]
}

lazy val rooms = TableQuery[RoomTable]

val roomSetup = DBIO.seq(
  rooms.schema.create,
  rooms ++= Seq(Room("Air Lock"), Room("Pod"), Room("Brain Room"))
)

val setupResult = exec(roomSetup)
```
</div>


## Selects

それでは、シンプルな例としてルームのIDのリストを返す方法から始めましょう。

```scala mdoc
val action = sql""" select "id" from "room" """.as[Long]

Await.result(db.run(action), 2.seconds)
```

プレーンSQLクエリを実行する方法は、これまでに見てきた他のクエリと似ています：通常通り`db.run`を呼び出します。

大きな違いは、クエリの構築方法にあります。実行したいSQLと、期待される結果の型を`as[T]`を使って指定します。
そして、返される結果は`Query`ではなく実行するためのアクションになります。

`as[T]`メソッドはかなり柔軟です。ルームのIDとルームのタイトルを返すこともできます：

```scala mdoc
val roomInfo = sql""" select "id", "title" from "room" """.as[(Long,String)]

exec(roomInfo)
```

結果の型として`(Long, String)`のタプルを指定していることに注意してください。これはSQLの`SELECT`ステートメントのカラムと一致しています。

`as[T]`を使って任意の結果型を構築することができます。後で自分のアプリケーションのケースクラスを使う方法も見ていきます。

SQL補間子の最も便利な機能の一つは、クエリ内でScalaの値を参照できることです：

```scala mdoc
val roomName = "Pod"

val podRoomAction = sql"""
  select
    "id", "title"
  from
    "room"
  where
    "title" = $roomName """.as[(Long,String)].headOption

exec(podRoomAction)
```

`$roomName`がScalaの値`roomName`を参照するために使用されていることに注意してください。
この値はクエリに安全に組み込まれます。
つまり、SQL補間子をこのように使用する際には、SQLインジェクション攻撃を心配する必要がありません。

<div class="callout callout-warning">

**文字列の危険性**

SQL補間子は、実行するSQLに完全な制御を必要とする状況に欠かせません。ただし、コンパイル時の安全性にはいくつかの欠点があります。例えば：

```scala mdoc
val t = 42

val badAction =
  sql""" select "id" from "room" where "title" = $t """.as[Long]
```

このコードはコンパイルはされますが、実行時にエラーとなります。なぜなら、`title`カラムの型が`String`であり、私たちが`Int`を提供したからです。

```scala mdoc
exec(badAction.asTry)
```

リフテッド埋め込みスタイルを使用した等価なクエリならば、コンパイル時に問題が検出されました。
この章の後に説明する`tsql`補間子は、クエリと型をコンパイル時にデータベースに接続してチェックすることで、この問題を解決します。

もう一つの危険は`#$`スタイルの代入です。これは_スプライシング_と呼ばれ、SQLエスケープを適用したくない場合に使用します。例えば、使用するテーブルの名前が変わる可能性があるかもしれません：

```scala mdoc
val table = "room"
val splicedAction = sql""" select "id" from "#$table" """.as[Long]
```

このような場合には、`table`の値を`String`として扱いたくありません。もし扱ってしまうと、無効なクエリになります：`select "id" from "'message'"`（テーブル名の周りに二重引用符とシングルクオートがある点に注意してください）。これは有効なSQLではありません。

そのため、スプライシングを使って安全でないSQLを生成する可能性があります。この際の黄金のルールは、ユーザーから提供された入力に対して`#$`を絶対に使用しないことです。

確実に覚えておきましょう：ユーザーから提供された入力に対して`#$`を絶対に使用しないこと。
</div>



### Select with Custom Types

Slickはデフォルトで多くのデータ型をSQLデータ型と相互変換することができます。これまで見てきた例では、Scalaの`String`をSQLの文字列に変換したり、SQLのBIGINTをScalaの`Long`に変換したりすることがありました。これらの変換は`as[T]`を使って利用できます。

もしSlickが認識していない型を扱いたい場合は、変換を提供する必要があります。その役割を果たすのが`GetResult`型クラスです。

例として、メッセージのテーブルをいくつかの興味深い構造を持つもので設定してみましょう：

```scala mdoc
import org.joda.time.DateTime

case class Message(
  sender  : String,
  content : String,
  created : DateTime,
  updated : Option[DateTime],
  id      : Long = 0L
)
```

この段階で注目すべきは、`created`という型が`DateTime`であるということです。
これはJoda Timeのもので、Slickはこれに対する組み込みサポートを持っていません。

これが実行したいクエリです：

```scala mdoc:fail
sql""" select "created" from "message" """.as[DateTime]
```

これはコンパイルできません。なぜならSlickが`DateTime`について何も知らないからです。
これがコンパイルするようにするには、`GetResult[DateTime]`のインスタンスを提供する必要があります：

```scala mdoc:silent
import slick.jdbc.GetResult
import java.sql.Timestamp
import org.joda.time.DateTimeZone.UTC
```
```scala mdoc
implicit val GetDateTime =
  GetResult[DateTime](r => new DateTime(r.nextTimestamp(), UTC))
```

`GetResult`は、`r`（`PositionedResult`）から`DateTime`への関数を包んでいます。`PositionedResult`はデータベースの値にアクセスできるようにします（`nextTimestamp`、`nextLong`、`nextBigDecimal`など）。`nextTimestamp`から取得した値を`DateTime`のコンストラクタに渡しています。

この値の名前は重要ではありません。
重要なのは、この値が暗黙的であり、型が`GetResult[DateTime]`であることです。
これにより、`DateTime`を言及する際にコンパイラが変換関数を検索できるようになります。

これでアクションを作成できます：

```scala mdoc
sql""" select "created" from "message" """.as[DateTime]
```


### Case Classes

おそらくお気づきかと思いますが、Plain SQLクエリからcaseクラスを返す場合は、caseクラスに対して`GetResult`を提供する必要があります。メッセージテーブルの例で進めてみましょう。

メッセージには、ID、コンテンツ、送信者ID、タイムスタンプ、オプションのタイムスタンプが含まれることを思い出してください。

`GetResult[Message]`を提供するには、`Message`内のすべての型が`GetResult`インスタンスを持っている必要があります。
既に`DateTime`に対処しました。
そして、Slickは`Long`と`String`を処理する方法を知っています。
したがって、`Option[DateTime]`と`Message`自体が残ります。

オプションの値については、Slickは`nextXXXOption`メソッド（例：`nextLongOption`）を提供しています。
オプションの日時に対しては、`nextTimestampOption()`を使用してデータベースの値を読み取り、その後に`map`で適切な型に変換します：

```scala mdoc
implicit val GetOptionalDateTime = GetResult[Option[DateTime]](r =>
  r.nextTimestampOption().map(ts => new DateTime(ts, UTC))
)
```

個々のカラムをマップした後、`Message`の`GetResult`でそれらをまとめることができます。
これを構築する際に役立つ2つのヘルパーメソッドがあります：

- `<<`：適切な_nextXXX_メソッドを呼び出すためのもの

- `<<?`：値がオプションの場合に使用します

これを以下のように使用できます：

```scala mdoc
implicit val GetMessage = GetResult(r =>
   Message(sender  = r.<<,
           content = r.<<,
           created = r.<<,
           updated = r.<<?,
           id      = r.<<)
 )
```

これは、caseクラスのコンポーネントに対して暗黙の値を提供したために動作します。
フィールドの型がわかっているため、`<<`と`<<?`は各型の暗黙の`GetResult[T]`を使用できます。

これで`Message`の値を選択できます：

```scala mdoc
val messageAction: DBIO[Seq[Message]] =
  sql""" select * from "message" """.as[Message]
```

おそらくこの特定の例では、Plain SQLよりもlifted embedded styleの方が好ましいでしょう。
ただし、パフォーマンスの理由でPlain SQLを使用する場合があるかもしれないので、データベースの値を有意義なドメイン型に変換する方法を知っていると便利です。


<div class="callout callout-warning">

**`SELECT *`について**

この章ではコード例をページに収めるために`SELECT *`を使用することがあります。
ただし、コードベースではこれを避けることをお勧めします。なぜなら、テーブル外のSlick以外の場所で列が追加されると、クエリの結果が予期しない変更になります。結果のマッピングが失敗するかもしれません。

</div>

## Updates

[Chapter 3](#UpdatingRows) では、`update`メソッドを使用して行を変更する方法を見てきました。
行の現在の値を使用したい場合、バッチ更新は難しいということに気づきました。
使用した例は、メッセージのコンテンツに感嘆符を追加するものでした：

```sql
UPDATE "message" SET "content" = CONCAT("content", '!')
```

Plain SQLの更新クエリを使うと、これを行うことができます。対応するインターポレータは`sqlu`です：

```scala mdoc
val updateAction =
  sqlu"""UPDATE "message" SET "content" = CONCAT("content", '!')"""
```

`updateAction`は他のアクションと同様に構築されており、`db.run`で評価するまで実行されません。しかし実行される際には、各行の値に感嘆符が追加されます。これは、lifted embedded styleでは効率的に行えなかったことです。

`sql`インターポレータと同様に、変数にバインドするために`$`を使用することもできます：

```scala mdoc
val char = "!"
val interpolatorAction =
  sqlu"""UPDATE "message" SET "content" = CONCAT("content", $char)"""
```

これにより2つの利点が得られます：コンパイラは変数名のスペルミスを指摘してくれますし、[SQLインジェクション攻撃][link-wikipedia-injection]から入力が保護されます。

この場合、Slickが生成したステートメントは以下のようになります：

```scala mdoc
interpolatorAction.statements.head
```

### Updating with Custom Types

基本的な型（`String`や`Int`など）と一緒に作業することは問題ありませんが、時々より複雑な型を使用して更新したいことがあります。セレクトの結果をマッピングするための`GetResult`型クラスを見ましたが、更新に対してもこれに対応する`SetParameter`型クラスが存在します。

`DateTime`パラメータを設定する方法をSlickに教えることができます：

```scala mdoc
import slick.jdbc.SetParameter

implicit val SetDateTime = SetParameter[DateTime](
  (dt, pp) => pp.setTimestamp(new Timestamp(dt.getMillis))
 )
```

ここでの`pp`は`PositionedParameters`です。これはSlickの実装の詳細であり、SQLステートメントと値のプレースホルダーをラップしています。実質的には、更新ステートメントのどこに現れても`DateTime`をどのように扱うかを指定しています。

`Timestamp`（`setTimestamp`を介して）以外にも、`Boolean`、`Byte`、`Short`、`Int`、`Long`、`Float`、`Double`、`BigDecimal`、`Array[Byte]`、`Blob`、`Clob`、`Date`、`Time`、そして`Object`と`null`を設定することができます。`PositionedParameters`には`Option`型に対しても_setXXX_メソッドがあります。

`SetParameter`の定義でも`GetResult`との対称性があります。実際、`SetParameter`でも`>>`を使用することができます：

```scala mdoc:nest
implicit val SetDateTime = SetParameter[DateTime](
  (dt, pp) => pp >> new Timestamp(dt.getMillis))
```

これを実装することで、`DateTime`インスタンスを使用してPlain SQLの更新クエリを作成できます：

```scala mdoc
val now =
  sqlu"""UPDATE "message" SET "created" = ${DateTime.now}"""
```

`SetParameter[DateTime]`インスタンスがない場合、コンパイラは次のようなエラーを表示します：

```scala
could not find implicit SetParameter[DateTime]
```



## Typed Checked Plain SQL

We've mentioned the risks of Plain SQL, which can be summarized as not discovering a problem with your query until runtime.  The `tsql` interpolator removes some of this risk, but at the cost of requiring a connection to a database at compile time.

<div class="callout callout-info">
**Run the Code**

These examples won't run in the REPL.
To try these out, use the `tsql.scala` file inside the `chapter-07` folder.
This is all in the [example code base on GitHub][link-example].
</div>

### Compile Time Database Connections

To get started with `tsql` we provide a database configuration information on a class:

```scala
import slick.backend.StaticDatabaseConfig

@StaticDatabaseConfig("file:src/main/resources/application.conf#tsql")
object TsqlExample {
  // queries go here
}
```

The `@StaticDatabaseConfig` syntax is called an _annotation_. This particular `StaticDatabaseConfig` annotation is telling Slick to use the connection called "tsql" in our configuration file.  That entry will look like this:

```scala
tsql {
  profile = "slick.jdbc.H2Profile$"
  db {
    connectionPool = disabled
    url = "jdbc:h2:mem:chapter06; INIT=
       runscript from 'src/main/resources/integration-schema.sql'"
    driver = "org.h2.Driver"
    keepAliveConnection = false
  }
}
```

Note the `$` in the profile class name is not a typo. The class name is being passed to Java's `Class.forName`, but of course Java doesn't have a singleton as such. The Slick configuration does the right thing to load `$MODULE` when it sees `$`. This interoperability with Java is described in [Chapter 29 of _Programming in Scala_][link-pins-interop].

You won't have seen this when we introduced the database configuration in Chapter 1. That's because this `tsql` configuration has a different format, and combines the Slick profile (`slick.jdbc.H2Profile`) and the JDBC driver (`org.h2.Drvier`) in one entry.

A consequence of supplying a `@StaticDatabaseConfig` is that you can define one databases configuration for your application and a different one for the compiler to use. That is, perhaps you are running an application, or test suite, against an in-memory database, but validating the queries at compile time against a full-populated production-like integration database.

In the example above, and the accompanying example code, we use an in-memory database to make Slick easy to get started with.  However, an in-memory database is empty by default, and that would be no use for checking queries against. To work around that we provide an `INIT` script to populate the in-memory database.
For our purposes, the `integration-schema.sql` file only needs to contain one line:

```sql
create table "message" (
  "content" VARCHAR NOT NULL,
  "id"      BIGSERIAL NOT NULL PRIMARY KEY
);
```


### Type Checked Plain SQL

With the `@StaticDatabaseConfig` in place we can use `tsql`:

```scala
val action: DBIO[Seq[String]] = tsql""" select "content" from "message" """
```

You can run that query as you would `sql` or `sqlu` query.
You can also use custom types via `SetParameter` type class. However, `GetResult` type classes are not supported for `tsql`.

Let's get the query wrong and see what happens:

```scala
val action: DBIO[Seq[String]] =
  tsql"""select "content", "id" from "message""""
```

Do you see what's wrong? If not, don't worry because the compiler will find the problem:

```scala
type mismatch;
[error]  found    : SqlStreamingAction[
                        Vector[(String, Int)],
                        (String, Int),Effect ]
[error]  required : DBIO[Seq[String]]
```

The compiler wants a `String` for each row, because that's what we've declared the result to be.
However it has found, via the database, that the query will return `(String,Int)` rows.

If we had omitted the type declaration, the action would have the inferred type of `DBIO[Seq[(String,Int)]]`.
So if you want to catch these kinds of mismatches, it's good practice to declare the type you expect when using `tsql`.

Let's see other kinds of errors the compiler will find.

How about if the SQL is just wrong:

```scala
val action: DBIO[Seq[String]] =
  tsql"""select "content" from "message" where"""
```

This is incomplete SQL, and the compiler tells us:

```scala
exception during macro expansion: ERROR: syntax error at end of input
[error]   Position: 38
[error]     tsql"""select "content" from "message" WHERE"""
[error]     ^
```

And if we get a column name wrong...

```scala
val action: DBIO[Seq[String]] =
  tsql"""select "text" from "message" where"""
```

...that's also a compile error too:

```scala
Exception during macro expansion: ERROR: column "text" does not exist
[error]   Position: 8
[error]     tsql"""select "text" from "message""""
[error]     ^
```

Of course, in addition to selecting rows, you can insert:

```scala
val greeting = "Hello"
val action: DBIO[Seq[Int]] =
  tsql"""insert into "message" ("content") values ($greeting)"""
```

Note that at run time, when we execute the query, a new row will be inserted.
At compile time, Slick uses a facility in JDBC to compile the query and retrieve the meta data without having to run the query. 
In other words, at compile time the database is not mutated.


## Take Home Points

Plain SQL allows you a way out of any limitations you find with Slick's lifted embedded style of querying.

Two main string interpolators for SQL are provided: `sql` and `sqlu`:

- Values can be safely substituted into Plain SQL queries using `${expression}`.

- Custom types can be used with the interpolators providing an implicit `GetResult` (select) or `SetParameter` (update) is in scope for the type.

- Raw values can be spliced into a query with `#$`. Use this with care: end-user supplied information should never be spliced into a query.

The `tsql` interpolator will check Plain SQL queries against a database at compile time.  The database connection is used to validate the query syntax, and also discover the types of the columns being selected. To make best use of this, always declare the type of the query you expect from `tsql`.


## Exercises

For these exercises we will use a  combination of messages and users.
We'll set this up using the lifted embedded style:

```scala mdoc:reset:invisible
import slick.jdbc.H2Profile.api._
import scala.concurrent.{Await,Future}
import scala.concurrent.duration._
import scala.concurrent.ExecutionContext.Implicits.global

val db = Database.forConfig("chapter07")

def exec[T](action: DBIO[T]): T = Await.result(db.run(action), 4.seconds)
```

```scala mdoc:silent
case class User(
  name  : String,
  email : Option[String] = None,
  id    : Long = 0L
)

class UserTable(tag: Tag) extends Table[User](tag, "user") {
 def id    = column[Long]("id", O.PrimaryKey, O.AutoInc)
 def name  = column[String]("name")
 def email = column[Option[String]]("email")
 def * = (name, email, id).mapTo[User]
}

lazy val users = TableQuery[UserTable]
lazy val insertUsers = users returning users.map(_.id)

case class Message(senderId: Long, content: String, id: Long = 0L)

class MessageTable(tag: Tag) extends Table[Message](tag, "message") {
 def id       = column[Long]("id", O.PrimaryKey, O.AutoInc)
 def senderId = column[Long]("sender_id")
 def content  = column[String]("content")
 def * = (senderId, content, id).mapTo[Message]
}

lazy val messages = TableQuery[MessageTable]

val setup = for {
   _ <- (users.schema ++ messages.schema).create
   daveId <- insertUsers += User("Dave")
   halId  <- insertUsers += User("HAL")
   rowsAdded <- messages ++= Seq(
    Message(daveId, "Hello, HAL. Do you read me, HAL?"),
    Message(halId,  "Affirmative, Dave. I read you."),
    Message(daveId, "Open the pod bay doors, HAL."),
    Message(halId,  "I'm sorry, Dave. I'm afraid I can't do that.")
   )
} yield rowsAdded

exec(setup)
```

### Plain Selects

Let's get warmed up with some simple exercises.

Write the following four queries as Plain SQL queries:

- Count the number of rows in the message table.

- Select the content from the messages table.

- Select the length of each message ("content") in the messages table.

- Select the content and length of each message.

Tips:

- Remember that you need to use double quotes around table and column names in the SQL.

- We gave the database tables names which are singular: `message`, `user`, etc.

<div class="solution">
The SQL statements are relatively simple. You need to take care to make the `as[T]` align to the result of the query.

```scala mdoc
val q1 = sql""" select count(*) from "message" """.as[Int]
val a1 = exec(q1)

val q2 = sql""" select "content" from "message" """.as[String]
val a2 = exec(q2)
a2.foreach(println)

val q3 = sql""" select length("content") from "message" """.as[Int]
val a3 = exec(q3)

val q4 = sql""" select "content", length("content") from "message" """.as[(String,Int)]
val a4 = exec(q4)
a4.foreach(println)
```

```scala mdoc:invisible
assert(a1.head == 4, s"Expected 4 results for a1, not $a1")
assert(a2.length == 4, s"Expected 4 results for a2, not $a2")
assert(a3 == Seq(32,30,28,44), s"Expected specific lenghts, not $a3")
assert(a4.length == 4, s"Expected 4 results for a4, not $a4")
```
</div>

### Conversion

Convert the following lifted embedded query to a Plain SQL query.

```scala mdoc
val whoSaidThat =
  messages.join(users).on(_.senderId === _.id).
  filter{ case (message,user) =>
    message.content === "Open the pod bay doors, HAL."}.
  map{ case (message,user) => user.name }

exec(whoSaidThat.result)
```

Tips:

- If you're not familiar with SQL syntax, peak at the statement generated for `whoSaidThat` given above.

- Remember that strings in SQL are wrapped in single quotes, not double quotes.

- In the database, the sender's ID is `sender_id`.


<div class="solution">
There are various ways to implement this query in SQL.  Here's one of them...

```scala mdoc
val whoSaidThatPlain = sql"""
  select
    "name" from "user" u
  join
    "message" m on u."id" = m."sender_id"
  where
    m."content" = 'Open the pod bay doors, HAL.'
  """.as[String]

exec(whoSaidThatPlain)
```
</div>


### Substitution

Complete the implementation of this method using a Plain SQL query:

```scala
def whoSaid(content: String): DBIO[Seq[String]] =
  ???
```

Running `whoSaid("Open the pod bay doors, HAL.")` should return a list of the people who said that. Which should be Dave.

This should be a small change to your solution to the last exercise.

<div class="solution">
The solution requires the use of a `$` substitution:

```scala mdoc
def whoSaid(content: String): DBIO[Seq[String]] =
  sql"""
    select
      "name" from "user" u
    join
      "message" m on u."id" = m."sender_id"
    where
      m."content" = $content
    """.as[String]

exec(whoSaid("Open the pod bay doors, HAL."))

exec(whoSaid("Affirmative, Dave. I read you."))
```
</div>


### First and Last

This H2 query returns the alphabetically first and last messages:

```scala mdoc
exec(sql"""
  select min("content"), max("content")
  from "message" """.as[(String,String)]
)
```

In this exercise we want you to write a `GetResult` type class instance so that the result of the query is one of these:

```scala mdoc:silent
case class FirstAndLast(first: String, last: String)
```

The steps are:

1. Remember to `import slick.jdbc.GetResult`.

2. Provide an implicit value for `GetResult[FirstAndLast]`

3. Make the query use `as[FirstAndLast]`

<div class="solution">
```scala mdoc
import slick.jdbc.GetResult

implicit val GetFirstAndLast =
  GetResult[FirstAndLast](r => FirstAndLast(r.nextString(), r.nextString()))


val query =  sql""" select min("content"), max("content")
                    from "message" """.as[FirstAndLast]

exec(query)
```
</div>


### Plain Change

We can use Plain SQL to modify the database.
That means inserting rows, updating rows, deleting rows, and also modifying the schema.

Go ahead and create a new table, using Plain SQL, to store the crew's jukebox playlist.
Just store a song title. Insert a row into the table.

<div class="solution">
For modifications we use `sqlu`, not `sql`:

```scala mdoc
exec(sqlu""" create table "jukebox" ("title" text) """)

exec(sqlu""" insert into "jukebox"("title")
             values ('Bicycle Built for Two') """)

exec(sql""" select "title" from "jukebox" """.as[String])
```
</div>


### Robert Tables

We're building a web site that allows searching for users by their email address:

```scala mdoc
def lookup(email: String) =
  sql"""select "id" from "user" where "email" = '#${email}'"""

// Example use:
exec(lookup("dave@example.org").as[Long].headOption)
```

What the problem with this code?

<div class="solution">
If you are familiar with [xkcd's Little Bobby Tables](http://xkcd.com/327/),
the title of the exercise has probably tipped you off:  `#$` does not escape input.

This means a user could use a carefully crafted email address to do evil:

```scala mdoc
val evilAction = lookup("""';DROP TABLE "user";--- """).as[Long]
exec(evilAction)
```

This "email address" turns into two queries:

~~~ sql
SELECT * FROM "user" WHERE "email" = '';
~~~

and

~~~ sql
DROP TABLE "user";
~~~

Trying to access the users table after this will produce:

```scala mdoc
exec(users.result.asTry)
```

Yes, the table was dropped by the query.

Never use `#$` with user supplied input.
</div>
