---
title: Playwright MCPとAWS MCPのクライアントをVercel AI SDKで構築する
tags:
  - AWS
  - AI
  - MCP
  - Playwright
  - Vercel
private: false
updated_at: '2025-04-10T13:34:22+09:00'
id: 4f97b569415ba32f7275
organization_url_name: nri
slide: false
ignorePublish: false
---
# はじめに

昨今 AI エージェントの流れが本当に激しいですね。
特に最近は MCP 関連の話題に溢れているなと感じているところです。

先週末には Azure Functions で MCP サーバーを作成できるようになりましたね。
私は普段 Azure を触ることが多く、このニュースがきっかけで MCP を触ろうと思ったので、非常に大きな出来事でした。

https://techcommunity.microsoft.com/blog/appsonazureblog/build-ai-agent-tools-using-remote-mcp-with-azure-functions/4401059

「MCP」とは「Model Context Protocol」の略で、**生成 AI モデルに文脈情報を渡しやすくするための規格**です。生成 AI 界の USB-C とも言われていますね。
すでに見られている方も多いかと思いますが、みのるんさんが以下のスライドで大変わかりやすくまとめてくださってます。私もこれで勉強しましたので、非常におすすめです！

https://speakerdeck.com/minorun365/yasasiimcpru-men

ずっと MCP や AI エージェントを触りたいと思っては実行に移せていなかったのですが、今週からようやく触れることができましたので、備忘録も兼ねながら少しずつ記事にしたいと思います。

# 今回やりたいこと

今回は以下のようなことをやっていこうと思います。

- **（サブ）MCP サーバーを Azure Functions で作成する**
- **（メイン）Vercel AI SDK を MCP クライアントツールとして使い、様々な MCP サーバーを呼び出してみる**

