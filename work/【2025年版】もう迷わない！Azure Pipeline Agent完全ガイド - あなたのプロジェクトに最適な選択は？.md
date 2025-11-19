---
title: 【2025年版】もう迷わない！Azure Pipeline Agent完全ガイド - あなたのプロジェクトに最適な選択は？
tags:
  - Azure
  - AzureDevOps
  - AzurePipelines
  - CICD
  - devops
private: false
updated_at: '2025-11-19T08:38:29+09:00'
id: ''
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
なぜなら、エージェントの選択はビルドやデプロイの速度 / コスト / セキュリティ / 管理のしやすさなど、プロジェクトの成功に大きく影響するためです。

2024 年 11 月 18 日、約 1 年前に Azure DevOps チームは **「Managed DevOps Pools」** という新しいサービスを一般提供（GA）しました。
これは、従来の Microsoft-hosted Agent や Self-hosted Agent の課題を解決する可能性を秘めた、注目のサービスとなっています。

https://devblogs.microsoft.com/devops/managed-devops-pools-ga/

**本記事では、Azure Pipelines で利用可能な全 4 種類のエージェントを徹底比較し、推奨するユースケースとともに選択のポイントを紹介します。**
各エージェントの特徴 / メリット・デメリット / コスト / 実際のセットアップ方法まで、実践的な情報を解説していきます。

## Azure Pipelines Agent とは？

Azure Pipelines Agent は、CI/CD パイプラインを実際に実行する環境です。
ビルド / テスト / デプロイなどのタスクは、すべてこのエージェント上で動作します。

### 選べる 4 つのエージェント

Azure Pipelines では、以下の 4 種類のエージェントから選択できます。

| エージェント                              | 一言で表すと                         | こんな人におすすめ                       |
| ----------------------------------------- | ------------------------------------ | ---------------------------------------- |
| **Microsoft-hosted Agent**                | 手軽にすぐ始められる共有エージェント | 個人開発、小規模チーム、初心者           |
| **Self-hosted Agent**                     | 完全に自分で管理する専用エージェント | 完全なコントロールが必要、特殊な環境要件 |
| **Azure Virtual Machine Scale Set Agent** | 自動スケールする専用エージェント     | 既存 Azure インフラ活用、移行期          |
| **Managed DevOps Pools**                  | Microsoft 管理 + カスタマイズ可能    | 管理負荷削減、セキュアな CI/CD           |

### この記事の読み方

各エージェントについて、以下の構成で解説していきます。

1. **概要と特徴**：どんなエージェントなのか
2. **セットアップ方法**：実際の設定手順
3. **メリット/デメリット**：何が良くて何が悪いのか
4. **使用例**：実際のコード例

最後に、**比較表** と **選択のポイント** でどのエージェントを選択したらよいかについて紹介しています。

# Microsoft-hosted Agent

## Microsoft-hosted Agent の概要

Microsoft-hosted Agent は、Microsoft が提供・管理するエージェントです。
ユーザーがエージェントのインストールや保守を行う必要がなく、パイプラインの実行時に自動的にプロビジョニングされます。

https://learn.microsoft.com/ja-jp/azure/devops/pipelines/agents/hosted?view=azure-devops

## Microsoft-hosted Agent の特徴

Microsoft-hosted Agent には以下のような特徴があります。

| 特徴                     | 説明                                                               |
| ------------------------ | ------------------------------------------------------------------ |
| **即座に利用可能**       | エージェントのセットアップが不要で、すぐにパイプラインを実行できる |
| **自動メンテナンス**     | OS やツールのアップデートは Microsoft が自動的に行う               |
| **複数のイメージを提供** | Ubuntu、Windows、macOS など、様々な OS イメージが用意されている    |
| **クリーンな環境**       | 各ジョブは新しい VM で実行されるため、前回の実行の影響を受けない   |

## 利用可能なイメージ

Microsoft-hosted Agent では、以下のイメージが提供されています（2025 年 11 月時点）。

https://learn.microsoft.com/ja-jp/azure/devops/pipelines/agents/hosted?view=azure-devops&tabs=windows-images%2Cyaml

