# Week13 課題: GitHub Actionsの学習

## 概要
GitHub Actionsの基本を学習します。

## 目標
- GitHub Actionsの基本概念を理解する
- ワークフローの作成と実行
- GitHub Actionsのチュートリアルに従って動作確認

---

## 1. 今回やること

**GitHubの学習をします**

以下ページでGitHub Actionsの理解を進めてください。

### 学習リソース

#### GitHub Actions を理解する
一旦内容を軽く確認してください

📚 **リンク**: [GitHub Actions を理解する](https://docs.github.com/ja/actions/learn-github-actions/understanding-github-actions)

#### GitHub Actions のワークフロー構文
主に「ワークフローについて」「ワークフローコマンド」を確認してください

📚 **リンク**: [GitHub Actions のワークフロー構文](https://docs.github.com/ja/actions/using-workflows/workflow-syntax-for-github-actions)

---

## 2. 前準備

特にありません

---

## 3. GitHub Actionsを試す

### 手順

#### GitHub Actions を理解する
任意のリポジトリを作成し、チュートリアルの手順に従ってGitHub Actionsの動作確認を行ってください

📚 **リンク**: [GitHub Actions を理解する](https://docs.github.com/ja/actions/learn-github-actions/understanding-github-actions)

---

## 4. 後片付け

特にありません

---

## GitHub Actionsの基本概念

### ワークフロー（Workflow）
- `.github/workflows/` ディレクトリに配置するYAMLファイル
- リポジトリ内で自動化したい処理を定義
- イベントによってトリガーされる

### イベント（Events）
- ワークフローを実行するトリガー
- 例: `push`, `pull_request`, `schedule`, `workflow_dispatch`

### ジョブ（Jobs）
- ワークフロー内で実行される一連のステップ
- 並列実行または順次実行が可能
- 各ジョブは独立したランナーで実行される

### ステップ（Steps）
- ジョブ内で実行される個々のタスク
- アクションまたはシェルコマンドを実行

### アクション（Actions）
- 再利用可能なワークフローのコンポーネント
- GitHub Marketplaceから利用可能
- 自作も可能

### ランナー（Runners）
- ワークフローを実行する仮想マシン
- GitHub-hostedまたはSelf-hosted

---

## サンプルワークフロー

```yaml
name: Hello World Workflow

on:
  push:
    branches:
      - main

jobs:
  hello:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Say hello
        run: echo "Hello, GitHub Actions!"
      
      - name: Show date
        run: date
```

---

## よく使うイベント

### push
特定のブランチへのプッシュ時に実行
```yaml
on:
  push:
    branches:
      - main
      - develop
```

### pull_request
プルリクエストの作成・更新時に実行
```yaml
on:
  pull_request:
    branches:
      - main
```

### schedule
定期実行（Cron形式）
```yaml
on:
  schedule:
    - cron: '0 0 * * *'  # 毎日0時に実行
```

### workflow_dispatch
手動実行
```yaml
on:
  workflow_dispatch:
```

---

## よく使うアクション

### actions/checkout@v3
リポジトリのコードをチェックアウト
```yaml
- uses: actions/checkout@v3
```

### actions/setup-node@v3
Node.js環境のセットアップ
```yaml
- uses: actions/setup-node@v3
  with:
    node-version: '18'
```

### actions/upload-artifact@v3
成果物のアップロード
```yaml
- uses: actions/upload-artifact@v3
  with:
    name: build-files
    path: build/
```

---

## トラブルシューティング

### ワークフローが実行されない場合
- `.github/workflows/` ディレクトリに配置されているか確認
- YAMLファイルの構文エラーがないか確認
- トリガーイベントが正しく設定されているか確認

### ワークフローがエラーで失敗する場合
- GitHub Actionsのログを確認
- 各ステップのエラーメッセージを確認
- 必要な権限が付与されているか確認

---

## 参考リンク

- [GitHub Actions ドキュメント](https://docs.github.com/ja/actions)
- [GitHub Actions Marketplace](https://github.com/marketplace?type=actions)
- [ワークフロー構文](https://docs.github.com/ja/actions/using-workflows/workflow-syntax-for-github-actions)

---

## まとめ

この課題では、GitHub Actionsの基本概念とワークフローの作成方法を学習しました。GitHub Actionsを使うことで、CI/CD、テスト、デプロイなど、さまざまなタスクを自動化できます。次週以降では、実際のAWSへのデプロイを自動化していきます。
