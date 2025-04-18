# Helm + Kind + Argo CD GitOps チュートリアル（ECR 対応・最小構成）

> **目的** — Lightsail/EC2 上の Kind クラスタに AWS ECR のイメージを Pull し、Helm でデプロイしたアプリを **Argo CD** により GitOps で自動管理する構成を、実用最小限で構築します。

---

## ✅ 本チュートリアルの前提

- Lightsail または EC2 上の Ubuntu に VSCode Remote SSH で接続している
- Docker / Kind / kubectl / helm / AWS CLI がインストール済み
- GitHub リポジトリを所有しており、Helm Chart を管理可能

---

## 0️⃣ 事前チェック（30秒で完了）

| ✔︎ | チェック項目 | コマンド | 説明 |
|----|---------------|----------|------|
|    | AWS CLI       | `aws sts get-caller-identity` | IAM 認証確認 |
|    | Docker        | `docker info | grep "Default Runtime"` | rootless ではないこと |
|    | Kind           | `kind version` ≥ 0.23        | containerd 認証設定対応バージョン |
|    | kubectl       | `kubectl version --client`   | v1.30.x 以上推奨 |
|    | GitHub Token  | `echo $GH_TOKEN`             | Argo CD の repo 認証に使用 |
|    | ポート競合なし| `lsof -i:8080`               | Argo CD UI 用ポートが空いている |

---

## 🧠 Argo CD の Web UI をローカルブラウザから使う方法

### ✅ VSCode Remote SSH の場合（推奨構成）

- VSCode Remote SSH 経由で EC2 に接続し、VSCode ターミナルで以下を実行するだけ：

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

- その状態でローカルPCのブラウザから `https://localhost:8080` にアクセスすれば、Argo CD UI に接続できます。
- VSCode が裏で自動的に SSH トンネルを張ってくれているため、**追加の設定は不要です**。

### ✅ ターミナルのみの場合（ssh -L を明示的に使う）

```bash
ssh -i ~/.ssh/your-key.pem -L 8080:localhost:8080 ubuntu@<EC2 Public IP>
```

- これでローカルの 8080 ポートが EC2 内部の 8080 にトンネル接続されます。
- EC2 側では次のコマンドを実行：

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

その上で、ブラウザから `https://localhost:8080` へアクセスしてください。

---

## 1️⃣ プロジェクト初期化

```bash
mkdir -p ~/dev/k8s-kind-ubuntu-lightsail-api-03-argocd-ecr
cd ~/dev/k8s-kind-ubuntu-lightsail-api-03-argocd-ecr
```

---

## 2️⃣ Kind クラスタ作成（ECR Pull 対応）

```bash
export REGION=ap-northeast-1
export ACCOUNT_ID=986154984217
export ECR_REPO=container-nodejs-api-8000
export ECR_TOKEN=$(aws ecr get-login-password --region $REGION)

cat <<EOF > kind-cluster.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
containerdConfigPatches:
  - |-
    [plugins."io.containerd.grpc.v1.cri".registry.auths."${ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com"]
      username = "AWS"
      password = "${ECR_TOKEN}"
EOF

kind create cluster --config <(envsubst < kind-cluster.yaml)
```

---

## 3️⃣ Argo CD のインストールと初期化

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl get pods -n argocd
```

Argo CD の UI にアクセスするには：

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

ログイン用パスワード取得：

```bash
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

---

## 4️⃣ Helm Chart の雛形作成

```bash
helm create my-app-chart
```

`values.yaml` を以下のように編集：

```yaml
# container-nodejs-api-chart/values.yaml

replicaCount: 1

image:
  repository: 986154984217.dkr.ecr.ap-northeast-1.amazonaws.com/container-nodejs-api-8000
  tag: "latest"
  pullPolicy: Always

########################################
# Ingress (無効化)
########################################
ingress:
  enabled: false         # <= ここがポイント
  className: ""
  annotations: {}
  hosts: []
  tls: []

########################################
# ServiceAccount (無効化)
########################################
serviceAccount:
  create: false          # <= ここがポイント
  name: ""
  annotations: {}

########################################
# HPA / Autoscaling (無効化)
########################################
autoscaling:
  enabled: false         # <= ここがポイント
  minReplicas: 1
  maxReplicas: 3
  targetCPUUtilizationPercentage: 80

########################################
# Service設定
########################################
service:
  type: ClusterIP
  port: 8080

########################################
# コンテナポート
########################################
containerPort: 8080

########################################
# リソース設定 (例)
########################################
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 200m
    memory: 256Mi
```

---

## 5️⃣ Helm デプロイと動作確認

```bash
helm install container-api ./my-app-chart -f values.yaml
kubectl get pods
kubectl port-forward svc/container-api-my-app-chart -n default 8081:8000
```

→ `curl http://localhost:8081` でアプリ動作確認

---

## 6️⃣ Argo CD に Helm Chart を登録（GitOps）

```yaml
# argocd-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: container-api
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/kurosawa-kuro/k8s-kind-ubuntu-lightsail-api-03-argocd-ecr.git
    targetRevision: HEAD
    path: my-app-chart
    helm:
      valueFiles:
        - values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      selfHeal: true
      prune: true
```

```bash
kubectl apply -f argocd-app.yaml
```

Argo CD ダッシュボードにアプリが表示されていれば成功です。

---

## 7️⃣ GitOps による自動更新テスト

### レプリカ数を変更：
```bash
sed -i 's/replicaCount: 1/replicaCount: 2/' my-app-chart/values.yaml
git commit -am "scale: replicaCount 2"
git push origin main
```

### イメージタグを変更して再デプロイ：
```bash
TAG=v2
docker build -t $ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com/$ECR_REPO:$TAG .
docker push $ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com/$ECR_REPO:$TAG

# values.yaml を更新
sed -i "s/tag: .*/tag: $TAG/" my-app-chart/values.yaml
git commit -am "update image tag to $TAG"
git push origin main
```

---

## 🔚 完了と次のステップ

- GitHub Actions による CI → ECR 自動 Push
- Argo CD アプリのマルチ環境管理（dev / prod）
- Ingress 有効化、HPA 実装、ApplicationSet 利用

お疲れさまでした 🎉

