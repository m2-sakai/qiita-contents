---
title: Remix SPAとAzure Web PubSubでWebSocketチャットアプリを作る（基盤編）
tags:
  - Azure
  - IaC
  - CICD
  - AzureDevOps
  - Bicep
private: false
updated_at: '2024-12-22T06:32:33+09:00'
id: 26711bd340c0ce34edf2
organization_url_name: nri
slide: false
ignorePublish: false
---
# はじめに

今年もアドベントカレンダーの季節がやってきましたね！
毎回どんなネタで書こうかと迷うところなのですが、せっかくなら普段業務で使わない技術を一つでも取り入れたいですね。

私は業務でフロントエンド領域や Azure における技術支援を担当しておりますが、最近は Azure の IaC 関連の業務に携わることが増えています。
はたまた、フロントエンド領域は最近触ることが減ってきてしまっているのが現状です...

ということで今回は Azure では今まで扱ったことのないサービス、フロントエンド領域でも開発したことのない技術で開発してみます。

そこですぐに思い立ったのが **WebSocket** と **Remix** です。そして WebSocket と言えばチャットアプリが王道かなと思います。
今までに経験してきたものと組み合わせて、以下のような流れで開発していきます。

1. WebSocket を使ったリアルタイムチャットアプリを動かすための基盤を構築する
2. Remix SPA モードでアプリケーションを開発し、1 で構築した基盤にデプロイ～動作確認を行う

