# Basics {#Basics}

## Orientation

Slick は Scala の collections ライブラリに似たインターフェースでリレーショナルデータベースにアクセスするための Scala ライブラリです。クエリをコレクションのように扱って、 `map`、`flatMap`、`filter` などのメソッドで変換・結合してからデータベースに送信して結果を取得することができます。このテキストの大部分では、この方法でSlickを使用します。

標準的なSlickのクエリはプレーンなScalaで書かれています。これらはコンパイル時のエラーチェックの恩恵を受ける _type safe_ 式です。また、データベースに対して実行する前に、単純な断片から複雑なクエリを構築することができるように、_compose_ されています。Scalaでクエリを書くのが苦手な方は、SlickでSQLクエリも書けることをご存知でしょうか。

クエリに加えて、Slickはデータベースへの接続、スキーマの作成、トランザクションの設定など、リレーショナル・データベースでよく行われるすべての作業を支援します。また、JDBC(Java Database Connectivity)については、Slickの下にドロップダウンして直接扱うことも可能です（もし、それが馴染みのあるもので、必要だと思うのであれば）。

本書は、Slickを商用で使用するために必要なすべての知識をコンパクトにまとめた、無駄のないガイドです。

- 第1章では、ライブラリ全体の概要を説明し、データモデリング、データベースへの接続、クエリの実行の基本を実演しています。
- 第2章では、基本的なselectクエリを取り上げ、Slickのクエリ言語を紹介し、型推論と型チェックの詳細について掘り下げています。
- 第3章では、データの挿入、更新、削除のためのクエリについて説明します。
- 第4章では、カスタムカラムやテーブル型の定義など、データモデリングについて説明します。
- 第5章では、アクションと、複数のアクションを組み合わせる方法について説明します。
- 第6章では、結合や集約を含む高度な select クエリについて説明します。
- 第7章では、_Plain SQL_ クエリの簡単な概要を説明します。

<div class="callout callout-info">

**Slick isn't an ORM**

[Hibernate][link-hibernate] や [Active Record][link-active-record] などの他のデータベースライブラリに慣れている方は、Slick を _Object-Relational Mapping (ORM)_ ツールであると期待するかもしれません。しかし、Slickをこのように考えない方が良いでしょう。

ORMはオブジェクト指向のデータモデルをリレーショナルデータベースのバックエンドにマッピングしようとするものです。それに対して、Slickはクエリや行、列といった、よりデータベースに近いツールのセットを提供します。ORMの長所と短所をここで論じるつもりはありませんが、もしこの分野に興味があるなら、Slickのマニュアルにある[Coming from ORM to Slick][link-ref-orm] という記事を見てみてください。

もしあなたがORMに精通していないのであれば、おめでとうございます。心配事が一つ減りましたね!

</div>

## Running the Examples and Exercises

この第1章の目的は、Slickの中核となるコンセプトの高レベルな概要を提供し、簡単なエンドツーエンドの例であなたを立ち上げることです。この例は、本書の練習問題のGitリポジトリをクローンすることで今すぐ入手できます。

```bash
bash$ git clone git@github.com:underscoreio/essential-slick-code.git
Cloning into 'essential-slick-code'...

bash$ cd essential-slick-code

bash$ ls -1
README.md
chapter-01
chapter-02
chapter-03
chapter-04
chapter-05
chapter-06
chapter-07
```

本書の各章は、例題と演習を組み合わせた別々のsbtプロジェクトと関連付けられています。各章のディレクトリにsbtを実行するのに必要なものをすべてバンドルしています。

ここでは、_Slack_、_Gitter_、_IRC_ に似たチャットアプリケーションを実行例として使用する予定です。このアプリは、この本を読み進めるにつれて成長し、進化していきます。最終的には、テーブル、リレーションシップ、クエリを使用してモデル化された、ユーザー、メッセージ、ルームを持つことになります。

とりあえず、2人の有名人の簡単な会話から始めてみましょう。`chapter-01` ディレクトリに移動して、`sbt` コマンドで sbt を起動し、サンプルをコンパイルして実行し、何が起きるかを見てみましょう。

