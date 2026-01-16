---
title: GitBucket Markdown Enhanced Plugin にプレゼンテーション機能を追加しました(試験的)
tags:
  - JavaScript
  - Scala
  - GitBucket
private: false
updated_at: '2026-01-16T22:35:12+09:00'
id: 75bbec3f02e9e4b24bef
organization_url_name: null
slide: false
ignorePublish: false
---
## まだ書きかけです

:::note info
今後、機能制限に関する説明、ソース修正内容を追加予定です。
:::

## 概要

[GitBucket](https://gitbucket.github.io/)用のプラグイン [GitBucket Markdown Enhanced Plugin](https://github.com/yasumichi/gitbucket-markdown-enhanced)を開発しています。

GitBucket Markdown Enhanced Plugin は、GitBucket 標準のマークダウンレンダリングエンジンを置き換えるプラグインです。

目標は、[Visual Studio Code](https://code.visualstudio.com/) の [Markdown Preview Enhanced](https://shd101wyy.github.io/markdown-preview-enhanced/#/) 向けの markdown ファイルを軽易に Web で共有できる環境です。

[GitBucket Markdown Enhanced Plugin 0.9.0](https://github.com/yasumichi/gitbucket-markdown-enhanced/releases/tag/v0.9.0) で試験的に [reveal.js](https://revealjs.com/) によるプレゼンテーション機能を追加しましたので紹介します。

## 前回の記事

https://qiita.com/yasumichi/items/b45b7f98011cb28802ff

## プレゼンテーション機能の使い方

リポジトリのファイルビューアーにて、ファイルの内容の右上に ![](https://github.com/yasumichi/gitbucket-markdown-enhanced/raw/main/images/octicon-zap.png) のようなアイコンが表示されます。

このアイコンをクリックすると別ウィンドウでプレゼンテーションが表示されます。

例として、以下のようなマークダウンのファイルを用意します。

<pre>
### GitBucket Markdown Enhanced Plugin

### Presentation function

---

## mermaid

```mermaid
graph TD;
  A-->B;
  A-->C;
  B-->D;
  C-->D;
```

---

## WaveDrom

```wavedrom
{ signal : [
  { name: "clk",  wave: "p......" },
  { name: "bus",  wave: "x.34.5x",   data: "head body tail" },
  { name: "wire", wave: "0.1..0." },
]}
```

---

## code block

```js
function add(x, y) {
  return x + y;
}
```
</pre>

このファイルでプレゼンテーションを開始すると以下のようなスライドが表示されます。

![](https://raw.githubusercontent.com/yasumichi/gitbucket-markdown-enhanced/v0.9.0/images/slide1.png)

![](https://raw.githubusercontent.com/yasumichi/gitbucket-markdown-enhanced/v0.9.0/images/slide2.png)

![](https://raw.githubusercontent.com/yasumichi/gitbucket-markdown-enhanced/v0.9.0/images/slide3.png)

![](https://raw.githubusercontent.com/yasumichi/gitbucket-markdown-enhanced/v0.9.0/images/slide4.png)
