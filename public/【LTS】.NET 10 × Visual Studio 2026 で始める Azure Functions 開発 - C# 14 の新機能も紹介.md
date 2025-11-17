---
title: 【LTS】.NET 10 × Visual Studio 2026 で始める Azure Functions 開発 - C# 14 の新機能も紹介
tags:
  - C#
  - Azure
  - .NET
  - VisualStudio
  - AzureFunctions
private: false
updated_at: '2025-11-18T08:38:29+09:00'
id: e988886aa3854ffd492f
organization_url_name: nri
slide: false
ignorePublish: false
---

# はじめに

2025 年 11 月 11 日に .NET 10 がリリースされましたね！そして同時期に Visual Studio 2026 もリリースされました。
.NET は長期サポート（LTS）とスタンダード サポート（STS）の 2 つのリリースラインがありますが、今回の .NET 10 は **LTS バージョン** となります。

https://dotnet.microsoft.com/ja-jp/platform/support/policy/dotnet-core

https://learn.microsoft.com/ja-jp/visualstudio/releases/2026/release-notes

.NET 9 が 2024 年 11 月にリリースされてからちょうど 1 年が経過しましたので、順当にバージョンアップした形ですね。

また、Azure Functions も .NET 10 に対応したということで、本記事では Visual Studio 2026 を用いて .NET 10 で Azure Functions アプリケーションを作成してみようと思います。

https://learn.microsoft.com/ja-jp/azure/azure-functions/supported-languages?tabs=isolated-process%2Cv4&pivots=programming-language-csharp

:::note info

.NET 10 は **LTS バージョン** のため、サポート期間は約 3 年（2028 年 11 月まで）となります。
次の LTS バージョンはおそらく .NET 12 であり、 2027 年 11 月にリリースされる予定ですので、本番環境で使用する際はサポート期間を十分に考慮してください。

:::

# Visual Studio 2026 のインストール

.NET 10 での開発には Visual Studio 2026 が **推奨** されます。
これは、.NET 10 のプロジェクトテンプレートや最新の開発ツールが Visual Studio 2026 でのみ提供されているためです。

それではまず、Visual Studio 2026 をインストールします。下記のサイトから Visual Studio 2026 をダウンロードし、インストールします。
この時、自身が使いたいエディションを選択します。Visual Studio は有料エディションでも試用期間が 90 日と長いことが嬉しいポイントです。

| エディション | 説明                                                                    | 無料試用期間 |
| ------------ | ----------------------------------------------------------------------- | ------------ |
| Community    | 学生、オープンソースの共同作成者、個人用の無料で強力な IDE              | 元から無料   |
| Professional | 小規模チームに最適な Professional IDE                                   | 90 日間      |
| Enterprise   | あらゆる規模のチーム向けのスケーラブルなエンドツーエンド ソリューション | 90 日間      |

https://visualstudio.microsoft.com/ja/vs/

2022 版のインストール時と異なるのが、インストールを行う際に **以前のインストールから設定等を引き継いでインストールできること** です。これは嬉しいですね。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/40d25c36-f9bc-4aef-a013-70c66a377777.png)

インストール時のワークロードでは、**「Azure と AI の開発」** を選択しておくことをお勧めします。これにより Azure Functions の開発に必要なツールが一緒にインストールされます。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/38b62a03-3792-4252-8c99-ac1b25b2c01e.png)

これにて、Visual Studio 2026 のインストールは完了です！

# .NET 10 SDK のインストール

続いて、.NET 10 SDK をインストールします。
以下のサイトから .NET 10 SDK をダウンロードします。

https://dotnet.microsoft.com/ja-jp/download/dotnet/10.0

お使いの OS に合わせて、適切なインストーラーをダウンロードし、実行してください。
インストールが完了したら、コマンドプロンプトまたはターミナルで以下のコマンドを実行し、.NET 10 SDK が正しくインストールされていることを確認しましょう。

