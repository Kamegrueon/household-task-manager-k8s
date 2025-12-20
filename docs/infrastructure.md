# インフラストラクチャコンポーネント

アプリケーションを支えるインフラストラクチャ関連のネームスペースとリソースについて説明します。

---

## cert-manager ネームスペース

TLS証明書の自動発行・更新を担当するコンポーネントです。

### 稼働中のリソース

| Pod名 | タイプ | レプリカ | ステータス | 稼働期間 |
|------|--------|---------|-----------|---------|
| cert-manager-* | Deployment | 1/1 | Running | 12日 |
| cert-manager-cainjector-* | Deployment | 1/1 | Running | 12日 |
| cert-manager-webhook-* | Deployment | 1/1 | Running | 12日 |

### 1. cert-manager

#### 役割
- Certificate、Issuer、ClusterIssuerリソースの監視
- ACME (Let's Encrypt) プロトコルによる証明書取得
- 証明書の自動更新

#### 参照マニフェスト
- なし (Helmまたはマニフェストで手動インストール)

#### Service
- **cert-manager** (ClusterIP:10.99.180.249)
  - ポート: 9402 (metrics)

### 2. cert-manager-cainjector

#### 役割
- CA証明書の自動注入
- MutatingWebhookConfiguration、ValidatingWebhookConfigurationへのCA証明書挿入
- CustomResourceDefinitionへのCA証明書挿入

#### Service
- なし (内部処理のみ)

### 3. cert-manager-webhook

#### 役割
- Admission Webhook
- Certificateリソースのバリデーション
- リソース作成時の検証

#### Service
- **cert-manager-webhook** (ClusterIP:10.111.161.99)
  - ポート: 443

### ClusterIssuer: letsencrypt-prod

#### 役割
- Let's Encrypt本番環境からの証明書発行
- ACME HTTP-01/DNS-01チャレンジ

#### リソース詳細
- **名前**: letsencrypt-prod
- **ステータス**: Ready (True)
- **稼働期間**: 12日

#### 参照マニフェスト
- なし (手動作成またはHelmで管理)

#### 使用例
このクラスタで発行されている証明書:
- **household-task-manager-tls** (household-task-manager namespace)
  - ドメイン: household-task-mgr.duckdns.org
  - 有効期限: 90日間 (自動更新)

### Secret
- **cert-manager-webhook-ca**: Webhook用CA証明書
- **letsencrypt-prod**: ACME アカウント情報

### 証明書発行フロー

```
Ingress作成
 │ (cert-manager.io/cluster-issuer: letsencrypt-prod)
 ▼
cert-manager検出
 │
 ▼
Certificateリソース自動作成
 │
 ▼
ACME Challenge
 │ (HTTP-01: /.well-known/acme-challenge/)
 ▼
Let's Encrypt検証
 │
 ▼
証明書発行
 │
 ▼
Secretに保存
(household-task-manager-tls)
 │
 ▼
Ingressに自動適用
```

---

## ingress-nginx ネームスペース

外部からのHTTP/HTTPSトラフィックをクラスタ内のServiceにルーティングします。

### 稼働中のリソース

| Pod名 | タイプ | ステータス | 稼働期間 |
|------|--------|-----------|---------|
| ingress-nginx-controller-* | Deployment | Running | 12日 |
| ingress-nginx-admission-create | Job | Completed | 12日 |
| ingress-nginx-admission-patch | Job | Completed | 12日 |

### 1. Ingress Controller

#### 役割
- Ingressリソースの監視と設定反映
- Nginx設定の動的更新
- TLS/SSL終端処理
- バックエンドへのロードバランシング

#### 参照マニフェスト
- なし (公式マニフェストまたはHelmでインストール)

#### リソース詳細
- **タイプ**: Deployment
- **レプリカ数**: 1

#### Service
- **ingress-nginx-controller** (NodePort)
  - ClusterIP: 10.104.224.6
  - ポート: 80:30158/TCP, 443:30894/TCP
  - 外部アクセス: NodeのIP:30158 (HTTP), NodeのIP:30894 (HTTPS)

#### ConfigMap
- **ingress-nginx-controller**: Nginx設定パラメータ

#### 実際のアクセスフロー
```
インターネット (household-task-mgr.duckdns.org)
  │
  │ DNS解決 → グローバルIP
  ▼
ルーター (ポートフォワーディング)
  │ 80 → 192.168.0.202:30158
  │ 443 → 192.168.0.202:30894
  ▼
Ingress Controller (NodePort)
  │ TLS終端
  ▼
Frontend Service (ClusterIP)
  │
  ▼
Frontend Pods
```

### 2. Admission Webhook Jobs

#### ingress-nginx-admission-create
- **役割**: ValidatingWebhookConfigurationの作成
- **ステータス**: Completed
- **実行回数**: 1回成功

#### ingress-nginx-admission-patch
- **役割**: Webhook CA証明書のパッチ適用
- **ステータス**: Completed (1回再起動)
- **実行回数**: 1回成功

#### Secret
- **ingress-nginx-admission**: Webhook用証明書

### Ingress設定例

現在のクラスタで動作中:
```yaml
名前: household-task-manager-frontend
ホスト: household-task-mgr.duckdns.org
パス: / → frontend:80
TLS: household-task-manager-tls
```

---

## kube-flannel ネームスペース

クラスタネットワークCNI (Container Network Interface) プラグインです。

### 稼働中のリソース

| Pod名 | タイプ | レプリカ | ステータス | 稼働期間 |
|------|--------|---------|-----------|---------|
| kube-flannel-ds-* | DaemonSet | 3/3 | Running | 12日 |

### 1. Flannel DaemonSet

#### 役割
- オーバーレイネットワークの構築
- Pod間通信の実現
- ノード間ネットワークルーティング

#### 参照マニフェスト
- なし (公式マニフェストで手動インストール)

#### リソース詳細
- **タイプ**: DaemonSet (全ノードで稼働)
- **Pod数**: 3 (master×1, worker×2)

#### ConfigMap
- **kube-flannel-cfg**: Flannel設定
  - ネットワークレンジ
  - バックエンドタイプ (VXLAN/host-gw)

#### ネットワーク設定 (推定)
```
Pod CIDR: 10.244.0.0/16
  ├── rpi-master-1: 10.244.0.0/24
  ├── rpi-worker-1: 10.244.1.0/24
  └── rpi-worker-2: 10.244.2.0/24
```

#### 動作モード
おそらく **VXLAN** モード:
- オーバーレイネットワーク
- ノード間のL2トンネリング
- ファイアウォール越えに対応

---

## local-path-storage ネームスペース

ローカルストレージの動的プロビジョニングを提供します。

### 稼働中のリソース

| Pod名 | タイプ | レプリカ | ステータス | 稼働期間 |
|------|--------|---------|-----------|---------|
| local-path-provisioner-* | Deployment | 1/1 | Running | 12日 |

### 1. Local Path Provisioner

#### 役割
- PVCに対する動的PV作成
- ノードのローカルディスクを使用したストレージ提供
- hostPathベースのPersistentVolume管理

#### 参照マニフェスト
- なし (Rancher local-path-provisioner公式マニフェスト)

#### ConfigMap
- **local-path-config**: プロビジョナー設定
  - ストレージパス
  - ヘルパーイメージ
  - 削除ポリシー

#### StorageClass
- **名前**: local-path
- **プロビジョナー**: rancher.io/local-path
- **VolumeBindingMode**: WaitForFirstConsumer

#### 使用例
このクラスタで使用中:
- **postgres-pv-ssd** (10Gi)
  - バインド先: postgres-pvc
  - 保存場所: rpi-worker-1ノードのローカルディスク

#### 制約事項
- **ノード固定**: PVはノードローカルのため、Pod移動不可
- **バックアップ**: ノードレベルのバックアップが必要
- **高可用性**: レプリケーション機能なし

#### デフォルトストレージパス
通常は `/opt/local-path-provisioner/` 配下にボリュームが作成されます。

---

## ネットワーク全体図

```
┌─────────────────────────────────────────────────┐
│            外部インターネット                      │
│        household-task-mgr.duckdns.org           │
└────────────────┬────────────────────────────────┘
                 │
                 │ HTTPS (443)
                 ▼
┌─────────────────────────────────────────────────┐
│         ルーター (ポートフォワーディング)            │
│      443 → 192.168.0.202:30894                  │
└────────────────┬────────────────────────────────┘
                 │
                 │ ノードネットワーク (192.168.0.0/24)
                 ▼
┌─────────────────────────────────────────────────┐
│      Ingress Controller (NodePort:30894)        │
│         ingress-nginx namespace                 │
└────────────────┬────────────────────────────────┘
                 │
                 │ ClusterIP (10.96.0.0/12)
                 │ cert-manager (TLS証明書)
                 ▼
┌─────────────────────────────────────────────────┐
│       Frontend Service (10.111.239.192:80)      │
└────────────────┬────────────────────────────────┘
                 │
                 │ Pod Network (10.244.0.0/16)
                 │ Flannel (VXLAN)
                 ▼
┌─────────────────────────────────────────────────┐
│          Frontend Pods (2レプリカ)               │
│        10.244.x.x, 10.244.y.y                   │
└─────────────────────────────────────────────────┘
```

---

## トラブルシューティング

### cert-manager: 証明書発行失敗

```bash
# Certificate状態確認
kubectl get certificate -n household-task-manager

# 詳細確認
kubectl describe certificate household-task-manager-tls -n household-task-manager

# cert-managerログ確認
kubectl logs -n cert-manager deployment/cert-manager -f

# ChallengeやOrderリソース確認
kubectl get challenge,order -n household-task-manager
```

### Ingress: 外部アクセス不可

```bash
# Ingress状態確認
kubectl get ingress -n household-task-manager
kubectl describe ingress household-task-manager-frontend -n household-task-manager

# Ingress Controllerログ確認
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller -f

# NodePort確認
kubectl get svc -n ingress-nginx ingress-nginx-controller
```

### Flannel: Pod間通信エラー

```bash
# Flannel Pod状態確認
kubectl get pod -n kube-flannel

# Flannelログ確認
kubectl logs -n kube-flannel daemonset/kube-flannel-ds

# ノード上でルーティング確認
ssh rpi-worker-1
ip route
ip addr show flannel.1
```

### Local Path Provisioner: PVC Pending

```bash
# PVC状態確認
kubectl get pvc -n household-task-manager

# PV状態確認
kubectl get pv

# Provisionerログ確認
kubectl logs -n local-path-storage deployment/local-path-provisioner

# ノードのディスク容量確認
ssh rpi-worker-1
df -h /opt/local-path-provisioner
```

---

## メンテナンス

### cert-manager証明書更新

自動更新されますが、手動更新する場合:
```bash
kubectl delete certificate household-task-manager-tls -n household-task-manager
# Ingressに記載されているアノテーションで自動再作成されます
```

### Ingress Controller設定変更

```bash
# ConfigMap編集
kubectl edit configmap ingress-nginx-controller -n ingress-nginx

# または、アノテーションで個別設定
# Ingressリソースにアノテーション追加
```

### Flannel再起動

```bash
# DaemonSet再起動 (ローリング再起動)
kubectl rollout restart daemonset/kube-flannel-ds -n kube-flannel

# 特定ノードのみ
kubectl delete pod kube-flannel-ds-xxxxx -n kube-flannel
```

---

## セキュリティ考慮事項

### cert-manager
- ACME アカウント鍵の保護 (letsencrypt-prod Secret)
- Webhook TLS証明書の管理

### Ingress Controller
- レート制限設定
- ModSecurity WAF統合 (オプション)
- IP制限 (アノテーションで可能)

### Flannel
- ネットワークポリシー適用
- VXLAN暗号化 (IPsec連携可能)

### Local Path Provisioner
- hostPathのアクセス権限
- ノードレベルのディスク暗号化