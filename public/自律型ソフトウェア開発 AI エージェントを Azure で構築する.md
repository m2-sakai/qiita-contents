---
title: 自律型ソフトウェア開発 AI エージェントを Azure で構築する
tags:
  - GitHub
  - Azure
  - AI
  - 個人開発
  - AIエージェント
private: false
updated_at: '2025-12-09T07:02:46+09:00'
id: 9767925d550f1c00e40e
organization_url_name: nri
slide: false
ignorePublish: false
---
# はじめに

今年もアドベントカレンダーの季節がやってきましたね！
突然ですが、皆さん AWS サンプルの `remote-swe-agents` はご存じでしょうか。
`remote-swe-agents`とは GitHub と連携した完全自律型のソフトウェア開発エージェントを AWS 上で構築できるものとなります。

https://aws.amazon.com/jp/cdp/ai-agent-independent-software/

https://github.com/aws-samples/remote-swe-agents

今年もアドベントカレンダーが始まるということで、例年通り何かアプリを開発したいと思ってネタを探していたわけですが、最近業務でその `remote-swe-agents` を動かしてみることがありました。

動かしてみて一言、 **「すごい」** です。ボキャブラリーがなくてすみません。

このシステム、何がすごいかというと、**チャットで「このバグを修正して」と指示すると、自律的に AI エージェントが GitHub のコードを読み込み、修正し、Pull Request まで作成してくれるのです。その裏では、セッションごとに EC2 インスタンスが自動で立ち上がり、エージェントが動作するという仕組み** です。

さらに驚いたのが、その基盤がすべて **AWS CDK（TypeScript）で表現されていて、リポジトリを Clone して少し設定をして `cdk deploy` を実行すれば、ものの数十分で基盤が構築できてしまう** 点です。

AWS サンプルの `remote-swe-agents` の基盤構成図は、ざっくり以下のようになっています（[GitHub](https://github.com/aws-samples/remote-swe-agents) から引用しています）。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/bce482d8-186d-4c91-a3b7-fd74c6878d43.png)

私のアカウントやいくつか記事をご覧になっている方々はご存知かと思いますが、私はかれこれ 3 年間ほど Azure を触っています。AWS サンプルを動かしながら、どうしても思ってしまうわけです。

**「この構成、Azure でも作れないかな！？」**

ということで、[NRI xPalette Advent Calendar 2025](https://qiita.com/advent-calendar/2025/nri-xpalette) シリーズ 1 の 9 日目は、**Azure で自律型ソフトウェア開発 AI エージェントを構築してみた奮闘記** となります。

:::note warn
この記事は、AWS の `remote-swe-agents` を Azure に移行する過程での奮闘記です。完全に移行できたわけではなく、一部機能は実装中です。また、コードの品質についても、**まずは動かすことを優先した「動いた」レベル**であることをご理解ください。リファクタリングの余地は大いにあります！
:::

:::note info
AWS の `remote-swe-agents` を開発された方々、そして素晴らしいサンプルを公開してくださり、ありがとうございます。このプロジェクトがなければ、このような挑戦はできませんでした。
:::

また、今回のソースコードは以下の GitHub に格納しておりますので、是非ご覧ください。

https://github.com/m2-sakai/remote-swe-agents-azure

# 作成したアーキテクチャ

今回 Azure で作成したアーキテクチャは以下となります。

![remote-swe-agent-azure.drawio.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/2e3397ff-5fc9-4b32-bc1f-2aeef2ed47ff.png)

AWS と比較すると、以下のように各サービスを Azure のサービスに対応させています。
CloudFront は Azure Front Door に対応しますが、今回は簡略化のため対象外としています。

| 役割               | AWS サンプル      | Azure リソース      | 備考                           |
| ------------------ | ----------------- | ------------------- | ------------------------------ |
| フロント・API      | Lambda            | App Service         | コンテナでデプロイ             |
| 認証               | Cognito           | Microsoft Entra ID  | OAuth 2.0 / MSAL               |
| データベース       | DynamoDB          | Cosmos DB for NoSQL | チャットやセッション情報の保管 |
| ストレージ         | S3                | Blob Storage        | Worker ソースコード配置        |
| LLM                | Bedrock           | OpenAI Service      | Microsoft Foundry で構築       |
| イベント配信       | EventBridge       | Web PubSub          | リアルタイムイベント通知       |
| VM 管理            | EC2               | Virtual Machines    | セッションごとに起動           |
| VM イメージ        | EC2 AMI           | Compute Gallery     | カスタムイメージの保管         |
| VM イメージビルド  | EC2 Image Builder | VM Image Builder    | テンプレートからイメージ作成   |
| コンテナレジストリ | ECR               | Container Registry  | Docker イメージ管理            |
| シークレット管理   | Systems Manager   | Key Vault           | パスワード等の秘密情報         |
| ネットワーク       | VPC               | Virtual Network     |                                |

# どんなアプリに仕上がったか

本題に入る前に、どんなアプリに仕上がったか簡単に紹介します。

今回、ロジックは極力変更せずに Azure で構築したため、アプリケーションの UI やデザインは元の AWS サンプルとほぼ同じです。ただ、動きは変わらないものの、裏側のインフラはすべて Azure のサービスに置き換わっています。

**基本的な使い方は以下の通りです：**

1. **ログイン**: Microsoft Entra ID で認証を行います
2. **チャット開始**: 「○○ のバグを修正して」などと指示します
3. **自動処理**: バックグラウンドで VM が立ち上がり、AI エージェントが動作を開始します
4. **GitHub 連携**: エージェントがコードを読み込み、修正し、Pull Request を作成します
5. **リアルタイム通知**: Azure Web PubSub を通じて、状況がリアルタイムで画面に反映されます

実際に動かしてみた際の様子はこちらになります。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/9598564d-3730-4a44-bd8e-a8e707b79a33.png)

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/61e5dd9c-fe31-4b84-9476-5e56dfb331b7.png)