```bash: Terminal
dotnet --list-sdks
```

以下のように .NET 10.x.x が表示されていれば問題ありません。

```bash: Terminal
8.0.415 [C:\Program Files\dotnet\sdk]
9.0.102 [C:\Program Files\dotnet\sdk]
10.0.100 [C:\Program Files\dotnet\sdk]
```

# .NET 10 の新機能を深掘りする

Azure Functions プロジェクトを作成する前に、.NET 10 と Visual Studio 2026 の新機能について詳しく見ていきましょう。
.NET 10 は **LTS（長期サポート）バージョン** であり、約 3 年間のサポート（2028 年 11 月まで）が提供されます。そのため、本番環境での使用にも安心です。

.NET 10 には、SDK、ライブラリ、ASP.NET Core、Entity Framework Core など、様々なレイヤーで新機能が追加されています。
本記事では、その中でも **Runtime と C# 言語** に焦点を当てて解説します。

:::note info

.NET 10 の新機能をより詳しく知りたい方は、公式ドキュメントの [.NET 10 の新機能](https://learn.microsoft.com/ja-jp/dotnet/core/whats-new/dotnet-10/overview) をご覧ください。

:::

## .NET 10 Runtime の主な新機能

.NET 10 のランタイムには、パフォーマンスを大幅に向上させる多くの改善が含まれています。

### JIT コンパイラの改善

.NET 10 の JIT コンパイラには、以下のような重要な改善が施されています。

| 改善項目                     | 説明                                                                                                             |
| ---------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| インライン化の強化           | 仮想化解除されたメソッドのインライン化が可能になり、最適化の機会が増加                                           |
| コードレイアウトの最適化     | 3-opt ヒューリスティックを使用した新しいコード配置アルゴリズムにより、ホットパスの密度が向上                     |
| 配列インターフェースの最適化 | 配列インターフェースメソッドの仮想化解除とインライン化により、`IEnumerable` 経由の配列列挙のオーバーヘッドが削減 |
| ループ反転の改善             | グラフベースのループ認識により、`for` と `while` ステートメントの最適化の可能性が向上                            |

これらの改善により、.NET 10 アプリケーションは以前のバージョンと比較して、より高速に実行されるようになります。

### スタック割り当ての最適化

.NET 10 では、エスケープ解析が拡張され、より多くの場合でオブジェクトをスタックに割り当てることが可能になりました。

例えば以下のようなコードでは、ラムダ式 `func` もスタックに割り当てられるようになります。

```csharp
public static int Main()
{
    int local = 1;
    int[] arr = new int[100];
    var func = (int x) => x + local;  // .NET 10 ではスタックに割り当て
    int sum = 0;

    foreach (int num in arr)
    {
        sum += func(num);
    }

    return sum;
}
```

これにより、ガベージコレクションの負荷が軽減され、パフォーマンスが向上します。

### NativeAOT の改善

NativeAOT（Ahead-of-Time コンパイル）の型プレイニシャライザーが強化され、`conv.*` および `neg` オペコードのすべてのバリアントをサポートするようになりました。これにより、事前初期化可能なメソッドの範囲が拡大し、ランタイムパフォーマンスがさらに最適化されます。

### AVX10.2 サポート

.NET 10 では、x64 ベースプロセッサ向けに AVX（Advanced Vector Extensions）10.2 のサポートが導入されました。`System.Runtime.Intrinsics.X86.Avx10v2` クラスで新しい組み込み関数が利用可能になります。

:::note warn

AVX10.2 対応ハードウェアがまだ一般的に利用できないため、JIT の AVX10.2 サポートは現在デフォルトで無効になっています。

:::

## C# 14 の新機能

.NET 10 では **C# 14** が使用できます。C# 14 には以下のような新機能が追加されています。

### Field-backed properties（フィールドバックプロパティ）

自動実装プロパティから、カスタムの `get` および `set` アクセサーへの移行がスムーズになりました。`field` コンテキストキーワードを使用することで、明示的なバッキングフィールドを宣言することなく、コンパイラが生成したバッキングフィールドにアクセスできます。

```csharp
public class Person
{
    public string Name
    {
        get;
        set
        {
            if (string.IsNullOrWhiteSpace(value))
                throw new ArgumentException("Name cannot be empty");
            field = value;
        }
    }
}
```

### Extension ブロック

拡張メンバーを定義するための新しい構文が追加され、拡張メソッドに加えて拡張プロパティや静的拡張メンバーを宣言できるようになりました。

```csharp
public static class StringExtensions
{
    // インスタンス拡張メンバー
    extension(string source)
    {
        public bool IsEmpty => string.IsNullOrEmpty(source);
        public int WordCount => source.Split(' ').Length;
    }

    // 静的拡張メンバー
    extension(string)
    {
        public static string DefaultValue => string.Empty;
        public static string Combine(string first, string second) => $"{first} {second}";
    }
}
```

インスタンス拡張メンバーは `text.IsEmpty` のように呼び出し、静的拡張メンバーは `string.DefaultValue` のように呼び出せます。

### Null 条件代入

`?.` 演算子を使用した null 条件代入が可能になりました。

```csharp
person?.Name = "New Name";  // person が null でない場合のみ代入
```

### Partial コンストラクタとイベント

C# 13 で導入された partial メソッドおよびプロパティに加えて、partial インスタンスコンストラクタと partial イベントがサポートされるようになりました。

### `nameof` の機能拡張

`nameof` 式が非バインドジェネリック型（unbound generic types）をサポートするようになりました。

```csharp
string typeName = nameof(List<>);  // "List" を返す
```

## Visual Studio 2026 の新機能

Visual Studio 2026 には、生産性を大幅に向上させる多くの新機能が搭載されています。

### GitHub Copilot の深い統合

Visual Studio 2026 では、AI が **プラットフォームレベル** で深く統合されています。

| 機能                         | 説明                                                                                |
| ---------------------------- | ----------------------------------------------------------------------------------- |
| Adaptive Paste               | コードを貼り付けると、文脈に合わせて自動的に調整。Tab キーで提案を確認できます      |
| Code Actions                 | 右クリックメニューから Copilot に直接アクセス。説明、最適化、コメント生成などが可能 |
| Debugger Agent for Unit Test | 失敗したユニットテストを自動的にデバッグし、修正を提案                              |
| Profiler Agent               | パフォーマンスの問題を自動的に分析し、最適化を提案                                  |
| Enhanced Exception Analysis  | リポジトリコンテキストを活用した、より賢い例外分析                                  |

### デバッグの改善

Visual Studio 2026 では、デバッグ体験が大幅に向上しました。

| 機能                             | 説明                                                                           |
| -------------------------------- | ------------------------------------------------------------------------------ |
| インラインループ変数とパラメータ | デバッグ中に、ループ変数やメソッドパラメータの値がエディタ内に直接表示されます |
| インライン戻り値                 | 関数の戻り値が使用箇所に直接表示され、デバッグがより直感的に                   |
| インライン if ステートメント     | if 条件の評価結果が条件の隣に直接表示されます                                  |
| F5 パフォーマンス向上            | デバッガーの起動時間が最大 30% 高速化                                          |

### Markdown プレビューの強化

Markdown エディタに新しいプレビュー機能が追加されました。

- **プレビューのみモード**: プレビューだけを表示し、大きな画像や複雑な Mermaid ダイアグラムに集中できます
- **ズーム機能**: Mermaid ダイアグラムのプレビュー時にズームイン/アウトが可能

### Profiler Launch Page の刷新

プロファイラーの起動ページが再設計され、より使いやすくなりました。

- ツールの互換性が明確に表示される
- Copilot による状況に応じた推奨機能
- より直感的なレイアウト

# Azure Functions プロジェクトの作成

それでは実際に Visual Studio 2026 を使って.NET 10 で Azure Functions プロジェクトを作成していきます。

## プロジェクトの新規作成

Visual Studio 2026 を起動し、「新しいプロジェクトの作成」を選択します。
プロジェクトテンプレートの一覧から「Azure Functions」を検索し、選択して「次へ」を押下します。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/8b27e539-43fd-41a3-8840-45a973401cc3.png)

