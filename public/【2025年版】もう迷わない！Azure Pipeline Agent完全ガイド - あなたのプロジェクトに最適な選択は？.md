---
title: 【2025年版】もう迷わない！Azure Pipeline Agent完全ガイド - あなたのプロジェクトに最適な選択は？
tags:
  - Azure
  - devops
  - CICD
  - AzureDevOps
  - AzurePipelines
private: false
updated_at: '2025-12-09T17:49:25+09:00'
id: a33b449c282d72143120
organization_url_name: nri
slide: false
ignorePublish: false
---
# はじめに

みなさま、CI/CD の実行環境、何を使っていますでしょうか。
GitHub Actions, GitLab CI/CD, Jenkins, CircleCI と 選択肢は様々かと思いますが、私は最近の業務で Azure DevOps および Azure Pipelines を使用することが増えてきています。

https://learn.microsoft.com/ja-jp/azure/devops/user-guide/what-is-azure-devops?view=azure-devops

https://learn.microsoft.com/ja-jp/azure/devops/pipelines/get-started/what-is-azure-pipelines?view=azure-devops

Azure Pipelines を使用して CI/CD パイプラインを構築する際、**実際のパイプラインの実行環境となるエージェントの選択は非常に重要です。**
エージェントの選択はビルドやデプロイの速度 / コスト / セキュリティ / 管理のしやすさなど、プロジェクトの成功に大きく影響するためです。

また、2024 年 11 月、Azure DevOps チームは **「Managed DevOps Pools」** という新しいサービスを一般提供（GA）しました。
これは、従来の Microsoft-hosted Agent や Self-hosted Agent の課題を解決する可能性を秘めた、注目のサービスとなっています。

https://devblogs.microsoft.com/devops/managed-devops-pools-ga/

**本記事では、Azure Pipelines で利用可能な全 4 種類のエージェントを徹底比較し、推奨するユースケースとともに選択のポイントを紹介します。**

## Azure Pipelines Agent とは？

Azure Pipelines Agent は、CI/CD パイプラインを実際に実行する環境です。
ビルド / テスト / デプロイなどのタスクは、すべてこのエージェント上で動作します。

