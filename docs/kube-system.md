# kube-system ネームスペース

Kubernetesのコアシステムコンポーネントが稼働しているネームスペースです。

## 稼働中のリソース

### Pod一覧

| Pod名 | タイプ | レプリカ | ステータス | 稼働期間 | 再起動回数 |
|------|--------|---------|-----------|---------|-----------|
| coredns-* | Deployment | 2/2 | Running | 12日 | 0 |
| etcd-rpi-master-1 | Static Pod | 1 | Running | 12日 | 1 |
| kube-apiserver-rpi-master-1 | Static Pod | 1 | Running | 12日 | 11 |
| kube-controller-manager-rpi-master-1 | Static Pod | 1 | Running | 12日 | 392 |
| kube-proxy-* | DaemonSet | 3/3 | Running | 12日 | 0 |
| kube-scheduler-rpi-master-1 | Static Pod | 1 | Running | 12日 | 6 |
| sealed-secrets-controller-* | Deployment | 1/1 | Running | 12日 | 0 |

## Kubernetesコントロールプレーン

### 1. etcd

#### 役割
- Kubernetesクラスタの全データストア
- 分散Key-Valueストア
- クラスタ状態の永続化

#### リソース詳細
- **タイプ**: Static Pod (マニフェスト: /etc/kubernetes/manifests/etcd.yaml)
- **ノード**: rpi-master-1
- **稼働期間**: 12日
- **再起動回数**: 1回

#### 参照マニフェスト
- なし (kubeadmによる自動生成、マスターノード上のStatic Pod)

#### 特徴
- マスターノードの起動時にkubeletが直接起動
- Kubernetes APIに依存しない
- バックアップとリストアが重要

### 2. kube-apiserver

#### 役割
- Kubernetes APIの提供
- 全てのクラスタ操作のエントリポイント
- 認証・認可の実行
- etcdとの通信管理

#### リソース詳細
- **タイプ**: Static Pod
- **ノード**: rpi-master-1
- **稼働期間**: 12日
- **再起動回数**: 11回

#### 参照マニフェスト
- なし (kubeadmによる自動生成)

#### アクセス
- **Service**: kubernetes (ClusterIP:10.96.0.1:443)
- **外部アクセス**: kubectl設定経由

### 3. kube-controller-manager

#### 役割
- 各種コントローラの実行
- Deployment、ReplicaSet、StatefulSetなどの管理
- ノード監視とヘルスチェック
- ServiceAccountとトークン管理

#### リソース詳細
- **タイプ**: Static Pod
- **ノード**: rpi-master-1
- **稼働期間**: 12日
- **再起動回数**: 392回 (注意: 高頻度)

#### 参照マニフェスト
- なし (kubeadmによる自動生成)

#### 注意事項
再起動回数が多いため、以下を調査推奨:
- リソース不足の可能性
- 設定の問題
- ノードの安定性

### 4. kube-scheduler

#### 役割
- Podの配置先ノード決定
- リソース要求とノード容量のマッチング
- アフィニティ/アンチアフィニティルールの適用

#### リソース詳細
- **タイプ**: Static Pod
- **ノード**: rpi-master-1
- **稼働期間**: 12日
- **再起動回数**: 6回

#### 参照マニフェスト
- なし (kubeadmによる自動生成)

---

## DNS (CoreDNS)

### 5. CoreDNS

#### 役割
- クラスタ内DNS解決
- Service名からClusterIPへの名前解決
- 外部DNS問い合わせのフォワーディング

#### 参照マニフェスト
- なし (kubeadmによるアドオンとしてインストール)

#### リソース詳細
- **タイプ**: Deployment
- **レプリカ数**: 2
- **稼働期間**: 12日

#### Service
- **名前**: kube-dns
- **ClusterIP**: 10.96.0.10
- **ポート**: 53/UDP, 53/TCP, 9153/TCP (metrics)

#### ConfigMap
- **coredns**: CoreDNS設定ファイル (Corefile)

#### DNS設定例
全てのPodは自動的に以下のDNS設定を持つ:
- **nameserver**: 10.96.0.10
- **search**: <namespace>.svc.cluster.local svc.cluster.local cluster.local

#### 名前解決例
```
postgres.household-task-manager.svc.cluster.local
→ 10.104.220.94 (PostgreSQL Service ClusterIP)

backend.household-task-manager
→ 10.110.47.199 (Backend Service ClusterIP)
```

---

## ネットワーク (kube-proxy)

### 6. kube-proxy

#### 役割
- Service IPへのトラフィックルーティング
- iptables/IPVSルールの管理
- ClusterIP、NodePort、LoadBalancerの実装

#### 参照マニフェスト
- なし (kubeadmによるアドオン)

#### リソース詳細
- **タイプ**: DaemonSet (全ノードで稼働)
- **Pod数**: 3 (master×1, worker×2)
- **稼働期間**: 12日

#### ConfigMap
- **kube-proxy**: プロキシ設定

#### モード
- iptables mode (デフォルト)

---

## シークレット管理 (Sealed Secrets)

### 7. Sealed Secrets Controller