プロジェクト名や保存場所を適切に設定して「次へ」を押下します。

## Azure Functions の詳細設定

続いて、Azure Functions の詳細設定を行います。

| 設定項目                                            | 選択値                                | 説明                                           |
| --------------------------------------------------- | ------------------------------------- | ---------------------------------------------- |
| Functions worker                                    | .NET 10.0 Isolated (長期的なサポート) | .NET 10 で実行される分離ワーカープロセスモデル |
| Function                                            | Http trigger                          | HTTP リクエストをトリガーとする関数            |
| ランタイムストレージアカウントに Azurite を使用する | チェック                              | ローカル開発用のストレージエミュレーター       |
| Authorization level                                 | Function                              | 関数キーによる認証を要求                       |

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/b30f0696-5526-4ae9-b4c9-cd299277a792.png)

:::note warn
この時、`.NET 10.0 isolated` が選択できない場合は、Azure Functions Core Tools がインストールされていない or バージョンが古い可能性があります。
その場合、[こちら](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-run-local)からインストールできますので、試していただくことをお勧めします。
:::

「作成」を押下すると、.NET 10 ベースの Azure Functions プロジェクトが作成されます。

:::note info

**分離ワーカープロセスモデルとは**

Azure Functions の開発において、Functions の実行モデルは 2 つ存在します。

