---
title: "自作言語をwasm変換するコンパイラをrustで作った"
date: 2023-03-01T05:25:00+09:00
---

rust/wasmに入門するために、自作言語をwasmバイナリにマッピングするコンパイラを作りました。

<!--more-->

https://github.com/karamaru-alpha/wasm-compiler-in-rust

## 発表資料

{{< slide src="https://docs.google.com/presentation/d/e/2PACX-1vSCMNfNYYoCSB1gIV3HIcaXWtdf_1pnIEHUhCN11PIoe4xAj4j7h-IGtjeKipEXiw/embed?start=false&loop=false&delayms=3000" >}}

## 感想

ライブラリやコンパイラ基盤を用いずにバイナリを読み書きしたのは初めてだったので楽しかったです。

今後もrust/wasm追っていきたいです🍼
