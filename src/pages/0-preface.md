# Preface {-}

## What is Slick? {-}

[Slick][link-slick]は、関係データベースと連携するためのScalaライブラリです。
つまり、スキーマのモデリング、クエリの実行、データの挿入、データの更新などが可能です。

Slickを使用すると、Scalaでクエリを記述することができ、型付きのデータベースアクセスが提供されます。
クエリのスタイルは、データベースと通常のScalaコレクションを操作するのと似た方法で行います。

初めてSlickを使用する開発者は、Slickを最大限に活用するために支援が必要な場合が多いことを確認しました。
例えば、以下のようないくつかのキーコンセプトを把握する必要があります：

- _クエリ_: `map`、`flatMap`、`filter`などのコンビネータを使用して組み立てる方法；
- _アクション_: データベースに対して実行できる操作で、それ自体も組み立てることができるもの；および
- _フューチャー_: アクションの結果であり、一連のコンビネータもサポートしています。

この「Essential Slick」は、Slickの使用を始めたい人々のためのガイドとして作成されました。
この資料は、Scalaの初心者から中級者を対象としています。以下のものが必要です：

* Scalaの作業知識
  （[Essential Scala][link-essential-scala]などを推奨します）；
* 関係データベースの経験
  （行、列、結合、インデックス、SQLなどの概念に精通していること）；
* JDK 8以降がインストールされていること、プログラマ向けのテキストエディタまたはIDEが利用可能であること；および
* [sbt][link-sbt]ビルドツールが利用できること。

提示された資料は、Slickのバージョン3.3に焦点を当てています。例では、関係データベースとして[H2][link-h2-home]が使用されています。


## How to Contact Us {-}

このテキストに関するフィードバックは、以下の方法で提供できます：

* このテキストのソースリポジトリである[issues][link-book-issues]と[pull requests][link-book-pr]を利用する方法
* [Gitterチャンネル][link-underscore-gitter]への投稿
* "Essential Slick

"という件名で[hello@underscore.io][link-email-underscore]宛てのメールでご連絡いただく方法

## Getting help using Slick {-}

Slickの使用に関する質問がある場合は、[SlickのGitterチャンネル][link-slick-gitter]で質問するか、Stackoverflowの["slick"タグ][link-slick-so]を使用して質問してください。