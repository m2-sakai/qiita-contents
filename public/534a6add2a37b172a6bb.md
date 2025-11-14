---
title: 'AzureでのIaCツール選びに迷ったらこれ！ARMTemplate, Bicep, Terraform, Pulumi 比較ガイド'
tags:
  - Azure
  - IaC
  - Terraform
  - Pulumi
  - Bicep
private: false
updated_at: '2024-04-04T09:23:31+09:00'
id: 534a6add2a37b172a6bb
organization_url_name: nri
slide: false
ignorePublish: false
---
# 1. はじめに

ご無沙汰しております。
普段の専門はアプリ～ミドル領域ですが、最近業務で 「Azure × IaC」 について触れることがあったため本記事を執筆するに至りました。
少し私見も含んでいるのですが、ご容赦ください。

近年、ソフトウェア開発の現場では、「Infrastructure as Code (IaC)」というキーワードが急速に広がっています。
DevOps が浸透し CI/CD との相性が良いことも広がりを後押ししているのではないでしょうか。

一昔前までは、サーバーやネットワークといったインフラストラクチャは専門のエンジニアが手作業で設定し、管理していました。
しかし現在ではそのような複雑な手続きをコードで自動化することにより、開発フローやシステムの運用を効率化しようというプロジェクトが増えているように感じます。

## IaC とは？

Infrastructure as Code (IaC)とは、一言で言うとインフラ構成をコードで管理するということです。

サーバーやネットワークの設定、ストレージの作成など従来手作業で行われていた作業をコードに落とし込むことで、以下のようなメリットを享受することができます。

- 手作業に起因するミスの防止
- 設定内容の共有・再現性の確保
- バージョン管理

さらに一度書いたコードは再利用が可能であるため、同じ設定を何度も手作業で行う必要が無くなり効率化につながります。嬉しいことがたくさんありますね。

## Azure における IaC の必要性

Azure は Microsoft が提供するクラウドサービスですが、その規模と全機能を手動で管理することは困難であり、効率的かつ確実に運用していくためには IaC が必須です。
また Azure は頻繁に新しいサービスや機能が追加され、それに伴い設定内容も随時変更していく必要があります。
そこで、IaC を利用することでこれらの新しい変更を素早く反映させることが可能です。

Azure では特に「ARM Template」、「Bicep」、「Terraform」、「Pulumi」の 4 つが主な IaC ツールとして活用されています。
各ツールには特徴と利用ケースがあり、最適なツールを選択することが求められます。

この記事では私が実際に使ってみた上で、各ツールの特徴と具体的な使用方法を述べていきます。Azure を活用して IaC を進めたいと考えている方々の一助となれば幸いです。

## 今回作成するリソースと各ツールのバージョン

今回作成するリソースは以下となります。（リソースグループはすでに存在している（`m2-sakai-rg`）ことを前提としております。リージョンは `Japan East` です。）

|     | リソース             | 名前                  | 詳細（特記事項のみ）                                                                                                      |
| --- | -------------------- | --------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| 1   | ストレージアカウント | m2sakaistorageaccount | <ul><li>パフォーマンス：Standard</li><li>レプリケーション：LRS</li><li>アカウントの種類：StorageV2（汎用 v2）</li></ul> |
| 2   | App Service Plan     | m2-sakai-asp          | <ul><li>SKU：B1</li><li>OS：Windows</li></ul>                                                                             |
| 3   | Function App         | m2-sakai-functionapp  |                                                                                                                           |

また、各ツールのバージョン情報は下記になります。

| ツール    | バージョン | 確認方法            | 備考                            |
| --------- | ---------- | ------------------- | ------------------------------- |
| Azure CLI | 2.58.0     | `az version`        |                                 |
| bicep CLI | 0.26.54    | `az bicep version`  |                                 |
| terraform | v1.7.5     | `terraform version` |                                 |
| pulumi    | v3.111.1   | `pulumi version`    |                                 |
| node      | v20.9.0    | `node -v`           | Pulumi を Typescript で扱うため |

今回実装したソースコードについては以下のリポジトリで公開しておりますので、ご興味ある方がご参照いただけると幸いです。

https://github.com/m2-sakai/azure-iac-comparison

それでは各ツールの紹介に移りますが、読む時間のない方やてっとり早く比較表を参照したい方は [6. 各ツールの比較](#6-各ツールの比較)に飛んでいただければと思います。

# 2. ARM Template

## ARM Template の概要

Azure で IaC 化を進める際に、最も親しまれているのはこの「ARM Template」ではないでしょうか。

Azure Resource Manager (ARM) Template は、**Azure の独自**の IaC ツールであり、JSON(JavaScript Object Notation)形式のテンプレートを通じて Azure リソースのプロビジョニングを自動化します。

ARM Template は Azure のサービスに深く統合されており、Azure 環境内の任意のリソースを作成、構成、または削除することが可能です。

https://learn.microsoft.com/ja-jp/azure/azure-resource-manager/templates/overview

## ARM Template の特徴

ARM Template は、以下のような特徴を持っています。

1. ネイティブ IaC ツール：Azure のネイティブ IaC ツールであり、Azure の全リソースに対応しています。
2. 新機能の迅速なサポート：新しくリリースされた Azure サービスや機能がすぐにサポートされます。
3. JSON 形式での扱いやすさ：JSON 形式であるため、プログラミングの経験があれば容易に扱うことができ、VS Code などのエディタ上での編集が最適化されています。

## ARM Template による IaC のメリット/デメリット

### メリット

- Azure のすべてのリソースをサポートしている
- 新機能がすぐにサポートされる
- 公式のテンプレートライブラリが豊富である

### デメリット

- JSON 形式のため記述量が多くなり、コードの可読性に問題がある場合がある
- テストやデバッグが難しくテストフレームワークがない
- 他のクラウドサービスとの互換性がない（Azure 専用）

## ARM Template の実装方法

それでは実際に ARM Template を用いてリソースを作成していきます。

ARM Template を作成する際、一から自身で作り始めてももちろん構いません。
ですが、Azure のリソースには様々な機能があります。そのため、すべてのパラメータを理解して設定することは難しいと考えています（もちろんそうなった方が良いのですが）。

そのため、今回はまず Azure Portal でリソースを作成し、そこから以下のように「オートメーション > テンプレートのエクスポート」を実行し、ARM Template を出力する方法を取っております。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/97722b20-cdcb-0e3f-5cb7-56e837277a16.png)

