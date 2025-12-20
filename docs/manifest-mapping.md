# マニフェストファイルと実リソースの対応表

このドキュメントは、Gitリポジトリ内のマニフェストファイルと、実際にクラスタで稼働しているKubernetesリソースの対応関係を示します。

## 凡例

- ✅ マニフェストファイルあり、リソース稼働中
- ⚠️ マニフェストファイルなし、リソース稼働中 (手動作成/外部管理)
- 🔄 ArgoCD管理下
- 🔐 SealedSecret (暗号化)

---

## household-task-manager ネームスペース 🔄

| リソース種別 | リソース名 | マニフェストファイル | ステータス | 備考 |
|------------|-----------|-------------------|-----------|------|
| **Namespace** | household-task-manager | `base/namespace.yaml` | ✅ | |
| **StatefulSet** | postgres | `base/database/statefulset.yaml` | ✅ | |
| **Service** | postgres | `base/database/service.yaml` | ✅ | |
| **ConfigMap** | postgres-config | `base/database/configmap.yaml` | ✅ | |
| **SealedSecret** | database-secret | `base/database/sealed-secret.yaml` | ✅ 🔐 | |
| **PersistentVolume** | postgres-pv-ssd | `base/database/pv.yaml` | ✅ | |
| **PersistentVolumeClaim** | postgres-pvc | `base/database/pvc.yaml` | ✅ | |
| **Deployment** | backend | `base/backend/deployment.yaml` | ✅ | レプリカ数:2 |
| **Service** | backend | `base/backend/service.yaml` | ✅ | |
| **ConfigMap** | backend-config | `base/backend/configmap.yaml` | ✅ | |
| **SealedSecret** | backend-secret | `base/backend/sealed-secret.yaml` | ✅ 🔐 | |
| **Deployment** | frontend | `base/frontend/deployment.yaml` | ✅ | レプリカ数:2 |
| **Service** | frontend | `base/frontend/service.yaml` | ✅ | |
| **ConfigMap** | frontend-config | `base/frontend/configmap.yaml` | ✅ | |
| **Ingress** | household-task-manager-frontend | `overlays/production/ingress-frontend.yaml` | ✅ | TLS有効 |
| **Secret** | household-task-manager-tls | - | ⚠️ | cert-manager自動生成 |

### Kustomize構成

```
base/
├── namespace.yaml .......................... ✅ Namespace
├── kustomization.yaml ...................... Kustomize設定
├── database/
│   ├── configmap.yaml ...................... ✅ ConfigMap
│   ├── sealed-secret.yaml .................. ✅ SealedSecret
│   ├── pv.yaml ............................. ✅ PersistentVolume
│   ├── pvc.yaml ............................ ✅ PersistentVolumeClaim
│   ├── statefulset.yaml .................... ✅ StatefulSet
│   └── service.yaml ........................ ✅ Service
├── backend/
│   ├── configmap.yaml ...................... ✅ ConfigMap
│   ├── sealed-secret.yaml .................. ✅ SealedSecret
│   ├── deployment.yaml ..................... ✅ Deployment
│   └── service.yaml ........................ ✅ Service
└── frontend/
    ├── configmap.yaml ...................... ✅ ConfigMap
    ├── deployment.yaml ..................... ✅ Deployment
    └── service.yaml ........................ ✅ Service

overlays/production/
├── kustomization.yaml ...................... Kustomize overlay設定
├── ingress-frontend.yaml ................... ✅ Ingress
└── patches/
    └── replicas.yaml ....................... (推定) レプリカ数パッチ

argocd/
├── application.yaml ........................ ✅ ArgoCD Application
└── kustomization.yaml ...................... Kustomize設定
```

---

## argocd ネームスペース

