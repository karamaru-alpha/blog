---
title: "自作言語をwasm変換するコンパイラをrustで作った"
date: 2023-03-01T05:25:00+09:00
---

rust/wasmに入門するために、自作言語をwasmバイナリにマッピングするコンパイラを作りました。

機能は限定的で、四則演算を関数としてexportし、wasmで呼び出すまでを実装しています。

どちらも初めて1ヶ月程度の知識なので細かい点はご容赦くださいmm

<!--more-->

https://github.com/karamaru-alpha/wasm-compiler-in-rust

## 発表資料

https://speakerdeck.com/karamaru/zi-zuo-yan-yu-worustdewasmnikonpairusuru

{{< slide src="https://docs.google.com/presentation/d/e/2PACX-1vSCMNfNYYoCSB1gIV3HIcaXWtdf_1pnIEHUhCN11PIoe4xAj4j7h-IGtjeKipEXiw/embed?start=false&loop=false&delayms=3000" >}}

## 感想

ライブラリやコンパイラ基盤を用いずにバイナリを読み書きしたのは初めてだったので楽しかったです。

今後もrust/wasm追っていきたいです🍼
