---
title: "Go Conference 2024 に登壇しました！"
date: 2024-06-09T10:42:42+09:00
---

2024/06/08 に開催された Go Conference 2024 に登壇してきました！

初めて外部登壇した感想や採択から発表に至るまでの準備、セッションで話しきれなかったことについて書きたいと思います！

<!--more-->

## はじめに

[Go Conference 2024](https://gocon.jp/2024/) に登壇してきました！

- [gocon.jp - 詳解 "Fixing For Loops in Go 1.22" / 自作 linter を golangci-lint へコントリビュートした話](https://gocon.jp/2024/sessions/14/)
- [speakerdeck.com - スライド全文](https://speakerdeck.com/karamaru/xiang-jie-fixing-for-loops-in-go-1-dot-22-zi-zuo-linterwogolangci-linthekontoribiyutositahua)

{{< x src="https://twitter.com/karamaru_alpha/status/1799330747482054831?ref_src=twsrc%5Etfw" >}}

今回のブログでは、CfP の提出~調査~登壇までの流れを書いて、セッションでは話せなかったポイントなどについても記述したいと思います。

## 目次

- CfP を提出するまで
- CfP を提出してから登壇に至るまで
- 登壇当日
- セッションの要約・推しポイント・話しきれなかったこと

## CfP を提出するまで

今回のセッションはいわゆる「登壇駆動」でした。
登壇駆動というのは、面白そうな仕様や挙動を見つけて CfP を出し、その"後"にそれらの謎を証明していくという順番の発表/勉強手法のことです。
登壇界隈ではよく用いられる(自分も便利だと思う)テクニックですが、これってある種知識の前借りというか、登壇までに必ず提示した謎を証明しないといけないというリスクと責任が伴う行為ですよね。
このせいでスライド作成に行き詰まるなど非常に苦しい時間もあったのですが、なんとか解ききって発表にこぎつけられてよかったです...!

今回でいうと「Go1.22 でループ変数のアドレスを出力する時、fmt パッケージとビルトイン関数で挙動が違う」という"お題"だけ先に見つけて CfP を出しました。

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

(自分を含めた)多くの Gopher さんが「Go1.22 からイテレーション毎に値が再定義されるようになった」という話を 100 回は聞いていて、もう少し詳しい Gopher さんは「ループ変数がアドレス取得かクロージャキャプチャされる場合のみ再定義される」というところまで理解していたでしょう。
そんな前提知識があるからこそ、「ループ変数のアドレス取得がされているのにもかかわらず挙動が違う場合がある」というこのお題は、一見仕様と矛盾するように思えて大きく好奇心が掻き立てられました。

初めての登壇且つコントリビューションを含むということで「ChallengeSession」として応募しました。
public なセッション紹介には出ていませんが、運営への備考欄で熱い気持ちを書いた記憶があります 🔥

[gocon.jp - 詳解 "Fixing For Loops in Go 1.22" / 自作 linter を golangci-lint へコントリビュートした話](https://gocon.jp/2024/sessions/14/)

golangci-lint へのコントリビューションも CfP のタイトルに入れたのですが、結果登壇ではエスケープ解析を含むコンパイラの説明に 20 分かけ、残りの(残ってない)30 秒を linter の紹介に当てる結果になりました。

## CfP を提出してから登壇に至るまで

CfP が通ったのを知った時は「ずっと見ていた gocon に登壇できるんだ！」とすごく嬉しかったです！！🎉
「個人開発/研究で登壇する」のがエンジニア人生の 1 つの目標だったということもあり、田園都市線でにやにやしながらツイートしたのを覚えています 🚃

{{< x src="https://twitter.com/karamaru_alpha/status/1775902582915096806?ref_src=twsrc%5Etfw" >}}

さて、登壇駆動で CfP を通したのですから、次は謎を解明する義務がありますね。
幸い、ループについてのプロポーザルやドキュメント、大まかなコードの変更箇所は知っていたので、これを追っていけば自ずとわかるだろうと高を括っていました。これが大きな間違いでした orz

具体的には下記あたりで、for-loop についての議論やプロポーザル、実装箇所や FAQ を見ることができます。
登壇スライドは 1 章~5 章の構成なのですが、これらのソースは 1 章~3 章にあたります。

- https://github.com/golang/go/discussions/56010
- https://github.com/golang/go/issues/60078
- https://go.googlesource.com/proposal/+/master/design/60078-loopvar.md
- https://go.dev/wiki/LoopvarExperiment
- https://github.com/golang/go/issues/57969
- https://go-review.googlesource.com/c/go/+/411904

さて、上記で全てが説明できると思っていたのですが、実際そう簡単にはいきませんでした。

冒頭で見つけた fmt パッケージとビルトイン関数での挙動の違いについて、アップデートの差分を見るだけでは何も説明がつかなかったのです。 「どちらのケースであってもコンパイラが中間表現(IR)ノードを書き換えてループ変数の再宣言が行われている」という事実が自身のデバッグによって証明できてしまった時は本当に絶望しました orz

詳しくは発表スライドを参照して欲しいのですが、fmt とビルトインの挙動の違いにはエスケープ解析が関係していて、この辺りは当然今回のアップデートとは無関係だったんですね。
だからこの関係性に辿り着くまでに時間がかかりましたし、実際に fmt で引数がエスケープする原因を深掘るのにも骨が折れました。

そもそもビルトイン関数の`print`なんて普段誰も使いません(実際使うのは非推奨です)し、となると当然 stackoverflow などの質問サイトにも参考になるものがなかったんですよね。
謎の証明について何のとっかかりも無くなった瞬間はすごく焦りました...

でもだからこそいっぱい実験をして、(厳密には正確でないかも知れないけど)自分が提示した謎に対して筋を通した説明が見つかった時は純粋に嬉しかったですし、高揚しました。
みんなが知らないような面白いお題が見つかって、苦節がありつつもそれを説明できるようになったからには、これをしっかり伝えたい！という気持ちが強くなりました。
そのせいでスライドが 100 枚ぐらいになって、登壇で半分以上省く結果になるのですが...笑

## 登壇当日

朝から登壇するまではとても緊張していて、 お昼に海鮮丼を食べた時は吐きそうになっていたぐらいです笑

けど頑張って調べてきた自由研究ですし、個人的にはすごく面白い題材だったので、"[訊こえるか、僕の存在証明讃歌が！](https://www.youtube.com/watch?v=gGGgoW_vgWo)"というような気持ちで、自分のセッションが始まってからはその熱量で楽しく話すことができました 🔥

(自分の能力の範囲で)調べてきた結果をあんなに大勢の人に聞いてもらえるというのは、なんだか幸せな時間でした。
あぁ、よく外部登壇をしてる人は自身の発信力を上げるどうこうよりも、この楽しさを知ってからなんだろうなと感じました。

正直登壇中に聞いてくださっている方々の表情を伺う余裕はなかったのですが、発表が終わってから X を見返すと予想よりも楽しんでくれた方が多かったようでとっっっても嬉しかったです！☺️
頑張ってよかったって思いました、、、あとみんな優しいです、、🥺

以下嬉しかったツイート抜粋です....! (無許可なので消して欲しい場合はご連絡ください mm)

{{< x src="https://twitter.com/tenntenn/status/1799353321997869165?ref_src=twsrc%5Etfw" >}}

{{< x src="https://twitter.com/y_taka_23/status/1799354192542494787?ref_src=twsrc%5Etfw" >}}

{{< x src="https://twitter.com/__syumai/status/1799353720729387350?ref_src=twsrc%5Etfw" >}}

{{< x src="https://twitter.com/_ryukez/status/1799360539824722262?ref_src=twsrc%5Etfw" >}}

{{< x src="https://twitter.com/B_Sardine/status/1799355116019188067?ref_src=twsrc%5Etfw" >}}

{{< x src="https://twitter.com/HYorimitsu/status/1799354079174676495?ref_src=twsrc%5Etfw" >}}

{{< x src="https://twitter.com/p1ass/status/1799354554578096331?ref_src=twsrc%5Etfw" >}}

嬉しいので全部にいいねしちゃいました 🥰
聞いてくれた方々、ありがとうございました！！！🎉

## セッションの要約・推しポイント・話しきれなかったこと

各章の要約と推しポイントを列挙してみました。
スライドを全て見るのがめんどくさい人や、当日省略したところを知りたい人はこちらを参照してください mm

#### 0~2 章: プロポーザルやデザインドックからアップデート内容を確認する

2 章までは、みなさまがよく知っているループに関するアップデート内容の振り返りでした。
Gocon に来るような人にとっては常識だろうと思い、発表では全カットとなった部分です。

推しポイントはやはり[Go クイズ](https://speakerdeck.com/qualiarts/xiang-jie-fixing-for-loops-in-go-1-dot-22-zi-zuo-linterwogolangci-linthekontoribiyutositahua?slide=6)です。
リリースノートの一歩先で、直感に反するようなループの挙動について Go1.22 以前と以降でそれぞれ問題を作ったのでぜひ解いてみてください。
自己紹介より前に話したかった、有識者ほど引っかかるお気に入りの導入です。

発表で伝えられなくて最も惜しいと思ったのは[discussion における C#チームの意見](https://speakerdeck.com/qualiarts/xiang-jie-fixing-for-loops-in-go-1-dot-22-zi-zuo-linterwogolangci-linthekontoribiyutositahua?slide=29)です。
C#5.0 で Go1.22 と似たアップデートがあったのをご存知でしょうか？
また、C#では for 文と for-each 文でループ変数のインスタンス割り当ての挙動が異なることを知っていますか？
Go のセマンティクス変更を後押しした面白い議論の 1 つなので、ぜひご覧ください。

#### 3 章: コンパイラの内部実装を読み解く

3 章ではコンパイラの役割を紹介した後、Go1.22 から加わった loopvar パッケージについてコードリーディングしました。
どのようなケースでループ変数の暗黙的な再宣言が必要か判断され、中間表現(IR)ノードの書き換えが起きるのかを確認しました。

推しポイントは、[Go のソースコードを変更して実際に IR ノードを出力するなどのデバッグを行った部分](https://speakerdeck.com/qualiarts/xiang-jie-fixing-for-loops-in-go-1-dot-22-zi-zuo-linterwogolangci-linthekontoribiyutositahua?slide=39)ですかね。
普段 Go 自体のコードを変更して再ビルドすることは少ないと思うので、(自分をはじめとした)言語のデバッグ経験がない人にとってはいい例になったのではないでしょうか 🙆

#### 4 章: エスケープ解析

4 章ではエスケープ解析とアドレス割り当て、fmt パッケージの関連性について触れました。
この章は最も力を入れた部分で、推しポイントは 4 つあります。

1 つ目は[fmt パッケージとビルトインのプリント関数において引数のエスケープ結果が異なるという事象の解説](https://speakerdeck.com/qualiarts/xiang-jie-fixing-for-loops-in-go-1-dot-22-zi-zuo-linterwogolangci-linthekontoribiyutositahua?slide=72)です。
本セッションの根幹部分でしたね。
一見引数のプリントするだけの処理でもレキシカルスコープを外れる可能性について示唆した[Go クイズ](https://speakerdeck.com/qualiarts/xiang-jie-fixing-for-loops-in-go-1-dot-22-zi-zuo-linterwogolangci-linthekontoribiyutositahua?slide=75)もぜひご覧ください。

2 つ目は[標準パッケージから見るエスケープ対策](https://speakerdeck.com/qualiarts/xiang-jie-fixing-for-loops-in-go-1-dot-22-zi-zuo-linterwogolangci-linthekontoribiyutositahua?slide=52)です。`io.Reader`を参考に、変数をヒープに乗せないための面白い工夫を紹介しました。
賢ぶっても仕方がないので正直に話しますが、この例は「[GopherCon2019 Understanding Allocations: the Stack and the Heap](https://www.youtube.com/watch?v=ZMZpH4yT7M0)」から持ってきたものです。
こちらの発表はエスケープ解析の理解を深める上でとても参考になるので、興味のある方はぜひみてみてください。

3 つめは[Go ディレクティブを用いて引数をエスケープさせない fmt.Println を作成する方法](https://speakerdeck.com/qualiarts/xiang-jie-fixing-for-loops-in-go-1-dot-22-zi-zuo-linterwogolangci-linthekontoribiyutositahua?slide=80)です。
こちらは登壇で省略した部分で、 ディレクティブ`go:noescape`と`go:linkname`を使うことで、引数をエスケープさせない`fmt.Println`を作るという試みです。
変わり種なので省きましたが、取り組み自体は面白いと思っているので、こちらもぜひご参照ください。

4 つめは[アセンブリベースでのスタック割り当ての挙動を見てみる](https://speakerdeck.com/qualiarts/xiang-jie-fixing-for-loops-in-go-1-dot-22-zi-zuo-linterwogolangci-linthekontoribiyutositahua?slide=58)です。
こちらも発表で省略した部分です。変数がスタック割り当てになる時、実際に同じアドレスが使いまわされる挙動を確認してみよう！という趣旨のコーナーになっています。
正直自分の理解があっているか怪しい部分でもあるので、間違っていた場合はごめんなさい mm

#### 5 章: 周辺ツールと karamaru-alpha/copyloopvar、静的解析について

5 章では公式から提供されたツール(`gcflags loopvar`, `bisect`)の紹介と、僕が作成した Go1.22 から不要になったループ変数の再宣言を検知するリンター[copyloopvar の紹介](https://speakerdeck.com/qualiarts/xiang-jie-fixing-for-loops-in-go-1-dot-22-zi-zuo-linterwogolangci-linthekontoribiyutositahua?slide=83)をしました。

本番では 30 秒で終わらせた部分ですね。笑

話す時間がなかったというのはもちろんそうなのですが、tenntenn さんが 300 ページぐらいの解説記事を既に書かれていてこれでいいじゃん感がすごかったんです笑 (熱量がすごい)
とても自分の説明で賄える範囲ではないと考え、今回詳細は省略させていただきました mm

本セッションの目的の 1 つでもあった「OSS 初心者であっても言語のアップデート駆動という形で何かをつくってみるといいのではないか」ということが伝えられてよかったです。
ちなみに、golangci-lint では今回紹介した copyloopvar の他に、[intrange](https://github.com/ckaznocha/intrange)という range over int を推奨する linter が Go1.22 のアップデート契機で追加されました。

以上が要約と推しポイントです。1 つでも気になった部分があれば、ぜひスライドをご覧ください mm

[speakerdeck.com - スライド全文](https://speakerdeck.com/qualiarts/xiang-jie-fixing-for-loops-in-go-1-dot-22-zi-zuo-linterwogolangci-linthekontoribiyutositahua)

## おわりに

初めての外部登壇でしたが、とても楽しかったです！🎉

聞いてくれた方々はもちろん、貴重な経験の舞台を整えてくれた運営の方々には大感謝です 🙇

絶対次もどこかで登壇したいし面白い題材を見つけていきたいです。 最高のイベントでした、ありがとうございました！

![pass.png](pass.png)

![sign.png](sign.png)
