# household-task-manager ネームスペース

アプリケーション本体が稼働しているネームスペースです。

## 稼働中のリソース

### Pod一覧

| Pod名 | タイプ | レプリカ | ステータス | 稼働期間 |
|------|--------|---------|-----------|---------|
| backend-55f6f59b67-* | Deployment | 2/2 | Running | 9日 |
| frontend-c48b45fd7-* | Deployment | 2/2 | Running | 9日 |
| postgres-0 | StatefulSet | 1/1 | Running | 12日 |

## 1. Database (PostgreSQL)

### 役割
- アプリケーションデータの永続化
- PostgreSQL 16 (Alpine版)による高速・軽量なデータベース

### 参照マニフェスト
- `base/database/statefulset.yaml` - StatefulSet定義
- `base/database/service.yaml` - Service定義
- `base/database/configmap.yaml` - 設定
- `base/database/sealed-secret.yaml` - 暗号化されたシークレット
- `base/database/pv.yaml` - PersistentVolume定義
- `base/database/pvc.yaml` - PersistentVolumeClaim定義

### 詳細設定

#### StatefulSet設定
```yaml
名前: postgres
レプリカ数: 1
イメージ: postgres:16-alpine
ポート: 5432
```

#### NodeSelector
- 特定ノードに固定: `rpi-worker-1`
- 理由: PersistentVolumeがローカルストレージのため

#### 環境変数ソース
- **ConfigMap** (`postgres-config`):
  - `POSTGRES_DB`: household_task_manager
  - `POSTGRES_USER`: postgres
- **Secret** (`database-secret`):
  - `POSTGRES_PASSWORD`: (暗号化)

#### ストレージ
- **PVC**: postgres-pvc
- **容量**: 10Gi
- **StorageClass**: local-path
- **PV**: postgres-pv-ssd (Bound状態)
- **マウントパス**: /var/lib/postgresql/data
- **サブパス**: postgres

#### リソース制限
- **Requests**: CPU 250m, Memory 256Mi
- **Limits**: CPU 500m, Memory 512Mi

#### ヘルスチェック
- **Liveness**: `pg_isready -U postgres` (30秒後から10秒間隔)
- **Readiness**: `pg_isready -U postgres` (5秒後から5秒間隔)

#### Service
- **タイプ**: ClusterIP
- **ポート**: 5432
- **ClusterIP**: 10.104.220.94

---

## 2. Backend (FastAPI)

### 役割
- RESTful APIの提供
- データベースマイグレーション管理
- ビジネスロジックの実装

### 参照マニフェスト
- `base/backend/deployment.yaml` - Deployment定義
- `base/backend/service.yaml` - Service定義
- `base/backend/configmap.yaml` - 設定
- `base/backend/sealed-secret.yaml` - 暗号化されたシークレット
- `overlays/production/patches/replicas.yaml` - レプリカ数オーバーライド (推定)

### 詳細設定

#### Deployment設定
```yaml
名前: backend
レプリカ数: 2
イメージ: ghcr.io/kamegrueon/household-task-manager-backend:main
イメージプルポリシー: Always
ポート: 8000
```

#### InitContainers
1. **wait-for-db**
   - イメージ: busybox:1.36
   - 役割: PostgreSQLが起動するまで待機
   - コマンド: `nc -z postgres 5432` を成功するまでループ

2. **run-migrations**
   - イメージ: ghcr.io/kamegrueon/household-task-manager-backend:main
   - 役割: Alembicを使用したDBマイグレーション実行
   - コマンド: `alembic upgrade head`
   - 環境変数: backend-config, backend-secret

#### 環境変数ソース
- **ConfigMap** (`backend-config`):
  - `ENVIRONMENT`: production
  - `CORS_ORIGINS`: https://household-task-mgr.duckdns.org
- **Secret** (`backend-secret`):
  - データベース接続情報など (暗号化)

#### リソース制限
- **Requests**: CPU 250m, Memory 256Mi
- **Limits**: CPU 500m, Memory 512Mi

#### ヘルスチェック
- **Liveness**: HTTP GET /health:8000 (30秒後から10秒間隔)
- **Readiness**: HTTP GET /readiness:8000 (10秒後から5秒間隔)

#### Service
- **タイプ**: ClusterIP
- **ポート**: 8000
- **ClusterIP**: 10.110.47.199

---

## 3. Frontend (React/Nginx)

### 役割
- Webユーザーインターフェースの提供
- 静的ファイルの配信
- バックエンドAPIへのプロキシ

### 参照マニフェスト
- `base/frontend/deployment.yaml` - Deployment定義
- `base/frontend/service.yaml` - Service定義
- `base/frontend/configmap.yaml` - 設定
- `overlays/production/ingress-frontend.yaml` - Ingress定義

### 詳細設定

#### Deployment設定
```yaml
名前: frontend
レプリカ数: 2
イメージ: ghcr.io/kamegrueon/household-task-manager-frontend:main
イメージプルポリシー: Always
ポート: 80
```

#### リソース制限
- **Requests**: CPU 100m, Memory 128Mi
- **Limits**: CPU 200m, Memory 256Mi

#### ヘルスチェック
- **Liveness**: HTTP GET /:80 (10秒後から10秒間隔)
- **Readiness**: HTTP GET /:80 (5秒後から5秒間隔)

#### Service
- **タイプ**: ClusterIP
- **ポート**: 80
- **ClusterIP**: 10.111.239.192

---

## 4. Ingress (外部公開)

### 役割
- HTTPSでのフロントエンドアプリケーション外部公開
- TLS証明書の自動管理
- SSL/TLSターミネーション