| 実行モデル         | 説明                                                                  |
| ------------------ | --------------------------------------------------------------------- |
| インプロセスモデル | 関数コードが Functions ホストプロセスと同じプロセスで実行されるモデル |
| 分離ワーカーモデル | 関数コードが別の .NET ワーカープロセスで実行されるモデル              |

.NET の初期はインプロセスモデルのみの提供でしたが、.NET 5 から分離ワーカーモデルが登場し、現在では**分離ワーカーモデルが推奨**されています。

Functions ランタイムとユーザーコードが別々のプロセスで実行されることで、より柔軟な開発が可能になり、.NET の最新機能を利用できるようになるためです。

そのため、.NET 9 からはインプロセスモデルの提供はありません。.NET 10 も同じく、**分離ワーカーモデルのみの提供** となっています。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/e6c2be40-23ce-4b14-94ea-bb9209ef1bf4.png)

:::

:::note warn

インプロセスモデルのサポートは **2026 年 11 月 10 日に終了します。**
もしまだインプロセスモデルを使用している方は、分離ワーカーモデルにアプリを移行することを強くお勧めします。

https://azure.microsoft.com/ja-jp/updates?id=retirement-support-for-the-inprocess-model-for-net-apps-in-azure-functions-ends-10-november-2026

:::

## 作成されたコードの確認

プロジェクトが作成されると、以下のようなサンプルコードが自動生成されます。

```csharp:Function1.cs
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Azure.Functions.Worker;
using Microsoft.Extensions.Logging;

namespace Dotnet10FunctionApp;

public class Function1
{
    private readonly ILogger<Function1> _logger;

    public Function1(ILogger<Function1> logger)
    {
        _logger = logger;
    }

    [Function("Function1")]
    public IActionResult Run([HttpTrigger(AuthorizationLevel.Function, "get", "post")] HttpRequest req)
    {
        _logger.LogInformation("C# HTTP trigger function processed a request.");
        return new OkObjectResult("Welcome to Azure Functions!");
    }
}
```

このコードは、HTTP GET/POST リクエストを受け付けて "Welcome to Azure Functions!" というメッセージを返すシンプルな関数です。このコードのポイントは以下となります。

