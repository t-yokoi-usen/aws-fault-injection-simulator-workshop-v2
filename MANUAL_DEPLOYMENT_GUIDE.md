# AWS Fault Injection Simulator Workshop - 手動デプロイメントガイド

## 概要
CodeBuildの45分タイムアウト制限を回避するための手動デプロイメント手順です。

## 前提条件
- AWS CLI がインストール・設定済み
- Node.js 22.x がインストール済み
- Docker がインストール済み
- 適切なIAM権限を持つAWSアカウント

## 環境変数の設定

以下の環境変数を事前に設定してください：

```bash
# 基本設定
export AWS_DEFAULT_REGION="us-east-1"
export ASSET_BUCKET="fis-asset-1e22"
export GIT_REPO_URL="https://github.com/k-yokoi-usen/aws-fault-injection-simulator-workshop-v2.git"
export GIT_BRANCH="main"
export EE_TEAM_ROLE_ARN="arn:aws:iam::891623778107:role/admin_switch_role"
export IS_EVENT_ENGINE="false"

# Docker Hub認証情報（Secrets Managerから取得）
export DOCKERHUB_USER=$(aws secretsmanager get-secret-value --secret-id "arn:aws:secretsmanager:us-east-1:891623778107:secret:docker_hub_secret-ukh2pH" --query 'SecretString' --output text | jq -r '.DOCKER_HUB_USER')
export DOCKERHUB_PASS=$(aws secretsmanager get-secret-value --secret-id "arn:aws:secretsmanager:us-east-1:891623778107:secret:docker_hub_secret-ukh2pH" --query 'SecretString' --output text | jq -r '.DOCKER_HUB_PASSWORD')
```

## Phase 1: Install & Setup

### 1.1 Docker起動
```bash
# Dockerデーモンを起動（Linux環境の場合）
sudo nohup /usr/local/bin/dockerd --host=unix:///var/run/docker.sock --host=tcp://127.0.0.1:2375 --storage-driver=overlay2 &

# Dockerが起動するまで待機
timeout 15 sh -c "until docker info; do echo .; sleep 1; done"
```

### 1.2 必要なツールのインストール
```bash
# AWS CDKのインストール
npm install -g aws-cdk

# CDK Toolkitの確認
CDK_STACK=$(aws cloudformation list-stacks --query 'StackSummaries[?(StackName==`CDKToolkit` && StackStatus==`CREATE_COMPLETE`)].StackId' --output text)
echo "CDK Toolkit Status: $CDK_STACK"
```

## Phase 2: Pre-build

### 2.1 Docker Hub認証
```bash
# Docker Hubへのログイン
echo "Logging in to Docker Hub..."
echo $DOCKERHUB_PASS | docker login -u $DOCKERHUB_USER --password-stdin
```

### 2.2 リポジトリのクローン
```bash
# ワークショップリポジトリのクローン
if [ -z "$GIT_BRANCH" ]; then
    git clone --single-branch ${GIT_REPO_URL}
else
    git clone --branch ${GIT_BRANCH} --single-branch ${GIT_REPO_URL}
fi

cd aws-fault-injection-simulator-workshop-v2
```

## Phase 3: Build & Deploy

### 3.1 VSCode Server のデプロイ
```bash
cd supporting-services/vscode-server
npm install
npm run build

# CDK Bootstrap（必要に応じて）
../../scripts/cdkbootstrap.sh

# VSCode Server スタックのデプロイ
cdk deploy VscodeServerStack --require-approval never
```

