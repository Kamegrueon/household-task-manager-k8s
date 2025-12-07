# Household Task Manager - Kubernetes Manifests

Raspberry Pi k8sクラスタ用のKubernetesマニフェスト

## 📁 ディレクトリ構成

```
.
├── argocd/              # Argo CD Application定義
├── base/                # 基本マニフェスト
│   ├── backend/        # バックエンド (FastAPI)
│   ├── database/       # PostgreSQL
│   └── frontend/       # フロントエンド (React + Nginx)
└── overlays/
    └── production/     # 本番環境用オーバーレイ
```

## 🚀 デプロイ前の準備

### 1. 必要なツールのインストール

#### cert-manager (Let's Encrypt用)
```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml
```

#### ClusterIssuer作成
```bash
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-email@example.com  # 変更してください
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
