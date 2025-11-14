---
title: Azure で別テナントのリソースをマネージド ID とプライベートエンドポイントを用いてセキュアに接続する
tags:
  - Azure
  - RBAC
  - AzureFunctions
  - ManagedIdentity
  - PrivateEndpoint
private: false
updated_at: '2025-05-26T09:12:26+09:00'
id: 31f2fbd3bbead2451617
organization_url_name: nri
slide: false
ignorePublish: false
---
# はじめに

皆様いかがお過ごしでしょうか。
私は業務で Azure を利用することが多いのですが、権限管理には **「ユーザー割り当てマネージド ID」** を用いることが多いです。

しかし、[こちら](https://learn.microsoft.com/ja-jp/entra/identity/managed-identities-azure-resources/managed-identities-faq#can-i-use-a-managed-identity-to-access-a-resource-in-a-different-directorytenant)の通り、マネージド ID は別テナントのリソースにはアクセスできないと公式ドキュメントに記載されており、そのようなユースケースの場合どうしたらよいのかと考えていました。

:::note alert

> マネージド ID を使って違うディレクトリやテナント内のリソースへアクセスできますか?

**いいえ。現在、マネージド ID ではクロスディレクトリのシナリオはサポートされていません。**

:::

そんな時、5/8 にマネージド ID を用いたフェデレーション ID 資格情報の作成が GA されたとのアナウンスがあり、これを用いて別テナントのリソースにアクセスできるようになりました。

https://devblogs.microsoft.com/identity/access-cloud-resources-across-tenants-without-secrets-ga/

今回はその機能を使って **別テナントのリソースをマネージド ID を用いて触ってみよう！** という記事となります。
またそれだけではなく、別テナントのリソースへの接続は **プライベートエンドポイント** を用いてセキュアに接続しようと思います。

# 今回作成する構成

先に今回作成する構成を以下に示します。
A テナントと B テナントを用意し、A テナントの Functions から B テナントのストレージアカウントや Key Vault に接続してみようと思います。

:::note info
実装の比較のために A テナントにも同様にストレージアカウントと Key Vault を用意しています。
:::

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/02ed4a49-a77c-4711-a2d1-4de0e6fc9698.png)

細かいプライベートエンドポイントやマネージド ID 向けの設定方法は後述しますので、そのまま読み進めていただけたらよいのですが、**今回の記事ではプライベートエンドポイントの作成とマネージド ID の作成、および Functions の実装方法に特化します**ので、以下のリソース作成については本記事では割愛させていただきます。

- 仮想ネットワーク / プライベート DNS ゾーン
- Functions のインフラリソース（アプリ実装についてはこの記事で触れます）
- ストレージアカウント → コンテナ → Blob
- Key Vault → シークレット

# テナント跨ぎのプライベートエンドポイント

それではテナント跨ぎでプライベートエンドポイントを使用する方法について紹介します。

## プライベートエンドポイントについて

プライベートエンドポイント (Private Endpoint) とは、**Azure リソース（例：ストレージアカウントや Key Vault など）に対し、Azure 仮想ネットワーク内のプライベート IP アドレス経由で安全にアクセスできるエンドポイントです。**
これにより、リソースへのトラフィックはインターネットを経由せず、Azure Backbone ネットワーク内に閉域されます。

https://learn.microsoft.com/ja-jp/azure/private-link/private-endpoint-overview

特に、VNet にある Functions や仮想マシンから外部テナントのリソースへセキュアにアクセスしたい場合、プライベートエンドポイントは強力な選択肢になります。

クロステナント（テナント跨ぎ）でもプライベートエンドポイントの作成が可能ですが、通常と異なり **承認** フローが発生する点に注意が必要です。以下では、その具体的な構築手順を紹介します。

## 構築

それではプライベートエンドポイントの構築に移ります。
前述の通り、ストレージアカウントと Key Vault は作成済みの状態が前提となります。

### 【テナント B】接続したいリソースのリソース ID を取得する