# 開発環境

今回開発に使用した環境は以下の通りです。

| 項目       | バージョン / 内容 |
| ---------- | ----------------- |
| OS         | Windows 11 + WSL2 |
| Node.js    | v22.19.0          |
| TypeScript | 5.7.3             |
| Next.js    | 15.3.3            |
| Azure CLI  | 2.75.0            |
| Bicep CLI  | 0.39.26           |

# ディレクトリ構成

今回は AWS サンプルの構成を踏襲しつつ、Azure に合わせた構成としています。
最も大きな違いは、**基盤構築が AWS CDK から Bicep に変わっている**点です。

```text
remote-swe-agents-azure/
├── packages/
│   ├── agent-core/          # 共通ロジック（Cosmos DB、Blob Storage等）
│   ├── webapp/              # Next.jsフロントエンド（App Service）
│   └── worker/              # AIエージェント処理（Azure VM）
├── bicep/
│   ├── templates/
│   │   └── main.bicep       # メインテンプレート
│   ├── parameters/
│   │   └── main.bicepparam  # パラメータファイル
│   └── modules/             # 各リソースのモジュール
│       ├── app-service/
│       ├── cosmos-db/
│       ├── cognitive-services/
│       ├── key-vault/
│       ├── web-pubsub/
│       └── ... (その他)
├── docker/
│   └── webapp.Dockerfile    # WebAppコンテナ用
└── README.md
```

# 基盤構築編

それではまず、上記アーキテクチャに沿って基盤を構築していきます。

## 移行するにあたっての設計ポイント

AWS サンプルから Azure に移行するにあたり、特に重要だったポイントをいくつかピックアップして紹介します。

### アプリケーション

今回、Azure App Service に Next.js アプリを Docker コンテナとしてデプロイしています。

https://learn.microsoft.com/ja-jp/azure/app-service/overview

構成図でいうと以下の赤枠の部分です。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/2fe84b72-f2e6-4a3e-b0c8-a01b250f2919.png)

