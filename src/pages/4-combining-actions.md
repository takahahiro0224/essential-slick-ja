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

val db = Database.forConfig("chapter04")

def exec[T](action: DBIO[T]): T = Await.result(db.run(action), 2.seconds)

def freshTestData = Seq(
  Message("Dave", "Hello, HAL. Do you read me, HAL?"),
  Message("HAL",  "Affirmative, Dave. I read you."),
  Message("Dave", "Open the pod bay doors, HAL."),
  Message("HAL",  "I'm sorry, Dave. I'm afraid I can't do that.")
)

exec(messages.schema.create andThen (messages ++= freshTestData))
```
# Combining Actions {#combining}

複数のアクションで構成されたコードを書くことがあると思います。あるアクションが次々と実行される単純なシーケンスが必要な場合もあれば、あるアクションが他のアクションの結果に依存するような、より高度なものが必要な場合もあるでしょう。

Slickでは、_action combinators_ を使用して、いくつかのアクションを1つのアクションに変換します。そして、この結合されたアクションを、単一のアクションと同様に実行することができます。また、これらの結合されたアクションを _transaction_ で実行することもできます。

この章では、これらのコンビネータに焦点を当てます。`map`、`fold`、`zip` などは Scala のコレクションライブラリでおなじみのものです。`sequence`や`asTry`のようなものは、あまり馴染みがないかもしれません。この章では、それらの多くをどのように使うかについて例を挙げて説明します。

これはSlickの重要なコンセプトです。アクションを組み合わせることに慣れるために、時間をかけるようにしてください。

## Combinators Summary

複数のアクションを実行すると、それぞれのアクションを実行し、その結果を使用して、別のアクションを実行するという誘惑にかられるかもしれません。この場合、複数のFutureを処理する必要があり、できる限り避けることをお勧めします。

その代わりに、アクションを実行するための面倒な詳細ではなく、アクションとそれらがどのように組み合わされるかに焦点を当てます。Slickはこれを可能にするためにコンビネーターのセットを提供します。

詳細な説明に入る前に、以下の2つの表を見てみてください。これらは、アクションで利用できる主要なメソッドと、`DBIO`で利用できるコンビネータをリストアップしています。


--------------------------------------------------------------------
Method              Arguments                       Result Type     
------------------- -----------------------------   ----------------
`map` (EC)           `T => R`                        `DBIO[R]`        

`flatMap` (EC)       `T => DBIO[R]`                  `DBIO[R]`

`filter` (EC)         `T => Boolean`                  `DBIO[T]`

`named`             `String`                        `DBIO[T]`

`zip`               `DBIO[R]`                       `DBIO[(T,R)]`

`asTry`                                             `DBIO[Try[T]]`

`andThen` or `>>`   `DBIO[R]`                       `DBIO[R]`

`andFinally`        `DBIO[_]`                       `DBIO[T]`

`cleanUp` (EC)       `Option[Throwable]=>DBIO[_]`    `DBIO[T]`

`failed`                                            `DBIO[Throwable]`
----------------------------------------------------------------------

: Combinators on action instances of `DBIOAction`, specifically a `DBIO[T]`.
  Types simplified.
  (EC) Indicates an execution context is required.

: `DBIOAction`（具体的には`DBIO[T]`）のアクションインスタンスに対するコンビネータ。型は簡略化されています。(EC) は実行コンテキストが必要であることを示しています。


---------------------------------------------------------------------------
Method       Arguments                       Result Type                   
------------ ------------------------------- ------------------------------
`sequence`   `TraversableOnce[DBIO[T]]`      `DBIO[TraversableOnce[T]]`

`seq`        `DBIO[_]*`                      `DBIO[Unit]`                   

`from`       `Future[T]`                     `DBIO[T]`

`successful` `V`                             `DBIO[V]`

`failed`     `Throwable`                     `DBIO[Nothing]`

`fold` (EC)   `(Seq[DBIO[T]], T)  (T,T)=>T`   `DBIO[T]`
----------------------------------------------------------------------------

: `DBIO[T]`オブジェクト対するコンビネータ。型は簡略化されています。(EC) は実行コンテキストが必要であることを示しています。 

## Combinators in Detail

### `andThen` (or `>>`)

あるアクションを別のアクションの後に実行する最も簡単な方法は、`andThen`です。組み合わせたアクションは両方とも実行されますが、2番目の結果だけが返されます：

```scala mdoc
val reset: DBIO[Int] =
  messages.delete andThen messages.size.result