まず、プライベートエンドポイントで接続したいリソース（例：ストレージアカウント、Key Vault）を表示し、それぞれのリソース ID を確認しておきます。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/e490c8ad-9d20-468f-801d-724f39a7e290.png)

:::note info

リソース ID は作成したリソースの「JSON ビュー」から確認することができます。

:::

### 【テナント A】プライベートエンドポイントを作成する

続いて、テナント A でプライベートエンドポイントを作成します。
作成する際、以下の設定で「リソース ID またはエイリアスを使って Azure リソースに接続します。」を選択し、先ほど控えたリソース ID を入力します。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/02c0b56f-1e43-4792-a603-ba14e524956b.png)

作成が完了すると、以下のように **Pending** の状態で止まります。
同テナントの場合はすぐに **Approved** の状態となりますが、別テナントの場合は承認が必要となるため保留状態となります。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/1fbdcd11-9e8b-40b3-8a09-5f68225f6ae7.png)

:::note warn

通常はプライベートエンドポイントを作成すると同時にプライベート DNS ゾーンと統合できますが、別テナントのリソース ID を選択している場合は作成時に統合できません。

そのため、[後述](#テナント-aプライベートエンドポイントをプライベート-dns-ゾーンに紐づける)のようにプライベートエンドポイントの承認が終わった後に紐づける必要があります。
:::

### 【テナント B】プライベートエンドポイントを承認する

続いて、テナント B のリソース（例：Key Vault、ストレージアカウント等）を表示し、前段で作成したプライベートエンドポイントを承認します。

リソースの「プライベートエンドポイント接続」の画面を確認すると「保留中」となっているものがあるため、こちらを選択して「承認」を押下します。

承認後、テナント A のプライベートエンドポイントの接続状態が **Pending** → **Approved** となっていれば、プライベートエンドポイント経由でアクセス可能状態となります。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/fad73fa9-5c97-4c3e-abe1-1f400405520c.png)

### 【テナント A】プライベートエンドポイントをプライベート DNS ゾーンに紐づける

これが最後の設定となります。
プライベートエンドポイントを承認しただけでは、接続元（今回は Functions）から IP アドレスを引くことができません。

そのため、[【テナント A】プライベートエンドポイントを作成する](#テナント-aプライベートエンドポイントを作成する)では実施できなかった、**プライベート DNS ゾーンの統合**を実施します。

「作成したプライベートエンドポイント」>「DNS の構成」>「構成の追加」から以下のように作成してある仮想ネットワークおよびプライベート DNS ゾーンを入力します。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/bc5cae74-1670-4434-a747-a169b0453657.png)

これで、Functions から名前解決ができるようになり、別テナントの Azure リソースにプライベートエンドポイント経由でアクセスができるようになりました！

# テナント跨ぎのユーザー割り当てマネージド ID 認証

それでは本記事のメインとなる、マネージド ID を用いて別テナントのリソースにアクセスする方法について紹介します。

## ユーザー割り当てマネージド ID について

まず、マネージド ID とは **Azure リソース（例：VM、App Service、Function など）が、Microsoft Entra ID 認証が必要なサービス（例：Key Vault、Azure SQL Database、Storage Account など）へ安全にアクセスできる ID を自動的に管理・提供する仕組みです。**

これにより、アプリケーションやサービスのために埋め込みの認証情報（シークレットや証明書）をハードコーディングせずに、セキュアな認証を自動化できます。

https://learn.microsoft.com/ja-jp/entra/identity/managed-identities-azure-resources/overview

また、マネージド ID には個別の Azure リソースにそれぞれ割り当てられる **システム割り当てマネージド ID** と、ユーザーが任意に作成でき複数の Azure リソースに割り当てられる **ユーザー割り当てマネージド ID** の 2 種類が存在します。

今回は、マネージド ID の管理を一元化できるユーザー割り当てマネージド ID を利用します。

https://learn.microsoft.com/ja-jp/entra/identity/managed-identities-azure-resources/how-manage-user-assigned-managed-identities?pivots=identity-mi-methods-azp

