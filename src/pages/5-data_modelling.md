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

- 空の`HList`は，シングルトンオブジェクト`HNil`で表現される
- より長い`HList`は、`::`演算子を使って値を前置することで形成され、新しい型のリストを作成する

それぞれの`HList`の型と値が互いに影響し合っていることに注意してください。
`longerHList`は`Int`型、`String`型、`Boolean`型の値からなり、その型は`Int`型、`String`型、`Boolean`型からなります。
要素の型が保持されているので、それぞれの正確な型を考慮したコードを書くことができます。

Slickは、カラムの`HList`とその値の`HList`を対応付ける`ProvenShape`を生成することができます。
例えば、`Rep[Int] :: Rep[String] :: HNil`のシェイプは`Int :: String :: HNil`の型の値をマッピングします。

#### Using HLists Directly

上記の例では、`HList` を使って大きなテーブルをマッピングすることができます。デフォルトプロジェクションはこんな感じです

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

`HList`の型や値を書くのは面倒でエラーが出やすいので、`AttrHList`という型のエイリアスを導入して対応しています。



```scala mdoc:invisible
import slick.jdbc.H2Profile.api._
import scala.concurrent.{Await,Future}
import scala.concurrent.duration._

val db = Database.forConfig("chapter05")

def exec[T](action: DBIO[T]): T = Await.result(db.run(action), 4.seconds)
```

このテーブルを扱うには、`AttrHList`のインスタンスを挿入、更新、選択、修正する必要があります。例えば、以下のようになります：

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

パターン・マッチや、`HList`上で定義された`head`、`apply`、`drop`、`fold`などの様々な型保存メソッドを用いて、クエリ結果HListから値を抽出することができます。


```scala mdoc
val id: Long = myAttrs.head
val productId: Long = myAttrs.tail.head
val name1: String = myAttrs(2)
val value1: Int = myAttrs(3)
```

#### Using HLists and Case Classes

実際には、HList表現を通常のクラスにマッピングして、作業をしやすくしたいものです。
Slickの`<>`演算子は、タプル型と同様に`HList`型でも動作します。
これを使うには、ケースクラスの`apply`と`unapply`の代わりに独自のマッピング関数を作る必要がありますが、それ以外はタプルで見たのと同じアプローチです。

ただし、`mapTo`マクロは、`HList`とケースクラスの間のマッピングを生成してくれます。


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

このようなパターンがあることに気づくでしょう。

```scala
 def * = (some hlist).mapTo[case class with the same fields]
```


このように、私たちのテーブルは、Scalaのケースクラスで定義されています。ケースクラスを使って、普通にデータを照会したり変更したりすることができます。

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

スキーマの決定的な記述がコードである場合もあれば、データベースそのものである場合もあります。
後者は、レガシーデータベースや、スキーマがSlickアプリケーションから独立して管理されているデータベースで作業する場合に当てはまります。

データベースが組織内のソース・トゥルースと見なされる場合、[Slick code generator][link-ref-gen] は重要なツールとなります。
データベースに接続し、テーブル定義を生成し、生成されたコードをカスタマイズすることができます。行数の多いテーブルの場合、HList表現が生成されます。

手作業でスキーマをリバースエンジニアリングするよりも便利です。

</div>

## Table and Column Representation

行をどのように表現し、マッピングするかがわかったところで、次はテーブルとそのカラムの表現について詳しく見ていきましょう。
特に、NULL可能なカラム、外部キー、主キー、複合キーの詳細、テーブルに適用できるオプションについて調べます。


### Nullable Columns {#null-columns}

SQLで定義されたカラムは、デフォルトでNULL可能です。
つまり、値として `NULL` を含むことができます。もしNULL可能なカラムが必要なら、Scalaでは`Option[T]`として自然にモデル化されます。

オプションのメールアドレスを持つ`User`を作ってみましょう。


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


メールアドレスの有無にかかわらず、ユーザーを挿入することができます。

```scala mdoc
val program = (
  users.schema.create >>
  (users += User("Dave", Some("dave@example.org"))) >>
  (users += User("HAL"))
)

exec(program)
```

そしてselectクエリで取得できます。

```scala mdoc
exec(users.result).foreach(println)
```

