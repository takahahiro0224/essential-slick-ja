# Data Modelling {#Modelling}

データベースへの接続、クエリーの実行、データの変更といった基本的な操作ができるようになりました。次は、より豊かなデータモデルと、アプリケーションの連携について説明します。


この章では以下について学習します。

- アプリケーションの構成方法を理解できる 

- case classで列をモデル化することの代替案を検討する

- より豊富なデータ型をカラムに格納する 

- テーブルのモデリングに関する知識を深め、オプション値や外部キーを導入する

上記のために、チャットアプリケーションのスキーマを拡張して、メッセージ以外にも対応できるようにします。

## Application Structure

これまでのところ、すべてのサンプルは1つのScalaファイルで書かれています。この方法は、より大きなアプリケーションのコードベースにはスケールしません。このセクションでは、アプリケーションコードをモジュールに分割する方法について説明します

また、これまではSlickのH2プロファイルを独占的に使用してきました。実際のアプリケーションを書くときには、異なる状況でプロファイルを切り替えられるようにする必要があることがよくあります。例えば、本番環境ではPostgreSQLを使い、ユニットテストではH2を使うことがあります。

このパターンの例は、[サンプルプロジェクト][link-example]の _chapter-05_ の _structure.scala_ で見ることができます。


### Abstracting over Databases

ここでは、複数の異なるデータベースプロファイルで動作するコードを書く方法について見てみましょう。以前書いたときは...

```scala mdoc:silent
import slick.jdbc.H2Profile.api._
```


...私たちはH2に自分自身をロックしていました。今回は様々なプロファイルで動作するインポートを書きたい。
幸い、Slickはプロファイルの共通スーパータイプである`JdbcProfile`と呼ばれるトレイトを提供しています。


```scala mdoc:silent
import slick.jdbc.JdbcProfile
```

`JdbcProfile`は具象オブジェクトではないため、直接インポートすることはできません。その代わりに、アプリケーションに`JdbcProfile`型の依存関係を注入し、そこからインポートする必要があります。私たちが使用する基本的なパターンは、次のとおりです。

* データベース実装を1つ（またはいくつかの）のトレイトに分離する

* Slickプロファイルをabstract `val` として宣言し、そこからインポートする

* プロファイルを具体化するためにデータベーストレイトを拡張する 

以下がこのパターンのシンプルな形になります。

```scala mdoc:silent
trait DatabaseModule {
  // Declare an abstract profile:
  val profile: JdbcProfile

  // Import the Slick API from the profile:
  import profile.api._

  // Write our database code here...
}

object Main1 extends App {
  // Instantiate the database module, assigning a concrete profile:
  val databaseLayer = new DatabaseModule {
    val profile = slick.jdbc.H2Profile
  }
}
```

このパターンでは、abstract`val`を使用してプロファイルを宣言します。これは、`import profile.api._` と書くので十分です。
コンパイラは、valがイミュータブルな`JdbcProfile`であることを認識しています（どのプロファイルかはまだ言っていません）。
`DatabaseModule`をインスタンス化するときに、`profile`を選択したプロファイルにバインドします。


### Scaling to Larger Codebases

アプリケーションの規模が大きくなると、コードを複数のファイルに分割して管理する必要があります。
これを実現するには、上記のパターンを拡張して、トレイトファミリーにする必要があります。



```scala mdoc:silent
trait Profile {
  val profile: JdbcProfile
}

trait DatabaseModule1 { self: Profile =>
  import profile.api._

  // Write database code here
}

trait DatabaseModule2 { self: Profile =>
  import profile.api._

  // Write more database code here
}

// Mix the modules together:
class DatabaseLayer(val profile: JdbcProfile) extends
  Profile with
  DatabaseModule1 with
  DatabaseModule2

// Instantiate the modules and inject a profile:
object Main2 extends App {
  val databaseLayer = new DatabaseLayer(slick.jdbc.H2Profile)
}
```

ここでは、`profile`の依存関係をそれ自身の`Profile`トレイトに分解しています。
データベース実装の各モジュールは、`Profile`を自分型として指定します。
つまり、`Profile`を拡張するクラスによってのみ拡張することができます。
これにより、モジュール群全体で`profile`を共有することができます。

異なるデータベースで動作させるために、データベース実装をインスタンス化する際に、異なるプロファイルを注入します：

```scala mdoc
val anotherDatabaseLayer = new DatabaseLayer(slick.jdbc.PostgresProfile)
```

この基本パターンは、アプリケーションを構成する上で合理的な方法です。

<!--
### Namespacing Queries

We can exploit the expanded form of `TableQuery[T]`, a macro, to provide a location to store queries.
`TableQuery[T]`'s expanded form is:

~~~ scala
(new TableQuery(new T(_)))`
~~~

Using this, we can provide a module to hold `Message` queries:

~~~ scala
object messages extends TableQuery(new MessageTable(_)) {

  def messagesFrom(name: String) =
    this.filter(_.sender === name)

  val numSenders = this.map(_.sender).distinct.length
}
~~~

This adds values and methods to `messages`:

~~~ scala
val action =
  messages.numSenders.result
~~~
-->

## Representations for Rows

前の章では、行をケースクラスでモデリングしました。
これは一般的な使用パターンであり、我々も推奨している方法ですが、タプル、ケースクラス、`HList`など、いくつかの表現方法があります。
ここでは、Slickがデータベースのカラムとクラスのフィールドをどのように関連付けるかについて、より詳細に調べることにしましょう。


### Projections, `ProvenShapes`, `mapTo`, and `<>`

Slickでテーブルを宣言する際、"default projection"を指定する`*`メソッドを実装することが必須になっています。

```scala mdoc:silent
class MyTable(tag: Tag) extends Table[(String, Int)](tag, "mytable") {
  def column1 = column[String]("column1")
  def column2 = column[Int]("column2")
  def * = (column1, column2)
}
```

<div class="callout callout-info">

**Expose Only What You Need**

行の定義から情報を除外することで、情報を隠すことができます。デフォルトプロジェクションは、何をどのような順序で返すかを制御し、行の定義に従います。

例えば、使われていないレガシーなカラムを持つテーブルのすべてをマッピングする必要はないのです。

</div>

プロジェクションは、データベースのカラムとScalaの値との間のマッピングを提供します。
上記のコードでは、`*`の定義は、データベースから`extends Table`句で定義された`(String, Int)`タプルに`column1`と`column2`をマッピングすることです。

Tableクラスの`*`の定義を見てみると、紛らわしいことが書いてあります。


~~~ scala
abstract class Table[T] {
  def * : ProvenShape[T]
}
~~~

`*` の型は、実際には`ProvenShape`と呼ばれるもので、サンプルで指定したような列のタプルではありません。
ここでは明らかに他のことが行われています。
Slickは暗黙の変換を使用して、私たちが提供した列から`ProvenShape`オブジェクトを構築しています。

`ProvenShape`の内部構造については、本書の範囲を超えていることは確かです。
Slickは、互換性のある`ProvenShape`を生成できるのであれば、どんなScalaの型でもプロジェクションとして使用できることは言うまでもありません。
`ProvenShape`生成のルールを見てみると、どのようなデータ型をマッピングできるのかがわかると思います。
ここでは、最も一般的な3つのユースケースを紹介します。


```scala mdoc:reset:invisible
import slick.jdbc.H2Profile.api._
import scala.concurrent.{Await,Future}
import scala.concurrent.duration._
val db = Database.forConfig("chapter05")
def exec[T](action: DBIO[T]): T = Await.result(db.run(action), 4.seconds)
import scala.concurrent.ExecutionContext.Implicits.global

// Because I can't get the code in the enumerated list to render and indent nicely with tut yet.
// ...and I've not tried with mdoc

// 1
    class MyTable1(tag: Tag) extends Table[String](tag, "mytable") {
      def column1 = column[String]("column1")
      def * = column1
    }

// 2
  class MyTable2(tag: Tag) extends Table[(String, Int)](tag, "mytable") {
    def column1 = column[String]("column1")
    def column2 = column[Int]("column2")
    def * = (column1, column2)
  }

// 3
    case class User(name: String, id: Long)
    class UserTable3(tag: Tag) extends Table[User](tag, "user") {
      def id   = column[Long]("id", O.PrimaryKey, O.AutoInc)
      def name = column[String]("name")
      def * = (name, id).<>(User.tupled, User.unapply)
    }
