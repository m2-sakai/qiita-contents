---
title: AzureのIaCツール比較第二弾！ ~Azure Pipelinesを用いてデプロイする方法~
tags:
  - Azure
  - IaC
  - CICD
  - AzureDevOps
  - AzurePipelines
private: false
updated_at: '2024-04-16T15:56:10+09:00'
id: 6b4fb951755fa121640a
organization_url_name: nri
slide: false
ignorePublish: false
---
# はじめに

皆様、突然ですが以下の記事はご覧いただけましたでしょうか。

https://qiita.com/s_w_high/items/534a6add2a37b172a6bb

上記の記事では、Azure 上で使える 4 つの IaC ツールである ARM Template、Bicep、Terraform、Pulumi について、それぞれの概要や特徴、実装方法、コミュニティの活発さなど複数の観点から比較を行いました。

例えば、ARM Template と Bicep は、Azure 固有のリソースに対する深い制御が可能である一方、Terraform と Pulumi はクラウドプロバイダ間での可搬性を強調しており、マルチクラウド環境での利用が考慮されておりました。まだご覧になっていない方はこちらの記事も参照していただけると幸いです。

今回の記事では、前回の続きとしてこれらの IaC ツールで記載したリソースを Azure Pipelines を用いて Azure にデプロイする方法について解説します。
ただ前回同様、今回デプロイするリソースは非常に少ないため、一概に全てのリソースに当てはまるわけではないことに注意いただけたらと思います。

今回使用したパイプライン定義（Yaml ファイル）は各 IaC のコードと一緒に以下の GitHub に格納されておりますので、ご覧いただけたら幸いです。

https://github.com/m2-sakai/azure-iac-comparison

# Azure Pipelines とは？

まず、実際のパイプライン定義に移る前に Azure Pipelines について少し触れようと思います。

Azure Pipelines は Azure DevOps のサービスの一つであり、CI/CD の実現を効率的に行うことができるサービスです。
GitHub や BitBucket など多くのソースリポジトリと統合し、また他の Azure DevOps サービスともシームレスに連携することができます。
また、マルチプラットフォーム/マルチクラウドのサポートにより、Windows/Mac/Linux 環境でのビルドやテストを行うことができ、Azure だけでなく AWS, GCP 等他のクラウド環境にも対応しております。

より詳しい解説を参照したい際は公式ドキュメントをご参照ください。

https://azure.microsoft.com/ja-jp/products/devops/pipelines

# 各 IaC ツールを Azure Pipelines でデプロイする準備

ではここから実際の設定や実装に移ります。
Azure DevOps のプロジェクトは作成されている前提で話を進めますので、Azure DevOps のプロジェクトの作成方法について知りたい場合は公式ドキュメントをご参照ください。

https://learn.microsoft.com/ja-jp/azure/devops/organizations/projects/create-project?view=azure-devops&tabs=browser

## Service Connections の設定

Azure Pipelines を使用して CI/CD パイプラインを実行するには、Service Connections の設定が必須です。
Service Connections とは、簡単に言うと Azure Pipelines が外部サービスと接続できるようにする手段です。例えば、Azure Pipelines がダイレクトに Azure リソースを操作するためには、Azure サブスクリプションへのアクセス権が必要であり、Azure Resource Manager を通じた Service Connection を設定する必要があります。

今回は Azure リソースを操作する必要があるため、「Azure Resource Manager」を選択し、以下の情報を入力すると Service Connections が作成できます。

| 項目                    | 選択/入力内容                                                                        |
| ----------------------- | ------------------------------------------------------------------------------------ |
| Authentication method   | Workload Identity federation (automatic)                                             |
| Scope level             | Subscription                                                                         |
| Subscription            | デプロイしたい Azure のサブスクリプション ID                                         |
| Resource group          | デプロイしたい Azure のリソースグループ名                                            |
| Service connection name | 作成したい Service Connections 名（今回は `m2-sakai-service-connection` としました） |
| Security                | 「Grant access permission to all pipelines」にチェックを入れる                       |