以下に作成した ARM Template のファイルを示します。
デメリットにも記載しましたが、やはり JSON 形式というだけあって記述量がかなり多いですね。。。
App Service Plan のような他のリソースに依存しないようなリソースは記述量が減りますが、それでも多いように感じます。

<details><summary><b>----------------------------------------<br>&nbsp;&nbsp;&nbsp;ソースコードを表示（ストレージアカウント）<br>&nbsp;------------------------------------------</b></summary><div>

```json:arm_template/Create_StorageAccount_template.json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "param_storageAccount_name": {
      "type": "String"
    }
  },
  "variables": {},
  "resources": [
    {
      "type": "Microsoft.Storage/storageAccounts",
      "apiVersion": "2023-01-01",
      "name": "[parameters('param_storageAccount_name')]",
      "location": "japaneast",
      "sku": {
        "name": "Standard_LRS",
        "tier": "Standard"
      },
      "kind": "StorageV2",
      "properties": {
        "dnsEndpointType": "Standard",
        "defaultToOAuthAuthentication": false,
        "publicNetworkAccess": "Enabled",
        "allowCrossTenantReplication": false,
        "minimumTlsVersion": "TLS1_2",
        "allowBlobPublicAccess": false,
        "allowSharedKeyAccess": true,
        "networkAcls": {
          "bypass": "AzureServices",
          "virtualNetworkRules": [],
          "ipRules": [],
          "defaultAction": "Allow"
        },
        "supportsHttpsTrafficOnly": true,
        "encryption": {
          "requireInfrastructureEncryption": false,
          "services": {
            "file": {
              "keyType": "Account",
              "enabled": true
            },
            "blob": {
              "keyType": "Account",
              "enabled": true
            }
          },
          "keySource": "Microsoft.Storage"
        },
        "accessTier": "Hot"
      }
    },
    {
      "type": "Microsoft.Storage/storageAccounts/blobServices",
      "apiVersion": "2023-01-01",
      "name": "[concat(parameters('param_storageAccount_name'), '/default')]",
      "dependsOn": [
        "[resourceId('Microsoft.Storage/storageAccounts', parameters('param_storageAccount_name'))]"
      ],
      "sku": {
        "name": "Standard_LRS",
        "tier": "Standard"
      },
      "properties": {
        "containerDeleteRetentionPolicy": {
          "enabled": true,
          "days": 7
        },
        "cors": {
          "corsRules": []
        },
        "deleteRetentionPolicy": {
          "allowPermanentDelete": false,
          "enabled": true,
          "days": 7
        }
      }
    },
    {
      "type": "Microsoft.Storage/storageAccounts/fileServices",
      "apiVersion": "2023-01-01",
      "name": "[concat(parameters('param_storageAccount_name'), '/default')]",
      "dependsOn": [
        "[resourceId('Microsoft.Storage/storageAccounts', parameters('param_storageAccount_name'))]"
      ],
      "sku": {
        "name": "Standard_LRS",
        "tier": "Standard"
      },
      "properties": {
        "protocolSettings": {
          "smb": {}
        },
        "cors": {
          "corsRules": []
        },
        "shareDeleteRetentionPolicy": {
          "enabled": true,
          "days": 7
        }
      }
    },
    {
      "type": "Microsoft.Storage/storageAccounts/queueServices",
      "apiVersion": "2023-01-01",
      "name": "[concat(parameters('param_storageAccount_name'), '/default')]",
      "dependsOn": [
        "[resourceId('Microsoft.Storage/storageAccounts', parameters('param_storageAccount_name'))]"
      ],
      "properties": {
        "cors": {
          "corsRules": []
        }
      }
    },
    {
      "type": "Microsoft.Storage/storageAccounts/tableServices",
      "apiVersion": "2023-01-01",
      "name": "[concat(parameters('param_storageAccount_name'), '/default')]",
      "dependsOn": [
        "[resourceId('Microsoft.Storage/storageAccounts', parameters('param_storageAccount_name'))]"
      ],
      "properties": {
        "cors": {
          "corsRules": []
        }
      }
    }
  ]
}
```

</div></details>

<details><summary><b>----------------------------------------<br>&nbsp;&nbsp;&nbsp;ソースコードを表示（App Service Plan）<br>&nbsp;------------------------------------------</b></summary><div>

```json:arm_template/Create_AppServicePlan_template.json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "param_asp_name": {
      "type": "String"
    }
  },
  "variables": {},
  "resources": [
    {
      "type": "Microsoft.Web/serverfarms",
      "apiVersion": "2023-01-01",
      "name": "[parameters('param_asp_name')]",
      "location": "Japan East",
      "sku": {
        "name": "B1",
        "tier": "Basic",
        "size": "B1",
        "family": "B",
        "capacity": 1
      },
      "kind": "app",
      "properties": {
        "perSiteScaling": false,
        "elasticScaleEnabled": false,
        "maximumElasticWorkerCount": 1,
        "isSpot": false,
        "reserved": false,
        "isXenon": false,
        "hyperV": false,
        "targetWorkerCount": 0,
        "targetWorkerSizeId": 0,
        "zoneRedundant": false
      }
    }
  ]
}
```