```

1. 単一の列の定義は、列の値を列の型パラメーターの値にマッピングするシェイプを生成します。例えば、`Rep[String]`の列は、`String`型の値をマッピングします。

    ```scala
    class MyTable1(tag: Tag) extends Table[String](tag, "mytable") {
      def column1 = column[String]("column1")
      def * = column1
    }
    ```
    <!-- If you change this ^^ code update the invisible block above -->

2. データベースカラムのタプルは、その型パラメーターのタプルをマッピングします。例えば、`(Rep[String], Rep[Int])` は `(String, Int)` にマップされます。
    
   ```scala
    class MyTable2(tag: Tag) extends Table[(String, Int)](tag, "mytable") {
      def column1 = column[String]("column1")
      def column2 = column[Int]("column2")
      def * = (column1, column2)
    }
    ```
    <!-- If you change this ^^ code update the invisible block above -->

3. `ProvenShape[A]`があれば、"projection operator"の`<>`を使って`ProvenShape[B]`に変換することができます。このサンプルでは、`A`を`String`と`Int`のタプルにすると、`ProvenShape[A]`が得られることが分かっています（前の例）。`A`-`B`間の各変換を行う関数を与え、Slickは結果型を構築します。ここでは、`B`は`User`ケースクラスです。

    ```scala
    case class User(name: String, id: Long)

    class UserTable3(tag: Tag) extends Table[User](tag, "user") {
      def id   = column[Long]("id", O.PrimaryKey, O.AutoInc)
      def name = column[String]("name")
      def * = (name, id).<>(User.tupled, User.unapply)
    }
    ```
    <!-- If you change this ^^ code update the invisible block above -->


プロジェクションオペレーター `<>`は、さまざまな型のマッピングを可能にする隠し味です。
カラムのタプルをある型`B`と相互変換できる限り、`B`のインスタンスをデータベースに格納することができます。

今まで`<>`を見なかったのは、`mapTo`マクロがプロジェクションを構築してくれるからです。
ほとんどの場合、`<>`よりも`mapTo`の方が便利で効率的です。
しかし、`<>`は利用可能であり、マッピングをよりコントロールする必要がある場合は知っておく価値があります。
また、Slick 3.2以前に作成されたコードベースでは、この方法をよく目にすることになるでしょう。


`<>` は2つの引数を持ちます。

* `A => B`の関数で、既存の型のアンパックされた行レベルエンコーディング `(String, Long)`から私たちが好む表現である`User`に変換します。
* `B => Option[A]`の関数で, 逆向きに変換します。 

必要であれば、これらの関数を手作業で供給することも可能です。

```scala mdoc:silent
def intoUser(pair: (String, Long)): User =
  User(pair._1, pair._2)

def fromUser(user: User): Option[(String, Long)] =
  Some((user.name, user.id))
```

結果このようになります。

```scala mdoc:silent
class UserTable(tag: Tag) extends Table[User](tag, "user") {
  def id   = column[Long]("id", O.PrimaryKey, O.AutoInc)
  def name = column[String]("name")
  def * = (name, id).<>(intoUser, fromUser)
}
```
`User`の例では、`User.tupled`と`User.unapply`によってケースクラスがこれらの関数を提供しているので、自分たちで構築する必要はありません。
しかし、行のパッケージングとアンパッケージングをより精巧に行うために、独自の関数を提供できることを覚えておくと便利です。
この章の練習問題で、このことを説明します。

このセクションでは、プロジェクションの詳細について見てきました。しかし、一般的には、`mapTo`マクロで多くの場面で十分です。

### Tuples versus Case Classes

Slickがケースクラスと値のタプルをマッピングすることができることを見てきました。
しかし、どちらを使うべきなのでしょうか？ある意味では、ケースクラスとタプルの間にほとんど違いはありません。しかし、ケースクラスは2つの重要な点でタプルと異なります。

まず、ケースクラスにはフィールド名があり、コードの読みやすさを向上させることができます。

```scala mdoc
val dave = User("Dave", 0L)
dave.name // case class field access

val tuple = ("Dave", 0L)
tuple._1 // tuple field access
```

次に、ケースクラスは、同じフィールド型を持つ他のケースクラスと区別する型を持っています。

```scala mdoc:fail
case class Dog(name: String, id: Long)

val user = User("Dave", 0L)
val dog  = Dog("Lassie", 0L)


// Different types (a warning, but when compiled -Xfatal-warnings....)
user == dog
```

このような理由から、データベースの行を表現する際には、原則としてケースクラスを使用することをお勧めします。


### Heterogeneous Lists

Slickがデータベースのテーブルをタプルとケースクラスにマッピングする方法について見てきました。
Scalaのベテランは、このアプローチの重要な弱点であるタプルとケースクラスが22フィールドで限界に達することを認識しています[^scala211-limit22]。


[^scala211-limit22]: Scala 2.11では、22以上のフィールドを持つケースクラスを定義できるようになりましたが、タプルと関数はまだ22に制限されています。
これについては、[ブログ記事](https://underscore.io/blog/posts/2016/10/11/twenty-two.html)にも書いています。

エンタープライズデータベースのレガシーテーブルで、数十から数百のカラムを持つ恐ろしい話を聞いたことがある人は多いでしょう。
これらの行をどのようにマッピングすればよいのでしょうか？
幸い、Slickは非常に多くの列を持つテーブルをサポートする[`HList`][link-slick-hlist]の実装を提供しています。


その動機付けとして、設計が不十分な商品属性を保存するためのレガシーテーブルについて考えてみましょう。

```scala mdoc
case class Attr(id: Long, productId: Long /* ...etc */)

class AttrTable(tag: Tag) extends Table[Attr](tag, "attrs") {
  def id        = column[Long]("id", O.PrimaryKey, O.AutoInc)
  def productId = column[Long]("product_id")
  def name1     = column[String]("name1")
  def value1    = column[Int]("value1")
  def name2     = column[String]("name2")
  def value2    = column[Int]("value2")
  def name3     = column[String]("name3")
  def value3    = column[Int]("value3")
  def name4     = column[String]("name4")
  def value4    = column[Int]("value4")
  def name5     = column[String]("name5")
  def value5    = column[Int]("value5")
  def name6     = column[String]("name6")
  def value6    = column[Int]("value6")
  def name7     = column[String]("name7")
  def value7    = column[Int]("value7")
  def name8     = column[String]("name8")
  def value8    = column[Int]("value8")
  def name9     = column[String]("name9")
  def value9    = column[Int]("value9")
  def name10    = column[String]("name10")
  def value10   = column[Int]("value10")
  def name11    = column[String]("name11")
  def value11   = column[Int]("value11")
  def name12    = column[String]("name12")
  def value12   = column[Int]("value12")

  def * = ??? // we'll fill this in below
}
```

あなたの組織にこのようなテーブルがないことを祈りますが、事故は起こるものです。

このテーブルには26のカラムがあり、フラットなタプルを使ってモデル化するには多すぎます。
幸いなことに、Slickは任意の数のカラムに対応する代替マッピング表現を提供しています。
この表現は、heterogeneous listまたは`HList`[^hlist]と呼ばれます。


[^hlist]: `HList`については、[shapeless][link-shapeless]など他のライブラリで聞いたことがあるかもしれません。
ここでは、Slick独自の`HList`の実装について話しているのであって、shapelessの実装について話しているのではありません。
shapelessの`HList`は[slickless][link-slickless]というライブラリで利用することができます。

`HList`は、リストとタプルのハイブリッドのようなものです。リストのように任意の長さを持つが、タプルのように各要素を異なる型にすることができます。
以下はその例です。


```scala mdoc:silent
import slick.collection.heterogeneous.{HList, HCons, HNil}
import slick.collection.heterogeneous.syntax._
```
```scala mdoc
val emptyHList = HNil

val shortHList: Int :: HNil = 123 :: HNil

val longerHList: Int :: String :: Boolean :: HNil =
  123 :: "abc" :: true :: HNil
```

`HList`は`List`と同様に再帰的に構築されるため、任意の大きさの値の集まりをモデル化することができます。
`HList`s are constructed recursively like `List`s,
allowing us to model arbitrarily large collections of values:

- 空の`HList`は，シングルトンオブジェクト`HNil`で表現される
- より長い`HList`は、`::`演算子を使って値を前置することで形成され、新しい型のリストを作成する

それぞれの`HList`の型と値が互いに影響し合っていることに注意してください。
`longerHList`は`Int`型、`String`型、`Boolean`型の値からなり、その型は`Int`型、`String`型、`Boolean`型からなります。
要素の型が保持されているので、それぞれの正確な型を考慮したコードを書くことができます。

Slickは、カラムの`HList`とその値の`HList`を対応付ける`ProvenShape`を生成することができます。
例えば、`Rep[Int] :: Rep[String] :: HNil`のシェイプは`Int :: String :: HNil`の型の値をマッピングします。

#### Using HLists Directly

We can use an `HList` to map the large table in our example above.
Here's what the default projection looks like:

```scala mdoc:reset:invisible
import slick.jdbc.H2Profile.api._
```

```scala mdoc:silent
import slick.collection.heterogeneous.{ HList, HCons, HNil }
import slick.collection.heterogeneous.syntax._
import scala.language.postfixOps
```
```scala mdoc

