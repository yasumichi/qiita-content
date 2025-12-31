---
title: GitBucket Markdown Enhanced Plugin に vega および vega-lite サポートを追加しました(とりまJSONのみ)
tags:
  - JavaScript
  - Scala
  - GitBucket
  - vega
  - vega-lite
private: false
updated_at: '2025-12-31T17:32:27+09:00'
id: fe740aeeeafd1e25a2a7
organization_url_name: null
slide: false
ignorePublish: false
---
## 大晦日に何をやっているのか

言わないで…

言わないで…

さよならは…(これ以上は、●ASRAC に怒られる)

## 概要

[GitBucket](https://gitbucket.github.io/)用のプラグイン [GitBucket Markdown Enhanced Plugin](https://github.com/yasumichi/gitbucket-markdown-enhanced)を開発しています。

GitBucket Markdown Enhanced Plugin は、GitBucket 標準のマークダウンレンダリングエンジンを置き換えるプラグインです。

目標は、[Visual Studio Code](https://code.visualstudio.com/) の [Markdown Preview Enhanced](https://shd101wyy.github.io/markdown-preview-enhanced/#/) 向けの markdown ファイルを軽易に Web で共有できる環境です。

しばらく別のプラグインの開発をしていたため、久しぶりの更新です。

- [GitBucket Flexible Gantt Plugin](https://github.com/yasumichi/gitbucket-flexible-gantt-plugin) [関連記事](https://qiita.com/yasumichi/items/431c5f411addfe39313d)
- [GitBucket Commit Graphs Chart.js Plugin](https://github.com/yasumichi/gitbucket-commitgraphs-chartjs-plugin) - [関連記事](https://zenn.dev/yasumichi/articles/50c8e7d5360aca)

今回は、Markdown Preview Enhanced が対応している [Vega and Vega-lite](https://shd101wyy.github.io/markdown-preview-enhanced/#/diagrams?id=vega-and-vega-lite) サポートを追加した話です。

:::note info
今のところ、Markdown Preview Enhanced が対応している YAML[^yaml] 形式には対応していません。JSON[^json] のみ対応しています。
:::

[^yaml]: YAML Ain't Markup Language (← Yet Another Markup Language) - [YAML Ain’t Markup Language (YAML™) revision 1.2.2](https://yaml.org/spec/1.2.2/)

[^json]: JavaScript Object Notation - [RFC 8259: The JavaScript Object Notation (JSON) Data Interchange Format](https://www.rfc-editor.org/rfc/rfc8259)

## 前回の記事

https://qiita.com/yasumichi/items/891578c48573de80b412

## 機能の説明

`vega` あるいは `vega-lite` 用のコードブロックに専用の JSON を書くとグラフが表示されます。

![vega-lite.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/36738/a1104883-2f93-408d-a37e-3e8499316ff9.png)

以下のコードブロックが、上の画像に変換されます。

<details><summary>サンプルコード</summary>
<pre>
```vega-lite
{
  "$schema": "https://vega.github.io/schema/vega-lite/v6.json",
  "description": "A simple bar chart with embedded data.",
  "data": {
    "values": [
      {
        "a": "A",
        "b": 28
      },
      {
        "a": "B",
        "b": 55
      },
      {
        "a": "C",
        "b": 43
      },
      {
        "a": "D",
        "b": 91
      },
      {
        "a": "E",
        "b": 81
      },
      {
        "a": "F",
        "b": 53
      },
      {
        "a": "G",
        "b": 19
      },
      {
        "a": "H",
        "b": 87
      },
      {
        "a": "I",
        "b": 52
      }
    ]
  },
  "mark": "bar",
  "encoding": {
    "x": {
      "field": "a",
      "type": "ordinal"
    },
    "y": {
      "field": "b",
      "type": "quantitative"
    }
  }
}
```
</pre>
</details>

## vega 関連のスクリプトファイルを LICENSE とともに同梱

専用のディレクトリ [src/main/resources/gme/assets/vega](https://github.com/yasumichi/gitbucket-markdown-enhanced/tree/main/src/main/resources/gme/assets/vega)を掘り、以下のスクリプトを LICENSE とともに配置しました。

- [vega](https://github.com/vega/vega)
- [vega-lite](https://github.com/vega/vega-lite)
- [vega-embed](https://github.com/vega/vega-embed)

## Plugin クラスの javaScripts メソッドに上記ファイルを読み込むための記述を追加

[src/main/scala/Plugin.scala](https://github.com/yasumichi/gitbucket-markdown-enhanced/blob/fa7639934a348ac35434ffefa91617c733930a87/src/main/scala/Plugin.scala#L115) に以下の記述を追加しました。

```scala:src/main/scala/Plugin.scala
      |<script src="${jsPath}/vega/vega-6.2.0.js" type="text/javascript">
      |</script>
      |<script src="${jsPath}/vega/vega-lite-6.4.1.js" type="text/javascript">
      |</script>
      |<script src="${jsPath}/vega/vega-embed-7.0.2.js" type="text/javascript">
      |</script>
```

## MarkdownEnhancedNodeRenderer に処理を追加

[src/main/scala/io/github/yasumichi/gme/MarkdownEnhancedNodeRenderer.scala](https://github.com/yasumichi/gitbucket-markdown-enhanced/blob/fa7639934a348ac35434ffefa91617c733930a87/src/main/scala/io/github/yasumichi/gme/MarkdownEnhancedNodeRenderer.scala#L212) に専用のメソッドを追加しました。

```scala:src/main/scala/io/github/yasumichi/gme/MarkdownEnhancedNodeRenderer.scala
  private def renderVega(
      html: HtmlWriter,
      node: FencedCodeBlock,
      context: NodeRendererContext,
      language: String
  ): Unit = {
    html.withAttr().attr("id", s"vega-${vegaId}").tag("div")
    html.tag("/div")
    html.withAttr().attr("type", language).attr("class", "vega").attr("data-target", s"#vega-${vegaId}").tag("script")
    vegaId = vegaId + 1;
    html.rawIndentedPre(node.getContentChars())
    html.tag("/script")
  }
```

グラフを埋め込むための &lt;div&gt; タグと JSON を埋め込むための &lt;script&gt; タグを出力しています。

JavaScript として処理されないように [type](https://developer.mozilla.org/ja/docs/Web/HTML/Reference/Elements/script/type) 属性を `vage` または `vega-lite` にしています。([WaveDrom](https://wavedrom.com/) 方式)

また、グラフを埋め込むための &lt;div&gt; タグを決定するために [data-](https://developer.mozilla.org/ja/docs/Web/HTML/Reference/Global_attributes/data-*)target 属性にセレクターを付与しています。

`vegaId` はクラスの private フィールドです。

このメソッドを[コードブロックの処理を振り分けるところ](https://github.com/yasumichi/gitbucket-markdown-enhanced/blob/fa7639934a348ac35434ffefa91617c733930a87/src/main/scala/io/github/yasumichi/gme/MarkdownEnhancedNodeRenderer.scala#L96)から、呼び出しています。

## 最終的に描画を行うスクリプト

[src/main/resources/gme/assets/gme.js](https://github.com/yasumichi/gitbucket-markdown-enhanced/blob/fa7639934a348ac35434ffefa91617c733930a87/src/main/resources/gme/assets/gme.js#L26) に最終的な描画を行う関数を追加しました。

```js:src/main/resources/gme/assets/gme.js
    var renderVega = function() {
        return new Promise((resolve, reject) => {
            let vegaList = document.querySelectorAll('.vega');
            vegaList.forEach((node, index) => {
                let vegaId = node.getAttribute('data-target');
                try {
                    let vegaData = JSON.parse(node.textContent);
                    vegaEmbed(vegaId, vegaData);
                } catch (error) {
                    let node = document.querySelector(vegaId)
                    if (node) {
                        node.textContent = error.message;
                    }
                }
            });
            resolve();
        });
    }
```

一応、[Promise](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Promise) を使って非同期処理にしています。

主な処理は以下の通りです。

- [class](https://developer.mozilla.org/ja/docs/Web/HTML/Reference/Global_attributes/class) 属性が `vega` であるノードを探し、[forEach](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Array/forEach) で走査
  - ノードの [data-](https://developer.mozilla.org/ja/docs/Web/HTML/Reference/Global_attributes/data-*)target 属性の値を取得
  - ノードの [textContent](https://developer.mozilla.org/ja/docs/Web/API/Node/textContent) プロパティを [JSON.parse()](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/JSON/parse) でオブジェクトに変換
  - [vegaEmbed](https://vega.github.io/vega-lite/usage/embed.html) でターゲットノードにグラフを埋め込み

この関数を以下のそれぞれで呼び出しています。

- [DOMContentLoaded](https://developer.mozilla.org/ja/docs/Web/API/Document/DOMContentLoaded_event) が発火した時(閲覧時)
- リポジトリビューアーでファイル編集時にプレビューされた時
- issue コメントなどを編集時にプレビューされた時

## 最後までお読みいただきありがとうございました。

今年も最後までお付き合いいただきありがとうございました。

それでは、よいお年を。
