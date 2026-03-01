# Week15 課題: GitHub Actionsによる自動デプロイ（バックエンド→ECR & ECS）

## 概要
10週目でバックエンドのコンテナイメージをビルドしてECRに配置しましたが、ソースを修正するたびにコンテナイメージのビルドや、ECSのコンテナイメージが入替えを手動で実装するのは、作業ミスもつながるので、GitHub Actionsを利用して、GitHubの資材が更新されたら自動的にECRへのプッシュ&ECSコンテナ入替えが実施されるようにします。

## 目標
- デプロイ用のIAM Roleの修正（ECRとECSの権限追加）
- GitHub Actionsの構築（バックエンド用）
- 動作確認（GitHubへpushするとECR&ECSに自動デプロイされる）

---

## アーキテクチャ図

```
[GitHub] → [GitHub Actions] → [ECR]
                                ↓
                              [ECS]
                                ↓
                              [RDS]
```

### 今回の範囲
GitHubの資材を自動的にECRにプッシュし、ECSコンテナに入替えを実施します。

---

## 1. 今回やること

10週目でバックエンドのコンテナイメージをビルドしてECRに配置しましたが、ソースを修正するたびにコンテナイメージのビルドや、ECSのコンテナイメージが入替えを手動で実装するのは、作業ミスもつながるので、GitHub Actionsを利用して、GitHubの資材が更新されたら自動的にECRへのプッシュ&ECSコンテナ入替えが実施されるようにします。

---

## 2. 前準備

特にありません

---

## 3. デプロイ用のIAM Role修正

GitHub ActionsからAWSのS3に資材を配置するため認証用のIAM Roleを14週目で作成しました。

今回はこのIAM Roleを使って、ECRへのプッシュ&ECSコンテナ入替えを実装するので必要な権限を追加で設定します。

### IAM Roleの修正手順

1. **コンソール**にアクセスし、「IAM」サービスを開く
2. 左ペインから「ロール」を選択
3. ロール名「GitHubActions」のリンクをクリックする

### 許可ポリシーの追加

**「許可を追加」を選択**

**「許可ポリシー」で以下をチェック:**
- **「AmazonEC2ContainerRegistryFullAccess」**
- **「AmazonECS_FullAccess」**

**「許可を追加」を選択**

---

## 4. GitHub Actionsの構築

ビルド用の資材をcloneしてください

**GitHubリポジトリ**: https://github.com/compose-ap/ph5-AWS-week12

※week14でcloneしたものが残っている場合はそのまま利用してください

### GitHub Actionsの設定ファイルを作成します

ターミナルで以下のコマンドを実行してください

```bash
cd ph5-AWS-week12
mkdir -p .github/workflows
touch .github/workflows/deploy_backend.yml
```

Visual Studio Codeで`deploy_backend.yml`を開いて以下のように設定してください

**以下の箇所を修正します:**
- SERVICE_NAME: カリキュラム内で今まで作っていれば基本そのままでOK
- ROLE_TO_ASSUME: <作成したIAM RoleのARNにしてください>