| ポイント             | 説明                                                               |
| -------------------- | ------------------------------------------------------------------ |
| 依存性の注入         | コンストラクタで `ILogger` を注入し、ロギング機能を使用            |
| `[Function]` 属性    | 関数の名前を定義                                                   |
| `[HttpTrigger]` 属性 | HTTP トリガーを定義し、認証レベルと HTTP メソッドを指定            |
| 戻り値               | `IActionResult` を返すことで、様々な HTTP レスポンスを柔軟に返せる |

また、Functions 作成時に生成される、csproj ファイルは以下のようになっています。

```xml:xxx.csproj
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <AzureFunctionsVersion>v4</AzureFunctionsVersion>
    <OutputType>Exe</OutputType>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
  </PropertyGroup>

  <ItemGroup>
    <FrameworkReference Include="Microsoft.AspNetCore.App" />
    <PackageReference Include="Microsoft.ApplicationInsights.WorkerService" Version="2.23.0" />
    <PackageReference Include="Microsoft.Azure.Functions.Worker" Version="2.50.0" />
    <PackageReference Include="Microsoft.Azure.Functions.Worker.ApplicationInsights" Version="2.50.0" />
    <PackageReference Include="Microsoft.Azure.Functions.Worker.Extensions.Http.AspNetCore" Version="2.1.0" />
    <PackageReference Include="Microsoft.Azure.Functions.Worker.Sdk" Version="2.0.6" />
  </ItemGroup>

</Project>
```

csproj ファイルには、プロジェクトの設定とパッケージの依存関係が定義されています。それぞれの項目は以下のようになっています。

**PropertyGroup セクション**

| プロパティ            | 設定値  | 説明                                                                                |
| --------------------- | ------- | ----------------------------------------------------------------------------------- |
| TargetFramework       | net10.0 | .NET 10 をターゲットとしている                                                      |
| AzureFunctionsVersion | v4      | Azure Functions ランタイムのバージョン 4 を使用する                                 |
| OutputType            | Exe     | 実行可能ファイル（`.exe`）として出力される（分離ワーカーモデルで必要）              |
| ImplicitUsings        | enable  | 暗黙的な `using` ディレクティブを有効化し、一般的な名前空間を自動的にインポートする |
| Nullable              | enable  | `null` 許容参照型を有効化し、より安全なコードを書けるよう                           |

**ItemGroup セクション**

| パッケージ                                                  | 説明                                                                        |
| ----------------------------------------------------------- | --------------------------------------------------------------------------- |
| Microsoft.AspNetCore.App                                    | ASP.NET Core のフレームワーク参照（`HttpRequest` / `IActionResult` を使用） |
| Microsoft.ApplicationInsights.WorkerService                 | Application Insights によるアプリケーション監視機能を提供                   |
| Microsoft.Azure.Functions.Worker                            | 分離ワーカーモデルのコア機能を提供                                          |
| Microsoft.Azure.Functions.Worker.ApplicationInsights        | 分離ワーカーモデルでの Application Insights 統合                            |
| Microsoft.Azure.Functions.Worker.Extensions.Http.AspNetCore | HTTP トリガーで ASP.NET Core の統合機能を提供（`HttpRequest` の使用など）   |
| Microsoft.Azure.Functions.Worker.Sdk                        | Functions のビルドとデプロイに必要な SDK                                    |

# ローカルでの実行確認

それでは作成した Azure Functions をローカル環境で実行してみましょう。

Visual Studio のツールバーから「F5」キーを押すか、「デバッグの開始」ボタンを押下します。
すると、Azure Functions Core Tools が起動し、以下のようなコンソールが表示されます。

```bash
Azure Functions Core Tools
Core Tools Version:       4.x.xxxx Commit hash: N/A (64-bit)
Function Runtime Version: 4.x.x.xxxxx

Functions:

        Function1: [GET,POST] http://localhost:7071/api/Function1

For detailed output, run func with --verbose flag.
```