### 3.2 Pet Adoptions メインデプロイメント
```bash
cd ../../PetAdoptions/cdk/pet_stack/
npm install
npm run build
mkdir -p ./out

# 各スタックを順次デプロイ（タイムアウトを避けるため）
echo "=== Deploying Services Stack ==="
cdk deploy Services --context admin_role="${EE_TEAM_ROLE_ARN}" --context is_event_engine="${IS_EVENT_ENGINE}" --require-approval=never --verbose -O ./out/out.json

echo "=== Deploying ServicesSecondary Stack ==="
cdk deploy ServicesSecondary --context admin_role="${EE_TEAM_ROLE_ARN}" --context is_event_engine="${IS_EVENT_ENGINE}" --require-approval=never --verbose -O ./out/out.json

echo "=== Deploying Network Peering ==="
cdk deploy NetworkRegionPeering --require-approval=never --verbose -O ./out/out.json

echo "=== Deploying Network Routes Main ==="
cdk deploy NetworkRoutesMain --require-approval=never --verbose -O ./out/out.json

echo "=== Deploying Network Routes Secondary ==="
cdk deploy NetworkRoutesSecondary --require-approval=never --verbose -O ./out/out.json

echo "=== Deploying S3 Replica ==="
cdk deploy S3Replica --require-approval=never --verbose -O ./out/out.json

echo "=== Deploying Applications ==="
cdk deploy Applications --require-approval=never --verbose -O ./out/out.json

echo "=== Deploying Applications Secondary ==="
cdk deploy ApplicationsSecondary --require-approval=never --verbose -O ./out/out.json

echo "=== Deploying FIS Serverless ==="
cdk deploy FisServerless --require-approval never --verbose -O ./out/out.json

echo "=== Deploying Observability ==="
cdk deploy Observability --require-approval never --verbose -O ./out/out.json

echo "=== Deploying Observability Secondary ==="
cdk deploy ObservabilitySecondary --require-approval never --verbose -O ./out/out.json

echo "=== Deploying User Simulation Stack ==="
cdk deploy UserSimulationStack --require-approval never --verbose -O ./out/out.json

# ユーザーシミュレーションタグの削除
sh ../../../az-experiment/scripts/removeusersimtags.sh

echo "=== Deploying User Simulation Stack Secondary ==="
cdk deploy UserSimulationStackSecondary --require-approval never --verbose -O ./out/out.json

echo "=== Deploying Observability Dashboard ==="
cdk deploy ObservabilityDashboard --require-approval never --verbose -O ./out/out.json
```

### 3.3 Intro Experiment のデプロイ
```bash
cd ../../../intro-experiment/cdk
npm install
cdk deploy --require-approval never --verbose -O ./out/out.json
```

## Phase 4: 検証

### 4.1 デプロイメント結果の確認
```bash
# CloudFormationスタックの確認
aws cloudformation list-stacks --stack-status-filter CREATE_COMPLETE UPDATE_COMPLETE --query 'StackSummaries[?contains(StackName, `Services`) || contains(StackName, `Observability`) || contains(StackName, `UserSimulation`)].{StackName:StackName,Status:StackStatus}'

# 出力ファイルの確認
ls -la ./aws-fault-injection-simulator-workshop-v2/PetAdoptions/cdk/pet_stack/out/
```

### 4.2 重要なリソースの確認
```bash
# EKSクラスターの確認
aws eks list-clusters --region us-east-1

# RDSクラスターの確認
aws rds describe-db-clusters --region us-east-1

# S3バケットの確認
aws s3 ls | grep petadoptions
```

## トラブルシューティング

### よくある問題と解決方法

1. **Docker認証エラー**
   ```bash
   # Docker Hubの認証情報を再確認
   docker logout
   echo $DOCKERHUB_PASS | docker login -u $DOCKERHUB_USER --password-stdin
   ```

2. **CDKデプロイエラー**
   ```bash
   # CDK Bootstrapの再実行
   cdk bootstrap aws://${AWS_ACCOUNT_ID}/${AWS_DEFAULT_REGION}
   ```

3. **権限エラー**
   ```bash
   # IAMロールの確認
   aws sts get-caller-identity
   aws iam get-role --role-name admin_switch_role
   ```

4. **リソース制限エラー**
   ```bash
   # サービス制限の確認
   aws service-quotas get-service-quota --service-code eks --quota-code L-1194D53C
   ```

## 推定実行時間
- Phase 1: 5-10分
- Phase 2: 5分
- Phase 3: 2-3時間（スタック数とリソース作成時間による）
- Phase 4: 5-10分

**合計: 約3-4時間**

## 注意事項
- 各フェーズでエラーが発生した場合は、ログを確認してから次に進んでください
- EKSクラスターの作成には特に時間がかかります（30-45分程度）
- リソース制限に達した場合は、AWSサポートに制限緩和を依頼してください