```bash
bash$ cd chapter-01

bash$ sbt
# sbt log messages...

> compile
# More sbt log messages...

> run
Creating database table

Inserting test data

Selecting all messages:
Message("Dave","Hello, HAL. Do you read me, HAL?",1)
Message("HAL","Affirmative, Dave. I read you.",2)
Message("Dave","Open the pod bay doors, HAL.",3)
Message("HAL","I'm sorry, Dave. I'm afraid I can't do that.",4)

Selecting only messages from HAL:
Message("HAL","Affirmative, Dave. I read you.",2)
Message("HAL","I'm sorry, Dave. I'm afraid I can't do that.",4)
```

上記のような出力が得られたら、おめでとうございます。これでセットアップは完了です。この本の残りの部分を通して、例題や演習を実行する準備ができています。もし何かエラーが発生したら、私たちの [Gitter channel][link-underscore-gitter] にお知らせください。

<div class="callout callout-info">

**New to sbt?**

sbtを初めて実行すると、インターネットから多くのライブラリの依存関係をダウンロードし、ハードディスクにキャッシュします。これは2つのことを意味します。

- 開始するにはインターネット接続が必要であること。 
- 最初に実行する `compile` コマンドは完了するまでに時間がかかるかもしれません。 

もし、sbtを使ったことがなければ、[sbt Getting Started Guide][link-sbt-tutorial] が役に立つかもしれません。

</div>

## Working Interactively in the sbt Console

Slickのクエリは `Future` 値として非同期で実行されます。
これらはScala REPLで扱うには面倒ですが、REPLからSlickを探せるようにしたいのです。
そこで、手っ取り早くスピードアップするために
サンプルプロジェクトは `exec` メソッドを定義し、コンソールからサンプルを実行するための基本要件をインポートしています。

これは `sbt` を起動して、 `console` コマンドを実行することで確認することができます。
その結果、次のような出力が得られます。

```scala
> console
[info] Starting scala interpreter...
[info]
Welcome to Scala 2.12.1 (Java HotSpot(TM) 64-Bit Server VM, Java 1.8.0_112).
Type in expressions for evaluation. Or try :help.

scala> import slick.jdbc.H2Profile.api._
import Example._
import scala.concurrent.duration._
import scala.concurrent.Await
import scala.concurrent.ExecutionContext.Implicits.global
db: slick.jdbc.H2Profile.backend.Database = slick.jdbc.JdbcBackend$DatabaseDef@ac9a820
exec: [T](program: slick.jdbc.H2Profile.api.DBIO[T])T
res0: Option[Int] = Some(4)
scala>
```

私たちの `exec` ヘルパーはクエリを実行し、その出力を待ちます。
この章の後のほうで、 `exec` とこれらのインポートについて完全な説明があります。
とりあえず、すべての `message` の行を取得する小さな例を示します。

```scala
exec(messages.result)
// res1: Seq[Example.MessageTable#TableElementType] =
// Vector(Message(Dave,Hello, HAL. Do you read me, HAL?,1),
//       Message(HAL,Affirmative, Dave. I read you.,2),
//       Message(Dave,Open the pod bay doors, HAL.,3),
//       Message(HAL,I'm sorry, Dave. I'm afraid I can't do that.,4))
```

この章では、クエリの構築と実行、そして `exec` の使用について説明します。
もし上記がうまくいったら、素晴らしい。これで開発環境は整いました。

## Example: A Sequel Odyssey

上で見たテストアプリケーションは、[H2][link-h2-home] を使ってインメモリデータベースを作成し、一つのテーブルを作成してテストデータを投入し、いくつかのサンプルクエリを実行しました。このセクションの残りの部分では、コードを順を追って説明し、これから起こることの概要を説明します。本文中ではコードの重要な部分を再現しますが、演習のコードベースでも同様に追うことができます。

<div class="callout callout-warning">

**Choice of Database**

本書のすべての例では、[H2][link-h2-home] データベースを使用しています。H2 は Java で書かれており、アプリケーションコードの横でインプロセスで実行されます。私たちが H2 を選んだ理由は、システム管理を一切行わず、Scala を書くことに専念できるためです。

