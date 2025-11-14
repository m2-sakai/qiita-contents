---
title: Azure MCP Server で Azure の リソースを AI エージェントから操作できるようになった！！
tags:
  - Azure
  - AI
  - MCP
  - AIエージェント
  - MCPサーバー
private: false
updated_at: '2025-04-23T07:11:19+09:00'
id: aafbedfcc3c6bcadc6da
organization_url_name: nri
slide: false
ignorePublish: false
---
# はじめに

4/17 に Azure MCP Server のパブリックプレビュー版が公開されましたね。
AWS も先日 [AWS MCP Server](https://awslabs.github.io/mcp/) が公開され、様々なクラウドや企業が MCP サーバーを公開している流れが来ているな、と感じています。

https://devblogs.microsoft.com/azure-sdk/introducing-the-azure-mcp-server/

そこで今回はこのパブリックプレビュー版の Azure MCP Server を触ってみようと思います。どんなことができるのか楽しみです。

:::note warn

この Azure MCP Server はパブリック プレビュー段階のため、**今後実装が大幅に変更される可能性があることにご注意ください。**

:::

# Azure MCP Server の導入

それでは、Azure MCP Server を導入してみようと思います。動作環境は VS Code + GitHub Copilot を用いています。
まずは、[GitHub](https://github.com/Azure/azure-mcp/) の「Install Azure MCP Server」をクリックします。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/e73a3643-957a-4aef-a5a0-fee6f73c0305.png)

「Visual Studio Code を開く」を押下すると以下のように表示されますので、「Install Server」を押下します。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/d184b98b-1c02-4ec7-9074-5bc32f6dd4f8.png)

すると、VS Code の設定ファイルである `settings.json` に以下のように Azure MCP Server の設定が追加されます。

```json: settings.json
{
    "servers": {
        "Azure MCP Server": {
            "command": "npx",
            "args": ["-y", "@azure/mcp@latest", "server", "start"]
        }
    }
}
```

:::note warn

`npx` コマンドが表示されているため、お察しの方もいるかと思いますが、**Azure MCP Server の導入には Node.js が必要です。**
インストールされていない場合は、[こちら](https://docs.npmjs.com/downloading-and-installing-node-js-and-npm)の手順に従ってインストールしてください。

:::

GitHub Copilot を「agent」モードに変更すると、以下のように Azure MCP Server が起動できていることがわかります。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/a4dfd4f4-b546-46d1-b6eb-a0d46ec7dd96.png)

これで、Azure MCP Server の導入は以上です。非常に簡単ですね！！

:::note info

Azure MCP Server をローカル環境で立ち上げる方法もあります。
その場合は以下のコマンドで Azure MCP Server を起動し、VS Code で SSE 設定値を追加することで使用できるようになります。

```bash: Terminal
npx -y @azure/mcp@latest server start --transport sse
```

```json: settings.json
{
    "servers": {
        "Azure MCP Server": {
            "type": "sse",
            "url": "http://localhost:5008/sse"
        }
    }
}
```

:::

# Azure MCP Server でできること（2025/4/23 現在）

Azure MCP Server は現在、以下の機能がサポートされています（[GitHub](https://github.com/Azure/azure-mcp/)から引用しております）。
これを見ると、Cosmos DB アカウントの一覧を取得できたり Azure CLI が実行できたりと、Azure のリソースの操作できるようです。

| リソース                                | 操作                                                                                 |
| --------------------------------------- | ------------------------------------------------------------------------------------ |
| **Azure Cosmos DB (NoSQL Databases)**   | List Cosmos DB accounts                                                              |
|                                         | List and query databases                                                             |
|                                         | Manage containers and items                                                          |
|                                         | Execute SQL queries against containers                                               |
| **Azure Storage**                       | List Storage accounts                                                                |
|                                         | Manage blob containers and blobs                                                     |
|                                         | List and query Storage tables                                                        |
|                                         | Get container properties and metadata                                                |
| **Azure Monitor (Log Analytics)**       | List Log Analytics workspaces                                                        |
|                                         | Query logs using KQL                                                                 |
|                                         | List available tables                                                                |
|                                         | Configure monitoring options                                                         |
| **Azure App Configuration**             | List App Configuration stores                                                        |
|                                         | Manage key-value pairs                                                               |
|                                         | Handle labeled configurations                                                        |
|                                         | Lock/unlock configuration settings                                                   |
| **Azure Resource Groups**               | List resource groups                                                                 |
|                                         | Resource group management operations                                                 |
| **Azure CLI Extension**                 | Execute Azure CLI commands directly                                                  |
|                                         | Support for all Azure CLI functionality                                              |
|                                         | JSON output formatting                                                               |
|                                         | Cross-platform compatibility                                                         |
| **Azure Developer CLI (azd) Extension** | Execute Azure Developer CLI commands directly                                        |
|                                         | Support for template discovery, template initialization, provisioning and deployment |
|                                         | Cross-platform compatibility                                                         |

また、Azure MCP Command は以下にリファレンスがあるので合わせてご参照ください。

https://github.com/Azure/azure-mcp/blob/main/docs/azmcp-commands.md

# Azure MCP Server を使ってリソースを操作する

それではできること一覧が確認できたので、その中でいくつか動かしてみたいと思います。今回は以下を実行してみます。

- [リソースグループの一覧を表示する](#リソースグループの一覧を表示する)
- [ストレージアカウントの一覧を表示する](#ストレージアカウントの一覧を表示する)

## リソースグループの一覧を表示する

実際にリソースグループの一覧を取得してみようと思います。以下のように質問してみました。

:::note info

Azure のリソースグループの一覧を取得して。回答は日本語でお願いします。

:::

すると、以下が実行され、見事リソースグループ一覧が取得できました！

- `@azure get auth state`：Azure への認証状態の確認
- `@azure get selected subscription`：選択されているサブスクリプション情報を確認
- `azmcp-extension-az`：リソースグループの一覧を取得

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/159dc79f-9d4a-4978-b9f0-f72156fcfba4.png)

:::note warn

初めて使用する方は、GitHub Copilot for Azure にログインするように促される場合があります。
その場合はポップアップに従ってログインしてください。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/89b917f4-75a3-4a77-9fc2-61c938f7ce9f.png)

:::

## ストレージアカウントの一覧を表示する

次に、ストレージアカウントの一覧を取得してみようと思います。私の環境では、`Gen-m2sakai-je-RG-01` というリソースグループに `genm2sakaist01` というストレージアカウントを作成しています。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/f370ad5d-fb66-4d82-b9a2-89ab124a104b.png)

それでは以下のように質問してみます。

:::note info

ストレージアカウントの一覧を取得してください。

:::

すると、先程と同様に以下が実行され、一度認証に失敗したものの見事作成されたストレージアカウントが取得できました。

- `@azure get auth state`：Azure への認証状態の確認
- `@azure get selected subscription`：選択されているサブスクリプション情報を確認
- `azmcp-storage-account-list` or `azmcp-extension-az`：ストレージアカウントの一覧を取得

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/5a5e4bb0-7dc3-4b32-9ded-f9398679487f.png)

# おわりに

いかがでしたでしょうか。
本記事では、先日パブリックプレビュー版として公開された Azure MCP Server を触ってみました。
少し実行に時間がかかるものの、AI エージェントから Azure リソースを操作することができるのは面白いですね。これからの機能拡張に期待したいと思います！
AI 周りのツールがどんどんと登場し、キャッチアップが大変なところですが、一緒に頑張っていきましょう。

最後まで読んでいただき、ありがとうございました。