これにて Service Connections の設定が完了です。以下のように作成できております。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/ce3ac231-edce-4bce-d2e3-bb236f6d8c7b.png)

詳細な作成方法については以下の公式ドキュメントを参考にしていただければと思います。

https://learn.microsoft.com/ja-jp/azure/devops/pipelines/library/service-endpoints?view=azure-devops&tabs=yaml

## Azure Pipelines の設定

Service Connections の設定が完了したため、Azure Pipelines の設定に移ります。
Azure Pipelines を実行するためには、パイプラインのジョブを定義するための Yaml ファイル（今回は全て `azure-pipelines.yml`）が必要となります。

そのため、一旦適当な初期パイプライン定義を各フォルダ（`arm_template`, `bicep`, `terraform`, `pulumi`）にそれぞれ配置しておきます。
以下のパイプライン定義は Azure Pipelines から「Starter pipelines」を選択した場合に入力される初期パイプライン定義です。今回はまずはこちらを入れておきます。

```yaml: <各フォルダ>/azure-pipelines.yml
# Starter pipeline
# Start with a minimal pipeline that you can customize to build and deploy your code.
# Add steps that build, run tests, deploy, and more:
# https://aka.ms/yaml

trigger:
- main

pool:
  vmImage: ubuntu-latest

steps:
- script: echo Hello, world!
  displayName: 'Run a one-line script'

- script: |
    echo Add other tasks to build, test, and deploy your project.
    echo See https://aka.ms/yaml
  displayName: 'Run a multi-line script'
```

各リポジトリにパイプライン定義が配置できたら、各パイプラインを作成します。
「Azure Pipelines > New Pipelines」を押下し、各 IaC ツールそれぞれに対してパイプラインを作成します。この時、先程リポジトリに登録した`azure-pipelines.yml`の格納場所を指定する必要があるため、間違えないように注意してください。

作成できたら以下のような感じになります。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/c1760811-09cf-4802-b231-d403ae94f419.png)

# ARM Template のパイプライン定義

ここから各 IaC ツールパイプライン定義の実行に移ります。
ARM Template をデプロイする際には、すでに用意されている`AzureResourceManagerTemplateDeployment@3`の Task を使用することでデプロイすることができます。

以下のパラメータを設定することでデプロイできます。

| 項目                           | 説明                                                                                        |
| ------------------------------ | ------------------------------------------------------------------------------------------- |
| deploymentScope                | デプロイのスコープ。今回はリソースグループに対してデプロイするため、`Resource Group` を指定 |
| azureResourceManagerConnection | 上記で作成した Service Connections 名                                                       |
| subscriptionId                 | デプロイしたいサブスクリプション ID                                                         |
| resourceGroupName              | デプロイしたいリソースグループ                                                              |
| location                       | デプロイしたいリージョン                                                                    |
| csmFile                        | デプロイしたい ARM Template テンプレート                                                    |
| csmParametersFile              | デプロイしたい ARM Template テンプレートに対応したパラメータファイル                        |

他のパラメータの詳細については以下のドキュメントをご参照ください。

https://learn.microsoft.com/ja-jp/azure/devops/pipelines/tasks/reference/azure-resource-manager-template-deployment-v3?view=azure-pipelines

最終的に作成されたパイプライン定義は以下になります。
Functions は Storage Account と App Service Plan が存在しないと作成できないため、一番最後のジョブで定義しております。