exec(reset)
```

最初のクエリの結果は無視されるので、使うことはできません。後で、`flatMap`で結果を利用して、次に実行するアクションを選択できるようにする方法を説明します。


<div class="callout callout-warning">

**Combined Actions Are Not Automatically Transactions**

デフォルトでは、アクションを組み合わせても、1つのトランザクションにはなりません。[この章の終わり][Transactions]で、組み合わせたアクションを単一トランザクションで実行するのが非常に簡単であることを確認します：

```scala
db.run(actions.transactionally)
```
</div>

### `DBIO.seq`

実行したいアクションがたくさんある場合、`DBIO.seq`を使ってそれらを組み合わせることができます：


```scala mdoc:silent
val resetSeq: DBIO[Unit] =
  DBIO.seq(messages.delete, messages.size.result)
```

これはどちらかというと`andThen`でアクションを組み合わせるようなものですが、最後の値まで捨てられるのです。


### `map`

アクションに対するマッピングは、データベースからの値の変換を設定する方法です。変換は、データベースから返されたアクションの結果に対して実行されます。

例として、メッセージの内容を返すが、テキストを反転させるアクションを作成することができます：


```scala mdoc
// Restore the data we deleted in the previous section
exec(messages ++= freshTestData)

import scala.concurrent.ExecutionContext.Implicits.global

val text: DBIO[Option[String]] =
  messages.map(_.content).result.headOption

val backwards: DBIO[Option[String]] =
  text.map(optionalContent => optionalContent.map(_.reverse))

exec(backwards)
```

ここでは、`backwards`というアクションを作成し、実行すると、`text`アクションの結果に関数が適用されるようにしました。この場合、関数はオプションの`String`に`reverse`を適用するものです。

なお、この例では、mapを3回使用しています：

- `Option[String]`の結果に`reverse`を適用するための`Option map`。
- `content`カラムだけを選択するためのクエリ上の`map`。
- `map`をアクションに追加することで、アクションが実行されたときに結果が変換されるようになります。
- 
コンビネーターだらけですね！

この例では、`Option[String]`を別の`Option[String]`に変換しています。`map`が値の型を変更すると、`DBIO`の型も変更されるのはご想像の通りです：


```scala mdoc
text.map(os => os.map(_.length))
```

`DBIOAction`の最初の型パラメータは、`Option[String]`ではなく、`Option[Int]`（lengthがIntを返すため）になっていることに注意してください。

<div class="callout callout-info">

**Execution Context Required**

メソッドには、Execution Contextが必要なものと必要でないものがあります。例えば、`map`はそうですが、`andThen`はそうではありません。どうなんでしょう？

その理由は、`map`によってアクションを結合する際に任意のコードを呼び出すことができるからです。Slickはそのコードを自分のExecution Contextで実行させることができません。なぜなら、Slickのスレッドを長時間拘束することになるのかどうか、知る術がないからです。

一方、`andThen`のようにカスタムコードなしでアクションを組み合わせるメソッドは、Slick自身のExecution Context上で実行することができます。したがって、`andThen`で利用するためのExecution Contextは必要ありません。

Execution Contextが必要かどうかは、コンパイラが教えてくれます：


~~~
Cannot find an implicit ExecutionContext. You might pass
  an (implicit ec: ExecutionContext) parameter to your method
  or import scala.concurrent.ExecutionContext.Implicits.global.
~~~

Slickのマニュアルでは[Database I/O Actions][link-ref-actions]のセクションでこのことについて説明しています。
</div>


### `DBIO.successful` and `DBIO.failed`

アクションを組み合わせる際に、単純な値を表すアクションを作成する必要がある場合があります。Slickでは、そのために`DBIO.successful`を用意しています：

```scala mdoc:silent
val ok: DBIO[Int] = DBIO.successful(100)
```

`flatMap`について議論するときにその例を見てみましょう。

また失敗の場合は`Throwable`が値となります。

```scala mdoc:silent
val err: DBIO[Nothing] =
  DBIO.failed(new RuntimeException("pod bay door unexpectedly locked"))
