# Kubernetes リソース概要

このドキュメントは、Household Task Manager K8sクラスタで稼働している全てのKubernetesリソースの概要を記載しています。

## クラスタ情報

- **マスターノード**: rpi-master-1
- **ワーカーノード**: rpi-worker-1, rpi-worker-2 (推定)
- **Kubernetesバージョン**: v1.28以降
- **ネットワークCNI**: Flannel

## ネームスペース一覧

| ネームスペース | 用途 | 稼働開始 |
|--------------|------|---------|
| argocd | GitOps CD/継続的デリバリー | 12日前 |
| cert-manager | TLS証明書管理 | 12日前 |
| default | デフォルトネームスペース | 12日前 |
| household-task-manager | アプリケーション本体 | 12日前 |
| ingress-nginx | Ingressコントローラ | 12日前 |
| kube-flannel | ネットワークCNI | 12日前 |
| kube-node-lease | ノードリース管理 | 12日前 |
| kube-public | パブリック情報 | 12日前 |
| kube-system | Kubernetesシステムコンポーネント | 12日前 |
| local-path-storage | ローカルストレージプロビジョナー | 12日前 |

## アーキテクチャ概要

```
┌─────────────────────────────────────────────────────────┐
│                     外部アクセス                          │
└──────────────┬────────────────────┬─────────────────────┘
               │                    │
               │ HTTPS              │ Tailscale VPN
               │ (DuckDNS)          │
               ▼                    ▼
┌──────────────────────┐  ┌─────────────────────────┐
│  Ingress Controller  │  │ Tailscale LoadBalancer  │
│   (ingress-nginx)    │  │  (ArgoCD Access)        │
└──────────┬───────────┘  └────────┬────────────────┘
           │                       │
           │ cert-manager          │
           │ (Let's Encrypt)       │
           │                       │
           ▼                       ▼
┌──────────────────────┐  ┌─────────────────────────┐
│    Frontend (2)      │  │   ArgoCD Server         │
│   Nginx + React      │  │   (GitOps)              │
└──────────┬───────────┘  └─────────────────────────┘
           │
           │
           ▼
┌──────────────────────┐
│    Backend (2)       │
│   FastAPI/Python     │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│   PostgreSQL (1)     │
│   StatefulSet        │
│   (Persistent SSD)   │
└──────────────────────┘
```

## リソース統計

### コンポーネント別Pod数

| ネームスペース | Pod数 | レプリカ設定 |
|--------------|-------|-------------|
| household-task-manager | 5 | frontend:2, backend:2, postgres:1 |
| argocd | 9 | application-controller:1, server:1, dex:1, redis:1, repo-server:1, notifications:1, applicationset:1, operator:1, tailscale:1 |
| ingress-nginx | 3 | controller:1, jobs:2(completed) |
| cert-manager | 3 | cert-manager:1, cainjector:1, webhook:1 |
| kube-system | 8 | coredns:2, control-plane:4, kube-proxy:3, sealed-secrets:1 |
| kube-flannel | 3 | flannel-ds:3(DaemonSet) |
| local-path-storage | 1 | provisioner:1 |

### ストレージ

- **PersistentVolume**: 1個 (postgres-pv-ssd, 10Gi, local-path)
- **PersistentVolumeClaim**: 1個 (postgres-pvc, 10Gi, Bound)

### ネットワーク

- **Service**: 21個
- **Ingress**: 1個 (household-task-manager-frontend)
- **LoadBalancer**: 1個 (argocd-server-tailscale, Tailscale経由)

### セキュリティ

- **SealedSecret**: 2個 (backend-secret, database-secret)
- **Secret**: 15個
- **ClusterIssuer**: 1個 (letsencrypt-prod)
- **Certificate**: TLS証明書自動発行・更新

## GitOps管理

ArgoCD Applicationによって以下のリソースが管理されています:

- **Application名**: household-task-manager
- **リポジトリ**: https://github.com/Kamegrueon/household-task-manager-k8s.git
- **同期ポリシー**: 自動同期、自動プルーン、自動修復
- **デプロイパス**: overlays/production
- **ステータス**: Synced & Healthy

## 参照マニフェストファイル

このクラスタのリソースは、以下のGitリポジトリのマニフェストファイルによって定義されています:

- **base/**: 基本リソース定義 (Kustomize)
- **overlays/production/**: 本番環境設定 (Kustomize overlay)
- **argocd/**: ArgoCD Application定義

詳細は各ネームスペース別のドキュメントを参照してください。