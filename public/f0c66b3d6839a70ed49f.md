---
title: Remix SPAとAzure Web PubSubでWebSocketチャットアプリを作る（アプリ編）
tags:
  - Azure
  - websocket
  - React
  - Remix
  - 個人開発
private: false
updated_at: '2024-12-23T09:17:28+09:00'
id: f0c66b3d6839a70ed49f
organization_url_name: nri
slide: false
ignorePublish: false
---
# はじめに

今年もあと 1 週間ほどで終わってしまいますね、、時の流れは早いものです。
突然ですが、みなさま以下の記事はご覧になっていただけましたでしょうか。

https://qiita.com/s_w_high/items/26711bd340c0ce34edf2

今回の記事は上記の記事の続編となります。
前回の記事でも紹介したように、今年のアドベントカレンダーでは WebSocket を使ったチャットアプリケーションを Azure で構築した奮闘記を記事にしております。主に以下のような流れで開発していきます。

1. WebSocket を使ったリアルタイムチャットアプリを動かすための基盤を構築する
2. Remix SPA モードでアプリケーションを開発し、1 で構築した基盤にデプロイ～動作確認を行う

前回の記事で 1 に当たる基盤を設計し IaC（Bicep）で構築する部分まで紹介しました。
今回の記事は 2に当たる 1 で構築した基盤に開発したアプリケーションをデプロイし動作確認まで行った話になります。

アーキテクチャでいうと、主に以下の部分に当たる部分を紹介します。簡単に要点に当たる部分も記載しています。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/25cc9c5b-f8a1-2ba6-fad5-1950fbb809a4.png)

また、今回のソースコードは以下の GitHub に格納しておりますので、是非ご覧ください。
今回はアプリ編ということで、`frontend` 及び `backend` ディレクトリの話になります。

https://github.com/m2-sakai/chatapp-azure

# 完成したアプリ

細かい技術スタックや実装の話に入る前にどんなアプリケーションなのかご紹介します。
今回は以下のようなシンプルはチャットアプリケーションを開発しました。

![demo.gif.gif](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/46f6bf6d-060c-ffd0-ed2f-d13f8993a50e.gif)

# 技術スタックと開発環境

