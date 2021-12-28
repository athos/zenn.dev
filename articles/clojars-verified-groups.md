---
title: "Clojarsの承認済みグループのトレンドを調べる"
emoji: "📊"
type: "tech"
topics: ["clojure","clojars"]
published: true
---

これは[Clojureアドベントカレンダー](https://qiita.com/advent-calendar/2021/clojure) 15日目の記事です。

## はじめに

Clojureライブラリの配布リポジトリである[Clojars](https://clojars.org/)では、今年4月からライブラリを登録するのに「承認済みグループ名」が[必須になりました](https://github.com/clojars/clojars-web/wiki/Verified-Group-Names)。これによって、これまでライブラリに自由につけられたグループ名が、実在するドメイン名をベースにした事前に承認されたグループ名でなければいけなくなりました。

ところで、4月からこれまでの間にリリースされたライブラリの中で、どれぐらいの割合がすでに承認済みグループの下でリリースされているのでしょうか？また、承認済みグループ名としてはどういったものが多く使われる傾向にあるのでしょうか？

この記事では、これらの答えについて調べてみたいと思います。
ここでは、以下の3ステップにより承認済みグループのトレンドについて調べます：

- 期間中にリリースされたライブラリの情報を収集する
- ライブラリが属するグループを抽出し、承認済みグループかどうか調べる
- 収集したライブラリおよびグループの情報を集計する

実装コードについては https://github.com/athos/clojars-analysis に上げているので、興味がある方はそちらをご確認下さい。

集計結果のみ知りたい方は集計結果のセクションまでスクロールしましょう👇

## ライブラリ情報の収集

まず、期間中にリリースされたライブラリの情報を集めます。Clojarsは登録されているライブラリやグループに関する情報をAPIを通じて取得できるようになっているのでこれが使えそうに思えます。

しかし、Clojarsの提供するAPIはページング機能の使い勝手がよくなく、今回のように比較的長期間にリリースされたライブラリの情報を収集する目的にはあまり適しません。

そこで別の手段として、clojars-rssを使うことを考えます。[clojars-rss](https://github.com/athos/clojars-rss)は拙作の非公式RSSフィードで、Clojarsに登録されたライブラリの情報をRSSとして見られるように整形したものです。GitHub Pages上でホスティングされていて、RSSフィードの元となるデータはgitリポジトリで管理されていて、1時間おきにClojarsのAPIを叩いて更新しています。

ライブラリの更新情報は次のように行ごとのEDNマップとして記録されています：

```clojure
{:jar_name "kurosawa.web", :group_name "org.purefn", :version "2.1.23", :description "The Kurosawa web library.", :created 1639667977701}
{:jar_name "kurosawa.core", :group_name "org.purefn", :version "2.1.23", :description "The root Kurosawa library.", :created 1639667995471}
{:jar_name "kurosawa", :group_name "org.purefn", :version "2.1.23", :description "A catch-all project that brings in all Kurosawa libs.", :created 1639668031311}
...
```

なので、そのファイルをコミット履歴を遡って読み込んでいくことで期間中にリリースされたライブラリ情報を収集することができます：

```clojure
(let [head (current-commit-sha)
      commits (->> (exec "git rev-list"
                         "--after" start-date
                         "--before" end-date
                         "HEAD")
                   str/split-lines)]
  (try
    (reduce (fn [ret sha]
            (exec "git reset --hard" sha)
            (let [edn-file (io/file repo-path "data" "stable.edn")
                  m (load-edn edn-file)]
              (merge-with into ret m)))
          {}
          commits)
    (finally
      (exec "git reset --hard" head))))
```

## グループの承認ステータス確認

ライブラリのリリース情報とグループ情報が抽出できたので、次にどのグループが承認済みグループなのかを確認していきます。

グループの承認ステータスはClojarsの各ライブラリのページやAPIからは取得できないようになっています[^1]が、グループに属するユーザのページには各グループの承認ステータスがバッジで表示されています。グループに属するユーザはClojars APIのグループに関するエンドポイントから取得できるので、たとえばグループ`G`が承認済みグループかどうかを確認するには、

- Clojars APIでグループ`G`のメンバーユーザ`U`を取得
- メンバーユーザ`U`のClojars上のページから、ユーザ`U`が属する承認済みグループを収集し、その中にグループ`G`が含まれているかを確認

という手順を踏めばよさそうです。
ClojarsのwebページはIDやクラスが機械的に処理しやすいようにつけられていないので若干面倒ですが、[Enlive](https://github.com/cgrand/enlive)で適当にスクレイピングします：

```clojure
(require '[clj-http.client :as http]
         '[net.cgrand.enlive-html :as html])

(defn- group-member [group]
  (let [{:keys [body]} (http/get (str "https://clojars.org/api/groups/" group)
                                 {:as :json})]
    (:user (first body))))

(defn- check-verification [group]
  (let [user (group-member group)
        verified-groups (-> (URL. (str "https://clojars.org/users/" user))
                            html/html-resource
                            (html/select [:div
                                          (html/has [:> :span.verified-group])
                                          html/first-child
                                          html/first-child])
                            (#(into #{} (map (comp first :content)) %)))]
    (contains? verified-groups group)))
```

## 集計結果

以上のようにして収集できたライブラリおよびグループの情報について集計した結果を見ていきましょう。

まず、4月以降にリリースされた全ライブラリ中に占める、承認済みグループに属するライブラリの割合の推移について見てみます：

![](/images/clojars-verified-groups/verified-lib-rate-transition.png)

こちらは同じライブラリの異なるバージョンのリリースを別々にカウントして集計した結果になります。この結果を見ると、承認グループの制度開始直後の数ヶ月こそ未承認グループの下でリリースされたライブラリ数の方が多かったものの、すでに承認済みグループの下でリリースされるライブラリが半数以上を占めていることが分かります。

次に、4月以降にライブラリをリリースしたグループ全体に占める承認済みグループの割合も見てみます：

![](/images/clojars-verified-groups/verified-group-rate-transition.png)

こちらの結果では、おおむね順調に承認済みグループの占める割合が大きくなってきているのが見てとれます。承認済みグループの数自体はまだ未承認グループを超えるまでには至っていないようですが、今後数ヶ月〜1年程度で情勢が塗り替えられそうな勢いではあります。

リリース毎の集計結果とグループ毎の集計結果を総合してどう解釈するべきかは分かりませんが、承認済みグループの下でリリースされているライブラリは総じて新興のライブラリであり、新興ライブラリは枯れたライブラリと比較して活発にリリースを重ねる傾向にあるため、承認済みグループの数としては少なくても承認済みグループ下でのリリース数は多くなる、という状況なのかなと推測しています。

最後に承認済みグループの名前として使われるドメインの傾向について集計した結果を見てみましょう：

![](/images/clojars-verified-groups/verified-group-name-treemap.png)

承認済みグループの名前としてよく使われるトップレベルドメインは、上から順に`org`, `com`, `io`そして`net`という結果になりました。

一番多いのはClojarsがあらかじめ用意している`org.clojars.<username>`を使ったグループのようで、Clojarsが用意しているもう1つの承認グループ名である`net.clojars.<username>`と合わせて半数近く(46%)の割合を占めています。ClojarsのアカウントをGitHubアカウントと紐付けることで使えるようになる`com.github.<username>`や`io.github.<username>`も比較的上位を占め、2割程度のシェアを持ちます。

上記以外の`org.*`, `com.*`, `io.*`のグループが自己申請による承認グループの上位といえそうで、企業のグループとしては`com.*`と`io.*`が、その他の団体のグループとしては`org.*`と`io.*`が人気のようです。

全体の6%程度は他のグループと異なるユニークなトップレベルドメインを持つグループでした。ちなみに、`jp`ドメインを持つ承認済みグループも`jp.xcoo`の1グループのみのようでした🤔

## まとめ

Clojarsの承認済みグループのトレンドについて調べました。

承認済みグループの名前に迷う場合は、

- 個人ユーザなら、`org.clojars.<username>`か`com.github.<username>`
- 企業ユーザなら、`com.*`か`io.*`
- その他の団体なら、`org.*`か`io.*`

あたりが無難な選択になりそうです。

[^1]: これは意図的な措置のようで、グループの承認ステータスを公開することで、承認済みグループの制度が始まる以前にリリースされたライブラリが「信頼できない」ライブラリと誤って認識されて忌避されるのを防ぐ狙いがあるそうです。詳しくは[こちらのイシュー](https://github.com/clojars/clojars-web/pull/805)を参照下さい。
