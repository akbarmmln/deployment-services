# KONFIGURASI INGRESS NGINX

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```
```bash
helm version
```

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
```
```bash
helm repo update
```

```bash
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.ipFamilyPolicy=PreferDualStack \
  --set controller.service.ipFamilies="{IPv4,IPv6}"
```