type AttrHList =
  Long :: Long ::
  String :: Int :: String :: Int :: String :: Int ::
  String :: Int :: String :: Int :: String :: Int ::
  String :: Int :: String :: Int :: String :: Int ::
  String :: Int :: String :: Int :: String :: Int ::
  HNil

class AttrTable(tag: Tag) extends Table[AttrHList](tag, "attrs") {
  def id        = column[Long]("id", O.PrimaryKey, O.AutoInc)
  def productId = column[Long]("product_id")
  def name1     = column[String]("name1")
  def value1    = column[Int]("value1")
  def name2     = column[String]("name2")
  def value2    = column[Int]("value2")
  def name3     = column[String]("name3")
  def value3    = column[Int]("value3")
  def name4     = column[String]("name4")
  def value4    = column[Int]("value4")
  def name5     = column[String]("name5")
  def value5    = column[Int]("value5")
  def name6     = column[String]("name6")
  def value6    = column[Int]("value6")
  def name7     = column[String]("name7")
  def value7    = column[Int]("value7")
  def name8     = column[String]("name8")
  def value8    = column[Int]("value8")
  def name9     = column[String]("name9")
  def value9    = column[Int]("value9")
  def name10    = column[String]("name10")
  def value10   = column[Int]("value10")
  def name11    = column[String]("name11")
  def value11   = column[Int]("value11")
  def name12    = column[String]("name12")
  def value12   = column[Int]("value12")

  def * = id :: productId ::
          name1 :: value1 :: name2 :: value2 :: name3 :: value3 ::
          name4 :: value4 :: name5 :: value5 :: name6 :: value6 ::
          name7 :: value7 :: name8 :: value8 :: name9 :: value9 ::
          name10 :: value10 :: name11 :: value11 :: name12 :: value12 ::
          HNil
}

val attributes = TableQuery[AttrTable]
```

Writing `HList` types and values is cumbersome and error prone,
so we've introduced a type alias of `AttrHList`
to help us.

```scala mdoc:invisible
import slick.jdbc.H2Profile.api._
import scala.concurrent.{Await,Future}
import scala.concurrent.duration._

val db = Database.forConfig("chapter05")

def exec[T](action: DBIO[T]): T = Await.result(db.run(action), 4.seconds)
```

Working with this table involves inserting, updating, selecting, and modifying
instances of `AttrHList`. For example:

```scala mdoc
import scala.concurrent.ExecutionContext.Implicits.global

val program: DBIO[Seq[AttrHList]] = for {
  _ <- attributes.schema.create
  _ <- attributes += 0L :: 100L ::
        "name1"  :: 1  :: "name2"  :: 2  :: "name3"  :: 3  ::
        "name4"  :: 4  :: "name5"  :: 5  :: "name6"  :: 6  ::
        "name7"  :: 7  :: "name8"  :: 8  :: "name9"  :: 9  ::
        "name10" :: 10 :: "name11" :: 11 :: "name12" :: 12 ::
        HNil
  rows <- attributes.filter(_.value1 === 1).result
} yield rows


val myAttrs: AttrHList = exec(program).head
```

We can extract values from our query results `HList` using pattern matching
or a variety of type-preserving methods defined on `HList`,
including `head`, `apply`, `drop`, and `fold`:

```scala mdoc
val id: Long = myAttrs.head
val productId: Long = myAttrs.tail.head
val name1: String = myAttrs(2)
val value1: Int = myAttrs(3)
```

#### Using HLists and Case Classes

In practice we'll want to map an HList representation
to a regular class to make it easier to work with.
Slick's `<>` operator works with `HList` shapes as well as tuple shapes.
To use it we'd have to produce our own mapping functions in place of the case class `apply` and `unapply`,
but otherwise this approach is the same as we've seen for tuples.

However, the `mapTo` macro will generate the mapping between an HList and a case class for us:

```scala mdoc:reset:invisible
import slick.jdbc.H2Profile.api._
import slick.collection.heterogeneous.{ HList, HCons, HNil }
import slick.collection.heterogeneous.syntax._
import scala.language.postfixOps
import scala.concurrent.{Await,Future}
import scala.concurrent.duration._
val db = Database.forConfig("chapter05")
def exec[T](action: DBIO[T]): T = Await.result(db.run(action), 4.seconds)
import scala.concurrent.ExecutionContext.Implicits.global
```

```scala mdoc:silent
// A case class for our very wide row:
case class Attrs(id: Long, productId: Long,
  name1: String, value1: Int, name2: String, value2: Int /* etc */)

class AttrTable(tag: Tag) extends Table[Attrs](tag, "attributes") {
  def id        = column[Long]("id", O.PrimaryKey, O.AutoInc)
  def productId = column[Long]("product_id")
  def name1     = column[String]("name1")
  def value1    = column[Int]("value1")
  def name2     = column[String]("name2")
  def value2    = column[Int]("value2")
  /* etc */

  def * = (
    id :: productId ::
    name1 :: value1 :: name2 :: value2 /* etc */ ::
    HNil
  ).mapTo[Attrs]
}

val attributes = TableQuery[AttrTable]
```

Notice the pattern is:

```scala
 def * = (some hlist).mapTo[case class with the same fields]
```

With this in place our table is defined on a plain Scala case class.
We can query and modify the data as normal using case classes:

```scala mdoc
val program: DBIO[Seq[Attrs]] = for {
  _    <- attributes.schema.create
  _    <- attributes += Attrs(0L, 100L, "n1", 1, "n2", 2 /* etc */)
  rows <- attributes.filter(_.productId === 100L).result
} yield rows

exec(program)
```

<div class="callout callout-info">
**Code Generation**

Sometimes your code is the definitive description of the schema;
other times it's the database itself.
The latter is the case when working with legacy databases,
or database where the schema is managed independently of your Slick application.

When the database is considered the source truth in your organisation,
the [Slick code generator][link-ref-gen] is an important tool.
It allows you to connect to a database, generate the table definitions,
and customize the code produced.
For tables with wide rows, it produces an HList representation.

Prefer it to manually reverse engineering a schema by hand.
</div>

## Table and Column Representation

Now we know how rows can be represented and mapped,
let's look in more detail at the representation of the table and the columns it comprises.
In particular we'll explore nullable columns,
foreign keys, more about primary keys, composite keys,
and options you can apply to a table.

### Nullable Columns {#null-columns}

Columns defined in SQL are nullable by default.
That is, they can contain `NULL` as a value.
Slick makes columns non-nullable by default---if
you want a nullable column you model it naturally in Scala as an `Option[T]`.

Let's create a variant of `User` with an optional email address:

```scala mdoc:reset:invisible
import slick.jdbc.H2Profile.api._
import scala.concurrent.{Await,Future}
import scala.concurrent.duration._
val db = Database.forConfig("chapter05")
def exec[T](action: DBIO[T]): T = Await.result(db.run(action), 4.seconds)
import scala.concurrent.ExecutionContext.Implicits.global
```

```scala mdoc
case class User(name: String, email: Option[String] = None, id: Long = 0L)

class UserTable(tag: Tag) extends Table[User](tag, "user") {
  def id    = column[Long]("id", O.PrimaryKey, O.AutoInc)
  def name  = column[String]("name")
  def email = column[Option[String]]("email")

  def * = (name, email, id).mapTo[User]
}

lazy val users = TableQuery[UserTable]
lazy val insertUser = users returning users.map(_.id)
```

We can insert users with or without an email address:

```scala mdoc
val program = (
  users.schema.create >>
  (users += User("Dave", Some("dave@example.org"))) >>
  (users += User("HAL"))
)

