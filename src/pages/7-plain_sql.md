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

リスクについて述べた通り、Plain SQLのリスクはクエリの問題がランタイムまでわからないということです。`tsql`インターポレータは、このリスクの一部を取り除きますが、コンパイル時にデータベースへの接続が必要です。

<div class="callout callout-info">

**コードの実行**

これらの例はREPLで実行することはできません。
これらを試すには、`chapter-07`フォルダ内の`tsql.scala`ファイルを使用してください。
これはすべて、[GitHub上のサンプルコードベース][link-example]にあります。
</div>


### Compile Time Database Connections

`tsql`を始めるには、クラスにデータベース設定情報を提供します：

```scala
import slick.backend.StaticDatabaseConfig

@StaticDatabaseConfig("file:src/main/resources/application.conf#tsql")
object TsqlExample {
  // queries go here
}
```

`@StaticDatabaseConfig`構文はアノテーションと呼ばれます。この特定の`StaticDatabaseConfig`アノテーションは、Slickに対して設定ファイル内の「tsql」という名前の接続を使用するように指示しています。そのエントリは次のようになります：


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

プロファイルクラス名内の`$`はタイポではありません。クラス名はJavaの`Class.forName`に渡されますが、もちろんJavaにはそのようなシングルトンは存在しません。クラス名が`$`を見ると、Slickの設定は`$`を見ると`$MODULE`をロードする適切な方法を取ります。このJavaとの互換性は[「Programming in Scala」の第29章][link-pins-interop]で説明されています。

これはChapter 1でデータベース設定を紹介したときに見たことはありません。これは「tsql」設定が異なる形式を持っており、Slickプロファイル（`slick.jdbc.H2Profile`）とJDBCドライバ（`org.h2.Driver`）を1つのエントリで組み合わせています。

`@StaticDatabaseConfig`を提供することの結果として、アプリケーション用の1つのデータベース設定と、コンパイラが使用する別のデータベース設定を定義できます。つまり、おそらくアプリケーションまたはテストスイートを、インメモリデータベースで実行するかもしれませんが、コンパイル時にはフルにポピュレートされたプロダクションのような統合データベースに対してクエリを検証するかもしれません。

上記の例と関連するサンプルコードでは、Slickを使い始めやすくするためにインメモリデータベースを使用しています。ただし、インメモリデータベースはデフォルトでは空であり、クエリをチェックするのには役立ちません。そのため、インメモリデータベースをポピュレートするために`INIT`スクリプトを提供しています。
この場合、`integration-schema.sql`ファイルには1行だけ含まれていれば十分です：

```sql
create table "message" (
  "content" VARCHAR NOT NULL,
  "id"      BIGSERIAL NOT NULL PRIMARY KEY
);
```


### Type Checked Plain SQL

`@StaticDatabaseConfig`があるため、`tsql`を使用できます：

```scala
val action: DBIO[Seq[String]] = tsql""" select "content" from "message" """
```

このクエリを、`sql`や`sqlu`クエリと同様に実行できます。
`SetParameter`型クラスを介してカスタム型も使用できます。ただし、`tsql`では`GetResult`型クラスは

サポートされていません。

クエリを誤って記述してみましょう：

```scala
val action: DBIO[Seq[String]] =
  tsql"""select "content", "id" from "message""""
```

何が間違っているかわかりますか？わからない場合は心配しないでください。なぜなら、コンパイラが問題を見つけてくれるからです：

```scala
type mismatch;
[error]  found    : SqlStreamingAction[
                        Vector[(String, Int)],
                        (String, Int),Effect ]
[error]  required : DBIO[Seq[String]]
```

コンパイラは、各行に対して`String`が必要であると判断しています。なぜなら、結果が`String`であると宣言したからです。
ただし、データベースを通じて、クエリが`(String,Int)`行を返すことがわかっています。

もし型宣言を省略した場合、アクションは`DBIO[Seq[(String,Int)]]`と推論されるでしょう。
このようなミスマッチをキャッチしたい場合は、`tsql`を使用する際に期待される型を宣言するのが良い習慣です。

コンパイラが検出する他の種類のエラーを見てみましょう。

たとえば、SQL自体が間違っている場合：

```scala
val action: DBIO[Seq[String]] =
  tsql"""select "content" from "message" where"""
```

これは不完全なSQLであり、コンパイラは次のように教えてくれます：

```scala
exception during macro expansion: ERROR: syntax error at end of input
[error]   Position: 38
[error]     tsql"""select "content" from "message" WHERE"""
[error]     ^
```

列名を間違えた場合も同様にコンパイルエラーです：

```scala
val action: DBIO[Seq[String]] =
  tsql"""select "text" from "message" where"""
```

```scala
Exception during macro expansion: ERROR: column "text" does not exist
[error]   Position: 8
[error]     tsql"""select "text" from "message""""
[error]     ^
```

もちろん、行を選択するだけでなく、挿入もできます：

```scala
val greeting = "Hello"
val action: DBIO[Seq[Int]] =
  tsql"""insert into "message" ("content") values ($greeting)"""
```

注意すべきは、実行時にクエリを実行すると新しい行が挿入されることです。
コンパイル時には、SlickはJDBC内の機能を使用してクエリをコンパイルし、クエリを実行せずにメタデータを取得します。
つまり、コンパイル時にデータベースは変更されません。



## Take Home Points


Plain SQLは、Slickのリフト埋め込みスタイルの制限から抜け出すための方法を提供します。

