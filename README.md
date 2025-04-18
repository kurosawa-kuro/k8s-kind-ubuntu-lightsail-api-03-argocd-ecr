### Helm Chart 最小構成チュートリアル（Ingress/HPA/ServiceAccount 無効化 + GitOps対応）

このチュートリアルでは、**Helm** の標準雛形をベースに **Ingress/HPA/ServiceAccount** を「設定で無効化」しつつ、**Kubernetes** 上に **Deployment/Service** のみをデプロイします。  
**Kind** を使ってローカルクラスターを簡単に立ち上げ、AWS ECR に置いたイメージを Pull するところまで行い、さらに **GitOpsスタイル** のアップグレード例も示します。

> **ポイント**  
> - テンプレートファイルは削除/改変せず、**拡張しやすい構成**  
> - NodePort / Ingress 不要。**port-forward** による簡易アクセスのみ  
> - **Kind** で ECR のイメージを取得するための設定  
> - **GitOps** を意識したアップグレード手順を解説  

---

#### 目次

1. [事前準備](#1-事前準備)  
2. [Kind クラスターを作成](#2-kind-クラスターを作成)  
3. [Helm Chart 雛形を作成](#3-helm-chart-雛形を作成)  
4. [values.yaml で Ingress/HPA/ServiceAccount 無効化設定](#4-valuesyaml-で-ingresshpaserviceaccount-無効化設定)  
5. [テンプレートの確認 (helm template)](#5-テンプレートの確認-helm-template)  
6. [インストール (helm install)](#6-インストール-helm-install)  
7. [動作確認 (port-forward)](#7-動作確認-port-forward)  
8. [アップグレード例（replicaCount 変更・GitOps対応版）](#8-アップグレード例replicacount-変更gitops対応版)  
9. [アンインストール](#9-アンインストール)  
10. [まとめ](#10-まとめ)  
11. [ECR へのイメージPushと GitOps による Helm 自動アップデート](#11-ecr-へのイメージpushと-gitops-による-helm-自動アップデート)

---

### 1. 事前準備

- **Docker** がインストール済み & `docker ps` 実行可能  
- **Kind** (Kubernetes in Docker) がインストール済み  
- **Helm** がインストール済み  
- **AWS CLI** で `aws ecr get-login-password` が使える (ECRログイン可能)  
- `kubectl` で Kubernetes にアクセスできる状態  

---

### 2. Kind クラスターを作成

#### 2-1. AWS ECR へのログイン用トークンを取得

```bash
ECR_TOKEN=$(aws ecr get-login-password --region ap-northeast-1)
```

#### 2-2. Kind 用の設定ファイル (kind-cluster.yaml)

```bash
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
```

#### 2-3. Kind クラスター起動

```bash
# （不要ならクラスター削除）
kind delete cluster

kind create cluster --config kind-cluster.yaml
kind get clusters
# => "kind" と表示されればOK
```

これでローカル環境に Kind クラスターが立ち上がり、**ECRからのPull** に対応できます。

---

### 3. Helm Chart 雛形を作成

```bash
mkdir -p ~/dev/k8s-kind-helm-tutorial
cd ~/dev/k8s-kind-helm-tutorial

helm create my-app-chart
```

フォルダ構成：
```
my-app-chart/
├── Chart.yaml
├── values.yaml
├── charts/
├── templates/
│   ├── _helpers.tpl
│   ├── deployment.yaml
│   ├── hpa.yaml
│   ├── ingress.yaml
│   ├── service.yaml
│   ├── serviceaccount.yaml
│   └── tests/
└── .helmignore
```

---

### 4. values.yaml で Ingress/HPA/ServiceAccount 無効化設定

`my-app-chart/values.yaml` を以下のように書き換えます（抜粋）。  
特に `ingress.enabled: false` / `autoscaling.enabled: false` / `serviceAccount.create: false` が重要です。

```yaml
replicaCount: 1

image:
  repository: 986154984217.dkr.ecr.ap-northeast-1.amazonaws.com/container-nodejs-api-8080
  tag: "latest"
  pullPolicy: Always

containerPort: 8080

service:
  type: ClusterIP
  port: 8080

serviceAccount:
  create: false
  name: ""
  annotations: {}

ingress:
  enabled: false
  className: ""
  annotations: {}
  hosts: []
  tls: []

autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 3
  targetCPUUtilizationPercentage: 80

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 200m
    memory: 256Mi
```

---

### 5. テンプレートの確認 (helm template)

```bash
cd my-app-chart
helm template . --values values.yaml
```

- `enabled: false` などのフラグにより、`ingress.yaml` / `hpa.yaml` / `serviceaccount.yaml` は出力されません。  
- **Deployment と Service のみ** 出力されればOK。

---

### 6. インストール (helm install)

```bash
helm install container-api . --values values.yaml

# Pod/Serviceの確認
kubectl get pods
kubectl get svc
helm list
```

- Pod が `Running` & `1/1` になればデプロイ成功

---

### 7. 動作確認 (port-forward)

NodePort / Ingress を立てていないので、`kubectl port-forward` でアクセスします。

```bash
kubectl port-forward service/container-api-my-app-chart 8080:8080
# => "Forwarding from 127.0.0.1:8080 -> 8080" 等が表示

# 別のターミナルから
curl -v http://localhost:8080/
# => {"status":"ok", "timestamp":"..."} 等のレスポンスでOK
```

> Service名は `リリース名-チャート名` 形式 (例: `container-api-my-app-chart`) なので適宜ご確認を。

---

### 8. アップグレード例（replicaCount 変更・GitOps対応版）

#### 8-1. `values.yaml` を編集して replicaCount を変更

```yaml
# my-app-chart/values.yaml

replicaCount: 2
```

#### 8-2. Git にコミット & Push

```bash
cd ~/dev/k8s-kind-helm-tutorial/my-app-chart
git add values.yaml
git commit -m "Increase replicaCount to 2"
git push origin main
```

#### 8-3. Argo CD 等が自動で反映

- Argo CD がこのGit差分を検知して `helm upgrade` 相当を実行し、Pod が2つに増えます。  
- 確認：

```bash
kubectl get pods
# => Podが2つなら成功
```

> **学習目的で即時変更したい場合**は `helm upgrade container-api . --set replicaCount=2` としてもOK。  
> ただし本番運用を想定するなら **GitOps**（＝すべてGit管理）を推奨します。

---

### 9. アンインストール

```bash
helm uninstall container-api
```

- これで Deployment, Service 等のリソースが削除されます。

---

### 10. まとめ

- **Kind + Helm + AWS ECR** で最小構成のアプリをローカルで試せる  
- **Ingress/HPA/ServiceAccount** はテンプレートを削除せずに `enabled` / `create` フラグで無効化  
- NodePort / Ingress を使わず `port-forward` で動作確認  
- GitOps 前提であれば、**`helm upgrade` の代わりに `values.yaml` → GitPush** でクラスタを更新  
- 将来的に Ingress を有効化したり HPA を導入したりする際も、該当フラグを `true` にするだけで拡張可能  

---

おっしゃる通り、README に **Argo CD の導入手順** が記述されていないようですね。GitOps を前提とするチュートリアルであれば、本来**Argo CD のセットアップ手順**が明確に記載されているべきです。  
以下に、既存の流れと整合性を保ちつつ、**「章立て11. ECR へのイメージPushと GitOps による Helm 自動アップデート」**の前後に組み込むべき Argo CD 導入手順を提案いたします。

---

## ✅ 差し込み候補：10.5 Argo CD の導入とセットアップ（GitOps 実践前提）

```md
### 10.5 Argo CD の導入とセットアップ（GitOps 実践前提）

GitOps 運用の前提として、Argo CD を Kind クラスタに導入します。

#### ステップ 1：Argo CD のインストール（公式マニフェスト）

```bash
kubectl create namespace argocd

kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

#### ステップ 2：Argo CD サーバーへのアクセス（port-forward）

```bash
kubectl port-forward svc/argocd-server -n argocd 8083:443
```

ブラウザで `https://localhost:8083` を開きます（自己署名証明書の警告あり）。

#### ステップ 3：初期パスワードの取得

```bash
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d && echo
```

初期ユーザー名は `admin` です。

---

#### ステップ 4：Argo CD CLI（任意）

ローカル環境でも CLI を使いたい場合は、以下の通り：

```bash
VERSION=$(curl --silent "https://api.github.com/repos/argoproj/argo-cd/releases/latest" | grep '"tag_name":' | sed -E 's/.*"v([^"]+)".*/\1/')
curl -sSL -o argocd "https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64"
chmod +x argocd
sudo mv argocd /usr/local/bin/
```

---

#### ステップ 5：アプリケーション定義の登録

以下のようなマニフェストで GitHub 上の Helm Chart を Argo CD に登録します。

`argocd-app.yaml`（例）：

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

これで、Git に Push すれば Argo CD 経由で自動反映される GitOps サイクルが完成します。
```

---

### 🔁 README に追記すべき場所（ご提案）

| 章番号 | 現在の項目 | 追記提案 |
|--------|------------|-----------|
| `10`   | まとめ       | `10.5 Argo CD の導入` を `11 ECR Push〜GitOps` の前に差し込む |

---

もしご希望であれば、**Markdown形式でそのままコピペ可能なフルセクション**を作成し、PR用に整えた形でも提供できます。  
ご要望ありましたらお申し付けくださいませ 🙌

### 11. ECR へのイメージPushと GitOps による Helm 自動アップデート

1. **Dockerイメージをビルド**  
2. **ECR にPush**  
3. **`values.yaml` の `image.tag` を変更 → GitPush**  
4. **Argo CD（等）が自動反映**  

#### 11-1. 新しいタグを決めてビルド & Push

```bash
REPO="container-nodejs-api-8080"
REGION="ap-northeast-1"
ACCOUNT_ID="986154984217"
NEXT_TAG="v2" # お好みのタグ

docker build -t $ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com/$REPO:$NEXT_TAG .
aws ecr get-login-password --region $REGION | \
  docker login --username AWS --password-stdin $ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com
docker push $ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com/$REPO:$NEXT_TAG
```

#### 11-2. `values.yaml` を更新 → GitPush

```bash
sed -i "s/tag: .*/tag: \"$NEXT_TAG\"/" values.yaml

git add values.yaml
git commit -m "Update image tag to $NEXT_TAG"
git push origin main
```

#### 11-3. Argo CD が自動適用

- 最新イメージへと差し替えられた Pod が再起動し、アップデートが完了します。
- `kubectl get pods` などで稼働状況を確認できます。

