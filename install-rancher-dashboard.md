# KONFIGURASI RANCHER DASHBOARD

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```
```bash
helm version
```

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
```

```bash
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
```
```bash
helm repo update
```

```bash
kubectl create namespace cattle-system
```

```bash
helm install rancher rancher-stable/rancher \
  --namespace cattle-system \
  --create-namespace \
  --set hostname=<CONTROL_PLANE_PUBLIC_IP>.nip.io \
  --set replicas=1 \
  --set bootstrapPassword=admin \
  --set service.ipFamilyPolicy=PreferDualStack \
  --set service.ipFamilies="{IPv4,IPv6}"
```