exec(program)
```

and retrieve them again with a select query:

```scala mdoc
exec(users.result).foreach(println)
```

So far, so ordinary.
What might be a surprise is how you go about selecting all rows that have no email address.
You might expect the following to find the one row that has no email address:

```scala mdoc
// Don't do this
val none: Option[String] = None
val badQuery = exec(users.filter(_.email === none).result)
```

Despite the fact that we do have
one row in the database no email address,
this query produces no results.

Veterans of database administration will be familiar with this interesting quirk of SQL:
expressions involving `null` themselves evaluate to `null`.
For example, the SQL expression `'Dave' = 'HAL'` evaluates to `false`,
whereas the expression `'Dave' = null` evaluates to `null`.

Our Slick query above amounts to:

~~~ sql
SELECT * FROM "user" WHERE "email" = NULL
~~~

The SQL expression `"email" = null` evaluates to `null` for any value of `"email"`.
SQL's `null` is a falsey value, so this query never returns a value.

To resolve this issue, SQL provides two operators: `IS NULL` and `IS NOT NULL`,
which are provided in Slick by the methods `isEmpty` and `isDefined` on any `Rep[Option[A]]`:

--------------------------------------------------------------------------------------------------------
Scala Code              Operand Column Types               Result Type        SQL Equivalent
----------------------- ---------------------------------- ------------------ --------------------------
`col.?`                 `A`                                `Option[A]`        `col`

`col.isEmpty`           `Option[A]`                        `Boolean`          `col is null`

`col.isDefined`         `Option[A]`                        `Boolean`          `col is not null`

--------------------------------------------------------------------------------------------------------

: Optional column methods.
  Operand and result types should be interpreted as parameters to `Rep[_]`.
  The `?` method is described in the next section.


We can fix our query by replacing our equality check with `isEmpty`:

```scala mdoc
val myUsers = exec(users.filter(_.email.isEmpty).result)
```

which translates to the following SQL:

~~~ sql
SELECT * FROM "user" WHERE "email" IS NULL
~~~


### Primary Keys

We had our first introduction to primary keys in Chapter 1,
where we started setting up `id` fields using
the `O.PrimaryKey` and `O.AutoInc` column options:

~~~ scala
def id = column[Long]("id", O.PrimaryKey, O.AutoInc)
~~~

These options do two things:

- they modify the SQL generated for DDL statements;

- `O.AutoInc` removes the corresponding column
  from the SQL generated for `INSERT` statements,
  allowing the database to insert an auto-incrementing value.

In Chapter 1 we combined `O.AutoInc` with
a case class that has a default ID of `0L`,
knowing that Slick will skip the value in insert statements:

```scala
case class User(name: String, id: Long = 0L)
```

While we like the simplicity of this style,
some developers prefer to wrap primary key values in `Options`:

```scala
case class User(name: String, id: Option[Long] = None)
```

In this model we use `None` as the primary key of an unsaved record
and `Some` as the primary key of a saved record.
This approach has advantages and disadvantages:

- on the positive side it's easier to identify unsaved records;

- on the negative side it's harder to get the value of a primary key for use in a query.

Let's look at the changes we need to make to our `UserTable`
to make this work:

```scala mdoc:reset:invisible
import slick.jdbc.H2Profile.api._
import scala.concurrent.{Await,Future}
import scala.concurrent.duration._
val db = Database.forConfig("chapter05")
def exec[T](action: DBIO[T]): T = Await.result(db.run(action), 4.seconds)
import scala.concurrent.ExecutionContext.Implicits.global
```

```scala mdoc
case class User(id: Option[Long], name: String)

class UserTable(tag: Tag) extends Table[User](tag, "user") {
  def id     = column[Long]("id", O.PrimaryKey, O.AutoInc)
  def name   = column[String]("name")

  def * = (id.?, name).mapTo[User]
}

lazy val users = TableQuery[UserTable]
lazy val insertUser = users returning users.map(_.id)
```

The key thing to notice here is that
we *don't* want the primary key to be optional in the database.
We're using `None` to represent an *unsaved* value---the database
assigns a primary key for us on insert,
so we can never retrieve a `None` via a database query.

We need to map our non-nullable database column to an optional field value.
This is handled by the `?` method in the default projection,
which converts a `Rep[A]` to a `Rep[Option[A]]`.


### Compound Primary Keys

There is a second way to declare a column as a primary key:

~~~ scala
def id = column[Long]("id", O.AutoInc)
def pk = primaryKey("pk_id", id)
~~~

This separate step doesn't make much of a difference in this case.
It separates the column definition from the key constraint,
meaning the schema will include:

~~~ sql
ALTER TABLE "user" ADD CONSTRAINT "pk_id" PRIMARY KEY("id")
~~~

The `primaryKey` method is more useful for defining *compound* primary keys
that involve two or more columns.

Let's look at this by adding the ability for people to chat in rooms.
First we need a table for storing rooms, which is straightforward:

```scala mdoc
// Regular table definition for a chat room:
case class Room(title: String, id: Long = 0L)

class RoomTable(tag: Tag) extends Table[Room](tag, "room") {
 def id    = column[Long]("id", O.PrimaryKey, O.AutoInc)
 def title = column[String]("title")
 def * = (title, id).mapTo[Room]
}

lazy val rooms = TableQuery[RoomTable]
lazy val insertRoom = rooms returning rooms.map(_.id)
```

Next we need a table that relates users to rooms.
We'll call this the *occupant* table.
Rather than give this table an auto-generated primary key,
we'll make it a compound of the user and room IDs:

```scala mdoc
case class Occupant(roomId: Long, userId: Long)

class OccupantTable(tag: Tag) extends Table[Occupant](tag, "occupant") {
  def roomId = column[Long]("room")
  def userId = column[Long]("user")

  def pk = primaryKey("room_user_pk", (roomId, userId))

  def * = (roomId, userId).mapTo[Occupant]
}

lazy val occupants = TableQuery[OccupantTable]
```

We can define composite primary keys using tuples or `HList`s of columns
(Slick generates a `ProvenShape` and inspects it to find the list of columns involved).
The SQL generated for the `occupant` table is:

~~~ sql
CREATE TABLE "occupant" (
  "room" BIGINT NOT NULL,
  "user" BIGINT NOT NULL
)

ALTER TABLE "occupant"
ADD CONSTRAINT "room_user_pk" PRIMARY KEY("room", "user")
~~~

Using the `occupant` table is no different from any other table:

```scala mdoc
val program: DBIO[Int] = for {
   _        <- rooms.schema.create
   _        <- occupants.schema.create
  elenaId   <- insertUser += User(None, "Elena")
  airLockId <- insertRoom += Room("Air Lock")
  // Put Elena in the Room:
  rowsAdded <- occupants += Occupant(airLockId, elenaId)
} yield rowsAdded

exec(program)
```

```scala mdoc:invisible
val assure_o1 = exec(occupants.result)
assert(assure_o1.length == 1, "Expected 1 occupant")
```

Of course, if we try to put Dave in the Air Lock twice,
the database will complain about duplicate primary keys.


### Indices

We can use indices to increase the efficiency of database queries
at the cost of higher disk usage.
Creating and using indices is the highest form of database sorcery,
different for every database application,
and well beyond the scope of this book.
However, the syntax for defining an index in Slick is simple.
Here's a table with two calls to `index`:

```scala mdoc
class IndexExample(tag: Tag) extends Table[(String,Int)](tag, "people") {
  def name = column[String]("name")
  def age  = column[Int]("age")

  def * = (name, age)

  def nameIndex = index("name_idx", name, unique=true)
  def compoundIndex = index("c_idx", (name, age), unique=true)
}
```

The corresponding DDL statement produced due to `nameIndex` will be:

~~~ sql
CREATE UNIQUE INDEX "name_idx" ON "people" ("name")
~~~

We can create compound indices on multiple columns
just like we can with primary keys.
In this case (`compoundIndex`) the corresponding DDL statement will be:

~~~ sql
CREATE UNIQUE INDEX "c_idx" ON "people" ("name", "age")
~~~


### Foreign Keys {#fks}

Foreign keys are declared in a similar manner to compound primary keys.

The method `foreignKey` takes four required parameters:

 * a name;

 * the column, or columns, that make up the foreign key;

 * the `TableQuery` that the foreign key belongs to; and

 * a function on the supplied `TableQuery[T]` taking
   the supplied column(s) as parameters and returning an instance of `T`.

We'll step through this by using foreign keys to connect a `message` to a `user`.
We do this by changing the definition of `message` to reference
the `id` of its sender instead of their name:

```scala mdoc
case class Message(
  senderId : Long,
  content  : String,
  id       : Long = 0L)

class MessageTable(tag: Tag) extends Table[Message](tag, "message") {
  def id       = column[Long]("id", O.PrimaryKey, O.AutoInc)
  def senderId = column[Long]("sender")
  def content  = column[String]("content")

  def * = (senderId, content, id).mapTo[Message]

  def sender = foreignKey("sender_fk", senderId, users)(_.id)
}

lazy val messages = TableQuery[MessageTable]
```

The column for the sender is now a `Long` instead of a `String`.
We have also defined a method, `sender`,
providing the foreign key linking the `senderId` to a `user` `id`.

The `foreignKey` gives us two things.
First, it adds a constraint to the DDL statement generated by Slick:

~~~ sql
ALTER TABLE "message" ADD CONSTRAINT "sender_fk"
  FOREIGN KEY("sender") REFERENCES "user"("id")
  ON UPDATE NO ACTION
  ON DELETE NO ACTION
~~~

<div class="callout callout-info">
**On Update and On Delete**

A foreign key makes certain guarantees about the data you store.
In the case we've looked at there must be a `sender` in the `user` table
to successfully insert a new `message`.

So what happens if something changes with the `user` row?
There are a number of [_referential actions_][link-fkri] that could be triggered.
The default is for nothing to happen, but you can change that.

Let's look at an example.
Suppose we delete a user,
and we want all the messages associated with that user to be removed.
We could do that in our application, but it's something the database can provide for us:

```scala mdoc
class AltMsgTable(tag: Tag) extends Table[Message](tag, "message") {
  def id       = column[Long]("id", O.PrimaryKey, O.AutoInc)
  def senderId = column[Long]("sender")
  def content  = column[String]("content")