表示された URL（`http://localhost:7071/api/Function1`）にブラウザまたは curl コマンドでアクセスしてみましょう。

```bash: Terminal
curl http://localhost:7071/api/Function1
```

以下のようなレスポンスが返ってくれば成功です！

```
Welcome to Azure Functions!
```

# Azure へのデプロイ

ローカルでの動作確認が取れたところで、実際に Azure にデプロイしてみましょう。

## Function App の作成

Azure Portal にアクセスし、Function App を作成します。
`ホーム > Function App` と移動し、`＋作成`を押下して以下のように設定します。
Azure 上ではまだ Preview となっていますが、近いうちに GA されるでしょう。

| 設定項目                 | 設定値                                     |
| ------------------------ | ------------------------------------------ |
| サブスクリプション       | 任意のサブスクリプション                   |
| リソースグループ         | 新規作成または既存のものを選択             |
| 関数アプリ名             | 任意の名前                                 |
| ランタイムスタック       | .NET                                       |
| バージョン               | 10 (LTS), isolated worker model（Preview） |
| 地域                     | Japan East など                            |
| オペレーティングシステム | Windows または Linux                       |
| プランの種類             | App Service プラン                         |

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/0e828ac7-3236-486d-b695-64a3a7b2453d.png)

「確認および作成」を押下し、Function App を作成します。

:::note info

**プランの種類について**

- **従量課金（Consumption）**: 実行時間とメモリ使用量に基づいて課金
- **Premium**: 予め確保されたインスタンスで実行。高パフォーマンス
- **App Service プラン**: 既存の App Service プランを使用

:::

## デプロイされた関数の動作確認

Azure Portal で Function App に移動し、「関数」から作成した関数を選択します。
「コードとテスト」タブから「テスト/実行」を選択し、テストを実行してみましょう。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/2ad00964-10d3-464c-9f63-96d2124e45a4.png)

"Welcome to Azure Functions!" というレスポンスが返ってくれば、デプロイ成功です！

また、関数の URL にブラウザからアクセスすることもできます。
ただし、Authorization level を Function に設定しているため、関数キーをクエリパラメータとして付与する必要があります。

```
https://m2-sakai-dotnet10-functionapp.azurewebsites.net/api/Function1?code=xxxxxxxxxx
```

関数キーは、Azure Portal の「関数」→「関数キー」から確認できます。

# .NET 10 と C# 14 の新機能を試してみる

それでは、実際に .NET 10 と C# 14 の新機能を使った関数を作成してみましょう。
今回は以下のようにコードを修正します。

```csharp:Function1.cs
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Azure.Functions.Worker;
using Microsoft.Extensions.Logging;

namespace FunctionApp1
{
    public class Function1
    {
        private readonly ILogger<Function1> _logger;

        public Function1(ILogger<Function1> logger)
        {
            _logger = logger;
        }

        [Function("Function1")]
        public IActionResult Run([HttpTrigger(AuthorizationLevel.Function, "get", "post")] HttpRequest req)
        {
            _logger.LogInformation("C# HTTP trigger function processed a request.");

            // C# 14 の field-backed property を使った例
            var functionInfo = new FunctionInfo
            {
                Name = "Function1",
                Version = Environment.Version.ToString(),
                Runtime = ".NET 10",
                Timestamp = DateTime.UtcNow
            };

            return new OkObjectResult(new
            {
                message = "Welcome to Azure Functions with .NET 10!",
                functionInfo,
                features = new[]
                {
                    ".NET 10 LTS Runtime",
                    "C# 14 Language",
                    "Azure Functions v4",
                    "Isolated Worker Model"
                }
            });
        }
    }

    // C# 14 の field-backed property を使用
    public class FunctionInfo
    {
        // field キーワードで検証ロジック付きプロパティを簡潔に記述
        public string Name
        {
            get;
            set
            {
                if (string.IsNullOrWhiteSpace(value))
                    throw new ArgumentException("Name cannot be empty");
                field = value;
            }
        }

        public string Version
        {
            get;
            set
            {
                if (string.IsNullOrWhiteSpace(value))
                    throw new ArgumentException("Version cannot be empty");
                field = value;
            }
        }

        public string Runtime { get; set; } = string.Empty;
        public DateTime Timestamp { get; set; }
    }
}
```