```

これは、本章の後半で取り上げるトランザクションの内部で特に重要な役割を果たします。

### `flatMap`

ああ `flatMap`. 素晴らしい `flatMap`.

このメソッドは、アクションを連続させ、各ステップで何をしたいかを決める力を与えてくれます。
`flatMap`のシグネチャーは、他の場所で見かける`flatMap`と同じような感じになっているはずです。

~~~ scala
// Simplified:
def flatMap[S](f: R => DBIO[S])(implicit e: ExecutionContext): DBIO[S]
~~~

つまり、あるアクションの値に依存し、別のアクションを評価する関数を`flatMap`に与えるのです。

例として、クルーのメッセージをすべて削除し、何件削除されたかというメッセージを投稿するメソッドを書いてみましょう。これには、私たちがよく知っている`INSERT`と`DELETE`が含まれます：

```scala mdoc:silent
val delete: DBIO[Int] =
  messages.delete

def insert(count: Int) =
  messages += Message("NOBODY", s"I removed ${count} messages")
```

`flatMap`でまずできることは、これらのアクションを順番に実行することです：

```scala mdoc
import scala.concurrent.ExecutionContext.Implicits.global

val resetMessagesAction: DBIO[Int] =
  delete.flatMap{ count => insert(count) }

// resetMessagesAction: DBIO[Int] = FlatMapAction(
//   slick.jdbc.JdbcActionComponent$DeleteActionExtensionMethodsImpl$$anon$4@586ff5d7,
//   <function1>,
//   scala.concurrent.impl.ExecutionContextImpl$$anon$3@25bc3124[Running, parallelism = 2, size = 1, active = 0, running = 0, steals = 111, tasks = 0, submissions = 0]
// )

exec(resetMessagesAction)
// res5: Int = 1
```

表示される`1`は`insert`の結果で、挿入された行の数です。

この1回の操作で、期待通りの2つのSQL式が生成されます：


``` sql
delete from "message";
insert into "message" ("sender","content")
  values ('NOBODY', 'I removed 4 messages');
```

シーケンスだけでなく、`flatMap`はどのアクションを実行するかを制御することもできます。これを説明するために、`resetMessagesAction`のバリエーションとして、最初のステップでメッセージが削除されなかった場合、メッセージを挿入しないものを作成することにします：


```scala mdoc:silent:silent
val logResetAction: DBIO[Int] =
  delete.flatMap {
    case 0 => DBIO.successful(0)
    case n => insert(n)
  }
```

メッセージが挿入されなかった場合は、`0`という結果が正しいと判断しています。しかし、ここで重要なのは、`flatMap`はアクションの組み合わせ方を任意にコントロールできるということです。

時々、コンパイラが`flatMap`について文句を言い、型を特定するためにあなたの助けが必要になることがあります。`DBIO[T]`は`DBIOAction[T,S,E]`の別名で、ストリーミングとエフェクトを符号化することを思い出してください。挿入や選択などのエフェクトを混在させる場合、結果のアクションに適用する型パラメータを明示的に指定する必要がある場合があります：


``` scala
query.flatMap[Int, NoStream, Effect.All] { result => ... }
```


...しかし、多くの場合、コンパイラがこれらを解決してくれるでしょう。


<div class="callout callout-info">

**Do it in the database if you can**

アクションを組み合わせてクエリーを連続させることは、Slickの強力な機能です。しかし、複数のクエリを1つのデータベースクエリに減らすことができるかもしれません。それができるのであれば、その方がよいでしょう。

例として、「insert if not exists」を次のように実装することができます：

```scala mdoc:silent:silent
// Not the best way:
def insertIfNotExists(m: Message): DBIO[Int] = {
  val alreadyExists =
    messages.filter(_.content === m.content).result.headOption
  alreadyExists.flatMap {
    case Some(m) => DBIO.successful(0)
    case None    => messages += m
  }
}
```

しかし先ほどの ["More Control over Inserts"](#moreControlOverInserts) で見たように、SQL文1つで同じ効果を得ることができます。 

1つのクエリが、一連のクエリよりも優れた性能を発揮することがよくあります（ただし、常にそうとは限りません）。お客様のご判断にお任せします。

</div>



### `DBIO.sequence`

`DBIO.seq`と名前が似ていますが、`DBIO.sequence`は異なる目的を持っています。これは`DBIO`のシーケンスを受け取り、シーケンスの`DBIO`を返すものです。ちょっと難しい話ですが、例で説明します。

例えば、データベース内のすべてのメッセージ（行）のテキストを反転させたいとします。まず、こんなところから始めます：


```scala mdoc:silent
def reverse(msg: Message): DBIO[Int] =
  messages.filter(_.id === msg.id).
  map(_.content).
  update(msg.content.reverse)
