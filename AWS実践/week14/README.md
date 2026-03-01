# Week14 課題: GitHub Actionsによる自動デプロイ（フロントエンド→S3）

## 概要
12週目でフロントの資材を手動でS3に配置しましたが、これを修正するたびに手動で実装するのは、作業ミスもつながるので、GitHub Actionsを利用して、GitHubの資材が更新されたら自動的にS3へ配置されるようにします。

## 目標
- GitHub ActionsからAWSのS3に資材を配置するための認証用のIAM Roleを作成
- GitHub Actionsの構築
- 動作確認（GitHubへpushするとS3に自動デプロイされる）

---

## アーキテクチャ図

```
[GitHub] → [GitHub Actions] → [S3]
                                ↑
                          [CloudFront]
```

### 今回の範囲
GitHubの資材を自動的にS3に配置し、CloudFrontで配信できるようにします。

---

## 1. 今回やること

12週目でフロントの資材を手動でS3に配置しましたが、これを修正するたびに手動で実装するのは、作業ミスもつながるので、GitHub Actionsを利用して、GitHubの資材が更新されたら自動的にS3へ配置されるようにします。

---

## 2. 前準備

特にありません

---

## 3. デプロイ用のIAM Role作成

GitHub ActionsからAWSのS3に資材を配置するため認証用のIAM Roleを作成します

### IAM Roleの作成手順

1. **コンソール**にアクセスし、「IAM」サービスを開く
2. 左ペインから「ロール」を選択
3. 「ロールを作成」を選択

### 信頼されたエンティティタイプの選択
- **「ウェブアイデンティティ」を選択**
- **「アイデンティティプロバイダー」**のリストボタンの「新規作成」の右側の画面にある（例えば理解されているはずです）

### プロバイダの設定
- **「プロバイダのタイプ」で「OpenID Connect」を選択**
- **「プロバイダのURL」に `https://token.actions.githubusercontent.com` を入力**
- **「サムプリント」の「サムプリントを取得」を選択**
- **「サムプリント」が「1b511abaed59c6ce207077c0bf0e043b1382612」となっていることを確認**

### Audience（オーディエンス）の設定
- **「対象者」に「sts.amazonaws.com」を入力**
- **「プロバイダを追加」を選択**

### 信頼されたエンティティを選択
- （再度同じ画面に戻る（例えば理解されているはずです））
- **「アイデンティティプロバイダー」のリロードボタンを選択**
- **「アイデンティティプロバイダー」で「token.actions.githubusercontent.com」選択**
- **「Audience」で「sts.amazonaws.com」を選択**
- **「次へ」を選択**

### 許可ポリシーの設定
- **「許可ポリシー」で「AmazonS3FullAccess」「CloudFrontFullAccess」をチェック**
- **「次へ」を選択**

### ロール名の設定
- **「ロール名」に「GitHubActions」を入力**
- **「ロールを作成」を選択**

### 信頼関係の編集

作成されたロール「GitHubActionsRole」を選択して内容を確認する

「信頼関係」タブを開く

「信頼ポリシーを編集」を選択

「Condition」の部分を以下のように編集します:

**以下の箇所を修正します:**
- posse-apの部分を自分で作成しているGitHubのアカウント
- ph5-AWS-week12の部分を、現在使っているGitHubのリポジトリ

```json
"Condition": {
  "StringEquals": {
    "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
  },
  "StringLike": {
    "token.actions.githubusercontent.com:sub": [
      "repo:posse-ap/ph5-AWS-week12:*"
    ]
  }
}
```

**「ポリシーを更新」を選択**

---

## 4. GitHub Actionsの構築

ビルド用の資材をcloneしてください

**GitHubリポジトリ**: https://github.com/posse-ap/ph5-AWS-week12

※week12でcloneしたものが残っている場合はそのまま利用してください

### GitHub Actionsの設定ファイルを作成します

ターミナルで以下のコマンドを実行してください

