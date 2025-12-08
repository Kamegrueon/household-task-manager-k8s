# Tailscale経由でArgo CDにアクセスする設定

このディレクトリには、Tailscale経由でArgo CDのWeb UIにアクセスするための設定が含まれています。

## セットアップ手順

### 1. Tailscale OAuth Clientの作成

1. [Tailscale Admin Console](https://login.tailscale.com/admin/settings/oauth)にアクセス
2. 「Generate OAuth Client」をクリック
3. 以下の権限を設定：
   - **Devices: Write**
4. Client IDとClient Secretをコピー

### 2. OAuth SecretのSealed Secret作成

```bash
# Tailscaleの認証情報でSecretを作成し、Sealed Secretに変換
kubectl create secret generic operator-oauth \
  --from-literal=client_id=<YOUR_CLIENT_ID> \
  --from-literal=client_secret=<YOUR_CLIENT_SECRET> \
  -n argocd --dry-run=client -o yaml | \
kubeseal --controller-namespace=kube-system --format=yaml > argocd/tailscale/oauth-secret-sealed.yaml
```

### 3. kustomization.yamlの更新

`argocd/tailscale/kustomization.yaml`を編集し、以下の行のコメントを解除：

```yaml
resources:
  - rbac.yaml
  - operator.yaml
  - oauth-secret-sealed.yaml  # この行のコメントを解除
  - argocd-tailscale-service.yaml
```

`argocd/kustomization.yaml`も編集し、以下の行のコメントを解除：

```yaml
resources:
  - application.yaml
  - tailscale  # この行のコメントを解除
```

### 4. Tailscale Operatorのデプロイ

```bash
# 変更をコミット
git add argocd/
git commit -m "feat: add Tailscale support for ArgoCD access"
git push

# または、直接適用する場合
kubectl apply -k argocd/tailscale/
```

### 5. Tailscale Operatorの動作確認

```bash
# Operatorのログを確認
kubectl logs -n argocd -l app=operator -f

# Tailscale Serviceの状態を確認
kubectl get svc -n argocd argocd-server-tailscale

# LoadBalancerのIPが割り当てられるまで待つ
kubectl wait --for=jsonpath='{.status.loadBalancer.ingress}' \
  svc/argocd-server-tailscale -n argocd --timeout=300s
```

### 6. Tailscale経由でのアクセス

Tailscale Operatorが正常に動作すると、Tailscaleネットワークに `argocd` というホスト名で表示されます。

```bash
# Tailscaleネットワーク上のデバイスを確認
tailscale status

# ArgoCD Web UIにアクセス
# ブラウザで以下のURLを開く:
# http://argocd
# または
# https://argocd
```

## トラブルシューティング

### Operatorが起動しない場合

```bash
# Podの状態を確認
kubectl get pods -n argocd -l app=operator

# Podのログを確認
kubectl logs -n argocd -l app=operator

# Secretが正しく作成されているか確認
kubectl get secret operator-oauth -n argocd
kubectl describe secret operator-oauth -n argocd
```

### Serviceが公開されない場合

```bash
# Serviceの詳細を確認
kubectl describe svc argocd-server-tailscale -n argocd

# Operatorのログでエラーを確認
kubectl logs -n argocd -l app=operator | grep -i error
```

### Tailscaleネットワークに表示されない場合

1. Tailscale Admin Consoleで、タグ `tag:k8s` が正しく設定されているか確認
2. OAuth Clientの権限に「Devices: Write」が含まれているか確認
3. Operatorのログでエラーメッセージを確認

## セキュリティに関する注意

- OAuth Client SecretはSealed Secretとして暗号化されています
- Tailscaleの `tag:k8s` タグを使用してアクセス制御を行うことができます
- Tailscaleの[ACL](https://login.tailscale.com/admin/acls)で、どのデバイスがArgo CDにアクセスできるかを制御できます

## 参考リンク

- [Tailscale Kubernetes Operator](https://tailscale.com/kb/1236/kubernetes-operator/)
- [Tailscale OAuth Clients](https://tailscale.com/kb/1215/oauth-clients/)
- [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets)
