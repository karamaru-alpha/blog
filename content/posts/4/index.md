---
title: "自作言語をwasm変換するコンパイラをrustで作った"
date: 2023-03-01T05:25:00+09:00
---

rust/wasmに入門するために、自作言語をwasmバイナリにマッピングするコンパイラを作りました。
https://github.com/karamaru-alpha/wasm-compiler-in-rust

<!--more-->

## 発表資料

{{< slide src="https://docs.google.com/presentation/d/e/2PACX-1vRey87iD7F3VUToThId4xfPzCWoY4yRj1ssO9DjIu_hOX2vEh-MM7Mn1YUc6ZIQCA/embed?start=false&loop=false&delayms=3000" >}}


## 感想

ライブラリやコンパイラ基盤を用いずにバイナリを読み書きしたのは初めてだったので楽しかったです。

今後もrust/wasm追っていきたいです🍼
