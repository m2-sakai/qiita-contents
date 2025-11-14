---
title: >-
  もうコードは書かなくていい？Microsoft Learn Docs MCP Server × GitHub Copilot for Azure で
  Azure での開発を自然言語で完全自動化する
tags:
  - Azure
  - AI
  - MCP
  - mslearn
  - githubcopilot
private: false
updated_at: '2025-11-14T15:50:17+09:00'
id: b33311d9baf1a835c27d
organization_url_name: nri
slide: false
ignorePublish: false
---

# はじめに

先週今までパブリックプレビュー版だった GitHub Copilot for Azure が GA されましたね。

https://devblogs.microsoft.com/all-things-azure/announcing-general-availability-of-github-copilot-for-azure-now-with-agent-mode/

GitHub Copilot for Azure は AI を活用した開発アシスタントとして **Visual Studio Code から直接 Azure 上でアプリケーションを構築・管理することができます。**
普段業務で Azure を活用している身としては非常に嬉しい内容でした。

また、6 月中旬には **MicroSoft Learn のドキュメント情報を LLM に付加できる** Microsoft Learn Docs MCP Server も公開されました。
どうしても LLM が学習している内容しか今まで返してくれなかったところから、Microsoft Learn Docs MCP Server を使うことで、**最新のドキュメント情報も** LLM から返してくれるようになったということです。こちらも非常に嬉しいですね。

https://github.com/MicrosoftDocs/mcp

::: note warn

**Microsoft Learn Docs MCP Server は現在パブリックプレビュー版です。**
一般公開前に実装が大幅に変更される可能性があることにご注意ください。

:::

これらの登場により、GitHub Copilot 上で Azure での開発体験が大幅に上昇したのではないかと感じています。

そこで今回は、この**GitHub Copilot for Azure** と **Microsoft Learn Docs MCP Server** を活用して、自然言語ベースで Azure Functions を作成できるのか試してみたいと思います。

::: note info
**MCP とは**

**MCP（Model Context Protocol）** は、近年注目を集める大規模言語モデル（LLM）がデータソースや外部ツールとやり取りするための、新しいオープン標準プロトコルです。
この登場により生成 AI モデルに文脈情報を渡しやすくなりました。**生成 AI 界の USB-C** とも言われていますね。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/f744c183-3e73-4543-843f-7550eb7d4145.png)

MCP に関する詳しい説明は本記事では割愛させていただきますので、MCP について知りたい方は以下のような記事や資料から学んでいただけるとよいかと思います。

- https://speakerdeck.com/minorun365/yasasiimcpru-men
- https://qiita.com/syukan3/items/f74b30240eaf31cb2686

:::

# セットアップ

それでは、**GitHub Copilot for Azure** と **Microsoft Learn Docs MCP Server** のセットアップから始めていきます。

## GitHub Copilot for Azure のセットアップ

まずは GitHub Copilot for Azure のセットアップから入ります。
[GA されたブログ](https://devblogs.microsoft.com/all-things-azure/announcing-general-availability-of-github-copilot-for-azure-now-with-agent-mode/)にも始め方は書いてありますが、こちらにより詳細を記載します。

### Visual Stidio Code の拡張機能としてインストールする

GitHub Copilot for Azure は Visual Studio Code（以下 VS Code）の拡張機能としてインストールすることができます。

https://marketplace.visualstudio.com/items?itemName=ms-azuretools.vscode-azure-github-copilot

VS Code を開き、Extension から「GitHub Copilot for Azure」を検索しインストールします。もし GitHub Copilot の拡張機能が入っていない方は一緒にインストールしておきます。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/d084e399-0a67-4c32-99aa-25b81a1c75be.png)

### 試しに触ってみる

それではインストールした GitHub Copilot for Azure を触ってみるため、VS Code で GitHub Copilot を開きます。（Windows の場合は `ctrl + shift + i` で開けます）

GitHub Copilot for Azure は `@azure` と宣言することで使用することができます。試しにリソースグループの一覧を表示してもらいました。

```
@azure リソースグループの一覧を表示して
```

すると、以下のように Azure テナントのリソースグループの情報が取得できました。

::: note warn

GitHub Copilot for Azure を初めて使用する方や Azure に事前にログインしていない方は、**初回に認証を求められる場合があります。**