| リソース種別 | リソース名 | マニフェストファイル | ステータス | 備考 |
|------------|-----------|-------------------|-----------|------|
| **Application** | household-task-manager | `argocd/application.yaml` | ✅ 🔄 | GitOps管理 |
| **Service** | argocd-server-tailscale | `argocd/tailscale/argocd-tailscale-service.yaml` | ✅ | LoadBalancer |
| **StatefulSet** | ts-argocd-server-tailscale-bxpzc | - | ⚠️ | Tailscale Operator管理 |
| **Deployment** | operator | - | ⚠️ | Tailscale Operator (Helm) |
| **StatefulSet** | argocd-application-controller | - | ⚠️ | ArgoCD公式マニフェスト |
| **Deployment** | argocd-applicationset-controller | - | ⚠️ | ArgoCD公式マニフェスト |
| **Deployment** | argocd-dex-server | - | ⚠️ | ArgoCD公式マニフェスト |
| **Deployment** | argocd-notifications-controller | - | ⚠️ | ArgoCD公式マニフェスト |
| **Deployment** | argocd-redis | - | ⚠️ | ArgoCD公式マニフェスト |
| **Deployment** | argocd-repo-server | - | ⚠️ | ArgoCD公式マニフェスト |
| **Deployment** | argocd-server | - | ⚠️ | ArgoCD公式マニフェスト |
| **Secret** | repo-household-task-manager-k8s | - | ⚠️ | ArgoCD Web UIで作成 |
| **Secret** | sh.helm.release.v1.tailscale-operator.v1 | - | ⚠️ | Helm管理 |

### インストール方法

- **ArgoCD本体**: 公式マニフェスト (`kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/...`)
- **Tailscale Operator**: Helm Chart (`helm install tailscale-operator ...`)
- **Tailscale Service**: リポジトリ内マニフェスト (`argocd/tailscale/argocd-tailscale-service.yaml`)

---

## cert-manager ネームスペース

| リソース種別 | リソース名 | マニフェストファイル | ステータス | 備考 |
|------------|-----------|-------------------|-----------|------|
| **Deployment** | cert-manager | - | ⚠️ | 公式マニフェストまたはHelm |
| **Deployment** | cert-manager-cainjector | - | ⚠️ | 公式マニフェストまたはHelm |
| **Deployment** | cert-manager-webhook | - | ⚠️ | 公式マニフェストまたはHelm |
| **ClusterIssuer** | letsencrypt-prod | - | ⚠️ | 手動作成 |
| **Secret** | letsencrypt-prod | - | ⚠️ | cert-manager自動生成 |

### インストール方法

```bash
# 推定される方法:
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.x.x/cert-manager.yaml

# または Helm:
helm install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace
```

### ClusterIssuer作成 (推定)

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: <your-email>
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
```

---

## ingress-nginx ネームスペース

| リソース種別 | リソース名 | マニフェストファイル | ステータス | 備考 |
|------------|-----------|-------------------|-----------|------|
| **Deployment** | ingress-nginx-controller | - | ⚠️ | 公式マニフェスト |
| **Job** | ingress-nginx-admission-create | - | ⚠️ | 公式マニフェスト (完了) |
| **Job** | ingress-nginx-admission-patch | - | ⚠️ | 公式マニフェスト (完了) |
| **Service** | ingress-nginx-controller | - | ⚠️ | NodePort |

### インストール方法

```bash
# 推定される方法:
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.x.x/deploy/static/provider/baremetal/deploy.yaml