**ポイントは以下の通りです：**

- アプリケーションは **Azure Container Registry** にイメージを格納し、デプロイする
- **VNet 統合** により、Private Endpoint で保護されたリソース（Cosmos DB、Key Vault 等）にアクセス可能となる
- Key Vault からシークレット（API キー等）を **環境変数** として注入する`@Microsoft.KeyVault(VaultName=xxx;SecretName=yyy)`

Bicep での実装のポイントとしては、App Service の `linuxFxVersion` に `DOCKER|<レジストリ>.azurecr.io/<イメージ名>:<タグ>` 形式で指定することで、コンテナに対応した形で App Service を作成できることです。以下にその部分だけ抽出した例を示します。

```bicep: bicep/modules/app-service/app_module.bicep
resource appService 'Microsoft.Web/sites@2024-11-01' = {
  name: appServiceName
  kind: 'app,linux,container'
  properties: {
    siteConfig: {
      linuxFxVersion: 'DOCKER|<レジストリ>.azurecr.io/<イメージ名>:<タグ>'
    }
  }
}
```

今回作成した `bicep/modules/app-service/app_module.bicep` の全量はこちらです。パラメータは [bicep/parameters/main.bicepparam](https://github.com/m2-sakai/remote-swe-agents-azure/blob/main/bicep/parameters/main.bicepparam) を参照ください。

https://github.com/m2-sakai/remote-swe-agents-azure/blob/main/bicep/modules/app-service/app_module.bicep

:::note info
App Service には **そのままビルドした Next.js アプリをデプロイすることもできます**が、今回は AWS サンプルを踏襲してコンテナでデプロイしています。
個人的にはコンテナの方が**簡単かつ他のアーキテクチャに応用が利く**と感じました。

そのままデプロイする方法は [Azure App Service に Next.js をデプロイする方法](https://zenn.dev/keit0728/articles/3e6310f8e06963) などの記事を参照ください。
:::

### AI エージェント動作環境

Worker が動作する環境は、AWS サンプルと同じく VM を利用します。

https://learn.microsoft.com/ja-jp/azure/virtual-machines/

構成図でいうと以下の赤枠の部分です。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/d5d33ae3-f0aa-4039-9542-30fe70e1d948.png)

**以下の流れで作成・起動します：**

1. **Image Template** を作成する
   - Ubuntu 24.04 ベース
   - VM に必要なライブラリやツール（GitHub CLI, Azure CLI など）をインストール
   - 起動時スクリプトとして、以下のようにしておく
     - Worker ソースコード（tar.gz）を Blob Storage からダウンロード
     - `npx tsx src/main.ts` で Worker 起動
   - systemd サービスとして自動起動
2. **Azure Image Builder** でカスタムイメージをビルドし、**Compute Gallery** に保存する
   ```bash: Terminal
   az image builder run -g <リソースグループ名> -n <イメージテンプレート名>
   ```
3. **WebApp から VM を起動する**
   - Azure SDK を使用し、WebApp から VM を起動する
   - 作成時にユーザー割り当てマネージド ID を付与し、アクセス権限を付与する

実装例を紹介します。ここで肝となるのは、**Image Template で指定したセットアップスクリプト**と、**WebApp から VM を起動する部分** です。
本章では基盤構築部分のため、Image Template のセットアップスクリプトを紹介します。このスクリプトをイメージ作成時に実行しています。

https://github.com/m2-sakai/remote-swe-agents-azure/blob/main/bicep/parameters/main.bicepparam#L337-L533

:::note info
**実装における工夫点やハマりポイント**：

- VM イメージビルド時に Worker のソースコードをイメージに含めようとしましたが、Image Builder の実行環境ではユーザー割り当てマネージド ID が利用できず、Blob Storage へのアクセスができませんでした。そこで、**ソースコードは起動時にダウンロードする**方式に変更しています。
- ソースコードの更新検知に **ETag** を使用し、Blob Storage 上のファイルが更新されていればダウンロード・ビルドし直す仕組みを実装しています。

:::

### イベント処理

AWS では EventBridge でイベントを配信していましたが、Azure では同様のサービスとして、**Azure Web PubSub** を採用しました。

https://learn.microsoft.com/ja-jp/azure/azure-web-pubsub/overview

構成図でいうと以下の赤枠の部分です。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/042ca24a-8e23-4a34-aaa9-9c294fac9704.png)

