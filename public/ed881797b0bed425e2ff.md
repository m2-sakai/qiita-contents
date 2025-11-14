---
title: Bicepのバージョン管理とモジュール化の促進
tags:
  - Azure
  - IaC
  - AzureContainerRegistry
  - AzureDevOps
  - Bicep
private: false
updated_at: '2025-05-07T07:49:17+09:00'
id: ed881797b0bed425e2ff
organization_url_name: nri
slide: false
ignorePublish: false
---
# はじめに

皆様、いかがお過ごしでしょうか。
私は以前から Azure の IaC ツールである Bicep についてよく触っているのですが、最近以下のようなことを感じていました。

- **自分が作った Bicep のモジュールを他の人（チーム）に展開したい場合ってどうするの？**
- **展開したいけど、バージョン管理ってどうするの？**
- **Bicep って Java でいう [Maven Central Repository](https://central.sonatype.com/)、JavaScript でいう [npm Registry](https://docs.npmjs.com/cli/v8/using-npm/registry) みたいなパッケージ管理ツールってないの？**
- **そもそも Bicep ってどういう単位でバージョン管理するのがいいの？**

という疑問を持ったわけです。
今回はそんな同じような課題を抱えている方や、Bicep を学んでいる方に対して少しでもお役に立てればと、**Bicep のバージョン管理方法**について検討していこうと思います。

その前に、**Azure の IaC って他に何があるの？そこから知りたい！** という方は、以下の記事を出したためこちらもご覧いただけると大変嬉しいです。

https://qiita.com/s_w_high/items/534a6add2a37b172a6bb

# Bicep とは

先程、Azure の IaC ツールについては[私の以前の記事](https://qiita.com/s_w_high/items/534a6add2a37b172a6bb)を掲載しましたが、Bicep については本記事でも簡単に紹介しておきます。

Bicep は、ARM Template の後継として **Microsoft が提供している新しい DSL（Domain Specific Language）ベースの IaC ツール**です。

ARM Template とは異なり、Bicep は JSON ではなく、**独自のより簡潔な構文**を提供しており、これにより Azure リソースの管理を容易にしています。

Bicep のコードは **ARM Template にコンパイルされる**ため、同様に ARM REST API を通じて Azure リソースに対する操作を行います。
これにより、ARM Template と同等の Azure リソースのサポートと、新サービスや機能の迅速なサポートを実現しています。

https://learn.microsoft.com/ja-jp/azure/azure-resource-manager/bicep/overview?tabs=bicep

# Bicep のモジュール化

本題に入る前に、Bicep には**モジュール**という機能があります。

https://learn.microsoft.com/ja-jp/azure/azure-resource-manager/bicep/modules

モジュールとは、**Bicep ファイルを別の Bicep ファイルから参照し、特定のリソースや構成を再利用するための単位**です。モジュール化し、複数回呼び出したり異なるパラメーターを渡したりすることで、基本的なリソース構築のパターンを統一することができます。

例えば、仮想ネットワーク(VNet)を作成する専用のテンプレートをモジュール化すれば、別々のファイルからそのモジュールを呼び出すことで仮想ネットワークを簡単に作成することができ、設定を変更したい時はそのモジュールだけを編集すれば呼び出している全てのファイルに変更を反映できます。

以下のようなフォルダ構成の場合を想定します。

```text
├── main.bicep       # メインのBicepファイル
├── vnetModule.bicep # VNet構成用のモジュール
```

この時、`vnetModule.bicep` は以下のような仮想ネットワークを作成する Bicep ファイルです。

```ts: vnetModule.bicep
@description('仮想ネットワークのリソース名')
param vnetName string

@description('仮想ネットワークのIPアドレスプレフィックス')
param addressPrefix string

resource vnet 'Microsoft.Network/virtualNetworks@2024-05-01' = {
  name: vnetName
  location: resourceGroup().location
  properties: {
    addressSpace: {
      addressPrefixes: [
        addressPrefix
      ]
    }
  }
}

output vnetId string = vnet.id
```

`main.bicep` ファイルでは、`module` と宣言することで先程の `vnetModule.bicep` を読み込み、仮想ネットワークを作成することができます。

```ts: main.bicep
@description('仮想ネットワークのリソース名')
param vnetName string = 'myVnet'

@description('仮想ネットワークのIPアドレスプレフィックス')
param addressPrefix string = '10.0.0.0/16'

module vnet 'vnetModule.bicep' = { // `vnetModule.bicep` を読み込んで仮想ネットワークを作成する
  name: 'vnetDeployment'
  params: {
    vnetName: vnetName
    addressPrefix: addressPrefix
  }
}

output vnetId string = vnet.outputs.vnetId
```

このように、パラメータ（`vnetName`, `addressPrefix`）を変更するだけで、別々のファイルから仮想ネットワークを作成することができます。
モジュールは他の言語でいう関数のようなもので Bicep コードの再利用性や保守性を高める重要な機能です。
モジュールを効果的に使用することで、**複雑な構築プロセスを簡素化し、プロジェクトの一貫性を保つことができます。**

# Bicep のバージョン管理

それでは本題に移ります。
今回は以下のようなケースを考えます。

- 「A」Git リポジトリで Bicep のモジュール集を作成する
- そのモジュールを他のプロジェクト（以下でいう「B」Git リポジトリ）で使用する

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/ce184fe9-7f3a-44bc-bedb-bf250d490932.png)

[Bicep のモジュール化](#bicep-のモジュール化)に記載した通り、Bicep はファイル単位で読み込むことができるため、バージョン管理をする単位として、以下の 2 通りが挙げられます。

| バージョン管理単位                                                                 | 説明                                                                                                                                                                                                                                     | バージョン管理方法                                                                                                                                                                                                                                           |
| ---------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| [リポジトリ単位](#git-サブモジュールを利用しリポジトリ単位でバージョニングする)    | **Git リポジトリ全体**でバージョン管理する<br><font color="blue">**リポジトリ全体としてバージョンが統一される**</font>が、<font color="red">**変更がないファイルも一緒にバージョンが上がってしまう**</font>                              | [**Git サブモジュール**](https://git-scm.com/book/ja/v2/Git-%E3%81%AE%E3%81%95%E3%81%BE%E3%81%96%E3%81%BE%E3%81%AA%E3%83%84%E3%83%BC%E3%83%AB-%E3%82%B5%E3%83%96%E3%83%A2%E3%82%B8%E3%83%A5%E3%83%BC%E3%83%AB) の機能を利用して他の Git リポジトリに組み込む |
| [ファイル単位](#azure-container-registry-を利用しファイル単位でバージョニングする) | **Git リポジトリの各 Bicep ファイルごと**にバージョン管理する<br><font color="blue">**各ファイルごとにバージョンを定められるため柔軟ではある**</font>が、<font color="red">**どのファイルがどのバージョンなのかという管理が煩雑**</font> | [**Azure Container Registry**](https://azure.microsoft.com/ja-jp/products/container-registry) を利用して Bicep ファイルを管理する                                                                                                                            |

## Git サブモジュールを利用し、リポジトリ単位でバージョニングする

この方法は、リポジトリ全体でバージョニングを行い、[Git サブモジュール](https://git-scm.com/book/ja/v2/Git-%E3%81%AE%E3%81%95%E3%81%BE%E3%81%96%E3%81%BE%E3%81%AA%E3%83%84%E3%83%BC%E3%83%AB-%E3%82%B5%E3%83%96%E3%83%A2%E3%82%B8%E3%83%A5%E3%83%BC%E3%83%AB)の機能で他のリポジトリに組み込む方法です。イメージは以下のようになります。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/70466c97-f144-4503-a364-452f3407374c.png)

以下に詳細な手順を記載します。
まず、B リポジトリを `git clone`し、以下のコマンドで A リポジトリをサブモジュールとして追加します。

```bash: Terminal
git submodule add <A リポジトリの URL>
```

すると B リポジトリにサブモジュールとして A リポジトリと、 `.gitmodules` ファイルが追加されます。
もし Tag でバージョン管理をしている場合は、以下のように `.gitmodules`に Tag 名を指定しておくと、現在取り込まれているサブモジュールのバージョンが明確になります。

```yaml: .gitmodules
[submodule "<A リポジトリ名>"]
    path = <A リポジトリ名>
    url = <A リポジトリの URL>
    tag = v1.0.0 # ここに取り込みたいバージョンを記載する
```

取り込んでいる A リポジトリのバージョンを更新したい場合は、`.gitmodules` の `tag` を新しいバージョンに変更した上で以下のコマンドを実行します。

```bash: Terminal
git submodule update --remote
```

以上が、リポジトリ単位でバージョン管理する方法となります。

## Azure Container Registry を利用し、ファイル単位でバージョニングする

この方法は、ファイル単位でバージョニングを行い、そのファイルを Azure Container Registry に保存しておく方法です。イメージは以下のようになります。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/a37c228f-8910-4bd4-8019-23438b8841a7.png)

そもそも Azure Container Registry ってコンテナイメージのレジストリだよね？と思う方もいらっしゃるかと思いますが、Bicep ファイルも管理することが可能です。

https://learn.microsoft.com/ja-jp/azure/azure-resource-manager/bicep/private-module-registry?tabs=azure-powershell

:::note warn

Azure Container Registry を使用するには、以下のバージョンが必要になりますので、ご注意ください。

- Bicep CLI ：**0.4.1008 以降**
- Azure CLI で使用する場合：**2.31.0 以降**
- Azure PowerShell で使用する場合：**7.0.0 以降**

:::

以下に詳細な手順を記載します。私は今回 Azure CLI を使用します。
まず、Azure Container Registry を作成します。料金プランはどれでも構いませんが、プライベートエンドポイントを利用したい場合は Premium である必要があります。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/a97d4970-85e9-4045-aa1b-6013f584b7a2.png)

次に、デプロイする Bicep ファイルを用意します。今回は [Bicep のモジュール化](#bicep-のモジュール化)で用意した `vnetModule.bicep` を使用します。

まず、デプロイする前に Azure Container Registry へ認証するために、Azure CLI をログイン状態にしておきます。

```bash: Terminal
az login
```

次に、以下のコマンドで Bicep ファイルをデプロイします。

```bash: Terminal
az bicep publish --file vnetModule.bicep --target br:<Azure Container Registryのリソース名>.azurecr.io/bicep/modules/vnet:<バージョン>
```

:::note info
`az bicep publish`のリファレンスについては、[こちら](https://learn.microsoft.com/ja-jp/cli/azure/bicep?view=azure-cli-latest#az-bicep-publish)をご参照ください。
:::

すると、Azure Container Registry に以下のように保存されていることが確認できます。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/34848ac8-e1e4-49f7-8485-7da709f19087.png)

マニフェストは以下のようになっています。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/e03f0026-f35c-4181-a17f-e79ea0efd3e3.png)

次に、この保存したモジュールを `main.bicep` ファイルから呼び出します。

```diff_tsx: main.bicep
@description('仮想ネットワークのリソース名')
param vnetName string = 'myVnet'

@description('仮想ネットワークのIPアドレスプレフィックス')
param addressPrefix string = '10.0.0.0/16'

- module vnet 'vnetModule.bicep' = {
+ module vnet 'br:myazurecontainerregisry.azurecr.io/bicep/modules/vnet:v1.0.0' = { // Azure Container Registry の URL にする
  name: 'vnetDeployment'
  params: {
    vnetName: vnetName
    addressPrefix: addressPrefix
  }
}

output vnetId string = vnet.outputs.vnetId
```

また余談ですが、もし Bicep のパラメータファイル（`.bicepparam`）にこのファイルを指定したい場合は、以下のように `using` 句でも指定可能です。

```ts: main.bicepparam
using 'br:myazurecontainerregisry.azurecr.io/bicep/modules/vnet:v1.0.0'
```

:::note info

Bicep のパラメータファイルについては[こちら](https://learn.microsoft.com/ja-jp/azure/azure-resource-manager/bicep/parameters)をご参照ください。

:::

以上が、Azure Container Registry を用いてファイル単位でバージョニングする方法となります。

# おわりに

いかがでしたでしょうか。今回は Bicep のバージョン管理について記事にしてみました。

Bicep には、[Maven Central Repository](https://central.sonatype.com/) や [npm Registry](https://docs.npmjs.com/cli/v8/using-npm/registry) のように公開されているパッケージ管理ツールはないものの、**Azure Container Registry を利用すれば、自組織に簡単に Bicep ファイルを配布することができます。**
Azure Container Registry を使う場合はタグを打ったら自動的にデプロイされる、といったような CI/CD パイプラインを構築することになりそうですね。

リポジトリ単位でバージョン管理するか、ファイル単位でバージョン管理するかはプロジェクト次第になるとは思いますが、皆様もサブモジュールや Azure Container Registry を利用して、Bicep の共通化を促進していきましょう！

最後まで読んでくださり、ありがとうございました。
