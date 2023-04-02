---
title: "wasmプラグインの作成/呼び出しを言語跨ぎでできるExtismを試す"
date: 2023-04-01T17:06:45+09:00
---

先日は[WASM I/O](https://wasmio.tech)がありましたね。
中でも、wasmプラグインの作成/呼び出しを言語跨ぎでできる[Extism](https://extism.org)が気になったので試してみました。

<!--more-->

https://github.com/karamaru-alpha/extism-trial

主なできることとしては以下です
- PDKを用いて、wasmプラグインを作成できる
- SDKを用いて、wasmプラグインを呼び出せる
  - 呼び出し元から関数を渡してwasm内で実行できる

まだまだドキュメントが乏しく、現在の仕様で動くrepositoryがなかったので参考になれば幸いですmm

### 一言

安全で高速、言語を超えたポータブルな実装ができるのがブラウザ外wasmの利点です。

言語跨ぎの呼び出しを簡易化したことで、 wasmモジュールを積み木のように組み合わせるだけで処理が完結してしまう世界に一歩近づいた感じでワクワクしますね。

(実際に、wasm版dockerhubである[wapm](https://wapm.io/)や、wasmモジュールをChart的に管理してpipeline的に組み合わせる[Scale](https://scale.sh/)なども出てるし)
