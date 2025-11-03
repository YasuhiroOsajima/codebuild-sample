# codebuild-sample
StepFunctions -> CodeBuild -> S3

## ゴールと全体像
- Step Functions が CodeBuild を同期実行（完了まで待機）。
- CodeBuild は Docker 特権モードで起動し、ECR へログイン → 画像 pull → docker save で tar を生成 → S3 へアップロード。
- すべて ap-northeast-1（東京） 前提。

## 1) 事前準備（S3バケットとIAM）

### 1-1. S3バケット（成果物置き場）
```
REGION=ap-northeast-1
BUCKET=my-eb-nginx-tars-$(date +%Y%m%d%H%M%S)
aws s3api create-bucket \
  --bucket $BUCKET \
  --region $REGION \
  --create-bucket-configuration LocationConstraint=$REGION

# （任意）暗号化・ブロック公開設定
aws s3api put-bucket-encryption --bucket $BUCKET --server-side-encryption-configuration '{
  "Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]
}'
aws s3api put-public-access-block --bucket $BUCKET --public-access-block-configuration '{
  "BlockPublicAcls": true, "IgnorePublicAcls": true, "BlockPublicPolicy": true, "RestrictPublicBuckets": true
}'
```

### 1-2. CodeBuild サービスロール（最小権限）
必要権限のポイント
- ECR ログイン & イメージ取得: ecr:GetAuthorizationToken, ecr:BatchGetImage, ecr:GetDownloadUrlForLayer, ecr:BatchCheckLayerAvailability
- S3 へ put: s3:PutObject, s3:AbortMultipartUpload, s3:ListBucket, など
- CloudWatch Logs 出力
- （Step Functions から CodeBuild を起動するので）Step Functions の実行ロールは別途用意
```
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
cat > codebuild-role-trust.json <<'JSON'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "codebuild.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
JSON

aws iam create-role \
  --role-name cb-nginx-tar-role \
  --assume-role-policy-document file://codebuild-role-trust.json

cat > codebuild-inline-policy.json <<JSON
{
  "Version": "2012-10-17",
  "Statement": [
    {"Effect":"Allow","Action":[
      "logs:CreateLogGroup","logs:CreateLogStream","logs:PutLogEvents"
    ],"Resource":"*"},
    {"Effect":"Allow","Action":[
      "ecr:GetAuthorizationToken","ecr:BatchGetImage",
      "ecr:GetDownloadUrlForLayer","ecr:BatchCheckLayerAvailability",
      "ecr:DescribeImages"
    ],"Resource":"*"},
    {"Effect":"Allow","Action":[
      "s3:PutObject","s3:AbortMultipartUpload","s3:ListBucket","s3:GetBucketLocation"
    ],"Resource":[
      "arn:aws:s3:::${BUCKET}",
      "arn:aws:s3:::${BUCKET}/*"
    ]}
  ]
}
JSON

aws iam put-role-policy \
  --role-name cb-nginx-tar-role \
  --policy-name cb-nginx-tar-inline \
  --policy-document file://codebuild-inline-policy.json
```

※ ECR が別アカウントの場合は、相手側 ECR リポジトリポリシーにこの CodeBuild ロールの pull 許可を付与してください。

## 2) CodeBuild プロジェクト
### 2-1. buildspec（ECR から pull → tar → S3 put）
ポイント
- PrivilegedMode: true（Docker CLI を使うため必須）
- sourceType: NO_SOURCE（ソース不要、buildspec をプロジェクト側に持つ）
- aws ecr get-login-password でログイン
- docker pull → docker save（ファイルサイズが大きくても aws s3 cp は自動でマルチパート）
- 変数は環境変数で渡す（画像URI、出力先パスなど）

