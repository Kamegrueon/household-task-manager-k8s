# household-task-manager-k8s

これは、家庭内タスク管理アプリケーション「Household Task Manager」をKubernetes上にデプロイするためのマニフェストリポジトリです。

このリポジトリは、[Household Task Manager](https://github.com/Kamegrueon/household-task-manager)をKubernetesクラスタにデプロイするための設定ファイル群を管理します。

アプリケーションは以下の3つのコンポーネントで構成されています。
- **frontend**: Webフロントエンド (React, Vue, etc.)
- **backend**: APIサーバー (Go, Node.js, etc.)
- **database**: データベース (PostgreSQL, MySQL, etc.)

デプロイには[Argo CD](https://argo-cd.readthedocs.io/en/stable/)を用いたGitOpsワークフローを採用しています。

## ディレクトリ構成

```
.
├── argocd/               # Argo CD関連の設定
│   ├── application.yaml  # Argo CD Applicationマニフェスト
│   └── ...
├── base/                 # 環境共通の基本マニフェスト
│   ├── backend/          # BackendのDeployment, Serviceなど
│   ├── database/         # DatabaseのStatefulSet, PVCなど
│   └── frontend/         # FrontendのDeployment, Serviceなど
└── overlays/             # 環境別の設定オーバーレイ
    └── production/       # 本番環境向けの設定
        ├── ingress-frontend.yaml
        └── patches/
```

- **`base/`**: 全ての環境で共通となる基本的なマニフェストを配置します。Argo CDはこれをベースとして利用します。
- **`overlays/`**: `production`や`staging`など、環境ごとの差分（パッチ）を設定します。例えば、レプリカ数やIngressの設定などをここで上書きます。
- **`argocd/`**: このアプリケーション自体をArgo CDで管理するための`Application`マニフェストを配置します。

## 🚀 デプロイ方法

### 🔧 初回セットアップ（ブートストラップ）

クラスタを初めて構築する際は、以下の手順で手動デプロイを行います。

#### 前提条件
- Kubernetesクラスタが稼働中
- ArgoCD, cert-manager, ingress-nginx, Sealed Secrets Controller がインストール済み
- kubeseal CLI がローカルにインストール済み
- SSH秘密鍵（GitHubアクセス用）が手元にある

#### 手順

1. **ClusterIssuer のデプロイ**
   ```bash
   kubectl apply -k cluster-resources/
   kubectl wait --for=condition=Ready clusterissuer/letsencrypt-prod --timeout=60s
   ```

2. **ArgoCD Repository Secret の作成**
   ```bash
   # 現在のSecretを取得（既存環境から）
   ssh rpi-master-1 'kubectl get secret -n argocd repo-household-task-manager-k8s -o yaml' > /tmp/current-secret.yaml

   # または、新規作成する場合
   cp argocd/repo-secret/repo-secret.yaml.template argocd/repo-secret/repo-secret.yaml
   # SSH秘密鍵をBase64エンコードして repo-secret.yaml に設定
   cat ~/.ssh/id_rsa | base64 -w 0  # この値を sshPrivateKey に設定

   # SealedSecret化
   kubeseal -f argocd/repo-secret/repo-secret.yaml \
            -w argocd/repo-secret/sealed-secret.yaml \
            --controller-namespace kube-system

   # 平文ファイル削除
   rm argocd/repo-secret/repo-secret.yaml

   # SealedSecretをGitにコミット
   git add argocd/repo-secret/sealed-secret.yaml
   git commit -m "Add ArgoCD repository SealedSecret"
   git push

   # デプロイ
   kubectl apply -f argocd/repo-secret/sealed-secret.yaml
   kubectl wait --for=condition=Synced sealedsecret/repo-household-task-manager-k8s \
     -n argocd --timeout=60s
   ```

3. **ArgoCD Application のデプロイ**
   ```bash
   kubectl apply -f argocd/application.yaml
   kubectl wait --for=jsonpath='{.status.sync.status}'=Synced \
     application/household-task-manager -n argocd --timeout=300s
   ```

4. **動作確認**
   ```bash
   # 全Podの起動を確認
   kubectl get pods -n household-task-manager

   # TLS証明書の発行を確認
   kubectl get certificate -n household-task-manager

   # 外部アクセステスト
   curl -I https://household-task-mgr.duckdns.org
   ```

### 📝 通常運用時のデプロイ

初回セットアップ後の通常運用では、GitにコミットするだけでArgo CDが自動的にデプロイします。

#### 前提条件

- 初回セットアップが完了していること
- Kubernetesクラスタへのアクセスが可能であること (`kubectl`)
- [Argo CD](https://argo-cd.readthedocs.io/en/stable/)がクラスタにインストールされていること

#### 自動デプロイの仕組み

Argo CDは、このリポジトリの`overlays/production`ディレクトリを監視し、マニフェストの変更を自動的にクラスタに同期します。

1. このリポジトリで YAML ファイルを編集
2. Git にコミット＆プッシュ
3. Argo CD が自動的に変更を検出してデプロイ

## ⚙️ 設定

### シークレット管理

このリポジトリでは、`base/backend/sealed-secret.yaml`のように[Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets)を使用して暗号化されたSecretを管理しています。

新しいSecretを追加・更新する手順は以下の通りです。

1. `kubeseal`コマンドラインツールをインストールします。
2. 暗号化されていないSecretをYAMLファイルとして作成します（例: `my-secret.yaml`）。
   ```yaml
   apiVersion: v1
   kind: Secret
   metadata:
     name: my-secret-name
   data:
     KEY: dmFsdWUK # base64エンコードした値
   ```
3. `kubeseal`コマンドでSecretを暗号化します。
   ```bash
   kubeseal < my-secret.yaml > sealed-secret.yaml --controller-namespace <sealed-secrets-namespace>
   ```
4. 生成された`sealed-secret.yaml`をリポジトリに追加し、Gitにコミットします。

## 🌐 アプリケーションへのアクセス

`overlays/production/ingress-frontend.yaml`で定義されたIngress経由でフロントエンドにアクセスできます。
デプロイ後、Ingressコントローラから割り当てられたIPアドレスまたはホスト名を確認してください。

```bash
kubectl get ingress -n household-tasks
```

## 🔄 クリーンな再デプロイ

既存リソースを削除して、マニフェストから完全に再構築する手順です。

### ⚠️ 警告
- PostgreSQL のデータが削除されます。必ずバックアップを取得してください。
- サービスがダウンタイムします（復元まで約5分）。
- TLS証明書が再発行されます（Let's Encrypt レート制限に注意）。

### データバックアップ
```bash
# PostgreSQLバックアップ
kubectl exec -it -n household-task-manager statefulset/postgres -- \
  pg_dump -U postgres household_task_manager > backup.sql

# TLS証明書バックアップ
kubectl get secret household-task-manager-tls -n household-task-manager -o yaml > tls-backup.yaml

# Sealed Secrets鍵バックアップ
kubectl get secret -n kube-system sealed-secrets-keygs4s4 -o yaml > sealed-secrets-key-backup.yaml
```

### リソース削除

```bash
# 1. ArgoCD Application削除（自動的にhousehold-task-manager namespaceのリソースを削除）
kubectl delete application household-task-manager -n argocd

# 2. Namespace削除（念のため）
kubectl delete namespace household-task-manager

# 3. PersistentVolume削除（データ削除を伴う）
kubectl delete pv postgres-pv-ssd

# 4. ClusterIssuer削除
kubectl delete clusterissuer letsencrypt-prod

# 5. ArgoCD Repository Secret削除（オプション）
kubectl delete secret repo-household-task-manager-k8s -n argocd
```

### リソース復元

```bash
# 1. ClusterIssuer デプロイ
kubectl apply -k cluster-resources/
kubectl wait --for=condition=Ready clusterissuer/letsencrypt-prod --timeout=60s

# 2. ArgoCD Repository Secret デプロイ
kubectl apply -f argocd/repo-secret/sealed-secret.yaml
kubectl wait --for=condition=Synced sealedsecret/repo-household-task-manager-k8s \
  -n argocd --timeout=60s

# 3. ArgoCD Application デプロイ
kubectl apply -f argocd/application.yaml
kubectl wait --for=jsonpath='{.status.sync.status}'=Synced \
  application/household-task-manager -n argocd --timeout=300s

# 4. 全Pod起動待機
kubectl wait --for=condition=Ready pod -l app.kubernetes.io/name=household-task-manager \
  -n household-task-manager --timeout=300s

# 5. データ復元（必要な場合）
kubectl cp backup.sql household-task-manager/postgres-0:/tmp/backup.sql
kubectl exec -it -n household-task-manager statefulset/postgres -- \
  psql -U postgres household_task_manager < /tmp/backup.sql
```

### 検証

```bash
# Podステータス確認
kubectl get pods -n household-task-manager

# TLS証明書確認
kubectl get certificate -n household-task-manager

# 外部アクセステスト
curl -I https://household-task-mgr.duckdns.org

# データベース接続テスト
kubectl exec -it -n household-task-manager deployment/backend -- \
  sh -c 'echo "SELECT 1;" | psql $DATABASE_URL'
```

##
Contributing

コントリビューションを歓迎します！バグ報告や機能追加の提案は、IssueやPull Requestでお知らせください。