```yaml:arm_template/azure-pipelines.yml
# 手動実行
trigger: none

pool:
  vmImage: ubuntu-latest

variables:
  serviceConnectionName: 'm2-sakai-service-connection'
  resourceGroupName: 'm2-sakai-rg'
  location: 'Japan East'

stages:
  # ストレージアカウントのデプロイ
  - stage: DeployStorageAccountsStage
    displayName: 'Deploy Storage Accounts'
    jobs:
      - job: DeployStorageAccountsResource
        displayName: 'Deploy Storage Accounts'
        steps:
          - task: AzureResourceManagerTemplateDeployment@3
            displayName: 'Deploy Storage Accounts m2sakaistorageaccount'
            inputs:
              deploymentScope: 'Resource Group'
              azureResourceManagerConnection: $(serviceConnectionName)
              subscriptionId: $(subscriptionId)
              resourceGroupName: $(resourceGroupName)
              location: $(location)
              templateLocation: 'Linked artifact'
              csmFile: 'arm_template/Create_StorageAccount_template.json'
              csmParametersFile: 'arm_template/Create_StorageAccount_parameter.json'
              deploymentMode: 'Incremental'
  # App Service Plan のデプロイ
  - stage: DeployAppServicePlan
    displayName: 'Deploy App Service Plan'
    jobs:
      - job: DeployAppServicePlan
        displayName: 'Deploy App Service Plan Resource'
        steps:
          - task: AzureResourceManagerTemplateDeployment@3
            displayName: 'Deploy AppServicePlan m2-sakai-asp'
            inputs:
              deploymentScope: 'Resource Group'
              azureResourceManagerConnection: $(serviceConnectionName)
              subscriptionId: $(subscriptionId)
              resourceGroupName: $(resourceGroupName)
              location: $(location)
              templateLocation: 'Linked artifact'
              csmFile: 'arm_template/Create_AppServicePlan_template.json'
              csmParametersFile: 'arm_template/Create_AppServicePlan_parameter.json'
              deploymentMode: 'Incremental'
  # Functions のデプロイ
  - stage: DeployFunctionApp
    displayName: 'Deploy Function App'
    jobs:
      - job: DeployFunctionApp
        displayName: 'Deploy Function App Resource'
        steps:
          - task: AzureResourceManagerTemplateDeployment@3
            displayName: 'Deploy Function App m2-sakai-functionapp'
            inputs:
              deploymentScope: 'Resource Group'
              azureResourceManagerConnection: $(serviceConnectionName)
              subscriptionId: $(subscriptionId)
              resourceGroupName: $(resourceGroupName)
              location: $(location)
              templateLocation: 'Linked artifact'
              csmFile: 'arm_template/Create_Functions_template.json'
              csmParametersFile: 'arm_template/Create_Functions_parameter.json'
              deploymentMode: 'Incremental'

```

実行結果は以下のように表示されます。Storage Account と App Service Plan は特に依存関係がないので並列に回しても良いのですが、今回はわかりやすくするためにあえて直列実行としております。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/9420cf04-52a2-0d29-21f4-739a8755cef0.png)

# Bicep のパイプライン定義

続いて Bicep のパイプライン定義を作成します。

と言っても、前回の記事でも紹介した通り、Bicep は ARM Template にコンパイルされてから実行されます。そのため、使用する Task は ARM Template と同じ `AzureResourceManagerTemplateDeployment@3` になります。設定項目も、ARM Template ファイルの部分を Bicep ファイルに変更するだけなので、ここでの詳細の説明は割愛します。