```

これは、1つのメッセージに対する更新アクションを返す、わかりやすいメソッドです。すべてのメッセージに適用することができます...

```scala mdoc:silent
// Don't do this
val manyUpdates: DBIO[Seq[DBIO[Int]]] =
  messages.result.
  map(msgs => msgs.map(reverse))
```


...これで、アクションを返すアクションが出来ました！クレイジーな型シグネチャに注目してください。

結合のようなことをしようとすると、このような厄介な状況に陥ることがあります。このような獣をどう走らせるかがパズルです。

そこで、`DBIO.sequence` が役に立ちます。`msgs.map(reverse)` で多数のアクションを生成するのではなく、`DBIO.sequence` を使って単一のアクションを返します：


```scala mdoc:silent
val updates: DBIO[Seq[Int]] =
  messages.result.
  flatMap(msgs => DBIO.sequence(msgs.map(reverse)))
```

違いとしては
- `Seq[DBIO]`を`DBIO.sequence`でラップして、`DBIO[Seq[Int]]`を1つにしています。
- `flatMap`を使ってシーケンスと元のクエリを結合しています。

最終的には、他のアクションと同じように走らせることができる正気のタイプです。

もちろん、この1アクションは多くのSQL文に変化します：

```sql
select "sender", "content", "id" from "message"
update "message" set "content" = ? where "message"."id" = 1
update "message" set "content" = ? where "message"."id" = 2
update "message" set "content" = ? where "message"."id" = 3
update "message" set "content" = ? where "message"."id" = 4
```

### `DBIO.fold`

多くのScalaコレクションが、値を結合する方法として`fold`をサポートしていることを思い出してください：

```scala mdoc
List(3,5,7).fold(1) { (a,b) => a * b }

1 * 3 * 5 * 7
```

Slickでも同じようなことができます。一連のアクションを実行し、その結果をある値に還元する必要がある場合、`fold`を使用します。

例として、実行するレポートがいくつもあるとします。これらのレポートをすべて1つの数値にまとめたいと思います。

```scala mdoc:silent
// Pretend these two reports are complicated queries
// that return Important Business Metrics:
val report1: DBIO[Int] = DBIO.successful(41)
val report2: DBIO[Int] = DBIO.successful(1)

val reports: List[DBIO[Int]] =
  report1 :: report2 :: Nil
```

関数を使って `reports` を `fold` (畳み込み)することができます。

しかし、スタートポジションを考えることも必要です：

```scala mdoc:silent
val default: Int = 0
```

最後に、レポートをまとめるためのアクションを作成します：

```scala mdoc
val summary: DBIO[Int] =
  DBIO.fold(reports, default) {
    (total, report) => total + report
}

exec(summary)
// res8: Int = 42
```


`DBIO.fold`は、あなたが用意する関数でアクションを組み合わせることが結果を得ることができる方法です。他のコンビネータと同様に、アクションそのものを実行するまで、関数は実行されません。この場合、すべてのレポートが実行され、その値の合計が報告されます。


### `zip`

`DBIO.seq`がアクションを組み合わせ、その結果を無視することを見てきました。また、`andThen`はアクションを組み合わせ、1つの結果を保持することも見てきました。もし両方の結果を残したいのであれば、`zip`が使えます。


```scala mdoc
val zip: DBIO[(Int, Seq[Message])] =
  messages.size.result zip messages.filter(_.sender === "HAL").result

// Make sure we have some messages from HAL:
exec(messages ++= freshTestData)


exec(zip)

```

このアクションは、両方のクエリの結果を表すタプル（メッセージの総数のカウントと、HALからのメッセージ）を返します。


### `andFinally` and `cleanUp`

`cleanUp`と`andFinally`の2つのメソッドは、Scalaの`catch`と`finally`のように少し動作します。

`cleanUp`はアクションが完了した後に実行され、`Option[Throwable]`としてあらゆるエラー情報にアクセスすることができます。


```scala mdoc:silent
// An action to record problems we encounter:
def log(err: Throwable): DBIO[Int] =
  messages += Message("SYSTEM", err.getMessage)