ここまでは普通ですね。
意外なのは、メールアドレスのない行をすべて選択する方法です。メールアドレスがない1行を見つけるには、次のようにするとよいでしょう。



```scala mdoc
// Don't do this
val none: Option[String] = None
val badQuery = exec(users.filter(_.email === none).result)
```

データベースにはメールアドレスがない行が1つあるにもかかわらず、このクエリでは結果が出ません。

データベース管理のベテランなら、SQLの面白い癖をよくご存じでしょう。
例えば、SQL式 `'Dave' = 'HAL'` は `false` と評価されますが、式 `'Dave' = null` は `null` と評価されます。

上記のSlickクエリは次のようになります。

~~~ sql
SELECT * FROM "user" WHERE "email" = NULL
~~~

SQL式 `"email" = null`は、`"email"`の任意の値に対して`null`と評価されます。
SQLの`null`はfalseyな値なので、このクエリが値を返すことはありません。

この問題を解決するために、SQLには`IS NULL` と `IS NOT NULL` という2つの演算子が用意されています。
Slickでは、任意の`Rep[Option[A]]`に対する`isEmpty`と`isDefined`というメソッドで提供されています。


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


等号チェックを`isEmpty`に置き換えることで、このクエリを修正することができます。

```scala mdoc
val myUsers = exec(users.filter(_.email.isEmpty).result)
```

これは以下のSQLに変換されます。

~~~ sql
SELECT * FROM "user" WHERE "email" IS NULL
~~~


### Primary Keys

第1章では、`O.PrimaryKey`と`O.AutoInc`のカラムオプションを使用して、`id`フィールドを設定することから始まりました。


~~~ scala
def id = column[Long]("id", O.PrimaryKey, O.AutoInc)
~~~


これらのオプションは、2つのことを行います。

- DDL文のために生成されたSQLを修正する

- `O.AutoInc`は、INSERT文で生成されるSQLから対応するカラムを削除し、データベースが自動インクリメントする値を挿入できるようにする

第1章では、Slickがinsert文で値をスキップすることを承知で、`O.AutoInc`とデフォルトIDが`0L`であるケースクラスを組み合わせました。

```scala
case class User(name: String, id: Long = 0L)
```

私たちはこのスタイルのシンプルさを気に入っていますが、開発者の中には、主キー値を`Option`で包むことを好む人もいます。

```scala
case class User(name: String, id: Option[Long] = None)
```


このモデルでは、保存されていないレコードの主キーとして`None`を、保存されたレコードの主キーとして`Some`を使用します。
この方法にはメリットとデメリットがあります。

- メリットとして、保存されていないレコードを特定しやすくなる
- デメリットとしては、クエリで使用するために主キーの値を取得するのが困難であること

この機能を実現するために、`UserTable`に加えるべき変更点を見てみましょう。

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

ここで重要なのは、主キーをデータベースのオプションにしたくないということです。
挿入時にデータベースが主キーを割り当てるので、データベースクエリで`None`を取得することはできないのです。

NULLでないデータベースのカラムを、オプションのフィールド値にマッピングする必要があります。
これは、デフォルトのプロジェクションの `?` メソッドで処理され、`Rep[A]` を `Rep[Option[A]]` に変換しています。


### Compound Primary Keys

カラムを主キーとして宣言する2つ目の方法があります。

~~~ scala
def id = column[Long]("id", O.AutoInc)
def pk = primaryKey("pk_id", id)
~~~

この別個のステップは、この場合、あまり大きな違いはありません。
これは、カラムの定義をキー制約から分離するもので、スキーマに含まれることを意味します。

~~~ sql
ALTER TABLE "user" ADD CONSTRAINT "pk_id" PRIMARY KEY("id")
~~~

`primaryKey`メソッドは、2つ以上のカラムを含む複合プライマリキーを定義する場合に便利です。

ここでは、人々が部屋でチャットする機能を追加して見てみましょう。まず、ルームを保存するためのテーブルが必要ですが、これは簡単です。


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

次に、ユーザーと部屋を関連付けるテーブルが必要です。
これを入居者テーブル（`OccupantTable`）と呼ぶことにします。このテーブルには自動生成された主キーを与えるのではなく、ユーザーと部屋のIDの複合キーにします。



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