Azure Pipelines では、以下の 4 種類のエージェントから選択できます。
先に各エージェントのユースケースを知りたい方は [選択のポイント](#選択のポイント)を参照ください。

| エージェント                                                              | 一言で表すと                                   |
| ------------------------------------------------------------------------- | ---------------------------------------------- |
| [**Microsoft-hosted Agent**](#microsoft-hosted-agent)                     | 手軽にすぐ始められる実行環境                   |
| [**Self-hosted Agent**](#self-hosted-agent)                               | 完全にユーザー自身で管理する（専用の）実行環境 |
| [**Virtual Machine Scale Set Agent<br>（以下 VMSS Agent）**](#vmss-agent) | Self-hosted をスケールできる（専用の）実行環境 |
| [**Managed DevOps Pools**](#managed-devops-pools)                         | Microsoft 管理 + カスタマイズ可能な実行環境    |

# Microsoft-hosted Agent

## Microsoft-hosted Agent の概要

Microsoft-hosted Agent は、Microsoft が提供・管理するエージェントです。
ユーザーがエージェントのインストールや保守を行う必要がなく、パイプラインの実行時に自動的にプロビジョニングされます。
また、各ジョブは新しい VM で実行されるため、前回の実行の影響を受けないという特徴も存在します。

https://learn.microsoft.com/ja-jp/azure/devops/pipelines/agents/hosted?view=azure-devops

## 利用可能なイメージ

Microsoft-hosted Agent では、以下のイメージが提供されています（2025 年 11 月時点）。

https://learn.microsoft.com/ja-jp/azure/devops/pipelines/agents/hosted?view=azure-devops&tabs=windows-images%2Cyaml

| イメージ名     | 説明                                                       |
| -------------- | ---------------------------------------------------------- |
| ubuntu-latest  | 最新の Ubuntu イメージ（現在は Ubuntu 24.04）              |
| ubuntu-24.04   | Ubuntu 24.04 LTS                                           |
| ubuntu-22.04   | Ubuntu 22.04 LTS                                           |
| windows-latest | 最新の Windows イメージ<br>（現在は windows-2025）         |
| windows-2025   | Windows Server 2025（2024 年 11 月リリース）               |
| windows-2022   | Windows Server 2022                                        |
| windows-2019   | Windows Server 2019<br>（2025 年 12 月 31 日に非推奨予定） |
| macos-latest   | 最新の macOS イメージ（現在は macOS 15）                   |
| macos-15       | macOS 15 (Sequoia)                                         |
| macos-14       | macOS 14 (Sonoma)                                          |
| macos-13       | macOS 13 (Ventura)                                         |

## Microsoft-hosted Agent のメリット/デメリット

**メリット**

| 観点                 | 説明                                                                                                |
| -------------------- | --------------------------------------------------------------------------------------------------- |
| **導入コスト**       | エージェントのセットアップが不要で、すぐにパイプラインを実行できる                                  |
| **管理・運用コスト** | OS やツールのアップデートは Microsoft が自動的に行う                                                |
| **金額コスト**       | 月 1,800 分/並列ジョブ 1 つのまでは無料で使用可能（並列ジョブを増やすと課金が増えるので注意は必要） |

:::note info

Microsoft-hosted Agent のコスト体系については[こちら](https://azure.microsoft.com/ja-jp/pricing/details/devops/azure-devops-services/)を参照ください。

:::

**デメリット**

| 観点                   | 説明                                                                                                                                               |
| ---------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| **カスタマイズ自由度** | 実行環境に必要なライブラリやツールを事前にインストールできないため、毎回ジョブの前にインストールが必要となる                                       |
| **ネットワーク**       | デプロイ先のリソースにはインターネット経由でのアクセスとなる（そのため、機密情報を扱う KeyVault のようなサービスを利用する際にノックアウトとなる） |

## Microsoft-hosted Agent の使用方法

パイプライン定義（YAML）で以下のように`pool.vmImage`で上記のイメージを指定することで簡単に使用できます。

```yaml: azure-pipelines.yml
pool:
  vmImage: 'ubuntu-latest'
```

## Azure DevOps での Agent 確認方法

Azure DevOps ポータルで「Project Settings」→「Agent pools」→「Azure Pipelines」を選択すると、Microsoft-hosted Agent の情報を確認できます。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/066258dd-b88c-474a-8674-01787d67ffb0.png)

# Self-hosted Agent

## Self-hosted Agent の概要

Self-hosted Agent は、ユーザーが自身で管理する VM 上にインストールして使用するエージェントです。
オンプレミスのサーバーや Azure VM、他のクラウドプロバイダーの VM など、任意の環境にインストールできます。

https://learn.microsoft.com/ja-jp/azure/devops/pipelines/agents/agents?view=azure-devops&tabs=yaml%2Cbrowser#install

## Self-hosted Agent のメリット/デメリット

**メリット**

| 観点                   | 説明                                                                 |
| ---------------------- | -------------------------------------------------------------------- |
| **カスタマイズ自由度** | 必要なソフトウェアやツールを自由にインストール・設定可能             |
| **ネットワーク**       | VNet に接続してプライベートリソースへアクセス可能       |
| **起動時間**           | 常時稼働の場合は即座に実行でき、キャッシュを活用した高速ビルドが可能 |

**デメリット**

| 観点                 | 説明                                                                                 |
| -------------------- | ------------------------------------------------------------------------------------ |
| **管理・運用コスト** | インフラの管理・保守が必要で、セキュリティパッチの適用もユーザー自身で行う必要がある |
| **導入コスト**       | 初期セットアップに時間がかかり、高可用性の確保も必要                                 |
| **金額コスト**         | インフラコストが発生する（並列ジョブの増加はコスト小で対応できるため、大きなデメリットではない）             |

## Self-hosted Agent のセットアップ

Self-hosted Agent をセットアップする手順は以下の通りです。

1. Azure DevOps ポータルで「Project Settings」→「Agent pools」→「Add pool」を選択
2. 「New」,「Self-hosted」を選択し、プール名を入力
3. 作成されたプールから「New agent」を選択
4. エージェントの作成方法が Azure DevOps の画面に表示されるため、インストールする VM でスクリプトを実行（例として、Linux の場合の例を以下に記載）
5. 認証トークンを使用してエージェントを登録

```bash: Terminal
# Linuxの場合の例
mkdir myagent && cd myagent
wget https://vstsagentpackage.azureedge.net/agent/4.x.x/vsts-agent-linux-x64-3.x.x.tar.gz
tar zxvf vsts-agent-linux-x64-4.x.x.tar.gz
./config.sh
./run.sh
```

:::note info
Self-hosted Agent は **Docker コンテナ内でも実行可能です。**
コンテナ化によりエージェントの環境を統一でき、セットアップが簡素化されます。詳細は [Docker でセルフホステッドエージェントを実行する](https://learn.microsoft.com/ja-jp/azure/devops/pipelines/agents/docker?view=azure-devops) を参照してください。
:::

## Self-hosted Agent の使用方法

パイプライン定義（YAML）で以下のように指定することで使用できます。

```yaml: azure-pipelines.yml
pool: 'My-Self-Hosted-Agent-Pool' # Self-hosted Agent Poolの名前
```

:::note warn

Microsoft-hosted Agent の設定とは異なり、`pool` にプール名を指定します。少しややこしいですが、そのような仕様となっています。

:::

## Azure DevOps での Agent 確認方法

Azure DevOps ポータルで「Project Settings」→「Agent pools」→「<作成したプール名>」を選択すると、Self-hosted Agent の状態を確認できます。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/ee27a5ac-0a4c-4273-86c1-1f979f6b0025.png)

# VMSS Agent

## VMSS Agent の概要

VMSS Agent は、必要に応じて自動的にスケールアップ・スケールダウンできるセルフホステッドエージェントです。

Microsoft-hosted Agent と Self-hosted Agent の中間的な選択肢として位置づけられ、専用エージェントを常時実行する必要性を減らします。

https://learn.microsoft.com/ja-jp/azure/devops/pipelines/agents/scale-set-agents?view=azure-devops

## VMSS Agent のメリット/デメリット

**メリット**

| 観点                   | 説明                                                                                         |
| ---------------------- | -------------------------------------------------------------------------------------------- |
| **金額コスト**         | 自動スケーリングにより使用していない時はスケールインして不要な VM を削除（コスト効率が高い） |
| **カスタマイズ自由度** | カスタムイメージで必要なソフトウェアをプリインストール可能で、VM のサイズも選択可能          |
| **ネットワーク**       | VNet に接続してプライベートリソースへアクセス可能                               |

**デメリット**

| 観点                 | 説明                                                                                                    |
| -------------------- | ------------------------------------------------------------------------------------------------------- |
| **導入コスト**       | 作成・管理に Azure の知識が必要（Self-Hosted Agent や Managed DevOps Pools と比較して学習コストが高い） |
| **起動時間**         | スケールアウト時に VM の起動で時間がかかる場合がある                                                    |
| **管理・運用コスト** | VM イメージの更新やセキュリティパッチの適用はユーザー自身で管理が必要                                   |

## VMSS Agent のセットアップ

VMSS Agent をセットアップする手順は以下の通りです。

### 1. Azure 仮想マシンスケールセットの作成

Azure Portal または Azure CLI で仮想マシンスケールセットを作成します。
ここでは Azure CLI を使った例を以下に示します。

```bash: Terminal
az vmss create \
--name vmssagentspool \
--resource-group vmssagents \
--image Ubuntu2204 \
--vm-sku Standard_D2_v4 \
--storage-sku StandardSSD_LRS \
--authentication-type SSH \
--generate-ssh-keys \
--instance-count 2 \
--disable-overprovision \
--upgrade-policy-mode manual \
--single-placement-group false \
--platform-fault-domain-count 1 \
--load-balancer "" \
--orchestration-mode Uniform
```

:::note warn
VMSS Agent は　**macOS ではサポートされていません** のでご注意ください。
（Windows Server 2016/2019、Windows 10 Client、Ubuntu Linux のみ対応）
:::

### 2. エージェントプールの作成

Azure DevOps ポータルで「Project Settings」→「Agent pools」→「Add pool」を選択し、以下を設定します。

| 設定項目                                               | 説明                                                 |
| ------------------------------------------------------ | ---------------------------------------------------- |
| **プールの種類**                                       | 「Azure 仮想マシンスケールセット」を選択             |
| **Azure サブスクリプション**                           | スケールセットを含むサブスクリプションを選択         |
| **仮想マシンスケールセット**                           | 作成したスケールセットを選択                         |
| **最大仮想マシン数**                                   | プールで同時に実行できる VM の最大数を指定           |
| **スタンバイ状態にしておくエージェント数**             | 常に待機状態で保持しておくエージェントの数を指定     |
| **アイドル状態のエージェントを削除するまでの待機時間** | ジョブ完了後、エージェントを削除するまでの時間を指定 |

## VMSS Agent の使用方法

パイプライン定義（YAML）で以下のように指定することで使用できます。
指定の仕方は Self-hosted Agent の場合と同様に、`pool`にプール名を指定します。

```yaml: azure-pipelines.yml
pool: 'My-Vmss-Agent-Pool' # VMSS Agent Poolの名前
```

## Azure DevOps での Agent 確認方法

Azure DevOps ポータルで「Project Settings」→「Agent pools」を選択すると、VMSS Agent が表示されていることを確認できます。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/c971e37c-4737-4893-989c-a58f6c560d86.png)

# Managed DevOps Pools

## Managed DevOps Pools の概要

Managed DevOps Pools は、2024 年 11 月に GA された新しいエージェントサービスです。

https://learn.microsoft.com/ja-jp/azure/devops/managed-devops-pools/overview?view=azure-devops

:::note info

Managed DevOps Pools は VMSS Agent を進化させたサービスであり、より簡単に管理できスケーラビリティと信頼性が向上していると言われています。

そのため、新規にエージェントプールを作成する場合は、**Managed DevOps Pools の利用を推奨します。**

:::

## Managed DevOps Pools のメリット/デメリット

**メリット**

| 観点                   | 説明                                                                                                                                              |
| ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| **管理・運用コスト**   | インフラの管理は Microsoft が行うため、保守の負担が軽減される（自身の VM イメージは対象外）                                                       |
| **カスタマイズ自由度** | カスタムイメージを使用して必要なツールをプリインストール可能で、1 つのプール内で複数のカスタムイメージを使用可能                                  |
| **ネットワーク**       | VNet に接続してプライベートリソースへアクセス可能                                                                                    |
| **起動時間**           | プール内で待機状態のため起動時間が短く、エージェントの状態を最大 7 日間維持可能（キャッシュによる高速化）                                   |
| **その他**             | 最大 2 日間の長時間ワークフローに対応（Microsoft-hosted は最大 6 時間）、需要に応じたスケーリングが可能 |

**デメリット**

| 観点           | 説明                                                               |
| -------------- | ------------------------------------------------------------------ |
| **金額コスト** | カスタムイメージを使用する場合は、インフラコストが発生する         |
| **導入コスト** | カスタムイメージを使用する場合は、イメージの作成・管理に知識が必要 |

## Managed DevOps Pools のセットアップ

Managed DevOps Pools をセットアップする手順は以下の通りです。

### 1. ディベロッパーセンターを作成し、プロジェクトを作成する

Managed DevOps Pool を使用するには、まずディベロッパーセンターを作成します。
Azure Portal から「Azure Deployment Environments」を選択し、適切な名前で作成します。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/c59cb2ea-c3a7-4be8-8785-971e3a6be347.png)

その後、作成したデベロッパーセンターで「管理」→ 「プロジェクト」と進み、プロジェクトを適切な名前で作成します。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/76134c8c-715b-4bcf-8721-8d3dddcf73a7.png)

### 2. Managed DevOps Pools を作成

続いて、Managed DevOps Pools を作成します。
Azure Portal から「Managed DevOps Pools」を選択し、適切な名前で作成します。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/17b60146-1c46-4fdf-876e-cb26051d7011.png)

### 3. Managed DevOps Pools で各種設定を行う

Managed DevOps Pools では、以下の設定を行うことができます（作成時に同時に設定することもできます）。
ここでは細かい設定内容は割愛しますが、仮想ネットワークと紐づけたりと便利な機能が多々存在します。

| 設定項目       | 内容                                                                                   |
| -------------- | -------------------------------------------------------------------------------------- |
| **Pool**       | プールの基本設定（名前、リージョン、VM サイズなど）                                    |
| **Scaling**    | 自動スケーリングの設定（最小/最大エージェント数、スケジュール設定など）                |
| **Networking** | VNet との統合設定（サブネット、プライベートエンドポイントなど） |
| **Security**   | セキュリティ設定（アクセス制御、ファイアウォール設定など）                             |
| **Identity**   | マネージド ID の設定（Azure リソースへのアクセス権限など）                             |

:::note info

Managed DevOps Pools では、Microsoft Hosted Agent のイメージも使用できますが、**Azure Compute Gallery で共有したカスタムイメージも使用できます。**
そのため、事前にツールをインストールした VM でパイプラインを実行可能です。

カスタムイメージの作成・共有については [Azure Compute Gallery でイメージを格納、共有する](https://learn.microsoft.com/ja-jp/azure/virtual-machines/shared-image-galleries?tabs=vmsource%2Cazure-cli)を参照ください。

:::

## Managed DevOps Pools の使用方法

パイプライン定義（YAML）で以下のように指定することで使用できます。
Managed DevOps Pools では、`demands`でイメージのエイリアスを指定することができます。

```yaml: azure-pipelines.yml
pool:
    name: 'My-Managed-Devops-Pool' # Managed DevOps Pools の名前
    demands:
      - ImageOverride -equals windows-2022 #イメージ名はエイリアスの使用が可能
```

## Azure DevOps での確認方法

Azure DevOps ポータルで「Project Settings」→「Agent pools」を選択すると、作成した Managed DevOps Pools が表示されていることを確認できます。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/e239a8ba-c7ca-4cd2-a70d-908f4e4e5ce7.png)

# 各 Agent の比較表とユースケース

それでは、これまで紹介してきた 4 つのエージェントを様々な観点から比較していきます。

## 総合比較表

以下の表では、メリット / デメリットで示した観点から各エージェントを比較しています。

| 観点                                                         | Microsoft-hosted         | Self-hosted                | VMSS                   | Managed DevOps Pools                       |
| ------------------------------------------------------------ | ------------------------ | -------------------------- | ---------------------- | ------------------------------------------ |
| **導入コスト**<br>（セットアップの手間）                     | ◎ 不要                   | × 必要                     | × 必要               | 〇 最小限                                  |
| **管理・運用コスト**<br>（OS 更新やパッチ適用の手間）        | ◎ Microsoft              | △ ユーザー                 | △ ユーザー             | 〇 Microsoft<br>（カスタムイメージは除く） |
| **金額コスト**<br>（月々の費用）                             | ◎ 無料枠あり             | △ インフラコスト           | △ インフラコスト       | △ インフラコスト                           |
| **カスタマイズ自由度**<br>（ソフトウェアのプリインストール） | × 制限あり               | ◎ 完全な自由度             | ◎ 完全な自由度         | 〇 カスタムイメージで対応可能              |
| **ネットワーク**<br>（プライベートリソースへのアクセス）     | × インターネット経由のみ | ◎ VNet 接続可能 | ◎ VNet 接続可能        | ◎ VNet 統合可能                            |
| **起動時間**<br>（ジョブ開始までの待ち時間）                 | 〇 2-3 分                | ◎ 即座（常時稼働時）       | △ スケールアウト時遅延 | ◎ 短い（プール内で待機）                   |

## 選択のポイント

それぞれのエージェントには明確な強みがあります。
**「どれが一番」というものはなく、「どれがあなたのプロジェクトに最適か」が重要**です。

**選択の基本フロー**：

1. **まずは Microsoft-hosted Agent で始める** → ほとんどのケースで十分
2. **制約に直面したら Managed DevOps Pools を検討** → 現代的で管理が楽
3. **特殊な要件があれば Self-hosted Agent または VMSS Agent** → 完全な制御が必要な場合

**各エージェントが最適なケース**：

| エージェント             | 採用するユースケース                                     |
| ------------------------ | -------------------------------------------------------- |
| **Microsoft-hosted**     | 個人プロジェクト、検証用プロジェクト、即座に始めたい場合 |
| **Self-hosted**          | カスタマイズ要件がある場合、オンプレミスインフラ活用     |
| **VMSS Agent**           | 既存 Azure インフラを活用する必要がある                  |
| **Managed DevOps Pools** | 管理負荷を軽減しながらもセキュアな CI/CD を実現したい    |

# おわりに

本記事では、Azure Pipelines で利用可能な 4 種類のエージェント（Microsoft-hosted Agent、Self-hosted Agent、VMSS Agent、Managed DevOps Pools）について、それぞれの特徴や使用方法を詳しく解説しました。

特に、Managed DevOps Pools は 2024 年 11 月に GA され、今後の CI/CD 環境において重要な選択肢となることは間違いありません。

この記事が、Azure Pipelines のエージェント選択で迷っているあなたの助けになれば幸いです。最後まで読んでいただき、ありがとうございました！

## 参考文献

- [Microsoft-hosted agents - Azure Pipelines | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/devops/pipelines/agents/hosted?view=azure-devops)
- [Self-hosted agents - Azure Pipelines | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/devops/pipelines/agents/agents?view=azure-devops)
- [Azure Virtual Machine Scale Set agents - Azure Pipelines | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/devops/pipelines/agents/scale-set-agents?view=azure-devops)
- [Managed DevOps Pools documentation | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/devops/managed-devops-pools/overview?view=azure-devops)
- [Managed DevOps Pools と Azure Virtual Machine Scale Sets エージェントの比較 | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/devops/managed-devops-pools/migrate-from-scale-set-agents?view=azure-devops)
- [Azure Compute Gallery documentation | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/virtual-machines/azure-compute-gallery)