// Pretend this is important work which might fail:
val work = DBIO.failed(new RuntimeException("Boom!"))

val action: DBIO[Int] = work.cleanUp {
  case Some(err) => log(err)
  case None      => DBIO.successful(0)
}
```



```scala mdoc:crash
exec(action)
```

...しかし、`cleanUp`は私たちに副次的な効果をもたらしてくれます。

```scala mdoc
exec(messages.filter(_.sender === "SYSTEM").result)
```

```scala mdoc:invisible
{
  val c = exec(messages.filter(_.sender === "SYSTEM").length.result)
  assert(c == 1, s"Expected one result not $c")
}
```

`cleanUp`と`andFinally`は、成功・失敗にかかわらず、アクションの後に実行されます。`cleanUp`は、以前に失敗したアクションに対応して実行されます。`andFinally`は、成功・失敗にかかわらず常に実行され、`cleanUp`が見る`Option［Throwable］`へのアクセスはありません。

### `asTry`

アクションに対してasTryを呼び出すと、アクションの型が`DBIO[T]`から`DBIO[Try[T]]`に変更されます。つまり、例外ではなく、Scalaの`Success[T]`と`Failure`で動作できるようになります。

仮に、例外を投げる可能性のあるアクションがあったとします。


```scala mdoc:silent
val tryAction = DBIO.failed(new RuntimeException("Boom!"))
```

アクションを`asTry`と組み合わせることで、これを`Try`の内側に配置することができます：

```scala mdoc
exec(tryAction.asTry)
```

成功したアクションは`Success[T]`と評価されます。

```scala mdoc
exec(messages.size.result.asTry)
```


## Logging Queries and Results

アクションを組み合わせて、実行中のクエリを確認できるのは便利ですね。

これまで、クエリの`insertStatement`などのメソッドや、アクションの`statements`メソッドを使って、クエリのSQLを取得する方法について見てきました。

これらはSlickの実験に便利ですが、Slickが実行したときにすべてのクエリを確認したいこともあります。ロギングを設定することでそれを実現することができます。

Slickは[SLF4J][link-slf4j]というロギングインターフェイスを使用しています。これを設定することで、実行中のクエリに関する情報を取得することができます。

練習問題の`build.sbt`ファイルは、[Logback][link-logback]というSLF4J互換のロギングバックエンドを使用しており、*src/main/resources/logback.xml* というファイルで設定されています。
このファイルでは、ロギングをdebugレベルにすることで、statementロギングを有効にすることができます。

``` xml
<logger name="slick.jdbc.JdbcBackend.statement" level="DEBUG"/>
```

これにより、Slickはスキーマの変更を含むすべてのクエリを記録するようになります。

```
DEBUG slick.jdbc.JdbcBackend.statement - Preparing statement:
  delete from "message" where "message"."sender" = 'HAL'
```

下の表に示すように、各種ロガーのレベルを変更することができます。


-----------------------------------------------------------------------------------------------------------------------------
Logger                                                             Will log...
-----------------------------------------------------------------  ----------------------------------------------------------
`slick.jdbc.JdbcBackend.statement`                                 SQL sent to the database.

`slick.jdbc.JdbcBackend.parameter`                                 Parameters passed to a query.

`slick.jdbc.StatementInvoker.result`                               The first few results of each query.

`slick.session`                                                    Session events such as opening/closing connections.

`slick`                                                            Everything!
-----------------------------------------------------------------  ----------------------------------------------------------

: Slick loggers and their effects.

特に`StatementInvoker.result`のロガーはかなりキュートです。以下は、selectクエリを実行したときの例です。

```
result - /--------+----------------------+----\
result - | sender | content              | id |
result - +--------+----------------------+----+
result - | HAL    | Affirmative, Dave... | 2  |
result - | HAL    | I'm sorry, Dave. ... | 4  |
result - \--------+----------------------+----/
```
`parameter`と`statement`の組み合わせで、`?`プレースホルダに結合された値を表示することができます。例えば、行を追加するときに、挿入される値を見ることができます。


```
statement - Preparing statement: insert into "message" 
   ("sender","content")  values (?,?)