[Vercel AI SDK](https://sdk.vercel.ai/docs/introduction) は、3/21 に[バージョン 4.2](https://vercel.com/blog/ai-sdk-4-2) がリリースされ、MCP がサポートされました。
実験的な機能もまだまだございますが、これにより Vercel AI SDK を利用する方は、MCP のクライアント機能を簡単にアプリケーションに組み込み、MCP サーバーが提供するツールを活用できることになります。

今回実装したアプリケーションは GitHub で公開しております。
突貫で作ったので参考にする程度に見ていただけると幸いです...

https://github.com/m2-sakai/azure-functions-mcp

https://github.com/m2-sakai/vercel-ai-sdk-mcp

:::note warn
今回 Vercel AI SDK で作成した MCP クライアントから、**自作した Azure Functions の MCP サーバーは実行できておりません。**

**GitHub Copilot や [MCP Inspector](https://github.com/modelcontextprotocol/inspector) からは接続できている**のですが、アプリケーションからだと実行時にフリーズしてしまいます...

もしその部分が今後完成したらこちらの記事も更新したいと思っておりますので、この点についてはご容赦いただければと思います。
:::

# 完成したもの

今回完成したものは以下のようなチャットアプリとなります。
メッセージを送信し、AI から返事が返ってくるシンプルなアプリケーションです。今回はこのアプリケーションに MCP クライアントを構築しています。
以下の画像だと、[playwright-mcp](https://github.com/microsoft/playwright-mcp) を呼び出して特定の Web サイトの情報を抽出させて AI に回答させています。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/4f3ce760-f407-4229-b2cf-a5e9310746b7.png)

# MCP サーバーを Azure Functions で作成する

本題に入る前に、Azure Functions で MCP サーバーを作成したので、こちらを紹介します。
すでにたくさんの方が記事にされていますが、私も現在時刻を取得する簡単な MCP サーバーを作成します。

## Azure Functions プロジェクトの作成 & 各種インストール

まずは Azure Functions のプロジェクトを作成します。公式ドキュメントに手順が記載されておりますので、それに沿って作成します。
今回私は Visual Studio Code で作成し、言語は C#を選択しています。

https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-develop-vs-code?tabs=node-v4%2Cpython-v2%2Cisolated-process%2Cquick-create&pivots=programming-language-csharp

:::note info
MCP サーバーは .NET8 以降でないと動作しないため、**それより下位のバージョンは使用できません。**
また、プロジェクト作成時にどのトリガーで作るか迷われる方もいらっしゃるかと思いますが、ひとまず**HTTP トリガー**で作っておけば特段問題はございません。
:::

続いて、**MCP 拡張機能**をインストールします。現在はプレビュー版となっています。

```bash: Terminal
dotnet add package Microsoft.Azure.Functions.Worker.Extensions.Mcp --version 1.0.0-preview.1
```

MCP 拡張機能は Blob ストレージを使用するため、Azurite をローカル開発用にインストールします。Visual Studio Code の拡張機能から [Azurite](https://marketplace.visualstudio.com/items?itemName=Azurite.azurite) をインストールし、コマンドパレットから「Azurite: Start」で起動します。

また、`local.settings.json` を以下のように書き換えます。ポイントは `AzureWebJobsStorage` の部分です。

```diff_json: local.settings.json
{
  "IsEncrypted": false,
  "Values": {
+   "AzureWebJobsStorage": "UseDevelopmentStorage=true",
    "FUNCTIONS_WORKER_RUNTIME": "dotnet-isolated"
  }
}
```

また、ここで `<プロジェクト名>.csproj` の一部を変更します。
私の環境だと、公式で作成した Azure Functions のテンプレートの `Microsoft.Azure.Functions.Worker.Sdk` の `v2.0.0` だと動作しませんでした。そのため、以下のようにバージョンを `v2.0.2` にあげる必要があります。

:::note warn
この部分については他の方の記事には存在しなかったため、動作環境起因の可能性があります。
:::

```diff_xml: <プロジェクト名>.csproj
-    <PackageReference Include="Microsoft.Azure.Functions.Worker.Sdk" Version="2.0.0" />
+    <PackageReference Include="Microsoft.Azure.Functions.Worker.Sdk" Version="2.0.2" />

```

## Azure Functions MCP サーバーの開発

続いて、現在時刻を取得するための MCP ツールを作成します。

```csharp: MyMcpTools.cs
namespace AzureFunctionsMcp
{
    public class MyMcpTools
    {
        [Function(nameof(GetCurrentTime))]
        public string GetCurrentTime(
        [McpToolTrigger("getcurrenttime", "Gets the current time. If no timezone is specified, the tool will return the time in UTC.")] ToolInvocationContext context,
        [McpToolProperty("timezone", "string", "The name of the timezone.")] string timezone = "UTC"
        )
        {
            try
            {
                TimeZoneInfo timeZoneInfo = TimeZoneInfo.FindSystemTimeZoneById(timezone);
                DateTime currentTime = TimeZoneInfo.ConvertTimeFromUtc(DateTime.UtcNow, timeZoneInfo);
                var response = new
                {
                    timezone = timeZoneInfo.StandardName,
                    time = currentTime.ToString("yyyy-MM-dd HH:mm:ss", CultureInfo.InvariantCulture),
                    displayName = timeZoneInfo.DisplayName
                };
                return JsonSerializer.Serialize(response);
            }
            catch (TimeZoneNotFoundException)
            {
                return $"The timezone '{timezone}' was not found.";
            }
            catch (InvalidTimeZoneException)
            {
                return $"The timezone '{timezone}' is invalid.";
            }
            catch
            {
                return "Could not get the current time.";
            }
        }
    }
}
```

次に、分離ワーカーモデルのエントリーポイントとなる `Program.cs`に MCP ツールを登録します。

```diff_java: Program.cs
var builder = FunctionsApplication.CreateBuilder(args);

builder.ConfigureFunctionsWebApplication();
+ builder.EnableMcpToolMetadata();

+ builder.ConfigureMcpTool(nameof(MyMcpTools.GetCurrentTime))
+      .WithProperty("timezone", "string", "The timezone.");

builder.Build().Run();
```

## MCP サーバーの起動

実行環境の準備が整いましたら、MCP サーバーを起動します。以下のコマンドで関数を起動します。

```bash: Terminal
func start
```

しばらくすると以下のような出力結果となれば起動成功です。MCP サーバーは次のアドレスでアクセス可能になります。
`http://localhost:7071/runtime/webhooks/mcp/sse`

```bash: Terminal
Functions:

	GetCurrentTime: mcpToolTrigger
```

:::note info

きちんと MCP サーバーが動作しているのか確認したい場合は、GitHub Copilot Agent から呼び出すか、以下の MCP Inspector を利用すると良いかと思います。
https://github.com/modelcontextprotocol/inspector

```bash: Terminal
npx @modelcontextprotocol/inspector
```

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/e7634d15-a481-4cb6-88fb-77f447bf56be.png)

:::

## MCP サーバーを Azure にデプロイ

それでは、作成した MCP サーバーを Azure にデプロイします。Azure ポータルで Functions を作成下のち、以下のコマンドを実行します。

:::note warn
MCP サーバーの Azure Functions は現在 **Windows で動作しない**という情報が散見されますので、作成する際は **Linux** で作成しましょう。

https://github.com/Azure-Samples/remote-mcp-functions-dotnet/issues/8
:::

```bash: Terminal
func azure functionapp publish <your-function-app-name>
```

Azure Functions で起動後は次のアドレスでアクセス可能になります。
`https://<your-function-app-name>.azurewebsites.net/runtime/webhooks/mcp/sse`

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/4eb411a1-6222-4f87-8f86-6d0c8ff13413.png)

:::note info

Azure Functions の MCP サーバーを実行する際は、**`mcp_extension`システムキー**が必要になります。「アプリ キー」のタブから忘れずに取得しておいてください。

:::

これにて、前半の MCP サーバーを Azure Functions で作成するフェーズは終了です。

# Vercel AI SDK を MCP クライアントツールとして使い、様々な MCP サーバーを呼び出してみる

それではメインの方になります。MCP の話題だとどうしてもサーバー側の話が多いかと思いますが、それを呼び出すためのクライアントが今後はどんどん出てくると思っています。

## プロジェクトのセットアップ & 各種インストール

Vercel ということで、Next.js でセットアップを行います。
詳しい手順は以下の公式ドキュメントを参照ください。

https://nextjs.org/docs/app/getting-started/installation

```bash: Terminal
npx create-next-app@latest
```

次に、今回は Vercel AI SDK を OpenAI で使用するため、必要なライブラリをインストールしておきます。

```bash: Terminal
npm i ai @ai-sdk/react @ai-sdk/openai zod
```

OpenAI で LLM を利用するためには **API キー**が必要です。事前に以下の URL から API キーを取得しておきます。選択するモデルにもよりますが、LLM の使用には料金が発生する場合があるため利用する際はご注意ください。

https://platform.openai.com/api-keys

API キーを取得できたら、`.env` に環境変数として入れておきます。[Vercel AI SDK の OpenAI Provider](https://sdk.vercel.ai/providers/ai-sdk-providers/openai) を利用するため、デフォルトの `OPENAI_API_KEY` というキーで値を入れておきます。

:::note alert
くれぐれも API キーは GitHub や他クラウドに Upload しないでください。
:::

```bash: .env
OPENAI_API_KEY=<取得したキー>
```

## 利用する MCP サーバーの起動

ここまででプロジェクトのセットアップと各種インストールが終了したので、利用する MCP サーバーを起動しておきます。

今回起動する MCP サーバーは以下の 3 つとなります。

- [MCP サーバーを Azure Functions で作成する](#mcp-サーバーを-azure-functions-で作成する)で作成した Azure Functions：現在時刻を取得する
- [AWS MCP](https://awslabs.github.io/mcp/)の[AWS Documentation MCP Server](https://awslabs.github.io/mcp/servers/aws-documentation-mcp-server/)：AWS のドキュメントにアクセスし、推奨する回答を返却する
- [Playwright MCP](https://github.com/microsoft/playwright-mcp)：ブラウザにアクセスし、情報を取得する

### AWS Documentation MCP Server の利用

AWS Documentation MCP Server は公開されているものになるため、誰でも利用可能です。ですが、利用するためには以下のコマンドから `uv` と `Python3.10` 以上をインストールしておく必要があります。詳細は公式ドキュメントを参照ください。

https://awslabs.github.io/mcp/servers/aws-documentation-mcp-server/

```bash: Terminal
curl -LsSf https://astral.sh/uv/install.sh | sh
uv python install 3.10
```

これだけで AWS Documentation MCP Server を利用する準備が整いました。簡単ですね！！

### Playwright MCP サーバーの起動

続いて、Playwright MCP サーバーの起動します。今回はローカルで起動した Playwright MCP サーバーと SSE で通信することとします。
まず、以下のコマンドを実行して Playwright MCP サーバーを起動します。コマンドの詳細は公式ページを参照ください。

https://github.com/microsoft/playwright-mcp

```bash: Terminal
npx @playwright/mcp@latest --port 8931 --headless
```

起動すると `http://localhost:8931/sse` が SSE のエンドポイントとなります。

## ルートハンドラーの開発

MCP サーバーが起動できたため、それに接続するルートハンドラーを作成します。
今回は Vercel AI SDK の `useChat` を利用するため、デフォルトのエンドポイントである `api/chat` に相当する `app/api/chat/route.ts` を作成します。

ここでポイントとなるのが以下の点です。

- MCP クライアントはサーバーとの接続方法（トランスポート）として、`SSE (Server-Sent Events)`、`stdio`、`カスタムトランスポート`を提供しているため、**使用する MCP サーバーによって使い分ける**
- **複数の MCP Tools を登録する**ように `tools` オブジェクトを構成する
- ストリーミング通信は **Vercel AI SDK の `streamText`** を利用する
- ストリーミング応答が完了したら、**MCP クライアントの接続を閉じる**

```ts: app/api/chat/route.ts
import { streamText } from 'ai';
import { experimental_createMCPClient as createMCPClient } from "ai";
import { Experimental_StdioMCPTransport as StdioMCPTransport } from 'ai/mcp-stdio';
import { openai } from '@ai-sdk/openai';

export async function POST(req: Request) {
  try {
    // AWS MCP クライアントの作成
    const awsMcpClient = await createMCPClient({
      transport: new StdioMCPTransport({
        command: "uvx",
        args: ["awslabs.aws-documentation-mcp-server@latest"],
        env: {
          FASTMCP_LOG_LEVEL: "ERROR"
        },
      }),
    })

    // Azure Functions MCP クライアントの作成
    const azMcpClient = await createMCPClient({
       transport: {
         type: "sse",
         url: "https://<your-function-app-name>.azurewebsites.net/runtime/webhooks/mcp/sse",
         headers: {
           "x-functions-key": process.env.AZURE_FUNCTIONS_KEY || "",
         },
       },
     });

    // Playwright MCP クライアントの作成
    const playwrightMcpClient = await createMCPClient({
      transport: {
        type: "sse",
        url: "http://localhost:8931/sse"
      },
    });

    const { messages } = await req.json();

    // MCP サーバーからツール定義を取得
    const awsMcpTool = await awsMcpClient.tools();
    const azMcpTool = await azMcpClient.tools();
    const playwrightMcpTool = await playwrightMcpClient.tools();

    const tools = {
      ...awsMcpTool,
      ...azMcpTool,
      ...playwrightMcpTool,
    }

    // Vercel AI SDK の streamText 関数を使用して LLM とのストリーミング通信を開始
    const result = streamText({
      model: openai('gpt-4o'),
      messages,
      tools,
      onFinish: async () => {
        // ストリーミング応答が完了したら、MCP クライアントの接続を閉じる
        await awsMcpClient.close();
        await azMcpClient.close();
        await playwrightMcpClient.close();
      },
    });

    return result.toDataStreamResponse();

  } catch (error) {
    console.error('Error: ', error);
  }
}

```

## チャット画面の開発

続いて、上記のルートハンドラーを呼び出すためのチャット画面を開発します。今回は他に画面はないため、`app/page.tsx`を編集します。
ポイントとなる点は以下の点です。

- インタラクティブな画面のため、`use client` でクライアントサイドで実行する
- `@ai-sdk/react` の `useChat` フックの `handleSubmit` を実行することでルートハンドラーを呼び出す

```tsx: app/page.tsx
'use client';

import { useChat } from '@ai-sdk/react';

export default function Chat() {
  const { messages, input, handleInputChange, handleSubmit, status } =
    useChat();
  return (
    <div className='flex flex-col items-center w-full min-h-screen bg-gray-100 dark:bg-gray-900'>
      <div className='flex flex-col w-full max-w-5xl p-6 mt-12 bg-white rounded-lg shadow-lg dark:bg-gray-800'>
        <h1 className='text-2xl font-bold text-center text-gray-800 dark:text-gray-100'>
          Chat with Open AI
        </h1>
        <div className='flex flex-col gap-4 mt-4 overflow-y-auto max-h-[40rem]'>
          {messages
            .filter((message) =>
              message.parts.some((part) => part.type === 'text' && part.text.trim() !== '')
            )
            .map((message) => (
              <div
                key={message.id}
                className={`p-3 rounded-lg ${
                  message.role === 'user'
                    ? 'bg-blue-100 text-blue-900 self-end'
                    : 'bg-gray-200 text-gray-900 self-start'
                }`}
              >
                <span className='font-semibold'>
                  {message.role === 'user' ? 'User: ' : 'AI: '}
                </span>
                {message.parts.map((part, i) => {
                  switch (part.type) {
                    case 'text':
                      return (
                        <span key={`${message.id}-${i}`} className='block'>
                          {part.text}
                        </span>
                      );
                  }
                })}
              </div>
            ))}
        </div>
        <form
          onSubmit={handleSubmit}
          className='flex flex-col gap-2 mt-4'
        >
          <input
            type='text'
            value={input}
            onChange={handleInputChange}
            className='w-full p-3 border border-gray-300 rounded-lg dark:bg-gray-700 dark:text-gray-100 dark:border-gray-600'
            placeholder='メッセージを入力してください'
            disabled={status !== 'ready'}
          />
          <button
            type='submit'
            className='px-4 py-2 text-white bg-blue-500 rounded-lg hover:bg-blue-600 disabled:bg-gray-400 disabled:cursor-not-allowed'
            disabled={status !== 'ready'}
          >
            {status !== 'ready' ? '・・・' : '送信'}
          </button>
        </form>
      </div>
    </div>
  );
}
```

これで実装の全てが完了しました！お疲れ様でした。

## Next.js サーバーの起動

実装が完了したため、以下のコマンドで起動します。起動すると、`http://localhost:3000` で画面を表示することができます。

```bash: Terminal
npm run dev
```

すると、以下のような画面が表示されます。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/6e5edd5b-84e2-45d7-ba83-d8ebaa80bbff.png)

試しに
**「https://news.yahoo.co.jp/ にアクセスして、今日のニュースでインパクトのあるものを 3 つ挙げてください」**
と入力すると、Playwright MCP が呼び出されその Web 情報を出力してくれます。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/43edf803-bc42-4cd0-b1fa-9d00d8e74b95.png)

本当に MCP サーバーが呼ばれているのか分かりにくいため、AWS Documentation MCP でも試してみます。
**「AWS Lambda とはどんなサービスですか？」**
と質問し、開発者ツールを確認したところ、AWS Documentation MCP の `search_documentation` が呼ばれていることが確認できました！！

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/7170c32e-bcb4-41f8-8371-b5ae98a4a6dd.png)

:::note warn

冒頭にも記載しましたが、**Azure Functions の MCP サーバーに接続しようとすると、MCP クライアント初期化時に止まってしまい、実行できていません。**
ログからは接続できているように見えるため、設定不備はなさそうですが、この点は今後の課題とし、改善できたらこの記事を更新する予定です。

:::

# おわりに

いかがでしたでしょうか。今回は Azure Functions で MCP サーバーを作成した話と、Vercel AI SDK で MCP クライアントを構築し、MCP サーバーを呼び出す方法を紹介しました。

AI エージェントが台頭し、技術の進化が留まるところを知りません。
どうしても触ってみないとキャッチアップが難しいものもありますが、この波に乗り遅れないよう頑張りましょう！

またアップデートがあればお伝えしようと思います。最後まで読んでいただきありがとうございました。