この例では、以下の C# 14 の新機能を使用しています。

| 機能                    | 使用箇所                                    | 説明                                                                                               |
| ----------------------- | ------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| Field-backed properties | `FunctionInfo.Name` と `Version` プロパティ | `field` キーワードを使用することで、明示的なバッキングフィールドを宣言せずに検証ロジックを追加可能 |

再度デプロイして動作を確認してみます。
新しい関数を実行すると、以下のようなレスポンスが返ってきます。

```json
{
  "message": "Welcome to Azure Functions with .NET 10!",
  "functionInfo": {
    "name": "Function1",
    "version": "10.0.0",
    "runtime": ".NET 10",
    "timestamp": "2025-11-14T12:34:56.789Z"
  },
  "features": [
    ".NET 10 LTS Runtime",
    "C# 14 Language",
    "Azure Functions v4",
    "Isolated Worker Model"
  ]
}
```

# おわりに

いかがでしたでしょうか。
本記事では、.NET 10 と Visual Studio 2026 を使用して Azure Functions アプリケーションを作成する方法を紹介しました。

.NET 10 は **LTS（長期サポート）バージョン** であり、2028 年 11 月 14 日までの約 3 年間サポートが提供されます。そのため、本番環境での使用にも安心です。
また、C# 14 の新機能（field-backed properties、extension ブロックなど）や Visual Studio 2026 の AI 統合機能（Adaptive Paste、Debugger Agent など）により、開発体験が大幅に向上しています。

特に注目したいのは、分離ワーカーモデルへの完全移行です。.NET 10 では、より柔軟で高速な実行環境が提供され、最新の .NET 機能をフルに活用できるようになりました。
一方で、インプロセスモデルは 2026 年 11 月 10 日にサポート終了となるため、既存のアプリケーションをお持ちの方は、計画的な移行をお勧めします。

Visual Studio 2026 の GitHub Copilot 統合は、コード作成からデバッグ、パフォーマンス最適化まで、開発のあらゆる場面で AI がサポートしてくれます。
最新技術に触れることで新しい知識を身につけることができますし、将来的なバージョンアップにもスムーズに対応できるようになります。

みなさんもぜひ .NET 10、C# 14、Visual Studio 2026 を使って、Azure Functions での開発を楽しんでみてください！

最後まで読んでいただき、ありがとうございました。

# 参考文献

- [.NET 10 のサポート ポリシー](https://dotnet.microsoft.com/ja-jp/platform/support/policy/dotnet-core)
- [Visual Studio 2026 リリースノート](https://learn.microsoft.com/ja-jp/visualstudio/releases/2026/release-notes)
- [Azure Functions のドキュメント](https://learn.microsoft.com/ja-jp/azure/azure-functions/)
- [Azure Functions でサポートされている言語](https://learn.microsoft.com/ja-jp/azure/azure-functions/supported-languages?tabs=isolated-process%2Cv4&pivots=programming-language-csharp)
- [C# 14 の新機能](https://learn.microsoft.com/ja-jp/dotnet/csharp/whats-new/csharp-14)
- [.NET 10 の新機能](https://learn.microsoft.com/ja-jp/dotnet/core/whats-new/dotnet-10/overview)
- [Azure Functions の .NET 分離ワーカー プロセス ガイド](https://learn.microsoft.com/ja-jp/azure/azure-functions/dotnet-isolated-process-guide)
