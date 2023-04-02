---
title: "wasmプラグインの作成/呼び出しを多言語でできるExtismを試す"
date: 2023-04-01T17:06:45+09:00
---

先日はWASM I/Oがありましたね。
中でも、wasmプラグインの作成/呼び出しを言語跨ぎでできるExtismが気になったので試してみました。

<!--more-->

![img.png](img.png)

https://extism.org/

主なできることとしては以下です
- PDKを用いて、wasmプラグインを作成できる
- SDKを用いて、wasmプラグインを呼び出せる
  - 呼び出し元から関数を渡してwasm内で実行できる

とりあえず、Goで作成したwasmプラグインをRustから呼び出すコードを書いてみました。
https://github.com/karamaru-alpha/extism-trial

まだまだドキュメントが乏しく、現在の仕様で動くrepositoryがなかったので参考になれば幸いですmm

### 一言

[WASM I/O](https://wasmio.tech/)見ないとなぁ。英語できないからながら聞きできないのが悔しいです。

安全で高速、言語を超えたポータブルな実装ができるのがブラウザ外wasmの利点です。

言語跨ぎの呼び出しを簡易化したことで、 wasmモジュールを積み木のように組み合わせるだけで処理が完結してしまう世界に一歩近づいた感じでワクワクしますね。

(実際に、wasm版dockerhubである[wapm](https://wapm.io/)や、wasmモジュールをChart的に管理してpipeline的に組み合わせる[Scale](https://scale.sh/)なども出てるし)