従来、マネージド ID は **所属テナント内のリソース**しか直接アクセスできませんでした。
しかし、フェデレーション ID 資格情報（Federated Identity Credential）を組み合わせることで、クロステナント（別テナント）リソースへのアクセスが可能になりました。

https://learn.microsoft.com/ja-jp/entra/workload-id/workload-identity-federation-config-app-trust-managed-identity?tabs=microsoft-entra-admin-center

今回は、この新機能を利用して、A テナントのマネージド ID で B テナントのリソースに認証・アクセスしてみます。

## 構築

それではユーザー割り当てマネージド ID 関連の構築に移ります。

### 【テナント A】ユーザー割り当てマネージド ID を作成する

まず、Azure ポータルからマネージド ID を作成します。
Azure ポータルから「マネージド ID」を押下し、任意の名前で作成します。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/e437082f-a568-4a04-a3c9-0df4c077f0d0.png)

### 【テナント A】Entra ID でアプリを登録し、フェデレーション ID 資格情報を作成する

続いて、A テナントの「Microsoft Entra ID」でアプリ登録を行います。「アプリの登録」から以下のように設定します。このアプリケーションが他のテナントにアクセスするため、マルチテナント用として作成します。

| 種類                               | 値                                                   |
| ---------------------------------- | ---------------------------------------------------- |
| 名前                               | 任意のアプリケーションの名前                         |
| サポートされているアカウントの種類 | 任意の組織ディレクトリのアカウント（マルチテナント） |

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/bf6b43a8-a0d7-40bc-9c7a-d7a48b8e9bbb.png)

続いて作成したアプリの「証明書とシークレット」から、**フェデレーション資格情報**を登録します。
「資格情報を追加」を押下し、以下のように設定します。

| 種類                               | 値                           |
| ---------------------------------- | ---------------------------- |
| フェデレーション資格情報のシナリオ | Managed Identity             |
| マネージド ID の選択               | 作成したマネージド ID を選択 |
| 名前                               | 任意の名前                   |

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/befc9d00-5340-4316-899a-77d28f06cf6d.png)

これでテナント A でのアプリ登録は完了です。この後 Functions アプリの実装で、このアプリのアプリケーション ID を利用するため、控えておいてください。

### 【テナント B】サービスプリンシパルをプロビジョニングする

続いて、テナント B にサービスプリンシパルを登録します。
この登録方法はテナント B の管理ポリシーや権限の設定に依存しているので様々な登録方法がありますが、一例として私が実施した方法を記載します。

以下の URL をブラウザから実行し、テナント B のログイン情報でログイン → アクセス許可を承認します。

```
https://login.microsoftonline.com/<テナントBのテナントID>/oauth2/authorize?client_id=<作成したテナントAのアプリのクライアントID>&response_type=code
```

すると、テナント B 側でも同じ名前のアプリケーション（サービスプリンシパル）が作成されています。

### 【テナント B】サービスプリンシパルに権限を設定する

続いて、プロビジョニングしたサービスプリンシパルに接続したいリソースへの権限情報を付与します。
今回の構成だと、以下の権限を割り当てました。

| リソース             | 権限                             |
| -------------------- | -------------------------------- |
| Key Vault            | キーコンテナー管理者             |
| ストレージアカウント | ストレージ Blob データ共同作成者 |

これにより、A テナントのマネージド ID を起点とした連携で B テナントのリソースへ **権限を持って** アクセスできる状態となります。

# Functions アプリの開発

それでは、テナント A の Functions を開発していきます。
今回は Key Vault とストレージアカウントに接続してデータが取得できることが確認できればよいため、以下の API を実装します。また、開発には .NET 8 を用いています。

- Key Vault からシークレットを取得して返却する
- ストレージアカウントから Blob データを取得し、Base64 変換して返却する

## 別テナントからトークンを取得し、接続用クライアントを実装する

各リソースに接続するために接続用クライアント（Key Vault の場合は `SecretClient`、ストレージアカウントの場合は `BlobServiceClient`）を作成します。

接続用クライアントを作成する場合、クレデンシャル情報が必要になります。
今回のシナリオでは別テナントにアクセスするために以下の情報が必要になります。

