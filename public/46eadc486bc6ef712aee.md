---
title: 社内PCAPも安心！外部API不要・完全ローカルLLMで動く問題生成ツール「pcap2quiz」
tags:
  - LLM
  - pcap
private: false
updated_at: '2026-09-13T12:30:43+09:00'
id: 46eadc486bc6ef712aee
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

https://github.com/yasumichi/pcap2quiz

## 筆者の開発環境について

| 項目    | 値                                             |
| ------- | ---------------------------------------------- |
| CPU     | 11th Gen Intel(R) Core(TM) i7-1165G7 @ 2.80GHz |
| メモリ  | 16GB                                           |
| OS      | Fedora Linux 44 (Workstation Edition)          |
| Ollama  | 0.34.0                                         |
| LLM     | gemma4:e2b                                     |
| Python  | 3.14.7                                         |
| pyshark | 0.6                                            |
| ollama  | 0.6.2                                          |

## 開発の経緯

職場の教育機関でプログラミングやネットワークなどについて教えています。

アプリケーションプロトコルの実習に解析使用するパケットキャプチャが古く、更新がままならない状態でした。

そこで AI を活用して、パケットキャプチャから、パケット解析練習問題を作成できないか、試してみることにしました。

当初は、[Gemini](https://gemini.google.com/app) にパケットキャプチャをアップロードして問題を作成させてみました。

- [dns-remoteshell.pcap解析演習](https://gemini.google.com/share/8af4dd7364a5?skid=5bcf521e-2c1e-45e0-b8cd-95390aca5c97)

これは、[SampleCaptures - Wireshark Wiki](https://wiki.wireshark.org/samplecaptures) から拝借した `dns-remoteshell.pcap` を元に作成してもらったクイズです。

与えたプロンプトは、次のとおりです。

> dns-remoteshell.pcap を解析し、SOC要員候補に解析の練習をさせる問題を作成 

それなりに満足のいく練習問題が作成できました。

しかし、社内の PCAP を Gemini にアップロードするのは問題があります。

そこでローカル LLM を利用して同様のことができないかを考えました。

## ローカル LLM にどうやってパケットキャプチャを読ませるか?

[Kali Linux](https://www.kali.org/) にどのようなアプリケーションがインストールされているのか確認している際に [Scapy](https://scapy.net/) という [Python](https://www.python.org/) ライブラリを見つけました。

Scapy は、インタラクティブなパケット操作ライブラリでパケットキャプチャファイルから、Python オブジェクトを生成できます。また、管理者権限が必要ですが、ゼロから任意のパケットを作成し、送出することもできるようです。

最初は、これを研究していましたが、LLM に使うには出力がイマイチであると感じました。

次に [Wireshark](https://scapy.net/) に付属している [tshark](https://www.wireshark.org/docs/man-pages/tshark.html) というコマンドラインツールの出力が使えないかを考えました。

これを Python で利用できないか調べたところ、[PyShark](https://kiminewt.github.io/pyshark/) という tshark のラッパーが見つかりました。

:::note info

Gemini に推敲させたところ、なぜ、Pyshark なのかの説明について、以下の提案がありました。

> Scapy だと生のバイナリ解析が必要でプロトコル解釈の整形が大変だが、tshark/pyshark だと Wireshark の強力なプロトコル解析（フィールド展開・テキスト化）をそのまま活かせるため

:::

## まずは Gemini に試作してもらう

以下のプロンプトを与えました。

> pyshark でpcap ファイルを解析しollama経由で gemma4 に渡し、SOC要員向けのクイズをHTMLやaiken形式で出力するプログラム

最初のプログラムを実際に動かしてみると以下のようなエラーが発生して動きませんでした。

<details>
  <summary>初回のエラー</summary>

```
 Traceback (most recent call last):

File "/home/username/Downloads/pcap_quiz_generator.py", line 162, in <module>

main()

~~~~^^

File "/home/username/Downloads/pcap_quiz_generator.py", line 146, in main

summary = extract_pcap_summary(args.pcap)

File "/home/username/Downloads/pcap_quiz_generator.py", line 12, in extract_pcap_summary

cap = pyshark.FileCapture(pcap_path, keep_packets=False)

File "/home/username/.local/share/mise/installs/python/3.14.7/lib/python3.14/site-packages/pyshark/capture/file_capture.py", line 38, in __init__

super(FileCapture, self).__init__(display_filter=display_filter, only_summaries=only_summaries,

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

decryption_key=decryption_key, encryption_type=encryption_type,

^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

...<3 lines>...

include_raw=include_raw, eventloop=eventloop,

^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

custom_parameters=custom_parameters, debug=debug)

^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

File "/home/username/.local/share/mise/installs/python/3.14.7/lib/python3.14/site-packages/pyshark/capture/capture.py", line 88, in __init__

self._setup_eventloop()

~~~~~~~~~~~~~~~~~~~~~^^

File "/home/username/.local/share/mise/installs/python/3.14.7/lib/python3.14/site-packages/pyshark/capture/capture.py", line 183, in _setup_eventloop

self.eventloop = asyncio.get_event_loop_policy().get_event_loop()

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~^^

File "/home/username/.local/share/mise/installs/python/3.14.7/lib/python3.14/asyncio/events.py", line 718, in get_event_loop

raise RuntimeError('There is no current event loop in thread %r.'

% threading.current_thread().name)RuntimeError: There is no current event loop in thread 'MainThread'. 
```

</details>

Gemini にエラーの原因を聞いてみました。

> このエラーの原因は Python 3.14（または Python 3.10以降）で asyncio.get_event_loop() の仕様変更・非推奨化が行われたこと にあります。
>
> PyShark（内部で asyncio を利用）がメインスレッドでイベントループを自動取得しようとした際、新しい Python バージョンではイベントループが自動生成されずに RuntimeError: There is no current event loop... が発生しています。

修正案を適用してみましたが、似たようなエラーが発生します。

<details>
  <summary>修正後のエラー</summary>

```
 Traceback (most recent call last):

File "/home/username/Downloads/pcap_quiz_generator.py", line 169, in <module>

main()

~~~~^^

File "/home/username/Downloads/pcap_quiz_generator.py", line 153, in main

summary = extract_pcap_summary(args.pcap)

File "/home/username/Downloads/pcap_quiz_generator.py", line 19, in extract_pcap_summary

cap = pyshark.FileCapture(pcap_path, keep_packets=False)

File "/home/username/.local/share/mise/installs/python/3.14.7/lib/python3.14/site-packages/pyshark/capture/file_capture.py", line 38, in __init__

super(FileCapture, self).__init__(display_filter=display_filter, only_summaries=only_summaries,

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

decryption_key=decryption_key, encryption_type=encryption_type,

^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

...<3 lines>...

include_raw=include_raw, eventloop=eventloop,

^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

custom_parameters=custom_parameters, debug=debug)

^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

File "/home/username/.local/share/mise/installs/python/3.14.7/lib/python3.14/site-packages/pyshark/capture/capture.py", line 88, in __init__

self._setup_eventloop()

~~~~~~~~~~~~~~~~~~~~~^^

File "/home/username/.local/share/mise/installs/python/3.14.7/lib/python3.14/site-packages/pyshark/capture/capture.py", line 200, in _setup_eventloop

asyncio.set_child_watcher(asyncio.SafeChildWatcher())

^^^^^^^^^^^^^^^^^^^^^^^^^

File "/home/username/.local/share/mise/installs/python/3.14.7/lib/python3.14/asyncio/__init__.py", line 74, in __getattr__

raise AttributeError(f"module {__name__!r} has no attribute {name!r}")AttributeError: module 'asyncio' has no attribute 'set_child_watcher'
```

</details>

こちらも Gemini に聞いてみます。

> 原因は Python 3.14 で asyncio.set_child_watcher / SafeChildWatcher が完全に削除（非推奨を経て廃止）されたためです。
> 
> pyshark ライブラリの内部コードが Python 3.12/3.14 以降の asyncio 仕様変更に対応しておらず、古い関数を呼び出そうとしてクラッシュしています。

修正プログラムを動かしてみたところ、HTML は出力されましたが、クイズは動作しませんでした。

## Google Antigravity CLI に切り替える

Gemini の Web UI でプログラムを修正していくのは効率が悪いので [Google Antigravity CLI](https://antigravity.google/) を利用することにしました。

まずは、現状の問題点と仕様を確定するために以下のプロンプトを与えました。

> pyshark でpcap ファイルを解析しollama経由で gemma4 に渡し、SOC要員向けのクイズをHTMLやaiken形式で出力するプログラムを作成しようとしている。 @[pcap2quiz.py] は、Gemini に作らせたサンプルである。
> 現状、実際に動作する HTML ファイルは生成されない。
> 改善案を含め、まずは、仕様の確定を行いたい。AGENTS.md の案を提案してください。

以下は、回答の冒頭です。

```
  pcap2quiz.py の現状コードおよび、実際に動作するHTML（あるいはAiken形式）が生成されない・パースに失敗する原因や課題を踏まえ、機能拡張・改善案を盛り込んだ AGENTS.md の提案を作成しました。
  ──────
  ### 原因の分析と改善のポイント

  1. LLM出力の不安定さ・パース失敗
      • 単純なテキスト指示プロンプトだけでは、Markdownの装飾コードブロック（```html や解説テキスト）が混ざったり、HTMLタグが破損して完全なHTMLが得られないケースが多く発生します。
      • 対策: JSON Schemaによる Structured Outputs (JSON) でクイズデータ（問題、選択肢、正解、解説）を取得し、プログラム側でHTMLテンプレートやAiken形式テキストへ確実に変換・レンダリングする方式を推奨します。
  2. PCAP解析の精度と情報量不足
      • パケットごとの単発サマリ（先頭300パケット等）だけでは、通信セッション全体のフローや統計（IP/ポート別通信量、通信時間帯、通信の目的等）がLLMに伝わりにくい状態です。
      • 対策: エントリごとのパケット情報だけでなく、セッション/フロー単位の統計（会話ペア、DNSリクエスト集計、HTTPリクエストURI一覧等）を事前抽出してLLMへ入力します。
  3. モデル指定と互換性
      • デフォルトが gemma2 になっているため、要望通り gemma:4（または実行環境のOllamaモデル名）を柔軟に扱えるよう設定できるようにします。
  ──────
```

- [当初のAGENTS.md](https://github.com/yasumichi/pcap2quiz/blob/b85463c/AGENTS.md)

この AGENTS.md を元にプログラムを修正してもらいました。

- [pcap2quiz.py](https://github.com/yasumichi/pcap2quiz/blob/b85463cffa4519199802bca4784667a76199554e/pcap2quiz.py)

### ポイント：正常通信でのハルシネーションを防ぐプロンプト設計

しかしながら、何の攻撃の兆候もないパケットを渡してもあたかも攻撃の兆候があったようなクイズを作成されてしまいました。

そこで修正を依頼します。

> サイバー攻撃の兆候がないパケットであってもその兆候があるものとして解釈しているように思われる。モデルに渡すプロンプトを改善できないか?

以下、回答です。

```
  指摘の通り、現在のプロンプトには以下のような問題があり、「正常な通信まで無理やり攻撃と解釈してしまう（ハルシネーション・誤検知）」 原因になっています。

  ### 原因分析
      • ログ内に攻撃がなくても、モデルが指示に引きずられて「暗号化通信（TLS）＝ C2通信」「通常のWebアクセス ＝ Web攻撃」などと無理な解釈を行って問題を作成してしまいます。
  2. 正常/異常の判断プロセスの欠如
      • プロンプトで「まず通信が正常か異常かを正しく客観的に分析・分類すること」を求めていないため、何でも攻撃シナリオに当てはめようとしてしまいます。

  ──────
  ### プロンプトの改善案

  プロンプトに以下の指針を追加・変更することを提案します。

  1. 客観的・事実に基く解析の徹底 (Fact-based Analysis)
      • 「PCAPに存在する事実のみに基づいて問題を作成すること。根拠のない攻撃シナリオを創作・決めつけないこと」を明記。
  2. 「正常通信理解」と「異常検知」のバランス
      • 攻撃の兆候がない正常なPCAPの場合は、無理に攻撃とみなさず、「正常なプロトコル動作の理解」「通信フローの分析手法」「正常通信と不審通信の見極め方（例：正常なDNS/HTTP通信と判断できる理由）」 を問うクイズを作成させる。
  3. 明示的な判定フェーズの導入
      • 「①提示されたサマリに明確な異常・不審通信が含まれるか判断する」「②異常がある場合はその検知・分析を問う」「③異常がない場合は正常なトラフィック解析やプロトコルの仕様・分析観点を問う」というステップを指示。

  ──────
```

[SampleCaptures - Wireshark Wiki](https://wiki.wireshark.org/samplecaptures) から拝借した `dns-icmp.pcapng` を元に作成したサンプルが次のリンクです。

- [PCAP解析サマリに基づくSOCアナリスト向け実践クイズ（10問）](https://www.catv296.ne.jp/~always/pcap2quiz_sample.html)

まだ、妥当性を確認していませんが、それなりの問題ができたと思います。

## まとめと ToDo

ローカル LLM（Ollama）と PyShark を組み合わせることで、社内の機密情報や実際のトラフィックデータを含む PCAP であっても、外部へデータを出さずに安全に教材化できる見通しが立ちました。

同様の悩みを持つ方のヒントになれば幸いです。

- 生成されたクイズの精度・妥当性のさらなる評価
- Aiken 形式の検証（Moodle 等への取り込みテスト）
- 対応プロトコルの拡充

