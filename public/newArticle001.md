---
title: PlantUML の SourceStringReader.generateImage() は deprecated だってさ
tags:
  - ''
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---
## 概要

[GitBucket](https://gitbucket.github.io/)用のプラグイン [GitBucket Markdown Enhanced Plugin](https://github.com/yasumichi/gitbucket-markdown-enhanced)を開発しています。

GitBucket Markdown Enhanced Plugin は、GitBucket 標準のマークダウンレンダリングエンジンを置き換えるプラグインです。

目標は、[Visual Studio Code](https://code.visualstudio.com/) の [Markdown Preview Enhanced](https://shd101wyy.github.io/markdown-preview-enhanced/#/) 向けの markdown ファイルを軽易に Web で共有できる環境です。

今回は、[PlantUML](https://plantuml.com/) でガントチャートが描画できないことに気づいたので PlantUML のバージョンを変更してみました。

その際、既存のコードのコンパイル時に警告が表示されたため、調査した結果について記事にします。

## 出力された警告

```
method generateImage in class SourceStringReader is deprecated[warn] \path\to\gitbucket-markdown-enhanced\src\main\scala\io\github\yasumichi\gme\MarkdownEnhancedNodeRenderer.scala:161:12: method generateImage in class SourceStringReader is deprecated
[warn]     reader.generateImage(os, new FileFormatOption(FileFormat.SVG))
[warn]            ^
[warn] one warning found
```

SourceStringReader クラスの generateImage() メソッドは deprecated だそうです。

## 対処

代わりに outputImage() メソッドを使えば良いようです。


## 余談

あと、テーブルの右寄せが gitbucket.css に打ち消されるんだけど…

gitbucket側

```html
   <td>cell 21</td>
   <td style="text-align: left">cell 22</td>
   <td style="text-align: center">cell 22</td>
   <td style="text-align: right">cell 22</td>
```

gme側

```html
<td>cell 21</td><td align="left">cell 22</td><td align="center">cell 22</td><td align="right">cell 22</td>
```

```css:gitbucket.css
div.markdown-body table th,
div.markdown-body table td {
  padding: 8px;
  line-height: 20px;
  text-align: left;
  vertical-align: top;
  border-top: 1px solid #dddddd;
}
```

こいつに負けちゃうので JavaScript で暫定対処


[I think it is unnecessary to set text-align in the cells of a table created with markdown. #3899](https://github.com/gitbucket/gitbucket/pull/3899)