```bash
cd ph5-AWS-week12
mkdir -p .github/workflows
touch .github/workflows/deploy.yml
```

Visual Studio Codeで`deploy.yml`を開いて以下のように設定してください

**以下の箇所を修正します:**
- arn:aws:iam::207095571838:role/GitHubActionsRole の部分を、作成したIAM RoleのARNにしてください
- ph5aws-silent-web の部分を、フロントの資材を置くS3バケット名にしてください
- E3D2X7GTFZDJCS の部分を、CloudFrontのIDにしてください

```yaml
name: Deploy AWS

on:
  push:
    branches:
      - main

env:
  ROLE_TO_ASSUME: arn:aws:iam::207095571838:role/GitHubActionsRole
  FRONT_BUCKET_NAME: ph5aws-silent-web
  CLOUDFRONT_ID: E3D2X7GTFZDJCS

jobs:
  deploy:
    name: Deploy
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    
    steps:
      - name: Checkout
        uses: actions/checkout@v3
      
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v1
        with:
          role-to-assume: ${{ env.ROLE_TO_ASSUME }}
          role-session-name: github-actions-${{ github.run_id }}
          aws-region: ap-northeast-1
      
      - name: Get AWS Account
        run: aws sts get-caller-identity
      
      - name: Build
        run: |
          cd front
          cp .env.aws .env
          rm .env.dev
          yarn install
          yarn build
      
      - name: Deploy
        run: |
          aws s3 sync front/build/ s3://${{ env.FRONT_BUCKET_NAME }}/
          --delete --exact-timestamps
          aws cloudfront create-invalidation --distribution-id ${{ env.CLOUDFRONT_ID }} --paths "/*"
```

---

## 5. 動作確認

上記の変更をGitHubにプッシュしてください

その後、GitHub Actionsが正常に動作することを確認してください

### 確認内容
本当に変更されたことを確認するために、以下の変更履歴を実際してから再度プッシュして、変更が反映されることを確認してください

**対象ファイル**: `ph5-AWS-week12/backend/routes/api.php`

**変更箇所**:
- 変更前: User List
- 変更後: User List GitHub Actions Test

---

## トラブルシューティング

### GitHub Actionsが失敗する場合

#### 権限エラーが発生する場合
- IAM Roleの信頼関係が正しく設定されているか確認
- IAM Roleに必要なポリシーがアタッチされているか確認
- リポジトリ名とブランチ名が正しいか確認

#### ビルドエラーが発生する場合
- package.jsonの依存関係を確認
- Node.jsのバージョンを確認
- .envファイルが正しく配置されているか確認

#### S3へのアップロードが失敗する場合
- バケット名が正しいか確認
- IAM RoleにS3への書き込み権限があるか確認

#### CloudFrontのキャッシュが更新されない場合
- CloudFront IDが正しいか確認
- create-invalidationコマンドが実行されているか確認
- 数分待ってから再度アクセスしてみる

---

## GitHub Actionsのポイント

### permissions設定
```yaml
permissions:
  id-token: write
  contents: read
```
これにより、OIDCを使用したAWSの認証が可能になります。

### 環境変数の使用
```yaml
env:
  ROLE_TO_ASSUME: ${{ env.ROLE_TO_ASSUME }}
```
再利用可能な値は環境変数として定義します。

### ステップの構成
1. **Checkout**: リポジトリのコードを取得
2. **Configure AWS Credentials**: AWS認証
3. **Get AWS Account**: アカウント情報確認（デバッグ用）
4. **Build**: フロントエンドのビルド
5. **Deploy**: S3へのアップロードとCloudFrontキャッシュの無効化

---

## まとめ

この課題では、GitHub Actionsを使ってフロントエンドの自動デプロイを実現しました。mainブランチへのpushをトリガーに、ビルドからS3へのアップロード、CloudFrontのキャッシュ無効化まで自動化されます。これにより、手動デプロイのミスを防ぎ、開発効率が大幅に向上します。
