---
title: GitBucket Markdown Enhanced Plugin でシンタックスハイライトを有効にする
tags:
  - JavaScript
  - Scala
  - Markdown
  - GitBucket
private: false
updated_at: '2025-12-21T16:48:06+09:00'
id: aa52da49d52b89ecfde6
organization_url_name: null
slide: false
ignorePublish: false
---
## 概要

[GitBucket](https://gitbucket.github.io/)用のプラグイン [GitBucket Markdown Enhanced Plugin](https://github.com/yasumichi/gitbucket-markdown-enhanced)を開発しています。

GitBucket Markdown Enhanced Plugin は、GitBucket 標準のマークダウンレンダリングエンジンを置き換えるプラグインです。

目標は、[Visual Studio Code](https://code.visualstudio.com/) の [Markdown Preview Enhanced](https://shd101wyy.github.io/markdown-preview-enhanced/#/) 向けの markdown ファイルを軽易に Web で共有できる環境です。

今回は、GitBucket の標準機能に搭載されているシンタックスハイライトを有効にします。

## 前回の記事

https://qiita.com/yasumichi/items/d4b0cf525470a98e86dc

## flexmark-java のコードハイライトに対する立場を検索してみた

度々、Issue が投稿されるようですが、代表的な issue を挙げます。

[Is the library support code highlight? · Issue #183 · vsch/flexmark-java](https://github.com/vsch/flexmark-java/issues/183)

> No, this library cannot highlight your code (jet?). But you could use something like prism.js if your app is for web.

flexmark-java としては、シンタックスハイライトを行えないが、[prism.js](https://prismjs.com/) のようなものを使えば実現できるよと。

GitBucket では、シンタックスハイライトに [highlight.js](https://highlightjs.org/) を使用しており、マークダウンレンダラーを差し替えても highlight.js に関連するスクリプトが読み込まれるようになっています。

クライアント負荷やリポジトリビューアーのファイルビューとの整合を考えると prism.js を使うよりも既存機能を使った方が良さそうです。

## 生成されるコードブロックの違い

比較に使ったマークダウンは、次のとおりです。

<pre>
```scala
  private def renderFencedCodeBlock(
      node: FencedCodeBlock,
      context: NodeRendererContext,
      html: HtmlWriter
  ): Unit = {
    val htmlOptions: HtmlRendererOptions = context.getHtmlOptions()
    val language: BasedSequence =
      node.getInfoDelimitedByAny(htmlOptions.languageDelimiterSet)
    val info: String = node.getInfo().toString()

    logger.debug("FencedCodeBlock getInfo: " + node.getInfo().toString())

    if (language.equals("plantuml") || language.equals("puml")) {
      renderPlantUML(html, node)
    } else if (language.equals("wavedrom")) {
      renderWaveDrom(html, node)
    } else if (language.equals("dot") || language.equals("viz")) {
      renderDot(html, node, info)
    } else {
      context.delegateRender()
    }
  }
```
</pre>

このマークダウンを元に GitBucket の標準機能が生成する HTML と GitBucket Markdown Enhanced Plugin が生成する HTML を比較してみます。

### 標準

```html
          <pre class="prettyprint lang-scala">  private def renderFencedCodeBlock(
      node: FencedCodeBlock,
      context: NodeRendererContext,
      html: HtmlWriter
  ): Unit = {
    val htmlOptions: HtmlRendererOptions = context.getHtmlOptions()
    val language: BasedSequence =
      node.getInfoDelimitedByAny(htmlOptions.languageDelimiterSet)
    val info: String = node.getInfo().toString()

    logger.debug("FencedCodeBlock getInfo: " + node.getInfo().toString())

    if (language.equals("plantuml") || language.equals("puml")) {
      renderPlantUML(html, node)
    } else if (language.equals("wavedrom")) {
      renderWaveDrom(html, node)
    } else if (language.equals("dot") || language.equals("viz")) {
      renderDot(html, node, info)
    } else {
      context.delegateRender()
    }
  }</pre>
```

### GitBucket Markdown Enhanced Plugin

```html
<pre><code class="language-scala">  private def renderFencedCodeBlock(
      node: FencedCodeBlock,
      context: NodeRendererContext,
      html: HtmlWriter
  ): Unit = {
    val htmlOptions: HtmlRendererOptions = context.getHtmlOptions()
    val language: BasedSequence =
      node.getInfoDelimitedByAny(htmlOptions.languageDelimiterSet)
    val info: String = node.getInfo().toString()

    logger.debug("FencedCodeBlock getInfo: " + node.getInfo().toString())

    if (language.equals("plantuml") || language.equals("puml")) {
      renderPlantUML(html, node)
    } else if (language.equals("wavedrom")) {
      renderWaveDrom(html, node)
    } else if (language.equals("dot") || language.equals("viz")) {
      renderDot(html, node, info)
    } else {
      context.delegateRender()
    }
  }
</code></pre>
```

### 違い

- 標準機能では prettyprint というクラスと pre 要素に直接コードが記述されている。
  - pre 要素に prettyprint というクラスとプログラミング言語を表すクラス `lang-言語名` というクラスが指定されている。
- GitBucket Markdown Enhanced Plugin では、pre 要素の子として code 要素がある。
  - pre 要素ではなく code 要素にクラスとプログラミング言語を表す `language-言語名` というクラスが指定されている。

GitBucket Markdown Enhanced Plugin 側で標準機能に合わせた出力に修正していくことにします。

## 修正ソース

[src\main\scala\io\github\yasumichi\gme\MarkdownEnhancedNodeRenderer.scala](https://github.com/yasumichi/gitbucket-markdown-enhanced/blob/07737ae6eaf4e2378026d7aff8ff7bbef206d45d/src/main/scala/io/github/yasumichi/gme/MarkdownEnhancedNodeRenderer.scala)(修正前)

今まで GitBucket Markdown Enhanced Plugin で独自の拡張を行う言語名以外は、flexmark-java の既存のレンダラーに委譲していましたが、そこも独自実装することにします。

ただ、独自で処理しない mermaid は、他のレンダラー([GitLabExtension](https://github.com/vsch/flexmark-java/wiki/Extensions#gitlab-flavoured-markdown))に委譲するようにしています。

```scala
  private def renderFencedCodeBlock(
      node: FencedCodeBlock,
      context: NodeRendererContext,
      html: HtmlWriter
  ): Unit = {
    val htmlOptions: HtmlRendererOptions = context.getHtmlOptions()
    val language: BasedSequence =
      node.getInfoDelimitedByAny(htmlOptions.languageDelimiterSet)
    val info: String = node.getInfo().toString()

    logger.debug("FencedCodeBlock getInfo: " + node.getInfo().toString())

    if (language.equals("plantuml") || language.equals("puml")) {
      renderPlantUML(html, node)
    } else if (language.equals("wavedrom")) {
      renderWaveDrom(html, node)
    } else if (language.equals("dot") || language.equals("viz")) {
      renderDot(html, node, info)
    } else if (language.equals("mermaid")) {
      context.delegateRender()
    } else {
      renderPrittyPrint(html, node, context, language.toString())
    }
  }
```

最後の else 節で呼び出している `renderPrittyPrint` メソッドは、次のとおりです。

```scala
  private def renderPrittyPrint(
      html: HtmlWriter,
      node: FencedCodeBlock,
      context: NodeRendererContext,
      language: String
  ): Unit = {
    html
      .withAttr()
      .attr("class", s"prettyprint lang-${language}")
      .tag("pre")
    html.rawIndentedPre(node.getContentChars().toString())
    html.tag("/pre")
  }
```

`html.rawIndentedPre(node.getContentChars().toString())` にたどり着くまでが苦労しました。

これまで使用してきた [HtmlWriter](https://github.com/vsch/flexmark-java/blob/bcfe84a3ab6d23d04adce3e5a0bae45c6b791d14/flexmark/src/main/java/com/vladsch/flexmark/html/HtmlWriter.java) クラス[^1] の `append` や `text` などのメソッドでは、行頭のインデントが削除されてしまうのです。

[^1]:正確には基本クラス [HtmlAppendableBase](https://github.com/vsch/flexmark-java/blob/master/flexmark-util-html/src/main/java/com/vladsch/flexmark/util/html/HtmlAppendableBase.java)

### 2025/12/11 追記

> ただ、独自で処理しない mermaid は、他のレンダラー([GitLabExtension](https://github.com/vsch/flexmark-java/wiki/Extensions#gitlab-flavoured-markdown))に委譲するようにしています。

math ブロックも [GitLabExtension](https://github.com/vsch/flexmark-java/wiki/Extensions#gitlab-flavoured-markdown) に委譲する必要がありました。最新版で修正しています。

## スクリーンショット

ここまでの修正でスクリーンショットのとおり、GitBucket 標準のシンタックスハイライトが適用されるようになりました。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/36738/29579836-86b8-4590-afc1-d77fa7f13338.png)

## 入手先

[Releases · yasumichi/gitbucket-markdown-enhanced](https://github.com/yasumichi/gitbucket-markdown-enhanced/releases) から、最新バージョンを入手できます。