**リアルタイムのイベント通信をこのサービスで実現**しており、仕組みは以下の通りです。

- WebApp から Worker へのイベント通知（メッセージ受信、停止命令等）
- Worker から WebApp へのステータス通知（処理状況、完了通知等）
- ユーザー割り当てマネージド ID で認証し、トークンを取得
- WebSocket 接続で双方向通信

Azure Web PubSub では、**グループ**という概念を使って Worker 固有のチャンネルを作成し、特定の Worker にのみメッセージを配信できるようにしています。

```typescript
// Worker側：グループに参加
await webPubSubClient.joinGroup(`worker/${workerId}`);

// WebApp側：特定のWorkerにメッセージ送信
await serviceClient.sendToGroup(webPubSubHub, `worker/${workerId}`, event);
```

### AI

AWS では Amazon Bedrock（Claude）を使用していましたが、Azure では Azure OpenAI Service に変更しました。また、OpenAI は Microsoft Foundry 上に構築しています。

https://learn.microsoft.com/ja-jp/azure/ai-foundry/what-is-azure-ai-foundry?view=foundry&preserve-view=true

構成図でいうと以下の赤枠の部分です。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/fc75d5b1-3c02-4006-9615-2c2b9139b414.png)

**今回使用したモデル**：

- GPT-4o
- GPT-4.1
- GPT-5-mini
- o4-mini

モデルは`bicep/parameters/main.bicepparam`で配列定義し、Bicep でループ処理してデプロイメントを作成しています。

https://github.com/m2-sakai/remote-swe-agents-azure/blob/main/bicep/parameters/main.bicepparam#L304-L325

### データベース

データベースは AWS でも NoSQL であるため、Cosmos DB for NoSQL を採用しました。

https://learn.microsoft.com/ja-jp/azure/cosmos-db/nosql/

構成図でいうと以下の赤枠の部分です。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/1e02508b-e2c4-4996-8f87-1c2bb54f9fd1.png)

**パーティションキー設計**：

DynamoDB では `PK`（Partition Key）と `SK`（Sort Key）の 2 つでアイテムを管理していましたが、Cosmos DB では**パーティションキーは 1 つのみ**としました。そのため、`PK` をパーティションキーとし、`SK` は通常のプロパティとして格納しています（そのため、コンテナは一つだけになっています）。

https://github.com/m2-sakai/remote-swe-agents-azure/blob/main/bicep/parameters/main.bicepparam#L274-L278

**データ構造例**：

| アイテム種類   | PK                   | SK            | 説明                   |
| -------------- | -------------------- | ------------- | ---------------------- |
| セッション     | `sessions`           | `{workerId}`  | チャットセッション情報 |
| メッセージ     | `message-{workerId}` | `{timestamp}` | 会話履歴               |
| トークン使用量 | `token-{workerId}`   | `{modelId}`   | モデル別トークン使用量 |
| 認証セッション | `auth-sessions`      | `{sessionId}` | ログインセッション     |

## 基盤デプロイ

それでは、こちらの構成を Bicep で構築します。
Bicep のソースコードの構成は、以前書いた記事の設計思想に従って作成しています。

https://qiita.com/s_w_high/items/e67dc796401e2694c5f3

:::note warn

