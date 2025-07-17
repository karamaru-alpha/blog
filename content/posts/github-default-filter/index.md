---
title: "【GitHub】PR一覧画面でフィルターのデフォルト値を設定したい！"
date: 2025-07-17T00:12:48+09:00
---

「is:pr is:open」に追加して「draft:false」とかをデフォルトでつけたい！

<!--more-->

## 先に結論

Arc Boosts で js を仕込んで書き換える。

@see: https://github.com/orgs/community/discussions/51751#discussioncomment-10602945

## 動機

GitHub の PR/Issue 検索窓のフィルター、使ってますか！

僕は「created:YYYY-MM-DD..YYYY-MM-DD」構文が一番好きです ☺️

cf. [Filtering and searching issues and pull requests](https://docs.github.com/en/issues/tracking-your-work-with-issues/using-issues/filtering-and-searching-issues-and-pull-requests)

{{< figure src="search.png" class="center" width="600" >}}

さて、大人数で開発してると DraftPR で画面がいっぱいになってうんざりすること、ありますよね。

そんな時は検索窓に「draft:false」を追加するんですけど、毎回指定するのもナンセンスというか、遷移したらデフォルトでこのフィルターがついて欲しいと思ったんですよね！

一方、現在はデフォルトのフィルターを保存する機能は GitHub に備わっておらず、ユーザー側で頑張る必要があります mm

手順を以下に示します。

## 手順

Arc ユーザーであれば[Arc Boosts](https://arcboosts.com/boosts)という機能でサイトに script を仕込むことができます。以下、Arc 前提で進めます。（Arc ユーザーでない場合は[Tampermonkey](https://chromewebstore.google.com/detail/tampermonkey/dhdgffkkebhmkfjojejmpbldmpobfkfo?hl=en)などのユーザースクリプトプラグインで対応する感じになります）

まず、ブラウザ左下のメニューから「New Boost」を選択します。

{{< figure src="image.png" class="center" width="300" >}}

するとこんなフォントや色彩を変えられるパレットが出てきてきます。
見た目変えられるだけでちょっと楽しい。

{{< figure src="sample.png" class="center" width="400" >}}

さて、「Code」を押すとユーザースクリプトを仕込める画面が出てくるので、以下コードをぺたっと貼り付けましょう。

これで、初回遷移時（フィルターが URL に存在しない場合）にラベルが「is:pr is:open draft:false」になります 👏

{{< figure src="js.png" class="center" width="600" >}}

```js
const DEFAULT_FILTERS = ["is:pr", "is:open", "draft:false"];

const updateAnchor = (element) => {
  const href = element.getAttribute("href");
  if (!href || !href.startsWith("/")) return;
  const url = new URL(href, window.location.origin);
  if (!url.pathname.endsWith("/pulls")) return;

  const query = url.searchParams.get("q")?.split(" ");

  if (!query) {
    url.searchParams.set("q", DEFAULT_FILTERS.join(" "));
    element.setAttribute("href", url.href);
    return;
  }

  if (!query.includes("is:open") || query.includes("draft:false")) return;
  query.push("draft:false");
  url.searchParams.set("q", query.join(" "));
  element.setAttribute("href", url.href);
};

const updateAll = () => {
  const elements = document.querySelectorAll("a[href]");
  for (const element of elements) updateAnchor(element);
};

const observe = (mutations) => {
  for (const mutation of mutations) {
    switch (mutation.type) {
      case "childList":
        for (const node of mutation.addedNodes) {
          if (node.nodeType !== Node.ELEMENT_NODE) continue;
          if (node.tagName === "A") updateAnchor(node);
          for (const anchor of node.querySelectorAll("a[href]"))
            updateAnchor(anchor);
        }
        break;
      case "attributes":
        if (mutation.attributeName !== "href") break;
        updateAnchor(mutation.target);
        break;
      default:
    }
  }
};

const observer = new MutationObserver(observe);

observer.observe(document.documentElement, {
  childList: true,
  subtree: true,
  attributes: true,
  attributeFilter: ["href"],
});

document.addEventListener("DOMContentLoaded", updateAll);
```

後からフィルターを操作した場合にデフォルト値に強制的に戻されるようなこともなくいい感じです。素敵 👏

## 終わりに

全部冒頭に貼った issue に書いてあることなんですけど、プチ感動したので書いてみました。

こういうのでいいんだよ。こういうので。。。！