| イメージ名     | OS                  | 説明                                                       |
| -------------- | ------------------- | ---------------------------------------------------------- |
| ubuntu-latest  | Ubuntu              | 最新の Ubuntu イメージ（現在は Ubuntu 24.04）              |
| ubuntu-24.04   | Ubuntu 24.04        | Ubuntu 24.04 LTS                                           |
| ubuntu-22.04   | Ubuntu 22.04        | Ubuntu 22.04 LTS                                           |
| windows-latest | Windows             | 最新の Windows イメージ<br>（現在は Windows Server 2025）  |
| windows-2025   | Windows Server 2025 | Windows Server 2025（最新）                                |
| windows-2022   | Windows Server 2022 | Windows Server 2022                                        |
| windows-2019   | Windows Server 2019 | Windows Server 2019<br>（2025 年 12 月 31 日に非推奨予定） |
| macos-latest   | macOS               | 最新の macOS イメージ（現在は macOS 14）                   |
| macos-14       | macOS 14            | macOS 14 (Sonoma)                                          |
| macos-13       | macOS 13            | macOS 13 (Ventura)                                         |

## Microsoft-hosted Agent の使用方法

パイプライン定義（YAML）で以下のように`pool.vmImage`で上記のイメージを指定することで簡単に使用できます。

```yaml: azure-pipelines.yml
pool:
  vmImage: 'ubuntu-latest'
```

## Microsoft-hosted Agent のメリット/デメリット

**メリット**

- インフラ管理が不要で、すぐに使い始められる
- 常に最新の状態に保たれる
- 複数の OS やバージョンを簡単に切り替えられる
- スケーラビリティが高く、並列実行が容易
- 無料枠がある（月 1,800 分/並列ジョブ 1 つ）

**デメリット**

- カスタマイズの自由度が低い
- 特定のソフトウェアやツールをプリインストールできない
- プライベートネットワークへのアクセスが難しい
- 無料枠を超えると従量課金が発生する

## Azure DevOps での確認方法

Azure DevOps ポータルで「Project Settings」→「Agent pools」→「Azure Pipelines」を選択すると、Microsoft-hosted Agent の情報を確認できます。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/066258dd-b88c-474a-8674-01787d67ffb0.png)

# Self-hosted Agent

## Self-hosted Agent の概要

Self-hosted Agent は、ユーザーが自身で管理するマシン上にインストールして使用するエージェントです。
オンプレミスのサーバー、Azure VM、他のクラウドプロバイダーの VM など、任意の環境にインストールできます。

https://learn.microsoft.com/ja-jp/azure/devops/pipelines/agents/agents?view=azure-devops&tabs=yaml%2Cbrowser#install

## Self-hosted Agent の特徴

Self-hosted Agent には以下のような特徴があります。

| 特徴                                 | 説明                                                     |
| ------------------------------------ | -------------------------------------------------------- |
| **完全なカスタマイズ**               | 必要なソフトウェアやツールを自由にインストール・設定可能 |
| **プライベートネットワークアクセス** | 企業内のプライベートリソースに直接アクセス可能           |
| **永続的な環境**                     | ジョブ間でキャッシュやアーティファクトを保持可能         |
| **コスト管理**                       | 既存のインフラを活用でき、実行時間による課金がない       |

:::note info
Self-hosted Agent は **Docker コンテナ内でも実行可能です。**
コンテナ化によりエージェントの環境を統一でき、セットアップが簡素化されます。詳細は [Docker でセルフホステッドエージェントを実行する](https://learn.microsoft.com/ja-jp/azure/devops/pipelines/agents/docker?view=azure-devops) を参照してください。
:::

## Self-hosted Agent のセットアップ

Self-hosted Agent をセットアップする手順は以下の通りです。

1. Azure DevOps ポータルで「Project Settings」→「Agent pools」→「Add pool」を選択
2. 「New」,「Self-hosted」を選択し、プール名を入力
3. 作成されたプールから「New agent」を選択
4. エージェントの作成方法が表示されるため、インストールするマシンでスクリプトを実行
5. 認証トークンを使用してエージェントを登録

```bash: Terminal
# Linuxの場合の例
mkdir myagent && cd myagent
wget https://vstsagentpackage.azureedge.net/agent/4.x.x/vsts-agent-linux-x64-3.x.x.tar.gz
tar zxvf vsts-agent-linux-x64-4.x.x.tar.gz
./config.sh
./run.sh
```

## Self-hosted Agent の使用方法

パイプライン定義（YAML）で以下のように指定することで使用できます。

```yaml: azure-pipelines.yml
pool: 'My-Self-Hosted-Agent-Pool' # Self-hosted Agent Poolの名前
```

:::note warn

Microsoft-hosted Agent の設定とは異なり、`pool` にプール名を指定します。少しややこしいですが、そのような仕様となっています。

:::

## Self-hosted Agent のメリット/デメリット

**メリット**

- 完全なカスタマイズが可能
- プライベートネットワークリソースへアクセス可能
- ジョブ実行時間による課金がない
- キャッシュを活用した高速ビルドが可能

**デメリット**

- インフラの管理・保守が必要
- セキュリティパッチの適用が必要
- 初期セットアップに時間がかかる
- 高可用性の確保が必要

## Azure DevOps での確認方法

Azure DevOps ポータルで「Project Settings」→「Agent pools」→「<作成したプール名>」を選択すると、Self-hosted Agent の状態を確認できます。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/ee27a5ac-0a4c-4273-86c1-1f979f6b0025.png)

