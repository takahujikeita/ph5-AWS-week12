# Week10 課題: ECS + ECR + RDS + ALB構成

## 概要
今回の課題では、ECSコンテナからDBアクセスを可能にし、ECRにNginxとphp-fpmのDockerイメージを作成して、ALB経由でECSにアクセスしてRDSのデータを表示します。

## 目標
- ECSコンテナからDBアクセスを可能にする
- ECRにNginxとphp-fpmのDockerイメージを作成
- ALB経由でECSにアクセスしてRDSのデータを表示

---

## 手順

### 1. ECRリポジトリの作成

#### 1.1 Nginxリポジトリの作成
1. AWSマネジメントコンソールでECRサービスを開く
2. 「リポジトリを作成」をクリック
3. リポジトリ名: `ph5aws-nginx`
4. その他の設定はデフォルトのまま作成

#### 1.2 PHP-FPMリポジトリの作成
1. 同様に2つ目のリポジトリを作成
2. リポジトリ名: `ph5aws-phpfpm`
3. 作成後、2つのリポジトリが表示されることを確認

### 2. Dockerイメージのビルドとプッシュ

#### 2.1 GitHubリポジトリのクローン
```bash
git clone https://github.com/[リポジトリURL]
cd [リポジトリディレクトリ]
```

#### 2.2 ECRへのログイン
```bash
aws ecr get-login-password --region ap-northeast-1 | docker login --username AWS --password-stdin [アカウントID].dkr.ecr.ap-northeast-1.amazonaws.com
```

#### 2.3 Nginxイメージのビルドとプッシュ
```bash
# イメージのビルド
docker build -t ph5aws-nginx ./nginx

# タグ付け
docker tag ph5aws-nginx:latest [アカウントID].dkr.ecr.ap-northeast-1.amazonaws.com/ph5aws-nginx:latest

# ECRへプッシュ
docker push [アカウントID].dkr.ecr.ap-northeast-1.amazonaws.com/ph5aws-nginx:latest
```

#### 2.4 PHP-FPMイメージのビルドとプッシュ
```bash
# イメージのビルド
docker build -t ph5aws-phpfpm ./php-fpm

# タグ付け
docker tag ph5aws-phpfpm:latest [アカウントID].dkr.ecr.ap-northeast-1.amazonaws.com/ph5aws-phpfpm:latest

# ECRへプッシュ
docker push [アカウントID].dkr.ecr.ap-northeast-1.amazonaws.com/ph5aws-phpfpm:latest
```

#### 2.5 プッシュの確認
ECRコンソールで各リポジトリにイメージがプッシュされたことを確認します。

### 3. ECSタスク定義の作成

#### 3.1 タスク定義の設定
1. ECSコンソールで「タスク定義」→「新しいタスク定義の作成」
2. 起動タイプ: Fargate
3. タスク定義名: 任意の名前を設定
4. タスクロール: ecsTaskExecutionRole
5. タスクメモリ: 適切なサイズを選択
6. タスクCPU: 適切なサイズを選択

#### 3.2 コンテナの追加

**Nginxコンテナ:**
- コンテナ名: nginx
- イメージURI: [アカウントID].dkr.ecr.ap-northeast-1.amazonaws.com/ph5aws-nginx:latest
- メモリ制限: ソフト制限 512MB
- ポートマッピング: 80

**PHP-FPMコンテナ:**
- コンテナ名: php-fpm
- イメージURI: [アカウントID].dkr.ecr.ap-northeast-1.amazonaws.com/ph5aws-phpfpm:latest
- メモリ制限: ソフト制限 512MB
- 環境変数: RDSの接続情報を設定
  - DB_HOST
  - DB_NAME
  - DB_USER
  - DB_PASSWORD

#### 3.3 タスク定義の作成
すべての設定を確認して、タスク定義を作成します。

### 4. ALBヘルスチェックの設定変更

#### 4.1 問題の確認
ECSサービスを起動すると、ヘルスチェックが失敗してタスクが繰り返し起動・停止する場合があります。

#### 4.2 ヘルスチェックの修正
1. EC2コンソールで「ロードバランサー」を開く
2. 対象のALBを選択
3. 「ターゲットグループ」タブを開く
4. ターゲットグループを選択→「ヘルスチェック」タブ
5. 「編集」をクリック
6. 成功コードを `200-399` に変更（302リダイレクトを許容）
7. 変更を保存

**理由**: アプリケーションが302リダイレクトを返す場合、デフォルトの200のみの設定ではヘルスチェックが失敗するため、200-399の範囲を許容するように変更します。

### 5. ECSサービスの起動

1. ECSコンソールでクラスターを選択
2. 「サービス」タブで「作成」をクリック
3. 起動タイプ: Fargate
4. タスク定義: 作成したタスク定義を選択
5. サービス名: 任意の名前を設定
6. タスク数: 1
7. ロードバランサー: 既存のALBを選択
8. ターゲットグループ: 作成済みのターゲットグループを選択
9. サービスを作成

### 6. 動作確認

#### 6.1 ECSタスクの確認
- ECSコンソールでタスクが正常に実行中（RUNNING）になっていることを確認
- ヘルスチェックが正常に通過していることを確認

#### 6.2 アプリケーションへのアクセス
1. ALBのDNS名をブラウザで開く
2. アプリケーションが表示されることを確認
3. RDSからデータが正常に取得・表示されることを確認

---

## Windows環境での設定

### AWS CLIツールのインストール（PowerShell）

#### 前提条件
- Windows 10以降
- PowerShell 5.1以降

#### インストール手順

1. PowerShellを管理者として開く

2. AWS Toolsのインストール
```powershell
Install-Module -Name AWS.Tools.Installer
```

3. 必要なAWSモジュールのインストール
```powershell
Install-AWSToolsModule AWS.Tools.ECR
Install-AWSToolsModule AWS.Tools.ECS
```

4. AWS認証情報の設定
```powershell
Set-AWSCredential -AccessKey [YOUR_ACCESS_KEY] -SecretKey [YOUR_SECRET_KEY] -StoreAs default
Set-DefaultAWSRegion -Region ap-northeast-1
```

5. 動作確認
```powershell
Get-ECRRepository
```

---

## トラブルシューティング

### ヘルスチェックが失敗する場合
- ALBのセキュリティグループでHTTP(80)が許可されているか確認
- ターゲットグループのヘルスチェック設定を確認
- 成功コードに302が含まれているか確認（200-399）

### タスクが起動しない場合
- タスク定義のコンテナイメージURIが正しいか確認
- タスク実行ロールに必要な権限があるか確認
- CloudWatch Logsでエラーログを確認

### RDSに接続できない場合
- セキュリティグループでECSからRDSへの接続が許可されているか確認
- 環境変数でDB接続情報が正しく設定されているか確認
- RDSのエンドポイントが正しいか確認

---

## 参考情報

- [Amazon ECS ドキュメント](https://docs.aws.amazon.com/ecs/)
- [Amazon ECR ドキュメント](https://docs.aws.amazon.com/ecr/)
- [Application Load Balancer ドキュメント](https://docs.aws.amazon.com/elasticloadbalancing/)
- [Docker ドキュメント](https://docs.docker.com/)

---

## まとめ
この課題では、コンテナ化されたアプリケーションをAWS上で実行するための基本的な構成を学習しました。ECS、ECR、RDS、ALBを組み合わせることで、スケーラブルで管理しやすいアプリケーション環境を構築できます。
