# Using Different Database Products {#altdbs}
本書では、導入で述べたように、例題には主にH2が使用されています。しかし、Slickは他にもPostgreSQL、MySQL、Derby、SQLite、Oracle、およびMicrosoft Accessをサポートしています。

かつては、Oracle、SQL Server、またはDB2と一緒にSlickを本番環境で使用するにはLightbendから商用ライセンスが必要でした。
しかし、この制約は2016年初頭に取り除かれました[^slick-blog-open]。
ただし、自由でオープンなプロファイルを構築するための取り組みもあり、それによりFreeSlickプロジェクトが生まれました。
これらのプロファイルは引き続き利用可能で、詳細については[FreeSlickのGitHubページ](https://github.com/smootoo/freeslick)で確認できます。

[^slick-blog-open]: [https://scala-slick.org/news/2016/02/01/slick-extensions-licensing-change.html](https://scala-slick.org/news/2016/02/01/slick-extensions-licensing-change.html)。

## Changes

もしご自身の環境で本書の演習に別のデータベースを使用したい場合は、以下の詳細な手順に従う必要があります。

要約すると、以下の点を確認する必要があります：

- データベースがインストールされていること（この本の範囲を超える詳細は含まれていません）。
- 正しい名前のデータベースが利用可能であること。
- `build.sbt` ファイルに正しい依存関係が設定されていること。
- コード内で正しい JDBC ドライバが参照されていること。
- 正しい Slick プロファイルが使用されていること。

各章はそれぞれ独自のデータベースを使用していますので、これらの手順は各章ごとに適用する必要があります。

以下に2つのデータベースを用いた詳細な手順を示します。

## PostgreSQL

### Create a Database

以下の手順に従って、`essential` ユーザーで `chapter-01` という名前のデータベースを作成します。このデータベースはすべての例に使用され、以下のコマンドで作成できます。


~~~ sql
CREATE DATABASE "chapter-01" WITH ENCODING 'UTF8';
CREATE USER "essential" WITH PASSWORD 'trustno1';
GRANT ALL ON DATABASE "chapter-01" TO essential;
~~~

データベースが作成され、アクセス可能であることを確認します。

~~~ bash
$ psql -d chapter-01 essential
~~~

### Update `build.sbt` Dependencies

次の行を

~~~ scala
"com.h2database" % "h2" % "1.4.185"
~~~

次の行で置き換えます

~~~ scala
"org.postgresql" % "postgresql" % "9.3-1100-jdbc41"
~~~

すでに SBT で作業している場合は、変更されたビルドファイルを読み込むために `reload` を入力してください。IDE を使用している場合は、IDE プロジェクトファイルを再生成することを忘れないでください。


### Update JDBC References

`application.conf` のパラメータを以下のように置き換えます：

~~~ json
chapter01 = {
  connectionPool      = disabled
  url                 = jdbc:postgresql:chapter-01
  driver              = org.postgresql.Driver
  keepAliveConnection = true
  users               = essential
  password            = trustno1
}
~~~

### Update Slick Profile

インポートを以下のように変更します：

```scala
slick.jdbc.H2Profile.api._
```

to

```scala
slick.jdbc.PostgresProfile.api._
```


## MySQL

### Create a Database

以下の手順に従って、`essential` ユーザーで `chapter-01` という名前のデータベースを作成します。このデータベースはすべての例に使用され、以下のコマンドで作成できます。


~~~ sql
CREATE USER 'essential'@'localhost' IDENTIFIED BY 'trustno1';
CREATE DATABASE `chapter-01` CHARACTER SET utf8 COLLATE utf8_bin;
GRANT ALL ON `chapter-01`.* TO 'essential'@'localhost';
FLUSH PRIVILEGES;
~~~

データベースが作成され、アクセス可能であることを確認します。

~~~ bash
$ mysql -u essential chapter-01 -p
~~~

### Update `build.sbt` Dependencies

次の行を

~~~ scala
"com.h2database" % "h2" % "1.4.185"
~~~

次の行で置き換えます

~~~ scala
"mysql" % "mysql-connector-java" % "5.1.34"
~~~

すでに SBT で作業している場合は、変更されたビルドファイルを読み込むために `reload` を入力してください。IDE を使用している場合は、IDE プロジェクトファイルを再生成することを忘れないでください。
.

### Update JDBC References

`Database.forURL` のパラメータを以下のように置き換えます：

~~~ json
chapter01 = {
  connectionPool      = disabled
  url                 = jdbc:mysql://localhost:3306/chapter-01
                                      &useUnicode=true
                                      &amp;characterEncoding=UTF-8
                                      &amp;autoReconnect=true
  driver              = com.mysql.jdbc.Driver
  keepAliveConnection = true
  users               = essential
  password            = trustno1
}
~~~

`connectionPool` 行を見やすくフォーマットしましたが、実際にはすべての `&` パラメータは同じ行になります。



### Update Slick DriverProfile

インポートを以下のように変更します：

```scala
slick.jdbc.H2Profile.api._
```

to

```scala
slick.jdbc.MySQLProfile.api._
```