parameter - /--------+---------------------------\
parameter - | 1      | 2                         |
parameter - | String | String                    |
parameter - |--------+---------------------------|
parameter - | Dave   | Hello, HAL. Do you rea... |
parameter - | HAL    | I'm sorry, Dave. I'm a... |
parameter - \--------+---------------------------/
```



## Transactions {#Transactions}

これまでのところ、私たちがデータベースに加えたそれぞれの変更は、他のものとは独立して実行されています。つまり、挿入、更新、削除の各クエリは、他のクエリとは独立して成功または失敗することができます。

私たちはしばしば、トランザクションで一連の変更を結びつけて、それらがすべて成功するか、すべて失敗するようにしたい時があります。。Slickでは`transactionally`メソッドを使ってこれを行うことができます。

例として、映画の脚本を書き換えてみましょう。このとき、脚本がすべて完全に変化するか、何も変化しないかを確認したい。そのためには、古い脚本テキストを見つけ、新しいテキストに置き換える必要があります。


```scala mdoc
def updateContent(old: String) =
  messages.filter(_.content === old).map(_.content)

exec {
  (updateContent("Affirmative, Dave. I read you.").update("Wanna come in?") andThen
   updateContent("Open the pod bay doors, HAL.").update("Pretty please!") andThen
   updateContent("I'm sorry, Dave. I'm afraid I can't do that.").update("Opening now.") ).transactionally
}

exec(messages.result).foreach(println)
```

`transactionally`ブロックで行った変更は、ブロックが完了するまでの一時的なもので、完了後にコミットされ永続化されます。

手動で強制的にロールバックするには、適切な例外を指定して`DBIO.failed`を呼び出す必要があります。

```scala mdoc
val willRollback = (
  (messages += Message("HAL",  "Daisy, Daisy..."))                   >>
  (messages += Message("Dave", "Please, anything but your singing")) >>
  DBIO.failed(new Exception("agggh my ears"))                        >>
  (messages += Message("HAL", "Give me your answer do"))
  ).transactionally

exec(willRollback.asTry)
```

`willRollback`を実行した結果、データベースが変更されることはないでしょう。トランザクションブロックの内部では、`DBIO.failed`が呼ばれるまで挿入が行われることになります。

もし、組み合わせたアクションを包んでいる`.transactionally`を削除したら、組み合わせたアクションが失敗しても、最初の2つの挿入は成功するでしょう。


## Take Home Points

挿入、選択、削除、その他の形式のDatabase Actionは、`flatMap`やその他のコンビネータを使用して組み合わせることができます。これは、アクションを連続させたり、アクションを他のアクションの結果に依存させたりする強力な方法です。

アクションを組み合わせることで、結果が出るまで待つことや、自分で`Future`のシーケンスを扱う必要がありません。

ロギングシステムを設定することで、実行されたSQL文とデータベースから返された結果を監視できることを確認しました。

最後に、組み合わせたアクションをトランザクションの中で実行することもできることを確認しました。


## Exercises

### And Then what?

第1章では、スキーマの作成とデータベースへのデータ投入を別々のアクションとして行いました。新しく見つけた知識を使って、それらを組み合わせてみましょう。

この演習では、空のデータベースで開始することを想定しています。すでに REPL にいてデータベースが存在する場合は、最初にテーブルを削除する必要があります。


```scala mdoc
val drop:     DBIO[Unit]        = messages.schema.drop
val create:   DBIO[Unit]        = messages.schema.create
val populate: DBIO[Option[Int]] = messages ++= freshTestData

exec(drop)
```

<div class="solution">

私たちが用意した値を使えば、ワンアクションで新しいデータベースを作成することができます。


```scala mdoc:invisible
exec(drop.asTry >> create)
```
```scala mdoc
exec(drop andThen create andThen populate)
```

もし、返ってくる値が必要なければ、`DBIO.seq`を使うこともできます：

```scala mdoc
val allInOne = DBIO.seq(drop,create,populate)
val result = exec(allInOne)
```
</div>

### First!

メッセージを挿入するメソッドを作成してください。
ただし、それがデータベースの最初のメッセージである場合、その前に「First！」というメッセージを自動的に挿入するようにしてください。

メソッドシグネチャはこうしてください。

```scala
def prefixFirst(m: Message): DBIO[Int] = ???
```

`flatMap`アクションコンビネータの知識を使いましょう。

<div class="solution">

この問題には、2つの要素があります。

1. flatMapでカウントの結果を利用できること。
2. andThenを介して2つのインサートを結合すること。 


```scala mdoc
import scala.concurrent.ExecutionContext.Implicits.global