  def * = (senderId, content, id).mapTo[Message]

  def sender = foreignKey("sender_fk", senderId, users)(_.id, onDelete=ForeignKeyAction.Cascade)
}
```

Providing Slick's `schema` command has been run for the table,
or the SQL `ON DELETE CASCADE` action has been manually applied to the database,
the following action will remove HAL from the `users` table,
and all of the messages that HAL sent:

```scala mdoc:silent
users.filter(_.name === "HAL").delete
```

Slick supports `onUpdate` and `onDelete` for the five actions:

------------------------------------------------------------------
Action          Description
-----------     --------------------------------------------------
`NoAction`      The default.

`Cascade`       A change in the referenced table triggers a change in the referencing table.
                In our example, deleting a user will cause their messages to be deleted.

`Restrict`      Changes are restricted, triggered a constraint violation exception.
                In our example, you would not be allowed to delete a user who had
                posted a message.

`SetNull`       The column referencing the updated value will be set to NULL.

`SetDefault`    The default value for the referencing column will be used.
                Default values are discussion in [Table and Column Modifiers](#schema-modifiers),
                later in this chapter.
-----------     --------------------------------------------------

</div>


Second, the foreign key gives us a query that we can use in a join.
We've dedicated the [next chapter](#joins) to looking at joins in detail,
but here's a simple join to illustrate the use case:

```scala mdoc
val q = for {
  msg <- messages
  usr <- msg.sender
} yield (usr.name, msg.content)
```

This is equivalent to the query:

~~~ sql
SELECT u."name", m."content"
FROM "message" m, "user" u
WHERE "id" = m."sender"
~~~

...and once we have populated the database...

```scala mdoc
def findUserId(name: String): DBIO[Option[Long]] =
  users.filter(_.name === name).map(_.id).result.headOption

def findOrCreate(name: String): DBIO[Long] =
  findUserId(name).flatMap { userId =>
    userId match {
      case Some(id) => DBIO.successful(id)
      case None     => insertUser += User(None, name)
    }
}

// Populate the messages table:
val setup = for {
  daveId <- findOrCreate("Dave")
  halId  <- findOrCreate("HAL")

  // Add some messages:
  _         <- messages.schema.create
  rowsAdded <- messages ++= Seq(
    Message(daveId, "Hello, HAL. Do you read me, HAL?"),
    Message(halId,  "Affirmative, Dave. I read you."),
    Message(daveId, "Open the pod bay doors, HAL."),
    Message(halId,  "I'm sorry, Dave. I'm afraid I can't do that.")
  )
} yield rowsAdded

exec(setup)
```

...our query produces the following results, showing the sender name (not ID) and corresponding message:

```scala mdoc
exec(q.result).foreach(println)
```

<!--
Notice that we modelled the `Message` row using a `Long` `sender`,
rather than a `User`:

~~~ scala
case class Message(senderId: Long,
                   content: String,
                   id: Long = 0L)
~~~

That's the design approach to take with Slick.
The row model should reflect the row, not the row plus a bunch of other data from different tables.
To pull data from across tables, use a query.
-->


<div class="callout callout-info">
**Save Your Sanity With Laziness**

Defining foreign keys places constraints
on the order in which we have to define our database tables.
In the example above, the foreign key from `MessageTable`
to `UserTable` requires us to place the latter definition above
the former in our Scala code.

Ordering constraints make complex schemas difficult to write.
Fortunately, we can work around them using `def` and `lazy val`.

As a rule, use `lazy val` for `TableQuery`s and `def` foreign keys (for consistency with `column` definitions).
</div>


### Column Options {#schema-modifiers}

We'll round off this section by looking at modifiers for columns and tables.
These allow us to tweak the default values, sizes, and data types for columns
at the SQL level.

We have already seen two examples of column options,
namely `O.PrimaryKey` and `O.AutoInc`.
Column options are defined in [`ColumnOption`][link-slick-column-options],
and as you have seen are accessed via `O`.

The following example introduces four new options:
`O.Length`, `O.SqlType`, `O.Unique`, and `O.Default`.

```scala mdoc
case class PhotoUser(
  name   : String,
  email  : String,
  avatar : Option[Array[Byte]] = None,
  id     : Long = 0L)

class PhotoTable(tag: Tag) extends Table[PhotoUser](tag, "user") {
  def id     = column[Long]("id", O.PrimaryKey, O.AutoInc)

  def name   = column[String](
                 "name",
                 O.Length(64, true),
                 O.Default("Anonymous Coward")
                )

  def email = column[String]("email", O.Unique)

  def avatar = column[Option[Array[Byte]]]("avatar", O.SqlType("BINARY(2048)"))

  def * = (name, email, avatar, id).mapTo[PhotoUser]
}
```

In this example we've done four things:

1. We've used `O.Length` to give the `name` column a maximum length.
   This modifies the type of the column in the DDL statement.
   The parameters to `O.Length` are an `Int` specifying the maximum length,
   and a `Boolean` indicating whether the length is variable.
   Setting the `Boolean` to `true` sets the SQL column type to `VARCHAR`;
   setting it to `false` sets the type to `CHAR`.

2. We've used `O.Default` to give the `name` column a default value.
   This adds a `DEFAULT` clause to the column definition in the DDL statement.

3. We added a uniqueness constraint on the `email` column.

4. We've used `O.SqlType` to control the exact type used by the database.
   The values allowed here depend on the database we're using.


## Custom Column Mappings

We want to work with types that have meaning to our application.
This means converting data from the simple types the database uses
to something more developer-friendly.

We've already seen Slick's ability to map
tuples and `HList`s of columns to case classes.
However, so far the fields of our case classes
have been restricted to simple types such as
`Int` and `String`,

Slick also lets us control how individual columns are mapped to Scala types.
For example, perhaps we'd like to use
[Joda Time][link-jodatime]'s `DateTime` class
for anything date and time related.
Slick doesn't provide native support for Joda Time[^time],
but it's painless for us to implement it via Slick's
`ColumnType` type class:

[^time]: However since Slick 3.3.0 there is built-in support for `java.time.Instant`, `LocalDate`, `LocalTime`, `LocalDateTime`, `OffsetTime`, `OffsetDateTime`, and `ZonedDateTime`.
You'll very likely want to use these over the older Joda Time library.

```scala mdoc:reset:invisible
import slick.jdbc.H2Profile.api._
import scala.concurrent.{Await,Future}
import scala.concurrent.duration._
val db = Database.forConfig("chapter05")
def exec[T](action: DBIO[T]): T = Await.result(db.run(action), 4.seconds)
import scala.concurrent.ExecutionContext.Implicits.global
```

```scala mdoc:silent
import java.sql.Timestamp
import org.joda.time.DateTime
import org.joda.time.DateTimeZone.UTC
```
```scala mdoc
object CustomColumnTypes {
  implicit val jodaDateTimeType =
    MappedColumnType.base[DateTime, Timestamp](
      dt => new Timestamp(dt.getMillis),
      ts => new DateTime(ts.getTime, UTC)
    )
}
```

What we're providing here is two functions to `MappedColumnType.base`:

- one from a `DateTime` to
  a database-friendly `java.sql.Timestamp`; and

- one that does the reverse, taking a `Timestamp`
  and converting it to a `DateTime`.

Once we have declared this custom column type,
we are free to create columns containing `DateTimes`:

```scala mdoc:silent
case class Message(
  senderId  : Long,
  content   : String,
  timestamp : DateTime,
  id        : Long = 0L)

class MessageTable(tag: Tag) extends Table[Message](tag, "message") {

  // Bring our implicit conversions into scope:
  import CustomColumnTypes._

  def id        = column[Long]("id", O.PrimaryKey, O.AutoInc)
  def senderId  = column[Long]("sender")
  def content   = column[String]("content")
  def timestamp = column[DateTime]("timestamp")

  def * = (senderId, content, timestamp, id).mapTo[Message]
}

lazy val messages = TableQuery[MessageTable]
lazy val insertMessage = messages returning messages.map(_.id)
```

```scala mdoc:invisible
// Re-establish scenario after last reset

case class Occupant(roomId: Long, userId: Long)

class OccupantTable(tag: Tag) extends Table[Occupant](tag, "occupant") {
  def roomId = column[Long]("room")
  def userId = column[Long]("user")

  def pk = primaryKey("room_user_pk", (roomId, userId))

  def * = (roomId, userId).mapTo[Occupant]
}

lazy val occupants = TableQuery[OccupantTable]

case class User(id: Option[Long], name: String)

class UserTable(tag: Tag) extends Table[User](tag, "user") {
  def id     = column[Long]("id", O.PrimaryKey, O.AutoInc)
  def name   = column[String]("name")

  def * = (id.?, name).mapTo[User]
}

