# Week12 課題: フロントエンドとバックエンドの分離

## 概要
フロントエンドとバックエンドを構築し、フロントからバックエンドのAPIを呼び出してみます。
- フロントエンド: week11の構成を使います（CloudFront + S3）
- バックエンド: week10の構成を使います（ALB + ECS）

## 目標
- ローカル環境でフロントエンドとバックエンドの動作確認
- バックエンドのAWS環境構築（ECRへのイメージプッシュ）
- フロントエンドのビルドとS3へのデプロイ
- CloudFront経由でアプリケーション動作確認

---

## アーキテクチャ図

```
[Users] → [CloudFront] → [S3 (Frontend)]
                           ↓ (API Call)
                        [ALB] → [ECS (Backend)]
                                  ↓
                               [RDS]
```

### 通信情報
- フロントエンド: ブラウザ → CloudFront → S3
- バックエンド: ブラウザ → ALB → ECS

### アクセス方法
動作例は以下のようになります。
- ブラウザで `https://<GitHubID>.posse-ap.com/index.html` にアクセスすると、Reactのプログラムが取得されてブラウザで動作します
- Reactプログラムの中でAPIを呼び出す処理が書かれているので、ブラウザからAPI `https://api.<GitHubID>.posse-ap.com/api/dummy-data` が呼び出されます

### 通信経路
- **フロントエンド**: ブラウザ → CloudFront → S3
- **バックエンド**: ブラウザ → ALB → ECS

---

## 1. 今回やること

フロントとバックエンドを構築し、フロントからバックエンドのAPIを呼び出してみます。

### 使用する構成
- フロントエンド: week11の構成を使います
- バックエンド: week10の構成を使います

### 動作仕様
ブラウザで `https://<GitHubID>.posse-ap.com/index.html` にアクセスすると、Reactのプログラムが取得されてブラウザで動作します。

`http://localhost/api/dummy-data` にリダイレクトされます。

以下のようにユーザー覧のJSONが表示されることを確認してください。

**通信経路は以下となります:**
- フロントエンド: ブラウザ → CloudFront → S3
- バックエンド: ブラウザ → ALB → ECS

---

## 2. 前準備

week10とweek11の環境を削除している場合は再度作成してください。

**必要なリソースで主要なものは、CloudFront、S3、ALB、ECS となります**

※RDSは利用しません

---

## 3. ローカルの環境構築

ビルド用の資材をcloneしてください。

**GitHubリポジトリ**: https://github.com/posse-ap/ph5-AWS-week12

### バックエンドの環境構築

ローカルで環境構築し、動作確認をします。

まずバックエンドの環境を構築します。ターミナルで以下のコマンドを実行してください。

```bash
cd ph5-AWS-week12
docker-compose up -d
docker-compose exec app composer install
```

ブラウザで `http://localhost` にアクセスしてください。`http://localhost/api/dummy-data` にリダイレクトされます。

以下のようにユーザー覧のJSONが表示されることを確認してください。

```json
[{"name":"John Doe","email":"john.doe@example.com","age":30},{"name":"Jane Smith","email":"jane.smith@example.com","age":25},{"name":"Bob Johnson","email":"bob.johnson@example.com","age":35}]
```

### フロントエンドの環境構築

続いてフロントエンドの環境を構築します。ターミナルで以下のコマンドを実行してください。

```bash
cd front
npm install
npm start
```

ブラウザが自動的に起動し、`http://localhost:3000` にアクセスされます。

以下のようにユーザー覧が表示されることを確認してください。バックエンドから取得したJSONがリスト化されたものが画面に表示されます。

### User List
- John Doe
  john.doe@example.com
  30
- Jane Smith
  jane.smith@example.com
  25
- Bob Johnson
  bob.johnson@example.com
  35

---

## 4. バックエンドのAWS環境構築

week10の手順「3-3.ECRにイメージをプッシュ」を実施します。

「ph5-AWS-week12」ディレクトリで実施してください。

---

## 5. バックエンドの動作確認

ブラウザを起動して、`https://api.<GitHubID>.posse-ap.com/api/dummy-data` にアクセスしてみましょう。

---

## 6. フロントエンドのAWS環境構築

### フロントエンドの資材をビルドします

```bash
cd front
npm run build
```

コンソールにアクセスし、「S3」サービスを開く

バケット名「ph5aws-<GitHubID>-web」を選択

Finderで「/front/build」ディレクトリを開いて、全てのファイルをS3にドラッグしてアップロードします

アップロード後は以下のようになります:

```
ph5aws-silent-web
- asset-manifest.json
- favicon.ico
- index.html
- logo192.png
- logo512.png
- manifest.json
- robots.txt
- static/
  - css/
  - js/
  - media/
```

---

## 6. フロントエンドの動作確認

ブラウザを起動して、`https://<GitHubID>.posse-ap.com/index.html` にアクセスしてみましょう。

※事前にローカル環境のDockerコンテナを停止してからアクセスしてください

以下のように表示されます:

### User List
- John Doe
  john.doe@example.com
  30
- Jane Smith
  jane.smith@example.com
  25
- Bob Johnson
  bob.johnson@example.com
  35

---

## 7. 後片付け

ALB、ECS、RDSを使は、もしくは削除しましょう

---

## トラブルシューティング

### バックエンドAPIにアクセスできない場合
- ECSタスクが正常に起動しているか確認
- ALBのセキュリティグループで必要なポートが開いているか確認
- Route53でapi.<GitHubID>.posse-ap.comのレコードが正しく設定されているか確認

### フロントエンドが表示されない場合
- S3バケットにファイルが正しくアップロードされているか確認
- CloudFrontのキャッシュをクリアしてみる
- ブラウザのキャッシュをクリアしてみる

### CORSエラーが発生する場合
- バックエンドのCORS設定を確認
- ALBのリスナールールを確認

---

## まとめ

この課題では、フロントエンドとバックエンドを分離した構成を学習しました。フロントエンドは静的ファイルとしてS3にホストし、CloudFrontで配信します。バックエンドはECSでAPIサーバーとして動作させ、ALB経由でアクセスします。この構成により、各レイヤーを独立してスケールできる柔軟なアーキテクチャが実現できます。
