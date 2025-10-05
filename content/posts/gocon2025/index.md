---
title: "Go Conference 2025 に登壇しました！"
date: 2025-10-04T10:42:42+09:00
image: "thumbnail.jpg"
---

2025/09/27,28 に開催された Go Conference 2025 に登壇してきました！

登壇準備の過程でオランダ人の友達ができたこと、当日の雰囲気、セッションで話しきれなかったことなどについて書きたいと思います。

<!--more-->

## 雑談

> **「自分、好きみたいなんです。知らなかったことを知るのが。」**

[SHIROBAKO 13 話『好きな雲って何ですか？』](http://shirobako-anime.com/story/13.html)より。

{{< figure src="aoi.jpg" class="center" width="400" >}}

主人公に対してりーちゃんが調べ物のお手伝いを立候補する時のセリフ。

今井みどり（りーちゃん）はアニメ脚本家を夢見る大学生。アニメ制作会社で働く先輩のために調べ物（ディーゼル車について）を自律的に手伝うようになり、本職の脚本家に見込まれ弟子入りを果たす。

りーちゃんの、周りとの差に焦りつつも今できることを 1 つ 1 つ進めていく所や、好奇心のためなら損得勘定抜きで物事にのめり込める所がとても好き。

わかる。知ることって楽しいよね。そういう話を書きます！

（[SHIROBAKO](http://shirobako-anime.com/)、全社会人に見て欲しい...！）

## はじめに

**[Go Conference 2025](https://gocon.jp/2025/) に登壇してきました！** 楽しかった ☺️

CfP のテーマ選定や調査時苦労したこと、当日の様子などを記述します！

- [gocon.jp - コード生成なしでモック処理を実現！ovechkin-dm/mockio で学ぶメタプログラミング](https://gocon.jp/2025/talks/958503/)
- [speakerdeck.com - スライド全文](https://speakerdeck.com/karamaru/mockio-go-dynodexue-bumetapuroguramingu)
- [X - 告知ツイート](https://x.com/karamaru_alpha/status/1972210787487043977)

{{< slide src="https://speakerdeck.com/player/cdc2be8c99b04d1487cc3a82d73629b2" >}}

### 目次

- テーマ選定理由
- 苦戦した go-dyno 内部実装調査
- 作者との思い出
- 当日
- セッションの要約・推しポイント・話しきれなかったこと

## テーマ選定理由

セッションは以下の順でお話しました。本番では時間の関係で 前半 2 章はスキップでしたが 😇

1. そもそもモックとは何か？既存モックライブラリの仕組み
2. mockio の使い方・美しすぎる設計
3. 動的メソッド生成を行う go-dyno の仕組み

テーマの選定理由は、**[mockio](https://github.com/ovechkin-dm/mockio) というモックライブラリの設計思想が好きすぎた**からです。

**モックライブラリについて調べるのは昔から好き**で、外部登壇や社内勉強会で[設計比較](https://speakerdeck.com/karamaru/motukuraiburariapurotibi-jiao)を行ったり、簡易的な[自作](https://github.com/karamaru-alpha/mocka)を行ったりしていました。

欲しいのはモックオブジェクトなのか、スタブなのか？振る舞いの指定にどこまで型安全性を求めるのか？コード生成は許容するか？対象 I/F の指定方法は？などなど、選定に様々な観点があって奥深いんですよね！

そんな中、 **モックオブジェクトの事前生成を必要とせず、generics で型安全に振る舞いを指定できる**mockio を見つけて、設計思想に強く感動した記憶があります 👏 当時は star 数も 2 桁でした。

{{< details summary="mockio_sample.go" >}}

```go
// main.go
package main

type Calculator interface {
	Increment(n int) int
}

func Foo(calculator Calculator) int {
	return calculator.Increment(1)
}

// main_test.go
package main

import (
	"testing"

	"github.com/ovechkin-dm/mockio/v2/mock"
	"github.com/stretchr/testify/assert"
)

func Test_Foo(t *testing.T) {
	ctrl := mock.NewMockController(t)
     // runtimeでモックオブジェクト生成！
	mockCalculator := mock.Mock[Calculator](ctrl)
	mock.WhenSingle(mockCalculator.Increment(1)).ThenReturn(2)
	assert.Equal(t, 2, Foo(mockCalculator))
	mock.Verify(mockCalculator, mock.Times(1)).Increment(1)
}
```

{{< /details >}}

何度見ても美しいインターフェースだ...！[bytedance/mockey](https://github.com/bytedance/mockey)みたいにモンキーパッチで無理やりランタイムでモックする手法ならあったけど、型安全性まで generics で保証されている！

ふんわり中身を見てみると、**[go-dyno](https://github.com/ovechkin-dm/go-dyno)というライブラリが構造体・メソッドの動的生成を実現している**ことが分かりました。動的に振る舞いを変更できるなんてメタプログラミングみたいでかっこいい！そう思い調査を開始し、CfP にこのテーマを選択しました。

{{< details summary="mockio_sample.go" >}}

```go
package main

import (
	"fmt"
	"reflect"

	"github.com/ovechkin-dm/go-dyno/pkg/dyno"
)

type Calculator interface {
	IsEven(n int) bool
	IsOdd(n int) bool
}

func main() {
	dynamicCalculator, _ := dyno.Dynamic[Calculator](func(m reflect.Method, values []reflect.Value) []reflect.Value {
		if m.Name == "IsEven" {
			return []reflect.Value{reflect.ValueOf(values[0].Int()%2 == 0)}
		}
		if m.Name == "IsOdd" {
			return []reflect.Value{reflect.ValueOf(values[0].Int()%2 == 1)}
		}
		return []reflect.Value{}
	})
	fmt.Println(dynamicCalculator.IsOdd(1))
	// Output: true
	fmt.Println(dynamicCalculator.IsEven(1))
	// Output: false
}
```

{{</ details >}}

{{< x src="https://twitter.com/karamaru_alpha/status/1908879347445817487?ref_src=twsrc%5Etfw" >}}

## 苦戦した go-dyno 内部実装調査

**go-dyno がどのようにメソッドの振る舞いを動的に指定できているのか**の調査がとても大変だったー 😭

詳しくは発表資料参照なのですが、go-dyno は「メソッドの振る舞いは本来動的に切り替えられない」 → 「関数の振る舞いは動的に切り替えられる」 → 「**メソッドから関数を呼び出して引数はメソッドのやつを覗き見しよう！** 」という作戦のライブラリです。

このライブラリの仕組みを理解するには、前提知識として少なくとも以下が必要になります。
全て自分も馴染みが薄い部分だったので非常に理解に時間がかかりました 😭

- アセンブリの書き方
- unsafe.Pointer の使い方
- メソッドテーブルの仕組み
- CPU（レジスタ）とメモリ（スタック）による引数伝搬
- スタックフレームの積まれ方と内部構造
- 関数とメソッドの違い

上記知識は**全てスライドに Tips として紹介**しました。
このセッション自体、ライブラリの紹介というよりは、「道中で現れる **Go の深い部分を覗き見させる**」ことが目標だったので、ぜひ見てくれると嬉しいです！

{{< figure src="tips1.jpg" class="center" width="400" >}}

さて、最も理解が難しかったのは**メソッドと関数の引数共有トリック**です。

関数に引数を 2 つ（int 型と int 配列型）余分に渡すことで、直前に呼び出されたメソッドの引数をあたかも関数の引数かのように振る舞うことができる〜という部分です。

調べていくとレジスタとスタックの領域をスキップするためなんだと分かる訳ですが、当時初見の僕はこれがなぜ動作するのか理解できませんでした。

コード上には図解はないですし、そもそもメソッドと引数を共有するためのトリックなんだともコメントされてない訳ですし 😭

{{< figure src="trick.jpg" class="center" width="400" >}}

## 作者との思い出

ということで、**[mockio/go-dyno 作者の ovechkin さん](https://github.com/ovechkin-dm)に直接連絡して質問**することにしました。GitHub に SNS リンクがなかったのですが、なんとか見つけ出すことに成功しました。オランダに住んでいる人のようです。

追い詰められた人間って怖いですね。（思い返せば GoogleCloudNext の参加者の写真に写っていた PC のアドレスバーから[Go チームがブースで出題していたサイトを特定した](https://x.com/karamaru_alpha/status/1832987457476006015)過去もありました。）

「コード生成をせずにモック処理を行える**あなたのライブラリが好き**だ。内部処理をこのように理解してるんだけど、ここだけ理解できないから質問してもいいかな？」みたいな感じでダメ元連絡しました。すると、1 日も立たず返信を返してくれて、内部処理について教えてくださいました。

主に関数とメソッドで引数を共有するトリックについて、引数のレジスタ・スタック割り当てを利用していること、スタックが連続した時の FP の offset が 16 バイトになることを丁寧に教えてくれました（16 バイトの部分は OS や Go バージョンにより異なりビルドタグで制御される）。

お話する過程で、スタックフレームの構造や、関数とメソッドの違いなどについての理解が大きく深まりました。

連続するスタックフレームのオフセットについて[例示を僕のリポジトリにコミット](https://github.com/karamaru-alpha/playground/pull/1)してくれたり、登壇を応援してくれたり、本当に優しい人だった...🥰

いきなり押しかけた上、自分の物分かりもすごく悪かったのだけれど、根気強く・やさしく教えてくれた作者様には感謝してもしきれません。本当にありがとうございました！😭

{{< figure src="ovechkin1.jpg" class="center" width="300" >}}

作者さんの温かい協力もあり、全ての仕組みがわかった時本当に気持ちよかったなあ。
やっぱり**知らないことを知るのが楽しい！**

技術って好奇心を満たせるのもいいけど、今回みたいに**国を超えてお話できるきっかけ**にもなるから素敵だなって思いました。

発表資料を見て少しでもいいなと思ったら、**ぜひ[スポンサー](https://github.com/sponsors/ovechkin-dm)してあげてください**！自分も微力ながらさせていただきまいｓた 👏

{{< figure src="ovechkin2.jpg" class="center" width="300" >}}

{{< figure src="sponsor.jpg" class="center" width="400" >}}

## 当日

4 部屋ぶち抜きでの発表だったので人が多くてとても緊張しました。待機中に震えている時、スタッフの人に頑張って！的なことを言って貰えて非常に勇気をもらったことを覚えています ☺️

物量が多くて早口進行にはなりましたが、**「このライブラリのこのトリックがかっこいいでしょ！」という熱量**だったり、**「Go の中身ってこうなってるんだね！」みたいな驚き**を自分なりに共有できたかなと思います。

少しでも刺さってくれる人がいたら嬉しいなー！

ちなみに、スライドの準備でほぼ 2 轍状態でした 😇

{{< figure src="karamaru.jpg" class="center" width="400" >}}

さて、当日はスタッフとしてお手伝いもしていました。メインは今回初の試みであるワークショップの受付です。

まず、ワークショップ開催者には尊敬の意を表したいです。90 分間も人を釘付けにするコンテンツを作るって大変なことですよ。セッションよりも双方向コミュニケーションですから、退屈になられたらすぐ分かっちゃいますし。そんな中、参加者を小隊に分けて議論させたり、お菓子を用意したり、音楽を流したり、休憩を取り入れたり、、、飽きさせず楽しませる努力が随所に見られて本当に素敵でした。どのワークショップも評判がよく、来年も開催して欲しいという声が続出しているようですね 🎉

セッションもそうですけど、カンファレンスってメインコンテンツは運営ではない人（こちらも応募者）が作る訳じゃないですか。動機はどうあれ、最終的には参加者を楽しませるために努力・準備してくださった訳です。だからセッションやワークショップの後は、第一声はありがとう面白かったと声を大にして言ってあげたいなと改めて思いました。（どれも素晴らしいセッション・ワークショップでした 👏）

また、企画・会場の確保・web サイトの構築・スポンサー連携・CfP 選考など、素敵な場所を提供してくれた運営の皆様も本当にありがとうございました！おかげで楽しむことができました mm

{{< figure src="card.jpg" class="center" width="300" >}}

↑ [r-906 さんの T シャツ](https://booth.pm/ja/items/7211545?srsltid=AfmBOooNYGh9TN1gneTcU0v8WDG5r8u-uIonbXNH6RzMh2f2iKDofJhc)。可愛い。

## セッションの要約・推しポイント・話しきれなかったこと

当日はスキップした部分もあったので、発表資料を振り返りたいと思います。

### 1. モックとは何か？

発表ではスキップした部分です。

Go を始めてまもない頃、uber-go/mock(当時は golang/mock か)がどのように動作しているのか理解できなかった記憶があります。何これ魔法？みたいな。急に`EXPECT~`とか`モックオブジェクトで差し替え~`とか言われても分からない！みたいな。

なので発表資料では、そもそもモックってどういうことだっけ？という話から、既存モックライブラリの内部実装を自作してイメージするところまで解説しています。

**昔の自分に届け！**

{{< figure src="slide1-1.jpg" class="center" width="500" >}}
{{< figure src="slide1-2.jpg" class="center" width="500" >}}

cf. [speackerdeck.com - #1 モックとは？](https://speakerdeck.com/karamaru/mockio-go-dynodexue-bumetapuroguramingu?slide=11)

### 1. tips: 既存モックライブラリの設計比較について

発表でも触れる予定だった、上記で紹介したモックライブラリ比較の社内勉強会資料です。

主に型安全性や、コード生成の有無、モックオブジェクト or スタブなどの観点から、存在するモックライブラリの設計比較を確認できます。

[Comparison of golang mocking libraries](https://gist.github.com/maratori/8772fe158ff705ca543a0620863977c2)という gist も非常によくまとめられているので、併せてご確認ください。

**実務で使うモックライブラリを選定したい人に届け！**

{{< slide src="https://speakerdeck.com/player/251af6678ddb4ab191bfafb3f0df969f" >}}

### 2. mockio の使い方・美しすぎる設計

発表ではスキップした部分です。

テストファイルにおける 1 度目のメソッド実行を zero 値で返す様に振る舞いを指定することで、その後の返り値の期待を型安全に行うテクニックが紹介されています。
よくある`EXPEXT`的な記法が存在せず、メソッドをそのまま期待値の指定に使える設計...かっこいい 👏

発表で設計の素晴らしさに触れた一方、欠点にも触れる必要がありますね。発表では濁しましたが、reflect を多用しているためパフォーマンスが悪いことや、（可変長 generics みたいな言語機能がないため）返り値が 3 つ以上になると型安全性が消失することなど、mockio にはいくつかのデメリットが存在しています。

Ask the Speaker で聞かれた「Go のアップデートでアロケーション方法など変わったら壊れるのでは？」という疑問は最もで、[ドキュメント](https://ovechkin-dm.github.io/go-dyno/latest/internals/)にそこらへんのリスクと、対応について記述されています。
Go のアップデートタイミングさえ気遣えばすぐに壊れることは防げますから、ハックしていることを理由に実用性が低いと断定できるわけではなさそうです。

> Golang ABI ... we are confident that any necessary adjustments can be made with minimal disruption to the library's functionality.

実務で使えるほどの信頼性があるのか？と問われると即答はできませんが、そんなことは**かっこいいからどうでもいいのです**！👏

{{< figure src="slide2-1.jpg" class="center" width="500" >}}

cf. [speackerdeck.com - #2 mockio の使い方・美しすぎる設計](https://speakerdeck.com/karamaru/mockio-go-dynodexue-bumetapuroguramingu?slide=16)

### 3. go-dyno によるメタプログラミング

発表の大部分を使った部分です。

本来禁止されているメソッドの動的振る舞い指定に対して、 go-dyno がどのようにそれを実現しているのかを解説しました。メソッドをアセンブリで静的に記述し、そこから`reflect.MakeFunc`で振る舞いを指定した関数を呼び出し、引数はメソッド呼び出し時のものをハックして共有するという話でしたね。

特に補足はないのですが、やはり **Tips パート**が個人的にお気に入りの部分です ☺️

**このセッションの目的**は、mockio/go-dyno の設計を伝えることの他に、それを理解するために必要な **Go で使う基盤系の知識を図解して共有すること**にありました。

メソッドって Go でどう管理されてる？レジスタとスタックの違いは？アセンブリってどう書く？関数とメソッドの違いは？スタックフレームの構造は？引数の受け渡しってどうやる？...
これらを **Tips パート（タイトル赤線のスライド）として紹介**しました。

モックについて興味がない人にも学びがある発表になれてたらいいなー！

あと、go-dyno の図解は結構頑張りました！難しい題材だったから上手く解説できたか不安だけど、かっこよさが伝わったらいいな ☁️

特にクライマックスの、**レジスタとスタックの中身をハックして引数を覗き見するトリック**について、すごそう感が伝わると嬉しいです 🔥

{{< figure src="slide3-1.jpg" class="center" width="500" >}}

cf. [speackerdeck.com - #3 go-dyno によるメタプログラミング](https://speakerdeck.com/karamaru/mockio-go-dynodexue-bumetapuroguramingu?slide=25)

## 終わりに

発表もスタッフも本当に楽しかった！

思えば[Go Conference 2024](https://karamaru-alpha.com/posts/gocon2024/) が人生初登壇で、それからよく外部登壇するようになって、友達が増えたりしたんだっけ。
カンファレンスは難しいことが知れて面白いし、例え分からくてもこの技術が好きなんだろうなって熱量に触れるだけで楽しいですよね！

全ての運営者・登壇者・ワークショップホスト・参加者に感謝して、ブログを締めたいと思います。

いつか VTuber になって技術的な配信をしたいなーー！