:::

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/6489b61f-056f-4edd-a646-61d7c1628bf9.png)

また、デフォルトでは ask モードとなりますが、Agent モードでの使用もできるようになっています。以下のように **Agent モード** に切り替えて試してみます。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/41ed5818-6898-4f0d-8733-7a5703351e22.png)

Tools の設定から、GitHub Copilot for Azure の機能にチェックがついていることを確認しておきます。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/41a75b0a-f47c-47a7-b8c9-49e810b83ac8.png)

同じようにリソースグループの一覧を表示するような依頼をチャットすると中で`@azure`が実行されており、先程と同じようにリソースグループの一覧が表示されました。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/bf67bc02-311d-4d52-b536-0890a2539da2.png)

このように GitHub Copilot for Azure をインストールすることで、GitHub Copilot から Azure のリソースを操作することができることが確認できました！
より、詳細なチュートリアルが知りたい方は以下のドキュメントをご参照ください。

https://learn.microsoft.com/ja-jp/azure/developer/github-copilot-azure/get-started

## Microsoft Learn Docs MCP Server のセットアップ

続いて、Microsoft Learn Docs MCP Server のセットアップに移ります。

### Visual Stidio Code の MCP 設定にエンドポイントを追加する

VS Code の GitHub Copilot で Microsoft Learn Docs MCP Server を使用するためには、以下のように [GitHub リポジトリ](https://github.com/MicrosoftDocs/mcp)の「VS Code | Install Microsoft Docs MCP」を押下するか、自身で VS Code の設定（`settings.json`）に MCP サーバーを追加する必要があります。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/a3c6fd02-1ef1-4bce-9b04-87f18ffef5b3.png)

```json: settings.json
{
    "mcp": {
        "servers": {
            "azure-docs-mcp-server": {
                "type": "http",
                "url": "https://learn.microsoft.com/api/mcp"
            }
        }
    }
}
```

### 試しに触ってみる

まず、MCP サーバーを使用するために GitHub Copilot を **Agent モード** に切り替えておき、Tools の設定で追加した MCP サーバーを選択しておきます。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/dd65080b-9a6f-408b-9fdd-621a150c4db8.png)

この Tools の設定からもわかる通り、Microsoft Docs MCP Server には現在、以下のツールがサポートされています。

| ツール名                | 説明                                               | インプット                  |
| ----------------------- | -------------------------------------------------- | --------------------------- |
| `microsoft_docs_search` | Microsoft の公式ドキュメントに対して検索を実行する | `query(string)`: 検索クエリ |

それでは、今回は Azure の組み込みロールの ID を教えてもらうような質問を投げてみます。

```
Microsoft Docs MCP Serverを使って、Azureの組み込みロールである「ストレージ BLOB データの共同作成者」のIDを教えてください
```

すると、Microsoft Docs MCP Server の `microsoft_docs_search`ツールが起動し、情報を取得してくれました！

::: note info

MCP サーバーのツールが使われるかは、プロンプトの記載方法によって変わります。
そのため、MCP サーバーを確実に使用してもらうためには明示的に **「○○ を使って」** とプロンプトに記載することをお勧めします。

:::

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/1118b3ac-9cd9-41c8-94b8-eeb1b6196615.png)

以上で、GitHub Copilot for Azure と Microsoft Docs MCP Server のセッティングが完了しました。

# GitHub Copilot から Azure Functions を作成する

それではセットアップした機能を使って、GitHub Copilot から自然言語ベースで Azure Functions を実装し、デプロイできるのか試してみましょう。
今回 Azure Functions では Key Vault からシークレットを取得するような関数を用意することとします。

## GitHub Copilot を用いて Functions のソースコードを書く

まずは、GitHub Copilot に Functions のソースコードを書いてもらいます。
まっさらな状態の VS Code を立ち上げ、以下のような要件でソースコードを書いてもらいました。

```md
# やりたいこと

- Azure Function で KeyVault からシークレットを取得するような関数を実装したい
- HTTP トリガーで以下のエンドポイントを用意したい
  - HTTP メソッド：`GET`
  - エンドポイント：`/secret`
- 言語とフレームワーク：.NET 8 isolated

# 注意事項

- 作成するコードは Microsoft Docs MCP Server の情報もインプットに生成してください。
- Azure のベストプラクティスに沿ったコードを生成してください。
```

