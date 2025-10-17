# Vantiq CloudFormation Templates

このディレクトリには、TerraformからCloudFormationに変換されたVantiqインフラストラクチャのテンプレートが含まれています。

## ファイル構成

- `main.yaml` - Nested Stacksを使用するメインテンプレート
- `vpc.yaml` - VPCとネットワークリソース
- `eks.yaml` - EKSクラスターとマネージドノードグループ
- `rds.yaml` - PostgreSQL RDSインスタンス（Keycloak用）
- `bastion.yaml` - Bastionホスト（オプション）
- `eks-addon.yaml` - EKS EBS CSI Driverアドオン

## デプロイ方法

### 1. S3バケットの準備

Nested Stacksを使用するため、テンプレートをS3バケットにアップロードする必要があります：

```bash
# S3バケットを作成（バケット名は一意である必要があります）
aws s3 mb s3://your-cloudformation-templates-bucket

# テンプレートをアップロード
aws s3 cp vpc.yaml s3://your-cloudformation-templates-bucket/vantiq-cfn/
aws s3 cp eks.yaml s3://your-cloudformation-templates-bucket/vantiq-cfn/
aws s3 cp rds.yaml s3://your-cloudformation-templates-bucket/vantiq-cfn/
aws s3 cp bastion.yaml s3://your-cloudformation-templates-bucket/vantiq-cfn/
aws s3 cp eks-addon.yaml s3://your-cloudformation-templates-bucket/vantiq-cfn/
```

### 2. スタックのデプロイ

```bash
aws cloudformation create-stack \
  --stack-name vantiq-infrastructure \
  --template-body file://main.yaml \
  --parameters ParameterKey=TemplateS3Bucket,ParameterValue=your-cloudformation-templates-bucket \
               ParameterKey=ClusterName,ParameterValue=vantiq-cluster \
               ParameterKey=EnvName,ParameterValue=dev \
  --capabilities CAPABILITY_NAMED_IAM
```

### 3. パラメーターのカスタマイズ

主要なパラメーター（**すべて必須、デフォルト値なし**）：

- `ResourcePrefix`: 全リソース名のプレフィックス（vantiq-dev/vantiq-staging/vantiq-prod）
- `ClusterName`: EKSクラスター名
- `ClusterVersion`: EKSのバージョン
- `TemplateS3Bucket`: CloudFormationテンプレート格納S3バケット
- `TemplateS3Prefix`: S3内のテンプレートプレフィックス
- `DBPassword`: RDSパスワード（空の場合は自動生成）
- `EnableBastion`: Bastionホストを作成するか（true/false）
- `EnableEBSCSIDriver`: EBS CSI Driverアドオンを有効にするか（true/false）
- `WorkerAccessSSHKeyName`: ワーカーノード用SSHキー名
- `BastionAccessPublicKey`: Bastion用公開キー
- `WorkerAccessPrivateKey`: ワーカーノード用秘密キー

### 4. 環境別設定（Mappings）

プレフィックスごとの設定は`main.yaml`のMappingsセクションで定義：

- **vantiq-dev**: 開発環境用（小さいインスタンス、172.20.x.x）
- **vantiq-staging**: ステージング環境用（中程度、172.22.x.x） 
- **vantiq-prod**: 本番環境用（大きいインスタンス、172.21.x.x）

設定項目：
- VPC CIDR/サブネット設定
- インスタンスタイプ（EKS NodeGroup、RDS、Bastion）
- RDSストレージサイズ

**注意**: ResourcePrefixは上記のいずれかを指定する必要があります。

## 主な変更点（Terraformからの）

### 1. 固定3AZ構成
- TerraformのようなforEach動的構成から固定3AZ構成に変更
- AZ1、AZ2、AZ3で明示的にリソース定義

### 2. 6つのEKS NodeGroup対応
- VANTIQ、MongoDB、Keycloak、Grafana、Metrics、AI Assistantの6つのNodeGroup
- 各NodeGroupは異なるワークロード用途に最適化

### 3. リソース名統一
- ResourcePrefixパラメーターで全リソース名を統一管理
- 一貫性のあるネーミング規則

### 4. 環境別設定管理
- Mappingsセクションで環境（dev/staging/prod）別設定を定義
- VPC CIDR、インスタンスタイプ等を環境に応じて自動選択

### 5. パスワード管理
- RDSパスワードはSecretsManagerを使用した自動生成に対応
- 手動指定も可能

### 6. SSH キー管理
- KeyPairリソースでSSHキーを管理
- 公開キーの内容をパラメーターで指定

### 7. Nested Stacks
- モジュール間の依存関係を明確化
- 各コンポーネントを独立したスタックとして管理

### 8. Conditions
- オプショナルリソース（BastionやEKS Addon）の作成制御

## 注意事項

1. **S3バケット**: Nested StacksテンプレートをホストするS3バケットが必要
2. **IAMポリシー**: EKS作成に必要な権限が必要
3. **リージョン**: 使用するリージョンで3つ以上のAZが利用可能である必要
4. **バージョン互換性**: EKSとKubernetesのバージョン互換性を確認

## トラブルシューティング

- EKS作成失敗: IAM権限とVPC設定を確認
- Nested Stack失敗: S3テンプレートのアクセス権限を確認
- RDS作成失敗: サブネットグループとセキュリティグループ設定を確認