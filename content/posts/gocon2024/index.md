---
title: "Go Conference 2024 に登壇しました！"
date: 2024-06-09T10:42:42+09:00
---


2024/06/08に開催された Go Conference 2024 に登壇してきました！

初めて外部登壇した感想や採択から発表に至るまで準備、セッションで話しきれなかったことについて書きます！

<!--more-->

## はじめに

[Go Conference 2024](https://gocon.jp/2024/) に登壇してきました！

- [gocon.jp - 詳解 "Fixing For Loops in Go 1.22" / 自作linterをgolangci-lintへコントリビュートした話](https://gocon.jp/2024/sessions/14/)
- [speakerdeck - スライド全文](https://speakerdeck.com/qualiarts/xiang-jie-fixing-for-loops-in-go-1-dot-22-zi-zuo-linterwogolangci-linthekontoribiyutositahua)


{{< x src="https://twitter.com/karamaru_alpha/status/1799330747482054831?ref_src=twsrc%5Etfw" >}}

今回のブログでは、CfPの提出~調査~登壇までの流れを書いて、セッションでは話せなかったポイントなどについても記述したいと思います。

## 目次

- CfPを提出するまで
- CfPを提出してから登壇に至るまで
- 登壇当日
- セッションの要約・推しポイント・話しきれなかったこと

## CfPを提出するまで

今回のセッションはいわゆる「登壇駆動」でした。
登壇駆動というのは、面白そうな仕様や挙動を見つけてCfPを出し、その"後"にそれらの謎を証明していくという順番の発表/勉強手法のことです。
登壇界隈ではよく用いられる(自分も便利だと思う)テクニックですが、これってある種知識の前借りというか、登壇までに必ず提示した謎を証明しないといけないというリスクと責任が伴う行為ですよね。

今回でいうと「Go1.22でループ変数のアドレスを出力する時、fmtパッケージとビルトイン関数で挙動が違う」という"お題"だけ先に見つけてCfPを出しました。

```go
for v := range 3 {
	// 全てのイテレーションで"異なる"アドレスを出力する
	fmt.Println(&v)
}

for v := range 3 {
	// 全てのイテレーションで"同一の"アドレスを出力する
	println(&v)
}
```

このお題を見つけた時、心から面白いと思いました。

(自分を含めた)多くのGopherさんが「Go1.22からイテレーション毎に値が再定義されるようになった」という話を100回は聞いていて、もう少し詳しいGopherさんは「ループ変数がアドレス取得かクロージャキャプチャされる場合のみ再定義される」というところまで理解していたでしょう。
そんな前提知識があるからこそ、「ループ変数のアドレス取得がされているのにもかかわらず挙動が違う場合がある」というこのお題は、一見仕様と矛盾するように思えて好奇心が掻き立てられました。

golangci-lintへのコントリビューションもCfPのタイトルに入れたのですが、結果登壇ではエスケープ解析を含むコンパイラの説明に20分かけ、残りの(残ってない)30秒をlinterの紹介に当てる結果になりました。
厳密にいうとスライド自体は作成したんですが、どうしても20分では話しきれない枚数になってしまってomitしたんですよね。結果タイトルが半分釣りみたいになったのは申し訳なかったですmm

初めての登壇且つコントリビューションを含むということで「ChallengeSession」として応募しました。
publicなセッション紹介には出ていませんが、備考欄で熱い気持ちを書いた記憶があります。

[gocon.jp - 詳解 "Fixing For Loops in Go 1.22" / 自作linterをgolangci-lintへコントリビュートした話](https://gocon.jp/2024/sessions/14/)

## CfPを提出してから登壇に至るまで

CfPが通ったのを知った時は「ずっと見ていたgoconに登壇できるんだ！」とすごく嬉しかったです！！
「個人開発/研究で登壇する」のがエンジニア人生の1つの目標だったということもあり、田園都市線でにやにやしながらツイートしたのを覚えています笑

{{< x src="https://twitter.com/karamaru_alpha/status/1775902582915096806?ref_src=twsrc%5Etfw" >}}

さて、登壇駆動でCfPを通したのですから、次は謎を解明する義務がありますね。
幸い、ループについてのプロポーザルやドキュメント、大まかなコードの変更箇所は知っていたので、これを追っていけば自ずとわかるだろうと高を括っていました。これが大きな間違いでしたorz

具体的には下記あたりで、for-loopについての議論やプロポーザル、実装箇所やFAQを見ることができます。
登壇スライドは1章~5章の構成なのですが、これらのソースは1章~3章にあたります。
- https://github.com/golang/go/discussions/56010
- https://github.com/golang/go/issues/60078
- https://go.googlesource.com/proposal/+/master/design/60078-loopvar.md
- https://go.dev/wiki/LoopvarExperiment
- https://github.com/golang/go/issues/57969
- https://go-review.googlesource.com/c/go/+/411904

さて、上記で全てが説明できると思っていたのですが、実際そう簡単にはいきませんでした。

冒頭で見つけたfmtパッケージとビルトイン関数での挙動の違いについて、アップデートの差分を見るだけでは何も説明がつかなかったのです。 「どちらのケースであってもコンパイラが中間表現(IR)ノードを書き換えてループ変数の再宣言が行われている」という事実が自身のデバッグによって証明できてしまった時は本当に絶望しましたorz 

詳しくは発表スライドを参照して欲しいのですが、fmtとビルトインの挙動の違いにはエスケープ解析が関係していて、この辺りの変更は当然今回のアップデートとは無関係だったんですね。
だからこの関係性に辿り着くまでに時間がかかりましたし、実際にfmtとビルトインで引数がエスケープしない原因を深掘るのにも骨が折れましたorz


そもそもビルトイン関数の`print`なんて普段誰も使いませんし、となると当然stackoverflowなどの質問サイトにも参考になるものがなかったんですよね。
謎の証明について何のとっかかりも無くなった瞬間はすごく焦りましたし、自分の知識不足を恨みました。

でもだからこそ、(厳密には正確でないかも知れないけど)、自分が提示した謎に対して筋を通した説明が見つかった時は純粋に嬉しかったですし、高揚しました。
みんなが知らないような面白いお題が見つかって、苦節がありつつもそれを説明できるようになったからには、これをしっかり伝えたい！という気持ちが強くなりました。
そのせいでスライドが100枚ぐらいになって、登壇で半分以上省く結果になるのですが...笑

## 登壇当日

朝から登壇するまではとても緊張していて、 お昼に海鮮丼を食べた時は吐きそうになっていたぐらいです笑

けど頑張って調べてきた自由研究ですし、個人的にはすごく面白い題材だったので、"[訊こえるか、僕の存在証明讃歌が！](https://www.youtube.com/watch?v=gGGgoW_vgWo)"というような気持ちで、自分のセッションが始まってからはその熱量で楽しく話すことができました🔥

(自分の能力の範囲で)調べてきた結果をあんなに大勢の人に聞いてもらえるというのは、なんだか幸せな時間でした。
あぁ、よく外部登壇をしてる人は自身の発信力を上げるどうこうよりも、この楽しさを知ってからなんだろうなと感じました。

正直登壇中の記憶はあんまりないのですが、発表が終わってからXを見返すと楽しんでくれた人が予想よりも多かったようで、とっっっても嬉しかったです！☺️
頑張ってよかったって思いました、、、あとみんな優しい、、🥺


以下嬉しかったツイート抜粋です....! (無許可なので消して欲しい場合はご連絡くださいmm)

{{< x src="https://twitter.com/tenntenn/status/1799353321997869165?ref_src=twsrc%5Etfw" >}}

{{< x src="https://twitter.com/y_taka_23/status/1799354192542494787?ref_src=twsrc%5Etfw" >}}

{{< x src="https://twitter.com/__syumai/status/1799353720729387350?ref_src=twsrc%5Etfw" >}}

{{< x src="https://twitter.com/_ryukez/status/1799360539824722262?ref_src=twsrc%5Etfw" >}}

{{< x src="https://twitter.com/B_Sardine/status/1799355116019188067?ref_src=twsrc%5Etfw" >}}

{{< x src="https://twitter.com/HYorimitsu/status/1799354079174676495?ref_src=twsrc%5Etfw" >}}

{{< x src="https://twitter.com/p1ass/status/1799354554578096331?ref_src=twsrc%5Etfw" >}}

嬉しくて全部にいいねしました笑
聞いてくれた方々、ありがとうございました！！！

## セッションの要約・推しポイント・話しきれなかったこと

各章の推しポイントを列挙してみました。
スライドを全て見るのがめんどくさい人や、当日省略したところを知りたい人はこちらを参照してくださいmm

#### 0~2章: プロポーザルやデザインドックからアップデート内容を確認する

2章までは、みなさまがよく知っているループに関するアップデート内容の振り返りでした。
Goconに来るような人にとっては常識だろうと思い、発表では全カットとなった部分です。

推しポイントはやはり[Goクイズ](https://speakerdeck.com/qualiarts/xiang-jie-fixing-for-loops-in-go-1-dot-22-zi-zuo-linterwogolangci-linthekontoribiyutositahua?slide=6)です。
直感に反するようなループの挙動についてGo1.22以前と以降でそれぞれ問題を作ったので、ぜひ解いてみてください。
自己紹介より前に話したかった、有識者ほど引っかかる推しの導入です。

発表で伝えられなくて最も惜しいと思ったのは[discussionにおけるC#チームの意見](https://speakerdeck.com/qualiarts/xiang-jie-fixing-for-loops-in-go-1-dot-22-zi-zuo-linterwogolangci-linthekontoribiyutositahua?slide=29)です。
C#5.0でGo1.22と似たアップデートがあったのを知っていますか？
また、C#ではfor文とfor-each文でループ変数のインスタンス割り当ての挙動が異なることを知っていますか？
Goのセマンティクス変更を後押しした面白い議論の1つです。


#### 3章: コンパイラの内部実装を読み解く

3章ではコンパイラの役割を紹介した後、Go1.22から加わったloopvarパッケージについてコードリーディングしました。
どのようなケースでループ変数の暗黙的な再宣言が必要か判断され、中間表現(IR)ノードの書き換えが起きるのかを確認しました。

推しポイントは、[Goのソースコードを変更して実際にIRノードを出力するなどのデバッグを行った部分](https://speakerdeck.com/qualiarts/xiang-jie-fixing-for-loops-in-go-1-dot-22-zi-zuo-linterwogolangci-linthekontoribiyutositahua?slide=39)ですかね。
普段Go自体のコードを変更して再ビルドすることは少ないと思うので、(自分をはじめとした)言語のデバッグ経験がない人にとってはいい例になったのではないでしょうか🙆


#### 4章: エスケープ解析

4章ではエスケープ解析とアドレス割り当て、fmtパッケージの関連性について触れました。
この章は最も力を入れた部分で、推しポイントは4つあります。

1つ目は[fmtパッケージとビルトインのプリント関数において引数のエスケープ結果が異なるという事象の解説](https://speakerdeck.com/qualiarts/xiang-jie-fixing-for-loops-in-go-1-dot-22-zi-zuo-linterwogolangci-linthekontoribiyutositahua?slide=72)です。
本セッションの根幹部分でしたね。
一見引数のプリントするだけの処理でもレキシカルスコープを外れる可能性について示唆した[Goクイズ](https://speakerdeck.com/qualiarts/xiang-jie-fixing-for-loops-in-go-1-dot-22-zi-zuo-linterwogolangci-linthekontoribiyutositahua?slide=75)もぜひご覧ください。

2つ目は標準パッケージから見るエスケープ対策です。`io.Reader`を参考に、変数をヒープに乗せないための面白い工夫を紹介しました。
賢ぶっても仕方がないので正直に話しますが、この例は「[GopherCon2019 Understanding Allocations: the Stack and the Heap](https://www.youtube.com/watch?v=ZMZpH4yT7M0)」から持ってきたものです。
こちらの発表はエスケープ解析の理解を深める上でとても参考になるので、興味のある方はぜひみてみてください。

3つめは[Goディレクティブを用いた引数がエスケープしないfmt.Printlnを作成する方法](https://speakerdeck.com/qualiarts/xiang-jie-fixing-for-loops-in-go-1-dot-22-zi-zuo-linterwogolangci-linthekontoribiyutositahua?slide=80)です。
こちらは登壇で省略した部分なのです。
ディレクティブ`go:noescape`と`go:linkname`を使うことで、`fmt.Println`でループ変数のアドレスを出力するにもかかわらず、全てのイテレーションで同一のアドレスを出力させることができるという紹介です。
変わり種なので省きましたが試み自体は面白いと思っているので、こちらもぜひご参照ください。

4つめは[アセンブリベースでのスタック割り当ての挙動を見てみる](https://speakerdeck.com/qualiarts/xiang-jie-fixing-for-loops-in-go-1-dot-22-zi-zuo-linterwogolangci-linthekontoribiyutositahua?slide=58)です。
こちらも発表で省略した部分です。変数がスタック割り当てになる時、実際に同じアドレスが使いまわされる挙動を確認してみよう！という趣旨のコーナーになっています。
正直自分の理解があっているか怪しい部分でもあるので、間違っていた場合はごめんなさいmm

#### 5章: 周辺ツールとkaramaru-alpha/copyloopvar、静的解析について

5章では公式から提供されたツール(`gcflags loopvar`, `bisect`)の紹介と、僕が作成したGo1.22から不要になったループ変数の再宣言を検知するリンターcopyloopvarの紹介をしました。

本番では30秒で終わらせた部分ですね。笑

話す時間がなかったというのはもちろんそうなのですが、tenntennさんが300ページぐらいの解説記事を既に書かれていてこれでいいじゃん感がすごかったんです笑 (熱量がすごい)
とても自分の説明で賄える範囲ではないと考え、今回詳細は省略させていただきましたmm


以上が要約と推しポイントです。1つでも気になった部分があれば、ぜひスライドをご覧くださいmm

[speakerdeck - スライド全文](https://speakerdeck.com/qualiarts/xiang-jie-fixing-for-loops-in-go-1-dot-22-zi-zuo-linterwogolangci-linthekontoribiyutositahua)

## おわりに

初めての外部登壇でしたが、とても楽しかったです！

聞いてくれた方々はもちろん、貴重な経験の舞台を整えてくれた運営の方々には大感謝です🙇

絶対次もどこかで登壇したいし面白い題材を見つけていきたいです！

最高のイベントでした、ありがとうございました！


![pass.png](pass.png)

![sign.png](sign.png)
