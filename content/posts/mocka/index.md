---
title: "ぼくのかんがえたさいきょうのもっくらいぶらり！"
date: 2024-07-16T13:56:17+09:00
---


GoConnect#1(@サイボウズさん)でLTしてきました。
いくつかの既存モックライブラリの概観と、徹夜で作った自作モックライブラリの紹介をしました。

<!--more-->

## 雑談

今までの外部登壇や社内勉強会の発表資料(の中でも載せられるもの)をSpeakerDeckにあげました👏
誰かに刺さるといいなあ☁️

- [GoConference2024 - 詳解 “Fixing For Loops in Go 1.22” / 自作linterをgolangci-lintへコントリビュートした話](https://speakerdeck.com/karamaru/xiang-jie-fixing-for-loops-in-go-1-dot-22-zi-zuo-linterwogolangci-linthekontoribiyutositahua)
- [GoConnect#1 - モックライブラリ"mocka"](https://speakerdeck.com/karamaru/bokunokangaetasaikiyounomotukuraiburari)
- [社内勉強会 - 独自言語をRustを使ってwasmバイナリに変換する](https://speakerdeck.com/karamaru/zi-zuo-yan-yu-worustdewasmnikonpairusuru)
- [社内勉強会 - zshの起動速度を300ms->20msにしたりした](https://speakerdeck.com/karamaru/zsh-star-dan-desukutotupuhuan-jing-zui-su-womu-zhi-site)
- [社内勉強会 - 対isuconツール"isumaru"作った](https://speakerdeck.com/karamaru/dui-isuconmetorikusuturu-isumaruzuo-tuta)

## 要約

以下既存モックライブラリを褒める。哲学の違いを探す。
- [github.com/uber-go/mock](github.com/uber-go/mock)
- [github.com/vektra/mockery](github.com/vektra/mockery)
- [github.com/matryer/moq](github.com/matryer/moq)
- [github.com/gojuno/minimock](github.com/gojuno/minimock)
- [github.com/bytedance/mockey](github.com/bytedance/mockey)

上記のメリットをかき集めた徹夜で作った自作ライブラリ[mocka](https://github.com/karamaru-alpha/mocka)の紹介
- TypeSafe
- 引数とそれに応じた振る舞いを定義するモックオブジェクトを作れる
- 実装をまるまる入れ替えるスタブオブジェクトとしても扱える (Stabilize)
- yamlでモック対象を記述できる
- 複雑なマッチャーが存在しない (ConditionalExpect)
- メソッドチェーンでループしない (expect->times->return固定)
- 外部パッケージも簡単にモック可能
- **現状めっちゃToyです**

## 感想

既存ライブラリについて
- どれも推せるポイントがある👏
- gomock->mockery/minimockの派生や、moqやmockeyなど別手法など、様々な思想があって面白かった。

mockaについて
- モックライブラリの自作は(機能的にガバガバでも)動いたら楽しい！！
- 一方でちゃんと作り切るとなると結構大変そう(それはそう)。OSS作者リスペクト。
- いつか機能完備させたいなーー！(I/F埋め込みとか対応していないので...)

2回目の外部登壇について
- GoConに続いて外部登壇は2回目だった！
- 10m枠だったんだけど20m以上話してしまった...話すの楽しいんですよね🥺 今度から気を付けますorz
- 今後も(Goに限らず)興味の沸いたものについてはいっぱい発表していきたい！

## スライド

https://speakerdeck.com/karamaru/bokunokangaetasaikiyounomotukuraiburari

## 関連資料

- https://github.com/karamaru-alpha/mocka
- https://gotalk.connpass.com/event/322547/
