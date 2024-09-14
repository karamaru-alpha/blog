---
title: "ぼくのかんがえたさいきょうのもっくらいぶらり！"
date: 2024-07-13T13:56:17+09:00
---


GoConnect#1(@サイボウズさん)でLTしてきました。
いくつかの既存モックライブラリの概観と、徹夜で作った自作モックライブラリの紹介をしました。

<!--more-->

## 雑談

今までの外部登壇や社内勉強会の発表資料(の中でも載せられるもの)をSpeakerDeckにあげました👏

- [GoConference2024 - 詳解 “Fixing For Loops in Go 1.22” / 自作linterをgolangci-lintへコントリビュートした話](https://speakerdeck.com/karamaru/xiang-jie-fixing-for-loops-in-go-1-dot-22-zi-zuo-linterwogolangci-linthekontoribiyutositahua)
- [GoConnect#1 - モックライブラリ"mocka"](https://speakerdeck.com/karamaru/bokunokangaetasaikiyounomotukuraiburari)
- [社内勉強会 - 独自言語をRustを使ってwasmバイナリに変換する](https://speakerdeck.com/karamaru/zi-zuo-yan-yu-worustdewasmnikonpairusuru)
- [社内勉強会 - zshの起動速度を300ms->20msにしたりした](https://speakerdeck.com/karamaru/zsh-star-dan-desukutotupuhuan-jing-zui-su-womu-zhi-site)
- [社内勉強会 - 対isuconツール"isumaru"作った](https://speakerdeck.com/karamaru/dui-isuconmetorikusuturu-isumaruzuo-tuta)


誰かに刺さるといいなー！

## 要約

以下既存モックライブラリを褒めて、哲学の違いを探しました。
- [github.com/uber-go/mock](github.com/uber-go/mock)
- [github.com/vektra/mockery](github.com/vektra/mockery)
- [github.com/matryer/moq](github.com/matryer/moq)
- [github.com/gojuno/minimock](github.com/gojuno/minimock)
- [github.com/bytedance/mockey](github.com/bytedance/mockey)

上記のメリットをかき集めた徹夜で作った自作ライブラリ[mocka](https://github.com/karamaru-alpha/mocka)の紹介をしました。
- TypeSafe
- 引数とそれに応じた振る舞いを定義するモックオブジェクトを作れる
- 実装をまるまる入れ替えるスタブオブジェクトとしても扱える (Stabilize)
- yamlでモック対象を記述できる
- 複雑なマッチャーが存在しない (ConditionalExpect)
- メソッドチェーンでループしない (expect->times->return固定)
- 外部パッケージも簡単にモック可能
- **現状めっちゃToy！！！**

## 感想

既存ライブラリについて
- どれも推せるポイントがある👏
- 王道gomockをさらに使いやすくしたmockeryやminimockの派生は歴史がみれて面白かった。yaml管理や型安全のリメイク最高。
- moqはSpy/Stub寄りの方針で、gomock勢とは一線を画す思想が楽しかった。ライブラリが薄いのも人気の理由が伺える。
- モンキーパッチを使用するmockeyもI/Fを必要としないという点で飛び道具感あって見る分には面白くてよかった。

mockaについて
- モックライブラリの自作は好きなところ詰め込めるし動いたら楽しい！
- 一方でちゃんと作り切るとなると結構大変そう(それはそう)。けど1日の徹夜だけ(はちょっと盛ってるけどそれぐらい)で最低限動かしただけでも自分を褒めたいです😭
- いつか機能完備させたいーー！(I/F埋め込みとか対応していないので...)

2回目の外部登壇について
- GoConに続いて外部登壇は2回目だった！
- 10m枠だったんだけど20m以上話してしまった...話すの楽しいんですよね🥺 今度から気を付けますorz
- 今後も(Goに限らず)興味の沸いたものについてはいっぱい発表していきたい！

## スライド

https://speakerdeck.com/karamaru/bokunokangaetasaikiyounomotukuraiburari

## 関連リンク

- https://github.com/karamaru-alpha/mocka
- https://gotalk.connpass.com/event/322547/