SQLには、2つの主要な文字列インターポレータが提供されています: `sql` と `sqlu`：

- `${expression}` を使用して値を安全にPlain SQLクエリに挿入できます。

- カスタム型を使用できます。その際、型に対する暗黙の`GetResult`（select）または`SetParameter`（update）がスコープ内に存在する必要があります。

- 生の値を`#$`でクエリに挿入できます。注意して使用してください：エンドユーザーが提供する情報は、クエリに挿入してはいけません。

`tsql`インターポレータは、コンパイル時にデータベースでPlain SQLクエリをチェックします。データベース接続はクエリの構文を検証するために使用され、選択されている列の型も発見されます。これを最大限に活用するためには、常に`tsql`から期待するクエリの型を宣言してください。

## Exercises

これらの演習では、メッセージとユーザーの組み合わせを使用します。
これをリフト埋め込みスタイルを使用してセットアップします。

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

以下の4つのクエリをPlain SQLクエリとして記述します：

- メッセージテーブルの行数をカウントする。

- メッセージテーブルからコンテンツを選択する。

- メッセージテーブルの各メッセージ（"content"）の長さを選択する。

- メッセージのコンテンツと長さを選択する。

ヒント：

- SQLでは、テーブル名やカラム名をダブルクォートで囲む必要があります。

- データベースのテーブルは単数形の名前を使用しています：`message`、`user` など。

<div class="solution">

SQL文は比較的単純です。ただし、`as[T]`をクエリの結果に合わせることに注意が必要です。

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

以下は、リフテッド埋め込みスタイルのクエリをプレーンなSQLクエリに変換した例です。

```scala
val whoSaidThatSQL =
  sql"""SELECT u."name" 
        FROM "message" m 
        JOIN "user" u ON m."sender_id" = u."id" 
        WHERE m."content" = 'Open the pod bay doors, HAL.'""".as[String]

exec(whoSaidThatSQL)
```

クエリの生成結果を確認するために、上記に示した `whoSaidThat` に関するステートメントを参照してみてください。

ポイント：

- SQLでは、文字列は二重引用符ではなく、シングルクォートで囲まれています。

- データベース内では、送信者のIDは `sender_id` です。


<div class="solution">

SQLを実装する方法はさまざまあります。ここに一つの例を示します。

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

指定されたコンテンツを言った人を返すためのメソッド `whoSaid` をプレーンなSQLクエリを使用して実装しましょう。


```scala
def whoSaid(content: String): DBIO[Seq[String]] =
  ???
```

Running `whoSaid("Open the pod bay doors, HAL.")` should return a list of the people who said that. Which should be Dave.

This should be a small change to your solution to the last exercise.

この `whoSaid("Open the pod bay doors, HAL.")` を実行すると、"Open the pod bay doors, HAL." というコンテンツを言った人のリストが返されるはずです。この場合はDaveが該当します。

前の演習の解決策を小さな変更で利用しています。

<div class="solution">

この解決策には `$` の置換を使用する必要があります。

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

このH2クエリは、アルファベット順で最初と最後のメッセージを返します。

```scala mdoc
exec(sql"""
  select min("content"), max("content")
  from "message" """.as[(String,String)]
)
```

この演習では、クエリの結果が次のような形式になるように`GetResult`型クラスのインスタンスを実装しましょう。

```scala mdoc:silent
case class FirstAndLast(first: String, last: String)
```

手順は以下の通りです。

1. `import slick.jdbc.GetResult`を記憶してください。

2. `GetResult[FirstAndLast]`の暗黙の値を提供します。

3. クエリを`as[FirstAndLast]`を使用するように変更します。


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

私たちは、プレーンなSQLを使用してデータベースを変更することができます。
これは行の挿入、行の更新、行の削除、スキーマの変更も含みます。

さあ、クルーのジュークボックスのプレイリストを格納するための新しいテーブルを、プレーンなSQLを使用して作成しましょう。

<div class="solution">
変更のために`sql`ではなく`sqlu`を使用します。

```scala mdoc
exec(sqlu""" create table "jukebox" ("title" text) """)

exec(sqlu""" insert into "jukebox"("title")
             values ('Bicycle Built for Two') """)

exec(sql""" select "title" from "jukebox" """.as[String])
```
</div>


### Robert Tables

私たちは、電子メールアドレスでユーザーを検索できるウェブサイトを構築しています。

```scala
def lookup(email: String) =
  sql"""select "id" from "user" where "email" = '#${email}'"""

// Example use:
exec(lookup("dave@example.org").as[Long].headOption)
```

このコードの問題は何ですか？

<div class="solution">

もし[xkcdの「Little Bobby Tables」](http://xkcd.com/327/)についてご存知であれば、このエクササイズのタイトルがおそらくお気づきかもしれません：`#$` は入力をエスケープしません。

これは、ユーザーが注意深く作成した電子メールアドレスを使用して悪意のある操作を行う可能性があることを意味します。

```scala
val evilAction = lookup("""';DROP TABLE "user";--- """).as[Long]
exec(evilAction)
```

この「電子メールアドレス」は、次の2つのクエリに変換されます：

~~~ sql
SELECT * FROM "user" WHERE "email" = '';
~~~

そして

~~~ sql
DROP TABLE "user";
~~~

これによって、この後のユーザーテーブルへのアクセスは以下のようなエラーになります：

```scala
exec(users.result.asTry)
```

はい、テーブルはクエリによって削除されました。

絶対にユーザー提供の入力に `#$` を使用しないでください。

</div>