# Azure Virtual Machine Scale Set Agent

## Azure Virtual Machine Scale Set Agent の概要

Azure Virtual Machine Scale Set (VMSS) Agent は、必要に応じて自動的にスケールアップ・スケールダウンできるセルフホステッドエージェントです。
Microsoft-hosted Agent と Self-hosted Agent の中間的な選択肢として位置づけられ、弾力性のあるインフラで専用エージェントを常時実行する必要性を減らします。

https://learn.microsoft.com/ja-jp/azure/devops/pipelines/agents/scale-set-agents?view=azure-devops

## Azure Virtual Machine Scale Set Agent の特徴

VMSS Agent には以下のような特徴があります。

| 特徴                             | 説明                                                                 |
| -------------------------------- | -------------------------------------------------------------------- |
| **自動スケーリング**             | 需要に応じて自動的にエージェント数を増減（5 分間隔でサンプリング）   |
| **カスタマイズ可能**             | マシンのサイズやカスタムイメージを選択可能                           |
| **コスト効率**                   | 使用していない時はスケールインして不要な VM を削除                   |
| **柔軟な構成**                   | プライベート VNet への接続、カスタムソフトウェアのインストールが可能 |
| **自動再イメージ化**             | ジョブごとに新しい VM を使用するオプション（ステートレスモード）     |
| **Uniform オーケストレーション** | Azure VMSS の Uniform オーケストレーションモードをサポート           |

## Azure Virtual Machine Scale Set Agent のセットアップ

VMSS Agent をセットアップする手順は以下の通りです。

### 1. Azure 仮想マシンスケールセットの作成

Azure Portal または Azure CLI で仮想マシンスケールセットを作成します。

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

## Azure Virtual Machine Scale Set Agent の使用方法

パイプライン定義（YAML）で以下のように指定することで使用できます。
指定の仕方は Self-hosted Agent の場合と同様に、`pool`にプール名を指定します。

```yaml: azure-pipelines.yml
pool: 'My-Vmss-Agent-Pool' # VMSS Agent Poolの名前
```

## Azure Virtual Machine Scale Set Agent のメリット/デメリット

**メリット**

- 自動スケーリングによりコスト効率が高い
- カスタムイメージで必要なソフトウェアをプリインストール可能
- プライベート VNet に接続してプライベートリソースへアクセス可能
- Microsoft-hosted Agent よりも強力なマシンスペックを選択可能
- ジョブごとに新しい VM を使用できる（セキュリティ向上）

**デメリット**

- 自身の Azure サブスクリプションで実行するため、インフラコストが発生
- スケールセットの作成・管理に Azure の知識が必要
- スケールアウトに時間がかかる
- Managed DevOps Pools と比較して設定が複雑（Azure CLI コマンドの実行が必要）

:::note warn
VMSS Agent は　**macOS ではサポートされていません** のでご注意ください。
（Windows Server 2016/2019、Windows 10 Client、Ubuntu Linux のみ対応）
:::

## Azure DevOps での確認方法

Azure DevOps ポータルで「Project Settings」→「Agent pools」を選択すると、VMSS Agent が表示されていることを確認できます。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/c971e37c-4737-4893-989c-a58f6c560d86.png)

## Managed DevOps Pools との違い

VMSS Agent は Managed DevOps Pools の前身となるサービスです。主な違いは以下の通りです。

