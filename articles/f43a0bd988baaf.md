---
title: "Clojure CLIからヘルプを確認する"
emoji: "🕵️‍♂️"
type: "tech"
topics: ["clojure", "cli", "help"]
published: true
---

これは[Clojureアドベントカレンダー](https://qiita.com/advent-calendar/2021/clojure) 8日目の記事です。

ここ最近、Clojure CLIには`-X:deps`や`-Ttools`などのオプションを通じて使える機能がいろいろ追加されています。しかし、これらの機能のヘルプは`clj --help`には表示されないので、使い方やそもそもどういう機能があるのかがよく分からなかったりします。

ここでは、そういった場合にコマンドラインからサクッと機能や使い方を調べられる `help/dir`, `help/doc`の機能について紹介します。

## `help/dir`と`help/doc`

1.10.3.899 から Clojure CLI に`-X:deps`オプションから呼び出せる`help/dir`および`help/doc`という機能がつきました。これらは、REPLから使える`dir`マクロ(`clojure.repl/dir`)および`doc`マクロ(`clojure.repl/doc`)に相当する機能をコマンドラインから使えるようにしたものです。
つまり、それぞれ以下のような機能を提供します：

- `help/dir`: 名前空間名を指定すると、その名前空間で定義されたパブリックな関数の一覧を表示する
- `help/doc`: 名前空間名または関数名前を指定すると、そのドキュメント文字列を表示する

## 基本的な使い方

`help/dir`は以下のようにして使えます：

```sh
$ clj -X:deps help/dir :ns clojure.set
difference
index
intersection
join
map-invert
project
rename
rename-keys
select
subset?
superset?
union
$
```

`:ns <namespace>`で対象となる名前空間を指定します。

`help/doc`で関数のドキュメント文字列を表示するには以下のようにします：

```shell
$ clj -X:deps help/doc :fn clojure.core/conj
-------------------------
clojure.core/conj
([] [coll] [coll x] [coll x & xs])
  conj[oin]. Returns a new collection with the xs
    'added'. (conj nil item) returns (item).
    (conj coll) returns coll. (conj) returns [].
    The 'addition' may happen at different 'places' depending
    on the concrete type.
$
```

`:fn <name>`で対象となる関数の名前を指定します。ここで、関数名は名前空間修飾されている必要があります。

`help/doc`で名前空間のドキュメント文字列を表示するには、`:fn <name>`の代わりに`:ns <namespace>`を指定します：

```shell
$ clj -X:deps help/doc :ns clojure.set
Set operations such as union/intersection.

-------------------------
clojure.set/difference
([s1] [s1 s2] [s1 s2 & sets])
  Return a set that is the first set without elements of the remaining sets
-------------------------
clojure.set/index
([xrel ks])
  Returns a map of the distinct values of ks in the xrel mapped to a
  set of the maps in xrel with the corresponding values of ks.
-------------------------
 :
 :
-------------------------
clojure.set/union
([] [s1] [s1 s2] [s1 s2 & sets])
  Return a set that is the union of the input sets
$
```

`help/doc`に名前空間を指定した場合には、名前空間自身のドキュメント文字列だけでなく、その名前空間で定義されたパブリックな関数のドキュメント文字列も表示されます。なので、関数が大量に定義されている名前空間(たとえば`clojure.core`)を表示すると大変なことになるので注意しましょう。

## `help/dir`や`help/doc`のヘルプを確認する

`help/dir`や`help/doc`の使い方が分からなくなった場合は、これら自身のヘルプを確認することもできます：

```shell
$ clj -X:deps help/dir :ns help
dir
doc
$ clj -X:deps help/doc :fn help/doc
-------------------------
clojure.tools.cli.help/doc
([{:keys [ns fn], :as args}])
  Print doc for the specified namespace or function. If neither is specified, print docs
  for :ns-default.

  Options:
    :ns Print docs for namespace
    :fn Print docs for function
$
```

`help`はClojure CLIが内部的に定義している名前空間のエイリアスで、名前空間名の代わりとしてそのまま指定することができます。

## `-X:deps`の他の機能のヘルプを確認する

`-X:deps`から呼び出せる機能には`help/dir`や`help/doc`以外にもいろいろあります。それらのヘルプを`help/dir`や`help/doc`から確認するには、以下のようにします：

```javascript
$ clj -X:deps help/dir
basis
find-versions
git-resolve-tags
list
mvn-install
mvn-pom
prep
tree
$ clj -X:deps help/doc :fn find-versions
-------------------------
clojure.tools.cli.api/find-versions
([{:keys [lib tool], :as args}])
  Find available tool versions given either a lib (with :lib) or
  existing installed tool (with :tool). If lib, check all registered
  procurers and print one coordinate per line when found.
$
```

`help/dir`や`help/doc`は名前空間が省略された場合には`deps.edn`で指定された`:ns-default`が指定されたものと見做します。`-X:deps`で指定している`deps`というエイリアスは、`clojure.tools.cli.api`を`:ns-default`として指定されているため、`-X:deps help/dir`等で名前空間を省略すれば`clojure.tools.cli.api`からヘルプを引くことになります。

## CLIツールのヘルプを確認する

CLIツールのヘルプも`help/dir`, `help/doc`から確認できます。
CLIツールとは、`-T`オプションで起動できるプログラムで、`deps.edn`に定義することでプロジェクトレベルでインストールできる他、環境にグローバルにインストールすることもできます。

### プロジェクトレベルのCLIツールの場合

まず、プロジェクトレベルのCLIツールの場合について、見ていきましょう。

以下のように`deps.edn`にビルドの設定がしてあったとします：

```clojure
{...
 :aliases
 {:build
  {:deps
   {io.github.clojure/tools.build
    {:git/tag "v0.6.8" :git/sha "d79ae84"}}
   :ns-default build}
 ...}
```

ビルドスクリプトは以下のようになっている場合、

```clojure
(ns build
  (:require [clojure.tools.build.api :as b]))

(def lib 'my/lib1)
(def version (format "1.2.%s" (b/git-count-revs nil)))
(def class-dir "target/classes")
(def basis (b/create-basis {:project "deps.edn"}))
(def uber-file (format "target/%s-%s-standalone.jar" (name lib) version))

(defn clean [{:keys [path] :or {path "target"}}]
  (b/delete {:path path}))

(defn uberjar [_]
  (clean nil)
  (b/copy-dir {:src-dirs ["src" "resources"]
               :target-dir class-dir})
  (b/compile-clj {:basis basis
                  :src-dirs ["src"]
                  :class-dir class-dir})
  (b/uber {:class-dir class-dir
           :uber-file uber-file
           :basis basis
           :main my.lib.main}))
```

以下のようにすることで、ビルドタスクの一覧が得られます：

```sh
$ clj -A:deps -T:build help/dir
clean
uberjar
$
```

また、以下のようにして、各ビルドタスクの引数を確認することもできます：

```sh
$ clj -A:deps -T:build help/doc :fn clean
-------------------------
build/clean
([{:keys [path], :or {path "target"}}])
$
```

前節で説明したように、これらの場合でも名前空間名を省略したことにより`:ns-default`で指定した名前空間(この場合は`build`)が指定されたものと見做されています。

このあたりのオプションの指定はややトリッキーなので忘れがちですが、[公式リファレンス](https://clojure.org/reference/deps_and_cli#_help_functions)にケースごとのオプションの指定方法がリストアップされているので、分からなくなったらそちらを見に行くのがよいでしょう。

### 環境にグローバルにインストールされたCLIツールの場合

環境にグローバルにインストールにされたCLIについても同様にしてヘルプを確認できます。

たとえば、次のようにしてCLIツール [`deps-new`](https://github.com/seancorfield/deps-new) をインストールした場合、

```sh
$ clj -Ttools install io.github.seancorfield/deps-new '{:git/tag "v0.4.3"}' :as new
```

以下のようにして、そのインストールしたCLIツールのヘルプを確認できます：

```sh
$ clj -A:deps -Tnew help/dir
app
create
lib
pom
scratch
template
$ clj -A:deps -Tnew help/doc :fn app
-------------------------
org.corfield.new/app
([opts])
  Exec function to create an application project.
  `:name` -- a symbol (or string) identifying the project name,
  `:target-dir` -- optional string identifying the directory to
      create the new project in,
  `:overwrite` -- whether to overwrite an existing directory or,
      for `:delete`, to delete it first; if `:overwrite` is `nil`
      or `false`, an existing directory will not be overwritten.
$
```

上でCLIツールのインストールに使った`clj -Ttools`についてもまったく同様にして調べられます：

```sh
$ clj -A:deps -Ttools help/dir
install
list
remove
show
$ clj -A:deps -Ttools help/doc :fn install
clojure.tools.tools.api/install
([{:keys [as], :as args}])
  Install a tool under a local tool name for later use. On install, the tool is procured, and
  persisted with the tool name for later use.

  Options:
    lib-name (required) - value is coord map (git coords may omit sha)
    :as (required) - tool name

  Example:
    clj -Ttools install io.github.clojure/tools.deps.graph '{:git/tag "v1.0.63"}' :as deps-graph

  Also see:
    clj -X:deps find-versions :lib <lib>
$
```

## おわりに

今回は`help/dir`, `help/doc`の使い方を紹介しました。
`-X`プログラムにしても、CLIツールにしても、使い方を調べるのにドキュメントを見に行くのは面倒なので、CLIから手軽にヘルプを確認できるようにしておくと便利です👍