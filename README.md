# Helm + Kind + Argo CD GitOps チュートリアル（ECR 対応・最小構成）

> **目的** — ローカル Kind クラスタに AWS ECR のイメージを Pull し、Helm でデプロイしたアプリを **Argo CD** で GitOps 管理する最短ルートを示します。

---

## 0️⃣ 30 秒チェックリスト

| ✔︎ | 項目 | コマンド例 | 説明 |
|---|---|---|---|
|   | AWS CLI   | `aws sts get-caller-identity` | IAM／認証情報 OK か |
|   | Docker    | `docker info | grep "Default Runtime"` | rootless だと Kind と競合 |
|   | Kind      | `kind version` → ≥ 0.23 | containerdConfigPatches が効く版 |
|   | kubectl   | `kubectl version --client --short` | Kind v1.30 相当を推奨 |
|   | Port 8080 | `lsof -i:8080` が空 | port‑forward 衝突防止 |
|   | Git PAT   | `echo $GH_TOKEN` | Argo CD が Git を clone 可能 |

> **❌ が 1 つでも** → 必ず解消してから進みましょう。

---

## 1️⃣ プロジェクト準備

```bash
mkdir -p ~/dev/k8s-kind-ubuntu-lightsail-api-03-argocd-ecr
cd       ~/dev/k8s-kind-ubuntu-lightsail-api-03-argocd-ecr
```

---

## 2️⃣ ECR イメージ

```
986154984217.dkr.ecr.ap-northeast-1.amazonaws.com/container-nodejs-api-8080:latest
```

> **備考** — 実運用では `:latest` は避け、CI で Git SHA など一意タグを付与することを推奨。

---

## 3️⃣ Kind クラスタ（ECR 認証付き）

```
helm uninstall container-api
kind delete cluster
```

```bash
export REGION=ap-northeast-1 ACCOUNT_ID=986154984217
export ECR_REPO=container-nodejs-api-8080
export ECR_TOKEN=$(aws ecr get-login-password --region $REGION)

cat <<EOF > kind-cluster.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    extraMounts: []
    kubeadmConfigPatches: []
containerdConfigPatches:
  - |-
    [plugins."io.containerd.grpc.v1.cri".registry.auths."${ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com"]
      username = "AWS"
      password = "${ECR_TOKEN}"
EOF

kind create cluster --config <(envsubst < kind-cluster.yaml)
```

---

## 4️⃣ Argo CD インストール

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# ローカル UI 用 port‑forward
kubectl port-forward svc/argocd-server -n argocd 8080:443 &

# 初期パスワード
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo
```

**ログイン URL** : <https://localhost:8080> （ユーザー名 `admin`）

> **Point** — ログイン後すぐに **SSH Key** を登録（Settings › Repositories）しておくと認証系トラブルが激減します。

---

## 5️⃣ Helm Chart 雛形作成

```bash
helm create my-app-chart
```

---

## 6️⃣ values.yaml を最小構成に

```yaml
# my-app-chart/values.yaml 抜粋
replicaCount: 1
image:
  repository: 986154984217.dkr.ecr.ap-northeast-1.amazonaws.com/container-nodejs-api-8080
  tag: latest
  pullPolicy: Always

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

> **テンプレート削除禁止** — 将来の拡張に備え、不要リソースは `enabled: false` で無効化します。

---

## 7️⃣ デプロイ前ドライラン & インストール

```bash
helm template container-api ./my-app-chart -f values.yaml | \
  kubeconform -strict   # optional: スキーマ検証

helm install container-api ./my-app-chart -f values.yaml
```

---

## 8️⃣ Argo CD アプリ登録（GitOps 化）

`argocd-app.yaml` をリポジトリ直下に追加:

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

```bash
kubectl apply -f argocd-app.yaml
```

> **確認** — Argo CD ダッシュボード上で `Healthy / Synced` 表示になること。

---

## 9️⃣ GitOps 実験① — replicaCount を 2 に

```yaml
# my-app-chart/values.yaml
replicaCount: 2   # ← 変更
```

```bash
git add my-app-chart/values.yaml
git commit -m "scale: replicaCount 2"
git push origin main
```

数秒後、Pod が 2 つに増えることを確認:

```bash
kubectl get pods
```

---

## 🔟 GitOps 実験② — イメージ Tag 更新

```bash
TAG=v2
# Build & Push
DOCKER_IMG=$ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com/$ECR_REPO:$TAG
docker build -t $DOCKER_IMG .
aws ecr get-login-password --region $REGION | \
  docker login --username AWS --password-stdin $ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com
docker push $DOCKER_IMG

# values.yaml
image:
  tag: v2

git add my-app-chart/values.yaml
git commit -m "feat: update image tag to v2"
git push origin main
```

Argo CD が検知 → 新タグが自動展開されます。

---

## 1️⃣1️⃣ クリーンアップ

```bash
helm uninstall container-api
kind delete cluster
```

---

## トラブルシューティング早見表

| 症状 | 原因 | 対処 |
|------|------|------|
| `ImagePullBackOff` | ECR 認証ミス | `docker exec kind-control-plane crictl config` で auth を確認 |
| Argo CD が repo を読めない | PAT 権限不足 / SSH 未設定 | `repo-server` Pod の log を確認 |
| `unknown field "containerPort"` | values キー typo | `helm show values` で公式と diff |
| port‑forward が即切れる | Pod CrashLoop | `kubectl logs` で原因特定 |

---

## Next Step

* GitHub Actions で **ECR Build → Tag Bump → Push** を自動化し、Argo CD と完全 Pull 型運用へ。
* [helm‑unittest](https://github.com/helm-unittest/helm-unittest) で Chart テストを CI に組み込み、仕様変更の回帰を防止。

---

© 2025 — kurosawa‑kuro

