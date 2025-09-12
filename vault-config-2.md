# konfigurasi Vault Cluster dengan Consul
node 1 : 47.237.117.191 node 2 : 47.237.117.192 node 3 : 47.237.117.193 (sebagai load balancer nya)

### 1. Instalasi & Konfigurasi Vault (di semua node)
### 1.1 Download & Install Consul
```bash
sudo apt update
sudo apt install unzip curl -y
```
CARA 1:
```bash
CONSUL_VERSION="1.16.2"
curl -O https://releases.hashicorp.com/consul/${CONSUL_VERSION}/consul_${CONSUL_VERSION}_linux_amd64.zip
unzip consul_${CONSUL_VERSION}_linux_amd64.zip
sudo mv consul /usr/local/bin/
consul -v
```
### 2. Buat direktori dan user Vault
```bash
sudo mkdir -p /etc/vault.d /var/lib/vault/data
sudo useradd --system --home /etc/vault.d --shell /bin/false vault
sudo chown -R vault:vault /etc/vault.d /var/lib/vault/data
```
### 3. Konfigurasi Vault Raft (di semua node)
file ada di /etc/vault/config.hcl
```bash
listener "tcp" {
    address     = "0.0.0.0:8200"
    tls_disable = 1
}
storage "raft" {
    path    = "/var/lib/vault/data"
    node_id = "nodex"
    
    retry_join {
        leader_api_addr = "http://<IP_NODEOTHER_ke-1>:8200"
    }
    
    retry_join {
        leader_api_addr = "http://<IP_NODEOTHER_ke-n>:8200"
    }
}
api_addr = "http://<IP_NODE_PUBLIC_SELFT>:8200"
cluster_addr = "http://<IP_NODE_PUBLIC_SELFT>:8201"
ui = true
disable_mlock = true
```
### 4. Service systemd Vault (di semua node)
```bash
[Unit]
Description=Vault
Requires=network-online.target
After=network-online.target

[Service]
User=vault
Group=vault
ExecStart=/usr/local/bin/vault server -config=/etc/vault.d/vault.hcl
ExecReload=/bin/kill -HUP $MAINPID
KillMode=process
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```
### 5. Start vault (di semua node)
```bash
sudo systemctl daemon-reexec
sudo systemctl enable vault
sudo systemctl start vault
```
### 6. Inisialisasi dan Unseal Vault
```bash
export VAULT_ADDR='http://<IP_PUBLIC>:8200'
vault operator init (Lakukan di satu node saja)
vault operator unseal (Lakukan di semua node)
```
NOTES: Simpan unseal keys dan root token
### 7. Join Vault (jika ada tambahan node)
case: Node 3 (baru): 47.237.117.193
langkah 1, 2 dan 3, pada langkah ke-3 Jangan isi retry_join di sini, karena kita akan join secara manual
lalu joinkan dengan perintah:
```bash
vault operator raft join http:<LEADER_NODE_IP>:8200
```
### 8. Cek Status Cluster
```bash
vault operator raft list-peers
```

NOTED JIKA ADA SECURITY GROUP ATAU FIREWALL AKTIF, PERLU DILAKUKAN PENYESUAIN TERHADAP PORT BERIKUT: 8200, 8300, 8301, 8500, 8600, 8302
journalctl -u vault.service -n 50 --no-pager