</div></details>

<details><summary><b>----------------------------------------<br>&nbsp;&nbsp;&nbsp;ソースコードを表示（Function App）<br>&nbsp;------------------------------------------</b></summary><div>

```json:arm_template/Create_Functions_template.json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "param_functions_name": {
      "type": "String"
    },
    "param_subscription_id": {
      "type": "String"
    },
    "param_resource_group_name": {
      "type": "String"
    },
    "param_asp_name": {
      "type": "String"
    }
  },
  "variables": {},
  "resources": [
    {
      "type": "Microsoft.Web/sites",
      "apiVersion": "2023-01-01",
      "name": "[parameters('param_functions_name')]",
      "location": "Japan East",
      "kind": "functionapp",
      "properties": {
        "enabled": true,
        "hostNameSslStates": [
          {
            "name": "[concat(parameters('param_functions_name'), '.azurewebsites.net')]",
            "sslState": "Disabled",
            "hostType": "Standard"
          },
          {
            "name": "[concat(parameters('param_functions_name'), '.scm.azurewebsites.net')]",
            "sslState": "Disabled",
            "hostType": "Repository"
          }
        ],
        "serverFarmId": "[concat('/subscriptions/', parameters('param_subscription_id'), '/resourceGroups/', parameters('param_resource_group_name'), '/providers/Microsoft.Web/serverfarms/', parameters('param_asp_name'))]",
        "reserved": false,
        "isXenon": false,
        "hyperV": false,
        "vnetRouteAllEnabled": false,
        "vnetImagePullEnabled": false,
        "vnetContentShareEnabled": false,
        "siteConfig": {
          "numberOfWorkers": 1,
          "acrUseManagedIdentityCreds": false,
          "alwaysOn": true,
          "http20Enabled": false,
          "functionAppScaleLimit": 0,
          "minimumElasticInstanceCount": 0
        },
        "scmSiteAlsoStopped": false,
        "clientAffinityEnabled": false,
        "clientCertEnabled": false,
        "clientCertMode": "Required",
        "hostNamesDisabled": false,
        "customDomainVerificationId": "D70E92245E299D40808AD89BD4B0569E4666FC7468C3E84AA97A929A865000D0",
        "containerSize": 1536,
        "dailyMemoryTimeQuota": 0,
        "httpsOnly": true,
        "redundancyMode": "None",
        "publicNetworkAccess": "Enabled",
        "storageAccountRequired": false,
        "keyVaultReferenceIdentity": "SystemAssigned"
      }
    },
    {
      "type": "Microsoft.Web/sites/basicPublishingCredentialsPolicies",
      "apiVersion": "2023-01-01",
      "name": "[concat(parameters('param_functions_name'), '/ftp')]",
      "location": "Japan East",
      "dependsOn": [
        "[resourceId('Microsoft.Web/sites', parameters('param_functions_name'))]"
      ],
      "properties": {
        "allow": false
      }
    },
    {
      "type": "Microsoft.Web/sites/basicPublishingCredentialsPolicies",
      "apiVersion": "2023-01-01",
      "name": "[concat(parameters('param_functions_name'), '/scm')]",
      "location": "Japan East",
      "dependsOn": [
        "[resourceId('Microsoft.Web/sites', parameters('param_functions_name'))]"
      ],
      "properties": {
        "allow": false
      }
    },
    {
      "type": "Microsoft.Web/sites/config",
      "apiVersion": "2023-01-01",
      "name": "[concat(parameters('param_functions_name'), '/web')]",
      "location": "Japan East",
      "dependsOn": [
        "[resourceId('Microsoft.Web/sites', parameters('param_functions_name'))]"
      ],
      "properties": {
        "numberOfWorkers": 1,
        "defaultDocuments": [
          "Default.htm",
          "Default.html",
          "Default.asp",
          "index.htm",
          "index.html",
          "iisstart.htm",
          "default.aspx",
          "index.php"
        ],
        "netFrameworkVersion": "v8.0",
        "requestTracingEnabled": false,
        "remoteDebuggingEnabled": false,
        "httpLoggingEnabled": false,
        "acrUseManagedIdentityCreds": false,
        "logsDirectorySizeLimit": 35,
        "detailedErrorLoggingEnabled": false,
        "publishingUsername": "[concat('$', parameters('param_functions_name'))]",
        "scmType": "None",
        "use32BitWorkerProcess": false,
        "webSocketsEnabled": false,
        "alwaysOn": true,
        "managedPipelineMode": "Integrated",
        "virtualApplications": [
          {
            "virtualPath": "/",
            "physicalPath": "site\\wwwroot",
            "preloadEnabled": true
          }
        ],
        "loadBalancing": "LeastRequests",
        "experiments": {
          "rampUpRules": []
        },
        "autoHealEnabled": false,
        "vnetRouteAllEnabled": false,
        "vnetPrivatePortsCount": 0,
        "publicNetworkAccess": "Enabled",
        "cors": {
          "allowedOrigins": ["https://portal.azure.com"],
          "supportCredentials": false
        },
        "localMySqlEnabled": false,
        "ipSecurityRestrictions": [
          {
            "ipAddress": "Any",
            "action": "Allow",
            "priority": 2147483647,
            "name": "Allow all",
            "description": "Allow all access"
          }
        ],
        "scmIpSecurityRestrictions": [
          {
            "ipAddress": "Any",
            "action": "Allow",
            "priority": 2147483647,
            "name": "Allow all",
            "description": "Allow all access"
          }
        ],
        "scmIpSecurityRestrictionsUseMain": false,
        "http20Enabled": false,
        "minTlsVersion": "1.2",
        "scmMinTlsVersion": "1.2",
        "ftpsState": "FtpsOnly",
        "preWarmedInstanceCount": 0,
        "functionAppScaleLimit": 0,
        "functionsRuntimeScaleMonitoringEnabled": false,
        "minimumElasticInstanceCount": 0,
        "azureStorageAccounts": {}
      }
    },
    {
      "type": "Microsoft.Web/sites/hostNameBindings",
      "apiVersion": "2023-01-01",
      "name": "[concat(parameters('param_functions_name'), '/', parameters('param_functions_name'), '.azurewebsites.net')]",
      "location": "Japan East",
      "dependsOn": [
        "[resourceId('Microsoft.Web/sites', parameters('param_functions_name'))]"
      ],
      "properties": {
        "siteName": "[parameters('param_functions_name')]",
        "hostNameType": "Verified"
      }
    }
  ]
}
```