`buildspec-nginx-tar.yml`
```
version: 0.2
env:
  variables:
    IMAGE_URI: "XXXXXXXXXXXX.dkr.ecr.ap-northeast-1.amazonaws.com/nginx:mainline-alpine3.22"
    OUTPUT_FILE: "/tmp/nginx-mainline-alpine3.22.tar"
    ARTIFACT_BUCKET: ""   # Step Functions から上書き
    ARTIFACT_KEY: ""      # Step Functions から上書き (例: images/nginx-mainline-alpine3.22.tar)
phases:
  pre_build:
    commands:
      - echo "Logging in to ECR..."
      - aws --region ap-northeast-1 ecr get-login-password | docker login --username AWS --password-stdin XXXXXXXXXXXX.dkr.ecr.ap-northeast-1.amazonaws.com
      - echo "Pulling $IMAGE_URI"
      - docker pull "$IMAGE_URI"
  build:
    commands:
      - echo "Saving image to $OUTPUT_FILE"
      - docker save "$IMAGE_URI" -o "$OUTPUT_FILE"
      - ls -lh "$OUTPUT_FILE"
  post_build:
    commands:
      - echo "Uploading to s3://$ARTIFACT_BUCKET/$ARTIFACT_KEY"
      - aws s3 cp "$OUTPUT_FILE" "s3://$ARTIFACT_BUCKET/$ARTIFACT_KEY"
artifacts:
  files:
    - '**/*'   # 明示不要（S3に直接putするため）。残してもOK
```

### 2-2. プロジェクト作成
```
SERVICE_ROLE_ARN="arn:aws:iam::${ACCOUNT_ID}:role/cb-nginx-tar-role"

aws codebuild create-project \
  --name cb-ecr-nginx-to-tar \
  --region ap-northeast-1 \
  --source "{\"type\":\"NO_SOURCE\",\"buildspec\":${BUILD_SPEC}}" \
  --environment '{
    "type": "LINUX_CONTAINER",
    "image": "aws/codebuild/standard:7.0",
    "computeType": "BUILD_GENERAL1_MEDIUM",
    "privilegedMode": true
  }' \
  --service-role "$SERVICE_ROLE_ARN" \
  --artifacts '{"type":"NO_ARTIFACTS"}' \
  --timeout-in-minutes 30 \
  --queued-timeout-in-minutes 60 \
  --no-badge-enabled \
  --logs-config '{"cloudWatchLogs":{"status":"ENABLED"}}'
```

イメージサイズが大きい場合や複数回実行する場合は、computeType=BUILD_GENERAL1_LARGE や --secondary-artifacts は不要ですが、privilegedMode=true は忘れないでください。
VPC 内でインターネットなしで動かす場合は、VPC 設定と VPCエンドポイント（com.amazonaws.ap-northeast-1.ecr.api / ecr.dkr / S3 Gateway / Logs / STS など）を用意してください。

## 3) Step Functions（CodeBuild を同期実行）
### 3-1. ステートマシン定義（ASL）
- codebuild:startBuild.sync を使い、ビルド完了まで待機。
- 入力（S3出力先など）を CodeBuild の environmentVariablesOverride で渡す。
- リトライ（スロットリング等）を付与。

```
{
  "Comment": "Pull ECR image and export as tar via CodeBuild",
  "StartAt": "StartCodeBuild",
  "States": {
    "StartCodeBuild": {
      "Type": "Task",
      "Resource": "arn:aws:states:::codebuild:startBuild.sync",
      "Parameters": {
        "ProjectName": "cb-ecr-nginx-to-tar",
        "EnvironmentVariablesOverride": [
          {
            "Name": "ARTIFACT_BUCKET",
            "Type": "PLAINTEXT",
            "Value.$": "$.artifactBucket"
          },
          {
            "Name": "ARTIFACT_KEY",
            "Type": "PLAINTEXT",
            "Value.$": "$.artifactKey"
          }
        ]
      },
      "Retry": [
        {
          "ErrorEquals": ["CodeBuild.AWSCodeBuildException", "States.TaskFailed", "States.Timeout", "ThrottlingException"],
          "IntervalSeconds": 5,
          "BackoffRate": 2.0,
          "MaxAttempts": 5
        }
      ],
      "End": true
    }
  }
}
```
実行入力例：
```
{
  "artifactBucket": "my-eb-nginx-tars-20251103xxxxxx",
  "artifactKey": "images/nginx-mainline-alpine3.22.tar"
}
```