ということで、[NRI OpenStandia Advent Calendar 2024 シリーズ 2](https://qiita.com/advent-calendar/2024/nri-openstandia)の 9 日目は、何とか Azure でチャットアプリを作ろうとした私の、基盤構築編の奮闘記となります。
アプリ開発編は次回の記事で書こうと思いますので少々お待ちください。

また、今回のソースコードは以下の GitHub に格納しておりますので、是非ご覧ください。
今回は基盤編ということで、`infrastructure` ディレクトリの話になります。

https://github.com/m2-sakai/chatapp-azure

# 今回作る基盤構成

詳細な構築に入る前に、最終的なアーキテクチャ図を以下に示します。
今回はただアプリが動けばよいということではなく、できるだけプロダクション運用も可能な構成とすることを目指しています。詳細は本記事で後述しますが、ざっくりとしたポイントは以下となります。

- フロントエンドアプリケーションは Static WebApps でホスティングする
- Azure 環境は API Management のみパブリックに公開し、他リソースはプライベートエンドポイントや VNET 統合を用いることでセキュアに接続する
- 権限制御はユーザ割り当てマネージド ID を使用し、RBAC で制御する
- 監視は Azure Monitor + Application Insights でログやメトリクスを収集できるようにする

![architecture.drawio.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/3e3fff8a-14f9-47fd-d7d1-4b3538a97558.png)

# 基盤設計

ここから上記で紹介したアーキテクチャを設計した際の考慮点について紹介します。
おおよそポイントとなる（工夫した）点について記載しており、設計ポイント全てではないことにはご留意ください。

## ネットワーク

まずはネットワーク関連のポイントから以下に記します。本構成でのポイントは以下となります。

**① できるだけパブリックに公開する範囲は限定する**

1 点目はプロダクション運用なら当たり前のことになりますが、パブリック公開を極力抑えていることです。具体的には以下の点がポイントとなります。

- 外部公開するものは、Static WebApps における SPA アプリケーションと、API を受け付ける API Management のみ
- API Management は[外部モード](https://learn.microsoft.com/ja-jp/azure/api-management/api-management-using-with-vnet?tabs=stv2)として仮想ネットワークに構築し、バックエンドへの通信はプライベートとする
- API Management は WebSocket と Rest の双方を受け付けることができるため、到達したリクエストに応じてバックエンドの通信を振り分ける

**② すべてのサブネットにネットワークセキュリティグループを付与し、適切な規則を適用する**

こちらも Azure でアーキテクチャを構築する際には基本的な内容かもしれませんが、すべてのサブネットにはネットワークセキュリティグループを付与します。その際のポイントは以下となります。

- すべてのセキュリティ規則に **優先度 900 ですべての通信を拒否する** 設定を入れ、デフォルトの設定は無効にする
- セキュリティ規則には接続元 / 接続先サブネットの IP アドレス帯を指定する

一例として、以下に API Management へ VNet 統合させるサブネットのトラフィックの規則を示します。API Management からは接続先のサブネットにのみ通信ができるような設定を加えています。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/af0c5087-8a90-60bd-045f-9a2f4577d7c0.png)

**③ リソース間はプライベートエンドポイントを利用してセキュアに通信する**

Azure では様々なリソースをパブリックで構築できます。例えば、App Service では特に何もネットワークの設定を行わない場合はパブリックからアクセス可能となります。
ですが、セキュリティ意識を高くもつ場合は「サービスエンドポイント」や「プライベートエンドポイント」を用いて閉域網で通信することができます。

[サービスエンドポイント](https://learn.microsoft.com/ja-jp/azure/virtual-network/virtual-network-service-endpoints-overview)のドキュメントにも以下のように記載されているため、今回はプライベートエンドポイントを用いてリソース間の通信を行うこととします。

> Microsoft では、Azure プラットフォームでホストされているサービスへのセキュリティで保護されたプライベート アクセスには、Azure Private Link とプライベート エンドポイントを使用することをお勧めします。

その他、サービスエンドポイントとプライベートエンドポイントの使い分けについては以下を参照ください。

https://zenn.dev/tomot/articles/89d561c36bc52c

## 権限制御

続いて、権限制御のポイントについて記載します。クラウドサービスを使用する際には最初に躓く方も多いのではないでしょうか。本構成でのポイントは以下となります。

**① 権限にはユーザー割り当てマネージド ID を使用する**
Azure には、マネージド ID というサービスがあります。マネージド ID を使用すると、[Microsoft Entra 認証](https://learn.microsoft.com/ja-jp/entra/identity/authentication/overview-authentication)をサポートするサービスに対して認証を行うことができます。

https://learn.microsoft.com/ja-jp/entra/identity/managed-identities-azure-resources/overview

マネージド ID には **「システム割り当て」** と **「ユーザー割り当て」** の 2 つの種類があります。
「システム割り当て」は **権限を付与したいリソースに対して一意に作成される** のに対して、「ユーザー割り当て」は**一つのマネージド ID を複数のリソースに付与することができます。**

どちらを選択するかはプロジェクトの要件によるところですが、今回はマネージド ID の管理がしやすくなる**ユーザー割り当てマネージド ID** を使用します。

**② 権限制御のスコープは接続したいリソースに限る**
Azure には、権限管理をするときの概念として、**管理スコープ** があります。管理スコープには 4 つのレベルがあります。

- 管理グループ
- サブスクリプション
- リソースグループ
- リソース

① で作成したユーザー割り当てマネージド ID に権限を付与する際、どの範囲（スコープ）に対して権限を付与するかは非常に重要な設計ポイントです。
不必要なスコープにまで権限を与えてしまうと、意図せず他のリソースにアクセスすることができてしまうため、リソースが多くなればなるほどそのスコープ設計にはシビアになるべきです。

本構成では、最小スコープの**リソース**に対して権限を付与します。

上記のようなポイントを意識し、今回構築したユーザ割り当てマネージド ID には以下のような権限が追加されています。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/763f34dd-0b68-d10d-7c32-e442839dd80e.png)

::: note warn

Cosmos DB のデータへ書き込み/読み取りするロールは [Cosmos DB アカウントを管理するロール](https://learn.microsoft.com/ja-jp/azure/cosmos-db/role-based-access-control)とは異なり、Azure ポータル上から設定できません。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/802f4a16-9c83-ea31-bb63-61bd38bfb04f.png)

そのため、Azure CLI や IaC（今回はこちら）で設定する必要があります。
詳しくは以下を参照してください。
https://learn.microsoft.com/ja-jp/azure/cosmos-db/how-to-setup-rbac

:::


## 監視

本記事で紹介する設計ポイントの最後は監視です。プロダクション運用する際には監視（Observability）が不可欠です。
Observability では様々なポイントを監視する必要がありますが、まずは以下の 3 点が取れるかが重要です。

- **ログ：** システムの動作やイベントの履歴を記録したデータ
- **メトリクス：** システムパフォーマンスを定量的に表す指標。インフラ、ホスト、サービス、クラウドプラットフォーム、外部ソースなど、さまざまなソースから取得される
- **トレーシング：** サービス間のトランザクションやリクエストを追跡する手法

Azure では上記の項目を観測する方法として、[Azure Monitor](https://learn.microsoft.com/ja-jp/azure/azure-monitor/overview) と [Application Insights](https://learn.microsoft.com/ja-jp/azure/azure-monitor/app/app-insights-overview) のサービスを用いることが一般的です。
主にメトリクスは Azure Monitor、ログとトレーシングは Application Insights で取得することができます。

余談ですが、Application Insights のトレーシングの機能は素晴らしいです。
特に何か設定したわけでもないのにも関わらず、リクエストの依存関係を一目で表示してくれます。詳細は下記をご参照ください。

https://learn.microsoft.com/ja-jp/azure/azure-monitor/app/asp-net-dependencies

# 基盤構築

ここから実際の基盤構築に入ります。
基盤の構築は Azure ポータルからブラウザ上で作成する方法もありますが、今回は **IaC（Infrastructure as Code）** を利用して基盤を構築します。

## IaC とは

Infrastructure as Code (IaC)とは、一言で言うとインフラ構成をコードで管理するということです。

サーバーやネットワークの設定、ストレージの作成など従来手作業で行われていた作業をコードに落とし込むことで、以下のようなメリットを享受することができます。

- 手作業に起因するミスの防止
- 設定内容の共有・再現性の確保
- バージョン管理

さらに一度書いたコードは再利用が可能であるため、同じ設定を何度も手作業で行う必要が無くなり効率化につながります。不必要になったら消して、必要になったらすぐに作り直すことができるため、**コスト削減**にも繋がりますね。IaC は偉大です。

Azure で IaC を実現する際にはいくつかの手段があります。今回は [Bicep](https://learn.microsoft.com/ja-jp/azure/azure-resource-manager/bicep/overview?tabs=bicep) を用いて基盤を構築します。
他の手段や選び方については以下の記事でまとめておりますので、合わせてご参照いただけると嬉しいです。

https://qiita.com/s_w_high/items/534a6add2a37b172a6bb

## ディレクトリ構成

IaC のソースコードを開発する、と言ってもどのようなディレクトリ構成としたらよいのか迷う方も多いのではないでしょうか。
Bicep に限った話ではありませんが、IaC でインフラストラクチャをコード化する場合、ディレクトリ構成が適切に設計されているか否かによって、プロジェクトのスケーラビリティ、メンテナンス、チームでの作業効率が大きく変わってきます。

まず、今回検討した結果の IaC のソースコードのディレクトリ構成を示します。

```
/chatapp-azure/infrastructure
│
├── /modules                                      # 再利用可能なBicepモジュール
│   └── /api-management                           # リソース毎に作成する
│       ├── apim_module.bicep
│       └── apim_add-api_module.bicep
│
├── /templates                                    # デプロイ単位のBicepテンプレート
│   ├── /integration                              # integration（統合）関連のBicepテンプレート
│   │   └── create-apim_template.bicep
│   └── /network                                  # ネットワーク関連のBicepテンプレート
│       ├── create-nsg-rule_template
│       └── create-vnet-subnet_template.bicep
│
├── /parameters                                   # 環境ごとのパラメータファイル
│   ├── /integration                              # integration（統合）関連のパラメータファイル
│   |   └── /create-apim
│   │       └── create-apim-01_parameter.bicepparam
│   └── /network                                  # ネットワーク関連のパラメータファイル
│       ├── /create-nsg
│       |   ├── create-nsg-rule-sub-0_0_parameter.bicepparam
│       |   └── create-nsg-rule-sub-1_0_parameter.bicepparam
│       └── /create-vnet
│           └── create-vnet-subnet-01_parameter.bicepparam
│
├── pipelines                                     # CI/CDパイプラインの定義
│   ├── /build                                    # CI（ビルド）用パイプライン定義
│   │   └── azure-pipelines.yml
│   └── /deploy                                   # CD（デプロイ）用パイプライン定義
│       └── azure-pipelines.yml
│
├── .gitignore
└── README.md
```

以下にディレクトリの詳細は以下となります。

| ディレクトリ名 | 説明                                                                                                                                                                                                                                                                                                                                 |
| -------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **modules**    | `modules` ディレクトリには、再利用可能な Bicep モジュールを格納します。<br>個別のリソース（例えば、仮想ネットワーク、ストレージアカウント、コンピュートリソースなど）を定義するためのモジュールファイルが置かれます。これらのモジュールは、テンプレートからインポートして使用します。                                                |
| **templates**  | `templates` ディレクトリは、デプロイメント単位で作成する Bicep テンプレートを格納します。<br>これらのテンプレートファイルは、`modules` ディレクトリで定義されたリソースを読み込み、プロジェクト全体のインフラストラクチャを定義します。                                                                                              |
| **parameters** | `parameters` ディレクトリには、各環境ごとのパラメータファイル（`.bicepparam` ファイル）を配置します。<br>このフォルダ構成により、開発（dev）、ステージング（stg）、本番（prod）などの各環境ごとの設定を簡単に管理できます。 <rr>また、ファイル名は`templates`ディレクトリの Bicep ファイルと対応付けることで、可読性も向上させます。 |
| **pipelines**  | `pipelines` ディレクトリには、CI/CD パイプラインの定義を配置します。<br>デプロイには該当する Bicep ファイルが必要なため、このパイプライン定義も同じプロジェクトに配置します。これにより、自動デプロイプロセスが効率化されます。                                                                                                      |

Bicep を扱う際のディレクトリ構成については以下の記事でまとめておりますので、合わせてご参照ください。本構成もこちらを元に検討しています。

https://qiita.com/s_w_high/items/e67dc796401e2694c5f3

## モジュール作成（`modules`）

まず `modules` ディレクトリから整備を行います。
前述したように `modules` ディレクトリは再利用可能な単位で Bicep を作成します。
そのため、基本的に `modules` ディレクトリはどのアーキテクチャでも使いまわせるように作成します。作成する上でのポイントは以下となります。

- 基本的に**一つのファイルには一つの `resource` 定義しか作らない**（確実に必要なものについては複数記載しても OK）
- 既存のリソース情報が必要な場合は [**`existing` を用いて参照する**](https://learn.microsoft.com/ja-jp/azure/azure-resource-manager/bicep/existing-resource)
- ほとんど変わらない[パラメータ](https://learn.microsoft.com/ja-jp/azure/azure-resource-manager/bicep/parameters)は**デフォルト値を付与する**

以下に例として Application Insights の Bicep を示します。
Application Insights では、Log Analytics ワークスぺースのリソース情報が必要になるため、`existing` を用いて取得しています。

```js: modules/application-insights/appi_module.bicep
targetScope = 'resourceGroup'

@description('場所')
param location string = resourceGroup().location

@description('タグ名')
param tag object = {}

@description('Application Insightsのリソース名')
@minLength(1)
@maxLength(260)
param applicationInsightsName string

@description('Application Insightsコンポーネントが参照するアプリケーションの種類')
param kind string = 'web'

/* 他パラメータは省略 */

@description('Log Analytics ワークスペースのリソース名')
param logAnalyticsWorkspaceName string

resource existingLogAnalyticsWorkspace 'Microsoft.OperationalInsights/workspaces@2023-09-01' existing = {
  name: logAnalyticsWorkspaceName
}

resource applicationInsights 'Microsoft.Insights/components@2020-02-02' = {
  name: applicationInsightsName
  location: location
  tags: tag
  kind: kind
  properties: {
    Application_Type: applicationType
    Flow_Type: flowType
    Request_Source: requestSource
    RetentionInDays: retentionInDays
    WorkspaceResourceId: existingLogAnalyticsWorkspace.id
    IngestionMode: ingestionMode
    publicNetworkAccessForIngestion: publicNetworkAccessForIngestion
    publicNetworkAccessForQuery: publicNetworkAccessForQuery
  }
}

output applicationInsightsId string = applicationInsights.id
```

:::note info

Bicep を開発する際の Tips は以下を参照ください。
https://learn.microsoft.com/ja-jp/azure/azure-resource-manager/bicep/file

:::

## デプロイテンプレート作成（`templates`）

続いて `modules` ディレクトリの各種ファイルを読み込み、リソースをデプロイするための `templates` ディレクトリを作成します。
例えば、Cosmos DB とプライベートエンドポイント、マネージド ID への権限設定は 1:1:1 の関係にあるため一つのテンプレートにまとめます。
他にも、リソースを作成する `module` と診断設定を追加する `module` を組み合わせて一つのテンプレートにする等、デプロイしたい単位で作成します。こちらを考慮して、作成するポイントは以下となります。

- 複数の `module` を組み合わせ、**デプロイしたい単位で作成する**
  - 組み合わせる際は **[ `module` 機能](https://learn.microsoft.com/ja-jp/azure/azure-resource-manager/bicep/modules)を使用する**
- 依存関係を明確にするため **[`dependsOn`](https://learn.microsoft.com/ja-jp/azure/azure-resource-manager/bicep/resource-dependencies) を用いて Bicep のデプロイの順序を制御する**

以下に例として、Cosmos DB をデプロイするためのテンプレートを示します。
こちらのテンプレートでは前述のように以下のリソースをデプロイしております。

- Cosmos DB リソース
- Cosmos DB に接続するためのプライベートエンドポイント
- Cosmos DB に接続するためのロールをユーザー割り当てマネージド ID に付与

```js: templates/database/create-cosmos_template.bicep
targetScope = 'resourceGroup'

/*** param: 共通 ***/
@description('場所')
param location string = resourceGroup().location

@description('タグ名')
param tag object = {}

/*** param: Cosmos DB ***/
@description('Cosmos DB のリソース名')
param cosmosDbName string

/*** param: Private Endpoint ***/
@description('プライベートエンドポイントのリソース名')
@minLength(2)
@maxLength(64)
param privateEndpointName string

/* 他パラメータは省略 */

/*** param: Role Assignment ***/
@description('マネージドIDのリソース名')
@minLength(3)
@maxLength(128)
param userAssignedIdentityName string

/* 他パラメータは省略 */

/*** resource/module: Cosmos DB ***/
module cosmosModule '../../modules/cosmos-db/cosmos_module.bicep' = {
  name: '${cosmosDbName}_Deployment'
  params: {
    location: location
    tag: tag
    cosmosDbName: cosmosDbName
  }
  dependsOn: []
}

/*** resource/module: Private Endpoint ***/
module cosmosPepModule '../../modules/private-endpoint/pep_module.bicep' = {
  name: '${privateEndpointName}_Deployment'
  params: {
    location: location
    tag: tag
    privateEndpointName: privateEndpointName
    privateLinkServiceId: cosmosModule.outputs.cosmosDbId
    privateLinkServiceGroupIds: privateLinkServiceGroupIds
    privateLinkServiceConnectionState: privateLinkServiceConnectionState
    assignStaticPrivateIP: assignStaticPrivateIP
    privateIPAddressInfo: privateIPAddressInfo
    virtualNetworkName: virtualNetworkName
    subnetName: privateEndpointSubnetName
    privateDnsZoneName: privateDnsZoneName
  }
  dependsOn: [
    cosmosModule
  ]
}

/*** resource/module: Role Assignment ***/
module roleModule '../../modules/cosmos-db/cosmos_add-role_module.bicep' = {
  name: '${cosmosDbName}_role_Deployment'
  params: {
    cosmosDbName: cosmosDbName
    userAssignedIdentityName: userAssignedIdentityName
    roleDefinitionId: roleDefinitionId
  }
  dependsOn: [
    cosmosModule
  ]
}
```

## デプロイパラメータ作成（`parameters`）

続いて、`templates` ディレクトリに格納するファイルに対してパラメータファイルを作成します。
パラメータは各リソース毎に異なるため、デプロイする対象が異なればパラメータファイルも異なります。今回はリソース毎にパラメータファイルを作成します。作成するポイントは以下となります。

- Bicep のパラメータファイルは `.json` 形式ではなく、[**`.bicepparam` 形式**で作成する](https://learn.microsoft.com/ja-jp/azure/azure-resource-manager/bicep/parameter-files?tabs=Bicep)
- 前述の通りリソース毎に作成するため、**ファイル名にどのリソース名をデプロイするものなのか分かるようにしておく**（サブネット `sub-0_0` に適用するネットワークセキュリティグループ：`create-nsg-rule-sub-0_0_parameter.bicepparam`）

以下に上記で例に挙げた Cosmos DB の Bicep ファイル `templates/database/create-cosmos_template.bicep`に対応するパラメータファイルを示します。

```js: parameters/database/create-cosmos/create-cosmos-01_parameter.bicepparam
using '../../../templates/database/create-cosmos_template.bicep'

/*** param: 共通 ***/
param tag = {}

/*** param: Cosmos DB ***/
param cosmosDbName = 'cosmos-adcl-test-je-01'

/*** param: Private Endpoint ***/
param privateEndpointName = 'pep-adcl-test-je-cosmos-01'
param privateLinkServiceGroupIds = [
  'Sql'
]
param virtualNetworkName = 'vnet-adcl-test-je-01'
param privateEndpointSubnetName = 'sub-4_0'
param privateDnsZoneName = 'privatelink.documents.azure.com'

/*** param: Role Assignment ***/
param userAssignedIdentityName = 'id-adcl-test-je-01'
```

## CI の構築（`pipelines`）

最後に Bicep ファイルの品質を保つための CI（ビルドパイプライン）を構築します。
CD（デプロイパイプライン）については[パイプラインによるデプロイ](#パイプラインによるデプロイ)で記載します。

Bicep における CI でチェックする項目としては以下の 2 点が挙げられますが、今回の構築では 1 のみ検証しております。

1. 作成した Bicep ファイルに構文エラーとベストプラクティス違反がないかチェックする
2. 作成した Bicep ファイルが ARM Template に変換（ビルド）できるかチェックする

前者については、Bicep は[リンター](https://learn.microsoft.com/ja-jp/azure/azure-resource-manager/bicep/linter)が用意されており、対象ファイルに実行することで検証を行います。
以下にそのリンターを実行するスクリプトとパイプライン定義を示します。

```sh: bicep_lint.sh
#!/bin/bash

echo "**** bicep lint start ****"

directory=$1
err_cnt=0
while IFS= read -r file; do
  echo "checking... $file"
  res=$(az bicep lint --file "$file" 2>&1)
  if [ $? -ne 0 ] || echo "$res" | grep -q "BCP"; then
    err_cnt=$((err_cnt + 1))
    echo "$res"
  fi
done < <(find . -wholename "./${directory}/*.bicep" -o -wholename "./${directory}/*.bicepparam")

if [ $err_cnt -ge 1 ]; then
  exit 1
fi

echo "**** bicep lint finish ****"

exit 0
```

```yml: build/azure-pipelines.yml
pool:
  vmImage: ubuntu-latest

stages:
  # Check-Bicep: Bicepファイルの検証
  - stage: CheckBicep
    jobs:
      - job: CheckModules
        steps:
          - script: |
              bash $(Build.SourcesDirectory)/infrastructure/pipelines/build/bicep_lint.sh infrastructure/modules
            displayName: 'Lint Modules'
      - job: CheckTemplates
        steps:
          - script: |
              bash $(Build.SourcesDirectory)/infrastructure/pipelines/build/bicep_lint.sh infrastructure/templates
            displayName: 'Lint Templates'
  # Check-Bicep: Bicepparamファイルの検証
  - stage: CheckBicepparam
    jobs:
      - job: CheckParameters
        steps:
          - script: |
              bash $(Build.SourcesDirectory)/infrastructure/pipelines/build/bicep_lint.sh infrastructure/parameters
            displayName: 'Lint Parameters'
```

本パイプラインを実行することで、以下のように Azure Pipelines で CI が実行されます。
（Azure Pipelines と GitHub の連携については割愛）

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/9c7fa209-f6f3-f32f-8c40-414ed3f58dcd.png)

# パイプラインによるデプロイ

本章では、上記で作成した IaC ソースコードを Azure Pipelines でデプロイした内容について記載します。

## Azure Pipelines とは

Azure Pipelines は Azure DevOps のサービスの一つであり、CI/CD の実現を効率的に行うことができるサービスです。
GitHub や BitBucket など多くのソースリポジトリと統合し、また他の Azure DevOps サービスともシームレスに連携することができます。
また、マルチプラットフォーム/マルチクラウドのサポートにより、Windows/Mac/Linux 環境でのビルドやテストを行うことができ、Azure だけでなく AWS, GCP 等他のクラウド環境にも対応しております。

より詳しい解説を参照したい際は公式ドキュメントをご参照ください。

https://learn.microsoft.com/ja-jp/azure/devops/pipelines/get-started/what-is-azure-pipelines?view=azure-devops

## 設定

Azure Pipelines を正常に動作させるには、いくつかの設定項目が必要なためここでご紹介します。

### Agent の設定

Azure Pipelines を実行するには、実行環境となる Agent が必要となります。
主に使用される Agent について以下に記載します。今回はライトに使用できれば事足りたため、`Microsoft-hosted agents` を選択しました。

| 種別                                                                                                                                                                                                                                                        | 説明                                                                                                                           | 利用シーン                                                                                              |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------- |
| [Microsoft-hosted agents](https://learn.microsoft.com/ja-jp/azure/devops/pipelines/agents/agents?view=azure-devops&tabs=yaml%2Cbrowser#microsoft-hosted-agents)                                                                                             | Microsoft によってホストおよび管理されるエージェント                                                                           | <ul><li>検証用</li><li>常にクリーンな環境で実行したい場合</li><ul>                                      |
| [Self-hosted agents](https://learn.microsoft.com/ja-jp/azure/devops/pipelines/agents/agents?view=azure-devops&tabs=yaml%2Cbrowser#install)                                                                                                                  | 自分の VM でホストされている、自分が構成および管理するエージェント                                                             | <ul><li>閉域環境用</li><li>常に同じ環境で実行したい場合</li></ul>                                       |
| [Azure Virtual Machine Scale Set agents](<[https:l%2Cbrowser#azure-virtual-machine-scale-set-agents](https://learn.microsoft.com/ja-jp/azure/devops/pipelines/agents/agents?view=azure-devops&tabs=yaml%2Cbrowser#azure-virtual-machine-scale-set-agents)>) | Microsoft Azure Virtual Machine Scale Sets を使用するセルフホステッド エージェントの形式で、需要に合わせて自動スケーリング可能 | <ul><li>VM のスケールイン/アウトが受け入れられる場合</li><li>並列実行数を柔軟に変更したい場合</li></ul> |

:::note info
Azure Pipelines エージェントの詳細な説明については以下を参照ください。
https://learn.microsoft.com/ja-jp/azure/devops/pipelines/agents/agents?view=azure-devops&tabs=yaml%2Cbrowser
:::

### Service Connection の設定

Azure Pipelines を使用して Azure にリソースをデプロイする（Bicep を実行する）には、Service Connections の設定が必須です。
Service Connections とは、簡単に言うと Azure Pipelines が外部サービスと接続できるようにする手段です。例えば、Azure Pipelines がダイレクトに Azure リソースを操作するためには、Azure サブスクリプションへのアクセス権が必要であり、Azure Resource Manager を通じた Service Connection を設定する必要があります。

今回は Azure リソースを操作する必要があるため、「Azure Resource Manager」を選択し、必要事項を入力すると Service Connections が作成できます。（詳細な設定は割愛するため、ドキュメントをご参照ください）

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/28c3c7e5-58eb-0aeb-2283-53480fafcf77.png)

:::note info
Service Connection の詳細な説明については以下を参照ください。
https://learn.microsoft.com/ja-jp/azure/devops/pipelines/library/service-endpoints?view=azure-devops&tabs=yaml
:::

### 変数グループの作成

Azure Pipelines には、パイプラインで実行される際に環境変数として使用できる「変数グループ」という機能があります。
`Library` タブから使用でき、秘匿情報やセキュリティファイルも管理することが可能です。

今回は以下のように変数グループを作成し、サブスクリプション情報等はシークレット情報として保存しました。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/89933242-3e8c-b0fa-6dab-bf86b2728ece.png)

:::note info
変数グループ の詳細な説明については以下を参照ください。
https://learn.microsoft.com/ja-jp/azure/devops/pipelines/library/variable-groups?view=azure-devops&tabs=azure-pipelines-ui%2Cyaml
:::

## デプロイ

最後に Azure Pipelines でデプロイする部分となります。ここまで読んでいただきありがとうございました。
Bicep ファイル（ARM Template も同じ）をデプロイする場合、[`AzureResourceManagerTemplateDeployment@3`](https://learn.microsoft.com/ja-jp/azure/devops/pipelines/tasks/reference/azure-resource-manager-template-deployment-v3?view=azure-pipelines)の Task を使用することでデプロイすることができます。
`AzureResourceManagerTemplateDeployment@3` では、以下のパラメータを設定することでデプロイできます。

| 項目                           | 説明                                                                                        |
| ------------------------------ | ------------------------------------------------------------------------------------------- |
| deploymentScope                | デプロイのスコープ。今回はリソースグループに対してデプロイするため、`Resource Group` を指定 |
| azureResourceManagerConnection | 上記で作成した Service Connections 名                                                       |
| subscriptionId                 | デプロイしたいサブスクリプション ID                                                         |
| resourceGroupName              | デプロイしたいリソースグループ                                                              |
| location                       | デプロイしたいリージョン                                                                    |
| csmFile                        | デプロイしたい ARM Template テンプレート                                                    |
| csmParametersFile              | デプロイしたい ARM Template テンプレートに対応したパラメータファイル                        |

以下にネットワークセキュリティグループをデプロイする部分の Azure Pipelines 定義を示します。
ジョブは、Bicep パラメータ/パラメータファイルに対して 1 つ作成されるため、デプロイするリソース（パラメータファイル）が多いと行数も増えてしまいます。
そこで、今回は環境変数（`pipelines/deploy/variables.yml`）にデプロイしたいパラメータ情報を記載し、パイプライン定義ではその変数でループすることでパイプライン定義をできるだけスッキリ書く工夫を施しました。

```yml: pipelines/deploy/variables.yml
variables:
  ##########################################
  # Network Security Group
  ##########################################
  - name: nsgRuleParameterList
    value: 'sub-0_0,sub-1_0,sub-2_0,sub-3_0,sub-4_0' # カンマ区切りでパラメータファイル名の一部を記載
```

```yml: pipelines/deploy/azure-pipelines.yml
trigger:
  - none # デプロイは手動実行

variables:
  - group: adcl-variables    # 変数グループの参照
  - template: variables.yml  # 変数ファイルの参照

pool:
  vmImage: ubuntu-latest     # Microsoft-hosted agents の指定

stages:
  ##########################################
  # Network Security Group
  ##########################################
  - stage: NetworkSecurityGroup
    jobs:
      - ${{ each parameter in split(variables.nsgRuleParameterList, ',') }}: # 変数を配列に変換して、ループ処理
          - job: Create_NSG_${{ replace(parameter, '-', '_') }}
            condition: ne('${{ parameter }}', '')
            steps:
              - task: AzureResourceManagerTemplateDeployment@3
                displayName: 'Bicep deploy for ${{ parameter }}'
                inputs:
                  deploymentScope: 'Resource Group'
                  azureResourceManagerConnection: $(serviceConnectionName)
                  subscriptionId: $(subscriptionId)
                  resourceGroupName: $(resourceGroupName)
                  location: 'Japan East'
                  csmFile: infrastructure/templates/network/create-nsg-rule_template.bicep
                  csmParametersFile: infrastructure/parameters/network/create-nsg/create-nsg-rule-${{ parameter }}_parameter.bicepparam
```

デプロイしたい順番にパイプライン定義を実装し実行すると以下のような実行結果（実行途中の図）となり、本アーキテクチャが構築されます。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/943ba2d9-b9fe-3c08-3850-cc9851cc050f.png)

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/e561ada1-c208-b256-fdf6-db37a108951a.png)

# おわりに

いかがでしたでしょうか。この記事では Azure Web PubSub を使ったセキュアな環境を IaC で構築いたしました。
IaC で基盤構成をコード化しておくと環境の作り直しが即座に可能となるため、作って壊す、が簡単にできるクラウド環境には必須の技術です。
Azure における IaC は Bicep がこれからも使われてくるとは思うので、今後のアップデートに期待をしています。

次回はアプリ開発編となります。Remix SPA モードはまだ未経験なのできちんと使いこなせるか不安ですが、何とか形にしたいと思ってます。

他にも弊社ではアドベントカレンダーで様々な記事が投稿されておりますので、皆さんぜひご覧ください！

- [NRI OpenStandia Advent Calendar 2024 (シリーズ 1〜3)](https://qiita.com/advent-calendar/2024/nri-openstandia)
- [NRI OpenStandia (IAM 編) Advent Calendar 2024](https://qiita.com/advent-calendar/2024/nri-openstandia-iam)
- [一歩ずつ Rust に慣れていく TypeScript エンジニアの記録 Advent Calendar 2024](https://qiita.com/advent-calendar/2024/rust-from-ts)
- [midPoint by OpenStandia Advent Calendar 2024](https://qiita.com/advent-calendar/2024/midpoint-by-openstandia)