| 項目                 | VMSS Agent                     | Managed DevOps Pools           |
| -------------------- | ------------------------------ | ------------------------------ |
| **スケーリング粒度** | プールの最大サイズに対する割合 | 1 台単位で正確にスケール       |
| **プールサイズ**     | 数百台のエージェントをサポート | 数千台のエージェントをサポート |
| **複数イメージ**     | 1 つのイメージのみ             | 複数イメージをサポート         |
| **スケジュール機能** | スタンバイエージェント数は固定 | 柔軟なスケジュール設定が可能   |
| **複数組織**         | 単一の Azure DevOps 組織       | 複数組織をサポート             |

Managed DevOps Pools は VMSS Agent を進化させたサービスであり、より簡単に管理でき、スケーラビリティと信頼性が向上しています。
新規にエージェントプールを作成する場合は、Managed DevOps Pools の利用を推奨します。

# Managed DevOps Pools

## Managed DevOps Pools の概要

Managed DevOps Pools は、2024 年 11 月に GA された新しいエージェントサービスです。
Microsoft-hosted Agent と Self-hosted Agent の良いところを組み合わせた、ハイブリッドな選択肢として注目されています。

https://learn.microsoft.com/ja-jp/azure/devops/managed-devops-pools/overview?view=azure-devops

## Managed DevOps Pools の特徴

Managed DevOps Pools には以下のような特徴があります。

| 特徴                       | 説明                                                                            |
| -------------------------- | ------------------------------------------------------------------------------- |
| **Microsoft による管理**   | インフラの管理は Microsoft が行うため、保守の負担が軽減される                   |
| **カスタマイズ可能**       | カスタムイメージを使用して、必要なツールをプリインストール可能                  |
| **Azure 統合**             | Azure Virtual Network と統合し、プライベートリソースへアクセス可能              |
| **高度なスケーラビリティ** | 需要に応じて自動的にスケールイン/スケールアウト（1 台単位の精密なスケーリング） |
| **セキュリティ**           | Microsoft Entra ID との統合、マネージド ID のサポート                           |
| **長時間実行対応**         | 最大 2 日間の長時間ワークフローに対応（Microsoft-hosted Agent は最大 6 時間）   |
| **ステート保持**           | エージェントの状態を最大 7 日間維持可能（キャッシュヒットによる高速化）         |
| **データディスク追加**     | ディスク容量のためだけに大きな VM サイズを選択する必要がない                    |
| **複数イメージサポート**   | 1 つのプール内で複数のカスタムイメージを使用可能                                |

## Managed DevOps Pools のセットアップ

Managed DevOps Pools をセットアップする手順は以下の通りです。

### 1. デベロッパーセンターを作成し、プロジェクトを作成する

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
| **Networking** | Azure Virtual Network との統合設定（VNet、サブネット、プライベートエンドポイントなど） |
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

## Managed DevOps Pools のメリット/デメリット

**メリット**

- インフラ管理の負担が少ない（Microsoft が管理）
- カスタムイメージによる柔軟なカスタマイズ
- Azure VNet との統合でプライベートリソースへアクセス可能
- 自動スケーリングによる効率的なリソース利用
- エージェントの起動時間が短い（プール内で待機状態）
- セキュリティとコンプライアンスの要件を満たしやすい
- Azure Key Vault との統合でシークレット管理が可能

**デメリット**

- Microsoft-hosted Agent と比較してコストが高い可能性がある
- カスタムイメージの作成・管理に知識が必要

## Azure DevOps での確認方法

Azure DevOps ポータルで「Project Settings」→「Agent pools」を選択すると、作成した Managed DevOps Pools が表示されていることを確認できます。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/e239a8ba-c7ca-4cd2-a70d-908f4e4e5ce7.png)

# 各 Agent の比較表とユースケース

それでは、これまで紹介してきた 4 つのエージェントを様々な観点から比較していきます。

## 総合比較表

| 項目                   | Microsoft-hosted Agent | Self-hosted Agent           | VMSS Agent                          | Managed DevOps Pools               |
| ---------------------- | ---------------------- | --------------------------- | ----------------------------------- | ---------------------------------- |
| **管理者**             | ◎ Microsoft            | × ユーザー                  | △ ユーザー（部分的に自動化）        | 〇 Microsoft（カスタマイズ可）     |
| **セットアップ**       | ◎ 不要                 | × 必要                      | △ 中程度                            | 〇 最小限                          |
| **起動時間**           | ○ 中程度（2-3 分）     | ◎ 即座（常時稼働の場合）    | × 長い                              | ◎ 短い（プール内で待機）           |
| **カスタマイズ自由度** | × 制限あり             | ◎ 完全な自由度              | ○ カスタムイメージで柔軟            | ○ カスタムイメージ（一部制約あり） |
| **セキュリティ**       | ○ 基本的なセキュリティ | △ ユーザー管理が必要        | △ ユーザー管理が必要                | ◎ Microsoft 管理 + 高度な機能      |
| **コスト**             | ◎ 小規模なら低コスト   | △ インフラコスト + 管理工数 | △ インフラコスト + 中程度の管理工数 | ○ カスタムイメージ + 管理コスト    |

