---
title: "対isuconメトリクスツールisumaru作った"
date: 2023-10-02T00:12:48+09:00
---

対isuconメトリクスツール"isumaru"作りました。

<!--more-->

https://github.com/karamaru-alpha/isumaru

## 機能
複数サーバーに対して、以下の解析を自動化します。
- アクセスログの解析
- スロークエリログの解析
- fgprof(pprofの拡張。on cpu以外のメトリクスも取れる)

## 使用技術
- [go](https://github.com/golang/go)
- [svelte](https://github.com/sveltejs/svelte)
- [ansible](https://github.com/ansible/ansible)
- [alp](https://github.com/tkuchiki/alp)
- [slp](https://github.com/tkuchiki/slp)
- [fgprof](https://github.com/felixge/fgprof)


{{< slide src="https://speakerdeck.com/player/2c05d73c7cd5493993d2ad89bdb27f6e" >}}

昨年のisucon優勝チームnarusejun様を参考にさせていただきましたmm