### 参照マニフェスト
- `overlays/production/ingress-frontend.yaml`

### 詳細設定

#### Ingress設定
```yaml
名前: household-task-manager-frontend
IngressClass: nginx
ホスト: household-task-mgr.duckdns.org
```

#### TLS設定
- **証明書発行**: cert-manager経由でLet's Encryptから自動取得
- **ClusterIssuer**: letsencrypt-prod
- **シークレット名**: household-task-manager-tls
- **証明書ステータス**: 有効 (12日前発行)

#### アノテーション
- `cert-manager.io/cluster-issuer: "letsencrypt-prod"`
- `nginx.ingress.kubernetes.io/ssl-redirect: "true"`
- `nginx.ingress.kubernetes.io/force-ssl-redirect: "true"`

#### ルーティング
- **パス**: / (Prefix)
- **バックエンドサービス**: frontend:80

#### 実際のIPアドレス
- **ADDRESS**: 192.168.0.202 (Ingress Controller NodePort経由)

---

## ネットワークフロー

```
インターネット
    │
    │ HTTPS (443)
    ▼
household-task-mgr.duckdns.org
    │
    │ Let's Encrypt TLS
    ▼
Ingress Controller (ingress-nginx)
    │
    │ HTTP (80)
    ▼
Frontend Service (ClusterIP:10.111.239.192:80)
    │
    ▼
Frontend Pods (2レプリカ) :80
    │
    │ API呼び出し
    ▼
Backend Service (ClusterIP:10.110.47.199:8000)
    │
    ▼
Backend Pods (2レプリカ) :8000
    │
    │ PostgreSQL接続
    ▼
PostgreSQL Service (ClusterIP:10.104.220.94:5432)
    │
    ▼
PostgreSQL StatefulSet (1レプリカ) :5432
    │
    ▼
PersistentVolume (10Gi, rpi-worker-1)
```

## デプロイメント戦略

### GitOps自動デプロイ
- **ArgoCD Application**: household-task-manager
- **同期ポリシー**:
  - 自動同期: 有効
  - 自動プルーン: 有効
  - 自動修復: 有効
- **リトライ設定**:
  - 最大試行回数: 5回
  - バックオフ: 5秒から3分まで指数関数的に増加

### イメージ更新
- **backend**: mainブランチへのpush時に自動ビルド・デプロイ
- **frontend**: mainブランチへのpush時に自動ビルド・デプロイ
- **イメージタグ**: main (常に最新)

### ローリングアップデート
- Deployment (frontend/backend) は標準のローリングアップデート戦略
- StatefulSet (postgres) は手動管理

## ConfigMapとSecret管理

### 平文ConfigMap
- `postgres-config`: データベース名、ユーザー名
- `backend-config`: 環境変数、CORS設定
- `frontend-config`: フロントエンド設定 (詳細確認必要)

### SealedSecret (暗号化)
- `database-secret`: データベースパスワード
- `backend-secret`: バックエンド機密情報

暗号化されたシークレットは、Sealed Secrets Controllerによってクラスタ内で復号化され、通常のSecretとして使用されます。

## ストレージ管理

### PersistentVolume
- **名前**: postgres-pv-ssd
- **容量**: 10Gi
- **アクセスモード**: ReadWriteOnce (RWO)
- **リクレームポリシー**: Retain
- **StorageClass**: local-path
- **ステータス**: Bound

### PersistentVolumeClaim
- **名前**: postgres-pvc
- **ネームスペース**: household-task-manager
- **要求容量**: 10Gi
- **ステータス**: Bound (postgres-pv-ssdにバインド)

### ストレージプロビジョナー
- **Provisioner**: local-path-storage (Rancher Local Path Provisioner)
- **保存場所**: rpi-worker-1ノードのローカルディスク

## 監視とヘルスチェック

### Liveness Probe (生存確認)
- **PostgreSQL**: pg_isready コマンド
- **Backend**: /health エンドポイント (HTTP GET)
- **Frontend**: / ルートパス (HTTP GET)

### Readiness Probe (準備完了確認)
- **PostgreSQL**: pg_isready コマンド (より早い間隔)
- **Backend**: /readiness エンドポイント (より早い間隔)
- **Frontend**: / ルートパス (より早い間隔)

### 推奨される追加監視
- メトリクス収集 (Prometheus)
- ログ集約 (Loki/Fluentd)
- トレーシング (Jaeger/Tempo)

## セキュリティ考慮事項

### 実装済み
- TLS/HTTPS通信 (Let's Encrypt証明書)
- SealedSecrets による機密情報の暗号化
- CORS設定による不正アクセス防止
- NetworkPolicy (未確認、要調査)

### バックエンドIngressの削除
- セキュリティ向上のため、バックエンドへの直接外部アクセスを削除
- フロントエンド経由のアクセスのみ許可

## トラブルシューティング

### Pod再起動が多い場合
```bash
kubectl logs -n household-task-manager <pod-name> --previous
kubectl describe pod -n household-task-manager <pod-name>
```

### データベース接続エラー
1. PostgreSQL Podのステータス確認
2. Service DNS解決確認: `kubectl exec -it <backend-pod> -- nslookup postgres`
3. ネットワークポリシー確認

### マイグレーションエラー
```bash
kubectl logs -n household-task-manager <backend-pod> -c run-migrations
```

### Ingress接続エラー
1. Ingress Controllerログ確認
2. 証明書ステータス確認: `kubectl get certificate -n household-task-manager`
3. DNS設定確認 (DuckDNS)