# KONFIGURASI KUBERNETES CLUSTER (3 MASTER + 3 ETCD)

### Target Arsitektur
```bash
ETCD CLUSTER (external)
  etcd-1 10.0.0.11
  etcd-2 10.0.0.12
  etcd-3 10.0.0.13

CONTROL PLANE
  master-1 10.0.0.21
  master-2 10.0.0.22
  master-3 10.0.0.23

WORKER
  worker-1 10.0.0.31
  worker-2 10.0.0.32
  worker-3 10.0.0.33

CONTROL PLANE ENDPOINT (HA)
  lb.example.local:6443
```

## FASE 1 — NODE ADMIN (1 KALI SAJA)
> Bisa laptop admin atau **1 VM khusus**
> **DILUAR** node etcd / master

## 1.1 Struktur Direktori
```bash
mkdir -p pki/{ca,etcd,apiserver}
cd pki
```

## 1.2 Buat CA (1 KALI)
```bash
openssl genrsa -out ca/ca.key 4096

openssl req -x509 -new -nodes \
  -key ca/ca.key \
  -subj "/CN=kubernetes-ca" \
  -days 3650 \
  -out ca/ca.crt
```

## FASE 2 — TLS ETCD (PER NODE, DIBUAT TERPUSAT) -> NODE ADMIN SEKALIAN

## 2.1 Buat cert ETCD SERVER + PEER (per node)
```bash
openssl genrsa -out etcd/etcd-1.key 4096

openssl req -new -key etcd/etcd-1.key \
  -subj "/CN=etcd-1" \
  -out etcd/etcd-1.csr
```

```bash
openssl x509 -req \
  -in etcd/etcd-1.csr \
  -CA ca/ca.crt -CAkey ca/ca.key -CAcreateserial \
  -out etcd/etcd-1.crt \
  -days 365 \
  -extensions v3_etcd \
  -extfile <(cat <<EOF
[v3_etcd]
subjectAltName=IP:10.0.0.11,DNS:etcd-1
extendedKeyUsage=serverAuth,clientAuth
EOF
)
```

➡️ **Ulangi untuk etcd-2 & etcd-3**
(ganti IP & CN)

## 2.2. File yang dikirim ke tiap ETCD NODE
```bash
(x) ganti dengan nama etcd yang sudah dibuat, ke setiap node etcd

ca.crt
etcd-(x).crt
etcd-(x).key
```

## FASE 3 — Jalankan ETCD (di masing-masing node)

## 1.1 Set hostname (WAJIB)
```bash
hostnamectl set-hostname etcd-1
```
➡️ **selesaikan sampai semua node**

## 1.2 Install etcd (manual binary – DISARANKAN)
> Jangan pakai repo OS → versi sering ketinggalan

```bash
ETCD_VER=v3.5.13
curl -L https://github.com/etcd-io/etcd/releases/download/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz \
  | tar xz

mv etcd-${ETCD_VER}-linux-amd64/etcd* /usr/local/bin/
```

Cek:
```bash
etcd --version
```

## 1.3 Buat user & direktori
```bash
useradd -r -s /sbin/nologin etcd

mkdir -p /var/lib/etcd
mkdir -p /etc/etcd/pki

chown -R etcd:etcd /var/lib/etcd /etc/etcd
```

## 1.4 Copy TLS (PER NODE)
> ada pada FASE 2 poin 2.2

chmod ketat:
```bash
chmod 600 /etc/etcd/pki/*.key
chown etcd:etcd /etc/etcd/pki/*
```
