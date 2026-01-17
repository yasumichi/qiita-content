---
title: GitBucket Markdown Enhanced Plugin にプレゼンテーション機能を追加しました(試験的)
tags:
  - JavaScript
  - Scala
  - GitBucket
private: false
updated_at: '2026-01-17T10:42:21+09:00'
id: 75bbec3f02e9e4b24bef
organization_url_name: null
slide: false
ignorePublish: false
---
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

## GitBucket Markdown Enhanced Plugin 0.9.0 時点のバグと制限事項

### マークダウンファイルでなくてもプレゼンテーションを開くアイコンが表示される

クリックすると別ウィンドウが開きますが、当然、よくわからない内容が表示されます。

以下のコミットで修正していますので次バージョンで修正される予定です。

- [The open presentation icon will no longer be displayed for non-Markdo… · yasumichi/gitbucket-markdown-enhanced@227bd4d](https://github.com/yasumichi/gitbucket-markdown-enhanced/commit/227bd4d399d0cda5170c3d7ce4f212d042c7e0c2)

### リポジトリトップページからのプレゼンテーション表示に失敗する

回避手段としては、一旦、README.md を開いてからプレゼンテーションを開くアイコンをクリックしてください。

以下のコミットで修正していますので次バージョンで修正される予定です。

- [Fixed an issue where presentations could not be displayed from the re… · yasumichi/gitbucket-markdown-enhanced@d48981a](https://github.com/yasumichi/gitbucket-markdown-enhanced/commit/d48981a1799d2eb3a5cf81e6ad346d9379ece5b9)

### 対応している拡張記法はまだ極一部

現状で対応している拡張記法は、以下のとおりです。

- [KaTeX](https://katex.org/) (コードブロックをの除く)
- [mermaid](https://mermaid.js.org/)
- [WaveDrom](https://wavedrom.com/)

## ToDo

- [ ] 相対パスで指定された画像の表示
- [ ] 未対応の拡張記法への対応
- [ ] [reveal.js](https://revealjs.com/) のテーマをリポジトリごとに設定可能にする

## ソースの変更点

### コントローラーの追加

[src/main/scala/io/github/yasumichi/gme/controller/PresentationController.scala](https://github.com/yasumichi/gitbucket-markdown-enhanced/blob/v0.9.0/src/main/scala/io/github/yasumichi/gme/controller/PresentationController.scala) を追加しました。

`/:owner/:repository/presentation/*` のような URL へアクセスすると次項で紹介する Twirl テンプレートから、HTML を返す簡単なお仕事をしています。

なお、コントローラーを有効にするために [src/main/scala/Plugin.scala](https://github.com/yasumichi/gitbucket-markdown-enhanced/blob/v0.9.0/src/main/scala/Plugin.scala#L136) に以下のコードを追加しています。

```scala:src/main/scala/Plugin.scala
  override val controllers: Seq[(String, ControllerBase)] = Seq(
    "/*" -> new PresentationController()
  )
```

### Twirl テンプレートの追加

[src/main/twirl/gme/presentation.scala.html](https://github.com/yasumichi/gitbucket-markdown-enhanced/blob/v0.9.0/src/main/twirl/gme/presentation.scala.html) を追加しました。

[reveal.js](https://revealjs.com/) を動かすために必要なスタイルシート、JavaScript の他、拡張記法に必要な JavaScript を読み込むようにしています。

マークダウンソースは、以下のようにサーバーから取得するようにしています。

```html
<section
  data-markdown="@context.baseUrl/@repository.owner/@repository.name/raw/@id/@filePath"
>
```

:::note alert
Twirl テンプレートに直接、JavaScript を記述することもできますが、コールバック関数指定の際に `() => {}` のような記法を使うとコンパイルに失敗します。

可能であれば、JavaScript は、別ファイルにするのがお勧めです。
:::

### プレゼンテーション制御用の JavaScript の追加

[src/main/resources/gme/assets/presentation.js](https://github.com/yasumichi/gitbucket-markdown-enhanced/blob/v0.9.0/src/main/resources/gme/assets/presentation.js) を追加しました。

#### reveal.js の準備完了は、Reveal.initialize() に then() で捕捉する方が良さげ

[Events | reveal.js](https://revealjs.com/events/) で reveal.js の準備完了を捕捉する方法が、2種類紹介されていますが、`Reveal.on('ready', callbackFunction);` の方法はうまくいきませんでした。(私の理解が足りないだけかも。)

```js:src/main/resources/gme/assets/presentation.js
    Reveal.initialize({
        hash: true,

        // Learn about plugins: https://revealjs.com/plugins/
        plugins: [ RevealMarkdown, RevealHighlight, RevealNotes, RevealMath.KaTeX ],
        markdown: {
            gfm: true,
            renderer: renderer
        }
    }).then(revealReady);
```

#### mermaid はノードが非表示の状態で描画するとちゃんと描画されないのでスライド表示のタイミングで処理する

タイトルの通りですが、`mermaid.run()` 呼び出し時に対象ノードを限定する方法があったのでうまく活用します。

以下の処理を reveal.js の `ready` イベントと `slidechanged` イベントから呼び出すようにしています。

```js:src/main/resources/gme/assets/presentation.js
    const processMermaid = function (currentSlide) {
        let nodes = currentSlide.querySelectorAll('.mermaid');
        if (nodes.length > 0) {
            mermaid.run({nodes: nodes}).then(() => Reveal.layout());
        }
    };
```

また、スライドが表示された後に処理される関係上、縦位置のレイアウトが中央揃えにならないので mermaid の描画完了後に `Reveal.layout()` を呼び出すようにしています。

## 私が欲しかった理想の reveal.js + マークダウンのプレゼンテーション環境をめざして

前々から、reveal.js を軽易にかつインターネット接続なしで利用できる環境が欲しかったのですが、目標に一歩近づいたと思っています。

似たような環境があることは承知していますが、しっくり来るものがなかったのです。(割と [GROWI](https://growi.org/ja/) は理想に近いけど。)
