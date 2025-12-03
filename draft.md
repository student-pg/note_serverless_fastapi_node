**記事タイトル:**
**【保存版】FastAPI × Serverless Framework開発の正解ルート：Dockerで環境汚染ゼロの爆速デプロイ**

### 導入（はじめに）

モダンなWeb開発において、Python (FastAPI) の高速なバックエンド処理と、AWS Lambdaによるサーバーレスアーキテクチャの組み合わせは、スケーラビリティとコスト効率を両立する強力な選択肢です。

しかし、Windows環境でのローカル開発においては、PowerShellの権限設定や、Python/Node.jsのバージョン管理といった環境依存の問題が開発効率を落とす要因となりがちです。

そこで本記事では、**Dockerコンテナ技術を活用し、OS依存を排除した「再現性の高いハイブリッド開発環境」**の構築手法を解説します。

Node.js (Serverless Framework) とPython (FastAPI) が共存するDocker環境を構築することで、ローカル環境を汚さず、チーム開発でも一貫した動作を保証する **「正解ルート」** を示します。

また、開発プロセスにおいて多くのエンジニアが直面しやすい、AWS IAM権限の細かな設定やリージョン指定の罠についても、具体的な解決策と共に網羅しています。

### この記事で得られる知見
- **環境分離**: DockerだけでPython(FastAPI)とNode.js(Serverless Framework)が動く、クリーンなハイブリッド環境の構築手法

- **実践的デプロイ**: ローカル環境を汚さずにAWS Lambdaへ安全にデプロイするフロー

- **トラブルシューティング**: AWS権限周りやバージョン不整合など、実務でハマりやすいポイントの回避策

- **技術**
  * **言語:** Python (FastAPI)
  * **デプロイ:** AWS Lambda + API Gateway
  * **ツール:** Serverless Framework

---

### 事前準備（これから始める方へ）

この記事をスムーズに進めるために、以下の準備が必要です。まだの方は、別記事で詳しく解説していますので、そちらを先に済ませてから戻ってきてください。

> **🔗 準備編1：【初心者向け】AWSアカウント作成とセキュリティ設定（IAMユーザー）の完全ガイド**
> （※ここに、後で書く「AWSアカウント作成記事」のリンクを貼ります）

> **🔗 準備編2：Windowsユーザーのための環境構築トラブルシューティング**
> （※ここに、後で書く「エラー対応記事」のリンクを貼ります）

### 完成イメージ

最終的には、以下のようなAPIがAWS上で稼働し、世界中からアクセスできるようになります。

*(ここにブラウザのスクリーンショットを配置します)*

> **（※仮画像イメージ：以下のようなテキストをコードブロックで表現するか、ご自身のブラウザのスクショに差し替えてください）**
>
> ```json
> {
>   "message": "Hello from Serverless FastAPI!",
>   "environment": "AWS Lambda (via Mangum)"
> }
> ```

-----

### Step 0: 今回のアーキテクチャ

なぜ、あえてDockerを使うのか？ その理由と全体像を解説します。

Serverless FrameworkでPythonアプリケーションをデプロイする場合、通常はホストマシンにPythonとNode.jsの両環境が必要です。しかし、この方法はOSごとの差異（パスの区切り文字やライブラリの依存関係）により、環境構築のコストが高くなる傾向にあります。

この課題を解決するため、**Dockerコンテナ内に実行環境を完全にカプセル化する**アプローチを採用しました。

#### システム構成図

以下の図のように、開発作業はすべてDockerコンテナ内で行い、そこからAWSへ直接デプロイします。ホストPC（Windows）は「コードを書くだけ」の状態に保たれます。

<a href="./img/mermaid-diagram-2025-12-03-134008.png" target="_blank">
<img src="./img/mermaid-diagram-2025-12-03-134008.png" alt="アーキテクチャ図" style="display:block;margin:0 auto;max-width:900px;width:100%;height:auto;">
</a>

*（画像はクリックで原寸表示）*

#### ディレクトリ構成

プロジェクトの全体像は以下の通りです。

```text
serverless-fastapi-demo/
├── Dockerfile             # Node.jsベースにPythonを追加した特製イメージ
├── docker-compose.yml     # ポート設定とボリュームマッピング
├── serverless.yml         # AWSデプロイ設定
├── main.py                # FastAPIアプリケーション
├── requirements.txt       # Python依存ライブラリ
└── .env                   # AWS認証情報（※Gitには含めない！）
```