lazy val users = TableQuery[UserTable]
lazy val insertUser = users returning users.map(_.id)

case class Room(title: String, id: Long = 0L)

class RoomTable(tag: Tag) extends Table[Room](tag, "room") {
 def id    = column[Long]("id", O.PrimaryKey, O.AutoInc)
 def title = column[String]("title")
 def * = (title, id).mapTo[Room]
}

lazy val rooms = TableQuery[RoomTable]
```

<div class="callout callout-info">
**Reset Your Database**

If you've been following along in the REPL,
by now you're going to have a bunch of tables and rows.
Now is a good time to remove all of that.

You can exit the REPL and restart it.
H2 is holding the data in memory,
so turning it on and off again is one way to reset your database.

Alternatively, you can use an action:

```scala mdoc
val schemas = (users.schema ++
  messages.schema          ++
  occupants.schema         ++
  rooms.schema)

exec(schemas.drop)
```
</div>


Our modified definition of `MessageTable` allows
us to work directly with `Message`s containing `DateTime` timestamps,
without having to do cumbersome type conversions by hand:

```scala mdoc
val program = for {
  _ <- messages.schema.create
  _ <- users.schema.create
  daveId <- insertUser += User(None, "Dave")
  msgId  <- insertMessage += Message(
    daveId,
    "Open the pod bay doors, HAL.",
    DateTime.now)
} yield msgId

val msgId = exec(program)
```

Fetching the database row will automatically convert the `timestamp` field into the `DateTime` value we expect:

```scala mdoc
exec(messages.filter(_.id === msgId).result)
```

This model of working with semantic types is
immediately appealing to Scala developers.
We strongly encourage you to use `ColumnType` in your applications,
to help reduce bugs and let Slick take care of the type conversions.


### Value Classes {#value-classes}

```scala mdoc:reset-object:invisible
import slick.jdbc.H2Profile.api._
import scala.concurrent.{Await,Future}
import scala.concurrent.duration._
val db = Database.forConfig("chapter05")
def exec[T](action: DBIO[T]): T = Await.result(db.run(action), 4.seconds)
import scala.concurrent.ExecutionContext.Implicits.global
```

We are currently using `Long`s to model primary keys.
Although this is a good choice at a database level,
it's not great for our application code.

The problem is we can make silly mistakes,
such as trying to look up a `User` by primary key using the primary key from a `Message`.
They are both `Long`s, but trying to compare them makes no sense.
And yet the code would compile, and could possibly return a result.
But it's likely to be the wrong result.

We can prevent these kinds of problems using types.
The essential approach is to model primary keys
using [value classes][link-scala-value-classes]:

```scala mdoc
case class MessagePK(value: Long) extends AnyVal
case class UserPK(value: Long) extends AnyVal
```

A value class is a compile-time wrapper around a value.
At run time, the wrapper goes away,
leaving no allocation or performance overhead[^vcpoly] in our running code.

[^vcpoly]: It's not totally cost free: there [are situations where a value will need allocation][link-scala-value-classes], such as when passed to a polymorphic method.

To use a value class we need to provide Slick with `ColumnType`s
to use these types with our tables.
This is the same process we used for Joda Time `DateTime`s:

```scala mdoc
implicit val messagePKColumnType =
  MappedColumnType.base[MessagePK, Long](_.value, MessagePK(_))

implicit val userPKColumnType =
   MappedColumnType.base[UserPK, Long](_.value, UserPK(_))
```

Defining all these type class instances can be time consuming,
especially if we're defining one for every table in our schema.
Fortunately, Slick provides a short-hand called `MappedTo`
to take care of this for us:

```scala mdoc:reset-object:invisible
import slick.jdbc.H2Profile.api._
```

```scala mdoc
case class MessagePK(value: Long) extends AnyVal with MappedTo[Long]
case class UserPK(value: Long) extends AnyVal with MappedTo[Long]
```

When we use `MappedTo` we don't need to define a separate `ColumnType`.
`MappedTo` works with any class that:

 - has a method called `value` that
   returns the underlying database value; and

 - has a single-parameter constructor to
   create the Scala value from the database value.

Value classes are a great fit for the `MappedTo` pattern.

Let's redefine our tables to use our custom primary key types.
We will convert `User`...

```scala mdoc:reset-object:invisible
import slick.jdbc.H2Profile.api._
import scala.concurrent.{Await,Future}
import scala.concurrent.duration._
val db = Database.forConfig("chapter05")
def exec[T](action: DBIO[T]): T = Await.result(db.run(action), 4.seconds)
import scala.concurrent.ExecutionContext.Implicits.global
case class MessagePK(value: Long) extends AnyVal with MappedTo[Long]
case class UserPK(value: Long) extends AnyVal with MappedTo[Long]
```

```scala mdoc
case class User(name: String, id: UserPK = UserPK(0L))

class UserTable(tag: Tag) extends Table[User](tag, "user") {
  def id   = column[UserPK]("id", O.PrimaryKey, O.AutoInc)
  def name = column[String]("name")
  def * = (name, id).mapTo[User]
}

lazy val users = TableQuery[UserTable]
lazy val insertUser = users returning users.map(_.id)
```

...and `Message`:

```scala mdoc
case class Message(
  senderId : UserPK,
  content  : String,
  id       : MessagePK = MessagePK(0L))

class MessageTable(tag: Tag) extends Table[Message](tag, "message") {
  def id        = column[MessagePK]("id", O.PrimaryKey, O.AutoInc)
  def senderId  = column[UserPK]("sender")
  def content   = column[String]("content")

  def * = (senderId, content, id).mapTo[Message]

  def sender = foreignKey("sender_fk", senderId, users) (_.id, onDelete=ForeignKeyAction.Cascade)
}

lazy val messages      = TableQuery[MessageTable]
lazy val insertMessage = messages returning messages.map(_.id)
```

Notice how we're able to be explicit:
the `User.id` and `Message.senderId` are `UserPK`s,
and the `Message.id` is a `MessagePK`.

We can lookup values if we have the right kind of key:

```scala mdoc
users.filter(_.id === UserPK(0L))
```

...but if we accidentally try to mix our primary keys, we'll find we cannot:

```scala mdoc:fail
users.filter(_.id === MessagePK(0L))
```

Values classes are a low-cost way to make code safer and more legible.
The amount of code required is small,
however for a large database it can still be an overhead.
We can either use code generation to overcome this,
or generalise our primary key type by making it generic:

```scala mdoc:reset-object:invisible
import slick.jdbc.H2Profile.api._
import scala.concurrent.{Await,Future}
import scala.concurrent.duration._
val db = Database.forConfig("chapter05")
def exec[T](action: DBIO[T]): T = Await.result(db.run(action), 4.seconds)
import scala.concurrent.ExecutionContext.Implicits.global
```

```scala mdoc
case class PK[A](value: Long) extends AnyVal with MappedTo[Long]

case class User(
  name : String,
  id   : PK[UserTable])

class UserTable(tag: Tag) extends Table[User](tag, "user") {
  def id    = column[PK[UserTable]]("id", O.AutoInc, O.PrimaryKey)
  def name  = column[String]("name")

  def * = (name, id).mapTo[User]
}

lazy val users = TableQuery[UserTable]

val exampleQuery =
  users.filter(_.id === PK[UserTable](0L))
```

With this approach we achieve type safety
without the boiler plate of many primary key type definitions.
Depending on the nature of your application,
this may be convenient for you.

The general point is that we can use the whole of the Scala type system
to represent primary keys, foreign keys, rows, and columns from our database.
This is enormously valuable and should not be overlooked.


### Modelling Sum Types

```scala mdoc:reset-object:invisible
import slick.jdbc.H2Profile.api._
import scala.concurrent.{Await,Future}
import scala.concurrent.duration._
val db = Database.forConfig("chapter05")
def exec[T](action: DBIO[T]): T = Await.result(db.run(action), 4.seconds)
import scala.concurrent.ExecutionContext.Implicits.global

case class MessagePK(value: Long) extends AnyVal with MappedTo[Long]
case class UserPK(value: Long) extends AnyVal with MappedTo[Long]

case class User(name: String, id: UserPK = UserPK(0L))

class UserTable(tag: Tag) extends Table[User](tag, "user") {
  def id   = column[UserPK]("id", O.PrimaryKey, O.AutoInc)
  def name = column[String]("name")
  def * = (name, id).mapTo[User]
}

lazy val users = TableQuery[UserTable]
```

We've used case classes extensively for modelling data.
Using the language of _algebraic data types_,
case classes are "product types"
(created from conjunctions of their field types).
The other common form of algebraic data type is known as a _sum type_,
formed from a _disjunction_ of other types.
We'll look at modelling these now.

As an example let's add a flag to our `Message` class
to model messages as important, offensive, or spam.
The natural way to do this is establish a sealed trait
and a set of case objects:

```scala mdoc
sealed trait Flag
case object Important extends Flag
case object Offensive extends Flag
case object Spam extends Flag