ここからの記載は、実際のパラメータ名に沿って記載をしています。
どのパラメータのことかわからない場合は、パラメータファイル： [bicep/parameters/main.bicepparam](https://github.com/m2-sakai/remote-swe-agents-azure/blob/main/bicep/parameters/main.bicepparam) を参照してください。

:::

### 事前準備

まず、Azure 環境にリソースグループを作成し、デプロイに必要な情報を集めます。

```bash
# リソースグループ作成
az group create --name m2-sakai-je-RG-01 --location japaneast

# サブスクリプションIDを取得
az account show --query id -o tsv
```

また、ユーザーを認証するために、**Microsoft Entra ID のアプリケーション登録を行い、クライアント ID とシークレットを取得しておきます**。

### Bicep コードの構築

ここまでの記載でソースコードの例として何度か示しておりますが、今回は以下のファイルでインフラを定義しています。

- **テンプレート：** [bicep/templates/main.bicep](https://github.com/m2-sakai/remote-swe-agents-azure/blob/main/bicep/templates/main.bicep)
- **パラメータ：** [bicep/parameters/main.bicepparam](https://github.com/m2-sakai/remote-swe-agents-azure/blob/main/bicep/parameters/main.bicepparam)

**main.bicep のポイントは以下となります**：

- モジュール化されたリソース定義を組み合わせて、すべてのインフラを構築する
- リソースの作成順序を考慮し、`dependsOn` 句で依存関係を明示的に記述する
- リソースに紐づくもの（権限, Private Endpoint）はまとめて記載して可読性を向上させる

例えば、Cosmos DB 関連のリソースは以下のように呼び出しています。
（Cosmos DB アカウント → Private Endpoint → 権限 → データベース → コンテナー）

https://github.com/m2-sakai/remote-swe-agents-azure/blob/main/bicep/templates/main.bicep#L328-L405

### デプロイ

今回、**Azure デプロイスタック** を使用してデプロイします。
デプロイスタックは、リソースのライフサイクルを一元管理できる機能で、スタックから削除されたリソースを自動的に削除することができます。

https://learn.microsoft.com/ja-jp/azure/azure-resource-manager/bicep/deployment-stacks?tabs=azure-powershell

以下のような `deploy-stack.sh` を作成し、上記の Bicep を実行するのみです。
今回はオプションとして以下のように設定しました。

- **`--action-on-unmanage 'deleteResources'`**：管理されなくなったリソースは削除する
- **`--deny-settings-mode 'denyDelete'`**：作成したリソースは削除不可（編集可能）

https://github.com/m2-sakai/remote-swe-agents-azure/blob/main/bicep/deploy-stack.sh

これらをデプロイすると以下のように一度にリソースが作成されます。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/d2205404-43ef-4e29-be2c-790dab0db609.png)

### カスタムイメージを作成

デプロイが完了したら、**Azure Image Builder** でカスタムイメージをビルドします。

```bash: Terminal
# イメージテンプレートを実行
az image builder run \
  --name m2-sakai-je-worker-template-01 \
  --resource-group m2-sakai-je-RG-01
```

ビルドには 30〜40 分程度かかります（正直こんなに時間がかかるのか、と感じました）。
完了すると、事前に作成していた Compute Gallery にイメージが保存されます。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/3020929a-2f3c-4eda-8d98-bc9b504da297.png)

### Worker のソースコードを配置

Worker インスタンス（VM）は起動時に実行するソースコードを Blob Storage から取得する設計にしています。Worker（`packages/worker`）と agent-core（`packages/agent-core`）を tar.gz に固めて、Blob Storage にアップロードします。

```bash: Terminal
# ワークスペースのルートで実行
tar -czf source.tar.gz packages/agent-core packages/worker package.json package-lock.json

# Azure CLIでアップロード
az storage blob upload \
  --account-name m2sakaijestorage01 \
  --container-name worker-source \
  --name source.tar.gz \
  --file source.tar.gz \
  --auth-mode login
```

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/b9bc18c3-f085-4e59-9940-ee84eca63ab7.png)

### Key Vault にシークレットを登録する

最後に、Key Vault に必要なシークレットを登録します。今回は以下の 3 つを登録します。

