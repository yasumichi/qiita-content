---
title: 【VSCode Qiita Editor】qiita-cli を VSCode に統合する拡張機能が欲しい
tags:
  - Markdown
  - VSCode
  - qiita-cli
private: false
updated_at: '2025-08-16T23:49:02+09:00'
id: dffcff0287e8efc11a3d
organization_url_name: null
slide: false
ignorePublish: false
---
## 概要

[Zenn CLI](https://zenn.dev/zenn/articles/install-zenn-cli)には、 VS Code に統合する非公式の拡張 [VS Code Zenn Editor](https://marketplace.visualstudio.com/items?itemName=negokaz.zenn-editor)があり、Zenn の記事作成に非常に重宝している。

[Qiita CLI](https://qiita.com/Qiita/items/666e190490d0af90a92b) にも同様の拡張があれば、便利そうなのだが、今のところ、該当する拡張機能を見つけることができない。

せめて、各記事をファイル名ではなく、title で表示するツリービューは欲しい。

そこで自分で拡張機能を作成してみるという試みです。

既にそういう拡張機能があるよ、という情報があれば、ご教示いただけますと幸いです。

## これまでのプロダクト

ちょっと脱線しますが、今までに開発に関わってきた公開されているプロダクトについて紹介します。

ニッチなところばかり攻めているのでちっともバズりませんが :smile:

| プロダクト | 説明 |
| --------- | ---- |
|[SeaPig](https://github.com/yasumichi/seapig) | Electron で作成したマークダウンエディタ |
|[expdftable](https://github.com/yasumichi/expdftable) | [PyMuPDF](https://github.com/pymupdf/PyMuPDF) により、PDF の表を Excel ブックに抽出する。|
|[DriedPotato](https://github.com/yasumichi/DriedPotato) | [Django](https://www.djangoproject.com/) で開発したチームの欲しいものリストを管理する Web アプリケーション |
|[mael for VBA](https://github.com/yasumichi/maelForVBA) | マークダウンから Excel 表を作成するアドイン。テストケースなどをマークダウンで管理できる。|
|[gitbucket-drawio-plugin](https://github.com/yasumichi/gitbucket-drawio-plugin)| GitBucket で Draw.io のファイルを表示するプラグイン。原作者に放置されていたので勝手に動くようにした。 |
|[GitBucket Markdown Enhanced Plugin](https://github.com/yasumichi/gitbucket-markdown-enhanced)| GitBucket で[Visual Studio Code](https://code.visualstudio.com/) の [Markdown Preview Enhanced](https://shd101wyy.github.io/markdown-preview-enhanced/#/) のようなマークダウンを処理できることを目指すプラグイン |

## 拡張機能の雛型を作成

御多分に漏れず、`yo code` で作成しています。具体的な手順は、多数の参考文献で紹介されていますのでそちらを参照してください。

ここでは、ハマった点だけ書いておきます。

- `generator-code` から先にインストールしたら、`yo` から認識されなかった。
  - `generator-code` を再インストールしたら認識した。
- タイミングが悪かったのか、`package.json` に書かれた `@types/vscode` のバージョンをインストールできず、Git リポジトリの初期化まで行かなかった… :sweat:
  - `@types/vscode` のバージョンを下げて、再度、`npm install`
  - `engines` の `vscode` のバージョンが、インストールバージョンと合ってなくて当初、デバック起動できなかった。

## 拡張機能を有効化するイベントの定義

[package.json](https://github.com/yasumichi/vscode-qiita-editor/blob/db6d254744f36c716e7e26e7691fe070f5b0f7c2/package.json) の `activationEvents` に拡張機能を有効化するイベントを定義する。

```json
  "activationEvents": [
    "workspaceContains:/qiita.config.json"
  ],
```

`workspaceContains` に `/qiita.config.json` を指定することでワークスペースに該当ファイルが含まれていれば、拡張機能を有効化できる。

## ツリービューの設定

ツリービューの作成は、以下のサイトを参考にした。

- [VSCodeのサイドメニューの拡張の作り方 #VSCode-Extension - Qiita](https://qiita.com/Teach/items/3622e159782f2baecaf1)

[package.json](https://github.com/yasumichi/vscode-qiita-editor/blob/db6d254744f36c716e7e26e7691fe070f5b0f7c2/package.json) の `contributes` に `views` の記述を追加した。

```json
  "contributes": {
    "views": {
      "explorer": [
        {
          "id": "qiita",
          "name": "Qiita Contents",
          "when": "qiita-editor.activated"
        }
      ]
    }
  }
```

`explorer` を指定するとエクスプローラーのサイドパネルにビューが追加される。

`when` でビューを有効化するタイミングを制御できる。

## activate 関数で qiita-editor.activated を有効にする

`src\extension.ts` の `activate` 関数で `vscode.commands.executeCommand` を呼び出し、コンテキストを設定する。

```ts
export async function activate(context: vscode.ExtensionContext) {
  vscode.commands.executeCommand('setContext', 'qiita-editor.activated', true);
}
```

これだけで何も表示できないツリービューが表示される。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/36738/c208252b-247e-412c-8580-b9c09c4ffdd1.png)

## ツリーの項目を表すクラスの作成

[src\treeView\qiitaTreeItem.ts](https://github.com/yasumichi/vscode-qiita-editor/blob/db6d254744f36c716e7e26e7691fe070f5b0f7c2/src/treeView/qiitaTreeItem.ts) を追加した。

```ts
// refer to https://qiita.com/Teach/items/3622e159782f2baecaf1
import * as vscode from 'vscode';

export class QiitaTreeItem {
    private _children: QiitaTreeItem[];
    private _parent: QiitaTreeItem | undefined | null;

    constructor(public name: string, public path: string = "") {
        this._children = [];
    }

    get parent(): QiitaTreeItem | undefined | null {
        return this._parent;
    }

    get children(): QiitaTreeItem[] {
        return this._children;
    }

    addChild(child: QiitaTreeItem) {
        child.parent?.removeChild(child);
        this._children.push(child);
        child._parent = this;
    }

    removeChild(child: QiitaTreeItem) {
        const childIndex = this._children.indexOf(child);
        if (childIndex >= 0) {
            this._children.splice(childIndex, 1);
            child._parent = null;
        }
    }
}
```

とりあえず、外部から設定できる項目は、以下の２つを用意した。

- name : frontmatter の `title` を保持させる。表示名となる。
- path : VS Code の uri。ファイルの場所を保持させる。

## データプロバイダーの作成

[src\treeView\qiitaTreeViewProvider.ts](https://github.com/yasumichi/vscode-qiita-editor/blob/db6d254744f36c716e7e26e7691fe070f5b0f7c2/src/treeView/qiitaTreeViewProvider.ts) を作成した。

当初は、記事を追加する部分は書かなかった。

```ts
// refer to https://qiita.com/Teach/items/3622e159782f2baecaf1
import * as vscode from 'vscode';
import { QiitaTreeItem } from './qiitaTreeItem';
import fs, { read } from 'fs';
import path from 'path';
import * as readline from 'readline';
import * as YAML from 'yaml';

export class QiitaTreeViewProvider implements vscode.TreeDataProvider<QiitaTreeItem> {
    private rootItems: QiitaTreeItem[];

    constructor() {
        const published = new QiitaTreeItem("Published");
        const drafts = new QiitaTreeItem("Drafts");
        this.rootItems = [
            published,
            drafts
        ];
    }

    onDidChangeTreeData?: vscode.Event<void | QiitaTreeItem | QiitaTreeItem[] | null | undefined> | undefined;

    getTreeItem(element: QiitaTreeItem): vscode.TreeItem | Thenable<vscode.TreeItem> {
        //const collapsibleState = element.children.length > 0 ? vscode.TreeItemCollapsibleState.Collapsed : vscode.TreeItemCollapsibleState.None;
        const collapsibleState = element.parent ?  vscode.TreeItemCollapsibleState.None : vscode.TreeItemCollapsibleState.Collapsed;
        return new vscode.TreeItem(element.name, collapsibleState);
    }

    getChildren(element?: QiitaTreeItem | undefined): vscode.ProviderResult<QiitaTreeItem[]> {
        return element ? element.children : this.rootItems;
    }

    getParent?(element: QiitaTreeItem): vscode.ProviderResult<QiitaTreeItem> {
        throw new Error('Method not implemented.');
    }

    resolveTreeItem?(item: vscode.TreeItem, element: QiitaTreeItem, token: vscode.CancellationToken): vscode.ProviderResult<vscode.TreeItem> {
        throw new Error('Method not implemented.');
    }

}
```

`getParent` や `resolveTreeItem` は、未実装のままである。

## データプロバイダーの登録

`src\extension.ts` の `activate` 関数でデータプロバイダーを登録する。

```ts
	const qiitaTreeViewProvider = new QiitaTreeViewProvider();
	vscode.window.registerTreeDataProvider("qiita", qiitaTreeViewProvider);
```

ここまででルートアイテムのみ、表示できるようになった。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/36738/9847b1c9-4b24-4f80-96e6-28117874f9e0.png)

## 投稿コンテンツの一覧を表示できるようにする

`QiitaTreeViewProvider` クラスのコンストラクターに以下のコードを追加した。

```ts
        vscode.workspace.findFiles("public/*.md").then(files => {
            files.forEach((val, index) => {
                const uri = val.path;
                const fullpath = val.path.slice(1);
                const rs = fs.createReadStream(fullpath, 'utf-8');
                const rl = readline.createInterface(rs);

                var yaml: string = "";
                var onMeta = false;
                var complete = false;

                rl.on('line', (line) => {
                    if (line.match(/^---$/)) {
                        if (onMeta) {
                            complete = true;
                            onMeta = false;
                            rl.close();
                        } else {
                            onMeta = true;
                        }
                    } else {
                        if (onMeta) {
                            yaml = yaml + line + "\n";
                        }
                    }
                });
                rl.on('close', () => {
                    rs.close();
                    if (complete) {
                        const result = YAML.parse(yaml);
                        if(result.id) {
                            const article = new QiitaTreeItem(result.title, uri);
                            published.addChild(article);
                        } else {
                            const article = new QiitaTreeItem(path.basename(fullpath) , uri);
                            drafts.addChild(article);
                        }
                    }
                });
            });
        });
```

ロジックは、[VS Code Zenn Editor](https://marketplace.visualstudio.com/items?itemName=negokaz.zenn-editor)からの拝借である…。 :sweat:

当初は、`title:` の行だけ処理する簡単なロジックにしていたが、以下のようになっている場合があるため、同様のロジックとした。

```yaml
title: >-
  Electron 7.0.x で webContents.printToPDF() が promisification
  されたけどドキュメントの例が直っていない件
```

簡単に言うと `---` から `---` までの間を取得して、YAML パーサーで解釈させるという流れである。

これで以下のように投稿コンテンツを一覧表示できるようになった。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/36738/4b02bc4c-14bf-4c2f-b16d-9918194b8227.png)

アイコンなどもないので何の色気(?)もないが…。

## 記事を選択したらエディタで開くようにする

`src\extension.ts` の `activate` 関数でデータプロバイダー登録要領を変更した。

```ts
	const qiitaTreeViewProvider = new QiitaTreeViewProvider();
	var treeView = vscode.window.createTreeView("qiita", { treeDataProvider: qiitaTreeViewProvider });
	treeView.onDidChangeSelection((ev) => {
		ev.selection.forEach(async (val,index) => {
			console.log(val.path);
			if (val.path.length > 0) {
				try {
					const doc = await vscode.workspace.openTextDocument(val.path);
					return await vscode.window.showTextDocument(doc, vscode.ViewColumn.One, true);
				} catch (e) {
					// 選択したファイルがテキストではない場合
					console.log(e);
				}
			}
		});
	});
```

`vscode.window.createTreeView` でツリービューを作成するようにし、作成したツリービューの `onDidChangeSelection` イベントを登録する。

ノードの `path` が設定されていれば、以下の２つの API を呼び出し、テキストエディタに表示するという流れである。

- `vscode.workspace.openTextDocument`
- `vscode.window.showTextDocument`

## 今後の ToDo

他に良い拡張機能が見つかれば、開発を辞めてしまうかもしれませんが…

- ノードに更新日時 `updated_at` を持たせ、更新日時順にソートさせるようにする。
- ファイル更新時にツリービューが更新できるようにする。
- `npx qiita new` をボタンで実行できるようにする。
- [ファイルアップロード](https://qiita.com/settings/uploading_images)のリンクを開けるようにする。
- モジュール分割を適切にしたい。
- 非同期処理をマジメにやる… :smile:

## 参考文献

- [Zenn CLI](https://zenn.dev/zenn/articles/install-zenn-cli)
- [VS Code Zenn Editor](https://marketplace.visualstudio.com/items?itemName=negokaz.zenn-editor)
  - [negokaz/vscode-zenn-editor: Zenn CLI を VS Code に統合する非公式の拡張](https://github.com/negokaz/vscode-zenn-editor/tree/main)
- [Extension API | Visual Studio Code Extension API](https://code.visualstudio.com/api)
- [vscode-extension-samples/tree-view-sample at main · microsoft/vscode-extension-samples](https://github.com/microsoft/vscode-extension-samples/tree/main/tree-view-sample)
- [VSCodeのサイドメニューの拡張の作り方 #VSCode-Extension - Qiita](https://qiita.com/Teach/items/3622e159782f2baecaf1)
- [知識0の状態からたった2時間でVSCodeの拡張機能を作った話 #Python - Qiita](https://qiita.com/_ken_/items/06161e8d4bac5e0bd209)
- [VSCode Extensions(拡張機能) 自作入門 〜VSCodeにおみくじ機能を追加する〜 #VSCode - Qiita](https://qiita.com/HelloRusk/items/073b58c1605de224e67e)
- [かんたん！VS Code拡張機能開発 | DevelopersIO](https://dev.classmethod.jp/articles/easy-vs-code-extension-development/)
- [VS Codeの拡張機能を自作しよう - たーせる日記](https://tercel-tech.hatenablog.com/entry/2024/03/07/233436)
- [TypeScript入門『サバイバルTypeScript』〜実務で使うなら最低限ここだけはおさえておきたいこと〜](https://typescriptbook.jp/)