タプルまたはカラムのHListを使用して複合主キーを定義することができます（Slickは`ProvenShape`を生成し、それを検査して関連するカラムのリストを見つけます）。
入居者テーブルに対して生成されたSQLは以下の通りです。

~~~ sql
CREATE TABLE "occupant" (
  "room" BIGINT NOT NULL,
  "user" BIGINT NOT NULL
)

ALTER TABLE "occupant"
ADD CONSTRAINT "room_user_pk" PRIMARY KEY("room", "user")
~~~

入居者テーブルの使い方は他のテーブルと変わりません。

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

もちろん、DaveをAir Lockに2回入れようとすれば、データベースは主キーの重複を指摘します。


### Indices

インデックスを使用することで、ディスク使用量を増やす代わりに、データベースクエリの効率を上げることができます。
インデックスの作成と使用は、データベースアプリケーションごとに異なる最高のデータベース魔術であり、この本の範囲をはるかに超えています。
しかし、Slickでインデックスを定義するための構文は簡単です。ここに、インデックスを2回呼び出すテーブルがあります。

```scala mdoc
class IndexExample(tag: Tag) extends Table[(String,Int)](tag, "people") {
  def name = column[String]("name")
  def age  = column[Int]("age")

  def * = (name, age)

  def nameIndex = index("name_idx", name, unique=true)
  def compoundIndex = index("c_idx", (name, age), unique=true)
}
```

`nameIndex`により生成される対応するDDL文は、次のようになります。


~~~ sql
CREATE UNIQUE INDEX "name_idx" ON "people" ("name")
~~~

主キーと同じように、複数のカラムに複合インデックスを作成することができます。
この場合（`compoundIndex`）、対応するDDL文は次のようになります。


~~~ sql
CREATE UNIQUE INDEX "c_idx" ON "people" ("name", "age")
~~~


### Foreign Keys {#fks}

外部キーは複合主キーと同様の方法で宣言します。
`foreignKey`メソッドは、4つの必須パラメータを取ります。


 * name

 * column または columns（外部キーを構成するもの）

 * 外部キーが所属する`TableQuery`

 * 与えられた`TableQuery[T]`のカラムをパラメータとして、`T`のインスタンスを返す関数

ここでは、外部キーを使って、メッセージとユーザーを結びつける方法を説明します。
メッセージの定義を変更し、名前の代わりに送信者の ID を参照するようにします。

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

`sender`のカラムは、Stringの代わりにLongになりました。
また、`sender`というメソッドを定義し、`senderId`を`user id`に結びつける外部キーを提供しています。

この外部キーによって、2つのことが可能になります。まず、Slickが生成するDDL文に制約が追加されます。


~~~ sql
ALTER TABLE "message" ADD CONSTRAINT "sender_fk"
  FOREIGN KEY("sender") REFERENCES "user"("id")
  ON UPDATE NO ACTION
  ON DELETE NO ACTION
~~~

<div class="callout callout-info">

**On Update and On Delete**

外部キーは、保存するデータについて一定の保証をするものです。
今回取り上げたケースでは、新しいメッセージをうまく挿入するには、user テーブルに送信者がいなければなりません。

では、user行に何か変更があった場合はどうなるのでしょうか？トリガーされる[参照アクション][link-fkri]はいくつもあります。デフォルトでは何も起こりませんが、これを変更することができます。

例を見てみましょう。あるユーザーを削除し、そのユーザーに関連するすべてのメッセージを削除したいとします。アプリケーションでそれを行うこともできますが、これはデータベースが提供してくれるものです。


```scala mdoc
class AltMsgTable(tag: Tag) extends Table[Message](tag, "message") {
  def id       = column[Long]("id", O.PrimaryKey, O.AutoInc)
  def senderId = column[Long]("sender")
  def content  = column[String]("content")

  def * = (senderId, content, id).mapTo[Message]

  def sender = foreignKey("sender_fk", senderId, users)(_.id, onDelete=ForeignKeyAction.Cascade)
}
```