case class Message(
  senderId : UserPK,
  content  : String,
  flag     : Option[Flag] = None,
  id       : MessagePK = MessagePK(0L))
```

There are a number of ways we could represent the flags in the database.
For the sake of the argument, let's use characters: `!`, `X`, and `$`.
We need a new custom `ColumnType` to manage the mapping:

```scala mdoc
implicit val flagType =
  MappedColumnType.base[Flag, Char](
    flag => flag match {
      case Important => '!'
      case Offensive => 'X'
      case Spam      => '$'
    },
    code => code match {
      case '!' => Important
      case 'X' => Offensive
      case '$' => Spam
    })
```

We like sum types because the compiler can ensure we've covered all the cases.
If we add a new flag (`OffTopic` perhaps),
the compiler will issue warnings
until we add it to our `Flag => Char` function.
We can turn these compiler warnings into errors
by enabling the Scala compiler's `-Xfatal-warnings` option,
preventing us shipping the application until we've covered all bases.

Using `Flag` is the same as any other custom type:

```scala mdoc
class MessageTable(tag: Tag) extends Table[Message](tag, "flagmessage") {
  def id       = column[MessagePK]("id", O.PrimaryKey, O.AutoInc)
  def senderId = column[UserPK]("sender")
  def content  = column[String]("content")
  def flag     = column[Option[Flag]]("flag")

  def * = (senderId, content, flag, id).mapTo[Message]

  def sender = foreignKey("sender_fk", senderId, users)(_.id, onDelete=ForeignKeyAction.Cascade)
}

lazy val messages = TableQuery[MessageTable]

exec(messages.schema.create)
```

We can insert a message with a flag easily:

```scala mdoc
val halId = UserPK(1L)

exec(
  messages += Message(
    halId,
    "Just kidding - come on in! LOL.",
    Some(Important)
  )
)
```

We can also query for messages with a particular flag.
However, we need to give the compiler a little help with the types:

```scala mdoc
exec(
  messages.filter(_.flag === (Important : Flag)).result
)
```

The _type annotation_ here is annoying.
We can work around it in two ways:

First, we can define a "smart constructor" method
for each flag that returns it pre-cast as a `Flag`:

```scala mdoc
object Flags {
  val important : Flag = Important
  val offensive : Flag = Offensive
  val spam      : Flag = Spam

  val action = messages.filter(_.flag === Flags.important).result
}
```

Second, we can define some custom syntax to
build our filter expressions:

```scala mdoc
implicit class MessageQueryOps(message: MessageTable) {
  def isImportant = message.flag === (Important : Flag)
  def isOffensive = message.flag === (Offensive : Flag)
  def isSpam      = message.flag === (Spam      : Flag)
}

messages.filter(_.isImportant).result.statements.head
```


## Take Home Points

In this Chapter we covered a lot of Slick's features
for defining database schemas.
We went into detail about defining tables and columns,
mapping them to convenient Scala types,
adding primary keys, foreign keys, and indices,
and customising Slick's DDL SQL.
We also discussed writing generic code
that works with multiple database back-ends,
and how to structure the database layer of your application
using traits and self-types.

The most important points are:

- We can separate the specific profile for our database (H2, Postgres, etc) from our tables.
  We assemble a database layer from a number of traits,
  leaving the profile as an abstract field
  that can be implemented at runtime.

- We can represent rows in a variety of ways: tuples, `HList`s,
  and arbitrary classes and case classes via the `mapTo` macro.

- If we need more control over a mapping from columns to other data structures, the `<>` method is available.

- We can represent individual values in columns
  using arbitrary Scala data types
  by providing `ColumnType`s to manage the mappings.
  We've seen numerous examples supporting
  typed primary keys such as `UserPK`,
  sealed traits such as `Flag`, and
  third party classes such as `DateTime`.

- Nullable values are typically represented as `Option`s in Scala.
  We can either define columns to store `Option`s directly,
  or use the `?` method to map non-nullable columns to optional ones.

- We can define simple primary keys using `O.PrimaryKey`
  and compound keys using the `primaryKey` method.

- We can define `foreignKeys`,
  which gives us a simple way of linking tables in a join.
  More on this next chapter.

Slick's philosophy is to keep models simple.
We model rows as flat case classes, ignoring joins with other tables.
While this may seem inflexible at first,
it more than pays for itself in terms of simplicity and transparency.
Database queries are explicit and type-safe,
and return values of convenient types.

In the next chapter we will build on the foundations of
primary and foreign keys and look at writing more
complex queries involving joins and aggregate functions.



## Exercises


### Filtering Optional Columns

Imagine a reporting tool on a web site.
Sometimes you want to look at all the users in the database, and sometimes you want to only see rows matching a particular value.

Working with the optional email address for a user,
write a method that will take an optional value,
and list rows matching that value.

The method signature is:

```scala
def filterByEmail(email: Option[String]) = ???
```

```scala mdoc:reset:invisible
import slick.jdbc.H2Profile.api._
import scala.concurrent.{Await,Future}
import scala.concurrent.duration._
val db = Database.forConfig("chapter05")
def exec[T](action: DBIO[T]): T = Await.result(db.run(action), 4.seconds)
import scala.concurrent.ExecutionContext.Implicits.global
```

Assume we only have two user records, one with an email address and one with no email address:

```scala mdoc:silent
case class User(name: String, email: Option[String], id: Long = 0)

class UserTable(tag: Tag) extends Table[User](tag, "filtering_3") {
  def id     = column[Long]("id", O.PrimaryKey, O.AutoInc)
  def name   = column[String]("name")
  def email  = column[Option[String]]("email")

  def * = (name, email, id).mapTo[User]
}

lazy val users = TableQuery[UserTable]

val setup = DBIO.seq(
  users.schema.create,
  users += User("Dave", Some("dave@example.org")),
  users += User("HAL ", None)
)

exec(setup)
```

We want `filterByEmail(Some("dave@example.org"))` to produce one row,
and `filterByEmail(None)` to produce two rows:

Tip: it's OK to use multiple queries.

<div class="solution">
We can decide on the query to run in the two cases from inside our application:

```scala mdoc
def filterByEmail(email: Option[String]) =
  email.isEmpty match {
    case true  => users
    case false => users.filter(_.email === email)
  }
```

You don't always have to do everything at the SQL level.

```scala mdoc
exec(
  filterByEmail(Some("dave@example.org")).result
).foreach(println)

exec(
  filterByEmail(None).result
).foreach(println)
```

</div>


### Matching or Undecided

```scala mdoc:reset:invisible
import slick.jdbc.H2Profile.api._
import scala.concurrent.{Await,Future}
import scala.concurrent.duration._
val db = Database.forConfig("chapter05")
def exec[T](action: DBIO[T]): T = Await.result(db.run(action), 4.seconds)
import scala.concurrent.ExecutionContext.Implicits.global

case class User(name: String, email: Option[String], id: Long = 0)

class UserTable(tag: Tag) extends Table[User](tag, "filtering_2") {
  def id     = column[Long]("id", O.PrimaryKey, O.AutoInc)
  def name   = column[String]("name")
  def email  = column[Option[String]]("email")

  def * = (name, email, id).mapTo[User]
}

lazy val users = TableQuery[UserTable]

val setup = DBIO.seq(
  users.schema.create,
  users += User("Dave", Some("dave@example.org")),
  users += User("HAL ", None)
)

exec(setup)
```

Not everyone has an email address, so perhaps when filtering it would be safer to exclude rows that don't match our filter criteria.
That is, keep `NULL` addresses in the results.

Add Elena to the database...

```scala mdoc
exec(
  users += User("Elena", Some("elena@example.org"))
)
```

...and modify `filterByEmail` so when we search for `Some("elena@example.org")` we only
exclude Dave, as he definitely doesn't match that address.

This time you can do this in one query.

Hint: if you get stuck thinking about this in terms of SQL,
think about it in terms of Scala collections.  E.g.,

```scala
List(Some("dave"), Some("elena"), None).filter( ??? ) == List(Some("elena", None))
```

<div class="solution">
This problem we can represent in SQL, so we can do it with one query:

```scala mdoc
def filterByEmail(email: Option[String]) =
  users.filter(u => u.email.isEmpty || u.email === email)
```

In this implementation we've decided that if you search for email addresses matching `None`, we only return `NULL` email address.
But you could switch on the value of `email` and do something different, as we did in previous exercises.

```scala mdoc:invisible
{
  val result = exec(filterByEmail(Some("elena@example.org")).result)
  assert(result.length == 2, s"Expected 2 results, not $result")
}

