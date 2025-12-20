# argocd ネームスペース

GitOpsツールArgoCD関連のリソースが稼働しているネームスペースです。

## 稼働中のリソース

### Pod一覧

| Pod名 | タイプ | レプリカ | ステータス | 稼働期間 |
|------|--------|---------|-----------|---------|
| argocd-application-controller-0 | StatefulSet | 1/1 | Running | 12日 |
| argocd-applicationset-controller-* | Deployment | 1/1 | Running | 12日 |
| argocd-dex-server-* | Deployment | 1/1 | Running | 12日 |
| argocd-notifications-controller-* | Deployment | 1/1 | Running | 12日 |
| argocd-redis-* | Deployment | 1/1 | Running | 12日 |
| argocd-repo-server-* | Deployment | 1/1 | Running | 12日 |
| argocd-server-* | Deployment | 1/1 | Running | 12日 |
| operator-* | Deployment | 1/1 | Running | 9日 |
| ts-argocd-server-tailscale-bxpzc-0 | StatefulSet | 1/1 | Running | 9日 |

## ArgoCD コンポーネント

### 1. Application Controller

#### 役割
- Kubernetes リソースの状態を監視
- GitリポジトリとKubernetesクラスタの状態を比較
- 自動同期処理の実行

#### リソース詳細
- **タイプ**: StatefulSet
- **レプリカ数**: 1
- **稼働期間**: 12日

### 2. ArgoCD Server

#### 役割
- Web UI/APIの提供
- CLIインターフェースのサポート
- 認証・認可の管理

#### リソース詳細
- **タイプ**: Deployment
- **レプリカ数**: 1
- **ポート**: 8080 (HTTP), 8083 (metrics)

#### Service
- **argocd-server** (ClusterIP:10.99.90.119)
  - ポート: 80 (HTTP), 443 (HTTPS)
- **argocd-server-metrics** (ClusterIP:10.97.63.206)
  - ポート: 8083 (metrics)

### 3. Repository Server

#### 役割
- Gitリポジトリのクローンと監視
- Helmチャート、Kustomize、プレーンマニフェストの処理
- マニフェスト生成

#### リソース詳細
- **タイプ**: Deployment
- **レプリカ数**: 1
- **再起動回数**: 2回 (8日前)
- **ポート**: 8081, 8084

#### Service
- **argocd-repo-server** (ClusterIP:10.105.221.30)

### 4. Dex Server

#### 役割
- OIDC (OpenID Connect) 認証プロバイダー
- SSOインテグレーション
- 外部認証システムとの連携

#### リソース詳細
- **タイプ**: Deployment
- **レプリカ数**: 1

#### Service
- **argocd-dex-server** (ClusterIP:10.104.29.5)
  - ポート: 5556, 5557, 5558

### 5. Redis

#### 役割
- キャッシュストレージ
- セッション管理
- 一時データストア

#### リソース詳細
- **タイプ**: Deployment
- **レプリカ数**: 1

#### Service
- **argocd-redis** (ClusterIP:10.98.167.113)
  - ポート: 6379

#### Secret
- **argocd-redis**: Redis認証情報

### 6. Notifications Controller

#### 役割
- アプリケーション状態変化の通知
- Webhook、Slack、メールなどへの通知配信

#### リソース詳細
- **タイプ**: Deployment
- **レプリカ数**: 1

#### Service
- **argocd-notifications-controller-metrics** (ClusterIP:10.108.44.39)
  - ポート: 9001

### 7. ApplicationSet Controller

#### 役割
- 複数のApplicationを一括管理
- テンプレートベースのApplication生成
- モノレポ/マルチクラスタサポート

#### リソース詳細
- **タイプ**: Deployment
- **レプリカ数**: 1

#### Service
- **argocd-applicationset-controller** (ClusterIP:10.102.218.230)
  - ポート: 7000, 8080

---

## Tailscale統合

### 8. Tailscale Operator

#### 役割
- Tailscale VPN統合
- プライベートネットワークでのArgoCD Web UIアクセス
- セキュアなリモートアクセス

#### 参照マニフェスト
- `argocd/tailscale/argocd-tailscale-service.yaml`

#### リソース詳細
- **Deployment**: operator
- **StatefulSet**: ts-argocd-server-tailscale-bxpzc
- **稼働期間**: 9日

#### Service設定
```yaml
名前: argocd-server-tailscale
タイプ: LoadBalancer
LoadBalancerClass: tailscale
```

#### アノテーション
- `tailscale.com/expose: "true"`
- `tailscale.com/hostname: "argocd"`
- `tailscale.com/tags: "tag:k8s-operator"`

#### LoadBalancer情報
- **External IP**: 100.80.43.111
- **Tailscale Hostname**: argocd.taile0cf4b.ts.net
- **ポート**: 80:30574, 443:31757

#### 接続方法
Tailscaleネットワークに接続後、以下のURLでアクセス可能:
- `http://argocd.taile0cf4b.ts.net`
- `https://argocd.taile0cf4b.ts.net`
- `http://100.80.43.111`

#### Secret
- **operator**: Tailscale Operator設定
- **operator-oauth**: OAuth認証情報
- **ts-argocd-server-tailscale-bxpzc-0**: Tailscale認証情報

---

## ArgoCD Application

### household-task-manager Application

