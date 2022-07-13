# 使用Cert-Manager申请证书

## 使用helm安装Cert-Manager

```bash
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.8.2 \
  --set installCRDs=true
```

## 创建cloudflare的token及k8s secret

登录cloudflare管理后台，在用户界面 `Profile > API Tokens > API Tokens`下创建一个token，并将其保存到本地。
- Permissions:
  - Zone - DNS - Edit
  - Zone - Zone - Read
- Zone Resources:
  - Include - All Zones

使用`kubectl apply -n cert-manager -f apitoken.yml`命令创建一个secret。

apitoken.yml:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cloudflare-api-token-secret
type: Opaque
stringData:
  api-token: xxxx
```

## 创建ClusterIssuer

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-issuer
spec:
  acme:
    email: yym900902@gmail.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-private-key
    solvers:
    - dns01:
        cloudflare:
          apiTokenSecretRef:
            name: cloudflare-api-token-secret
            key: api-token
```