#### 役割
- 暗号化されたシークレットの復号化
- SealedSecretリソースを通常のSecretに変換
- Gitリポジトリにシークレットを安全に保存

#### 参照マニフェスト
- なし (手動またはHelmでインストール)

#### リソース詳細
- **タイプ**: Deployment
- **レプリカ数**: 1
- **稼働期間**: 12日

#### Service
- **sealed-secrets-controller** (ClusterIP:10.96.240.246)
  - ポート: 8080 (HTTP)
- **sealed-secrets-controller-metrics** (ClusterIP:10.103.186.200)
  - ポート: 8081 (metrics)

#### Secret
- **sealed-secrets-keygs4s4**: コントローラの暗号化鍵ペア
  - タイプ: kubernetes.io/tls

#### 使用例
このクラスタでは以下のSealedSecretが使用されています:
- `base/database/sealed-secret.yaml` → `database-secret`
- `base/backend/sealed-secret.yaml` → `backend-secret`

#### kubesealコマンド
新しいSealedSecretを作成する場合:
```bash
kubectl create secret generic my-secret \
  --from-literal=key=value \
  --dry-run=client -o yaml | \
kubeseal -o yaml > my-sealed-secret.yaml
```

---

## ConfigMapとSecret

### ConfigMap
- **coredns**: CoreDNS設定ファイル
- **kube-proxy**: kube-proxy設定
- **kubeadm-config**: kubeadm初期化設定
- **kubelet-config**: kubelet設定
- **extension-apiserver-authentication**: API拡張認証設定
- **kube-apiserver-legacy-service-account-token-tracking**: ServiceAccountトークン追跡

### Secret
- **sealed-secrets-keygs4s4**: Sealed Secretsコントローラの暗号化鍵

---

## クラスタ情報ConfigMap

### cluster-info (kube-public namespace)

クラスタ検出用の情報が格納されています:
- CA証明書
- APIサーバーエンドポイント

---

## 監視とメトリクス

### 利用可能なメトリクスエンドポイント

1. **CoreDNS**: :9153/metrics
2. **Sealed Secrets Controller**: :8081/metrics

### 推奨監視項目

#### etcd
- etcd_server_has_leader
- etcd_disk_backend_commit_duration_seconds
- etcd_mvcc_db_total_size_in_bytes

#### kube-apiserver
- apiserver_request_duration_seconds
- apiserver_request_total
- process_resident_memory_bytes

#### kube-controller-manager
- 再起動回数の監視 (現在392回は異常)
- workqueue_depth
- process_cpu_seconds_total

#### CoreDNS
- coredns_dns_request_count_total
- coredns_dns_request_duration_seconds

---

## トラブルシューティング

### kube-controller-manager の高頻度再起動

**現象**: 392回の再起動

**調査コマンド**:
```bash
# SSH接続してStatic Podログ確認
ssh rpi-master-1
sudo crictl ps -a | grep kube-controller-manager
sudo crictl logs <container-id>

# または、マスターノード上で
sudo journalctl -u kubelet -n 100
```

**原因候補**:
- メモリ不足 (Raspberry Piの制約)
- リソースリミット設定
- etcd接続タイムアウト

### DNS解決できない

```bash
# CoreDNS Pod確認
kubectl get pod -n kube-system -l k8s-app=kube-dns

# DNS解決テスト
kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup kubernetes.default
```

### kube-proxy動作確認

```bash
# iptablesルール確認 (各ノードで)
sudo iptables -t nat -L -n | grep <service-name>

# kube-proxyログ確認
kubectl logs -n kube-system daemonset/kube-proxy
```

### Sealed Secrets復号化エラー

```bash
# コントローラログ確認
kubectl logs -n kube-system deployment/sealed-secrets-controller

# 公開鍵取得 (正常動作確認)
kubeseal --fetch-cert

# SealedSecret状態確認
kubectl get sealedsecret -n household-task-manager
kubectl describe sealedsecret <name> -n household-task-manager
```

---

## バックアップ推奨事項

### etcdバックアップ

```bash
# マスターノードで実行
sudo ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot.db \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

### Sealed Secrets鍵バックアップ

```bash
kubectl get secret -n kube-system sealed-secrets-keygs4s4 -o yaml > sealed-secrets-key-backup.yaml
```

**重要**: この鍵を失うと、既存のSealedSecretを復号化できなくなります。

---

## セキュリティ考慮事項

### 実装済み
- RBAC有効化
- ServiceAccountトークン自動マウント制御
- API Server TLS暗号化
- Sealed Secretsによる機密情報保護

### 推奨追加施策
- API Server監査ログ有効化
- PodSecurityPolicy/PodSecurityStandards適用
- NetworkPolicyによるトラフィック制御
- Secretの暗号化at rest (etcd暗号化)

---

## Static Podの場所

マスターノード (rpi-master-1) 上:
```
/etc/kubernetes/manifests/
├── etcd.yaml
├── kube-apiserver.yaml
├── kube-controller-manager.yaml
└── kube-scheduler.yaml
```

これらのファイルはkubeletが直接監視し、Pod起動を管理します。