## 3-2. Step Functions 実行ロール
必要権限のポイント
- codebuild:StartBuild / codebuild:BatchGetBuilds
- （CloudWatch Logs へ出力はステートマシン設定側で）

```
cat > sfn-trust.json <<'JSON'
{
  "Version":"2012-10-17",
  "Statement":[{
    "Effect":"Allow",
    "Principal":{"Service":"states.amazonaws.com"},
    "Action":"sts:AssumeRole"
  }]
}
JSON
aws iam create-role --role-name sfn-starts-codebuild-role --assume-role-policy-document file://sfn-trust.json

cat > sfn-inline-policy.json <<JSON
{
  "Version":"2012-10-17",
  "Statement":[
    {
      "Effect":"Allow",
      "Action": [
        "codebuild:StartBuild",
        "codebuild:BatchGetBuilds"
      ],
      "Resource": [
        "arn:aws:codebuild:ap-northeast-1:${ACCOUNT_ID}:project/cb-ecr-nginx-to-tar"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "events:PutTargets",
        "events:PutRule",
        "events:DescribeRule"
      ],
      "Resource": [
        "arn:aws:events:ap-northeast-1:${ACCOUNT_ID}:rule/StepFunctions*"
      ]
    }
  ]
}
JSON

aws iam put-role-policy --role-name sfn-starts-codebuild-role --policy-name sfn-starts-codebuild-inline --policy-document file://sfn-inline-policy.json
```

### 3-3. ステートマシン作成

```
cat > sfn-definition.json <<'JSON'
{ ... 上の ASL を貼り付け ... }
JSON

aws stepfunctions create-state-machine \
  --name sf-codebuild-ecr-to-tar \
  --role-arn "arn:aws:iam::${ACCOUNT_ID}:role/sfn-starts-codebuild-role" \
  --definition file://sfn-definition.json \
  --type STANDARD \
  --tracing-configuration enabled=true \
  --region ap-northeast-1
```

## 4) 動作確認
```
aws stepfunctions start-execution \
  --state-machine arn:aws:states:ap-northeast-1:${ACCOUNT_ID}:stateMachine:sf-codebuild-ecr-to-tar \
  --input '{"artifactBucket":"'"$BUCKET"'","artifactKey":"images/nginx-mainline-alpine3.22.tar"}' \
  --region ap-northeast-1
```
- 成功後、s3://<BUCKET>/images/nginx-mainline-alpine3.22.tar が生成されます。
- 途中のログは CodeBuild の CloudWatch Logs と Step Functions 実行履歴 を確認。

## 5) 本番向けの注意点・ベストプラクティス
- 権限の最小化：ECR リソース ARN を可能なら特定リポジトリに限定。S3 もバケット・プレフィックスを限定。
- コスト・時間：毎回 docker pull するため、頻度が高い場合は CodeBuild ローカルキャッシュや ECR pull-through cache も検討。
- サイズ：tar が大きくなると転送時間が伸びます。aws s3 cp は自動でマルチパートですが、ネットワーク帯域に注意。
- VPC 内でインターネットなし運用の場合：上記の VPC エンドポイント一式（ECR API/DKR、S3、Logs、STS）を必ず用意。
- 暗号化：S3 側で KMS 暗号化（SSE-KMS）を使う場合、CodeBuild ロールに KMS 暗号化/復号権限が必要。
- 監視・通知：CodeBuild/Step Functions に CloudWatch Alarm を設定し失敗通知（SNS/ChatOps）。
- タグ運用：プロジェクト、バケット、ロールにタグを付与（コスト配賦/棚卸し向け）。

## 6) 代替アプローチ（必要に応じて）
- CodeBuild の artifacts 機能を使わず、今回のように aws s3 cp で直接アップロードするのがシンプルで大容量にも強いです（推奨）。
- Step Functions から直接 ECR/Docker 操作は不可。Docker 実行環境が必要なので CodeBuild（または ECS/Fargate）経由が妥当。
- ECS タスクで実施も可能（docker ではなく ctr/skopeo 等）ですが、Fargate だと docker save がそのままは使えません。シンプルさでは CodeBuild が優位。