</div></details>

上記のファイルは、 Azure CLI を用いてデプロイすることができます。

```bash:Terminal
# Azure CLI にログイン
az login

# ARM Template を用いてリソースを作成
az deployment group create --resource-group <リソースグループ名> --template-file <ARM Template ファイル> --parameters <パラメータファイル>
```

以上で、ARM Template を用いてリソースを作成することができました。

# 3. Bicep

## Bicep の概要

Bicep は、ARM Template の後継として Microsoft が提供している新しい DSL（Domain Specific Language）ベースの IaC ツールです。

ARM Template とは異なり、Bicep は JSON ではなく、独自のより簡潔な構文を提供しており、これにより Azure リソースの管理を容易にしています。

Bicep のコードは **ARM Template にコンパイルされる**ため、同様に ARM REST API を通じて Azure リソースに対する操作を行います。
これにより、ARM Template と同等の Azure リソースのサポートと、新サービスや機能の迅速なサポートを実現しています。

https://learn.microsoft.com/ja-jp/azure/azure-resource-manager/bicep/overview?tabs=bicep

## Bicep の特徴

Bicep は、以下のような特徴を持っています。

1. ARM Template よりもシンプルで簡潔な構文：Bicep の構文はシンプルで簡潔なため、コードの読みやすさと保守性が向上しています。
2. 具体的な型システム：Bicep には具体的な型システムがあり、エラーチェックとインテリセンスが可能です。
3. モジュラーと再利用可能：Bicep ではカスタムリソースを定義するためのモジュールを作成し、再利用することができます。

## Bicep による IaC のメリット/デメリット

### メリット

- ARM Template からコンパイルされるため、ARM Template と同じく Azure の全リソースをサポートしている
- よりシンプルで簡潔な構文により、可読性と保守性が向上する
- モジュールでカスタムリソースを定義し再利用可能となる

### デメリット

- 他のクラウドプロバイダーとの互換性がない（Azure 専用）

## Bicep の実装方法

それでは ARM Template と同じく、Bicep を用いてリソースを作成していきます。

Bicep も ARM Template 同様、一から作成しても問題はございません（その方が勉強にはなるかもしれません）が、同じく設定値を全て把握するのは難しいため別の方法でファイルを作成します。

Bicep ファイルはデプロイされる際に、内部で ARM Template ファイルにコンパイルされます。そのため、**ARM Template を Bicep は相互に変換可能**となっております。

従って、ますは先程作成した ARM Template を Bicep ファイルにデコンパイルします。

```bash:Terminal
az bicep decompile --file <ARM Template ファイル>
```

こちらを実行して作成された Bicep が以下となります。
確かに ARM Template に比べると多少は読みやすくなったと感じます。

<details><summary><b>----------------------------------------<br>&nbsp;&nbsp;&nbsp;ソースコードを表示（ストレージアカウント）<br>&nbsp;------------------------------------------</b></summary><div>

```js:bicep/Create_StorageAccount_template.bicep
param param_storageAccount_name string

resource param_storageAccount_name_resource 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: param_storageAccount_name
  location: 'japaneast'
  sku: {
    name: 'Standard_LRS'
    tier: 'Standard'
  }
  kind: 'StorageV2'
  properties: {
    dnsEndpointType: 'Standard'
    defaultToOAuthAuthentication: false
    publicNetworkAccess: 'Enabled'
    allowCrossTenantReplication: false
    minimumTlsVersion: 'TLS1_2'
    allowBlobPublicAccess: false
    allowSharedKeyAccess: true
    networkAcls: {
      bypass: 'AzureServices'
      virtualNetworkRules: []
      ipRules: []
      defaultAction: 'Allow'
    }
    supportsHttpsTrafficOnly: true
    encryption: {
      requireInfrastructureEncryption: false
      services: {
        file: {
          keyType: 'Account'
          enabled: true
        }
        blob: {
          keyType: 'Account'
          enabled: true
        }
      }
      keySource: 'Microsoft.Storage'
    }
    accessTier: 'Hot'
  }
}

resource param_storageAccount_name_default 'Microsoft.Storage/storageAccounts/blobServices@2023-01-01' = {
  parent: param_storageAccount_name_resource
  name: 'default'
  sku: {
    name: 'Standard_LRS'
    tier: 'Standard'
  }
  properties: {
    containerDeleteRetentionPolicy: {
      enabled: true
      days: 7
    }
    cors: {
      corsRules: []
    }
    deleteRetentionPolicy: {
      allowPermanentDelete: false
      enabled: true
      days: 7
    }
  }
}

resource Microsoft_Storage_storageAccounts_fileServices_param_storageAccount_name_default 'Microsoft.Storage/storageAccounts/fileServices@2023-01-01' = {
  parent: param_storageAccount_name_resource
  name: 'default'
  sku: {
    name: 'Standard_LRS'
    tier: 'Standard'
  }
  properties: {
    protocolSettings: {
      smb: {}
    }
    cors: {
      corsRules: []
    }
    shareDeleteRetentionPolicy: {
      enabled: true
      days: 7
    }
  }
}

resource Microsoft_Storage_storageAccounts_queueServices_param_storageAccount_name_default 'Microsoft.Storage/storageAccounts/queueServices@2023-01-01' = {
  parent: param_storageAccount_name_resource
  name: 'default'
  properties: {
    cors: {
      corsRules: []
    }
  }
}

resource Microsoft_Storage_storageAccounts_tableServices_param_storageAccount_name_default 'Microsoft.Storage/storageAccounts/tableServices@2023-01-01' = {
  parent: param_storageAccount_name_resource
  name: 'default'
  properties: {
    cors: {
      corsRules: []
    }
  }
}
```