```yaml:arm_template/azure-pipelines.yml
# 手動実行
trigger: none

pool:
  vmImage: ubuntu-latest

variables:
  serviceConnectionName: 'm2-sakai-service-connection'
  resourceGroupName: 'm2-sakai-rg'
  location: 'Japan East'

stages:
  # ストレージアカウントのデプロイ
  - stage: DeployStorageAccountsStage
    displayName: 'Deploy Storage Accounts'
    jobs:
      - job: DeployStorageAccountsResource
        displayName: 'Deploy Storage Accounts'
        steps:
          - task: AzureResourceManagerTemplateDeployment@3
            displayName: 'Deploy Storage Accounts m2sakaistorageaccount'
            inputs:
              deploymentScope: 'Resource Group'
              azureResourceManagerConnection: $(serviceConnectionName)
              subscriptionId: $(subscriptionId)
              resourceGroupName: $(resourceGroupName)
              location: $(location)
              templateLocation: 'Linked artifact'
              csmFile: 'bicep/Create_StorageAccount_template.bicep'
              csmParametersFile: 'bicep/Create_StorageAccount_parameter.bicepparam'
              deploymentMode: 'Incremental'
  # App Service Plan のデプロイ
  - stage: DeployAppServicePlan
    displayName: 'Deploy App Service Plan'
    jobs:
      - job: DeployAppServicePlan
        displayName: 'Deploy App Service Plan Resource'
        steps:
          - task: AzureResourceManagerTemplateDeployment@3
            displayName: 'Deploy AppServicePlan m2-sakai-asp'
            inputs:
              deploymentScope: 'Resource Group'
              azureResourceManagerConnection: $(serviceConnectionName)
              subscriptionId: $(subscriptionId)
              resourceGroupName: $(resourceGroupName)
              location: $(location)
              templateLocation: 'Linked artifact'
              csmFile: 'bicep/Create_AppServicePlan_template.bicep'
              csmParametersFile: 'bicep/Create_AppServicePlan_parameter.bicepparam'
              deploymentMode: 'Incremental'
  # Functions のデプロイ
  - stage: DeployFunctionApp
    displayName: 'Deploy Function App'
    jobs:
      - job: DeployFunctionApp
        displayName: 'Deploy Function App Resource'
        steps:
          - task: AzureResourceManagerTemplateDeployment@3
            displayName: 'Deploy Function App m2-sakai-functionapp'
            inputs:
              deploymentScope: 'Resource Group'
              azureResourceManagerConnection: $(serviceConnectionName)
              subscriptionId: $(subscriptionId)
              resourceGroupName: $(resourceGroupName)
              location: $(location)
              templateLocation: 'Linked artifact'
              csmFile: 'bicep/Create_Functions_template.bicep'
              csmParametersFile: 'bicep/Create_Functions_parameter.bicepparam'
              deploymentMode: 'Incremental'
```

実行結果は以下のように表示されます。ARM Template と同じですね。
ただ、一つ気になることとして、やはり ARM Template にコンパイルしているからか、実行時間が ARM Template と比較して 2 倍弱かかっているように見受けられます。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/a945149e-98a7-def5-fe3d-effcd6060066.png)

# Terraform のパイプライン定義

続いて、Terraform のパイプライン定義を作成します。

Terraform は（Pulumi も）ARM Template や Bicep と違い、Azure ネイティブではないため、デフォルトで実行できるような Task が用意されていません。

そのため、以下の二つの方法を取る必要があります。

- 自身で Terraform をインストールして実行する（Task は CLI を使用できる Task であればなんでも可）
- Terraform を実行できるような Task をインストールして実行する

今回は、後者の方法を取りたいと思います。
Azure DevOps には様々な拡張機能が用意されており、以下から自由にインストールすることができます。

https://marketplace.visualstudio.com/azuredevops

