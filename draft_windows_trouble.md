#### 記事ドラフト：Windowsトラブルシューティング編

**タイトル:**
**【Windows】開発環境構築のトラブルシューティング集：PowerShell実行ポリシー・Dockerホットリロード・パス問題**

-----

### はじめに：Windowsでの開発は「罠」がいっぱい？

Web開発のドキュメントや記事を見ていると、コマンド例がMac/Linux向けで「Windowsだとうまくいかない……」と落ち込むことはありませんか？

この記事では、私が Serverless Framework × FastAPI の開発環境を構築する際に遭遇した、**Windows特有のエラーと解決策**をまとめました。
同じエラー画面を見て絶望しているあなたの助けになれば幸いです。

-----

### Case 1: PowerShellで `venv` が有効化できない

Pythonの仮想環境を作って、いざ `activate` しようとしたら……

```powershell
.\venv\Scripts\Activate.ps1
```

**エラー発生！**

> `.\venv\Scripts\Activate.ps1 : このシステムではスクリプトの実行が無効になっているため、ファイル ... を読み込むことができません。`

画面が真っ赤な文字で埋め尽くされます。これはPowerShellのセキュリティ設定（実行ポリシー）が、デフォルトで「スクリプト実行禁止」になっているためです。

**✅ 解決策**
PowerShellで以下のコマンドを実行し、実行ポリシーを「ローカルのスクリプトなら許可（RemoteSigned）」に変更します。

```powershell
Set-ExecutionPolicy RemoteSigned -Scope CurrentUser
```

※ 「実行ポリシーを変更しますか？」と聞かれたら `Y` を入力してEnterを押してください。

これで再度 `Activate.ps1` を実行すれば、無事に `(venv)` が表示されるはずです。

-----

### Case 2: Dockerでホットリロードが効かない

Dockerを使って開発している際、ホスト（Windows）側のコードを編集して保存したのに、コンテナ内のサーバーが再起動してくれない（変更が反映されない）ことがあります。

特に Windows と WSL2 (Docker Desktop) のファイルシステムの境界をまたぐ場合、ファイルの変更通知（inotify）がうまく伝わらないことが原因です。

**✅ 解決策**
Node.js (nodemon) を使っている場合は、ポーリングモード（ファイルを定期的に監視するモード）を有効にします。

`package.json` のスクリプトを以下のように修正してください。

```json
"scripts": {
  // --legacy-watch (-L) オプションを追加
  "dev": "nodemon -L index.js"
}
```

これで、Windows側で保存した瞬間にDocker側でも検知されるようになります。

-----

### Case 3: ツールはどこにインストールすべき？（Global vs Local vs Docker）

`npm install -g serverless` （グローバル）するべきか、`npm install -D serverless` （ローカル）するべきか、それともDockerの中か……。
初心者が最も混乱するポイントの一つです。

**結論：プロジェクトごとに環境を分けられる「Local」か「Docker」が正解です。**

| 方法 | メリット | デメリット | 推奨度 |
| :--- | :--- | :--- | :--- |
| **Global (`-g`)** | どこでもコマンドが使える | バージョンが固定され、他のプロジェクトと競合する | ❌ 非推奨 |
| **Local (`-D`)** | プロジェクトごとにバージョン管理できる | コマンドの頭に `npx` が必要 | ⭕ 推奨 |
| **Docker内** | PCを汚さず、チーム全員で同じ環境を使える | 設定が少し複雑 | **🏆 最強** |

私のメイン記事（FastAPI × Serverless）では、**「Docker内」** を採用しています。これが最も環境依存トラブルが少ない方法だからです。

> **🔗 メイン記事はこちら：[【保存版】FastAPI × Serverless Framework開発の正解ルート](draft.md\#導入はじめに)**


-----