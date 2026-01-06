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

## 1.5 systemd unit
> masuk ke node etcd : /etc/systemd/system/etcd.service
```bash
[Unit]
Description=etcd
Documentation=https://etcd.io
After=network.target

[Service]
User=etcd
Type=notify
ExecStart=/usr/local/bin/etcd \
  --name=etcd-1 \
  --data-dir=/var/lib/etcd \
  \
  --initial-advertise-peer-urls=https://10.0.0.11:2380 \
  --listen-peer-urls=https://10.0.0.11:2380 \
  \
  --advertise-client-urls=https://10.0.0.11:2379 \
  --listen-client-urls=https://10.0.0.11:2379,https://127.0.0.1:2379 \
  \
  --initial-cluster=etcd-1=https://10.0.0.11:2380,etcd-2=https://10.0.0.12:2380,etcd-3=https://10.0.0.13:2380 \
  --initial-cluster-state=new \
  --initial-cluster-token=etcd-cluster-1 \
  \
  --cert-file=/etc/etcd/pki/etcd-1.crt \
  --key-file=/etc/etcd/pki/etcd-1.key \
  --trusted-ca-file=/etc/etcd/pki/ca.crt \
  --client-cert-auth=true \
  \
  --peer-cert-file=/etc/etcd/pki/etcd-1.crt \
  --peer-key-file=/etc/etcd/pki/etcd-1.key \
  --peer-trusted-ca-file=/etc/etcd/pki/ca.crt \
  --peer-client-cert-auth=true \
  \
  --snapshot-count=10000 \
  --heartbeat-interval=100 \
  --election-timeout=1000

Restart=always
RestartSec=10
LimitNOFILE=40000

[Install]
WantedBy=multi-user.target
```

➡️ **ini sample untuk etcd-1. lakukan hal serupa dengan penggantian beberapa parameter sesuai node etcd nya. lakukan di semua node etcd**

## 1.6 Reload systemd
> masuk ke node etcd
```bash
systemctl daemon-reexec
systemctl daemon-reload
```
➡️ **lakukan di semua node etcd**

## 1.7 Start SEMUA node
> masuk ke node etcd
```bash
systemctl enable etcd
systemctl start etcd
```
➡️ **lakukan di semua node etcd**

## 1.8 Verifikasi status
```bash
systemctl status etcd
```

## 1.9 Validasi (dari salah satu node etcd)
```bash
ETCDCTL_API=3 etcdctl \
  --endpoints=https://10.0.0.11:2379 \
  --cacert=/etc/etcd/pki/ca.crt \
  --cert=/etc/etcd/pki/etcd-1.crt \
  --key=/etc/etcd/pki/etcd-1.key \
  member list
```

## 1.10 Cek kesehatan
```bash
ETCDCTL_API=3 etcdctl \
  --endpoints=https://10.0.0.11:2379,https://10.0.0.12:2379,https://10.0.0.13:2379 \
  --cacert=/etc/etcd/pki/ca.crt \
  --cert=/etc/etcd/pki/etcd-1.crt \
  --key=/etc/etcd/pki/etcd-1.key \
  endpoint health
```