今回はこの中から、ダウンロード数も一定数あり評価も高い [Azure Pipelines Terraform Tasks](https://marketplace.visualstudio.com/items?itemName=charleszipp.azure-pipelines-tasks-terraform)をダウンロードして使用することにします。

こちらをダウンロードすると、Task に `JasonBJohnson.azure-pipelines-tasks-terraform.azure-pipelines-tasks-terraform-cli.TerraformCLI@1` を使用することができ、簡単に実行することができます。
各パラメータの説明は拡張機能のドキュメントをご参照ください。

こうして、最終的に作成されたパイプライン定義が以下となります。

```yaml:arm_template/azure-pipelines.yml
# 手動実行
trigger: none

pool:
  vmImage: ubuntu-latest

variables:
  serviceConnectionName: 'm2-sakai-service-connection'
  resourceGroupName: 'm2-sakai-rg'
  location: 'Japan East'
  terraformVersion: '1.7.5'
  terraformWorkingDirectory: 'terraform'

stages:
  # Terraform Install
  - stage: InstallTerraform
    displayName: 'Install Terraform'
    jobs:
      - job: InstallTerraform
        displayName: 'Install Terraform'
        steps:
          - task: JasonBJohnson.azure-pipelines-tasks-terraform.azure-pipelines-tasks-terraform-installer.TerraformInstaller@1
            displayName: 'Install Terraform $(terraformVersion)'
            inputs:
              terraformVersion: $(terraformVersion)
          - task: JasonBJohnson.azure-pipelines-tasks-terraform.azure-pipelines-tasks-terraform-cli.TerraformCLI@1
            displayName: 'Check Terraform Version'
            inputs:
              command: 'version'
  # Terraform Execute
  - stage: ExecuteTerraform
    displayName: 'Execute Terraform'
    jobs:
      - job: ExecuteTerraform
        displayName: 'Execute Terraform'
        steps:
          - task: JasonBJohnson.azure-pipelines-tasks-terraform.azure-pipelines-tasks-terraform-cli.TerraformCLI@1
            displayName: 'Run terraform init'
            inputs:
              command: 'init'
              workingDirectory: $(terraformWorkingDirectory)
          - task: JasonBJohnson.azure-pipelines-tasks-terraform.azure-pipelines-tasks-terraform-cli.TerraformCLI@1
            displayName: 'Run terraform validate'
            inputs:
              command: 'validate'
              workingDirectory: $(terraformWorkingDirectory)
          - task: JasonBJohnson.azure-pipelines-tasks-terraform.azure-pipelines-tasks-terraform-cli.TerraformCLI@1
            displayName: 'Run terraform plan'
            inputs:
              command: 'plan'
              workingDirectory: $(terraformWorkingDirectory)
              environmentServiceName: $(serviceConnectionName)
              commandOptions: '-out=$(System.DefaultWorkingDirectory)/main.tfplan'
          - task: JasonBJohnson.azure-pipelines-tasks-terraform.azure-pipelines-tasks-terraform-cli.TerraformCLI@1
            displayName: 'Run terraform apply'
            condition: succeeded()
            inputs:
              command: 'apply'
              workingDirectory: $(terraformWorkingDirectory)
              environmentServiceName: $(serviceConnectionName)
              commandOptions: '$(System.DefaultWorkingDirectory)/main.tfplan'
```

実行結果は以下のように表示されます。Terraform の場合、リソースそれぞれを別に分けていないため、一つの stage ですべてのリソースを実行しております。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/3ad4f845-ad4c-aeb3-5cf4-0911f3d14a06.png)

# Pulumi のパイプライン定義

最後に Pulumi のパイプライン定義を作成します。はじめに申し上げておきますが、ここが一番大変でした。

まず、Pulumi も Terraform と同様 Azure DevOps にデフォルトで Task が用意されていないため簡単に実行するためには [Pulumi Azure Pipelines Task](https://marketplace.visualstudio.com/items?itemName=pulumi.build-and-release-task) をダウンロードする必要があります。

こちらをダウンロードすると`Pulumi@1`の Task が使用できるようになり、簡単に実行することができます。

::: note warn

`Pulumi@1` を使用して`pulumi up`コマンドを使用しようとすると、以下のようなエラーが出ることがあります。
おそらく Service Connections 側で認証ができていないようなエラーに見受けられるのですが、私の技量ではこの課題を解決することはできませんでした。
もしご存じの方はコメントしていただけると幸いです。

```
##[error]Error: Endpoint auth data not present: XXXXXXXXXXXXXXXXXXXXXX
```

私はこのエラーを回避するために、`pulumi up`の実行時のみは Azure CLI 実行用の Task を用いて実行することにしております。

```yaml
- task: AzureCLI@2
  displayName: 'Run Pulumi up'
  condition: succeeded()
  inputs:
    azureSubscription: $(serviceConnectionName)
    scriptType: 'bash'
    scriptLocation: 'inlineScript'
    inlineScript: |
      cd $(pulumiWorkingDirectory)
      npm install
      pulumi up --yes
```

:::

また、特にダウンロードは必要ないですが、今回 Pulumi は TypeScript で実装しているため、Node.js のインストールも別途必要となります。
最終的に作成されたパイプライン定義は以下のようになります。

```yaml:arm_template/azure-pipelines.yml
# 手動実行
trigger: none

pool:
  vmImage: ubuntu-latest

variables:
  serviceConnectionName: 'm2-sakai-service-connection'
  resourceGroupName: 'm2-sakai-rg'
  location: 'Japan East'
  pulumiWorkingDirectory: 'pulumi'
  pulumiStack: 'm2-sakai/azure-iac-pulumi/devops'

stages:
  # Node install
  - stage: Install
    displayName: 'Install Node.js'
    jobs:
      - job: Install
        displayName: 'Install Node.js'
        steps:
          - task: NodeTool@0
            displayName: 'Install Node.js'
            inputs:
              versionSpec: '20.x'
  # pulumi Execute
  - stage: PulumiExecute
    displayName: 'Execute Pulumi'
    jobs:
      - job: PulumiExecute
        displayName: 'Execute Pulumi'
        steps:
          - task: Pulumi@1
            displayName: 'Authenticating pulimi'
            inputs:
              command: 'login'
              cwd: '$(pulumiWorkingDirectory)/'
              stack: $(pulumiStack)
              createStack: true
            env:
              PULUMI_ACCESS_TOKEN: $(pulumiAccessToken)
          # Pulumi@1で pulumi upが実行できなかったためAzureCLI側で実行
          - task: AzureCLI@2
            displayName: 'Run Pulumi up'
            condition: succeeded()
            inputs:
              azureSubscription: $(serviceConnectionName)
              scriptType: 'bash'
              scriptLocation: 'inlineScript'
              inlineScript: |
                cd $(pulumiWorkingDirectory)
                npm install
                pulumi up --yes
```

実行結果は以下のように表示されます。Pulumi も Terraform と同様、一度のコマンドですべてリソースをデプロイしているため、stage は一つになっております。
ただ、`npm install` 依存ライブラリを取得している影響で、Terraform と比べるとこちらも実行時間が少し長くなっています。デプロイにかかる時間は Terraform と大差ないようです。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/63a7e53f-37a3-f3b2-fbc9-72d8cd8de814.png)