```bash: Terminal
# 0の準備で作成したアプリケーションのクライアントシークレット
az keyvault secret set \
  --vault-name m2-sakai-je-KV-01 \
  --name AzureAdClientSecret \
  --value "your-client-secret"

# VMの管理者パスワード
az keyvault secret set \
  --vault-name m2-sakai-je-KV-01 \
  --name VmAdminPassword \
  --value "your-strong-password"

# Azure Container Registryのパスワード
az keyvault secret set \
  --vault-name m2-sakai-je-KV-01 \
  --name AcrAdminPassword \
  --value "your-acr-password"
```

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/883c1493-1ff5-4608-88a6-e309883489d6.png)

これで基盤構築は完了です！お疲れさまでした！！！

# アプリ実装編

基盤ができたので、いよいよアプリケーションを実装していきます。
先ほど作成した Azure の各種リソースを使って、実際のアプリケーションコードを実装していきます。ここも一部ポイントとなっている部分を中心に紹介します。

## 共通部品（`agent-core`）

`agent-core` パッケージは、WebApp と Worker の両方から使用される共通ロジックです。

### AWS SDK から Azure SDK への置き換え

AWS サンプルのコードを Azure に移行するにあたり、以下のように SDK を置き換えました。ここでもすべては紹介できないため、一部主要な SDK を以下に記載します。

| 用途               | AWS（一つだけ抽出）               | Azure （一つだけ抽出）     |
| ------------------ | --------------------------------- | -------------------------- |
| NoSQL データベース | `@aws-sdk/lib-dynamodb`           | `@azure/cosmos`            |
| ストレージ         | `@aws-sdk/client-s3`              | `@azure/storage-blob`      |
| LLM                | `@aws-sdk/client-bedrock-runtime` | `openai`                   |
| 認証               | `aws-amplify`                     | `@azure/msal-node`         |
| VM 管理            | `@aws-sdk/client-ec2`             | `@azure/arm-compute`       |
| リアルタイム通信   | `@aws-sdk/client-eventbridge`     | `@azure/web-pubsub-client` |

### Cosmos DB クライアントの実装

基盤構築編で作成した Cosmos DB にアクセスするためのクライアントを実装します。
実装の全体像はこちらとなっています。

https://github.com/m2-sakai/remote-swe-agents-azure/blob/main/packages/agent-core/src/lib/azure/cosmos.ts

**実装におけるポイントは以下の通りです**：

- 認証はマネージド ID（今回はユーザー割り当て）を使用する
- 一度初期化したクライアントは再利用することで性能を高める
- AWS サンプルのコードを極力変更しないため、DynamoDB 互換の関数を実装する

この Cosmos DB の実装と同じように、Blob Storage や Key Vault の接続クライアントを作成しています。

### イベント処理

WebApp と Worker 間のリアルタイム通信も `agent-core` に実装しています。
実装の全体像はこちらとなります。

https://github.com/m2-sakai/remote-swe-agents-azure/blob/main/packages/agent-core/src/lib/events.ts

**実装におけるポイントは以下の通りです**：

- Managed Identity でトークンを取得し、Web PubSub REST API を呼び出す
- `channel` プロパティで送信先を指定する（`worker/{workerId}` など）
- WebApp へのイベント送信、Worker へのイベント送信を関数化する

### Worker インスタンス管理

チャットでメッセージを送信すると、バックグラウンドで VM を起動する必要があります。
この VM 起動・管理ロジックも `agent-core` に実装しており、`@azure/arm-compute` SDK を使っています。
実装の全体像はこちらとなります。

https://github.com/m2-sakai/remote-swe-agents-azure/blob/main/packages/agent-core/src/lib/worker-manager.ts

**VM 起動の流れは以下の通りです**：

1. セッション情報から `workerId` を取得する
2. **VM 名に `workerId` を含める**（`worker-{workerId}` 形式）
3. **タグに `RemoteSweWorkerId` として `workerId` を設定**（VM 内から取得できるようにする）
4. Compute Gallery に保存したカスタムイメージを指定し、VM を作成して起動を待つ