今回採用した技術スタックについてご紹介します。基盤（IaC）については[前回の記事](https://qiita.com/s_w_high/items/26711bd340c0ce34edf2)をご参照ください。

## フロントエンド

フロントエンドの実装には、以下のような技術を使用しています。

| カテゴリ           | 技術スタック         | バージョン/プラン |
| ------------------ | -------------------- | ----------------- |
| 言語               | TypeScript / React   | 5.1.6 / 18.2.0    |
| ランタイム         | Node.js              | 22.11.0           |
| ビルドツール       | Vite                 | 5.1.0             |
| フレームワーク     | Remix SPA モード     | 2.14.0            |
| CSS                | Tailwind CSS         | 3.4.15            |
| UI コンポーネント  | shadcn/ui            | 0.9.3             |
| ホスティングサイト | Azure Static WebApps | Free プラン       |

## バックエンド

バックエンドの実装には、以下のような技術を使用しています。
使用するサービスの説明は割愛させていただきますので、Azure の公式ドキュメントを参照ください。

| カテゴリ           | 技術スタック                       | バージョン/プラン |
| ------------------ | ---------------------------------- | ----------------- |
| API の公開先       | Azure Functions 分離ワーカーモデル | v4                |
| 言語               | .NET                               | 8.0.400           |
| WebSocket サービス | Azure Web PubSub                   | Standard_S1       |
| Database           | Azure Cosmos DB                    | -                 |

:::note info

Azure Functions には大きく分けて 2 種類のプロセスモデルが存在します。
Azure Functions の初期はインプロセスモデルのみサポートされていましたが、.NET6 から分離ワーカーモデルが登場し、現在はこちらが主流です。詳しくは[こちら](https://learn.microsoft.com/ja-jp/azure/azure-functions/dotnet-isolated-in-process-differences)をご参照ください。

| 実行モデル         | 説明                                                                            |
| ------------------ | ------------------------------------------------------------------------------- |
| 分離ワーカーモデル | 関数コードがホストプロセスとは **別の** .NET ワーカープロセスで実行されるモデル |
| インプロセスモデル | 関数コードがホストプロセスと **同じ** プロセスで実行されるモデル                |

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/2a5276ac-f377-d40d-f6b1-a63b763e5201.png)

ちなみに、**インプロセスモデルは 2026 年 11 月 10 日にサポート終了となります**。詳細は[こちら](https://azure.github.io/jpazpaas/2024/04/01/azure-functions-inprocess-end-of-support-FVN7-7PZ.html)をご覧ください。

:::

# フロントエンドの実装・デプロイ

## 画面一覧

今回フロントエンドで実装すべき画面や機能は以下の通りです。

| 画面           | 説明                                                             |
| -------------- | ---------------------------------------------------------------- |
| ユーザ入力画面 | 名前とメールアドレスを入力し、チャット画面に遷移する             |
| チャット画面   | Web PubSub と WebSocket 接続をオープンし、チャットをやり取りする |

## Remix SPA モード + shadcn/ui の環境構築

まず、Remix を SPA モードで動作させるためのアプリケーションを構築します。
細かい説明は割愛いたしますが、Remix は基本 SSR（サーバーサイドレンダリング）でアプリケーションを実行しますが、[2.5.0](<https://remix.run/docs/en/main/guides/spa-mode#:~:text=SPA%20Mode%20in-,2.5.0,-(RFC)%2C%20which>) から SPA モードが導入され、クライアントサイドのみで実行できるようになりました。

https://remix.run/docs/en/main/guides/spa-mode

まずは公式ドキュメントに沿って、プロジェクトを作成します。

```bash: Terminal
npx create-remix@latest --template remix-run/remix/templates/spa
```

もしくは、既存の Remix アプリケーションに以下のように `ssr: false` とすることで SPA モードとすることができます。

```ts: vite.config.ts
import { vitePlugin as remix } from "@remix-run/dev";
import { defineConfig } from "vite";

export default defineConfig({
  plugins: [
    remix({
      ssr: false,
    }),
  ],
});
```

続いて、[shadcn/ui](https://ui.shadcn.com/) の環境構築を行います。
shadcn/ui とは [Radix UI](https://www.radix-ui.com/) と [Tailwind CSS](https://tailwindcss.com/) を使って書かれた UI コンポーネントをまとめたもので、Tailwind CSS を通じてスタイルをカスタマイズできます。
[2023 JavaScript Rising Stars](https://risingstars.js.org/2023/en#section-all) では、全 Project で見事ランキング 1 位となっています。筆者もさすがに触らんわけにいかんだろ... の精神で導入してみました。

shadcn/ui は柔軟性とカスタマイズ性が非常に高く、必要なコンポーネントだけを選択しカスタマイズできます。

https://ui.shadcn.com/

プロジェクトへの導入については[公式ドキュメント](https://ui.shadcn.com/docs/installation/remix)が用意されていますので、それに沿って実施すれば簡単に導入できます。

```bash: Terminal
npx shadcn@latest init
```

以上で、実装前の環境構築は完了です！

## ユーザ入力画面の実装

ここから細かい実装に入るのですが、すべてのソースコードは載せられないので全量見たい方は [GitHub](https://github.com/m2-sakai/chatapp-azure) をご参照ください。

まずはユーザ入力画面です。ユーザ入力画面で実現したいことは以下の通りです。

- ユーザ名とメールアドレスを入力させる
- バックエンドにリクエストを送信し、問題なければチャットルームに遷移させる

今回は大層な認証機能を不要とし、だれでも同じチャットルームに入れるようにします。
Remix では、`app/routes` ディレクトリに画面ページを配置します。このユーザー入力画面は初期画面としたため、`_index.tsx` に記載します。

:::note info

Remix におけるルートファイルの命名については以下を参照ください。
https://remix.run/docs/en/main/file-conventions/routes

:::

以下に実装を示します。Remix の SPA モードでは [`clientAction`](https://remix.run/docs/en/main/route/client-action) を使用することで入力フォームからの Submit をすることができます。
また、フォームやボタンについては shadcn/ui の UI コンポーネントを用いています。

```tsx: app/routes/_index.tsx
// フォームが呼ばれたら API を実行し、ユーザの登録が完了したらチャット画面に遷移
export const clientAction = async ({ request }: ClientActionFunctionArgs) => {
  const body = await request.formData();
  const email = body.get('email') as string | null;
  const name = body.get('name') as string | null;
  localStorage.setItem('email', email ?? '');

  await insertUser(name, email);
  return redirect(`/chat`);
};

export default function Index() {
  const form = useForm<UserInfo>();

  return (
    <div className='flex flex-col items-center justify-center min-h-screen bg-gray-100'>
      <h1 className='text-xl font-bold mb-6'>以下にユーザ名とメールアドレスを入力してください</h1>
      <FormCn {...form}>
        <Form method='post' className='grid gap-4'>
          <FormField
            name='name'
            render={({ field }) => (
              <FormItem>
                <FormLabel>Name</FormLabel>
                <FormControl>
                  <Input type='text' placeholder='山田 太郎' autoComplete='name' {...field} />
                </FormControl>
                <FormMessage />
              </FormItem>
            )}
          />
          <FormField
            name='email'
            render={({ field }) => (
              <FormItem>
                <FormLabel>Email</FormLabel>
                <FormControl>
                  <Input type='text' placeholder='sample@gmail.com' autoComplete='email' {...field} />
                </FormControl>
                <FormMessage />
              </FormItem>
            )}
          />
          <Button type='submit'>送信</Button>
        </Form>
      </FormCn>
    </div>
  );
}
```

今回は上記のような実装を通して以下のようなユーザ入力画面を作成しました。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/56ab80d6-57b5-3dc9-bc03-5b8288c2893d.png)

## チャット画面の実装

続いて、チャット画面の実装です。チャット画面で実現したいことは以下の通りです。

- 自身で入力したチャットは右側、他人が入力したチャットは左側に配置する
- メッセージには、「誰が」「いつ」投稿したか分かる
- チャットは Enter で改行、Ctrl + Enter で送信できる
- すでにチャットルームに送られているチャットも表示する
- 常に画面の一番下部にスクロールし、チャットが送信されたら自動的にスクロールする
- 画面が表示されたら Azure Web PubSub と WebSocket 接続をオープンし、自身/他人がチャットを送信したら受信できる

以下に実装を示します。Remix の SPA モードでは [`clientLoader`](https://remix.run/docs/en/main/route/client-loader) を使用することでレンダリング前にデータを取得することができます。

```tsx: app/routes/chat.tsx
// clientLoaderで必要なデータを取得
export const clientLoader = async () => {
  try {
    const email = localStorage.getItem('email');
    const userData = await fetchUserData(email);
    const allChats = await fetchChatList();
    const accessToken = await fetchToken();
    return {
      userData,
      allChats,
      accessToken,
    };
  } catch (error) {
    console.error(`There was an error fetching the user data: ${error}`);
    return redirect(`/`);
  }
};

export default function Chat() {
  const { userData, allChats, accessToken } = useLoaderData<typeof clientLoader>();
  const [messages, setMessages] = useState<ChatMessage[]>(allChats);
  const [input, setInput] = useState('');
  const socketRef = useRef<ReconnectingWebSocket>();
  const messagesEndRef = useRef<HTMLDivElement>(null);

  // WebSocket 接続
  useEffect(() => {
    const negotiateAndConnect = async () => {
      if (socketRef.current) {
        return;
      }
      try {
        const websocket = new ReconnectingWebSocket(`${API_ROUTES.WSS_CONNECT}?access_token=${accessToken}`);
        socketRef.current = websocket;

        websocket.onopen = () => {
          console.log('WebSocket接続がオープンしました。');
        };

        websocket.onmessage = (event) => {
          const messageData = JSON.parse(event.data);
          setMessages((prevMessages) => [...prevMessages, messageData]);
        };
      } catch (error) {
        console.error('WebSocket接続中にエラーが発生しました:', error);
      }
    };

    negotiateAndConnect();

    return () => {
      if (socketRef.current) {
        socketRef.current.close();
      }
    };
  }, []);


  // 自動スクロール
  useEffect(() => {
    if (messagesEndRef.current) {
      messagesEndRef.current.scrollIntoView({ behavior: 'smooth' });
    }
  }, [messages]);

  // チャットをWeb PubSubに送信
  const sendMessage = () => {
    if (socketRef.current && socketRef.current.readyState === WebSocket.OPEN) {
      const message: ChatMessage = {
        id: uuidv4(),
        content: input,
        senderName: userData.name,
        senderEmail: userData.email,
        timestamp: new Date().toISOString(),
      };

      socketRef.current.send(JSON.stringify(message));
      setInput('');
    }
  };

  // Enterで改行、Ctrl + Enterで送信
  const handleKeyDown = (e: any) => {
    if (e.key === 'Enter' && e.ctrlKey) {
      e.preventDefault();
      sendMessage();
    }
  };

  return (
    <>
      <Header />
      <div className='flex flex-col justify-between h-screen'>
        <div className='flex-grow overflow-y-auto p-4 mt-[50px] mb-[100px]'>
          {messages.map((message) => (
            <div key={message.id} className={`mb-4 ${message.senderEmail !== userData.email ? 'text-left' : 'text-right'}`}>
              <div className='font-semibold'>{message.senderName}</div>
              <div className={`flex ${message.senderEmail !== userData.email ? 'justify-start' : 'justify-end'} mb-2`}>
                <div className={`rounded-lg px-4 py-2 max-w-64 shadow break-words whitespace-pre-wrap ${message.senderEmail !== userData.email ? 'bg-gray-300' : 'bg-blue-500 text-white'}`}>
                  {message.content}
                  <div className={`text-xs ${message.senderEmail !== userData.email ? 'bg-gray-300' : 'bg-blue-500 text-white'}`}>{new Date(message.timestamp).toLocaleString()}</div>
                </div>
              </div>
            </div>
          ))}
          <div ref={messagesEndRef} />
        </div>

        <div className='flex p-4 bg-gray-100 fixed bottom-0 left-0 right-0'>
          <textarea className='w-full p-2 border rounded-lg' value={input} onChange={(e) => setInput(e.target.value)} onKeyDown={handleKeyDown} placeholder='メッセージを入力...' rows={2} />
          <Button className='mt-8' onClick={sendMessage}>
            送信
          </Button>
        </div>
      </div>
    </>
  );
}
```

上記のような実装を通して以下のようなチャット画面を作成しました。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/b45556b9-b37c-08c0-490e-f0e158c4d801.png)

## デプロイ

フロントエンドアプリは、Azure Static WebApps を用いてホスティングします。Azure Static WebApps は静的コンテンツのホスティングサービスです。詳細は以下のドキュメントを参照ください。

https://learn.microsoft.com/ja-jp/azure/static-web-apps/overview

Azure Static WebApps に SPA アプリをデプロイするのは非常に簡単です。まず、Azure から「静的 Web アプリ」と検索し、「作成」を押下します。
その際に、以下のように GitHub とデプロイ連携を行います（今回は `frontend` ディレクトリに作成したので、以下のような設定となります）。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/0e0fc5d4-803f-03bf-04db-adfec20a9d41.png)

作成が完了すると、GitHub Actions が実行され、対象ブランチ（今回は `main`）に push されると自動的にデプロイされます。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/6238be24-9a13-ef47-8c21-2634b1ba771b.png)

デプロイが完了し Static WebApps の URL にアクセスすると、以下のような画面が表示されます（動作にはバックエンドの実装も必要です）。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/a06c01e0-3294-3adf-9e78-439714879c9c.png)

これにて、フロントエンドアプリケーションのデプロイは終了です！お疲れ様でした！

# バックエンドの実装・デプロイ

## API 機能一覧

今回バックエンドで実装すべき機能は以下の通りです。

| 機能                               | トリガー   | HTTP メソッド | 詳細                                                                                                                 |
| ---------------------------------- | ---------- | ------------- | -------------------------------------------------------------------------------------------------------------------- |
| ユーザ取得 API                     | HTTP       | GET           | Cosmos DB からユーザ情報を取得する                                                                                   |
| ユーザ登録 API                     | HTTP       | POST          | Cosmos DB にユーザ情報を登録する                                                                                     |
| チャット取得 API                   | HTTP       | GET           | Cosmos DB からチャット一覧を取得する                                                                                 |
| Web PubSub トークン取得 API        | HTTP       | GET           | Web PubSub に接続するためのトークン（WebSocket URL）を取得する                                                       |
| ブロードキャスト、チャット登録 API | Web PubSub | -             | Web PubSub からイベントを受信し、接続しているクライアントにブロードキャスト、及びチャット情報を Cosmos DB に登録する |

:::note info

Azure Functions のトリガーや Web PubSub のイベントトリガーについては以下を参照ください。

- [Azure Functions の HTTP トリガー](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-bindings-http-webhook-trigger?tabs=python-v2%2Cisolated-process%2Cnodejs-v4%2Cfunctionsv2&pivots=programming-language-csharp)
- [Azure Functions の Azure Web PubSub トリガー バインド](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-bindings-web-pubsub-trigger?tabs=isolated-process%2Cnodejs-v4&pivots=programming-language-csharp)
- [Azure Web PubSub サービスでイベント ハンドラーを構成する](https://learn.microsoft.com/ja-jp/azure/azure-web-pubsub/howto-develop-eventhandler)

:::

## Azure Functions の環境構築

Azure Functions を実装するには、Visual Studio を使用するのが簡単です。
Visual Studio Code でも開発は可能ですが、個人的には Visual Studio の方が開発体験は良いです。ちなみに筆者は Visual Studio 2022 Professional を使用していますが、無償版でも問題なく実装できるはずです。

以下のドキュメントに従って分離ワーカーモデル用のプロジェクトを作成します。

https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-develop-vs?pivots=isolated

## エントリーポイントの実装

Azure Functions の分離ワーカーモデルではアプリケーションのエントリーポイントとなる `Program.cs` が必要になります。
このファイルはアプリケーションが実行される際に呼び出されます。以下に今回の実装を示します。

```csharp: Program.cs
var host = new HostBuilder()
    .ConfigureFunctionsWebApplication()
    .ConfigureServices(services =>
    {
        // Application Insightsへのログ出力設定
        services.AddApplicationInsightsTelemetryWorkerService();
        services.ConfigureFunctionsApplicationInsights();

        // Cosmos DB, WebPubSub用接続クライアントの作成
        services.AddSingleton((s) =>　
        {
            var isConnectMsi = Boolean.Parse(Environment.GetEnvironmentVariable("COSMOS_CONNECT_MSI"));
            var retryPolicy = new CosmosClientOptions()
            {
                MaxRetryAttemptsOnRateLimitedRequests = 1
            };
            if (isConnectMsi)
            {
                var endpoint = Environment.GetEnvironmentVariable("COSMOS_ENDPOINT");
                return new CosmosClient(endpoint, new DefaultAzureCredential(), retryPolicy);

            } else
            {
                var connectionString = Environment.GetEnvironmentVariable("COSMOS_CONNECTION_STRING");
                return new CosmosClient(connectionString, retryPolicy);
            }
        });

        // WebPubSub用接続クライアントの作成
        services.AddSingleton((s) =>
        {
            var isConnectMsi = Boolean.Parse(Environment.GetEnvironmentVariable("WEBPUBSUB_CONNECT_MSI"));
            var endpoint = Environment.GetEnvironmentVariable("WEBPUBSUB_ENDPOINT");
            var hubName = Environment.GetEnvironmentVariable("WEBPUBSUB_HUB");
            if (isConnectMsi)
            {
                return new WebPubSubServiceClient(new Uri(endpoint), hubName, new DefaultAzureCredential());
            }
            else
            {
                var accessKey = Environment.GetEnvironmentVariable("WEBPUBSUB_ACCESSKEY");
                return new WebPubSubServiceClient(new Uri(endpoint), hubName, new Azure.AzureKeyCredential(accessKey));
            }
        });
    })
    // ロギングの設定
    .ConfigureLogging(logging =>
    {
        logging.SetMinimumLevel(LogLevel.Information);
        logging.Services.Configure<LoggerFilterOptions>(options =>
        {
            var defaultRule = options.Rules.FirstOrDefault(rule => rule.ProviderName == "Microsoft.Extensions.Logging.ApplicationInsights.ApplicationInsightsLoggerProvider");
            if (defaultRule is not null)
            {
                options.Rules.Remove(defaultRule);
            }
        });
    })

    .Build();

host.Run();
```

実装のポイントは以下となります。

- ログ監視を Application Insights で行うための設定を入れる
- Cosmos DB や Web PubSub の接続クライアントはアプリケーションで 1 つでよいため、シングルトンとしてインスタンスを生成し、使いまわすようにする（Dependency Injection）
  - 今回は Azure 環境 + マネージド ID 認証で接続するが、デバッグ用にローカル環境でも動作させるように `isConnectMsi` 変数で制御して接続クライアントの生成メソッドを切り替える
- ログレベルを調節できるようにロギングの設定を入れる

:::note info

Azure のマネージド ID については以下の記事を参照ください。

- [Azure リソースのマネージド ID とは](https://learn.microsoft.com/ja-jp/entra/identity/managed-identities-azure-resources/overview)
- [App Service と Azure Functions でマネージド ID を使用する方法](https://learn.microsoft.com/ja-jp/azure/app-service/overview-managed-identity?tabs=portal%2Chttp)

:::

## ユーザ系 API ・チャット系 API の実装

続いて、ユーザ取得/登録 API の実装です。ユーザ系 API もチャット系 API も単なる Cosmos DB との CRUD 操作になるため、同じ章で記載します。

.NET で Cosmos DB に接続できるようにするには `Microsoft.Azure.Cosmos` ライブラリを使用します。DB の設計については割愛しますのでご了承ください。
以下に `UserFunctions.cs` のデータ取得部分の実装を示します。他の登録系処理を参照したい場合は [GitHub](https://github.com/m2-sakai/chatapp-azure) を参照ください。

```csharp: UserFunctions.cs
[Function("GetUser")]
public async Task<HttpResponseData> GetUser([HttpTrigger(AuthorizationLevel.Anonymous, "get", Route = "user")] HttpRequestData req)
{
    string email = req.Query["email"];

    try
    {
        string query = $"SELECT * FROM c WHERE c.email = @email";
        QueryDefinition queryDefinition = new QueryDefinition(query).WithParameter("@email", email);
        // Program.csで作成したインスタンスを使用して、クエリ実行
        using FeedIterator<UserInfo> queryIterator = _container.GetItemQueryIterator<UserInfo>(queryDefinition);

        UserInfo user = null;
        if (queryIterator.HasMoreResults)
        {
            var response = await queryIterator.ReadNextAsync();
            user = response.FirstOrDefault();
        }

        if (user == null)
        {
            var response = req.CreateResponse(HttpStatusCode.NotFound);
            await response.WriteStringAsync("User not found");
            return response;
        }

        var httpResponse = req.CreateResponse(HttpStatusCode.OK);
        await httpResponse.WriteAsJsonAsync(user);
        return httpResponse;
    }
    // catch 処理は省略
}
```

:::note info

ローカル環境で Cosmos DB を作成するには、[Azure Cosmos DB エミュレーター](https://learn.microsoft.com/ja-jp/azure/cosmos-db/emulator)が簡単に使えて便利です。

:::

## トークン取得 API の実装

続いて、トークン取得 API を実装します。
クライアントから WebSocket 接続をオープンするためには、クライアントから WebSocket サービス（今回は Azure Web PubSub）に WebSocket 用 URL（`wss://`）でリクエストを送信する必要があります。

Azure Web PubSub を使用する場合、一時的に利用できる接続用 URL を取得することができますが、今回はクライアント → Azure Web PubSub ではなく、クライアント **→ API Management →** Azure Web PubSub となるため、アクセストークンのみクライアントに返却します。

以下に実装を示します。

```csharp: TokenFunctions.cs
[Function("GetToken")]
public async Task<HttpResponseData> GetToken([HttpTrigger(AuthorizationLevel.Anonymous, "get", Route = "token")] HttpRequestData req)
{
    // Program.csで作成したインスタンスを使用する
    // Web PubSubからサービス接続用URLを取得する
    var url = await _webPubSubServiceClient.GetClientAccessUriAsync();
    var query = HttpUtility.ParseQueryString(url.Query);
    string accessToken = query["access_token"];

    var response = req.CreateResponse(HttpStatusCode.OK);
    var res = new TokenResponse()
    {
        AccessToken = accessToken
    };
    await response.WriteAsJsonAsync(res);
    return response;
}
```

## イベントハンドラー（Web PubSub トリガー）の実装

最後に、イベントハンドラーとなる Web PubSub トリガー API の実装です。ここがバックエンド実装における **肝** になります。
というのも、WebSocket でチャットアプリを作るからには他ユーザがチャットを送信した場合は自動で受信できないと意味がありません。ですので、きちんと接続しているクライアントにはブロードキャストする必要があります。

そこで、Azure Web PubSub には **イベントハンドラー** という機能が存在します。
イベントハンドラーとは、Azure Web PubSub が何かしらのイベントを受け取った際（例：接続開始、メッセージ受信）、そのイベントを他の URL に WebHook で送信できる機能です。
イベントハンドラーの詳細と設定方法は以下を参照ください。

https://learn.microsoft.com/ja-jp/azure/azure-web-pubsub/howto-develop-eventhandler

また、Azure Functions でイベントハンドラーを構成する場合は Web PubSub トリガーを使用することでイベントを受信できます。
今回はユーザからチャットが送信された際にそのイベントを Functions で受信し、Cosmos DB に登録 + 接続クライアントにブロードキャストを実行しています。
以下に実装を示します。

```csharp: EventHandlerFunctions.cs
[Function("PublishSaveMessage")]
public async Task<UserEventResponse> PublishSaveMessage([WebPubSubTrigger("chatroom", WebPubSubEventType.User, "message")] UserEventRequest request)
{
    var message = JsonConvert.DeserializeObject<ChatMessage>(request.Data.ToString());

    try
    {
        // DBに既に同じIDがないか確認
        var query = new QueryDefinition("SELECT * FROM c WHERE c.id = @id")
                    .WithParameter("@id", message.Id);
        var iterator = _container.GetItemQueryIterator<ChatMessage>(query);
        var existingMessage = (await iterator.ReadNextAsync()).FirstOrDefault();

        if (existingMessage == null)
        {
            // Cosmos DBにチャットメッセージを保存
            await _container.CreateItemAsync(message, new PartitionKey(message.SenderEmail));

            // 接続クライアントにブロードキャスト
            await _webPubSubServiceClient.SendToAllAsync(RequestContent.Create(message), ContentType.ApplicationJson);
        }
        else
        {
            _logger.LogInformation("Message with the same ID already exists, skip.");
        }
    }
    // catch 処理は省略

    return new UserEventResponse
    {
    };
}
```

## デプロイ

最後に、作成した Functions を Azure にデプロイします。
今回は Azure Functions の Zip デプロイを利用します（本来はここも CI/CD にすべきですが、今後の改善点として現在は手動デプロイとしています）。

https://learn.microsoft.com/ja-jp/azure/azure-functions/deployment-zip-push

まずは以下のコマンドでアプリケーションを公開します。

```bash: Terminal
dotnet publish -c Release -p:UseAppHost=false
```

プロジェクトの `bin/Release/net8.0/publish` ディレクトリにデプロイファイル一式が作成されているため全て Zip 化し、以下のコマンドでデプロイします。

```bash: Terminal
az functionapp deployment source config-zip -g <resource_group> -n <app_name> --src <zip_file_path>
```

起動が成功すると、以下のように Azure Functions の画面に実装した関数が表示されます。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/8431c5b3-30b3-9253-16c8-6ebd1417919d.png)

こちらでバックエンドアプリケーションもデプロイ成功です！お疲れ様でした！！

# おわりに

いかがでしたでしょうか。
前回の記事で作成した基盤にデプロイするアプリケーションの実装を紹介しました。
昨今は AWS や Azure をはじめ、クラウド環境でアプリケーションを実行することがほとんどなため、個人開発でもできるだけクラウド環境で動作させるところまで実践した方がより身のためになると思います。

また、来年もアドベントカレンダーの季節には何か今まで扱ったことのない技術を取り入れて個人開発していきたいなと思いますので、来年も楽しみにしていてください。

最後に、重ねてになりますが他にも弊社ではアドベントカレンダーで様々な記事が投稿されておりますので、皆さんぜひご覧ください！

- [NRI OpenStandia Advent Calendar 2024 (シリーズ 1〜3)](https://qiita.com/advent-calendar/2024/nri-openstandia)
- [NRI OpenStandia (IAM 編) Advent Calendar 2024](https://qiita.com/advent-calendar/2024/nri-openstandia-iam)
- [一歩ずつ Rust に慣れていく TypeScript エンジニアの記録 Advent Calendar 2024](https://qiita.com/advent-calendar/2024/rust-from-ts)
- [midPoint by OpenStandia Advent Calendar 2024](https://qiita.com/advent-calendar/2024/midpoint-by-openstandia)