#### 参照マニフェスト
- `argocd/application.yaml`

#### 設定詳細
```yaml
名前: household-task-manager
プロジェクト: default
リポジトリ: https://github.com/Kamegrueon/household-task-manager-k8s.git
リビジョン: HEAD (mainブランチ)
パス: overlays/production
デプロイ先: household-task-manager namespace
```

#### 同期ポリシー
- **自動同期**: 有効
- **自動プルーン**: 有効 (Gitから削除されたリソースをクラスタからも削除)
- **自動修復**: 有効 (手動変更を自動的に元に戻す)
- **空のコミット許可**: 無効

#### 同期オプション
- **CreateNamespace**: true (ネームスペース自動作成)

#### リトライ設定
- **最大試行回数**: 5回
- **初期待機時間**: 5秒
- **バックオフ係数**: 2倍
- **最大待機時間**: 3分

#### ステータス
- **同期状態**: Synced
- **ヘルス状態**: Healthy

---

## ConfigMapとSecret

### ConfigMap
- **argocd-cm**: ArgoCD メイン設定 (9エントリ)
- **argocd-cmd-params-cm**: コマンドパラメータ
- **argocd-gpg-keys-cm**: GPG鍵管理
- **argocd-notifications-cm**: 通知設定
- **argocd-rbac-cm**: RBAC設定
- **argocd-ssh-known-hosts-cm**: SSH既知ホスト
- **argocd-tls-certs-cm**: TLS証明書

### Secret
- **argocd-initial-admin-secret**: 初期管理者パスワード
- **argocd-notifications-secret**: 通知用シークレット
- **argocd-redis**: Redis認証情報
- **argocd-secret**: ArgoCD内部シークレット (5エントリ)
- **repo-household-task-manager-k8s**: Gitリポジトリアクセストークン (3エントリ)

---

## アクセス方法

### 1. Tailscale経由 (推奨・設定済み)

Tailscaleネットワークに接続している場合:
```
https://argocd.taile0cf4b.ts.net
```

**メリット**:
- インターネットに公開しない
- VPN経由の安全なアクセス
- パブリックIPアドレス不要

### 2. kubectl port-forward

ローカル開発環境から:
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

アクセス: `https://localhost:8080`

### 3. 初期パスワード取得

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

**ユーザー名**: admin

---

## GitOps ワークフロー

```
開発者
  │
  │ git push
  ▼
GitHub Repository
(household-task-manager-k8s)
  │
  │ Webhook (optional)
  ▼
ArgoCD Repository Server
  │ クローン & マニフェスト生成
  ▼
ArgoCD Application Controller
  │ 状態比較 & 差分検出
  ▼
Kubernetes API Server
  │ リソース作成・更新
  ▼
Kubernetesクラスタ
(household-task-manager namespace)
```

### 同期プロセス
1. Repository ServerがGitリポジトリを定期的にポーリング
2. 変更検出時、Kustomizeでマニフェスト生成
3. Application Controllerが現在のクラスタ状態と比較
4. 差分があれば自動的にkubectl applyを実行
5. 同期完了後、ヘルスチェック実行

---

## 運用コマンド

### Applicationの状態確認
```bash
kubectl get application -n argocd
argocd app get household-task-manager
```

### 手動同期
```bash
argocd app sync household-task-manager
```

### ログ確認
```bash
# Application Controller
kubectl logs -n argocd argocd-application-controller-0

# Repository Server
kubectl logs -n argocd deployment/argocd-repo-server

# Server (API/UI)
kubectl logs -n argocd deployment/argocd-server
```

### リポジトリ接続確認
```bash
argocd repo list
```

---

## セキュリティ設定

### 実装済み
- Tailscale VPN経由のアクセス制限
- パブリックインターネット非公開
- Secret管理 (Gitリポジトリトークンなど)
- RBAC設定 (argocd-rbac-cm)

### 推奨事項
- 初期管理者パスワード変更
- SSO/OIDC統合 (Dex Server経由)
- Multi-factor Authentication (MFA)
- 監査ログ有効化

---

## Helm Chart情報

Tailscale OperatorはHelm Chartでインストールされています:

- **リリース名**: tailscale-operator
- **Secret**: sh.helm.release.v1.tailscale-operator.v1
- **稼働期間**: 9日

---

## トラブルシューティング

### Applicationが同期されない
```bash
# アプリケーション詳細確認
argocd app get household-task-manager

# 同期履歴確認
argocd app history household-task-manager

# リソース差分確認
argocd app diff household-task-manager
```

### Repository Serverエラー
```bash
# ログ確認
kubectl logs -n argocd deployment/argocd-repo-server -f

# リポジトリ接続テスト
argocd repo get https://github.com/Kamegrueon/household-task-manager-k8s.git
```

### Tailscaleアクセス不可
```bash
# Tailscale Pod状態確認
kubectl get pod -n argocd -l app=operator

# LoadBalancer状態確認
kubectl get svc argocd-server-tailscale -n argocd

# Tailscale ログ確認
kubectl logs -n argocd sts/ts-argocd-server-tailscale-bxpzc
```

### Redisエラー
```bash
# Redis Pod確認
kubectl get pod -n argocd -l app.kubernetes.io/name=argocd-redis

# Redis接続テスト
kubectl exec -it -n argocd deployment/argocd-redis -- redis-cli ping
```