**実装におけるポイントは以下の通りです**：

- Managed Identity（ユーザー割り当て）を VM にアタッチする
- タグで `workerId` を管理する（VM 内から IMDS で取得可能）

## チャット画面（`webapp`）

チャット画面は Next.js 15 (App Router) で構築された Web アプリケーションです。
App Service にコンテナとしてデプロイします。

### 認証の実装（Cognito → Entra ID）

AWS の Cognito から Microsoft Entra ID への移行が、WebApp における大きな変更点です。

**実装のポイントは以下の通りです**：

1. **MSAL（Microsoft Authentication Library）を使用**
2. **トークンが大きすぎて Cookie に入らない問題**を Cosmos DB に保存することで解決
3. **開発環境では認証をスキップ**できる仕組み

実装の全体像はこちらとなります。

https://github.com/m2-sakai/remote-swe-agents-azure/blob/main/packages/webapp/src/lib/auth.ts

**具体的な認証フローは以下の通りです**：

1. ユーザーがログインボタンをクリックする
2. Microsoft Entra ID の認証画面にリダイレクトする
3. 認証成功後、コールバック URL に戻る
4. 取得したトークンを Cosmos DB に保存する
5. セッション ID のみを Cookie に保存する

これにより、Cookie サイズの問題を回避しつつ、セキュアな認証を実現しています。

### agent-core の機能を呼び出し

WebApp は、`agent-core` パッケージの機能を呼び出すことで、VM 起動やイベント送信を実現しています。一部の例を以下に示します。

```typescript
// agent-coreから関数をインポート
import { startWorker } from '@remote-swe-agents-azure/agent-core/lib/worker-manager';
import { sendWorkerEvent } from '@remote-swe-agents-azure/agent-core/lib/events';
import {
  getSession,
  updateSession,
} from '@remote-swe-agents-azure/agent-core/lib/sessions';

// VM起動
await startWorker(workerId);

// Workerにイベント送信
await sendWorkerEvent(workerId, {
  type: 'onMessageReceived',
  // ...
});

// セッション取得・更新
const session = await getSession(workerId);
await updateSession(workerId, { status: 'running' });
```

### Docker イメージのビルド＆デプロイ

WebApp のデプロイは、以下の手順で行います：

```bash: Terminal
# 1. ACRにログイン
az acr login --name m2sakaijeacr01

# 2. Dockerイメージをビルド
docker build \
  -f docker/webapp.Dockerfile \
  -t m2sakaijeacr01.azurecr.io/remote-swe-agent-azure-webapp:<バージョンタグ> \
  .

# 3. ACRにプッシュ
docker push m2sakaijeacr01.azurecr.io/remote-swe-agent-azure-webapp:<バージョンタグ>
```

Azure App Service では「デプロイセンター」から、レジストリとイメージタグを指定できます。設定して保存すると自動で再起動され、最新のイメージが Pull されます。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/98bd7a4e-5ab4-4098-881f-8c1fca287c54.png)

## AI エージェント処理（`worker`）

VM 上で動作する AI エージェントのロジックです。
起動時に Blob Storage からソースコードをダウンロードし、systemd サービスとして実行されます。

### VM 情報の取得（EC2 → Azure IMDS）

Worker は VM 上で動作するため、自分自身の情報を取得する必要があります。
AWS では EC2 メタデータサービスを使用していましたが、Azure では **Instance Metadata Service (IMDS)** を使用します。実装の全体像はこちらです。

https://github.com/m2-sakai/remote-swe-agents-azure/blob/main/packages/worker/src/common/azure-vm.ts

**実装におけるポイントは以下の通りです**：

- IMDS は `169.254.169.254` という特別な IP アドレスでアクセスする
- リソースグループ名や VM 名などのメタデータを取得可能
- タグ情報も取得できるため、`workerId` もここから取得可能

### Web PubSub クライアント