# または Helm:
helm install ingress-nginx ingress-nginx/ingress-nginx --namespace ingress-nginx --create-namespace
```

---

## kube-system ネームスペース

| リソース種別 | リソース名 | マニフェストファイル | ステータス | 備考 |
|------------|-----------|-------------------|-----------|------|
| **Static Pod** | etcd-rpi-master-1 | - | ⚠️ | /etc/kubernetes/manifests/etcd.yaml |
| **Static Pod** | kube-apiserver-rpi-master-1 | - | ⚠️ | /etc/kubernetes/manifests/kube-apiserver.yaml |
| **Static Pod** | kube-controller-manager-rpi-master-1 | - | ⚠️ | /etc/kubernetes/manifests/kube-controller-manager.yaml |
| **Static Pod** | kube-scheduler-rpi-master-1 | - | ⚠️ | /etc/kubernetes/manifests/kube-scheduler.yaml |
| **Deployment** | coredns | - | ⚠️ | kubeadmアドオン |
| **DaemonSet** | kube-proxy | - | ⚠️ | kubeadmアドオン |
| **Deployment** | sealed-secrets-controller | - | ⚠️ | 公式マニフェストまたはHelm |
| **Secret** | sealed-secrets-keygs4s4 | - | ⚠️ | Controller自動生成 |

### インストール方法

- **Static Pods**: `kubeadm init` で自動生成
- **CoreDNS, kube-proxy**: kubeadm初期化時にアドオンとして自動インストール
- **Sealed Secrets**: 手動インストール

```bash
# Sealed Secrets インストール (推定):
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.x.x/controller.yaml
```

---

## kube-flannel ネームスペース

| リソース種別 | リソース名 | マニフェストファイル | ステータス | 備考 |
|------------|-----------|-------------------|-----------|------|
| **DaemonSet** | kube-flannel-ds | - | ⚠️ | 公式マニフェスト |
| **ConfigMap** | kube-flannel-cfg | - | ⚠️ | 公式マニフェスト |

### インストール方法

```bash
# 推定される方法:
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

---

## local-path-storage ネームスペース

| リソース種別 | リソース名 | マニフェストファイル | ステータス | 備考 |
|------------|-----------|-------------------|-----------|------|
| **Deployment** | local-path-provisioner | - | ⚠️ | Rancher公式マニフェスト |
| **ConfigMap** | local-path-config | - | ⚠️ | Rancher公式マニフェスト |

### インストール方法

```bash
# 推定される方法:
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml
```

---

## リポジトリ内ファイル一覧

### アプリケーションマニフェスト (ArgoCD管理) 🔄

```
household-task-manager-k8s/
├── base/
│   ├── namespace.yaml
│   ├── kustomization.yaml
│   ├── database/
│   │   ├── configmap.yaml
│   │   ├── sealed-secret.yaml
│   │   ├── database-secret.yaml.template (テンプレート)
│   │   ├── pv.yaml
│   │   ├── pvc.yaml
│   │   ├── statefulset.yaml
│   │   └── service.yaml
│   ├── backend/
│   │   ├── configmap.yaml
│   │   ├── sealed-secret.yaml
│   │   ├── backend-secret.yaml.template (テンプレート)
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   └── frontend/
│       ├── configmap.yaml
│       ├── deployment.yaml
│       └── service.yaml
├── overlays/
│   └── production/
│       ├── kustomization.yaml
│       ├── ingress-frontend.yaml
│       └── patches/
│           └── replicas.yaml (推定)
└── argocd/
    ├── application.yaml
    ├── kustomization.yaml
    └── tailscale/
        └── argocd-tailscale-service.yaml
```

### テンプレートファイル

- `base/backend/backend-secret.yaml.template`
- `base/database/database-secret.yaml.template`

これらはSealedSecretを作成するためのテンプレートです。

### SealedSecretの作成方法

```bash
# 1. テンプレートから平文Secretを作成
cp base/database/database-secret.yaml.template base/database/database-secret.yaml
# 値を編集

# 2. kubesealで暗号化
kubeseal -f base/database/database-secret.yaml -w base/database/sealed-secret.yaml

# 3. 平文ファイルを削除 (Gitにコミットしない!)
rm base/database/database-secret.yaml

# 4. SealedSecretをコミット
git add base/database/sealed-secret.yaml
git commit -m "Update sealed secret"
```

---

## 外部管理リソース (リポジトリ外)

以下のリソースはこのリポジトリ外で管理されています:

