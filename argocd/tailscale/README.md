# Tailscale経由でArgo CDにアクセスする設定

このディレクトリには、Tailscale経由でArgo CDのWeb UIにアクセスするための設定が含まれています。

## セットアップ手順

### 1. Tailscale OAuth Clientの作成

1. [Tailscale Admin Console](https://login.tailscale.com/admin/settings/oauth)にアクセス
2. 「Generate OAuth Client」をクリック
3. 以下の権限を設定：
   - **Devices: Write**
   - **Auth Keys: Write** ← これが重要！
4. **Tags**セクションで以下のタグを追加：
   - `tag:k8s-operator`
5. Client IDとClient Secretをコピー

### 2. 環境変数ファイルの作成

```bash
# ~/.tailscale-argocd.env ファイルを作成
cat > ~/.tailscale-argocd.env << 'EOF'
TAILSCALE_CLIENT_ID="your-client-id"
TAILSCALE_CLIENT_SECRET="your-client-secret"
EOF
```

### 3. Tailscale Operatorのインストール

```bash
# 環境変数を読み込む
source ~/.tailscale-argocd.env

# Helm repoを追加
ssh rpi-master-1 "helm repo add tailscale https://pkgs.tailscale.com/helmcharts && helm repo update"

# Tailscale operatorをインストール
ssh rpi-master-1 "helm install tailscale-operator tailscale/tailscale-operator \
  --namespace=argocd \
  --set-string oauth.clientId='${TAILSCALE_CLIENT_ID}' \
  --set-string oauth.clientSecret='${TAILSCALE_CLIENT_SECRET}'"
```

### 4. ArgoCD Serviceの作成

```bash
# ServiceをTailscale経由で公開
kubectl apply -f argocd/tailscale/argocd-tailscale-service.yaml

# LoadBalancer IPが割り当てられるまで待つ
kubectl wait --for=jsonpath='{.status.loadBalancer.ingress}' \
  svc/argocd-server-tailscale -n argocd --timeout=300s

# Service状態を確認
kubectl get svc argocd-server-tailscale -n argocd
```

### 5. 動作確認

```bash
# Operatorのログを確認
kubectl logs -n argocd -l app=operator

# Tailscaleネットワーク上のデバイスを確認
tailscale status

# または Tailscale Admin Consoleで確認
# https://login.tailscale.com/admin/machines
```

### 6. Tailscale経由でのアクセス

Tailscale Operatorが正常に動作すると、Tailscaleネットワークに `argocd` というホスト名で表示されます。

```bash
# ArgoCD Web UIにアクセス
# ブラウザで以下のURLを開く:
# http://argocd.taile<random>.ts.net
# または Tailscale IP (例: http://100.80.43.111)
```

## トラブルシューティング

### Operatorが起動しない場合

```bash
# Podの状態を確認
kubectl get pods -n argocd -l app=operator

# Podのログを確認
kubectl logs -n argocd -l app=operator

# Helm releaseの確認
helm list -n argocd
```

### Serviceが公開されない場合

```bash
# Serviceの詳細を確認
kubectl describe svc argocd-server-tailscale -n argocd

# Operatorのログでエラーを確認
kubectl logs -n argocd -l app=operator | grep -i error
```

### 一般的なエラーと解決策

#### エラー: "requested tags [tag:k8s-operator] are invalid or not permitted"
**原因**: OAuth Clientに `tag:k8s-operator` の使用権限がない
**解決策**: Tailscale Admin ConsoleでOAuth ClientのTagsに `tag:k8s-operator` を追加

#### エラー: "calling actor does not have enough permissions"
**原因**: OAuth Clientに必要な権限がない
**解決策**: OAuth Clientに以下の権限を付与:
- Devices: Write
- Auth Keys: Write

### Tailscaleネットワークに表示されない場合

1. Tailscale Admin Consoleで、タグ `tag:k8s-operator` が正しく設定されているか確認
2. OAuth Clientの権限に「Devices: Write」と「Auth Keys: Write」が含まれているか確認
3. Operatorのログでエラーメッセージを確認

## アンインストール

```bash
# Serviceを削除
kubectl delete svc argocd-server-tailscale -n argocd

# Tailscale operatorをアンインストール
ssh rpi-master-1 "helm uninstall tailscale-operator -n argocd"
```

## セキュリティに関する注意

- OAuth Client SecretはHelmのSecretとして暗号化されています
- Tailscaleの `tag:k8s-operator` タグを使用してアクセス制御を行うことができます
- Tailscaleの[ACL](https://login.tailscale.com/admin/acls)で、どのデバイスがArgo CDにアクセスできるかを制御できます

## 参考リンク

- [Tailscale Kubernetes Operator](https://tailscale.com/kb/1236/kubernetes-operator/)
- [Tailscale Helm Chart](https://github.com/tailscale/tailscale/tree/main/cmd/k8s-operator/deploy/chart)
- [Tailscale OAuth Clients](https://tailscale.com/kb/1215/oauth-clients/)
- [Tailscale Tags](https://tailscale.com/kb/1068/acl-tags/)