Worker も Web PubSub に接続し、WebApp からのイベントを受信します。
**グループ機能**を使って、特定の Worker にのみメッセージを配信しています。実装の全体像はこちらです。

https://github.com/m2-sakai/remote-swe-agents-azure/blob/main/packages/worker/src/entry.ts

**イベント処理の仕組みは以下の通りです**：

1. Worker は起動時に `worker/{workerId}` グループに参加する
2. WebApp は Worker にメッセージを送りたい場合、そのグループにブロードキャストする
3. Worker はブロードキャストメッセージを受信し、`channel` プロパティで自分宛か判定
4. 自分のセッション宛のメッセージのみ処理する

### エージェントループの実装

エージェントのメインロジックは、AWS サンプルとほぼ同じです。

https://github.com/m2-sakai/remote-swe-agents-azure/blob/main/packages/worker/src/agent/index.ts

**おおまかな流れは以下の通りです**：

1. Cosmos DB からメッセージ履歴を取得する
2. Azure OpenAI Service にリクエストを行い、回答を取得する（GPT-4o など）
3. Tool Call があれば実行する（GitHub へのアクセス、ファイル操作など）
4. 回答結果を Cosmos DB に保存する
5. 回答結果を WebApp に通知する（Web PubSub 経由）

このループが自律的に回り続けることで、「指示 → 分析 → 実装 → PR 作成」という一連の流れが自動化されています。

# 動作確認

## テスト用リポジトリの作成

まず、テスト用の GitHub リポジトリを用意します。
今回は、React で作成したシンプルな Web アプリケーションを用意しました（`npm create vite` で作成したデフォルトのアプリです）。

https://github.com/m2-sakai/react-sample

そして、GitHub の Issue に以下のような内容を作成しておきます。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/34d4376c-8e4d-41a4-8d83-b0071b0cca54.png)

## ログイン

それでは、App Service の URL にアクセスし `Microsoft Entra IDでログイン` ボタンを押下します。すると Microsoft の認証画面が表示され、認証が成功するとトップ画面が表示されることが確認できました！

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/3bef90c5-5829-4d9d-8ba6-658d1df2c7df.png)

## セッション作成 → エージェント回答

チャット画面で以下のように入力し、`Create Session` ボタンを押下します。

:::note info

GitHub 以下のリポジトリについて修正を行い、Pull Request を作成してください。

- リポジトリ：https://github.com/m2-sakai/react-sample
- 要件：Issue #1
- ブランチ：feature/issue-1

:::

すると、以下のようにチャットが表示され AI エージェントから回答が返ってきました！
要望通り、しっかり GitHub にアクセスして Pull Request を作成してくれたみたいですね。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/2496d61e-4613-40d1-a1cf-c65cccf3b164.png)

## Pull Request の確認

それでは最後に GitHub で Pull Request が作成されているか確認します。
リンクをクリックして GitHub を確認すると...

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/2817ec91-085e-4a85-b229-82775c177d68.png)

**ちゃんと Pull Request が作成されています！**
しっかり Issue の内容通りに、日本語版の README を作成してくれました。

こんな形で様々な Issue を AI が処理して自動で Pull Request を上げてくれるので、放っておいてもどんどん開発が進められますね！

# おわりに

本記事では AWS サンプルの `remote-swe-agents` を Azure に移行した話をお届けしました。
はじめは移行するだけ、と高を括っていましたが、予想以上に大変でした。同じような機能を持つリソースは存在するものの、微妙に使い勝手が異なったり SDK も異なるため、一から作った部分も多かったと感じています。
まだまだ移行できていない機能もたくさんありますし、改めて `remote-swe-agents` のすごさを実感しました。
とはいえ、最終的に動作するものが作れて本当に良かったと思っています。

弊社のアドベントカレンダーはまだまだ続きますので、ぜひこれからも読んでいただけると幸いです！

https://qiita.com/advent-calendar/2025/nri-xpalette

最後まで読んでいただき、ありがとうございました！