def prefixFirst(m: Message): DBIO[Int] =
  messages.size.result.flatMap {
    case 0 =>
      (messages += Message(m.sender, "First!")) andThen (messages += m)
    case n =>
      messages += m
    }

// Throw away all the messages:
exec(messages.delete)

// Try out the method:
exec {
  prefixFirst(Message("Me", "Hello?"))
}

// What's in the database?
exec(messages.result).foreach(println)
```
</div>

### There Can be Only One

アクションが1つの結果しか返さないことを保証するメソッド、`onlyOne`を実装してください。アクションが1つ以外の結果を返す場合、そのメソッドは例外で失敗するはずです。

以下は、メソッドのシグネチャーと2つのテストケースです。


```scala
def onlyOne[T](ms: DBIO[Seq[T]]): DBIO[T] = ???
```

`onlyOne`は引数としてアクションを受け取り、そのアクションが一連の結果を返す可能性があることがわかります。このメソッドからのリターンは、単一の値を返すアクションです。

例のデータでは、"Sorry "という単語を含むメッセージは1つだけなので、onlyOneはその行を返すと予想されます。



```scala mdoc
val happy = messages.filter(_.content like "%sorry%").result
```
```scala
// We expect... 
// exec(onlyOne(happy))
// ...to return a message.
```


しかし、"I "という単語を含むメッセージが2つ存在します。この場合、`onlyOne`は失敗するはずです。

```scala mdoc
val boom  = messages.filter(_.content like "%I%").result
```
```scala
// If we run this...
// exec(onlyOne(boom))
// we want a failure, such as:
// java.lang.RuntimeException: Expected 1 result, not 2
```

ヒント

- `onlyOne`のシグネチャは、`Seq[T]`を生成するアクションを受け取り、`T`を生成するアクションを返します。アクションコンビネータが必要です。 

- メソッドが失敗する可能性があるということは、どこかで`DBIO.successful`と`DBIO.failed`を使いたいわけです。



<div class="solution">

私たちのソリューションの基本は、与えられたアクションを、私たちが望む型を持つ新しいアクションにflatMapすることです。

```scala mdoc:silent
def onlyOne[T](action: DBIO[Seq[T]]): DBIO[T] = action.flatMap { ms =>
  ms match {
    case m +: Nil => DBIO.successful(m)
    case ys       => DBIO.failed(
        new RuntimeException(s"Expected 1 result, not ${ys.length}")
      )
  }
}
```

`+:`はScalaの標準機能であるSeq（Listの`::`に相当）の「cons」です。

`flatMap`は`ms`というアクションから結果を受け取り、それが単一のメッセージである場合はそれを返します。それ以外のメッセージの場合は、情報提供のメッセージとともに失敗します。

```scala mdoc
exec(populate)
```

```scala mdoc:crash
exec(onlyOne(boom))
```

```scala mdoc
exec(onlyOne(happy))
```
</div>

### Let's be Reasonable

ある馬鹿が我々のコードに例外を投げて、それを推論する能力を破壊しています。例外ではなく型を使って失敗の可能性をエンコードする `onlyOne` をラップした `exactlyOne` を実装します。

その後、テストケースを再実行します。


<div class="solution">

これを実装する方法はいくつかあります。最もシンプルなのは、`asTry`を使うことでしょう。

```scala mdoc
import scala.util.Try
def exactlyOne[T](action: DBIO[Seq[T]]): DBIO[Try[T]] = onlyOne(action).asTry

