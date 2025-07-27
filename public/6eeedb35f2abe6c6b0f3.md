---
title: WSL のコアファイルってどこに作られるの？と探した件
tags:
  - プログラミング
  - WSL
private: false
updated_at: '2025-01-20T21:30:56+09:00'
id: 6eeedb35f2abe6c6b0f3
organization_url_name: null
slide: false
ignorePublish: false
---
## 結論

`%USERPROFILE%` の `AppData\Local\Temp\wsl-crashes` にありました。

## 環境

```
wsl --version
WSL バージョン: 2.3.26.0
カーネル バージョン: 5.15.167.4-1
WSLg バージョン: 1.0.65
MSRDC バージョン: 1.2.5620
Direct3D バージョン: 1.611.1-81528511
DXCore バージョン: 10.0.26100.1-240331-1435.ge-release
Windows バージョン: 10.0.26100.2605
```

```
>wsl --list
Linux 用 Windows サブシステム ディストリビューション:
Ubuntu-24.04 (既定)
```

## コアファイルのパターンを調べた

```
$ cat /proc/sys/kernel/core_pattern
|/wsl-capture-crash %t %E %p %s
```

なんだ、これ。

## 見つけた参考文献

- [Problems generating corefiles with WSL2 · Issue #11997 · microsoft/WSL](https://github.com/microsoft/WSL/issues/11997)

[この辺り](https://github.com/microsoft/WSL/issues/11997#issuecomment-2351356656)に

> In wsl-2.3.17 core dumps are stored in \AppData\Local\Temp\wsl-crashes folder under your Windows home directory.

って書いてあって、その通りの場所に見つかりました。

`file` で確認したところ、形式はちゃんと `ELF 64-bit LSB core file` になってました。

## 結論

`%USERPROFILE%` の `AppData\Local\Temp\wsl-crashes` にありました。

気が向いたら、加筆修正します(｀･ω･´)ゞ