{
  val result = exec(filterByEmail(None).result)
  assert(result.length == 1, s"Expected 1 results, not $result")
}
```
</div>


### Enforcement

```scala mdoc:reset-object:invisible
import slick.jdbc.H2Profile.api._
import scala.concurrent.{Await,Future}
import scala.concurrent.duration._
val db = Database.forConfig("chapter05")
def exec[T](action: DBIO[T]): T = Await.result(db.run(action), 4.seconds)
import scala.concurrent.ExecutionContext.Implicits.global

case class MessagePK(value: Long) extends AnyVal with MappedTo[Long]
case class UserPK(value: Long) extends AnyVal with MappedTo[Long]

case class User(name: String, id: UserPK = UserPK(0L))

class UserTable(tag: Tag) extends Table[User](tag, "user") {
  def id   = column[UserPK]("id", O.PrimaryKey, O.AutoInc)
  def name = column[String]("name")
  def * = (name, id).mapTo[User]
}

lazy val users = TableQuery[UserTable]

case class Message(
  senderId : UserPK,
  content  : String,
  id       : MessagePK = MessagePK(0L))

class MessageTable(tag: Tag) extends Table[Message](tag, "msg_table") {
  def id       = column[MessagePK]("id", O.PrimaryKey, O.AutoInc)
  def senderId = column[UserPK]("sender")
  def content  = column[String]("content")

  def * = (senderId, content, id).mapTo[Message]

  def sender = foreignKey("sender_fk2", senderId, users)(_.id, onDelete=ForeignKeyAction.Cascade)
}

lazy val messages = TableQuery[MessageTable]

exec(messages.schema.create)
```

What happens if you try adding a message for a user ID of `3000`?

For example:

```scala
messages += Message(UserPK(3000L), "Hello HAL!")
```

Note that there is no user in our example with an ID of 3000.

<div class="solution">
We get a runtime exception as we have violated referential integrity.
There is no row in the `user` table with a primary id of `3000`.

```scala mdoc
val action = messages += Message(UserPK(3000L), "Hello HAL!")
exec(action.asTry)
```

```scala mdoc:invisible
{
  val result = exec(action.asTry)
  assert(result.toString contains "Referential", s"Expected referential integrity error, not $result")
}
```
</div>


### Mapping Enumerations

We can use the same trick that we've seen for `DateTime` and value classes to map enumerations.

Here's a Scala Enumeration for a user's role:

```scala mdoc
object UserRole extends Enumeration {
  type UserRole = Value
  val Owner   = Value("O")
  val Regular = Value("R")
}
```

Modify the `user` table to include a `UserRole`.
In the database store the role as a single character.

<div class="solution">

```scala mdoc:reset-object:invisible
import slick.jdbc.H2Profile.api._
import scala.concurrent.{Await,Future}
import scala.concurrent.duration._
val db = Database.forConfig("chapter05")
def exec[T](action: DBIO[T]): T = Await.result(db.run(action), 4.seconds)
import scala.concurrent.ExecutionContext.Implicits.global
case class UserPK(value: Long) extends AnyVal with MappedTo[Long]
```

The first step is to supply an implicit to and from the database values:

```scala mdoc
object UserRole extends Enumeration {
  type UserRole = Value
  val Owner   = Value("O")
  val Regular = Value("R")
}

import UserRole._
implicit val userRoleMapper =
  MappedColumnType.base[UserRole, String](_.toString, UserRole.withName(_))
```

Then we can use the `UserRole` in the table definition:

```scala mdoc
case class User(
  name     : String,
  userRole : UserRole = Regular,
  id       : UserPK = UserPK(0L)
)

class UserTable(tag: Tag) extends Table[User](tag, "user_with_role") {
  def id   = column[UserPK]("id", O.PrimaryKey, O.AutoInc)
  def name = column[String]("name")
  def role = column[UserRole]("role", O.Length(1,false))

  def * = (name, role, id).mapTo[User]
}

lazy val users = TableQuery[UserTable]
```

We've made the `role` column exactly 1 character in size.
</div>


### Alternative Enumerations

```scala mdoc:reset-object:invisible
import slick.jdbc.H2Profile.api._
object UserRole extends Enumeration {
  type UserRole = Value
  val Owner   = Value("O")
  val Regular = Value("R")
}
import UserRole._

import scala.concurrent.{Await,Future}
import scala.concurrent.duration._
val db = Database.forConfig("chapter05")
def exec[T](action: DBIO[T]): T = Await.result(db.run(action), 4.seconds)
import scala.concurrent.ExecutionContext.Implicits.global
```

Modify your solution to the previous exercise to store the value in the database as an integer.

If you see an unrecognized user role value, default it to a `UserRole.Regular`.

<div class="solution">
The only change to make is to the mapper, to go from a `UserRole` and `String`, to a `UserRole` and `Int`:

```scala mdoc
implicit val userRoleIntMapper =
  MappedColumnType.base[UserRole, Int](
    _.id,
    v => UserRole.values.find(_.id == v) getOrElse Regular
  )
```
</div>


### Custom Boolean

Messages can be high priority or low priority.

The database is a bit of a mess:

- The database value for high priority messages will be: `y`, `Y`, `+`, or `high`.

- For low priority messages the value will be: `n`, `N`, `-`, `lo`, or `low`.

Go ahead and model this with a sum type.

<div class="solution">
This is similar to the `Flag` example above,
except we need to handle multiple values from the database.

```scala mdoc
sealed trait Priority
case object HighPriority extends Priority
case object LowPriority  extends Priority

implicit val priorityType =
  MappedColumnType.base[Priority, String](
    flag => flag match {
      case HighPriority => "y"
      case LowPriority  => "n"
    },
    str => str match {
      case "Y" | "y" | "+" | "high"         => HighPriority
      case "N" | "n" | "-" | "lo"   | "low" => LowPriority
  })
```

The table definition would need a `column[Priority]`.
</div>

### Turning a Row into Many Case Classes

Our `HList` example mapped a table with many columns.
It's not the only way to deal with lots of columns.

Use custom functions with `<>` and map `UserTable` into a tree of case classes.
To do this you will need to define the schema, define a `User`, insert data, and query the data.

To make this easier, we're just going to map six of the columns.
Here are the case classes to use:

```scala mdoc:silent
case class EmailContact(name: String, email: String)
case class Address(street: String, city: String, country: String)
case class User(contact: EmailContact, address: Address, id: Long = 0L)
```

You'll find a definition of `UserTable` that you can copy and paste in the example code in the file _chapter-05/src/main/scala/nested_case_class.scala_.

<div class="solution">

In our huge legacy table we will use custom functions with `<>`...

```scala mdoc
class LegacyUserTable(tag: Tag) extends Table[User](tag, "legacy") {
  def id           = column[Long]("id", O.PrimaryKey, O.AutoInc)
  def name         = column[String]("name")
  def age          = column[Int]("age")
  def gender       = column[Char]("gender")
  def height       = column[Float]("height")
  def weight       = column[Float]("weight_kg")
  def shoeSize     = column[Int]("shoe_size")
  def email        = column[String]("email_address")
  def phone        = column[String]("phone_number")
  def accepted     = column[Boolean]("terms")
  def sendNews     = column[Boolean]("newsletter")
  def street       = column[String]("street")
  def city         = column[String]("city")
  def country      = column[String]("country")
  def faveColor    = column[String]("fave_color")
  def faveFood     = column[String]("fave_food")
  def faveDrink    = column[String]("fave_drink")
  def faveTvShow   = column[String]("fave_show")
  def faveMovie    = column[String]("fave_movie")
  def faveSong     = column[String]("fave_song")
  def lastPurchase = column[String]("sku")
  def lastRating   = column[Int]("service_rating")
  def tellFriends  = column[Boolean]("recommend")
  def petName      = column[String]("pet")
  def partnerName  = column[String]("partner")

  // The tuple representation we will use:
  type Row = (String, String, String, String, String, Long)

  // One function from Row to User
  def pack(row: Row): User = User(
    EmailContact(row._1, row._2),
    Address(row._3, row._4, row._5),
    row._6
  )

  // Another method from User to Row:
  def unpack(user: User): Option[Row] = Some(
    (user.contact.name, user.contact.email, user.address.street,
     user.address.city, user.address.country, user.id)
  )

  def * = (name, email, street, city, country, id).<>(pack, unpack)
}

lazy val legacyUsers = TableQuery[LegacyUserTable]
```

We can insert and query as normal:


```scala mdoc
exec(legacyUsers.schema.create)

exec(
  legacyUsers += User(
    EmailContact("Dr. Dave Bowman", "dave@example.org"),
    Address("123 Some Street", "Any Town", "USA")
   )
)
```

And we can fetch results:

```scala mdoc
exec(legacyUsers.result)
```

You can continue to select just some fields:

```scala mdoc
exec(legacyUsers.map(_.email).result)
```

However, notice that if you used `legacyUsers.schema.create`, only the columns defined in the default projection were created in the H2 database:

```scala mdoc
legacyUsers.schema.createStatements.foreach(println)
```
</div>