</div></details>

<details><summary><b>----------------------------------------<br>&nbsp;&nbsp;&nbsp;ソースコードを表示（App Service Plan）<br>&nbsp;------------------------------------------</b></summary><div>

```js:bicep/Create_AppServicePlan_template.bicep
param param_asp_name string

resource param_asp_name_resource 'Microsoft.Web/serverfarms@2023-01-01' = {
  name: param_asp_name
  location: 'Japan East'
  sku: {
    name: 'B1'
    tier: 'Basic'
    size: 'B1'
    family: 'B'
    capacity: 1
  }
  kind: 'app'
  properties: {
    perSiteScaling: false
    elasticScaleEnabled: false
    maximumElasticWorkerCount: 1
    isSpot: false
    reserved: false
    isXenon: false
    hyperV: false
    targetWorkerCount: 0
    targetWorkerSizeId: 0
    zoneRedundant: false
  }
}
```

</div></details>

<details><summary><b>----------------------------------------<br>&nbsp;&nbsp;&nbsp;ソースコードを表示（Function App）<br>&nbsp;------------------------------------------</b></summary><div>

```js:bicep/Create_Functions_template.bicep
param param_functions_name string
param param_subscription_id string
param param_resource_group_name string
param param_asp_name string

resource param_functions_name_resource 'Microsoft.Web/sites@2023-01-01' = {
  name: param_functions_name
  location: 'Japan East'
  kind: 'functionapp'
  properties: {
    enabled: true
    hostNameSslStates: [
      {
        name: '${param_functions_name}.azurewebsites.net'
        sslState: 'Disabled'
        hostType: 'Standard'
      }
      {
        name: '${param_functions_name}.scm.azurewebsites.net'
        sslState: 'Disabled'
        hostType: 'Repository'
      }
    ]
    serverFarmId: '/subscriptions/${param_subscription_id}/resourceGroups/${param_resource_group_name}/providers/Microsoft.Web/serverfarms/${param_asp_name}'
    reserved: false
    isXenon: false
    hyperV: false
    vnetRouteAllEnabled: false
    vnetImagePullEnabled: false
    vnetContentShareEnabled: false
    siteConfig: {
      numberOfWorkers: 1
      acrUseManagedIdentityCreds: false
      alwaysOn: true
      http20Enabled: false
      functionAppScaleLimit: 0
      minimumElasticInstanceCount: 0
    }
    scmSiteAlsoStopped: false
    clientAffinityEnabled: false
    clientCertEnabled: false
    clientCertMode: 'Required'
    hostNamesDisabled: false
    customDomainVerificationId: 'D70E92245E299D40808AD89BD4B0569E4666FC7468C3E84AA97A929A865000D0'
    containerSize: 1536
    dailyMemoryTimeQuota: 0
    httpsOnly: true
    redundancyMode: 'None'
    publicNetworkAccess: 'Enabled'
    storageAccountRequired: false
    keyVaultReferenceIdentity: 'SystemAssigned'
  }
}

resource param_functions_name_ftp 'Microsoft.Web/sites/basicPublishingCredentialsPolicies@2023-01-01' = {
  parent: param_functions_name_resource
  name: 'ftp'
  location: 'Japan East'
  properties: {
    allow: false
  }
}

resource param_functions_name_scm 'Microsoft.Web/sites/basicPublishingCredentialsPolicies@2023-01-01' = {
  parent: param_functions_name_resource
  name: 'scm'
  location: 'Japan East'
  properties: {
    allow: false
  }
}

resource param_functions_name_web 'Microsoft.Web/sites/config@2023-01-01' = {
  parent: param_functions_name_resource
  name: 'web'
  location: 'Japan East'
  properties: {
    numberOfWorkers: 1
    defaultDocuments: [
      'Default.htm'
      'Default.html'
      'Default.asp'
      'index.htm'
      'index.html'
      'iisstart.htm'
      'default.aspx'
      'index.php'
    ]
    netFrameworkVersion: 'v8.0'
    requestTracingEnabled: false
    remoteDebuggingEnabled: false
    httpLoggingEnabled: false
    acrUseManagedIdentityCreds: false
    logsDirectorySizeLimit: 35
    detailedErrorLoggingEnabled: false
    publishingUsername: '$${param_functions_name}'
    scmType: 'None'
    use32BitWorkerProcess: false
    webSocketsEnabled: false
    alwaysOn: true
    managedPipelineMode: 'Integrated'
    virtualApplications: [
      {
        virtualPath: '/'
        physicalPath: 'site\\wwwroot'
        preloadEnabled: true
      }
    ]
    loadBalancing: 'LeastRequests'
    experiments: {
      rampUpRules: []
    }
    autoHealEnabled: false
    vnetRouteAllEnabled: false
    vnetPrivatePortsCount: 0
    publicNetworkAccess: 'Enabled'
    cors: {
      allowedOrigins: [
        'https://portal.azure.com'
      ]
      supportCredentials: false
    }
    localMySqlEnabled: false
    ipSecurityRestrictions: [
      {
        ipAddress: 'Any'
        action: 'Allow'
        priority: 2147483647
        name: 'Allow all'
        description: 'Allow all access'
      }
    ]
    scmIpSecurityRestrictions: [
      {
        ipAddress: 'Any'
        action: 'Allow'
        priority: 2147483647
        name: 'Allow all'
        description: 'Allow all access'
      }
    ]
    scmIpSecurityRestrictionsUseMain: false
    http20Enabled: false
    minTlsVersion: '1.2'
    scmMinTlsVersion: '1.2'
    ftpsState: 'FtpsOnly'
    preWarmedInstanceCount: 0
    functionAppScaleLimit: 0
    functionsRuntimeScaleMonitoringEnabled: false
    minimumElasticInstanceCount: 0
    azureStorageAccounts: {}
  }
}

resource param_functions_name_param_functions_name_azurewebsites_net 'Microsoft.Web/sites/hostNameBindings@2023-01-01' = {
  parent: param_functions_name_resource
  name: '${param_functions_name}.azurewebsites.net'
  location: 'Japan East'
  properties: {
    siteName: param_functions_name
    hostNameType: 'Verified'
  }
}
```

