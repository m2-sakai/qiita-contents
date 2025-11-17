# Qiita Contents

[![Qiita](https://img.shields.io/badge/Qiita-@s__w__high-55C500?logo=qiita)](https://qiita.com/s_w_high)

このリポジトリは、Qiita に投稿した技術記事を管理するためのものです。

## 著者

- **Qiita**: [@s_w_high](https://qiita.com/s_w_high)
- **所属**: [野村総合研究所（NRI）](https://qiita.com/organizations/nri)

## ディレクトリ構成

```text
qiita-contents/
├── public/             # 公開済みの記事（Markdown形式）
├── work/               # 執筆中の記事（下書き）
├── qiita.config.json   # Qiita CLI設定ファイル
├── package.json        # npm設定ファイル
└── README.md
```

## セットアップ

このリポジトリを使用して記事を執筆・管理するには、以下の手順でセットアップしてください。

### 前提条件

- Node.js（v18 以上推奨）
- npm または yarn
- Git

### インストール

```bash
# 依存関係のインストール
npm install

# Qiita CLIのバージョン確認
npx qiita version
```

### Qiita CLI の使い方

```bash
# 新しい記事の作成
npx qiita new 記事のタイトル

# 記事のプレビュー（ローカルサーバー起動）
npx qiita preview

# Qiitaへの記事の投稿・更新
npx qiita publish 記事ファイル名.md

# すべての記事を一括で同期
npx qiita pull
```

プレビューサーバーは `http://localhost:8888` で起動します（`qiita.config.json` で設定可能）。

## 記事の執筆フロー

1. **下書き作成**: `npx qiita new` で新しい記事を作成（`work/` ディレクトリに配置）
2. **ローカルプレビュー**: `npx qiita preview` でプレビューしながら執筆
3. **公開準備**: フロントマター（メタデータ）を編集
   - `private: false` に設定して公開記事にする
   - `tags` を適切に設定
4. **記事公開**: 記事を `public/` ディレクトリに移動
5. **Git 管理**: 変更をコミット・プッシュ

## フロントマターの設定例

```yaml
---
title: 記事のタイトル
tags:
  - Azure
  - .NET
  - TypeScript
private: false
updated_at: '2025-11-18T00:00:00+09:00'
id: null
organization_url_name: nri
slide: false
ignorePublish: false
---
```

## 技術スタック・トピック

主に以下のトピックについて執筆しています：

- **クラウド**: Azure, AWS
- **IaC**: Azure Bicep, Terraform, AWS CDK, Pulumi
- **言語**: C#, .NET, TypeScript, JavaScript, Java
- **フレームワーク**: Next.js, Remix, .NET, Spring
- **AI/ML**: OpenAI, Azure OpenAI, MCP (Model Context Protocol)
- **認証**: Microsoft Entra ID, OAuth 2.0, Managed Identity
- **開発ツール**: Visual Studio, VS Code, GitHub Copilot

## 記事の閲覧

すべての公開記事は [Qiita プロフィール](https://qiita.com/s_w_high) からご覧いただけます。

## ライセンス

各記事の著作権は著者に帰属します。

## お問い合わせ

記事に関する質問や誤りの指摘などは、[Qiita の各記事のコメント欄](https://qiita.com/s_w_high)までお願いします。
