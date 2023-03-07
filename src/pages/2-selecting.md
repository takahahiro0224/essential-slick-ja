# Selecting Data {#selecting}

前章では、Slickのエンドツーエンドの概要を浅く説明しました。データをモデル化し、クエリを作成し、それをアクションに変換し、データベースに対してそれらのアクションを実行する方法について見てきました。次の2つの章では、Slickで実行できる様々なタイプのクエリについて、より詳しく見ていきます。

この章では、Slickの豊富なSQLの型安全なScalaリフレクションを使って、データの *選択（Select）* をカバーします。[第3章](#Modifying)では、レコードの挿入、更新、削除によるデータの *変更* について説明します。

selectクエリは、データを取得するための主な手段です。
この章では、単一のテーブルに対して操作する単純なselectクエリに限定して説明します。
[第6章](#joins)では、結合、集約、およびグループ化句を含む、より複雑なクエリについて見ていきます。

## Select All The Rows!

最も単純な select クエリは、 `Table` から生成される `TableQuery` です。

以下の例では、 `messages` が `MessageTable` に対する `TableQuery` です。

```scala mdoc
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
```

メッセージの型は `TableQuery[MessageTable]` で、これはより一般的な `Query` 型のサブタイプであり、Slick は select, update, delete クエリを表現するためにこれを使用します。これらの型については、次のセクションで説明します。

select クエリの SQL は、 `result.statements` を呼び出すことで見ることができます。

```scala mdoc
messages.result.statements.mkString
```

この `TableQuery` は、SQL の `select * from message` に相当するものです。

<div class="callout callout-warning">

**Query Extension Methods**

以下で説明する多くのメソッドと同様に、 `result` メソッドは実際には暗黙の変換によって `Query` に適用される拡張メソッドです。
これを動作させるには、 `H2Profile.api` に含まれるすべてのメソッドがスコープ内になければなりません。

```scala mdoc:silent
import slick.jdbc.H2Profile.api._
```
</div>

## Filtering Results: The *filter* Method

`filter`メソッドを使って、行の部分集合に対するクエリを作成することができます。

```scala mdoc
messages.filter(_.sender === "HAL")
```

`filter` のパラメータは、 `MessageTable` のインスタンスから、クエリの `WHERE` 節を表す `Rep[Boolean]` 型の値への関数である。

```scala mdoc
messages.filter(_.sender === "HAL").result.statements.mkString
```

Slick は `Rep` 型を使用して、個々のカラムだけでなく、カラムに対する式を表現することができます。
`Rep[Boolean]` は、テーブルの `Boolean` 値のカラムを表したり、
複数のカラムを含む `Boolean` 式を表すこともできます。
Slick は `A` 型の値を自動的に定数 `Rep[A]` に昇格させることができる。
また、以下に述べるように、式を構築するための一連のメソッドを提供します。

## The Query and TableQuery Types {#queryTypes}

この `filter` 式の型については、もう少し詳しい説明が必要でしょう。
Slickは全てのクエリを `Query[M, U, C]` という3つの型パラメータで表現しています。

 - `M` は *mixed* 型と呼ばれます。これは、 `map` や `filter` といったメソッドを呼び出す際に見られる、関数パラメータの型です。

 - `U` は *unpacked* 型と呼ばれます。これは、結果に収集される型です 
 
 - `C` は *collection* 型と呼ばれます。これは、結果をコレクションに蓄積していく型です。 

上記の例では、 `messages` は `Query` のサブタイプである `TableQuery` と呼ばれるものです。
以下は、Slick コードベースにおける定義の簡略版です。

``` scala
trait TableQuery[T <: Table[_]] extends Query[T, T#TableElementType, Seq] {
  // ...
}
```

`TableQuery` は、実際には `Table` (例: `MessageTable`) をmixed型として、テーブルの要素の型 (コンストラクタの type パラメータ、例: `Message`) をunpacked型として使用する `Query` です。
言い換えると、 `messages.filter` に渡す関数は、実際には `MessageTable` 型のパラメータを渡されます。

```scala mdoc
messages.filter { messageTable: MessageTable =>
  messageTable.sender === "HAL"
}
```

これは理にかなっています。 `messageTable.sender` は、上記の `MessageTable` で定義したカラムの 1 つです。
そして `messageTable.sender === "HAL"` は、 SQL 式 `message.sender = 'HAL'` を表す Scala 値を作成します。

これはSlickがクエリの型チェックをするための処理です。
`Query`は、作成に使用した `Table` の型にアクセスすることができます。
これにより、 `map` や `filter` などのコンビネータを使用する際に、 `Table` のカラムを直接参照することができるようになります。
全てのカラムは自身のデータ型を把握しているので、Slick は互換性のあるデータ型のカラムのみを比較することを保証します。
例えば、 `sender` と `Int` を比較しようとすると、型エラーが発生します。

```scala mdoc:fail
messages.filter(_.sender === 123)
```

<div class="callout callout-info">
<a name="constantQueries"/>

**Constant Queries**

これまで、私たちは `TableQuery` からクエリを構築してきました。
この本の大部分では、このようなケースが一般的です。
しかし、`select 1` のような、テーブルとは関係のない定型のクエリも作成できることを知っておく必要があります。

これには `Query` コンパニオンオブジェクトを使用します。

```scala mdoc:silent
Query(1)
```

上記はこのようなクエリを生成します。

```scala mdoc
Query(1).result.statements.mkString
```


</div>

`Query` オブジェクトの `apply` メソッドを使うと、スカラー値を `Query` に渡すことができます。

`select 1` のような定型のクエリを使用して、データベースへの接続を確認することができます。
これは、アプリケーションを起動するときに便利な機能です。また、最小限のリソースしか消費しないハートビートなシステムチェックにもなります。

[第3章](#moreControlOverInserts)では、`from`なしのクエリを使用する別の例を見ていきましょう。

## Transforming Results

<div class="callout callout-info">

**`exec`**

第1章で行ったように、REPLでクエリを実行するためにヘルパーメソッドを使用しています。

```scala mdoc:silent
import scala.concurrent.{Await,Future}
import scala.concurrent.duration._
```

```scala mdoc
val db = Database.forConfig("chapter02")

def exec[T](action: DBIO[T]): T =
  Await.result(db.run(action), 4.seconds)
```

これは、この章のサンプルのソースコードの `main.scala` ファイルに含まれています。
REPL でこれらの例を実行すれば、テキストを追うことができます。

また、スキーマとサンプルデータも設定しました。

```scala mdoc
def freshTestData = Seq(
  Message("Dave", "Hello, HAL. Do you read me, HAL?"),
  Message("HAL",  "Affirmative, Dave. I read you."),
  Message("Dave", "Open the pod bay doors, HAL."),
  Message("HAL",  "I'm sorry, Dave. I'm afraid I can't do that.")
)

exec(messages.schema.create andThen (messages ++= freshTestData))
```
</div>

### The *map* Method

時には、`Table` のすべてのカラムを選択したくないこともあります。
`Query` の `map` メソッドを使用すると、特定のカラムを選択して結果に含めることができます。
これは、クエリのmixed型とunpacked型の両方を変更します。

```scala mdoc
messages.map(_.content)
```

unpacked型（2番目の型パラメータ）が `String` に変更されたからです。
これで、実行時に `String` を選択するクエリができました。
このクエリを実行すると、各メッセージの `content` だけが取得されることがわかります。

```scala mdoc
val query = messages.map(_.content)

exec(query.result)
```

また、生成されたSQLが変更されていることに注目してください。
Slickは不正をしているわけではありません。実際にデータベースに、SQLの中でそのカラムに結果を限定するように指示しているのです。

```scala mdoc
messages.map(_.content).result.statements.mkString
```

最後に、新しいクエリのmixed型 (最初の型パラメータ) が `Rep[String]` に変更されたことに注目してください。
これは、このクエリに対して `filter` や `map` を実行したときに、 `content` カラムだけが渡されることを意味します。

```scala mdoc
val pods = messages.
  map(_.content).
  filter{content:Rep[String] => content like "%pod%"}

exec(pods.result)
```

このmixed型の変更は、 `map` によるクエリの合成を複雑にする可能性があります。
他の全ての操作を適用した後に、クエリに対する一連の変換の最終ステップとしてのみ、 `map` を呼び出すことを推奨します。

注目すべきは、Slick が `select` 句の一部としてデータベースに渡すことができるものであれば、何でも `map` することができるということです。
これには、個々の `Rep` や `Table` が含まれます。
また、これらのタプルも含まれます。
例えば、 `map` を使って、メッセージの `id` と `content` カラムを選択することができます。

```scala mdoc
messages.map(t => (t.id, t.content))
```

それに応じて，mixed型とunpacked型が変更され、SQLは予想通りに修正されます。

```scala mdoc
messages.map(t => (t.id, t.content)).result.statements.mkString
```

また、`mapTo`を使えば、カラムのセットをScalaのデータ構造にマッピングすることもできます。

```scala mdoc
case class TextOnly(id: Long, content: String)

val contentQuery = messages.
  map(t => (t.id, t.content).mapTo[TextOnly])

exec(contentQuery.result)
```

また、単一の列だけでなく、列の式もselectすることができます。

```scala mdoc
messages.map(t => t.id * 1000L).result.statements.mkString
```

つまり、 `map` はクエリの `SELECT` 部分を制御するための強力なコンビネータなのです。

<div class="callout callout-info">

**Query's *flatMap* Method**

`Query` には `Option` や `Future` と同様のモナドセマンティクスを持つ `flatMap` メソッドも用意されています。
`flatMap` は主にジョインに使用されるので、[6章](#joins) で説明します。
</div>

### *exists*

時には、クエリの結果の内容よりも、結果が存在するかどうかに興味があることがあります。
このような場合のために `exists` が用意されており、結果セットが空でなければ `true` を返し、そうでなければ`false` を返します。

既存のクエリを `exists` キーワードでどのように使用できるのか、簡単な例を見てみましょう。

```scala mdoc
val containsBay = for {
  m <- messages
  if m.content like "%bay%"
} yield m

val bayMentioned: DBIO[Boolean] =
  containsBay.exists.result
```

`containsBay` クエリは "bay" に言及しているすべてのメッセージを返します。
そして、このクエリを `bayMentioned` 式の中で使って、何を実行するかを決定します。

上記は、以下のような SQL を生成します。

~~~ sql
select exists(
  select "sender", "content", "id"
  from "message"
  where "content" like '%bay%'
)
~~~

[第3章](#moreControlOverInserts)でより便利な例を見ることができます。


## Converting Queries to Actions

クエリを実行する前に、クエリを *action* に変換する必要があります。
これは通常、クエリに対して `result` メソッドを呼び出すことで行います。
アクションはクエリのシーケンスを表します。まず、単一のクエリを表すアクションから始めて 、それらを組み合わせて複数のアクションを形成します。


アクションは `DBIOAction[R, S, E]` という型シグネチャを持ちます。3つの型パラメータは以下の通りです。

- `R` はデータベースから返されるデータの種類 (`Message`、`Person` など)。

- `S` は結果がストリームされるか (`Streaming[T]`) またはされないか (`NoStream`)

- `E` はエフェクトの種類で、推論されます。

多くの場合、アクションの表現を単純化して、`DBIO[T]`とすることができます。これは、`DBIOAction[T, NoStream, Effect.All]` のエイリアスです。

<div class="callout callout-info">

**Effects**

エフェクトはEssential Slickの範囲ではないので、このテキストの大部分では `DBIO[T]` の観点から作業することになります。

しかし、大まかに言えば、`Effect`はアクションにアノテーションを付ける方法です。
例えば、 `Read` や `Write` とマークされたクエリのみを受け付けるメソッドを書いたり、 `Read with Transactional` のような組み合わせを書いたりすることができます。

Slick の `Effect` オブジェクトで定義されているエフェクトは、以下のとおりです。

- データベースから読み込むクエリには `Read` を指定します。
- データベースへの書き込みを行うクエリには `Write` を指定します。
- `Schema` はスキーマに対するエフェクトを表します。
- トランザクションエフェクトには `Transactional` を指定します。
- `All` は上記のすべてに対応します。

Slick はクエリのエフェクトを推論します。例えば、`messages.result`は次のようになります。

~~~ scala
DBIOAction[Seq[String], NoStream, Effect.Read]
~~~

次の章では、挿入と更新について見ていきます。この場合、更新のために推論されるエフェクトは `DBIOAction[Int, NoStream, Effect.Write]` となります。

また、既存の型を拡張して、独自の `Effect` 型を追加することもできます。
</div>

## Executing Actions

アクションを実行するには、`db` オブジェクトの 2 つのメソッドのうちの 1 つにアクションを渡します。

- `db.run(...)` はアクションを実行し、すべての結果をひとつのコレクションにまとめて返します。
  これらは _materialized_ result として知られています。

- `db.stream(...)` はアクションを実行し、その結果を `Stream` に格納して返します。
  これにより、大量のメモリを消費することなく、大きなデータセットを逐次的に処理することができます。

この本では、materializedクエリのみを扱うことにします。
`db.run` はアクションの最終的な結果を `Future` として返します。
呼び出す際には、`ExecutionContext` がスコープ内になければなりません。


```scala mdoc
import scala.concurrent.ExecutionContext.Implicits.global

val futureMessages = db.run(messages.result)
```

<div class="callout callout-info">

**Streaming**

この本では、materializedクエリのみを扱います。
代替手段を知っておくために、ストリームを簡単に見てみましょう。

`db.stream` を呼び出すと、 `Future` の代わりに [`DatabasePublisher`][link-source-dbPublisher] オブジェクトが返されます。
これは、ストリームを操作するための3つのメソッドを公開しています。

- `subscribe` は Akka との統合を可能にします。
- `mapResult`: 指定された関数を元のPublisherから取得した結果にマッピングする `Publisher` を新規に作成します。
- 結果に対してサイドエフェクトを実行するための `foreach` です。

ストリーミングの結果は、[reactive streams][link-reactive-streams] のフィードとして使うことができます。
または [Akka streams or actors][link-akka-streams] に供給するために使用されます。あるいは
`foreach`を使用して、結果を `println` するような単純なこともできます。


```scala
db.stream(messages.result).foreach(println)
```

...これは最終的に各行を表示します。

この分野を探求したいのであれば、[Slick documentation on streaming][link-slick-streaming] から始めてみてください。
</div>

## Column Expressions

`filter` や `map` などのメソッドでは、テーブルのカラムをもとにした式を作成する必要があります。
`Rep` 型は、個々のカラムだけでなく、式を表現するためにも使用されます。
Slick では、式を構築するために `Rep` に様々な拡張メソッドを提供しています。

ここでは、最も一般的なメソッドについて説明します。
完全なリストは Slick のコードベースの [ExtensionMethods.scala][link-source-extmeth] にあります。

### Equality and Inequality Methods

メソッド `===` と `=!=` は任意の型の `Rep` に対して操作し、 `Rep[Boolean]` を生成します。
以下はその例です。

```scala mdoc
messages.filter(_.sender === "Dave").result.statements

messages.filter(_.sender =!= "Dave").result.statements.mkString
```

`<`、`>`、`<=` および `>=` メソッドは、任意の型の `Rep` に対して操作することができます (数値の列に限りません)。

```scala mdoc
messages.filter(_.sender < "HAL").result.statements

messages.filter(m => m.sender >= m.content).result.statements
```


-------------------------------------------------------------------------
Scala Code       Operand Types        Result Type        SQL Equivalent
---------------- -------------------- ------------------ ----------------
`col1 === col2`  `A` or `Option[A]`   `Boolean`          `col1 = col2`

`col1 =!= col2`  `A` or `Option[A]`   `Boolean`          `col1 <> col2`

`col1 < col2`    `A` or `Option[A]`   `Boolean`          `col1 < col2`

`col1 > col2`    `A` or `Option[A]`   `Boolean`          `col1 > col2`

`col1 <= col2`   `A` or `Option[A]`   `Boolean`          `col1 <= col2`

`col1 >= col2`   `A` or `Option[A]`   `Boolean`          `col1 >= col2`

-------------------------------------------------------------------------

: Repの比較メソッド。
  オペランドと結果の型は `Rep[_]` のパラメータとして解釈される必要があります。 

### String Methods

Slick は、文字列の連結のための `++` メソッド (SQL の `||` 演算子) を提供します。

```scala mdoc
messages.map(m => m.sender ++ "> " ++ m.content).result.statements.mkString
```

SQL の古典的な文字列パターンマッチングのための `like` メソッドです。

```scala mdoc
messages.filter(_.content like "%pod%").result.statements.mkString
```

また、Slick は `startsWith`, `length`, `toUpperCase`, `trim` などのメソッドも提供しています。
これらは DBMS によって実装が異なるので、以下の例は単に説明のためのものです。

---------------------------------------------------------------------
Scala Code              Result Type        SQL Equivalent
----------------------- ------------------ --------------------------
`col1.length`           `Int`              `char_length(col1)`

`col1 ++ col2`          `String`           `col1 || col2`

`c1 like c2`            `Boolean`          `c1 like c2`

`c1 startsWith c2`      `Boolean`          `c1 like (c2 || '%')`

`c1 endsWith c2`        `Boolean`          `c1 like ('%' || c2)`

`c1.toUpperCase`        `String`           `upper(c1)`

`c1.toLowerCase`        `String`           `lower(c1)`

`col1.trim`             `String`           `trim(col1)`

`col1.ltrim`            `String`           `ltrim(col1)`

`col1.rtrim`            `String`           `rtrim(col1)`

--------------------------------------------------------------------------------------------------------

: 文字列カラムのメソッド。
  オペランド (例: `col1`, `col2`) は `String` または `Option[String]` でなければなりません。
  オペランドと結果の型は、 `Rep[_]` のパラメータとして解釈される必要があります。

### Numeric Methods {#NumericColumnMethods}


Slick は、数値を持つ `Rep` を操作するための包括的なメソッド群を提供します。`Int`, `Long`, `Double`, `Float`, `Short`, `Byte`, そして `BigDecimal` といった数値を持つ `Rep` を操作するためのメソッドが用意されています。

--------------------------------------------------------------------------------------------------------
Scala Code              Operand Column Types               Result Type        SQL Equivalent
----------------------- ---------------------------------- ------------------ --------------------------
`col1 + col2`           `A` or `Option[A]`                 `A`                `col1 + col2`

`col1 - col2`           `A` or `Option[A]`                 `A`                `col1 - col2`

`col1 * col2`           `A` or `Option[A]`                 `A`                `col1 * col2`

`col1 / col2`           `A` or `Option[A]`                 `A`                `col1 / col2`

`col1 % col2`           `A` or `Option[A]`                 `A`                `mod(col1, col2)`

`col1.abs`              `A` or `Option[A]`                 `A`                `abs(col1)`

`col1.ceil`             `A` or `Option[A]`                 `A`                `ceil(col1)`

`col1.floor`            `A` or `Option[A]`                 `A`                `floor(col1)`

`col1.round`            `A` or `Option[A]`                 `A`                `round(col1, 0)`

--------------------------------------------------------------------------------------------------------

: 数値カラムのメソッド。
  オペランドと結果の型は `Rep[_]` のパラメータとして解釈される必要があります。

### Boolean Methods

また、Slickはboolean値の `Rep` を操作するメソッド群も提供しています。

--------------------------------------------------------------------------------------------------------
Scala Code              Operand Column Types               Result Type        SQL Equivalent
----------------------- ---------------------------------- ------------------ --------------------------
`col1 && col2`           `Boolean` or `Option[Boolean]`    `Boolean`          `col1 and col2`

`col1 || col2`           `Boolean` or `Option[Boolean]`    `Boolean`          `col1 or col2`

`!col1`                  `Boolean` or `Option[Boolean]`    `Boolean`          `not col1`

--------------------------------------------------------------------------------------------------------

: Bool型カラムのメソッド。
  オペランドと結果の型は `Rep[_]` のパラメータとして解釈される必要があります。

### Date and Time Methods

Slick は以下のカラムマッピングを提供します。Slick では、 `Instant`, `LocalDate`, `LocalTime`, `LocalDateTime`, `OffsetTime`, `OffsetDateTime`, そして `ZonedDateTime` のカラムマッピングを提供しています。
つまり、これらの型はすべて、テーブル定義のカラムとして使用することができるのです。

カラムがどのようにマッピングされるかは、使用しているデータベースに依存します。
データベースによって、時刻と日付に関する機能は異なるからです。
下の表は、一般的な 3 つのデータベースで使用される SQL 型を示しています。

--------------------------------------------------------------------------
Scala Type           H2 Column Type   PostgreSQL         MySQL
-------------------- ---------------- ----------------- ------------------
`Instant`            `TIMESTAMP`      `TIMESTAMP`        `TEXT`

`LocalDate`          `DATE`           `DATE`             `DATE`

`LocalTime`          `VARCHAR`        `TIME`             `TEXT`

`LocalDateTime`      `TIMESTAMP`      `TIMESTAMP`        `TEXT`

`OffsetTime`         `VARCHAR`        `TIMETZ`           `TEXT`

`OffsetDateTime`     `VARCHAR`        `VARCHAR`          `TEXT`

`ZonedDateTime`      `VARCHAR`        `VARCHAR`          `TEXT`

--------------------------------------------------------------------------

: 3つのデータベースの `java.time` 型から SQL カラム型へのマッピング。
  [Slick 3.3 Upgrade Guide][link-slick-ug-time] に完全なリストがあります。

文字列型やブール型とは異なり、 `java.time` 型には特別なメソッドはありません。
しかし、すべての型が等価メソッドを持っているので、 `===`, `>`, `<=` などを日付や時刻の型に対して期待通りに使用することができます。

### Option Methods and Type Equivalence {#type_equivalence}

Slick は SQL で nullableなカラムを `Rep` と `Option` 型でモデル化します。
これについては [5章](#Modelling) で詳しく説明します。
しかし、プレビューとして、データベースに nullableなカラムがある場合、それを `Table` でオプションとして宣言することを知っておいてください。
~~~ scala
final class PersonTable(tag: Tag) /* ... */ {
  // ...
  def nickname = column[Option[String]]("nickname")
  // ...
}
~~~

オプショナルな値に対するクエリになると
Slickは型の等価性に関してかなり賢いです。

型の等価性とはどういう意味でしょうか？
Slickは、オペランドが互換性のある型であることを確認するために、列式の型チェックを行います。
例えば、 `String` と `Int` を等価に比較することはできますが、 `String` と `Int` を比較することはできません。

```scala mdoc:fail
messages.filter(_.id === "foo")
```

興味深いことに、Slick は数値型に対して非常に細かいです。
例えば、 `Int` と `Long` を比較すると、型エラーとみなされます。

```scala mdoc:fail
messages.filter(_.id === 123)
```

裏を返せば
Slick はオプションと非オプションのカラムの等価性に関して賢いです。
オペランドが `A` と `Option[A]` 型の組み合わせである限り (`A` の値が同じであれば)、クエリは通常コンパイルされます。

```scala mdoc
messages.filter(_.id === Option(123L)).result.statements
```

ただし、オプションの引数は `Some` や `None` ではなく、厳密に `Option` 型でなければなりません。

```scala mdoc:fail
messages.filter(_.id === Some(123L)).result.statements
```

このような状況に陥った場合、値に型付けをすることができることを忘れないでください。

```scala mdoc
messages.filter(_.id === (Some(123L): Option[Long]) )
```


## Controlling Queries: Sort, Take, and Drop

クエリから返される結果の順番と数を制御するために使用される三種類の関数があります。
これは結果セットのページネーションに最適ですが、以下の表にあるメソッドは単独で使用することもできます。

-------------------------------------------
Scala Code             SQL Equivalent
---------------- --------------------------
`sortBy`         `ORDER BY`

`take`           `LIMIT`

`drop`           `OFFSET`

-------------------------------------------------

: クエリの結果を順序付け、スキップ、制限するためのメソッド。 

まず `sortBy` の例から順番に見ていきましょう。例えば、メッセージを送信者の名前の順に並べたいとします。

```scala mdoc
exec(messages.sortBy(_.sender).result).foreach(println)
```

逆順だとこのようになります。

```scala mdoc
exec(messages.sortBy(_.sender.desc).result).foreach(println)
```



複数の列でソートする場合は、列のタプルを返します。

```scala mdoc
messages.sortBy(m => (m.sender, m.content)).result.statements
```

さて、結果の並べ替えの方法がわかったところで、最初の5行だけを表示させたいとします。

```scala mdoc
messages.sortBy(_.sender).take(5)
```

ページ単位で情報を提示するのであれば、次のページ（6～10行目）を表示する方法が必要になります。

```scala mdoc
messages.sortBy(_.sender).drop(5).take(5)
```

これは以下のSQLと等価です。

~~~ sql
select "sender", "content", "id"
from "message"
order by "sender"
limit 5 offset 5
~~~~

<div class="callout callout-info">

**Sorting on Null columns**

この章の前のほうで、[オプションメソッドと型の等価性](#type_equivalence) を見たときに、nullable なカラムについて簡単に紹介しました。
Slick では、 `desc` や `asc` と組み合わせて使用できる 3 つの修飾子を提供しています。`nullFirst`, `nullsDefault`, `nullsLast` です。
これらは、結果セットの最初または最後に null を含めることで、期待通りの結果を得ることができます。
`nullsDefault` の動作は、SQL エンジンの優先順位を使用します。

今までの例では、まだ nullableなフィールドはありません。
しかし、ここでは、nullable なカラムのソートがどのようなものかを見てみましょう。

```scala
users.sortBy(_.name.nullsFirst)
```

上記のクエリに対して生成されるSQLは次のようになります。

~~~ sql
select "name", "email", "id"
from "user"
order by "name" nulls first
~~~

[5章](#Modelling)でnullableカラムを取り上げ、[example project][link-example] でnullableカラムでのソートの例を紹介しています。このコードは _chapter-05_ フォルダの _nulls.scala_ にあります。
</div>


## Conditional Filtering

これまで、`map`、`filter`、`take` などのクエリオペレーションを見てきました。
そして、後の章では結合と集約を見ることになります。
Slick を使って行う作業の多くは、これらの数個の操作に限られることでしょう。

他にも、 `filterOpt` と `filterIf` という2つのメソッドがあります。
これは、動的なクエリで、何らかの条件に基づいて行をフィルタリングしたい場合に役立ちます。

例えば、クルーメンバー（メッセージの送信者）でフィルタリングするオプションをユーザーに提供したいとします。
つまり、クルーメンバーを指定しなければ、全員のメッセージを取得することになります。

最初の試みは次のようになります。

```scala mdoc
def query1(name: Option[String]) =
  messages.filter(msg => msg.sender === name)
```

これは有効なクエリですが、`None` を与えると、すべての結果が得られるのではなく、結果が得られなくなります。
クエリにさらにチェックを追加して、例えば `|| name.isEmpty` も追加することができます。
しかし、私たちがしたいことは、値があるときだけフィルタリングすることです。そして、それを行うのが `filterOpt` です。


```scala mdoc
def query2(name: Option[String]) =
  messages.filterOpt(name)( (row, value) => row.sender === value )
```

このクエリは次のように読むことができます:
- オプションで `name` をフィルタリングします。
- そして、もし `name` に値があれば、その値を使ってクエリの `row` をフィルタリングすることができます。

その結果、クルーメンバーが存在しないときは、SQLに条件がないことになります。

```scala mdoc
query2(None).result.statements.mkString
```

そして、存在する場合は、その条件が適用されます。

```scala mdoc
query2(Some("Dave")).result.statements.mkString
```

<div class="callout callout-info">
一度 `filterOpt` を使うようになると、ショートハンド版を使うのが好ましいかもしれません。

```scala mdoc
def queryShortHand(name: Option[String]) =
  messages.filterOpt(name)(_.sender === _)
```

`query` の挙動は、この短いバージョンでも、本文で使用した長いバージョンでも同じです。
</div>

`filterIf` は似たような機能ですが、where条件のオン・オフを切り替えます。
例えば、"古い "メッセージを除外するオプションをユーザに与えることができます。

```scala mdoc
val hideOldMessages = true
val queryIf = messages.filterIf(hideOldMessages)(_.id > 100L)
queryIf.result.statements.mkString
```

ここでは、`hideOldMessages`が`true`なので、`ID > 100`という条件がクエリに追加されています。
もし false ならば、このクエリには where 節が含まれません。

`filterIf` と `filterOpt` の大きな利点は、それらを次々に連結して、簡潔な動的クエリを構築できることです。


```scala mdoc
val person = Some("Dave")

val queryToRun = messages.
  filterOpt(person)(_.sender === _).
  filterIf(hideOldMessages)(_.id > 100L)

queryToRun.result.statements.mkString
```


## Take Home Points

`TableQuery` から始めると、 `filter` と `map` を使って様々なクエリを作成することができます。
これらのクエリを構成する際に、 `Query` の型が追従するので、アプリケーション全体で型安全性を確保することができます。

クエリで使用する式は、拡張メソッドで定義されています。
クエリで使用する式は、 `Rep` の型に応じて、 `===`, `=!=`, `like`, `&&` などの拡張メソッドで定義されます。
`Option` 型との比較は、Slick が `Rep[T]` と `Rep[Option[T]]` を自動的に比較するので、簡単に行うことができます。

これまで、 `map` が SQL の `select` のように振る舞い、 `filter` が `WHERE` のように振る舞うことを見てきました。
[第6章](#joins)では、 `GROUP` と `JOIN` のSlickな表現について見ていきます。

新しい用語もいくつか紹介しました。

* _unpacked_ 型:  `String` のような通常のScalaの型です。

* _mixed_ 型: `Rep[String]`のようなSlickの列表現です。

クエリは `result` メソッドを用いてアクションに変換して実行します。
アクションをデータベースに対して実行するには、 `db.run` を使用します。

データベースアクションのコンストラクタ `DBIOAction` は、結果、ストリーミングモード、エフェクトを表す 3 つの引数を取ります。
`DBIO[R]`はこれを結果型だけに単純化したものです。

これまで見てきたクエリの作成方法は、 `update` や `delete` を使ってデータを変更する際に役立ちます。
これは次の章のトピックです。


## Exercises

この章で出てきたコードを試していない方は試してみてください。
[サンプルプロジェクト][link-example]では、コードは _chapter-02_ フォルダの _main.scala_ にあります。

それができたら、以下の練習問題に取り組んでみてください。
SBTで _triggered execution_ を使うと、簡単に試すことができます。

~~~ bash
$ cd example-02
$ sbt
> ~run
~~~

`~run` は、プロジェクトに変更がないかどうかを監視します。
変更が見られるとプログラムがコンパイルされて実行されます。
つまり、_main.scala_ を編集して、ターミナルウィンドウでその出力を見ることができるのです。

### Count the Messages

メッセージの数はどうやって数えるのでしょうか？

ヒント：Scala のコレクションでは、 `length` メソッドがコレクションのサイズを教えてくれます


<div class="solution">

```scala mdoc
val results = exec(messages.length.result)
```

`length`のエイリアスである `size` を使用することもできます。
```scala mdoc:invisible
messages.size
```
</div>

### Selecting a Message

for式を使ってid=1を持つメッセージを選択しましょう。
またidが999のメッセージを探そうとするとどうなりますか？

ヒント：私たちのidは `Long` です。
Scalaでは数字の後に`L`をつけると、例えば`99L`のように、longになります。


<div class="solution">

```scala mdoc
val id1query = for {
  message <- messages if message.id === 1L
} yield message

val id1result = exec(id1query.result)
```

存在しないidである `999` を要求すると、空のコレクションが返されます。

```scala mdoc:invisible
{
  val nnn = messages.filter(_.id === 999L)
  val rows = exec(nnn.result)
  assert(rows.isEmpty, s"Expected empty rows for id 999 in ex2, not $rows")
}
```
</div>

### One Liners

前回の練習問題のクエリを、for式を使わないように書き直してみましょう。
あなたはどちらのスタイルが好きですか？それはなぜですか？

<div class="solution">

```scala mdoc
val filterResults = exec(messages.filter(_.id === 1L).result)
```
</div>

### Checking the SQL

クエリに対して `result.statements` メソッドを呼び出すと、実行する SQL が得られます。
それを前回の演習に当てはめてみましょう。
どのようなクエリが得られますか？
このことから、`filter` が SQL にマップされていることについて何がわかるでしょうか？

<div class="solution">
このようなコードを実行する必要があります。

```scala mdoc
val sql = messages.filter(_.id === 1L).result.statements
println(sql.head)
```

`filter` が SQL の `where` 節にどのように対応するかがわかりますね。

</div>

### Is HAL Real?

データベースにHALからのメッセージがあるかどうか確認してみましょう。
ただし、データベースからブール値を返すだけです（存在するか確認するだけ）。


<div class="solution">

そうです、私たちはHALが「存在する」のかどうかを知りたいのです。

```scala mdoc
val queryHalExists = messages.filter(_.sender === "HAL").exists

exec(queryHalExists.result)
```

```scala mdoc:invisible
{
val found = exec(queryHalExists.result)
assert(found, s"Expected to find HAL, not: $found")
}
```

このクエリは、HALからのレコードがあるので、`true`を返します。
そして、Slickは以下のSQLを生成します。

```scala mdoc
queryHalExists.result.statements.head
```
</div>


### Selecting Columns

これまでは、 `Message` クラス、ブール値、またはカウント値を返してきました。
今度は、データベース内のすべてのメッセージを選択し、その `content` カラムだけを返すようにしたいと思います。

ヒント: メッセージをコレクションとして考え、コレクションに対して case クラスの単一のフィールドを返すために何をするのかを考えてみましょう。

このクエリでどのような SQL が実行されるかを確認してみましょう。

<div class="solution">

```scala mdoc
val contents = messages.map(_.content)
exec(contents.result)
```

以下のようにも書けます。

```scala mdoc
val altQuery = for { message <- messages } yield message.content
```

このクエリは、データベースから `content` カラムのみを返します。

```scala mdoc
altQuery.result.statements.head
```
</div>


### First Result

メソッド `head` と `headOption` は、 `result` に対する便利なメソッドです。
HAL が最初に送信したメッセージを探しましょう。

もし `head` を使って "Alice" からのメッセージを見つけたらどうなるでしょうか (Alice は何もメッセージを送っていないことに注意してください)。

<div class="solution">

```scala mdoc
val msg1 = messages.filter(_.sender === "HAL").map(_.content).result.head
```

"Affirmative, Dave. I read you." を生成するアクションを得られたはずです。

Alice の場合、空のコレクションの先頭を返そうとしているので、 `head` は実行時例外を発生させます。`headOption` を使用すると、例外を回避できます。


```scala mdoc
exec(messages.filter(_.sender === "Alice").result.headOption)
```
</div>

### Then the Rest

前回の練習では、HAL が最初に送ったメッセージを返しました。
今回は、HAL が次に送った 5 つのメッセージを探してください。
どんなメッセージが返ってくるでしょうか？

もし、HALの10番目から20番目までのメッセージを求めたらどうでしょうか？

<div class="solution">

`drop` と `take` を利用します

```scala mdoc
val msgs = messages.filter(_.sender === "HAL").drop(1).take(5).result
```

HALは全部で2つのメッセージしか持っていません。
したがって、私たちの結果セットには1つのメッセージが含まれるはずです。

```scala
Message(HAL,I'm sorry, Dave. I'm afraid I can't do that.,4)
```

```scala mdoc:invisible
{
  val nextFive = exec(msgs)
  assert(nextFive.length == 1, s"Expected 1 msgs, not: $nextFive")
}
```

これ以上メッセージを求めると、空のコレクションになってしまいます。

```scala mdoc
val allMsgs = exec(
            messages.
              filter(_.sender === "HAL").
              drop(10).
              take(10).
              result
          )
```

```scala mdoc:invisible
{
  assert(allMsgs.length == 0, s"Expected 0 msgs, not: $msgs")
}
```

</div>


### The Start of Something

`String` の `startsWith` メソッドは、その文字列が特定の文字列で始まるかどうかをテストします。
Slickでは、文字列のカラムに対してもこれを実装しています。

"Open "で始まるメッセージを検索しましょう。
そのクエリはSQLでどのように実装されていますか？


<div class="solution">

```scala mdoc
messages.filter(_.content startsWith "Open")
```

このクエリは `LIKE` という句で実装されています。

```scala mdoc
messages.filter(_.content startsWith "Open").result.statements.head
```
</div>

### Liking

Slick は `like` というメソッドを実装しています。
コンテンツに "do" を含むすべてのメッセージを検索してください。

大文字と小文字を区別しないようにできるでしょうか？

<div class="solution">

SQL の `like` 式に慣れていれば、このクエリの大文字小文字を区別するバージョンを見つけるのはそれほど難しくないでしょう。

```scala mdoc
messages.filter(_.content like "%do%")
```

大文字と小文字を区別しないようにするには、`content` フィールドで `toLowerCase` を使用するとよいでしょう。

```scala mdoc
messages.filter(_.content.toLowerCase like "%do%")
```

これは、`content` が `Rep[String]` であり、その `Rep` が `toLowerCase` を実装しているからです。
つまり、 `toLowerCase` は意味のある SQL に変換されます。

結果は3つあります。"_Do_ you read me", "Open the pod bay *do*ors", そして "I'm afraid I can't _do_ that" の3つになります。


```scala mdoc:invisible
{
  val likeDo = exec( messages.filter(_.content.toLowerCase like "%do%").result )

  assert(likeDo.length == 3, s"Expected 3 results, not $likeDo")
}
```
</div>

### Client-Side or Server-Side?

これは何をするものですか？ そしてなぜですか？

```scala
exec(messages.map(_.content.toString + "!").result)
```

<div class="solution">

Slickが生成するクエリは、次のようなものです。

```sql
select '(message Ref @421681221).content!' from "message"
```

```scala mdoc:invisible
{
  val weird = exec(messages.map(_.content.toString + "!").result).head
  assert(weird contains "Ref", s"Expected 'Ref' inside $weird")
}
```

これは奇妙な定数文字列のselect式です。

`_.content.toString + "!"` 式は `content` を文字列に変換し、感嘆符を追加します。
`content` とは何でしょうか？それは `Rep[String]` であり、コンテンツの `String` ではありません。
結局のところ、Slickの内部動作のようなものが見えていることになります。

このマッピングをデータベースで行うことはSlickでも可能です。
このとき、`Rep[T]` クラスという単位で作業することを忘れないようにしましょう。


```scala mdoc
messages.map(m => m.content ++ LiteralColumn("!"))
```

ここで `LiteralColumn[T]` は `Rep[T]` の型であり、SQL に挿入する定数値を保持するためのものである。
`++`メソッドは、任意の `Rep[String]` に対して定義された拡張メソッドの 1 つである。

`++`を使うと、希望通りのクエリが作成されます。


```sql
select "content"||'!' from "message"
```

このようにも書けます。

```scala mdoc
messages.map(m => m.content ++ "!")
```

`"!"` は `Rep[String]` にリフティングされました。

この演習では、 `map` や `filter` の内部では、 `Rep[T]` という単位で作業していることに注目してください。
利用できる操作に慣れる必要があります。
この章に含まれている表は、その助けとなるはずです。

</div>
