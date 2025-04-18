# Kubernetes環境構築チュートリアル

このチュートリアルでは、以下の内容を学びます：

1. Kindを使用したローカルKubernetesクラスターの構築
2. Helmを使用したアプリケーションのデプロイ
3. ArgoCDを使用したGitOps実践
4. AWS ECRとの連携

## 前提条件

- Docker
- kubectl
- Helm
- AWS CLI
- Kind
- GitHubアカウントとリポジトリ

## 1. Kindクラスターの構築

### 1.1 クラスター設定ファイルの作成

```bash
cat <<EOF > kind-cluster.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        system-reserved: memory=2Gi
        eviction-hard: memory.available<500Mi
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
  - containerPort: 30000
    hostPort: 30000
    protocol: TCP
EOF
```

### 1.2 クラスターの作成

```bash
kind create cluster --config kind-cluster.yaml
```

### 1.3 クラスターの確認

```bash
kubectl cluster-info
```

## 2. Ingressコントローラーのセットアップ

### 2.1 Ingress-nginxのインストール

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
```

### 2.2 Ingressコントローラーの準備完了を待つ

```bash
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s
```

## 3. AWS ECRとの連携

### 3.1 ECR認証情報の設定

```bash
# AWS認証情報の設定
aws configure

# ECRレジストリのログイン
aws ecr get-login-password --region ap-northeast-1 | docker login --username AWS --password-stdin 986154984217.dkr.ecr.ap-northeast-1.amazonaws.com

# Kubernetesシークレットの作成
kubectl create secret docker-registry ecr-secret \
  --docker-server=986154984217.dkr.ecr.ap-northeast-1.amazonaws.com \
  --docker-username=AWS \
  --docker-password=$(aws ecr get-login-password --region ap-northeast-1) \
  --namespace=default
```

## 4. Helmチャートの作成とデプロイ

### 4.1 Helmチャートの作成

```bash
# チャートの作成
helm create my-app-chart

# チャートディレクトリに移動
cd my-app-chart
```

### 4.2 values.yamlの編集

```bash
# values.yamlを編集
cat <<EOF > values.yaml
replicaCount: 1

image:
  repository: 986154984217.dkr.ecr.ap-northeast-1.amazonaws.com/container-nodejs-api-8000
  tag: latest
  pullPolicy: Always

nameOverride: ""
fullnameOverride: ""

service:
  type: ClusterIP
  port: 8080

resources:
  limits:
    cpu: 200m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi

imagePullSecrets:
  - name: ecr-secret
EOF
```

### 4.3 チャートのテンプレート確認

```bash
helm template . --values values.yaml
```

### 4.4 チャートのインストール

```bash
helm install container-api . --values values.yaml
```

### 4.5 デプロイの確認

```bash
kubectl get pods
kubectl get svc
```

## 5. ArgoCDのセットアップ

### 5.1 ArgoCDのインストール

```bash
# ArgoCD名前空間の作成
kubectl create namespace argocd

# ArgoCDのインストール
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# ArgoCDサーバーの準備完了を待つ
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=argocd-server -n argocd --timeout=300s
```

### 5.2 ArgoCDへのアクセス

```bash
# ArgoCDサーバーのポートフォワード
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

ブラウザで https://localhost:8080 にアクセスし、初期パスワードを取得：

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

### 5.3 GitHubリポジトリの準備

```bash
# 現在のディレクトリをGitリポジトリとして初期化
git init

# リモートリポジトリを追加（実際のGitHubリポジトリURLに置き換えてください）
git remote add origin https://github.com/yourusername/k8s-kind-ubuntu-lightsail-api-03-argocd-ecr.git

# 変更をコミット
git add .
git commit -m "Initial commit"

# リモートリポジトリにプッシュ
git push -u origin main
```

### 5.4 GitHub認証情報の設定

```bash
# GitHubのパーソナルアクセストークンを取得
# GitHubの設定 > Developer settings > Personal access tokens > Tokens (classic) > Generate new token
# 必要な権限: repo, workflow

# ArgoCDにGitHubリポジトリを追加
argocd repo add https://github.com/yourusername/k8s-kind-ubuntu-lightsail-api-03-argocd-ecr.git \
  --username yourusername \
  --password your-github-token
```

### 5.5 アプリケーションの登録

```bash
# ArgoCD CLIのインストール
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64

# ArgoCDにログイン
argocd login localhost:8080

# アプリケーションの登録（実際のGitHubリポジトリURLに置き換えてください）
argocd app create my-app \
  --repo https://github.com/yourusername/k8s-kind-ubuntu-lightsail-api-03-argocd-ecr.git \
  --path my-app-chart \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default
```

## 6. トラブルシューティング

### 6.1 イメージプルエラー

```bash
# イメージプルエラーの確認
kubectl describe pod <pod-name>

# ECRシークレットの確認
kubectl get secret ecr-secret -o yaml
```

### 6.2 Ingressエラー

```bash
# Ingressコントローラーのログ確認
kubectl logs -n ingress-nginx -l app.kubernetes.io/component=controller
```

### 6.3 ArgoCDエラー

```bash
# ArgoCDアプリケーションの状態確認
argocd app get my-app

# ArgoCDアプリケーションのログ確認
argocd app logs my-app

# GitHubリポジトリへのアクセスエラー
# 1. GitHubのパーソナルアクセストークンが正しいか確認
# 2. リポジトリが存在し、アクセス権限があるか確認
# 3. ArgoCDのリポジトリ設定を確認
argocd repo list
```

## 7. クリーンアップ

### 7.1 リソースの削除

```bash
# Helmリリースの削除
helm uninstall container-api

# ArgoCDアプリケーションの削除
argocd app delete my-app

# ArgoCDの削除
kubectl delete -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl delete namespace argocd

# Kindクラスターの削除
kind delete cluster
```

## 8. 参考リソース

- [Kind公式ドキュメント](https://kind.sigs.k8s.io/docs/)
- [Helm公式ドキュメント](https://helm.sh/docs/)
- [ArgoCD公式ドキュメント](https://argo-cd.readthedocs.io/)
- [AWS ECR公式ドキュメント](https://docs.aws.amazon.com/ecr/)
- [GitHub Personal Access Tokens](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token)

