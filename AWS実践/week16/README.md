# Week16 課題: ログ収集（CloudWatch Logs）

## 概要
今は /api/dummy-data で固定レスポンスを返していますが、usersテーブルを検索して、JSONでデータを送るようにします。さらに出力の変更を行います。

また、ログが出力されるようになったら、AWSでは自動的にCloudWatch Logsにログを出力することで、Docker/logからLaravelのログ情報を確認できます。

## 目標
- DB検索APIの実装
- ログ出力先の変更（標準エラー出力に変更）
- CloudWatch Logsでログ確認
- AWS環境にデプロイして動作確認

---

## アーキテクチャ図

```
[Users] → [CloudFront] → [S3 (Frontend)]
                           ↓ (API Call)
                        [ALB] → [ECS (Backend)]
                                  ↓
                               [RDS]
                                  ↓
                         [CloudWatch Logs]
```

### 今回の範囲
バックエンドのログをCloudWatch Logsに出力し、ログ情報を確認できるようにします。

---

## 1. 今回やること

今は /api/dummy-data で固定レスポンスを返していますが、usersテーブルを検索して、JSONでデータを送るようにします。

さらに出力の変更を行います。S3に出力していたログはs3:/storage/logs/laravel.logにログが出力しているのですが、ログ出力先をDocker/logから変更できるようにします。

Dockerのlogに出力することで、AWS上では自動的にCloudWatch Logsにログを出力する設定ができます。

---

## 2. 前準備

15週目の課題ができていることが前提となります

---

## 3. APIの実装をDB検索するように変更

backend/routes/api.phpに以下の記述を追加します。

### 変更前
```php
Route::get('/users', function() {
    return User::all();
});
```

### 変更後（DB検索してユーザー一覧がJSONで送信される）
```php
Route::get('/users', function() {
    Log::info('ログ出力カスト'); // ここを追加
    return User::all();
});
```

また、フロントから呼び出すAPIのURLも変更したいので、front/src/App.jsを以下のように編集します

### 変更前
```javascript
useEffect(() => {
  fetch('${apiUrl}/users') // ここを変更
    .then((response) => response.json())
    .then((data) => setUsers(data));
}, [apiUrl]);
```

### 変更後
```javascript
useEffect(() => {
  fetch('${apiUrl}/users') // ここを変更
    .then((response) => response.json())
    .then((data) => setUsers(data));
}, [apiUrl]);
```

localhost:3000にアクセスし、ダミーデータから変更されていれば成功です

---

## 4. ログの出力先を標準（エラー）出力に変更

backend/.env の LOG_CHANNEL を変更し、ログの出力先を変更します

### 変更内容
```env
LOG_CHANNEL=stderr
```

なお、stderrの定義は backend/config/logging.php に記載されています

---

## 5. ローカル環境で動作確認

以下のコマンドでログが確認できるようにします。

```bash
docker compose logs -f
```

ターミナルで以下のように「ログ出力カスト」と表示されれば成功です

```
ph5-aws-week12-app-1    | [2024-03-30 16:10:15] local.INFO: ログ出力カスト
```

次に: backend/routes/api.php を以下のように編集します

### コード変更
```php
Route::get('/users', function() {
    Log::info('ログ出力カスト'); // ここを追加
    return User::all();
});
```

---

## 5. AWS環境にデプロイして動作確認

以上の変更を GitHub に push しましょう

15週目の内容でしっかりと GitHub Actions が起動されます。Migration、Seederはコンテナ起動時に自動的に実行されます

※ ph5-AWS-week12/docker/php/startup.sh 参照

### ログの確認方法

ログは、ECS のクラスター > サービス > ログ にて確認できます

追加リバージョンのログが出力されていない場合は、タスクの定義を確認しましょう

「新しいリバージョンの作成」時に「ログの収集」にチェックを入れることで、CloudWatch Logsにログを出力する設定ができます

---

## CloudWatch Logsの確認方法

### ECSコンソールからの確認
1. ECSコンソールを開く
2. クラスターを選択
3. サービスを選択
4. 「ログ」タブを選択
5. ロググループを選択してログを確認

### CloudWatch Logsコンソールからの確認
1. CloudWatch Logsコンソールを開く
2. ロググループ `/ecs/ph5aws` を選択
3. ログストリームを選択
4. ログメッセージを確認

### ログの検索
CloudWatch Logs Insightsを使用して、ログを検索・分析できます。

```sql
fields @timestamp, @message
| filter @message like /ログ出力カスト/
| sort @timestamp desc
| limit 20
```

---

## トラブルシューティング

### ログが出力されない場合
- タスク定義で「ログの収集」が有効になっているか確認
- IAM RoleにCloudWatch Logsへの書き込み権限があるか確認
- LOG_CHANNELがstderrに設定されているか確認

### APIがエラーを返す場合
- RDSへの接続情報が正しいか確認
- ECSタスクのセキュリティグループでRDSへのアクセスが許可されているか確認
- データベースにusersテーブルが存在するか確認

### フロントエンドがデータを表示しない場合
- APIのURLが正しいか確認
- CORSの設定が正しいか確認
- ブラウザのコンソールでエラーを確認

---

## ログレベルについて

Laravelでは以下のログレベルが使用できます：

- **emergency**: システムが使用不可能
- **alert**: 即座に対応が必要
- **critical**: 重大な状態
- **error**: エラー状態
- **warning**: 警告状態
- **notice**: 通常だが重要な状態
- **info**: 情報メッセージ
- **debug**: デバッグ情報

```php
Log::emergency('システムダウン');
Log::alert('データベース接続失敗');
Log::critical('ディスク容量不足');
Log::error('処理エラー発生');
Log::warning('非推奨機能の使用');
Log::notice('ユーザーログイン');
Log::info('API呼び出し');
Log::debug('変数の値: ' . $value);
```

---

## まとめ

この課題では、LaravelアプリケーションのログをCloudWatch Logsに出力する設定を行いました。ログを標準エラー出力に変更することで、ECSのログドライバーが自動的にCloudWatch Logsに転送します。これにより、本番環境でのログ監視とトラブルシューティングが容易になります。