-----

修正案へのご承認、ありがとうございます！
「課題解決能力」と「技術的根拠」を前面に出したこのトーンなら、ポートフォリオとしても胸を張って提出できる内容になります。

それでは、この勢いで\*\*「Step 1: 最強のハイブリッドDocker環境を作る」\*\*の執筆に進みましょう。

ここは技術的な解説がメインになるパートです。単にコードを貼るだけでなく、**「なぜその設定が必要なのか？」**（例：なぜ `slim` なのか、なぜ `node_modules` を除外するのか）を解説することで、あなたの技術的な理解度の深さをアピールします。

以下のドラフトを VS Code の `draft.md` に追記してください。

-----
### Step 1: 最強のハイブリッドDocker環境を作る

#### 1\. Dockerfileの作成

プロジェクトルートに `Dockerfile` を作成します。

ここでは、以下のベストプラクティスを採用しています。

  * **軽量化:** ベースイメージに `node:24-slim` を採用し、イメージサイズを最小限に抑える。
  * **効率化:** `requirements.txt` をビルド時にインストールし、コンテナ起動ごとの待ち時間をなくす。
  * **セキュリティ:** 実行ユーザーを `root` ではなく `node` ユーザーに切り替え、権限を最小化する。

<!-- end list -->

```dockerfile:dockerfile
# 1. ベースイメージ
# Node.jsの軽量版(slim)を採用
FROM node:24-slim AS base

# 2. Python環境の追加
# Debian系なのでapt-getでPythonとpipをインストール
RUN apt-get update && apt-get install -y \
    python3 \
    python3-pip \
    python3-venv \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# 3. 依存関係のインストール (Dependencies Stage)
FROM base AS dependencies
WORKDIR /usr/src/app
COPY package*.json ./
# 開発用ライブラリも含めてインストール
RUN npm install

# 4. 実行用ステージ (Final Stage)
FROM base AS final
WORKDIR /usr/src/app

# Node.jsの依存関係をコピー
COPY --from=dependencies /usr/src/app/node_modules ./node_modules

# Pythonライブラリのインストール
# root権限でシステム全体にインストールし、全ユーザーから使えるようにする
COPY requirements.txt ./
RUN pip install -r requirements.txt --break-system-packages

# セキュリティ対策: 実行ユーザーを一般ユーザー(node)に切り替え
USER node

# アプリケーションコードをコピー
COPY --chown=node:node . .

# ポート公開 (Express:3000, FastAPI:8000)
EXPOSE 3000 8000

# デフォルトコマンド
CMD [ "npm", "run", "dev" ]
```

#### 2\. docker-compose.ymlの作成

次に、コンテナの起動設定を行う `docker-compose.yml` を作成します。
ここでのポイントは、\*\*「ホスト（Windows）とコンテナ（Linux）の環境分離」\*\*です。

```yaml:docker-compose.yml
services:
  app:
    build:
      context: .
      target: final
    restart: unless-stopped
    
    # ポートマッピング
    # 3000: Serverless Offline / 8000: FastAPI
    ports:
      - "3000:3000"
      - "8000:8000"
      
    # ボリューム設定（ここが重要！）
    volumes:
      # 1. コードの変更を即座にコンテナに反映（ホットリロード用）
      - .:/usr/src/app
      
      # 2. コンテナ内のライブラリをホスト側で上書きしないように保護
      # これがないと、Windows側の空フォルダで上書きされ、動かなくなります
      - /usr/src/app/node_modules
      
      # 3. Windows側のvenvをコンテナに読み込ませない
      # OSが違うため、WindowsのvenvはLinuxでは動きません
      - /usr/src/app/venv
      - /usr/src/app/__pycache__

    # AWS認証情報を環境変数として渡す
    env_file:
      - .env
```

#### 3\. 除外設定 (.dockerignore)

不要なファイルをコンテナに送らないよう、`.dockerignore` も設定します。特に `node_modules` と `venv` を除外することは、ビルド時間の短縮とエラー防止に必須です。

```text:.dockerignore
node_modules
venv
.git
.env
__pycache__
.serverless
```

これで、OSの差異に悩まされない堅牢な開発環境の定義が完了しました。

-----