exec(exactlyOne(happy))
```

```scala mdoc
exec(exactlyOne(boom))
```
</div>


### Filtering

There is a `DBIO` `filter` method, but it produces a runtime exception if the filter predicate is false.
It's like `Future`'s `filter` method in that respect. We've not found a situation where we need it.

However, we can create our own kind of filter.
It can take some alternative action when the filter predicate fails.

The signature could be:

```scala
def myFilter[T](action: DBIO[T])(p: T => Boolean)(alternative: => T) = ???
```

If you're not comfortable with the `[T]` type parameter,
or the by name parameter on `alternative`,
just use `Int` instead:

```scala
def myFilter(action: DBIO[Int])(p: Int => Boolean)(alternative: Int) = ???
```

Go ahead and implement `myFilter`.

We have an example usage from the ship's marketing department.
They are happy to report the number of chat messages, but only if that number is at least 100:

```scala
myFilter(messages.size.result)( _ > 100)(100)
```

<div class="solution">
This is a fairly straightforward example of using `map`:

```scala mdoc:silent
def myFilter[T](action: DBIO[T])(p: T => Boolean)(alternative: => T) =
  action.map {
    case t if p(t) => t
    case _         => alternative
  }
```
</div>

### Unfolding

This is a challenging exercise.

We saw that `fold` can take a number of actions and reduce them using a function you supply.
Now imagine the opposite: unfolding an initial value into a sequence of values via a function.
In this exercise we want you to write an `unfold` method that will do just that.

Why would you need to do something like this?
One example would be when you have a tree structure represented in a database and need to search it.
You can follow a link between rows, possibly recording what you find as you follow those links.

As an example, let's pretend the crew's ship is a set of rooms, one connected to just one other:

```scala mdoc
case class Room(name: String, connectsTo: String)

class FloorPlan(tag: Tag) extends Table[Room](tag, "floorplan") {
  def name       = column[String]("name")
  def connectsTo = column[String]("next")
  def * = (name, connectsTo).mapTo[Room]
}

lazy val floorplan = TableQuery[FloorPlan]

exec {
  (floorplan.schema.create) >>
  (floorplan += Room("Outside",     "Podbay Door")) >>
  (floorplan += Room("Podbay Door", "Podbay"))      >>
  (floorplan += Room("Podbay",      "Galley"))      >>
  (floorplan += Room("Galley",      "Computer"))    >>
  (floorplan += Room("Computer",    "Engine Room"))
}
```

For any given room it's easy to find the next room. For example:

~~~ sql
SELECT
  "connectsTo"
FROM
  "foorplan"
WHERE
  "name" = 'Podbay'

-- Returns 'Galley'
~~~

Write a method `unfold` that will take any room name as a starting point,
and a query to find the next room,
and will follow all the connections until there are no more connecting rooms.

The signature of `unfold` _could_ be:

```scala
def unfold(
  z: String,
  f: String => DBIO[Option[String]]
): DBIO[Seq[String]] = ???
```

...where `z` is the starting ("zero") room, and `f` will lookup the connecting room (an action for the query to find the next room).

If `unfold` is given `"Podbay"` as a starting point it should return an action which, when run, will produce: `Seq("Podbay", "Galley", "Computer", "Engine Room")`.

You'll want to accumulate results of the rooms you visit.
One way to do that would be to use a different signature:

```scala
def unfold(
  z: String,
  f: String => DBIO[Option[String]],
  acc: Seq[String] = Seq.empty
): DBIO[Seq[String]] = ???
```

<div class="solution">

The trick here is to recognize that:

1. this is a recursive problem, so we need to define a stopping condition;

2. we need `flatMap` to sequence queries ; and

3. we need to accumulate results from each step.

In code...

```scala mdoc:silent
def unfold(
  z: String,
  f: String => DBIO[Option[String]],
  acc: Seq[String] = Seq.empty
): DBIO[Seq[String]] =
  f(z).flatMap {
    case None    => DBIO.successful(acc :+ z)
    case Some(r) => unfold(r, f, acc :+ z)
  }
```

The basic idea is to call our action (`f`) on the first room name (`z`).
If there's no result from the query, we're done.
Otherwise we add the room to the list of rooms, and recurse starting from the room we just found.

Here's how we'd use it:

```scala mdoc
def nextRoom(roomName: String): DBIO[Option[String]] =
  floorplan.filter(_.name === roomName).map(_.connectsTo).result.headOption

val path: DBIO[Seq[String]] = unfold("Podbay", nextRoom)

exec(path)
```

```scala mdoc:invisible
{
  val r = exec(path)
  assert(r == List("Podbay", "Galley", "Computer", "Engine Room"), s"Expected 4 specific rooms, but got $r")
}
```
</div>
