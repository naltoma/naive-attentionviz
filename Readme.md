# AttentionViz（の一部）をBERTで再現してみる
[AttentionViz: A Global View of Transformer Attention](https://catherinesyeh.github.io/attn-docs/)でアテンションを可視化する際に「**アテンションに影響与えない範囲で積極的にベクトルをいじる**」というのが面白いので再現してみたい。

## どこに興味を持ったか
多次元ベクトルを2〜3次元空間に描画する際、一般的にはPCAなりで「なるべく情報量を保ちつつ低次元に落とし込む」ことを考える。

AttentionVizも最終的にはそうするのだけどその前に以下のような前提に基づき「**意図的にベクトルをいじっている**」ところが面白い。
- 重要なのはアテンションである（他見なくていいのかは置いておき、重要なのは確かだろう）。
- **アテンション求める部分の softmax, dot prodct に影響のない範囲でベクトル調整しちゃおうぜ！**（ヒャッハー！）。

## なぜ再現してみようとしたのか
- AttentioinVizでの狙いは「アテンションが高くなるペアはより近くなる」なんだけど、そもそもこういうアプローチでうまくいくのかを確認してみたい。
- デモサイトあるしGitHubでもコードっぽいの公開されてるんだけど、これらは用意された処理済みデータを観察することしかできない（多分）。
- 実際にLLMからQ, Kの重みを取得するといった部分を私が理解したかった。つまり勉強のため。

## 実際にやったこと
- 今回は1センテンス（"The brown capybara is sleeping now."）だけを用い、layer 0（1つ目の隠れ層）のみを対象とした。
- BERTからQ, K, Vの重みベクトルを取得。アテンションを計算してQ,K,Vが適切に取得できていることを確認。
- Q, Kの重みベクトルに対して、論文中の「5.1.1 Vector Nomalization」を実装。
  - **Key Translation**: 素朴に ``keys += a`` のように定数和とした。
  - **Scaling Queries and Keys**: 素朴に ``queries *= c``, ``keys *= 1.0/c`` とした。
  - これらの定数a, cは適当な範囲で「内積とコサイン距離の相関」の絶対値が最大となる組み合わせとした。ただし、前述の通りセンテンス数1個であること、対象とした層が1層だけであることが影響しているのか、aの探索範囲を少しでも大きくするとcがほとんど効かない。結果として相対的にほぼすべてのクエリがほぼ同じ場所に集中しがちになりやすい。ということで今回は探索幅を狭めにしています。
- 元論文 Fig 4 (b)相当を描画。
  - 求めたa,cを使いてベクトルを正規化し、内積 vs コサイン距離を描画。
- 元論文 Fig 4 (a)相当を描画。
  - 求めたa,cを使いてベクトルを正規化し、PCA(n_components=2)で描画を描画。なお元論文のこの図は ideal relationship（理想的な関係）とあるので、イメージ図だろうな。実際そういう風になるかを確認してみたくて描画してみました。
  - クエリとキーの直線は、クエリ基準で最大アテンションとなるキーに対して結びました。線の太さはアテンションの大きさです。
  - たまに赤線がありますが、これは意図的に観察したかったペアを選んでいます。
    - 具体的には brown をクエリとした際に、これがかかる（=アテンションが高くなると思われる） capybara をキーとし、それらの組み合わせが最大アテンションになってる場合に赤にしています。ただし capybara は3トークンに分割されているので、実際のキー対象は3種類になります。
    - より具体的には、「クエリ: bowrn <=> キー: cap/##y/##bara」のPCA空間における配置が、デフォルトよりも正規化後がより近くなって欲しい。

## 結論
BERTでQ, K, Vの重みベクトル取得する部分は十分うまくいった。他モデルでも似たような感じで書けるのかは未確認だけれども、なんとかなるといいな。

あくまでも1センテンスで1レイヤーのみしか処理していないのだけれども、
- 重み付け相関が高くなる定数a,cを見つける部分は想像以上に楽。かなり適当な数値を決め打ちで設定してもベクトル正規化すること自体は問題なさそうなレベル。
- 今回の範囲では「アテンションが高くなるペアがより近くなる」ようなケースはほとんど観察できなかった。一部はそれっぽくなってるケースもありました。
    - 結果の例
        - <a href="./result_layer0.html">layer 0</a> => head 12 では「クエリ:brown => キー:##bara」が近くなってる。
        - <a href="./result_layer1.html">layer 1</a>
        - <a href="./result_layer2.html">layer 2</a>

## やり残し
- 全レイヤーで統合した正規化。
    - 現時点では、layer_indexを変えることで他レイヤーについても参照できるようになっていますが、処理としては「指定したレイヤーだけで正規化する」ようになってます。このためレイヤー毎に異なる空間に写像されています。
- 重み付け相関
    - "We can thus choose a scale factor c such that the weighted correlation between query-key dot products and distances is maximized."
    - 上記部分を「正規化したQ,Kベクトルの内積とコサイン距離の相関が最大となるような定数cを求める」というように解釈したのだけど、実はこれは誤りかもしれない。正しくは「正規化したQ,Kベクトルの内積と、正規化したQ,KベクトルをPCA空間におけるユークリッド距離の相関が最大となるような定数cを求める」かもしれない。
- 「クエリ基準で最大アテンションとなるキーへ線を引く」こと自体は避けたほうが良かったかもしれない。アテンション行列内で上位N件を選択するほうが良かったかも（？）。
- 多数のサンプル＆全レイヤーでの観察。（この部分も再現する？）
- ファインチューニング前後での比較。

## tips的な何か
- 画像は out フォルダに保存するようにしています。無ければ作るようにしても良かったんだけど自分で用意しちゃったのでそのまま。
- Q,Kの重み取得しているところの layer_index を変えるだけで他レイヤーについても参照できます。ただしまじめに動作確認したのは layer_index = 0 のみ。

## 参考
- [AttentionViz: A Global View of Transformer Attention](https://catherinesyeh.github.io/attn-docs/): 元ネタ。
- [Source code for transformers.modeling_bert](https://huggingface.co/transformers/v3.2.0/_modules/transformers/modeling_bert.html#BertModel.forward): BertSelfAttention.forwardの部分が、Q, K, Vの重みベクトル取得する際に参考になった。
