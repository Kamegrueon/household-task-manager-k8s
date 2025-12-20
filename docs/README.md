# Household Task Manager K8s - ドキュメント

このディレクトリには、Household Task Manager Kubernetesクラスタの包括的なドキュメントが含まれています。

## 📚 ドキュメント一覧

### 1. [概要 (overview.md)](./overview.md)

クラスタ全体の概要を把握するための入門ドキュメントです。

**内容:**
- クラスタ情報 (ノード構成、バージョン)
- ネームスペース一覧
- アーキテクチャ概要図
- リソース統計
- GitOps管理の概要

**こんな時に読む:**
- 初めてこのクラスタに触れる時
- クラスタ全体の構成を把握したい時
- 新メンバーへのオンボーディング

---

### 2. [アプリケーション (household-task-manager.md)](./household-task-manager.md)

メインアプリケーションの詳細ドキュメントです。

**内容:**
- Frontend (React/Nginx) の詳細設定
- Backend (FastAPI/Python) の詳細設定
- Database (PostgreSQL) の詳細設定
- Ingress設定とTLS証明書
- ネットワークフロー
- デプロイメント戦略
- トラブルシューティング

**こんな時に読む:**
- アプリケーションのデプロイ方法を知りたい時
- Pod/Serviceの設定を確認したい時
- アプリケーションが動かない時のデバッグ
- ConfigMap/Secretの内容を確認したい時

---

### 3. [ArgoCD (argocd.md)](./argocd.md)

GitOpsツールArgoCDの詳細ドキュメントです。

**内容:**
- ArgoCD各コンポーネントの役割
- Tailscale VPN統合
- Application設定と同期ポリシー
- アクセス方法
- GitOpsワークフロー
- 運用コマンド

**こんな時に読む:**
- ArgoCDにアクセスしたい時
- 自動デプロイの仕組みを理解したい時
- 同期エラーをトラブルシューティングする時
- Tailscale経由でアクセスする時

---

### 4. [Kubernetesシステムコンポーネント (kube-system.md)](./kube-system.md)

Kubernetesのコアシステムの詳細ドキュメントです。

**内容:**
- コントロールプレーン (etcd, kube-apiserver, scheduler, controller-manager)
- DNS (CoreDNS)
- ネットワーク (kube-proxy)
- Sealed Secrets Controller
- バックアップ推奨事項

**こんな時に読む:**
- クラスタの基盤を理解したい時
- DNS解決の問題をデバッグする時
- Sealed Secretsの仕組みを知りたい時
- etcdのバックアップ方法を確認したい時

---

### 5. [インフラストラクチャ (infrastructure.md)](./infrastructure.md)

アプリケーションを支えるインフラコンポーネントの詳細ドキュメントです。

**内容:**
- cert-manager (TLS証明書管理)
- ingress-nginx (外部公開)
- Flannel (ネットワークCNI)
- Local Path Provisioner (ストレージ)
- ネットワーク全体図

**こんな時に読む:**
- TLS証明書の発行/更新を理解したい時
- Ingressの設定を変更したい時
- Pod間通信の仕組みを知りたい時
- ストレージの仕組みを理解したい時

---

### 6. [マニフェストファイル対応表 (manifest-mapping.md)](./manifest-mapping.md)

Gitリポジトリ内のマニフェストファイルと実際のリソースの対応関係を示すドキュメントです。

**内容:**
- 各リソースの参照マニフェストファイル
- Kustomize構成
- 外部管理リソースのインストール方法
- GitOpsフロー
- SealedSecretの作成方法

**こんな時に読む:**
- どのマニフェストファイルがどのリソースを作成するか知りたい時
- 新しいリソースを追加する方法を確認したい時
- SealedSecretを作成/更新したい時
- クラスタを再構築する時

---

## 🗂️ ドキュメント構成図