</div></details>

上記のファイルは、 Azure CLI を用いてデプロイすることができます。

```bash:Terminal
# Azure CLI にログイン
az login

# Bicep を用いてリソースを作成
az deployment group create --resource-group <リソースグループ名> --template-file <Bicep ファイル> --parameters <パラメータファイル>
```

以上で、Bicep を用いてリソースを作成することができました。

# 4. Terraform

## Terraform の概要

Terraform は HashiCorp が開発した OSS の IaC ツールで、クラウドリソースのプロビジョニングと管理を自動化します。

Terraform はプロバイダーと呼ばれるプラグインを通じて様々なクラウドサービスに対応しており、Azure だけでなく AWS や Google Cloud Platform など、多くのクラウドプロバイダーのリソースを管理することができます。

https://www.terraform.io/

## Terraform の特徴

Terraform の主な特徴は以下のとおりです。

1. 様々なクラウドに対応：AWS、Azure、GCP、その他のクラウドプロバイダーをサポートしており、同じ言語で異なる環境のコードを記述することができます。
2. 独自の言語、HCL(HashiCorp Configuration Language)：独自のプログラミング言語で、JSON との互換性と人間が読みやすい構文を提供します。
3. Plan と Apply ステップ：`terraform plan` コマンドで変更の予測と確認を行い、`terraform apply` コマンドでそれを適用するフローを持ちます。

## Terraform による IaC のメリット/デメリット

### メリット

- 主要なクラウドプロバイダーすべてをサポートしている
- 独自の言語 HCL を使用し、開発者が読みやすい構文である
- 記述的な構文で、実装したい具体的なインフラ構造を直接記述できる
- インフラの変更予測機能があり、変更を適用する前に影響を確認できる

### デメリット

- 独自の言語となるため、学習コストが比較的高い
- デバッグが難しい場合がある
- 最新のクラウドサービスの更新に追従するのが遅い場合がある

## Terraform の実装方法

それでは Terraform を用いてリソースを作成していきます。以下の MS Learn を参考にしながら作成していきます。

https://learn.microsoft.com/ja-jp/azure/developer/terraform/create-resource-group?tabs=azure-cli

Terraform は（後述する Pulumi も）Azure ネイティブではないため、基本的にはドキュメントを読みながら設定値を記述していくことになります。
基本的には Azure Portal で作成するような項目を設定値に記載すれば同じようなリソースが出来上がります。

以下が出来上がった `main.tf` です。変数ファイル等は省略させていただいてます。
こちらのファイルにストレージアカウント、App Service Plan、Function App のすべてのリソースが含まれています。
私自身、Terraform を触るのは初めてだったのですが、独自の書き方だからと言って書き方で戸惑うところはありませんでした。

ARM Template、Bicep は Azure ネイティブという強みはありますが、そこにこだわる必要がなければ選択肢に十分入るであろうと感じました。

```tf:terraform/main.tf
resource "azurerm_storage_account" "storage_account" {
  name                     = "${var.param_storageAccount_name}"
  resource_group_name      = "${var.param_resource_group_name}"
  location                 = "${var.param_location}"
  account_tier             = "Standard"
  account_replication_type = "LRS"
}

resource "azurerm_service_plan" "app_service_plan" {
  name                = "${var.param_asp_name}"
  resource_group_name = "${var.param_resource_group_name}"
  location            = "${var.param_location}"
  os_type             = "Windows"
  sku_name            = "B1"
}

resource "azurerm_windows_function_app" "function_app" {
  name                = "${var.param_functions_name}"
  resource_group_name = "${var.param_resource_group_name}"
  location            = "${var.param_location}"

  storage_account_name       = azurerm_storage_account.storage_account.name
  storage_account_access_key = azurerm_storage_account.storage_account.primary_access_key
  service_plan_id            = azurerm_service_plan.app_service_plan.id

  site_config {}
}
```

上記のファイルは、 以下のコマンドを用いてデプロイすることができます。

```bash:Terminal
# Azure CLI にログイン
az login

# Terraform を初期化
terraform init -upgrade

# Terraform 実行プランの作成
terraform plan -out main.tfplan

# Terraform 実行プランを適用
terraform apply main.tfplan
```