実際に実行してみた結果が以下となります。
`@azure`とあるため、GitHub Copilot for Azure が実行されていることがわかります。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/dc27e877-539c-46bc-97bf-e7620b2c5a1b.png)

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/49d62504-ba26-4ba6-ba38-38424f06ce81.png)

生成されたソースコードはこちらになります。

```csharp: GetSecretFunction.cs
using Azure.Identity;
using Azure.Security.KeyVault.Secrets;
using Microsoft.Azure.Functions.Worker;
using Microsoft.Extensions.Logging;
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Mvc;
using System.Threading.Tasks;
using System;

namespace azure_copilot
{
    public class GetSecretFunction
    {
        private readonly ILogger<GetSecretFunction> _logger;
        private readonly SecretClient _secretClient;
        private readonly string _secretName = "<your-secret-name>"; // 取得したいシークレット名を指定

        public GetSecretFunction(ILogger<GetSecretFunction> logger)
        {
            _logger = logger;
            // Key Vault の URI を環境変数から取得
            var keyVaultUri = Environment.GetEnvironmentVariable("KeyVaultUri");
            if (string.IsNullOrEmpty(keyVaultUri))
            {
                throw new InvalidOperationException("KeyVaultUri が設定されていません。");
            }
            // マネージドID/ローカル開発両対応の DefaultAzureCredential を利用
            _secretClient = new SecretClient(new Uri(keyVaultUri), new DefaultAzureCredential());
        }

        [Function("GetSecretFunction")]
        public async Task<IActionResult> Run([
            HttpTrigger(AuthorizationLevel.Function, "get", Route = "secret")
        ] HttpRequest req)
        {
            _logger.LogInformation("Key Vault からシークレットを取得します。");
            try
            {
                // クエリパラメータからシークレット名を取得（null許容で安全に取得）
                string? secretName = req.Query["name"].FirstOrDefault();
                if (string.IsNullOrEmpty(secretName))
                {
                    return new BadRequestObjectResult(new { error = "クエリパラメータ 'name' を指定してください。" });
                }
                // シークレット取得（ベストプラクティス：リトライはSDK内蔵）
                KeyVaultSecret secret = await _secretClient.GetSecretAsync(secretName);
                return new OkObjectResult(new { secret = secret.Value });
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Key Vault からのシークレット取得に失敗しました。");
                return new StatusCodeResult(StatusCodes.Status500InternalServerError);
            }
        }
    }
}
```

悪くないですね。認証も `DefaultAzureCrendential()` を用いてトークンを取得するようになってますし、Key Vault に接続するクライアントである `SecretClient` も毎回メソッドで呼ばれないようにコンストラクタで実装されています。

コメントでもベストプラクティスと書かれているので、その辺りも考慮してくれているように見えます。おそらく `@azure get code gen best practices` で情報を取得してきてくれているのでしょう。
ビルドして問題ないことも確認できたため、これで Functions のソースコード作成は終了です。自身でコードは 1 行も書いてないので非常に楽チンですね。

## GitHub Copilot を用いて Azure にリソースをデプロイする

それでは、Functions のソースコードが完成したため、リソースを作成していきます。今回必要なリソースは以下となります。

| リソース種別                 | リソース名                    | 用途                                                     |
| ---------------------------- | ----------------------------- | -------------------------------------------------------- |
| リソースグループ             | `m2-sakai-Qiita-je-RG-01`     | 今回の検証で作成するリソースをグループ化する             |
| Functions                    | `m2-sakai-Qiita-je-FUNC-01`   | 実装したソースコードをデプロイする（従量課金で作成する） |
| Key Vault                    | `m2-sakai-qiita-je-kv-01`     | シークレットを管理し、Functions から取得される           |
| ストレージアカウント         | `m2sakaiqiitajest01`          | Functions の Webjob ログを保存する                       |
| Log Analytics ワークスペース | `m2-sakai-Qiita-je-LOGANA-01` | Functions のログを管理する                               |
| Application Insight          | `m2-sakai-Qiita-je-APPINS-01` | Functions のログを管理する                               |