```
docs/
├── README.md ........................... このファイル (ドキュメントガイド)
├── overview.md ......................... クラスタ全体概要
├── household-task-manager.md ........... アプリケーション詳細
├── argocd.md ........................... ArgoCD詳細
├── kube-system.md ...................... システムコンポーネント詳細
├── infrastructure.md ................... インフラコンポーネント詳細
└── manifest-mapping.md ................. マニフェスト対応表
```

---

## 🚀 クイックスタート

### 新メンバー向け

1. **[overview.md](./overview.md)** でクラスタ全体を理解
2. **[household-task-manager.md](./household-task-manager.md)** でアプリケーションを理解
3. **[argocd.md](./argocd.md)** でデプロイフローを理解
4. **[manifest-mapping.md](./manifest-mapping.md)** でマニフェストとリソースの対応を確認

### トラブルシューティング時

#### アプリケーションが動かない
→ **[household-task-manager.md](./household-task-manager.md)** のトラブルシューティングセクション

#### ArgoCD同期エラー
→ **[argocd.md](./argocd.md)** の運用コマンドとトラブルシューティング

#### TLS証明書エラー
→ **[infrastructure.md](./infrastructure.md)** のcert-managerセクション

#### DNS解決エラー
→ **[kube-system.md](./kube-system.md)** のCoreDNSセクション

#### 外部アクセス不可
→ **[infrastructure.md](./infrastructure.md)** のingress-nginxセクション

---

## 📋 ネームスペース別リソース参照

| ネームスペース | 主要ドキュメント | 副ドキュメント |
|--------------|----------------|--------------|
| household-task-manager | [household-task-manager.md](./household-task-manager.md) | [manifest-mapping.md](./manifest-mapping.md) |
| argocd | [argocd.md](./argocd.md) | [overview.md](./overview.md) |
| cert-manager | [infrastructure.md](./infrastructure.md) | - |
| ingress-nginx | [infrastructure.md](./infrastructure.md) | - |
| kube-system | [kube-system.md](./kube-system.md) | - |
| kube-flannel | [infrastructure.md](./infrastructure.md) | - |
| local-path-storage | [infrastructure.md](./infrastructure.md) | - |

---

## 🔍 検索ガイド

### リソース名から探す

| リソース名 | ドキュメント | セクション |
|----------|------------|----------|
| postgres | [household-task-manager.md](./household-task-manager.md) | Database |
| backend | [household-task-manager.md](./household-task-manager.md) | Backend |
| frontend | [household-task-manager.md](./household-task-manager.md) | Frontend |
| household-task-manager-frontend (Ingress) | [household-task-manager.md](./household-task-manager.md) | Ingress |
| argocd-server | [argocd.md](./argocd.md) | ArgoCD Server |
| argocd-server-tailscale | [argocd.md](./argocd.md) | Tailscale統合 |
| cert-manager | [infrastructure.md](./infrastructure.md) | cert-manager |
| ingress-nginx-controller | [infrastructure.md](./infrastructure.md) | ingress-nginx |
| coredns | [kube-system.md](./kube-system.md) | DNS |
| sealed-secrets-controller | [kube-system.md](./kube-system.md) | Sealed Secrets |
| kube-flannel-ds | [infrastructure.md](./infrastructure.md) | Flannel |
| local-path-provisioner | [infrastructure.md](./infrastructure.md) | Local Path Storage |

### 機能から探す

| 機能 | ドキュメント | セクション |
|------|------------|----------|
| アプリケーションデプロイ | [household-task-manager.md](./household-task-manager.md) | デプロイメント戦略 |
| GitOps自動デプロイ | [argocd.md](./argocd.md) | GitOpsワークフロー |
| TLS証明書自動発行 | [infrastructure.md](./infrastructure.md) | cert-manager |
| 外部HTTPS公開 | [household-task-manager.md](./household-task-manager.md) | Ingress |
| データベース永続化 | [household-task-manager.md](./household-task-manager.md) | Database, ストレージ管理 |
| シークレット管理 | [kube-system.md](./kube-system.md) | Sealed Secrets |
| VPNアクセス (ArgoCD) | [argocd.md](./argocd.md) | Tailscale統合 |
| DNS名前解決 | [kube-system.md](./kube-system.md) | CoreDNS |
| ネットワーク通信 | [infrastructure.md](./infrastructure.md) | Flannel |
| ローカルストレージ | [infrastructure.md](./infrastructure.md) | Local Path Provisioner |

