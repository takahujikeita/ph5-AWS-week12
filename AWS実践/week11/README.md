# Week11 課題: CloudFront + S3構築

## 概要
CloudFrontとS3を構築し、オブジェクトストレージとCDNを利用したWebサイト配信環境を構築します。

## 目標
- S3バケットの作成とWebサイトホスティングの設定
- CloudFrontディストリビューションの作成
- SSL証明書の設定（ACM）
- Route53でドメイン設定
- ログ取得の設定

---

## 料金について

### S3の料金
S3は主にストレージの利用量で課金されます。
- **0.025USD/GB**: 1GBのファイルを置いて1ヶ月で約0.025USD
- リクエストの作法によって差はあるが、前期にならば十分安価

S3の価格: https://aws.amazon.com/jp/s3/pricing/

### CloudFrontの料金
CloudFrontは主に通信量で課金されます。
- **0.110USD/GB**: 1GBのファイルを外部送信して約0.11USD
- 1GBのファイルを外部送信して10億円程度になりますが、リソースの作法によって差はあるので前期にならば安価

CloudFrontの価格: https://aws.amazon.com/jp/cloudfront/pricing/

---

## 1. 今回やること

### アーキテクチャ図
```
[Users] → [CloudFront] → [S3 Bucket(オリジン)]
                           ↓
                        [S3 Bucket(ログ用)]
```

### 構成要素
- **CloudFront**: CDNサービス、S3をオリジンとして設定
- **S3**: オブジェクトストレージ、静的Webサイトのホスティング
- **ACM**: SSL証明書の発行
- **Route53**: DNSサービス、ドメイン管理

### 実現すること
CloudFrontとS3を構築します。S3はオブジェクトストレージと呼ばれており、ここにファイルを置いてインターネットに公開することの設定が可能です。

S3単体の場合は、S3に直接アクセスした場合となります。
- https://s3.ap-northeast-1.amazonaws.com/<バケット名>

CloudFrontと組み合わせると、S3の前にCDN機能が加わります。
- http://<バケット名>.s3-website.ap-northeast-1.amazonaws.com
- S3の前段のWebアクセスホスティング機能を利用

などで公開が可能です。

まずは、CloudFrontと組み合わせる場合で、独自のドメイン
- https://GitHubID>.posse-ap.com/

などで公開することが目標となります。

ドメインが自由に選択出来るため、CloudFront + S3で公開するケースが多いです。今回はCloudFront + S3を利用していきます。

---

## 2. 前準備

特にありません。

---

## 3. S3の作成

S3はバケットという単位で作成します。バケットは用途ごとに作成します。

### バケット作成の例：
①静的なサイトを公開する場合
- バケットを2つ作成します
- サイトのコンテンツを置くバケット
- CloudFrontのログを格納するバケット

②複雑なサイトを公開する場合
- 単体はフロントがReact、バックエンドがLaravel + MySQLという構成で
  バケットごとに各利用者の成果物のPDFをアップロードする要件がある場合
  バケットを3つ作成します
  - フロントの資材を置くバケット
  - バックエンドの各種雑用の成果物ファイルをアップロードするバケット
  - CloudFront、ALBのログを格納するバケット

今回は静的なサイトを構成していきますので、バケットを2つ作っていきます。

### 3-1. サイトの資材を置くバケットの作成

1. **コンソール**にアクセスし、「S3」サービスを開く
2. 「バケットを作成」を選択する
3. **バケット名**: `ph5aws-<GitHubID>-web` を入力
4. **リージョン**: 「アジアパシフィック（東京）ap-northeast-1」を選択
5. その他はデフォルト設定で、「バケットを作成」を選択

#### ファイルのアップロード
動作確認用のindex.htmlを作成してアップロードしておきます。

```bash
echo "hello s3" > index.html
```

作成したファイルをブラウザからアップロードします。

### 3-2. ログを置くバケットの作成

1. **コンソール**にアクセスし、「S3」サービスを開く
2. 「バケットを作成」を選択する
3. **バケット名**: `ph5aws-<GitHubID>-log` を入力
4. **リージョン**: 「アジアパシフィック（東京）ap-northeast-1」を選択
5. その他はデフォルト設定で、「バケットを作成」を選択