```yaml
env:
  SERVICE_NAME: ph5aws
  ROLE_TO_ASSUME: arn:aws:iam::207095571838:role/GitHubActionsRole

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
      
      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v1
      
      - name: Build Web
        id: build-web
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          ECR_REPOSITORY: ${{ env.SERVICE_NAME }}-phpfpm
          DOCKER_FILE: ./docker/php/Dockerfile.aws
        run: |
          docker build -t $ECR_REPOSITORY .
          docker tag $ECR_REPOSITORY:latest $ECR_REGISTRY/$ECR_REPOSITORY:latest
          echo "set-output name=image::$ECR_REGISTRY/$ECR_REPOSITORY:latest"
      
      - name: Build App
        id: build-app
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          ECR_REPOSITORY: ${{ env.SERVICE_NAME }}-phpfpm
          DOCKER_FILE: ./docker/php/Dockerfile.aws
        run: |
          docker build -t $ECR_REPOSITORY .
          docker tag $ECR_REPOSITORY:latest $ECR_REGISTRY/$ECR_REPOSITORY:latest
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:latest
          echo "set-output name=image::$ECR_REGISTRY/$ECR_REPOSITORY:latest"
      
      - name: Get ECS Task Definition
        id: create-taskdef
        run: |
          task-definition=$(aws ecs describe-task-definition --task-definition ${{ env.SERVICE_NAME }} | jq '.taskDefinition')
          container-name=${{ env.SERVICE_NAME }}-phpfpm
          image=${{ steps.build-app.outputs.task-definition }}
          echo "$task-definition" | jq --arg IMAGE "$image" --arg CONTAINER_NAME "$container-name" '.containerDefinitions[] | select(.name==$CONTAINER_NAME) .image=$IMAGE' > ./task_definition.json
      
      - name: Fill in the new image ID App
        id: render-taskdef-app
        uses: aws-actions/amazon-ecs-render-task-definition@v1
        with:
          task-definition: ./task_definition.json
          container-name: ${{ env.SERVICE_NAME }}-phpfpm
          image: ${{ steps.build-app.outputs.image }}
      
      - name: Deploy Amazon ECS
        uses: aws-actions/amazon-ecs-deploy-task-definition@v1
        with:
          task-definition: ${{ steps.render-taskdef-app.outputs.task-definition }}
          service: ${{ env.SERVICE_NAME }}
          cluster: ${{ env.SERVICE_NAME }}
          wait-for-service-stability: true
```

---

## 5. 動作確認

上記の変更をGitHubにプッシュしてください

その後、GitHub Actionsが正常に動作することを確認してください

### 確認内容
本当に変更されたことを確認するために、以下の変更履歴を実際してからプログラムが再デプロイされて、デプロイが完了した後に、ブラウザでアクセスして変更が反映されることを確認してください

**対象ファイル**: `ph5-AWS-week12/backend/routes/api.php`

**対象ステップ**: L20

**変更箇所**:
- 変更前: John Doe
- 変更後: Michael Andrew Fox

---

## トラブルシューティング

### GitHub Actionsが失敗する場合

#### ECRへのログインエラー
- IAM RoleにECRへの権限があるか確認
- ECRリポジトリが存在するか確認
- リージョンが正しいか確認

#### Dockerビルドエラー
- Dockerfileのパスが正しいか確認
- ビルドコンテキストが正しいか確認
- 必要なファイルが全て存在するか確認

#### ECSタスク定義の取得エラー
- タスク定義名が正しいか確認
- タスク定義が存在するか確認
- IAM RoleにECSへの読み取り権限があるか確認

#### ECSへのデプロイエラー
- サービス名とクラスター名が正しいか確認
- IAM RoleにECSへの更新権限があるか確認
- タスク定義の形式が正しいか確認

---

## GitHub Actionsのポイント

### マルチステップビルド
```yaml
- name: Build Web
  id: build-web
  
- name: Build App
  id: build-app
```
NginxとPHP-FPMの2つのコンテナイメージをビルドします。

### タスク定義の動的生成
```yaml
- name: Get ECS Task Definition
  id: create-taskdef
  run: |
    task-definition=$(aws ecs describe-task-definition --task-definition ${{ env.SERVICE_NAME }} | jq '.taskDefinition')
```
既存のタスク定義を取得し、新しいイメージIDで更新します。

### イメージIDの受け渡し
```yaml
echo "set-output name=image::$ECR_REGISTRY/$ECR_REPOSITORY:latest"
```
ビルドしたイメージのURIを後続のステップで利用できるように出力します。

---

## まとめ

この課題では、GitHub Actionsを使ってバックエンドの自動デプロイを実現しました。mainブランチへのpushをトリガーに、Dockerイメージのビルド、ECRへのプッシュ、ECSタスクの更新まで自動化されます。これにより、インフラの手動操作をなくし、継続的デリバリーを実現できます。