---

## 🛠️ よく使うコマンド

### クラスタ情報確認

```bash
# 全ネームスペースのリソース確認
kubectl get all --all-namespaces

# 特定ネームスペースのリソース確認
kubectl get all -n household-task-manager

# Ingressとcertificate確認
kubectl get ingress,certificate -n household-task-manager
```

### ArgoCD操作

```bash
# Application状態確認
argocd app get household-task-manager

# 手動同期
argocd app sync household-task-manager

# ログ確認
kubectl logs -n argocd deployment/argocd-server -f
```

### トラブルシューティング

```bash
# Pod状態確認
kubectl get pod -n household-task-manager

# Podログ確認
kubectl logs -n household-task-manager <pod-name> -f

# Pod詳細確認
kubectl describe pod -n household-task-manager <pod-name>

# Serviceエンドポイント確認
kubectl get endpoints -n household-task-manager

# Ingress状態確認
kubectl describe ingress household-task-manager-frontend -n household-task-manager
```

---

## 📝 ドキュメントメンテナンス

### ドキュメント作成日

- 2025-12-20 (自動生成)

### ドキュメント更新方針

以下の変更時にドキュメントを更新してください:

1. **新しいリソース追加時**
   - [manifest-mapping.md](./manifest-mapping.md) にマニフェスト対応を追加
   - 該当ネームスペースのドキュメントに詳細を追加

2. **アーキテクチャ変更時**
   - [overview.md](./overview.md) のアーキテクチャ図を更新
   - 影響するコンポーネントのドキュメントを更新

3. **設定変更時**
   - 該当リソースのドキュメントを更新
   - [manifest-mapping.md](./manifest-mapping.md) のマニフェストパスを確認

4. **トラブルシューティング知見蓄積時**
   - 各ドキュメントのトラブルシューティングセクションに追記

---

## 📞 サポート

### 質問・相談

- **クラスタ管理**: [overview.md](./overview.md)参照
- **アプリケーション**: [household-task-manager.md](./household-task-manager.md)参照
- **GitOps**: [argocd.md](./argocd.md)参照

### ドキュメント改善提案

このドキュメントは継続的に改善されるべきです。以下の場合は更新をお願いします:

- 実際の設定と異なる記述を発見した場合
- 新しいトラブルシューティング方法を見つけた場合
- 理解しにくい部分がある場合

---

## 🎯 ドキュメントの目的

このドキュメント群は以下の目的で作成されています:

1. **透明性**: クラスタで何が動いているか、誰でも把握できる
2. **再現性**: クラスタを再構築する際の手順が明確
3. **トラブルシューティング**: 問題発生時に迅速に対処できる
4. **知識共有**: チームメンバー間で知識を共有できる
5. **オンボーディング**: 新メンバーがすぐに理解できる

---

## 📊 ドキュメントカバレッジ

| カテゴリ | カバー率 | 備考 |
|---------|---------|------|
| アプリケーションリソース | 100% | 全リソース文書化済み |
| ArgoCD | 100% | Tailscale統合含む |
| Kubernetesコアコンポーネント | 100% | kube-system全体 |
| インフラコンポーネント | 100% | cert-manager, ingress-nginx, flannel, local-path |
| マニフェスト対応 | 100% | リポジトリ内全マニフェスト |
| トラブルシューティング | 80% | 各コンポーネントに基本的な手順記載 |
| 運用手順 | 70% | 主要な運用コマンド記載、詳細手順は今後追加予定 |

---

**Happy Kubernetes! 🚢**