---

## 3-3. CloudFrontの作成

1. **コンソール**にアクセスし、「CloudFront」サービスを開く
2. 「ディストリビューションを作成」を選択

### Origin（オリジン）の設定
- **オリジンドメイン**: 手順3-1で作成したS3バケットを指定
- **名前**: `ph5aws-<GitHubID>-web` を入力
- **オリジンアクセス**: 「Origin access control settings」を選択

### Origin access control の作成
- **「コントロール設定を作成」を選択**
- デフォルト設定のまま、「作成」を選択

### キャッシュポリシー
- **「CachingDisabled」を選択**

#### 理由
演習は「CachingOptimized」を選択しますが、キャッシュされると念願の場所で不具合がおきるかどうかの判断をすることが難しいので一旦短時間ではキャッシュしないよう設定します

### 代替ドメイン名（CNAME）の設定
- **「項目を追加」を選択**
- テキストボックスに `<GitHubID>.posse-ap.com` を入力

### カスタムSSL証明書
- **「証明書をリクエスト」を選択**

#### パブリック証明書をリクエスト
- **「次へ」を選択**
- **「完全修飾ドメイン名」に `<GitHubID>.posse-ap.com` を入力**
- その他はデフォルト設定で、「リクエスト」を選択
- 作成した証明書を選択し、「作成」を選択

### Route53レコードの作成
- **「Route53でレコードを作成」を選択**
- **「レコードを作成」を選択**

左ペインから「証明書を一覧」を選択し、作成した証明書のステータスが「発行済み」になるのを待ちます。

---

### S3バケットポリシーの更新

1. 手順3-1で作成したS3バケットを指定
2. ACLの設定が必要と表示された場合は設定するボタンを押下

### ログプレフィックスの設定
- **「cloudfront」を入力**
- **「cookieログ記録」で「オン」を選択**
- **「既定を保存」を選択**

演習ページは時間的かかわります。
ディストリビューションの「ステータス」が「有効」になるのを待ちます。

---

## 3-4. Route53の設定

コンソールにアクセスし、「Route53」サービスを開く

### レコードの作成
1. **「<GitHubID>.posse-ap.com」のホストゾーンを選択**
2. 「レコードを作成」を選択する

### レコードの設定
- 存在する場合は既存レコードをチェックし「レコードを編集」を選択
- 存在しない場合は「レコードを作成」をするのを選択

### 作成設定内容
- **レコード名**: 空欄
- **レコードタイプ**: 「A」
- **エイリアス**: 有効にする
- **トラフィックのルーティング先**: 「CloudFrontディストリビューションへのエイリアス」を選択
- 先程作成したCloudFrontを選択し、「レコードを作成」を選択

---

## 4. 動作確認

ブラウザで、`https://<GitHubID>.posse-ap.com/index.html`にアクセスすると、hello s3 と表示されることを確認してください

### ログの確認

その後、ログ用のバケットを開閲し、ログが出力されていることを確認してください。ファイルをダウンロードし解凍して、テキストエディタなどで開いて確認してください。

ブラウザからAPI(/url)もしてログが出力されることを確認してください。

以下のように記録されていればOKです:

```
#Version: 1.0
#Fields: date time x-edge-location sc-bytes c-ip cs-method cs(Host) cs-uri-stem sc-status ca(Referer) cs(User-Agent) cs-uri-query ca(Cookie) x-edge-result-type x-edge-request-id x-host-header cs-protocol cs-bytes time-taken x-forwarded-for ssl-protocol ssl-cipher x-edge-response-result-type cs-protocol-version fle-status fle-encrypted-fields c-port time-to-first-byte x-edge-detailed-result-type sc-content-type sc-content-len sc-range-start sc-range-end
2023-03-15    18:28:29    NRT57-P4    354
125.101.198.19    GET    d2jyv4swnn0m74.cloudfront.net
/index.html    200    -    -    -
52593    0.864    Error    application/xml -    -
```