## ユースケース別の推奨

| ユースケース                           | 推奨エージェント                                      | 理由                            |
| -------------------------------------- | ----------------------------------------------------- | ------------------------------- |
| **個人プロジェクト/小規模チーム/PoC**  | Microsoft-hosted Agent                                | 無料枠内で利用可能、管理不要    |
| **大規模・高頻度ビルド**               | VMSS Agent / Managed DevOps Pools                     | コスト効率、キャッシュ利用      |
| **プライベートリソースアクセス**       | Self-hosted Agent / VMSS Agent / Managed DevOps Pools | VNet 統合、プライベートアクセス |
| **高度なカスタマイズ必要**             | Self-hosted Agent                                     | 完全なコントロール              |
| **管理負荷を抑えつつカスタマイズ**     | Managed DevOps Pools                                  | Microsoft 管理+カスタムイメージ |
| **自動スケーリングが必要**             | VMSS Agent / Managed DevOps Pools                     | 需要に応じた自動スケール        |
| **セキュリティ・コンプライアンス重視** | Managed DevOps Pools                                  | Microsoft 管理、VNet 統合       |

## 選択のポイント

それぞれのエージェントには明確な強みがあります。
**「どれが一番」というものはなく、「どれがあなたのプロジェクトに最適か」が重要**です。

**選択の基本フロー**：

1. **まずは Microsoft-hosted Agent で始める** → ほとんどのケースで十分
2. **制約に直面したら Managed DevOps Pools を検討** → 現代的で管理が楽
3. **特殊な要件があれば Self-hosted Agent または VMSS Agent** → 完全な制御が必要な場合

**各エージェントが最適なケース**：

| エージェント             | 最適なケース                                                         |
| ------------------------ | -------------------------------------------------------------------- |
| **Microsoft-hosted**     | 個人プロジェクト、小規模チーム、標準的な CI/CD、すぐに始めたい       |
| **Self-hosted**          | 特殊なハードウェア、完全なコントロール、既存オンプレミスインフラ活用 |
| **VMSS Agent**           | 既存 Azure インフラ活用、Managed DevOps Pools への移行を計画中       |
| **Managed DevOps Pools** | 管理負荷削減、セキュアな CI/CD、高頻度ビルド                         |

# おわりに

いかがでしたでしょうか。
本記事では、Azure Pipelines で利用可能な 4 種類のエージェント（Microsoft-hosted Agent、Self-hosted Agent、Azure Virtual Machine Scale Set Agent、Managed DevOps Pools）について、それぞれの特徴や使用方法、メリット・デメリットを詳しく解説しました。

特に、Managed DevOps Pools は 2024 年 11 月に GA され、今後の CI/CD 環境において重要な選択肢となることは間違いありません。
Azure Key Vault 統合、プロキシサポート、Ubuntu 24.04 対応など、継続的な機能強化が行われており、今後もさらなる改善が期待されます。

この記事が、Azure Pipelines のエージェント選択で迷っているあなたの助けになれば幸いです。
最後まで読んでいただき、ありがとうございました！

## 参考文献

- [Microsoft-hosted agents - Azure Pipelines | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/devops/pipelines/agents/hosted?view=azure-devops)
- [Self-hosted agents - Azure Pipelines | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/devops/pipelines/agents/agents?view=azure-devops)
- [Azure Virtual Machine Scale Set agents - Azure Pipelines | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/devops/pipelines/agents/scale-set-agents?view=azure-devops)
- [Managed DevOps Pools documentation | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/devops/managed-devops-pools/overview?view=azure-devops)
- [Managed DevOps Pools と Azure Virtual Machine Scale Sets エージェントの比較 | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/devops/managed-devops-pools/migrate-from-scale-set-agents?view=azure-devops)
- [Azure Compute Gallery documentation | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/virtual-machines/azure-compute-gallery)