# 各 IaC ツールのデプロイの結果と比較

最後に各観点において各 IaC ツールの比較をしていきます。
冒頭にも記載しましたが、一概に全てのリソースに当てはまるわけではないことにご留意いただけたらと思います。

|                                | ARM Template | Bicep | Terraform | Pulumi |
| :----------------------------: | :----------: | :---: | :-------: | :----: |
| パイプライン作成・設定の容易さ |      ◎       |   ◎   |    〇     |   〇   |
|   安定性・エラーハンドリング   |      〇      |  〇   |    〇     |   △    |
|          デプロイ時間          |      ◎       |  〇   |    〇     |   △~〇    |

- **パイプライン作成・設定の容易さ**: ARM Template と Bicep は Azure 固有のツールのためすでに Azure DevOps に組み込まれている機能で完結できる一方、Terraform と Pulumi は個別に拡張機能をインストールする必要があるなど少し面倒な点があります。
- **安定性・エラーハンドリング**: ARM Template と Bicep は Azure 固有のツールなので安定性が高いと感じました。一方 Pulumi はエラーハンドリングでかなり苦しんだため、多少扱いずらい点もあると感じました。（もちろんエラー内容にもよりますが）
- **デプロイ時間**: ARM Template と Bicep は Azure 特化のツールであるため、Azure リソースを速やかにデプロイすることができますが、Bicep は ARM Template にコンパイルするため少し時間がかかります。Terraform と Pulumi はマルチクラウドをサポートするため、リソースのプロビジョニングに比較的時間がかかりますが、Pulumiは`npm install`で依存ライブラリを取得しているためその分実行時間が長くなる結果となりました。

個人的には Pulumi が好きなので、コミュニティが大きくなって使いやすくなってくれることを願っています。

# まとめ

いかがでしたでしょうか。
今回の記事では前回比較した IaC ツールを使用し、各ツールを Azure Pipelines を用いて Azure にデプロイする方法について紹介しました。
個人的な感想にはなりますが、やはり ARM Template, Bicep は Azure 専用ということもあり、簡単に始めることができると感じた一方、Terraform や Pulumi は外部から Task をダウンロードしてくる必要があったり、Node.js を使用できるようにしておいたりと比較的作成には時間がかかると感じました（慣れてしまえば速いですが）。

本記事が皆様の一助になれば幸いです。最後まで読んでくださりありがとうございました！