### インフラストラクチャ
- ArgoCD (公式マニフェストで手動インストール)
- cert-manager (公式マニフェストまたはHelmで手動インストール)
- ingress-nginx (公式マニフェストまたはHelmで手動インストール)
- Flannel CNI (公式マニフェストで手動インストール)
- Local Path Provisioner (公式マニフェストで手動インストール)
- Sealed Secrets Controller (公式マニフェストで手動インストール)
- Tailscale Operator (Helmで手動インストール)

### Kubernetesコアコンポーネント
- etcd, kube-apiserver, kube-controller-manager, kube-scheduler (kubeadm管理)
- CoreDNS, kube-proxy (kubeadm管理)

### 手動作成リソース
- ClusterIssuer (letsencrypt-prod)
- ArgoCD Repository Secret (repo-household-task-manager-k8s)

---

## GitOpsフロー

```
開発者
  │
  │ 1. コード変更 & git push
  ▼
GitHub Actions (CI)
  │ 2. Dockerイメージビルド & プッシュ
  │    ghcr.io/kamegrueon/household-task-manager-{backend,frontend}:main
  ▼
GitHub Repository
(household-task-manager-k8s)
  │ 3. マニフェストファイル
  ▼
ArgoCD
  │ 4. 自動同期検出
  │    overlays/production/ を監視
  ▼
Kustomize Build
  │ 5. マニフェスト生成
  │    base/ + overlays/production/ をマージ
  ▼
Kubernetes API
  │ 6. リソース適用
  ▼
Kubernetesクラスタ
(household-task-manager namespace)
  │ 7. Pod更新 (ローリングアップデート)
  ▼
稼働中のアプリケーション
```

---

## 推奨される管理方針

### このリポジトリで管理すべき

- ✅ アプリケーションリソース (household-task-manager namespace)
- ✅ ArgoCD Application定義
- ✅ アプリケーション固有のIngress
- ✅ SealedSecret

### 別リポジトリまたは手動管理

- ⚠️ インフラストラクチャコンポーネント (ArgoCD, cert-manager, ingress-nginxなど)
  - 推奨: 別の "infrastructure" リポジトリで管理
  - または: Helmfile/Flux/ArgoCD App of Appsパターン

- ⚠️ ClusterIssuer、StorageClass などクラスタ全体の設定
  - 推奨: "cluster-config" リポジトリで管理

### ディレクトリ構成改善案 (将来的)

```
household-task-manager-k8s/
├── apps/
│   └── household-task-manager/
│       ├── base/
│       └── overlays/
├── infrastructure/  (新規追加推奨)
│   ├── argocd/
│   ├── cert-manager/
│   ├── ingress-nginx/
│   └── sealed-secrets/
└── cluster-config/  (新規追加推奨)
    ├── cluster-issuers/
    └── storage-classes/
```

---

## トラブルシューティング: リソース不一致

### クラスタのリソースがマニフェストと一致しない場合

```bash
# ArgoCD管理下のリソース差分確認
argocd app diff household-task-manager

# 手動でKustomize出力を確認
kubectl kustomize overlays/production/

# 実際のクラスタリソースと比較
kubectl get <resource-type> <resource-name> -n household-task-manager -o yaml
```

### ArgoCD同期エラー

```bash
# 同期ステータス確認
argocd app get household-task-manager

# 強制同期
argocd app sync household-task-manager --force

# Prune (削除されたリソースのクリーンアップ)
argocd app sync household-task-manager --prune
```

### SealedSecret復号化エラー

1. Sealed Secrets Controllerが稼働中か確認
2. 暗号化に使用した鍵が同じか確認 (クラスタ再構築時は鍵が変わる)
3. 必要に応じて再暗号化

```bash
# 鍵確認
kubeseal --fetch-cert

# 再暗号化
kubeseal -f secret.yaml -w sealed-secret.yaml
```