以上で Terraform を用いてリソースを作成することができました。

# 5. Pulumi

## Pulumi の概要

Pulumi は Terraform と同じくクラウドインフラストラクチャーを作成、配置、および管理するための IaC ツール OSS です。
Pulumi は、**通常のプログラミング言語（JavaScript、TypeScript、Python、Go、.NET など）を使用して**クラウドインフラストラクチャを定義します。これにより、開発者は既に知っている言語を利用して IaC を実装できます。

Pulumi は、プラグインシステムを介して様々なクラウドプロバイダに対応しています。Azure、AWS、GCP、その他多くのクラウド、サービスプロバイダのリソースを管理することが可能です。

https://www.pulumi.com/

## Pulumi の特徴

Pulumi の主な特徴は以下のとおりです。

1. 一般的なプログラミング言語の使用：Pulumi は、JavaScript、TypeScript、Python、Go、.NET などの普通のプログラミング言語を使用してインフラをコード化します。これにより、制御フロー、エラーハンドリング、パッケージ管理、テスト、その他の言語機能を活用できます。
2. 様々なクラウドに対応：Pulumi はあらゆるクラウドプロバイダをサポートします。一つのプログラムでマルチクラウド構成を表現したり、一つのクラウドプロバイダから別のクラウドプロバイダに移行するといったシナリオも実現可能です。

## Pulumi による IaC のメリット/デメリット

### メリット

- 一般的なプログラミング言語が利用できるため、既存の知識やツールを用いて利用が可能となる
- クラウドプロバイダに依存せず、どのクラウドでもリソースも同じ言語でコード化できる
- npm や Python Packages など、既存のパッケージ管理システムを使ってコードの再利用が可能となる

### デメリット

- 既存の Terraform などの IaC ツールに比べてコミュニティやエコシステムが小さい
- プログラミング経験がない人にとっては学習コストが高い
- 無料枠を超えるとプロジェクトごとに料金が発生する

## Pulumi の実装方法

それでは Pulumi を用いてリソースをデプロイしていきます。

Pulumi は一般的なプログラミング言語を使用しますが、今回は TypeScript を使用します。
そのため、まずは Pulumi のプロジェクトを作成します。ここでは対話形式で実行される設定の詳細は割愛します。

```bash:Terminal
pulumi new azure-typescript
```

それでは、実際のリソースを作成するコードを実装します。まずはストレージアカウントを作成するコードです。

::: note info