Slickのスキーマコマンドがテーブルに対して実行されている、またはSQLの`ON DELETE CASCADE`アクションがデータベースに手動で適用されている場合、
次のアクションはユーザーテーブルからHALを削除し、HALが送信したメッセージもすべて削除します。


```scala mdoc:silent
users.filter(_.name === "HAL").delete
```
Slickは5つのアクションのうち、`onUpdate`と`onDelete`をサポートしています
Slick supports `onUpdate` and `onDelete` for the five actions:

------------------------------------------------------------------
Action          Description
-----------     --------------------------------------------------
`NoAction`      デフォルト

`Cascade`       参照されるテーブルの変更は、参照するテーブルの変更を誘発する。この例では、ユーザーを削除すると、そのユーザーのメッセージも削除されます

`Restrict`      変更が制限され、制約違反の例外が発生した。この例では、メッセージを投稿したユーザーを削除することは許可されません。 

`SetNull`       更新された値を参照しているカラムはNULLに設定されます。 

`SetDefault`    参照するカラムのデフォルト値が使用されます。デフォルト値については、本章で後述する[Table and Column Modifiers](#schema-modifiers)で説明します。 
                
-----------     --------------------------------------------------

</div>

第二に、外部キーは結合で使用できるクエリを提供します。[次の章](#joins)で結合の詳細を説明しますが、ここでは使用例を説明するために簡単な結合を示します：

```scala mdoc
val q = for {
  msg <- messages
  usr <- msg.sender
} yield (usr.name, msg.content)
```

これは、以下クエリと同等です。

~~~ sql
SELECT u."name", m."content"
FROM "message" m, "user" u
WHERE "id" = m."sender"
~~~

そして、データベースへの入力が完了したら...

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

このクエリでは、送信者名（IDではない）と対応するメッセージを示す、次のような結果が得られました。

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

外部キーを定義することで、データベースのテーブルを定義する順番に制約が生じます。
上の例では、`MessageTable`から`UserTable`への外部キーは、Scalaのコードでは後者の定義を前者よりも上に置く必要があります。

順序制約があると、複雑なスキーマを書くのが難しくなります。幸いなことに、`def`と`lazy val`を使うことで回避することができます。

原則として、`TableQuery`と`def`の外部キーには`lazy val`を使用します（カラム定義との整合性をとるため）。

</div>

### Column Options {#schema-modifiers}

このセクションの最後に、カラムとテーブルの修飾子について説明します。これらは、SQLレベルでカラムのデフォルト値、サイズ、データ型を調整することができます。

我々はすでに、`O.PrimaryKey` と `O.AutoInc` という2つのカラム・オプションの例を見てきました。
カラム・オプションは、[`ColumnOption`][link-slick-column-options] で定義され、これまで見てきたように、`O` を介してアクセスされます。

次の例では、4つの新しいオプションを導入しています：`O.Length`、`O.SqlType`、`O.Unique`、および `O.Default` です。



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

この例では、4つのことを行っています。

1. `O.Length`を使用して、`name`カラムに最大長を与えています。
   これにより、DDL文のカラムの型が変更されます。`O.Length`のパラメータは、最大長を指定する`Int`と、長さを可変とするかどうかを示す`Boolean`です。
   ブール値を`true`に設定すると、SQLカラムのタイプは`VARCHAR`になり、`false`に設定すると、タイプは`CHAR`になります。

2. `O.Default`を使用して、`name`カラムにデフォルト値を与えています。これにより、DDL文のカラム定義に`DEFAULT`句が追加されます。

3. `email`カラムに一意性制約を追加しました。

4. `O.SqlType`を使用して、データベースが使用する正確な型を制御しています。ここで許可される値は、使用しているデータベースによって異なります。



## Custom Column Mappings

私たちは、アプリケーションにとって意味のある型を扱いたいと考えています。
これは、データベースが使用する単純な型から、より開発者に優しい型にデータを変換することを意味します。

Slickがタプルやカラムの`HList`をケース・クラスにマッピングする機能はすでに見てきました。
しかし、これまでのところ、ケースクラスのフィールドは`Int`や`String`などの単純な型に限定されていました、

Slickでは、個々のカラムをScalaの型にマッピングする方法を制御することも可能です。
例えば、日付や時間に関連するものには[Joda Time][link-jodatime]の`DateTime`クラスを使用したいとします。SlickはJoda Time[^time]のネイティブサポートを提供しませんが、Slickの`ColumnType`型クラスで簡単に実装できます。

[^time]: しかし、Slick 3.3.0 からは `java.time.Instant`, `LocalDate`, `LocalTime`, `LocalDateTime`, `OffsetTime`, `OffsetDateTime`, `ZonedDateTime` が内蔵サポートされています。
古い Joda Time ライブラリよりも、これらのライブラリを使用したい場合が非常に多いでしょう。

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

ここで提供するのは、`MappedColumnType.base`に対する2つの関数です。

- 一つは`DateTime`から
  をデータベースに適した `java.sql.Timestamp` に変換します。

- もう一つはその逆で、`Timestamp`を`DateTime`に変換するものです。

一度このカスタムカラムの型を宣言したら
`DateTimes`を含むカラムを自由に作成することができます。


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

REPLでフォローしていた方、
今頃はテーブルと行の束になっていることでしょう。
今こそ、それらをすべて削除する良い機会です。

REPLを終了して、再起動することができます。
H2はデータをメモリに保持しています、
H2はメモリ上にデータを保持しているので、これをオン/オフすることは、データベースをリセットする1つの方法です。

あるいは、アクションを使うこともできます。


```scala mdoc
val schemas = (users.schema ++
  messages.schema          ++
  occupants.schema         ++
  rooms.schema)

exec(schemas.drop)
```
</div>

今回修正した `MessageTable` の定義では、`DateTime` のタイムスタンプを含む `Message` を直接扱うことができます。
`DateTime`のタイムスタンプを含む`Message`を直接扱えるようになりました、
面倒な型変換をすることなく、直接 `DateTime` のタイムスタンプを扱うことができます。


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

データベースの行を取得すると、`timestamp` フィールドが自動的に期待する `DateTime` 値に変換されます。

```scala mdoc
exec(messages.filter(_.id === msgId).result)
```

セマンティックな型を扱うこのモデルは、Scalaの開発者にとって魅力的なものです。
あなたのアプリケーションで `ColumnType` を使用することを強くお勧めします、



### Value Classes {#value-classes}

```scala mdoc:reset-object:invisible
import slick.jdbc.H2Profile.api._
import scala.concurrent.{Await,Future}
import scala.concurrent.duration._
val db = Database.forConfig("chapter05")
def exec[T](action: DBIO[T]): T = Await.result(db.run(action), 4.seconds)
import scala.concurrent.ExecutionContext.Implicits.global
```

現在、主キーのモデルとして`Long`を使用しています。
これはデータベースレベルでは良い選択ですが、アプリケーションのコードではあまり良くありません、
アプリケーションのコードにとってはあまり良い選択ではありません。

問題は、愚かな間違いを犯す可能性があることです、
例えば、`Message`の主キーを使って `User` を主キーで検索しようとします。
どちらも`Long`ですが、それを比較しようとすると意味がありません。
しかし、このコードはコンパイル可能であり、結果を返す可能性もあります。
しかし、それは間違った結果である可能性が高いでしょう。

このような問題は、型を使って防ぐことができます。
基本的なアプローチは、主キーをモデル化する[value class][link-scala-value-classes]を使うことです


```scala mdoc
case class MessagePK(value: Long) extends AnyVal
case class UserPK(value: Long) extends AnyVal
```

value classは、コンパイル時に値を包むラッパーです。
実行時には、このラッパーは取り除かれます、
実行中のコードに割り当てやパフォーマンスのオーバーヘッド[^vcpoly]を残しません。

[^vcpoly]: これは完全なコストフリーではありません。例えば、ポリモーフィックなメソッドに渡される場合など、[値の割り当てが必要な状況][link-scala-value-classes]があります。

value classを使用するにはSlickに `ColumnType` を提供し、これらの型をテーブルで使用する必要があります。
これは、Joda Timeの`DateTime`に使用したのと同じプロセスです。


```scala mdoc
implicit val messagePKColumnType =
  MappedColumnType.base[MessagePK, Long](_.value, MessagePK(_))

implicit val userPKColumnType =
   MappedColumnType.base[UserPK, Long](_.value, UserPK(_))
```

このような型クラスのインスタンスをすべて定義するのは時間がかかるものです。
特にスキーマの各テーブルに対して定義する場合は、時間がかかります。
幸いなことに、Slickは `MappedTo` と呼ばれるショートハンドを提供しており、これを利用することができます。

```scala mdoc:reset-object:invisible
import slick.jdbc.H2Profile.api._
```

```scala mdoc
case class MessagePK(value: Long) extends AnyVal with MappedTo[Long]
case class UserPK(value: Long) extends AnyVal with MappedTo[Long]
```


`MappedTo` を使用する場合、別途 `ColumnType` を定義する必要はありません。

`MappedTo` は以下のようなクラスで動作します。

- 基礎となるデータベースの値を返す `value` というメソッドを持つ。

- データベースの値から Scala の値を生成するためのシングルパラメータなコンストラクタを持つ

value classは `MappedTo` パターンに非常に適しています。

カスタム主キー型を使用するために、テーブルを再定義してみましょう。
ここでは、`User`... を変換します。

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

`Message`は以下のようにになります。

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

どのように明示できたかに注目してください。
`User.id` と `Message.senderId` は `UserPK` です。
そして、`Message.id`は`MessagePK`です。

正しい種類のキーがあれば、値を参照することができます。

```scala mdoc
users.filter(_.id === UserPK(0L))
```

...しかし、誤って主キーを混在させようとすると、失敗することがわかります。

```scala mdoc:fail
users.filter(_.id === MessagePK(0L))
```


value classは、コードをより安全に、より読みやすくするための低コストな方法です。
必要なコードの量はわずかです。
しかし、大規模なデータベースでは、オーバーヘッドになる可能性があります。
この問題を解決するには、コード生成を利用するか、主キー型を一般化します。

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

このアプローチにより、多くの主キー型定義に煩わされることなく、型安全性を実現
を実現しますが、多くの主キー型定義のような煩雑な作業は必要ありません。
アプリケーションの性質によりますが、あなたにとって便利なものになるでしょう。

一般的なポイントは、Scala の型システム全体を使ってデータベースの主キー、外部キー、行、カラムを表現できるということです。
これは非常に価値のあることであり、見過ごすわけにはいきません。


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


私たちは、データのモデリングにケースクラスを多用してきました。
代数的データ型の言語を使ったケースクラスは "product type"です
(フィールド・タイプの結合から作られる）。
代数的データ型のもう一つの一般的な形式は、"sum type"と呼ばれるものです。
これは、他のデータ型の和から作られます。
ここでは、これらのモデル化について説明します。

例として、`Message` クラスにフラグを追加してみましょう。
にフラグを追加して、メッセージを重要、不快、スパムとしてモデル化してみましょう。
これを行うための自然な方法は、密封された特質である
とケースオブジェクトのセットを作成します：

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

データベースでフラグを表現する方法はいくつもあります。
ここでは、文字で表現することにします: `!`、`X`、`$` とします。
マッピングを管理するために、新しい `ColumnType` が必要です。


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


コンパイラがすべてのケースをカバーしていることを確認できるため、私たちはsum typeを好みます。
もし、新しいフラグ（`OffTopic`かもしれません）を追加すると、`Flag => Char` 関数にそれを追加するまでコンパイラは警告を発します。
Scalaコンパイラの `-Xfatal-warnings` オプションを有効にすることで、このコンパイラの警告をエラーにすることができます。
このオプションを有効にすることで、全てをカバーするまでアプリケーションを出荷できないようにすることができます。

`Flag`の使い方は、他のカスタム型と同じです。

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

フラグを使ったメッセージを簡単に挿入することができます。

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




また、特定のフラグを持つメッセージの問い合わせも可能です。
しかし、コンパイラに型について少し手助けをしてあげる必要があります。

```scala mdoc
exec(
  messages.filter(_.flag === (Important : Flag)).result
)
```


```scala mdoc
object Flags {
  val important : Flag = Important
  val offensive : Flag = Offensive
  val spam      : Flag = Spam

  val action = messages.filter(_.flag === Flags.important).result
}
```
次に、フィルター式を構築するためのカスタム構文を定義することができます。

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


