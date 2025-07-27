---
title: チームの欲しいを管理する DriedPotato を Django で試作してみた
tags:
  - Python
  - Django
private: false
updated_at: '2024-08-24T17:48:00+09:00'
id: df784ed3e6f2eddd1a84
organization_url_name: null
slide: false
ignorePublish: false
---
自己満足のゴミ記事です。（書きかけ）

いわゆる欲しいものリストを管理するアプリケーション [DriedPotato](https://github.com/yasumichi/DriedPotato) を Django で試作してみました。

![DriedPotato_ja.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/36738/65a3f3c7-a92f-545b-5b9f-bfbd3a605b49.png)

- コメントちゃんと書けよ
- Bootstrap 使ってるってすぐ分かるよ…
- Default の user モデル使ってるよ…
- Excel でいいんじゃない？

など、色々、聞こえてきそうですが… :sweat:

## 経緯

私のチームでは、ナレッジの管理に [GROWI](https://growi.org/ja/) という wiki サービスを利用していますが、欲しいものリストを wiki で管理するという話になりました。確かに[Handsontable によるテーブル編集機能](https://docs.growi.org/ja/guide/features/table.html) を使えば、それなりに出来なくもなさそうですが、優先順位の変更などを考えると向いてない使い方かなと感じました。

そこで専用のアプリを書いてみようと思い、実行に移してみました。

まだ、チームのフィードバックをもらえてないので洗練できてないですし、一般的なチームに適合しているのか分かりません。

ここでは、ハマったことなどを随時、追加していこうかと思ってます。

## DriedPotato の試用手順

- Python のインストール、必要により venv 環境の構築
- 必要パッケージのインストール `pip install -r requirements.txt`
- データベースの設定(参考：[データベース | Django ドキュメント | Django](https://docs.djangoproject.com/ja/5.0/ref/databases/))[^1]
- データベースへのテーブル作成 `python manage.py migrate`
- 管理ユーザーの作成 `python manage.py createsuperuser`
- 開発サーバーの起動 `python manage.py runserver`
- 管理サイト http://127.0.0.1:8000/admin/ にログインし、ユーザーや欲しいものの分類を作成
- ブラウザで http://127.0.0.1:8000/ にアクセス

[^1]: デフォルトの sqlite を使う場合は不要

## PasswordChangeView（の派生クラス）の POST 先は自身だよ

最初、PasswordChangeView の派生クラスで表示する画面の POST 先を PasswordChangeDoneView の派生クラスにしていて パスワード変更ができずに悩んでました。（よく読め…）

### 参考リンク

- [Python + Djangoでパスワード変更画面を実装する #Python - Qiita](https://qiita.com/t-iguchi/items/67430e164de0e6701dc8)
- [【Django】パスワード変更フォームを自作する方法 | アントレプレナー](https://kosuke-space.com/django-password-change)

## uWsgi へのデプロイでハマったところ

以下、ubuntu-server での例です。

### uWsgi のインストールに先立ち PCRE の 開発パッケージをインストールしておきましょう

- [Installing uWSGI — uWSGI 2.0 documentation](https://uwsgi-docs.readthedocs.io/en/latest/Install.html)

この辺の情報を元に当初は、uWsgi のインストールに先立ち以下のパッケージをインストールしました。

```
$ sudo apt install build-essential python python-dev
```

しかし、uWsgi のログに以下のエラーが出力されてしまいました。

```
!!! no internal routing support, rebuild with pcre support !!!
```

このメッセージを消すには先にPCRE の 開発パッケージをインストールしてから、uWsgi をインストールする必要があります。

```
$ sudo apt install libpcre3-dev
```

- [HTTP で uWSGI を動かす │ htkyama.org](https://www.htkyama.org/netbsd/uwsgi.html)

### iniファイルを使用する場合、socket オプションも入れておきましょう

- [Django を uWSGI とともに使うには？ | Django ドキュメント | Django](https://docs.djangoproject.com/ja/5.0/howto/deployment/wsgi/uwsgi/)

このドキュメントに基づき、ini ファイルを作成し、

```
$ uwsgi --ini uwsgi.ini
```

で起動しましたが、ログに以下のようなエラーが出力されました。

```
The -s/--socket option is missing and stdin is not a socket.
```

起動時に `--socket` オプションを指定するか、ini ファイルに socket オプションを入れておきましょう。

```ini
[uwsgi]
chdir=/path/to/DriedPotato
module=DriedPotato.wsgi:application
master=True
pidfile=/tmp/DriedPotato-master.pid
vacuum=True
max-requests=5000
daemonize=/path/to/log/DriedPotato.log
socket=:8001    # この行を追加
```

- [Django Running on uWSGI with NGiNX - INI Method Not Working - Stack Overflow](https://stackoverflow.com/questions/31147641/django-running-on-uwsgi-with-nginx-ini-method-not-working)

### 直接 HTTP 通信を行う場合は、socket ではなく http

Nginx などを介さず、直接、ブラウザから uWsgi と通信を行う場合は、`socket` ではなく `http` で指定します。

```
http=0.0.0.0:8001
```

- [くりーむわーかー : Python Django+uWSGIの設定](https://cream-worker.blog.jp/archives/1078251491.html)

### 実用的には buffer-size を指定しましょう

実際にブラウザからアクセスしてみるとうまく接続できません。ログを見てみると以下のようなメッセージが出力されています。

```
invalid request block size: 21573 (max 4096)...skip
```

先ほどのページにも書いてありますが、`buffer-size` を指定します。

```
buffer-size=32768
```

### Python のスレッドサポートを有効にする場合はその指定も入れておきましょう

先ほどの ini ファイルでも起動は可能ですが、ログに以下のようなメッセージが出力されます。

```
*** Python threads support is disabled. You can enable it with --enable-threads ***
```

`--enable-threads` を指定して起動するか、ini ファイルに以下のような行を追加します。

```ini
enable-threads=true
threads=4
```

- [Python APサーバー、uWSGI | サーバーレシピ](https://server-recipe.com/3736/)

### 停止する場合は `--stop` オプションを使います

```
$ uwsgi --stop /tmp/DriedPotato-master.pid
```

- [NginxとuWSGIでHelloWorld #Python - Qiita](https://qiita.com/koyoru1214/items/57461b920dfc11f67683)