Pulumi で Azure を扱う際は [Azure Native](https://www.pulumi.com/registry/packages/azure-native/) を使用することが推奨されておりますが、今回は Terraform との違いを正しく把握するため、[Azure Classic](https://www.pulumi.com/registry/packages/azure/)を用いてリソースを定義しております。

:::

```ts:pulumi/src/createStorageAccount.ts
import * as pulumi from '@pulumi/pulumi';
import * as azure from '@pulumi/azure';
import { Account } from '@pulumi/azure/storage';

export const createStorageAccount = async (
  resourceGroupName: string = 'm2-sakai-rg',
  storageAccountName: string = 'm2sakaistorageaccount'
): Promise<Account> => {
  const config = new pulumi.Config();
  const location = config.get('location') || 'Japan East';

  const storageAccount = new azure.storage.Account(storageAccountName, {
    name: storageAccountName,
    resourceGroupName: resourceGroupName,
    location: location,
    accountTier: 'Standard',
    accountReplicationType: 'LRS',
  });

  return storageAccount;
};
```

続いて、App Service Plan を作成するコードです。

```ts:pulumi/src/createAppServicePlan.ts
import * as pulumi from '@pulumi/pulumi';
import * as azure from '@pulumi/azure';
import { ServicePlan } from '@pulumi/azure/appservice/servicePlan';

export const createAppServicePlan = async (
  resourceGroupName: string = 'm2-sakai-rg',
  appServicePlanName: string = 'm2-sakai-asp'
): Promise<ServicePlan> => {
  const config = new pulumi.Config();
  const location = config.get('location') || 'Japan East';

  const appServicePlan = new azure.appservice.ServicePlan(appServicePlanName, {
    osType: 'Windows',
    resourceGroupName: resourceGroupName,
    skuName: 'B1',
    location: location,
    name: appServicePlanName,
    perSiteScalingEnabled: false,
    zoneBalancingEnabled: false,
  });

  return appServicePlan;
};
```

続いて、Function App を作成するコードです。

```ts:pulumi/src/createFunctions.ts
import * as pulumi from '@pulumi/pulumi';
import * as azure from '@pulumi/azure';
import { WindowsFunctionApp } from '@pulumi/azure/appservice';

export const createFunctions = async (
  resourceGroupName: string = 'm2-sakai-rg',
  functionsName: string = 'm2-sakai-functionapp',
  appServicePlanId: pulumi.Output<string>,
  storageAccountName: pulumi.Output<string>,
  storageAccountPrimaryAccessKey: pulumi.Output<string>
): Promise<WindowsFunctionApp> => {
  const config = new pulumi.Config();
  const location = config.get('location') || 'Japan East';

  const functionapp = new azure.appservice.WindowsFunctionApp(functionsName, {
    name: functionsName,
    resourceGroupName: resourceGroupName,
    location: location,
    storageAccountName: storageAccountName,
    storageAccountAccessKey: storageAccountPrimaryAccessKey,
    servicePlanId: appServicePlanId,
    siteConfig: {},
  });

  return functionapp;
};
```

最後にこれらの関数を一度に呼び出す `index.ts` を作成すれば完成です。

```ts:pulumi/src/index.ts
import * as pulumi from '@pulumi/pulumi';
import { createStorageAccount } from './createStorageAccount';
import { createAppServicePlan } from './createAppServicePlan';
import { createFunctions } from './createFunctions';

export const main = async () => {
  const config = new pulumi.Config();

  // 1. Create Storage Account
  const storageAccount = await createStorageAccount(
    config.get('rgName'),
    config.get('storageAccountName')
  );

  // 2. Create App Service Plan
  const appServicePlan = await createAppServicePlan(
    config.get('rgName'),
    config.get('aspName')
  );

  // 3. Create Functions
  const functions = await createFunctions(
    config.get('rgName'),
    config.get('functionsName'),
    appServicePlan.id,
    storageAccount.name,
    storageAccount.primaryAccessKey
  );
};

main();
```

それでは実行してリソースを作成します。

```bash:Terminal
# Azure CLI にログイン
az login

# 実行
pulumi up
```

以上で、Pulumi を用いてリソースを作成することができました。
普段からアプリ開発をしている身としては同じ言語で書ける Pulumi の開発体験は非常に良いと思いました。そして何と言っても型安全で開発できるのは正義ですね。
ただ、同時に普段からプログラミング言語を触っていない方が大半の場合は採用するのに勇気がいりますね。。。

# 6. 各ツールの比較

以下に上記で記載したツールを比較します。

|                          | ARM Template                                                             | Bicep                                                                                   | Terraform                                                                               | Pulumi                                                                                 |
| ------------------------ | ------------------------------------------------------------------------ | --------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| **メリット**             | <ul><li> Azure ネイティブ</li><li>JSON 構文が理解しやすい</li></ul>      | <ul><li> ARM Template の複雑さを解消</li><li> シンプルな構文 </li></ul>                 | <ul><li> クラウドに依存しない</li><li> 豊富なエコシステム </li></ul>                    | <ul><li> クラウドに依存しない</li><li> 使用言語が選べる (JS, TS, Python 等) </li></ul> |
| **デメリット**           | <ul><li> 記述が冗長になることがある</li><li> デバッグが難しい </li></ul> | <ul><li> Azure 専用</li><li> 若干の学習コスト </li></ul>                                | <ul><li> 言語（HCL）独自性が強い</li><li> 遅延するアップデート </li></ul>               | <ul><li> 特定の言語を必要とする</li><li> 無料プランの制約 </li></ul>                   |
| **主に利用されるケース** | <ul><li> Azure の全リソースを管理したいとき </li></ul>                   | <ul><li> Azure 環境の全リソースを専用構文で管理したいとき </li></ul>                    | <ul><li> マルチクラウドを考慮する場合</li><li> 柔軟なインフラ管理が必要な場合</li></ul> | <ul><li> 特定のプログラミング言語で IaC を行いたいとき </li></ul>                      |
| **難易度**               | <ul><li> JSON 構文に慣れる必要あり</li><li> 周辺ツールが充実 </li></ul>  | <ul><li> Azure に特化した構文を学ぶ必要あり</li><li> 一部機能の補足などが必要</li></ul> | <ul><li> 専用の言語（HCL）の学習が必要</li><li> ドキュメンテーションが豊富 </li></ul>   | <ul><li> プログラミング経験が必要</li><li> 専用のツールや CLI の使用 </li></ul>        |

## 使い分けのポイント

|                                        | ARM Template                         | Bicep                                    | Terraform                              | Pulumi                                       |
| :------------------------------------- | :----------------------------------- | :--------------------------------------- | :------------------------------------- | :------------------------------------------- |
| **対象となるクラウドプロバイダー**     | Azure 専用                           | Azure 専用                               | マルチクラウド対応                     | マルチクラウド対応                           |
| **サポートするリソースや機能**         | Azure の全リソース                   | Azure の全リソース                       | 様々なリソースやプロバイダ             | 様々なリソースやプロバイダ                   |
| **学習コスト**                         | JSON 理解者向け                      | シンプルな新言語に対する意欲がある人向け | 独自言語(HCL)に対する意欲がある人向け  | 既存のプログラミング言語を活用したい人向け   |
| **既存のスキルセットや言語の好み**     | JSON が得意な人向け                  | Azure に特化して独自言語を学びたい人向け | 独自言語 (HCL) が好きな人向け          | 既存のプログラミング言語を活用したい人向け   |
| **エコシステムやコミュニティの大きさ** | Azure コミュニティに集中したい人向け | Azure コミュニティに集中したい人向け     | 豊富なエコシステムと大きなコミュニティ | クラウドに依らないコミュニティを求める人向け |

これらの要素を総合的に判断し、自身の目的や要件に最適な IaC ツールを選択することが重要かなと思います。

# 7. まとめ

本記事で紹介した ARM Template、Bicep、Terraform、そして Pulumi は、全て IaC(Infrastructure as Code)を実現するツール群です。
これらのツールは、各々特性やメリット、デメリットを持ち、用途や状況によって使い分けられます。

ARM Template と Bicep は Azure 専用で、Azure ネイティブの機能を活用できる一方で、他のクラウドプロバイダに対応できないという制約があります。
一方、Terraform と Pulumi はクラウドプロバイダに依存せず、マルチクラウドの環境でも使うことができます。

具体的な使い分けとしては、対象となるクラウドプロバイダ、サポートしてほしいリソースや機能、学習コストの承受範囲、既存のスキルセットやプログラミング言語の好み、そして求めるエコシステムやコミュニティの大きさ等のポイントを考慮する必要があります。

いずれのツールも一長一短であり、それぞれのニーズに最適なツールを選ぶことが求められます。

本記事が皆様のインフラ管理に役立つ情報を提供できていれば幸いです。最後まで読んでいただきありがとうございました。