それではこれらのリソースをすべて GitHub Copilot から作成していきましょう。

### リソースグループの作成

まずはリソースグループの作成です。
以下のようにリソース名を指定してチャットで依頼すると、GitHub Copilot for Azure が起動し、作成してくれたことがわかります。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/529b523e-c108-486f-bd89-9392ea6a1530.png)

実際に Azure ポータルを確認すると、しっかり出来上がってることが確認できました。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/8eb7ecba-9154-44ee-b957-7029db89dc45.png)

### ストレージアカウントの作成

続いてストレージアカウントを作成します。
リソースグループと同様に命令を出すと、自動で作成してくれました。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/bcfd5a7d-6220-474f-9ddd-5d7536c5dc0d.png)

こちらも Azure ポータルでしっかり確認できています。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/0c172169-d573-4b37-87a0-cdae26688b13.png)

### LogAnalytics / Application Insights の作成

続いてログ系のリソースです。
同じように命令したところ、一度 Application Insights が生成されていませんでしたが、原因を伝えたところ再実行され、問題なく作成されました。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/cddf8f6b-b6a1-45de-8266-bb8b62d73dad.png)

現時点でのリソース一覧は以下の通りです。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/939061d1-4c11-47b8-81f2-a98a3594da51.png)

### Key Vault の作成

それでは Key Vault も同じ要領で作成します。こちらも問題なく作成されました。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/4cbe7d6e-50d4-4b81-9847-ba99694f026a.png)

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/dddff9be-2377-4a23-9512-ca779b0a4dbe.png)

今回は Functions から取得する予定のシークレット（`TEST-SECRET: sample`）も登録しましたが、この登録部分については割愛します。

### Functions の作成

最後に、Functions の作成です。
今回の要件に合うように以下のように命令したところ、こちらも問題なく作成してくれました。
ストレージアカウントや、Application Insights との連携も問題なく設定されていました。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/10c91f3a-1e6f-411c-b1e9-daa0e0f041d2.png)

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/fb86521a-bcd9-4e1e-ae1f-e678f337999a.png)

これにて、すべてのリソースが作成できました。
作成したリソース一覧を念の為出力してもらい、確認も取れています。

ほんとに何もしなくてもチャットだけでいけちゃってます笑

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/3efbd9d1-fa62-4b31-b025-51793663bf6e.png)

## GitHub Copilot を用いてアプリをデプロイし動作確認

それでは [GitHub Copilot を用いて Functions のソースコードを書く](#github-copilot-を用いて-functions-のソースコードを書く)で作成した Functions アプリをデプロイし、問題なく動作するか確認していきます。

### Functions アプリのデプロイ

Functions アプリのデプロイももちろん GitHub Copilot から行います。
以下のようにデプロイを依頼したところ、そのままアプリケーションがデプロイされました。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/fc1f4c33-01d6-4db7-8bd7-37fd608aa5bd.png)

### 動作確認

最後に動作確認です。
Functions の URL および作成した関数のエンドポイントを指定したところ、Key Vault に登録しておいたシークレット（`TEST-SECRET: sample`）がしっかり取得できました！

ほんとに全部 GitHub Copilot くんがやってくれましたね。。。すごい。

```bash: Terminal
$ curl https://m2-sakai-qiita-je-func-01.azurewebsites.net/api/secret?name=TEST-SECRET\&code=xxxxx
{"secret":"sample"}
```

# おわりに

いかがでしたでしょうか。
MCP サーバーを登録しておけば、GitHub Copilot からほとんどのことができちゃうことがわかったかと思います。
ただ、今回 Microsoft Learn Docs MCP Server はあまり使いこなせなかった感はあります。
LLM が学習していない最新ドキュメント情報を答えさせるようなユースケースの場合に力を発揮してくれると感じました。
また、プロンプトの書き方が良くないとせっかく設定したツールも使ってくれないので、プロンプトの書き方も日々学んでいきたいですね。

これからはコードをあまり書かなくなるかもしれませんが、AI が書いたコードはしっかりレビューできるようにならないと、と改めて感じました。
みなさんも AI を使った開発現場に順応していきましょう！そして、使えるツールはふんだんに使って開発生産性を爆上げしましょう！
今後の動向にも注目です。最後まで読んでいただきありがとうございました！