- index.htmlはステータス200で取得出来ている
- favicon.icoはステータス403で取得できていない

といったことが確認できます。

---

## 5. ログ取得設定

追加でCloudFrontへのアクセスログ取得の設定を行います。今後動作が上手く行かない場合に、CloudFrontではアクセスが来ているのかといつのを確認するために利用します。

1. **コンソール**にアクセスし、「CloudFront」サービスを開く
2. 先程作成したディストリビューションを選択
3. 「一般」タブの「編集」を選択
4. 「標準ログ記録」で「オン」を選択

---

## 6. 動作確認

ブラウザで、`https://<GitHubID>.posse-ap.com/index.html`にアクセスすると、hello s3 と表示されることを確認してください。

その後、ログ用のバケットを開閲し、ログが出力されていることを確認してください。ファイルをダウンロードし解凍して、テキストエディタなどで開いて確認してください。

以下のように記録されていればOKです:

```
None
#Version: 1.0
#Fields: date time x-edge-location sc-bytes c-ip cs-method cs(Host) cs-uri-stem sc-status ca(Referer) cs(User-Agent) cs-uri-query ca(Cookie) x-edge-result-type x-edge-request-id x-host-header cs-protocol cs-bytes time-taken x-forwarded-for ssl-protocol ssl-cipher x-edge-response-result-type cs-protocol-version fle-status fle-encrypted-fields c-port time-to-first-byte x-edge-detailed-result-type sc-content-type sc-content-len sc-range-start sc-range-end
2023-03-15    18:28:29    NRT57-P4    354
125.101.198.19    GET    d2jyv4swnn0m74.cloudfront.net
/index.html    200
Mozilla/5.0%20(Macintosh;%20Intel%20Mac%20OS%20X%2010_15_7)%20AppleWebKit/537.36%20(KHTML,%20like%20Gecko)%20Chrome/111.0.0.0%20Safari/537.36    Error
HnBYaXEqAGIDAenkdGl2KMWvBJSayABmORGgwAD1M91xTbvvJzgvQ==
silent.posse-ap.com    https    506    0.883    -
TLSv1.3 TLS_AES_128_GCM_SHA256 Error    HTTP/2.0
-    -    52593    0.883    Miss    text/html    9    -
-
2023-03-15    18:28:29    NRT57-P4    482
125.101.198.19    GET    d2jyv4swnn0m74.cloudfront.net
/favicon.ico    403 https://silent.posse-ap.com/index.html
Mozilla/5.0%20(Macintosh;%20Intel%20Mac%20OS%20X%2010_15_7)%20AppleWebKit/537.36%20(KHTML,%20like%20Gecko)%20Chrome/111.0.0.0%20Safari/537.36    Miss
wnBYaXEqAGIDankdG12KMWvBJSayABmORGgwAD1M91xTbvvJzgvQ==
silent.posse-ap.com    https    506    0.883    -
TLSv1.3 TLS_AES_128_GCM_SHA256 Error    HTTP/2.0
-    -    52593    0.883    Miss    text/html    9
```

index.htmlはステータス200で取得出来ているfavicon.icoはステータス403で取得できていないといったことが確認できます

---

## 7. 後片付け

今回作成したS3、CloudFrontは費用が大きく発生しないリソースなので、削除などは行わなくて問題ありません。

---

## トラブルシューティング

### CloudFrontのディストリビューションが有効にならない場合
- SSL証明書が「発行済み」になっているか確認
- Route53でレコードが正しく作成されているか確認

### アクセスしても403エラーが表示される場合
- S3バケットポリシーが正しく設定されているか確認
- CloudFrontのOrigin access controlが正しく設定されているか確認

### SSL証明書が発行されない場合
- Route53にドメインが正しく設定されているか確認
- ACMで証明書のリクエストステータスを確認

---

## まとめ

この課題では、CloudFrontとS3を使った静的Webサイトのホスティングとログ収集の基本を学習しました。CDNを使うことで、高速なコンテンツ配信と、SSL/TLS対応、独自ドメインでの公開が可能になります。
