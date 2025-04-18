了解しました。  
以下に、**指定された流れに沿って構造と文脈を再構成した Helm + Argo CD + GitOps チュートリアル**を提示します。  
このまま `README.md` にコピペ可能な Markdown 形式です。GitHub 公開用にも適しています。

---

# Helm + Kind + Argo CD GitOps チュートリアル（最小構成・ECR対応）

このチュートリアルでは、以下を段階的に実施し、**GitOps による Kubernetes アプリケーションの運用**を習得します：

- Helm 標準テンプレートで最小構成アプリを作成
- Kind クラスタをローカルに構築し、ECR のイメージを Pull できるよう設定
- Argo CD を用いて GitOps による自動反映を確認

---

## 🔁 全体の流れ

1. ターゲットディレクトリ作成（Git管理前提）  
2. ECR イメージを決定  
3. Kind クラスタ作成（ECR Pull 設定付き）  
4. Argo CD 導入（GitOps準備）  
5. Helm Chart 作成（my-app-chart）  
6. `values.yaml` 設定 → Helm install（初回）  
7. Argo CD で Helm Chart を GitOps 化  
8. `replicaCount = 2` に変更 → Git Push → 自動反映確認  
9. ECR イメージ tag を更新 → Git Push → 自動反映確認  

---

## 1. ターゲットディレクトリ作成

```bash
mkdir -p ~/dev/k8s-kind-ubuntu-lightsail-api-03-argocd-ecr
cd ~/dev/k8s-kind-ubuntu-lightsail-api-03-argocd-ecr
```

---

## 2. ECR イメージの指定

使用するイメージ：

```
986154984217.dkr.ecr.ap-northeast-1.amazonaws.com/container-nodejs-api-8080:latest
```

---

## 3. Kind クラスタの作成（ECR対応）

```bash
ECR_TOKEN=$(aws ecr get-login-password --region ap-northeast-1)

cat <<EOF > kind-cluster.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
containerdConfigPatches:
  - |-
    [plugins."io.containerd.grpc.v1.cri".registry]
      [plugins."io.containerd.grpc.v1.cri".registry.auths."986154984217.dkr.ecr.ap-northeast-1.amazonaws.com"]
        username = "AWS"
        password = "$ECR_TOKEN"
EOF

kind create cluster --config kind-cluster.yaml
```

---

## 4. Argo CD の導入（GitOps準備）

```bash
kubectl create namespace argocd

kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl port-forward svc/argocd-server -n argocd 8083:443
```

初期パスワード取得：

```bash
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

ユーザー名は `admin`。ログイン後、パスワード変更を求められます。

---

## 5. Helm Chart の作成

```bash
helm create my-app-chart
```

---

## 6. `values.yaml` 編集 → Helm による初回デプロイ

以下の設定を `my-app-chart/values.yaml` に反映：

```yaml
replicaCount: 1
image:
  repository: 986154984217.dkr.ecr.ap-northeast-1.amazonaws.com/container-nodejs-api-8080
  tag: latest
  pullPolicy: Always

containerPort: 8080

service:
  type: ClusterIP
  port: 8080

ingress:
  enabled: false

autoscaling:
  enabled: false

serviceAccount:
  create: false
```

初回インストール：

```bash
cd my-app-chart
helm install container-api . --values values.yaml
```

---

## 7. Argo CD 連携設定（GitOps化）

`argocd-app.yaml` を作成：

```yaml
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

適用：

```bash
kubectl apply -f argocd-app.yaml
```

---

## 8. `replicaCount = 2` に変更 → Git Push

```yaml
# values.yaml を編集
replicaCount: 2
```

```bash
git add my-app-chart/values.yaml
git commit -m "Increase replicaCount to 2"
git push origin main
```

数秒以内に Argo CD が検知 → Pod が 2 に増加：

```bash
kubectl get pods
```

---

## 9. ECR の image tag を変更 → Git Push

```bash
# 新しいタグをビルド・Push
TAG="v2"
REPO="container-nodejs-api-8080"
REGION="ap-northeast-1"
ACCOUNT_ID="986154984217"

docker build -t $ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com/$REPO:$TAG .
aws ecr get-login-password --region $REGION | \
  docker login --username AWS --password-stdin $ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com
docker push $ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com/$REPO:$TAG
```

`values.yaml` を変更：

```yaml
tag: v2
```

```bash
git add my-app-chart/values.yaml
git commit -m "Update image tag to v2"
git push origin main
```

数秒後、Pod が再作成されて新しいバージョンへ更新されます。

---

## ✅ まとめ

| 項目 | 内容 |
|------|------|
| クラスタ管理 | Kind + ECR Pull 対応 |
| パッケージ管理 | Helm |
| GitOps | Argo CD による自動反映 |
| 構成 | Deployment + Service（最小構成） |
| テンプレート削除なし | 拡張性を重視し、`enabled: false` で制御 |
| 本番対応拡張 | Ingress/HPAなどは `values.yaml` で `true` にすれば導入可能 |

---

必要に応じて、以下も展開可能です：

- `GitHub Actions` で ECR ビルド＋Push 自動化
- Helm Chart を複数に分割（マイクロサービス対応）
- Argo CD アプリの複数管理

ご希望あれば次フェーズも一緒に設計いたします。  
この内容を `README.md` に反映したい場合は、PR用の差分も作成できますよ。
