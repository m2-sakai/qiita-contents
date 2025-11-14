---
title: >-
  Next.js アプリを OpenAI × Vercel から Azure OpenAI × App Service × Azure DevOps
  へ移行する
tags:
  - Azure
  - 個人開発
  - Next.js
  - AzureDevOps
  - AzureOpenAIService
private: false
updated_at: '2023-12-19T07:00:16+09:00'
id: 7bf78bae49705690ab12
organization_url_name: nri
slide: false
ignorePublish: false
---
# はじめに

ご無沙汰しております。今年のアドベントカレンダーも後半線真っ只中ではございますが、以下の [NRI OpenStandia Advent Calendar 2023](https://qiita.com/advent-calendar/2023/nri-openstandia) の 3 日目の記事は見ていただけましたでしょうか。

https://qiita.com/s_w_high/items/5590f00b273444012386

本記事はこの記事の続編となります。特段この記事の内容を知らなくても問題はございませんが、より本記事を楽しんでいただくためにも是非見ていただけたらと思います。上記の記事で

> 1. Next.js でアプリを開発し、Vercel にデプロイする
> 2. Azure で動作するようアプリを改修し、App Service に移行する
>
> 本記事はその中で前半の Vercel にデプロイするまでとなります。Azure に移行する記事は NRI OpenStandia Advent Calendar 2023 の 19 日目に公開予定です。

と、宣言した通り、 [NRI OpenStandia Advent Calendar 2023](https://qiita.com/advent-calendar/2023/nri-openstandia) の 19 日目は前回作成した Next.js のアプリを [Azure App Service](https://azure.microsoft.com/ja-jp/products/app-service) に移行する際の奮闘記を記載します。

# 最終的な構成

今回は単なるアプリケーションのみの移行だけではなく CI/CD も移行します。移行前は [Vercel](https://vercel.com/) にアプリをデプロイしており、Vercel 側に自動で CI/CD が組み込まれていました。ただ今回は Azure 上にデプロイするということで CI/CD 周りは [Azure DevOps](https://azure.microsoft.com/ja-jp/products/devops)（[Azure Pipelines](https://azure.microsoft.com/ja-jp/products/devops/pipelines)） を 使用していきます。
他にも前回作成したアプリケーションを Azure に載せていくため、各サービスそれぞれ移行していきます。最終的に完成した全体像（構成）は以下となります。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/db0ae9b5-8dad-ff5b-fa4d-26f8e13d3b6e.png)

| カテゴリ                               | 移行前（Vercel）                                                   | 移行後（Azure）                                                                                                                                       |
| -------------------------------------- | ------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| ホスティング                           | [Vercel](https://vercel.com/)                                      | [Azure App Service](https://azure.microsoft.com/ja-jp/products/app-service)（[Web Apps](https://azure.microsoft.com/ja-jp/products/app-service/web)） |
| シークレット管理<br>（本記事では省略） | [Vercel](https://vercel.com/docs/projects/environment-variables)   | [Azure Key Vault](https://azure.microsoft.com/ja-jp/products/key-vault)                                                                               |
| DB                                     | [Vercel Postgres](https://vercel.com/docs/storage/vercel-postgres) | [Azure Database for PostgreSQL](https://azure.microsoft.com/ja-jp/products/postgresql)                                                                |
| OpenAI API                             | [OpenAI API](https://openai.com/blog/openai-api)                   | [Azure OpenAI Service](https://azure.microsoft.com/ja-jp/products/ai-services/openai-service)                                                         |
| CI/CD                                  | [Vercel](https://vercel.com/workflow)                              | [Azure DevOps](https://azure.microsoft.com/ja-jp/products/devops)（[Azure Pipelines](https://azure.microsoft.com/ja-jp/products/devops/pipelines)）   |

また、移行が完了したソースコードは以下の GitHub に入れてますのでご興味あればご覧ください。

https://github.com/m2-sakai/weight-management-app-azure

:::note warn

本記事は既に Azure のサブスクリプションを取得していることを前提としています。本記事を参考に自身で試してみたい方はまず、Azure のサブスクリプションを取得してください。

Azure は従量課金制で始めることも、最大 30 日間無料で試すこともできます。使用していない時期はリソースを何も作らなければ料金もかかりませんので、是非お手元で Azure がどんなものなのか触っていただけると幸いです。

:::

# Vercel Postgres から Azure Database for PostgreSQL への移行

それでは実際の移行に移ります。移行はどのリソースから行っても構いませんが、作成したリソースの内容を他のリソースに設定する（例：Azure App Service の環境変数に DB の接続文字列を設定する）ことがあるため、本記事では理解しやすくするために他サービスに依存しないものから記載します。

まずは、DB を Azure Database for PostgreSQL に移行します。Azure Database for PostgreSQL は、Microsoft Azure クラウドプラットフォーム上で提供されるフルマネージドなリレーショナルデータベースサービスの一つで、PostgreSQL データベースエンジンをベースにしています。

## リソースの作成

それではリソースを作成します。
Azure Portal に移動し、`ホーム > Azure Database for PostgreSQL` と移動し、`＋作成`を押下します。
Azure Database for PostgreSQL には、「単一サーバー」と「フレキシブル サーバー」の 2 つのデプロイモデルがあります。
どちらを採用するかは要件によって様々ですが、今回はコストをできるだけ抑えたいため、「フレキシブル サーバー」を選択します。詳しい比較については以下をご参照ください。

https://learn.microsoft.com/ja-jp/azure/postgresql/flexible-server/concepts-compare-single-server-flexible-server

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/5893ba47-330f-07c2-30a7-26be8e8b2a7a.png)

また、今回は開発用のため一番低いスペックで作成します。これでだいたい $23.40/月です。現在（2023/12）のおおよそのレートだと 3000 円強です。DB ってけっこう高いですよね。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/fda071c6-c25c-e570-c9b3-e7aae19be03d.png)

その他の設定は割愛します。`確認および作成`を押下し、しばらくすると設定した名前のデータベースが作成されます。

### 接続確認

それでは作成したデータベースに接続します。接続方法は「Cloud Shell」から接続する方法と、自身の PC から接続する方法があります（もちろんパブリックアクセスを有効化する必要有）。今回は簡単のため、「Cloud Shell」から接続します。

`接続 > はい`と進むと Cloud Shell が起動します。初回の起動のみ、ストレージアカウントの作成が要求されますが、ご負担でない場合は作成しちゃいましょう。Cloud Shell が実行されるのを確認したらパスワードが要求されますので、DB 作成時に設定したパスワードを入力するとログインが可能です。Cloud Shell に記載されている以下が接続情報となりますので、どこかにメモしておくことをお勧めします。

```sh:Cloud Shell
psql "--host=m2-sakai-postgres-db.postgres.database.azure.com" "--port=5432" "--dbname=postgres" "--username=postgres" "--set=sslmode=require"
```

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/c8ca17cd-5e45-b665-a0f3-c5f7750352fc.png)

## アプリケーションの修正

それでは接続確認も取れたため、アプリケーションの実装を修正します。
以前は Vercel Postgres を使用していたため `@vercel/postgres` を用いて DB に接続していましたが、今回は別の DB となるためデータベースの ORM を提供するツール・ライブラリの一つである [Prisma](https://www.prisma.io/) を使用します。

### Prisma のセットアップ

以下のコマンドからライブラリをインストールします。

```bash
npm install prisma
```

続いて以下のコマンドで Prisma の初期化を行います。

```bash
npx prisma init
```

これにより `prisma`というディレクトリの中に `schema.prisma` というデータベースのスキーマ情報が含まれる Prisma の設定ファイルが作成されます。
また `.env`の中に以下のようなデータベースの接続 URL が記載されます（`.env`ファイルが存在しない場合は自動で作成されます）ので、先程作成した DB の接続方法を用いて接続 URL を構成します。

```sh:.env
DATABASE_URL="postgresql://postgres:xxxxxxx@m2-sakai-postgres-db.postgres.database.azure.com:5432/postgres?schema=public"
```

### Prisma のスキーマ定義を用いてテーブルを作成する

Prisma のセットアップが完了したところで、データベーススキーマを作成します。前回の記事では [`scripts/seed.js`](https://github.com/m2-sakai/weight-management-app/blob/master/scripts/seed.js) を作成し、そこに DDL を記載していました。今回は同じような構造のテーブル（`wm_users`、`wm_weights`）を作成するように `schema.prisma` を修正していきます。以下か完成した Prisma スキーマです。

```prisma:prisma/schema.prisma
generator client {
  provider = "prisma-client-js"
  binaryTargets = ["native", "debian-openssl-3.0.x"]
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL") 				// .env から読み取る DB の接続 URL
}

model User {
  id          String    @default(cuid()) @id 	// 主キーには @id をつける
  name        String    @db.VarChar(255)
  email       String    @unique
  height      Decimal   @db.Decimal(4,1)
  password    String
  goal        Decimal   @db.Decimal(4,1)
  @@map(name: "wm_users") 						// テーブル名は @@map で設定可能
}

model Weight {
  userId      String    @map(name: "user_id")
  weight      Decimal   @db.Decimal(4,1)
  date        DateTime  @db.Date 				// @db.Dateとすることで Date型になる
  @@unique(fields: [userId, date]) 				// 複合主キーは @@uniqueで設定可能
  @@map(name: "wm_weights")						// テーブル名は @@map で設定可能
}
```

続いて、実際に DB を作成するために以下のコマンドを実行します。

```bash
npx prisma db push
```

すると、以下のようにテーブルが作成されます。これにてテーブルの作成は終了です。

```bash:Cloud Shell
postgres=> \d wm_users
                      Table "public.wm_users"
  Column  |          Type          | Collation | Nullable | Default
----------+------------------------+-----------+----------+---------
 id       | text                   |           | not null |
 name     | character varying(255) |           | not null |
 email    | text                   |           | not null |
 height   | numeric(4,1)           |           | not null |
 password | text                   |           | not null |
 goal     | numeric(4,1)           |           | not null |
Indexes:
    "wm_users_pkey" PRIMARY KEY, btree (id)
    "wm_users_email_key" UNIQUE, btree (email)
```

### Prisma Client をインストールし、DB に対して CRUD 処理を可能にする

テーブルも作成されたため、アプリから DB に接続します。そのためにまずは Prisma Client をインストールする必要があります。

```bash
npm install @prisma/client
```

続いて、Prisma Client との接続を作成するために `app/lib/prisma.ts` ファイルを作成します。

```ts:app/lib/prisma.ts
import { PrismaClient } from '@prisma/client';

let prisma: PrismaClient;

if (process.env.NODE_ENV === 'production') {
  prisma = new PrismaClient();
} else {
  let globalWithPrisma = global as typeof globalThis & {
    prisma: PrismaClient;
  };
  if (!globalWithPrisma.prisma) {
    globalWithPrisma.prisma = new PrismaClient();
  }
  prisma = globalWithPrisma.prisma;
}

export default prisma;
```

最後に、実際にデータを取得する部分を修正します。本記事では代表的なユーザ情報を取得する部分のみ例を示します。他の実装は [GitHub](https://github.com/m2-sakai/weight-management-app-azure) をご参照ください。

```diff_typescript:app/lib/user.ts
'use server';
- import { sql } from '@vercel/postgres';
import { unstable_noStore as noStore } from 'next/cache';
import { User } from '@/app/types/User';
+ import prisma from '@/app/lib/prisma';

export const getUser = async (email: string): Promise<User | null> => {
  noStore();
  try {
-	 const user = await sql<User>`SELECT *
-		FROM wm_users
-		WHERE email=${email}`;
-    return user.rows[0];
+    const user = await prisma.user.findFirst({
+      where: {
+        email: email,
+      },
+    });
+    return user;
  } catch (error) {
    throw new Error('Database Error: Failed to get user. error: ' + error);
  }
};
```

:::note info

Prisma Client の API リファレンスは[こちら](https://www.prisma.io/docs/orm/reference/prisma-client-reference)をご参照ください。

:::

以上でアプリケーションの修正も終了し、DB の移行が完了しました。

# OpenAI から Azure OpenAI Service への移行

続いて、チャット機能で使用している OpenAI の API を移行します。Azure を使用するため、もちろん移行先は Azure OpenAI Service です。

Azure OpenAI Service は、Microsoft のクラウドプラットフォーム Azure 上で利用できる OpenAI のサービスです。AI を活用した自然言語処理や、自動テキスト生成、意味理解など、さまざまなアプリケーションの開発が可能になります。

特に GPT-3 などの強力な自然言語理解モデルはヒューマンライクなテキスト生成が可能で、より自然な人間との対話を実現します。今回はこちらの機能を使用します。

## Azure OpenAI Service へのアクセス許可申請

Azure OpenAI Service は、OpenAI の強力な言語モデルを REST API として利用できるサービスですが現在、このサービスへのアクセスは申請によってのみ許可されています。申請フォーム「Request Access to Azure OpenAI Service」に必要事項を入力して送信し、承認されたサブスクリプションにて利用が可能になります。申請フォームの URL は[こちら](https://aka.ms/oai/access)です。

執筆時時点で記入項目は 20 項目以上あります。ここでの説明は割愛しますが、使用するための第一歩ですので根気強くフォームを書きましょう。
申請後、審査が行われ以下のようなメールが届きます。以前は審査に 10 営業日かかっていたという話もお聞きしますが、最近は即日に完了することも多いそうです。私も申請当日に審査が完了しました。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/6f1d3b89-58b6-c8f0-75b3-2a88701586c2.png)

## リソースの作成

申請が通ったところで、晴れてリソースを作成します。
`ホーム > Azure AI services` と移動し、`＋作成`を押下します。今回は以下のように設定しました。料金はトークンベースの従量課金になります。詳しくは[こちら](https://azure.microsoft.com/ja-jp/pricing/details/cognitive-services/openai-service/)をご参照ください。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/1635394e-96c1-8355-d056-6cb2e6cf3598.png)

続いて、モデルをデプロイします。`モデルデプロイ > 展開の管理`と進むと Azure AI Studio に遷移します。`管理 > デプロイ`から、`＋新しいデプロイの作成`を押下します。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/9e832c3b-204b-3e7c-c258-99b04936f8f5.png)

これによりモデルがデプロイされ、API を実行できる状態となります。また、API の実行には「API キー」が必要になります。`ホーム > Azure AI services > [作成したAzure OpenAI] > キーとエンドポイント`に遷移すると、API キーが表示されています。

::: note alert

前回の記事でも言及しましたが、API キーは秘匿情報となります。もし漏れた場合大量にリクエストが送信され高額の課金が発生してしまう可能性があるため、保管には十分に注意してください。間違っても GitHub に push しないようにしましょう。

:::

## アプリケーションの修正

それでは先程作成した Azure OpenAI Service を使用するようにアプリケーションを修正します。
以前は OpenAI API のエンドポイントを直接実行していましたが、今回は `@azure/openai` をインストールしてそちらを使用して API を実行します。また、フロントエンドから直接 API を実行してしまうと、開発者ツール等からヘッダー情報が参照できてしまうため、[Server Actions](https://nextjs.org/docs/app/api-reference/functions/server-actions) を用いてサーバ側から API を実行します。

完成したコードが以下となります。専用ライブラリを用いることで実装がよりシンプルになりました。

```diff_shell:.env
- NEXT_PUBLIC_OPENAI_API_KEY="xxxxxxxx"
+ AZURE_OPENAI_ENDPOINT="https://m2-sakai-aoai.openai.azure.com/"
+ AZURE_OPENAI_API_KEY="xxxxxxxx"
+ AZURE_OPENAI_MODEL_NAME="m2-sakai-aoai-model-gpt35"
```

```diff_typescript:app/lib/chat.ts
+ 'use server';

import { Message } from '@/app/types/Message';
- import axios from 'axios';
+ import { OpenAIClient, AzureKeyCredential } from '@azure/openai';

+ const client = new OpenAIClient(
+   process.env.AZURE_OPENAI_ENDPOINT ?? '',
+   new AzureKeyCredential(process.env.AZURE_OPENAI_API_KEY ?? '')
+ );
+ const modelName = process.env.AZURE_OPENAI_MODEL_NAME ?? '';

export const chat = async (chats: Message[], message: Message): Promise<Message> => {
  const messages = [...chats, message].map((d) => ({
    role: d.role,
    content: d.content,
  }));

- const response = await axios.post(
-   'https://api.openai.com/v1/chat/completions',
-   {
-     model: 'gpt-3.5-turbo',
-     messages: messages
-   },
-   {
-     headers: {
-       'Content-Type': 'application/json',
-       Authorization: `Bearer ${process.env.NEXT_PUBLIC_OPENAI_API_KEY}`,
-     },
-   }
- );
-
- const data = await response.data;
- if (response.status !== 200) {
-   throw data.error || new Error(`Request failed with status ${response.status}`);
- }
- return data.choices[0].message as Message;

+ try {
+   const response = await client.getChatCompletions(modelName, messages);
+   return response.choices[0].message as Message;
+ } catch (error) {
+   throw new Error('Chat Error error: ' + error);
+ }
};
```

以上でアプリケーションの修正も終了し、OpenAI の移行が完了しました。

# Vercel から Azure App Service への移行

続いて、ホスティングサービスを Vercel から Azure App Service に移します。
Azure App Service は Microsoft Azure クラウドプラットフォーム上で提供される、アプリケーションを構築、ホスト、およびスケールするためのフルマネージドなプラットフォームサービス（PaaS）です。これにより、開発者はアプリケーションの構築に必要なインフラストラクチャやサーバーの管理について心配せず、アプリケーションの開発、デプロイ、スケールを簡単に行うことができます。

また、Azure App Service には、いくつかの異なるアプリケーションタイプに対応するための特定のサービスが含まれています。その中で、Web Apps は Web アプリケーションを構築およびホストするためのサービスとなっています。今回はその Web Apps に Next.js アプリをデプロイします。

https://azure.microsoft.com/ja-jp/products/app-service/web

## リソースの作成

では、リソースを作成します。が、App Service を載せる App Service プランを先に構築する必要があります。
App Service プランは特に設定項目が少ないので、ここでの説明は割愛しますが一言だけ付け加えておくと OS は Linux、価格プランは F1（Free）を選択しました。

それでは App Service プランが作成できたところで、App Service の作成に移ります。
`ホーム > App Service` と移動し、`＋作成 > ＋Webアプリ`を押下します。今回の Next.js アプリはサーバ側も必要としますので、[静的 Web アプリ（Static Web Apps）](https://azure.microsoft.com/ja-jp/products/app-service/static)ではなく Web Apps を選択しています。今回は以下のように設定しています。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/6983e617-ffc2-7bc1-ecc3-1a772d974472.png)

これにて、Next.js アプリを載せる App Service の作成が完了しました。

## アプリケーションの設定

続いて、アプリケーションの`.env`に記載していた内容を記載する「アプリケーション設定」の記載及びアプリの全般設定を行います。
`[作成した Web Apps] > 構成`と移動し、`.env`に記載していた内容を設定します。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/21eeda5c-3663-cee2-66e0-cc457fc8f3da.png)

::: note warn

`.env` の中には DB の接続文字列（パスワード）や、Azure OpenAI Service の API キー等、秘匿情報があります。
この値をそのままアプリケーションに設定することはセキュリティ上お勧めしません。きちんとシークレットを安全に保管できる Azure Key Vault に保管し、[Key Vault 参照](https://learn.microsoft.com/ja-jp/azure/app-service/app-service-key-vault-references?tabs=azure-cli)を利用するようにしましょう。

:::

また、次にアプリケーションが動作するための全般設定を行います。`全般設定`タブに移動し、以下のようにスタートアップコマンドに`npm run start`を入力し、App Service の起動時に Next.js アプリが起動するように設定します。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/fa39d8e8-2085-6166-fe3b-1d9efaa313c2.png)

以上で、アプリケーションの設定も完了し、起動する準備も整いました。

# Azure DevOps による CI/CD への移行

最後に Next.js アプリを GitHub に push したらそのまま App Service にデプロイされるように CI/CD パイプラインを構築します。GitHub をソースコード管理として用いる場合 [GitHub Actions](https://github.co.jp/features/actions) を使用する方も多いかと思います。が、今回は Azure にデプロイするということで、敢えて Azure DevOps を用います。

Azure DevOps は、Microsoft が提供する統合開発プラットフォームであり、開発者やプロジェクトチームがアプリケーションの開発、テスト、リリース、および運用を管理するためのツールとサービスを提供しています。また、ソフトウェア開発ライフサイクル全体をカバーし、CI/CD の実装を容易にしてくれます。

::: note warn

以降は、Azure DevOps の Organization 及び Project まで作成していることを前提としております。
無料で開始することもできますので、ご興味ある方は是非[こちら](https://azure.microsoft.com/ja-jp/products/devops#get-started)から始めていただけたらと思います。

:::

## Azure Pipelines の初期構築

Azure DevOps で CI/CD を実現する場合、使用するサービスは Azure Pipelines となります。
Azure Pipelines は CI/CD パイプラインの構築、テスト、デプロイを管理するためのツールであり、異なるプラットフォームやクラウドに対応していることが特徴です。

それでは実際にパイプラインを構築していきます。まず、`Pipelines`に移動し、`New Pipelines`を押下します。
そして、作成したい GitHub のリポジトリ、（今回は Next.js なので）Node.js におけるパイプラインを選択すると以下のような デフォルトの `azure-pipelines.yml` がプロジェクトに push されます。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/65d2a9fb-e18e-cef9-7c1c-677aed16f267.png)

```yaml:azure-pipelines.yml
# Node.js
# Build a general Node.js project with npm.
# Add steps that analyze code, save build artifacts, deploy, and more:
# https://docs.microsoft.com/azure/devops/pipelines/languages/javascript

trigger:
- master

pool:
  vmImage: ubuntu-latest

steps:
- task: NodeTool@0
  inputs:
    versionSpec: '10.x'
  displayName: 'Install Node.js'

- script: |
    npm install
    npm run build
  displayName: 'npm install and build'
```

このパイプラインでは

- Node.js をインストールする
- Node.js プロジェクトのビルドする

のジョブで構成されています。これにて GitHub と Azure Pipelines が連携され、GitHub にコードを push するたびにパイプラインが実行されます。

## 詳細なパイプライン構築

上記の段階でパイプラインの実行は実現できましたが、まだ Azure にデプロイはできていません。デプロイを実現するにはより詳細なパイプラインを構築する必要があります。
そこでまず Azure にデプロイするために必要となるのが、「Service Connection の設定」です。`Project Settings > Pipelines_Service Connection` と移動し、`New Service Connection`を押下して以下のように作成します。先程各リソースを作成したサブスクリプションと紐づけることで、リソースやアプリのデプロイを可能にします。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/5ae74506-1717-dc34-b509-c2bd1b5fd51d.png)

それでは、実際のパイプラインを構築しましょう。今回のパイプラインでは以下のようなジョブを構成します。

- ビルドステージ
  - Node.js（20 系）をインストールする
  - Next.js アプリをビルドする
  - ビルドしたモジュールは zip 化し、artifact に upload する
- デプロイステージ
  - ビルドしたモジュールを Web Apps にデプロイする

この構成で最終的に出来上がったパイプラインが以下となります。

```yaml:azure-pipelines.yml
trigger:
  - master

variables:
  azureSubscription: 'm2-sakai-service-connection'
  webAppName: 'weight-management-app-azure'
  environmentName: 'Production'
  vmImageName: 'ubuntu-latest'

stages:
  - stage: Build
    displayName: Build stage
    jobs:
      - job: Build
        displayName: Build
        pool:
          vmImage: $(vmImageName)

        steps:
          - task: NodeTool@0
            inputs:
              versionSpec: '20.x'
            displayName: 'Install Node.js'

          - script: |
              npm install
              npm run build --if-present
            displayName: 'npm install and build'

          - task: ArchiveFiles@2
            displayName: 'Archive files'
            inputs:
              rootFolderOrFile: '$(System.DefaultWorkingDirectory)'
              includeRootFolder: false
              archiveType: zip
              archiveFile: $(Build.ArtifactStagingDirectory)/$(Build.BuildId).zip
              replaceExistingArchive: true

          - upload: $(Build.ArtifactStagingDirectory)/$(Build.BuildId).zip
            artifact: drop

  - stage: Deploy
    displayName: Deploy stage
    dependsOn: Build
    condition: succeeded()
    jobs:
      - deployment: Deploy
        displayName: Deploy
        environment: $(environmentName)
        pool:
          vmImage: $(vmImageName)
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureWebApp@1
                  displayName: 'Azure Web App Deploy: weight-management-app-azure'
                  inputs:
                    azureSubscription: $(azureSubscription)
                    appType: webAppLinux
                    appName: $(webAppName)
                    package: $(Pipeline.Workspace)/drop/$(Build.BuildId).zip
```

## パイプライン実行及び動作確認

それではパイプラインが構築できたところで、GitHub にソースコードを push し、Azure にデプロイされるか確認します。
パイプラインが実行されると、以下のように Stages / Jobs の成功/失敗が一目で見て取れます。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/61b3c7a3-4804-a1fa-c503-7d01eb08d169.png)

全てのジョブが成功し App Service の URL にアクセスすると、晴れて前回作成したアプリケーションが表示され、各機能も動作することが確認出来ました。

![app_azure.gif.gif](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2344522/0606c5a9-1f62-bdbb-be40-5e43162a77c0.gif)

# おわりに

いかがでしたでしょうか。本記事では、前回の記事で作成した Next.js のアプリを Azure にデプロイするまでの奮闘記を記載しました。今や様々な技術があり、デプロイ先や CI/CD も様々な選択肢が存在します。ともかく一度触ってみて自身で動かした経験を持っておくと、いざ業務で使用するとなったときにスムーズにキャッチアップできると思います。
私もこれからも幅広く知識を身に着け、発信していこうと思います。ここまで読んでいただき誠にありがとうございました。

# 参考文献

今回も非常に多くの文献を参考にさせていただきました。心より感謝申し上げます。

- [Next.js､ Prisma､PostgreSQL でフルスタックアプリを作る](https://qiita.com/tomohiko_ohhashi/items/da804ed1f5870c9ce52d)
- [NextJS Deployment on App Service Linux](https://azureossd.github.io/2022/10/18/NextJS-deployment-on-App-Service-Linux/)
- [Azure OpenAI Service を使用してテキストの生成を開始する](https://learn.microsoft.com/ja-jp/azure/ai-services/openai/quickstart?tabs=command-line%2Cpython&pivots=programming-language-studio)