- [【テナント A】Entra ID でアプリを登録し、フェデレーション ID 資格情報を作成する](#テナント-aentra-id-でアプリを登録しフェデレーション-id-資格情報を作成する)で作成した **アプリのアプリケーション（クライアント）ID**
- 接続する先（今回だとテナント B）の**テナント ID**
- [【テナント A】ユーザー割り当てマネージド ID を作成する](#テナント-aユーザ割り当てマネージド-id-を作成する)で作成した **マネージド ID のクライアント ID**
- audience：**`"api://AzureADTokenExchange"`固定**

これらの情報を用いて、以下のように実装すると別テナントからトークンを取得し接続クライアントを作成することができます。

```csharp
public class CrossTenantAccessFunction
{
    private readonly ILogger<CrossTenantAccessFunction> _logger;
    private readonly SecretClient _secretClient;
    private readonly BlobServiceClient _blobServiceClient;

    public CrossTenantAccessFunction(
        ILogger<CrossTenantAccessFunction> logger)
    {
        _logger = logger;

        string appClientId = Environment.GetEnvironmentVariable("APP_CLIENT_ID");
        string tenantId = Environment.GetEnvironmentVariable("CROSS_TENANT_ID");
        string managedIdentityClientId = Environment.GetEnvironmentVariable("AZURE_CLIENT_ID");
        string audience = "api://AzureADTokenExchange";

        var msiCredential = new ManagedIdentityCredential(managedIdentityClientId);

        ClientAssertionCredential assertion = new(
            tenantId,
            appClientId,
            async (token) =>
            {
                var tokenRequestContext = new Azure.Core.TokenRequestContext(new[] { $"{audience}/.default" });
                var accessToken = await msiCredential.GetTokenAsync(tokenRequestContext).ConfigureAwait(false);
                return accessToken.Token;
            }
        );

        var keyvaultEndpoint = Environment.GetEnvironmentVariable("CROSS_KEYVAULT_ENDPOINT");
        if (string.IsNullOrEmpty(keyvaultEndpoint))
            throw new InvalidOperationException("CROSS_KEYVAULT_ENDPOINT environment variable is not set.");
        _secretClient = new SecretClient(new Uri(keyvaultEndpoint), assertion);

        var storageEndpoint = Environment.GetEnvironmentVariable("CROSS_BLOB_SERVICE_ENDPOINT");
        if (string.IsNullOrEmpty(storageEndpoint))
            throw new InvalidOperationException("CROSS_BLOB_SERVICE_ENDPOINT environment variable is not set.");
        _blobServiceClient = new BlobServiceClient(new Uri(storageEndpoint), assertion);
    }

    // 省略
}
```

:::note info

本部分の実装ですが、同一テナントの場合は以下のような実装となります。
同一テナントの場合はクレデンシャル情報には `DefaultAzureCredential`を用いることでマネージド ID を用いて認証するようにしています。

```csharp
// SecretClient
var keyvaultEndpoint = Environment.GetEnvironmentVariable("SAME_KEYVAULT_ENDPOINT");
if (string.IsNullOrEmpty(keyvaultEndpoint))
    throw new InvalidOperationException("SAME_KEYVAULT_ENDPOINT environment variable is not set.");
_secretClient = new SecretClient(new Uri(keyvaultEndpoint), new DefaultAzureCredential());

// BlobServiceClient
var storageEndpoint = Environment.GetEnvironmentVariable("SAME_BLOB_SERVICE_ENDPOINT");
if (string.IsNullOrEmpty(storageEndpoint))
    throw new InvalidOperationException("SAME_BLOB_SERVICE_ENDPOINT environment variable is not set.");
_blobServiceClient = new BlobServiceClient(new Uri(storageEndpoint), new DefaultAzureCredential());
```

:::

## 各 API の実装

それでは接続用のクライアントが作成できたため、上記の API を開発していきます。
特に目新しいことはしていないため、コードをそのまま載せます。

```csharp
[Function("GetSecretCrossTenant")]
public async Task<IActionResult> GetSecretFromKeyVault(
    [HttpTrigger(AuthorizationLevel.Function, "get")] HttpRequest req)
{
    string key = req.Query["key"];
    if (string.IsNullOrEmpty(key))
    {
        return new BadRequestObjectResult("Missing 'key' parameter.");
    }

    try
    {
        var secret = await _secretClient.GetSecretAsync(key);
        return new OkObjectResult(secret.Value.Value);
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "Failed to get secret from Key Vault.");
        return new NotFoundObjectResult($"Secret '{key}' not found.");
    }
}

[Function("DownloadBlobCrossTenant")]
public async Task<IActionResult> DownloadBlobAsBase64(
    [HttpTrigger(AuthorizationLevel.Function, "get")] HttpRequest req)
{
    string containerName = req.Query["container"];
    string blobName = req.Query["blob"];
    if (string.IsNullOrEmpty(containerName) || string.IsNullOrEmpty(blobName))
    {
        return new BadRequestObjectResult("Missing 'container' or 'blob' parameter.");
    }

    try
    {
        var containerClient = _blobServiceClient.GetBlobContainerClient(containerName);
        var blobClient = containerClient.GetBlobClient(blobName);

        using var ms = new MemoryStream();
        await blobClient.DownloadToAsync(ms);
        string base64 = Convert.ToBase64String(ms.ToArray());
        return new OkObjectResult(base64);
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "Failed to download blob.");
        return new NotFoundObjectResult($"Blob '{blobName}' in container '{containerName}' not found.");
    }
}
```

## デプロイ

それでは、作成された Functions をデプロイします。デプロイ方法は問いませんが、今回は Zip デプロイを利用しました。Zip デプロイについての説明は割愛しますので、ご興味ある方は以下リンクをご参照ください。

https://learn.microsoft.com/ja-jp/azure/azure-functions/deployment-zip-push

作成されると、以下のような Functions が出来上がっている状態となっています。
今回は比較のために同テナントのリソースにアクセスするための API も作っています。

| API                     | 権限                                                            |
| ----------------------- | --------------------------------------------------------------- |
| DownloadBlobCrossTenant | **別** テナントのストレージアカウントから Blob データを取得する |
| DownloadBlobSameTenant  | **同** テナントのストレージアカウントから Blob データを取得する |
| GetSecretCrossTenant    | **別** テナントの Key Vault からシークレットを取得する          |
| GetSecretSameTenant     | **同** テナントのストレージアカウントからシークレットを取得する |

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/2c51729c-eefd-484f-a7e5-79abbe23cae3.png)

## 疎通確認

それでは最後に疎通確認をします。
それぞれのテナントの Key Vault に`TEST-SECRET-001`のシークレットを入れて取得できるか検証します。

| Key Vault のテナント | シークレットの値 |
| -------------------- | ---------------- |
| テナント A           | TenantA          |
| テナント B           | TenantB          |

Functions の URL を実行してみると、見事それぞれのテナントの Key Vault からシークレットを取得することができました！

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/993a86dd-9477-4145-b196-6c3f2cf22b63.png)

# おわりに

いかがでしたでしょうか。
今回は Azure の別テナントに存在するリソースを「ユーザー割り当てマネージド ID」と「プライベートエンドポイント」を使ってセキュアに接続する方法を紹介しました。

プロジェクト現場だと、環境ごとにテナントが分かれてしまうこともあるかと思います。
別テナントのリソースにアクセスしたい、となったときは本記事を思い出していただき、セキュアに接続していただけたら幸いです。
特に、ユーザー割り当てマネージド ID を用いてフェデレーション ID 資格情報を作成するのは最近 GA されたばかりなので、ぜひ使ってみてはいかがでしょうか。

個人的には、各環境で共通の Azure Container Registry を作成し、そこからコンテナイメージを取得できるようになるといいなと思っています。
そのためには Azure CLI でのトークン交換がサポートされないといけないので、そこはアナウンスを待とうと思います。

最後まで読んでいただき、ありがとうございました！
