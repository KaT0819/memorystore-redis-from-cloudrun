# memorystore-redis-from-cloudrun


## gcloudのアップデート
``` shell
gcloud components update
```

## 環境変数の設定
``` shell
export PROJECT_ID={プロジェクトID}
export REGION=asia-northeast1
# Memorystore for RedisのインスタンスID
export REDIS_INSTANCE=myinstance
```

## Memorystore for Redis インスタンスを作成
``` shell
# プロジェクトセット
gcloud config set core/project ${PROJECT_ID}

# Redisインスタンスの作成
gcloud redis instances create ${REDIS_INSTANCE} --size=1 --region=${REGION} \
    --redis-version=redis_5_0

# IPアドレスとポート番号取得
gcloud redis instances describe ${REDIS_INSTANCE} --region=${REGION}

# ~~~
host: IPアドレス
# ...
port: ポート番号
# ~~~

# 確認したIPアドレスとポート番号を環境変数にセット（後でCloudRunの環境変数に指定）
export REDIS_HOST={IPアドレス}
export REDIS_PORT={ポート番号}

```

## サーバーレス VPC アクセスの構成
Redis インスタンスに接続するには、Cloud Run サービスが Redis インスタンスの承認済み VPC ネットワークにアクセスする必要があり、
そのためには、サーバーレス VPC アクセス コネクタが必要となる。

``` shell
# Redis インスタンスの承認済みネットワークの名前を確認
gcloud redis instances describe ${REDIS_INSTANCE} --region ${REGION} --format "value(authorizedNetwork)"

# Serverless VPC Access API の有効化
gcloud services enable vpcaccess.googleapis.com

# コネクタの作成（簡易版）
export CONNECTOR_NAME=redis-net

gcloud compute networks vpc-access connectors create ${CONNECTOR_NAME} \
--network default \
--region ${REGION} \
--range 10.8.0.0/28

# コネクタの作成（独自のサブネットを使用する場合）
gcloud compute networks vpc-access connectors create ${CONNECTOR_NAME} \
--region ${REGION} \
--subnet {SUBNET} \
# If you are not using Shared VPC, omit the following line.
--subnet-project {HOST_PROJECT_ID} \
# Optional: specify minimum and maximum instance values between 2 and 10, default is 2 min, 10 max.
--min-instances {MIN} \
--max-instances {MAX} \
# Optional: specify machine type, default is e2-micro
--machine-type {MACHINE_TYPE}

# 準備OKか確認
gcloud compute networks vpc-access connectors describe ${CONNECTOR_NAME} --region ${REGION}
```


## コンテナ イメージを構築
``` shell
# ビルド
gcloud builds submit --tag gcr.io/${PROJECT_ID}/visit-count

# デプロイ
gcloud run deploy \
--image gcr.io/${PROJECT_ID}/visit-count \
--platform managed \
--allow-unauthenticated \
--region ${REGION} \
--vpc-connector ${CONNECTOR_NAME} \
--set-env-vars REDISHOST=${REDIS_HOST},REDISPORT=${REDIS_PORT}
```