_MySQL_ や _PostgreSQL_ 、あるいは他のデータベースを使いたいかもしれません。[Appendix A](#altdbs) では、他のデータベースで動作させるために必要な変更について説明しています。しかし、少なくともこの最初の章ではH2にこだわることをお勧めします。そうすれば、データベース固有の問題にぶつかることなく、Slickの使用に自信を持つことができます。

</div>

### Library Dependencies

Scalaのコードに飛び込む前に、sbtの設定を見てみましょう。これはサンプルの `build.sbt` にあります。

```scala
name := "essential-slick-chapter-01"

version := "1.0.0"

scalaVersion := "2.13.3"

libraryDependencies ++= Seq(
  "com.typesafe.slick" %% "slick"           % "3.3.3",
  "com.h2database"      % "h2"              % "1.4.200",
  "ch.qos.logback"      % "logback-classic" % "1.2.3"
)
```

このファイルは、Slickプロジェクトに必要な最低限のライブラリ依存関係を宣言しています。

- Slick

- H2データベース

- ロギング・ライブラリ

もしMySQLやPostgreSQLのような別のデータベースを使う場合は、H2への依存をそのデータベースのJDBCドライバに置き換えるでしょう。

### Importing Library Code

データベース管理システムは、同じように作られているわけではありません。異なるシステムは異なるデータ型、異なるSQLの方言、異なるクエリ機能をサポートします。これらの機能をコンパイル時に確認できるようにモデル化するために、Slick はデータベース固有の _profile_ を介して API の大部分を提供します。例えば、H2 用の Slick API の大部分には、以下の `import` を介してアクセスします。

```scala mdoc:silent
import slick.jdbc.H2Profile.api._
```

Slickは暗黙の変換と拡張メソッドを多用するので、クエリやデータベースを扱う場所では一般的にこのインポートを含める必要があります。[第5章](#Modelling)では、特定のデータベースプロファイルを必要なときまでコードから除外する方法について説明します。

### Defining our Schema

私たちの最初の仕事は、データベースにどんなテーブルがあって、それをScalaの値や型にどうマッピングするかをSlickに伝えることです。Scalaで最も一般的なデータの表現はケースクラスです。そこで、まずは例のテーブルの一行を表す `Message` クラスを定義します。

```scala mdoc
case class Message(
  sender:  String,
  content: String,
  id:      Long = 0L)
```

次に `Table` オブジェクトを定義します。これはデータベースのテーブルに対応し、データベースのデータとケースクラスのインスタンスとの間でどのように行き来するかを Slick に指示します。

```scala mdoc
class MessageTable(tag: Tag) extends Table[Message](tag, "message") {

  def id      = column[Long]("id", O.PrimaryKey, O.AutoInc)
  def sender  = column[String]("sender")
  def content = column[String]("content")

  def * = (sender, content, id).mapTo[Message]
}
```

`MessageTable` は3つの `column` を定義しています。`id`, `sender`, そして `content` です。これらのカラムの名前と型、そしてデータベースレベルでの制約を定義します。例えば、 `id` は `Long` 値のカラムで、オートインクリメントの主キーでもあります。

`*`メソッドは、テーブルのカラムとケースクラスのインスタンスを対応させるための _default projection_ を提供します。
Slick の `mapTo` マクロは、 `Message` の 3 つのカラムと 3 つのフィールドの間に双方向のマッピングを作成します。

プロジェクションやデフォルトプロジェクションについては、[5章](#Modelling)で詳しく説明します。
今のところ、知っておくべきことは、この行によってデータベースに問い合わせをして、 `(String, String, Long)` のタプルの代わりに `Messages` を取得できるようになるということです。

1行目の `tag` は実装の詳細で、Slickが1つのクエリで複数のテーブルを使用できるようにするためのものです。
SQLのテーブルエイリアスのようなものだと考えてください。ユーザーコードでタグを指定する必要はありません。Slickが自動的にタグを指定します。

### Example Queries

Slick では、データベースに対してクエリを実行する前に、クエリを定義したり構成したりすることができます。まずは `TableQuery` オブジェクトを定義します。これはメッセージテーブルに対する単純な `SELECT *` 形式のクエリを表します。

```scala mdoc
val messages = TableQuery[MessageTable]
```

このクエリはすぐに実行されるわけではないことに注意してください（単に他のクエリを作るための手段として定義しているだけです）。例えば、 `filter` というコンビネータを使って、 `SELECT * WHERE` スタイルのクエリを作ることができます。

```scala mdoc
val halSays = messages.filter(_.sender === "HAL")
```

繰り返しますが、このクエリはまだ実行されていません。このクエリは、さらに多くのクエリのための構成要素として定義されています。これはSlickのクエリ言語の重要な部分を示しています。それは、貴重なコードの再利用を可能にする _composable_ 要素から作られています。

<div class="callout callout-info">

**Lifted Embedding**

もしあなたが専門用語が好きなら、これまで説明してきたことをSlickでは _lifted embedding_ アプローチと呼んでいることを知っておいてください。

- 行データを格納するためのデータ型を定義する（ケースクラス、タプル、その他の型）。
- データ型とデータベース間のマッピングを表す `Table` オブジェクトを定義する。
- データベースに対してクエリを実行する前に、 `TableQueries` とコンビネータを定義して、有用なクエリを生成する。

Lifted Embding は Slick を使う標準的な方法です。もう一つの方法は _Plain SQL querying_ と呼ばれ、[7章](#PlainSQL)で説明します。

</div>

### Configuring the Database

ここまでのコードはすべて、データベースに接続することなく書かれています。さて、いよいよ接続を開いて SQL を実行するときが来ました。まず、接続とトランザクションを管理するためのファクトリーとして機能する `Database` オブジェクトを定義することから始めます。

```scala mdoc
val db = Database.forConfig("chapter01")
```

`Database.forConfig` のパラメータは、`application.conf` ファイルからどの設定を使用するかを決定します。
このファイルは `src/main/resources` にあります。このファイルは以下のようなものです。

```scala
chapter01 {
  driver = "org.h2.Driver"
  url    = "jdbc:h2:mem:chapter01"
  keepAliveConnection = true
  connectionPool = disabled
}
```

この構文は、Akka や Play フレームワークでも使用されている [Typesafe Config][link-config] ライブラリに由来しています。

今回提供するパラメータは、基盤となる JDBC レイヤーを設定するためのものです。
`driver` パラメータは、選択した DBMS 用の JDBC ドライバの完全修飾クラス名です。

`url`パラメータは、標準的な[JDBC接続URL][link-jdbc-connection-url]です。
この例では、`"chapter01"` という名前のインメモリデータベースを作成します。

デフォルトでは、H2インメモリデータベースは最後の接続が閉じられると削除されます。
今回の例では、複数の接続を実行する予定なので
プログラムが完了するまでデータを保持するために `keepAliveConnection` を有効にします。

Slick はデータベース接続とトランザクションを自動コミットで管理します。
トランザクションについては[4章](#combining)で見ていきます。

<div class="callout callout-info">

**JDBC**

Javaを使った作業をしたことがない人は、Java Database Connectivity（JDBC）という言葉を聞いたことがないかもしれません。これは、ベンダーに依存しない方法でデータベースにアクセスするための仕様であり、
中立的な方法でデータベースにアクセスするための仕様です。つまり、接続する特定のデータベースに依存しないことを目的としています。

この仕様は、接続するデータベースごとに実装されたライブラリによって反映されます。このライブラリは、_JDBC driver_ と呼ばれます。

JDBCは _connection strings_ で動作します。これは上記のようなURLで、データベースがどこにあり、どのように接続するか（例えば、ログイン認証情報を提供することによって）をドライバに伝えるものです。

</div>

### Creating the Schema

これで `db` としてデータベースが設定されたので、それを使ってみましょう。

まずは `MessageTable` の `CREATE` 文から始めましょう。これは、 `TableQuery` オブジェクトである `messages` のメソッドを使って作成します。Slick のメソッド `schema` はスキーマの記述を取得します。`createStatements` メソッドによって、スキーマの記述を確認することができます。

```scala mdoc
messages.schema.createStatements.mkString
```
まだデータベースには送信していません。ステートメントを出力して、それが私たちの考えるものであることを確認したところです。

Slickでは、データベースに対して実行するのは _action_ です。これは `messages` スキーマに対するアクションを作成する方法です。

```scala mdoc
val action: DBIO[Unit] = messages.schema.create
```

この `messages.schema.create` 式の結果は `DBIO[Unit]` です。これは DB のアクションを表すオブジェクトで、実行すると `Unit` 型の結果で完了します。データベースに対して実行するものはすべて `DBIO[T]` (または、より一般的には `DBIOAction`) です。これには、クエリ、アップデート、スキーマの変更などが含まれます。

<div class="callout callout-info">

**DBIO and DBIOAction** 

本書では、アクションは `DBIO[T]` 型を持つものとして話を進めます。

これは単純化したものです。より一般的な型は `DBIOAction` で、特にこの例では `DBIOAction[Unit, NoStream, Effect.Schema]` となります。これらすべての詳細については、この本の後半で説明します。

しかし、`DBIO[T]`はSlickが提供する型の別名であり、使用しても全く問題ありません。

</div>

アクションを実行しましょう。

```scala mdoc
import scala.concurrent.Future
val future: Future[Unit] = db.run(action)
```

`run` の結果は `Future[T]` で、`T` はデータベースから返される結果の型です。スキーマの作成は副作用のある操作なので、結果の型は `Future[Unit]` となります。これは、先ほどのアクションの型 `DBIO[Unit]` と一致します。

`Future` は非同期です。つまり、いずれ出現する値のプレースホルダーです。Futureはある時点で「完了する」ものとも言い換えられます。プロダクションコードでは、Futureを使うことで、結果を待つためにブロッキングすることなく、計算を連鎖させることができます。しかし、このような簡単な例では、アクションが完了するまでブロックすることができます。

```scala mdoc
import scala.concurrent.Await
import scala.concurrent.duration._
val result = Await.result(future, 2.seconds)
```

### Inserting Data

テーブルをセットアップしたら、テストデータを挿入する必要があります。ここでは、デモ用にいくつかのテスト用 `Messages` を作成するヘルパーメソッドを定義します。

```scala mdoc
def freshTestData = Seq(
  Message("Dave", "Hello, HAL. Do you read me, HAL?"),
  Message("HAL",  "Affirmative, Dave. I read you."),
  Message("Dave", "Open the pod bay doors, HAL."),
  Message("HAL",  "I'm sorry, Dave. I'm afraid I can't do that.")
)
```

このテストデータの挿入がアクションとなります。

```scala mdoc
val insert: DBIO[Option[Int]] = messages ++= freshTestData
```
`message` の `++=` メソッドは、一連の `Message` オブジェクトを受け取り、それらをbulk `INSERT` クエリに変換します (`freshTestData` は通常の Scala `Seq[Message]` です)。
そして、 `db.run` によって `insert` を実行し、Futureが完了するとテーブルにデータが格納されます。


```scala mdoc
val insertAction: Future[Option[Int]] = db.run(insert)
```

insert操作の結果は、挿入された行の数です。
`freshTestData` には 4 つのメッセージが含まれているので、この場合、future が完了すると結果は `Some(4)` になります。


```scala mdoc
val rowCount = Await.result(insertAction, 2.seconds)
```

この結果はOptionalです。なぜなら、基盤となるJava APIはバッチ挿入の行数を保証していないからです（データベースによっては、単に `None` を返すものもあります）。
単一またはバッチの挿入と更新については、[3章](#Modifying)でさらに詳しく説明します。

### Selecting Data

これでデータベースに数行が登録されたので、データの選択を開始することができます。これは、 `messages` や `halSays` などのクエリを取得し、 `result` メソッドを使用してアクションに変換することで行います。

```scala mdoc
val messagesAction: DBIO[Seq[Message]] = messages.result

val messagesFuture: Future[Seq[Message]] = db.run(messagesAction)

val messagesResults = Await.result(messagesFuture, 2.seconds)
```

```scala mdoc:invisible
assert(messagesResults.length == 4, "Expected 4 results")
```

アクションの `statements` メソッドを使用して、H2 に発行された SQL を見ることができます。

```scala mdoc
val sql = messages.result.statements.mkString
```

```scala mdoc:invisible
assert(sql == """select "sender", "content", "id" from "message"""", s"Expected: $sql")
```

<div class="callout callout-info">

**The `exec` Helper Method**

アプリケーションでは、可能な限り `Future` でのブロッキングを避けるべきです。
しかし、この本の例では `Await.result` を多用することになるでしょう。
ここでは、例を読みやすくするために `exec` というヘルパーメソッドを導入します。

```scala mdoc
def exec[T](action: DBIO[T]): T =
  Await.result(db.run(action), 2.seconds)
```

`exec` が行うのは、与えられたアクションを実行し、その結果を待つだけです。
例えば、selectクエリを実行するには、次のように書きます。

```scala
exec(messages.result)
```

`Await.result` を使用することは、プロダクションコードでは強く推奨されません。
多くの Web フレームワークでは、ブロックせずに `Future` を操作するための直接的な手段が提供されています。
このような場合、最良の方法は、クエリの結果を `Future` に変換し、HTTP レスポンスの `Future` に変換してクライアントに送信することです。

</div>

テーブル内のメッセージの一部を取得したい場合、このクエリを修正したものを実行します。
例えば、 `filter` を `messages` に対して実行すると、次のようなクエリが作成されます。

```scala mdoc
messages.filter(_.sender === "HAL").result.statements.mkString
```

このクエリを実行するには、`result` を使ってクエリをアクションに変換します。
`db.run` でデータベースに対して実行し、 `exec` で最終的な結果を待ちます。

```scala mdoc
exec(messages.filter(_.sender === "HAL").result)
```

先ほど、このクエリを実際に生成し、変数 `halSays` に格納しました。
代わりにこの変数を実行することで、データベースからまったく同じ結果を得ることができます。

```scala mdoc
exec(halSays.result)
```

データベースに接続する前に、オリジナルの `halSays` を作成したことに注意してください。
これは、小さな部品からクエリを構成して、後でそれを実行するという概念を完全に示しています。

修飾子を重ねて、複数の追加句を持つクエリを作成することもできます。
例えば、カラムのサブセットを取得するために `map` をクエリに追加することができます。
これは、SQL の `SELECT` 句と `result` の戻り値の型を変更します。

```scala mdoc
halSays.map(_.id).result.statements.mkString

exec(halSays.map(_.id).result)
```

### Combining Queries with For Comprehensions


`Query` は _monad_ です。`map`, `flatMap`, `filter`, `withFilter` メソッドを実装しており、Scala と互換性のあるfor式を実現しています。
例えば、このようなスタイルで書かれた Slick クエリをよく見かけます。

```scala mdoc
val halSays2 = for {
  message <- messages if message.sender === "HAL"
} yield message
```

for式はメソッド呼び出しの連鎖のエイリアスであることを忘れないでください。
ここでやっていることは、`WHERE`句を含むクエリを作成することです。
クエリを実行するまでは、データベースには触れません。

```scala mdoc
exec(halSays2.result)
```

`Query` と同様に、 `DBIOAction` もmonadです。上で説明したのと同じメソッドを実装しており、for式と互換性があります。

スキーマの作成、データの挿入、そしてクエリの結果を一つのアクションにまとめることができます。データベース接続を行う前にこれを行うことができ、他のアクションと同じように実行します。
これを行うために、Slickは便利なアクションコンビネータを多数提供しています。例えば、`andThen`を使うことができます。

```scala mdoc
val actions: DBIO[Seq[Message]] = (
  messages.schema.create       andThen
  (messages ++= freshTestData) andThen
  halSays.result
)
```

`andThen` が行うのは、2つのアクションを組み合わせて、最初のアクションの結果を捨てさせることです。
上記の `actions` の最終結果は `andThen` チェーンの最後のアクションになります。

もし、あなたがファンキーなことをしたいのなら、 `>>` は `andThen` のエイリアスです。

```scala mdoc
val sameActions: DBIO[Seq[Message]] = (
  messages.schema.create       >>
  (messages ++= freshTestData) >>
  halSays.result
)
```

アクションを組み合わせることは、Slickの重要な機能です。
例えば、アクションを組み合わせる理由の1つは、単一のトランザクションの中にそれらを包み込むことです。
[第4章](#combining)では、このことを説明します。また、アクションはクエリと同じようにfor式で構成できることも説明します。

<div class="callout callout-danger">

*Queries, Actions, Futures... Oh My!*

クエリ、アクション、Futureの違いは、Slick 3 を初めて使う人にとっては大きな混乱の元となります。この3つの型は多くの特性を共有しています。`map`、`flatMap`、`filter` などのメソッドを持ち、内包と互換性があり、Slick API のメソッドを通してシームレスに相互作用しています。しかし、そのセマンティクスは全く異なっています。

- `Query` は単一のクエリの SQL を構築するために使用されます。`map` と `filter` の呼び出しは SQL の句を変更するが、作成されるクエリは 1 つだけです。

- `DBIOAction` は一連の SQL クエリを作成するために使用されます。`map` と `filter` を呼び出すと、クエリが連結され、その結果がデータベースから取得されたときに変換されます。`DBIOAction` は、トランザクションを定義するためにも使用されます。

- `Future` は、`DBIOAction` を実行した非同期の結果を変換するために使用されます。`Future` に対する変換は、データベースとの通信が終了した後に行われます。

多くの場合（例えば select クエリ）、最初に `Query` を作成し、`result` メソッドを使って `DBIOAction` に変換します。他の場合 (例えばinsertクエリ) は、Slick API が `Query` をバイパスして、すぐに `DBIOAction` を生成します。いずれの場合も、 `db.run(...)` を使って `DBIOAction` を実行し、その結果を `Future` に変換します。

時間をかけて `Query`, `DBIOAction`, `Future` を徹底的に理解することをお勧めします。それらがどのように使われ、どのように似ていて、どのように違うのか、それらの型パラメータが何を表しているのか、そしてそれらがどのようにお互いに流れ込んでいくのかを学びましょう。これはおそらく、Slick 3 を理解するための最大のステップになるでしょう。

</div>

## Take Home Points

この章では、スキーマの定義、データベースへの接続、データを取得するためのクエリの発行など、Slickの主要な側面について大まかに見てきました。

データベースからのデータは通常、テーブルの行に対応するケースクラスとタプルとしてモデル化します。これらの型とデータベースとの間のマッピングは、`MessageTable` のような `Table` クラスを用いて定義します。

クエリの定義には、`messages` などの `TableQuery` オブジェクトを作成し、`map` や `filter` などのコンビネータで変換します。
これらの変換は、コレクションに対する変換のように見えますが、返される結果を操作するのではなく、SQL コードをビルドするために使用されます。

クエリを実行するには、`result` メソッドでアクションオブジェクトを作成します。アクションは関連するクエリのシーケンスを構築し、それをトランザクションでラップするために使用されます。

最後に、データベースオブジェクトの `run` メソッドにアクションを渡すことで、データベースに対してアクションを実行します。その結果、`Future` が返されます。Futureが完了すると、結果が利用可能になります。

クエリ言語はSlickの中で最も豊富で重要な部分の一つです。次の章では、様々なクエリや変換について説明します。

## Exercise: Bring Your Own Data

サンプルのデータベースに対してクエリを実行することで、Slick を少し体験してみましょう。
`sbt` コマンドで sbt を起動し、 `console` と入力して Scala の対話型コンソールに入ります。
コントロールする前に、サンプルアプリケーションを実行するように sbt を設定しました。
そのため、テスト用のデータベースをセットアップして、すぐに使える状態から始める必要があります。

```bash
bash$ sbt
# sbt logging...

> console
# More sbt logging...
# Application runs...

scala>
```

まず、データベースに余分な台詞を挿入することから始めましょう。
このセリフは、映画『2001年』の開発後半にカットされたものです。
このセリフは、映画「2001年」の開発中にカットされましたが、ここで復活させることができます。

```scala mdoc
Message("Dave","What if I say 'Pretty please'?")
```

`messages`の `+=` メソッドを使用して、行を挿入する必要があります。
あるいは、`Seq` にメッセージを入れて、`++=` を使ってもよいでしょう。
もしものときのために、よくある落とし穴をいくつか紹介しておきます。

<div class="solution">
こちらが解答になります。

```scala mdoc
exec(messages += Message("Dave","What if I say 'Pretty please'?"))
```

戻り値は `1` 行が挿入されたことを示しています。
自動インクリメントの主キーを使用しているので、Slickは `Message` の `id` フィールドを無視し、新しい行に `id` を割り当てるようにデータベースに要求します。
次の章で説明するように、挿入クエリが行数の代わりに新しい `id` を返すようにすることは可能です。

ここで、うまくいかないかもしれないことをいくつか挙げてみましょう。

もし、 `+=` で作成したアクションを `db` に渡して実行させなければ、代わりに `Action` オブジェクトが返されることになります。

```scala mdoc
messages += Message("Dave","What if I say 'Pretty please'?")
```

Futureが完了するのを待たずに、Futureそのものだけを見ることになるのです

```scala mdoc
val f = db.run(messages += Message("Dave","What if I say 'Pretty please'?"))
```

```scala mdoc:invisible

  // Post-exercise clean up
  // We inserted a new message for Dave twice in the last solution.
  // We need to fix this so the next exercise doesn't contain confusing duplicates

  // NB: this block is not inside {}s because doing that triggered:
  // Could not initialize class $line41.$read$$iw$$iw$$iw$$iw$$iw$$iw$

  import scala.concurrent.ExecutionContext.Implicits.global
  val ex1cleanup: DBIO[Int] = for {
    _ <- messages.filter(_.content === "What if I say 'Pretty please'?").delete
    m = Message("Dave","What if I say 'Pretty please'?", 5L)
    _ <- messages.forceInsert(m)
    count <- messages.filter(_.content === "What if I say 'Pretty please'?").length.result
  } yield count
  val rowCountCheck = exec(ex1cleanup)
  assert(rowCountCheck == 1, s"Wrong number of rows after cleaning up ex1: $rowCountCheck")
```

</div>

ここで、Dave が送信したすべてのメッセージを選択して、新しいダイアログを取得します。
適切なクエリを `messages.filter` で作成し、その `result` メソッドで実行するアクションを作成する必要があります。
私たちが提供した `exec` ヘルパーメソッドを使用して、クエリを実行することを忘れないでください。

繰り返しになりますが、よくある落とし穴もソリューションに含まれています。

<div class="solution">
こちらがコードです。

```scala mdoc
exec(messages.filter(_.sender === "Dave").result)
```

もし実行結果が読みにくければ、各メッセージを順番に表示することができます。
`Future` は `Message` のコレクションとして評価されるので、 `Message => Unit` の関数、例えば `println` を使って `foreach` することができます。

```scala mdoc
val sentByDave: Seq[Message] = exec(messages.filter(_.sender === "Dave").result)
sentByDave.foreach(println)
```

以下の場合、うまくいかないかもしれません。

`filter` のパラメータは、通常の `==` ではなく、 `===` という三重等号演算子を使って作られていることに注意してください。
もし、 `==` を使用すると、面白いコンパイルエラーが発生します。


```scala mdoc:fail
exec(messages.filter(_.sender == "Dave").result)
```

ここでのトリックを理解するには、実際には `_.sender` と `"Dave"` を比較しようとしているのではないことに注目してください。
一般の等式は `Boolean` として評価されますが、 `===` は `Rep[Boolean]` 型の SQL 式を構築します。
(Slick は `Rep` 型を使用して、`Column` 自身だけでなく、`Column` 上の式を表現します)。
このエラーメッセージを最初に見たときは戸惑いますが、何が起こっているのかを理解すれば納得がいくでしょう。

最後に、もし `result` を呼び出すのを忘れると、コンパイルエラーが発生します。
というのも、 `exec` とそれをラップしている `db.run` は両方ともアクションを期待しているからです。

```scala mdoc:fail
exec(messages.filter(_.sender === "Dave"))
```

`Query` 型は冗長になりがちで、実際の問題の原因から目をそらすことになります。
(つまり、 `Query` オブジェクトをまったく期待していないということです)。
次の章では、 `Query` 型について詳しく説明します。

</div>
