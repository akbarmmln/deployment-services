# konfigurasi Vault Cluster dengan Consul
node 1 : 47.237.117.191 node 2 : 47.237.117.192 node 3 : 47.237.117.193 (sebagai load balancer nya)

### 1. Instalasi & Konfigurasi Consul (di semua node)
### 1.1 Download & Install Consul
```bash
sudo apt update
sudo apt install unzip curl -y
```
simpan dengan nama install_latest_consul.sh -> chmod +x install_latest_consul.sh
```bash
#!/bin/bash
# Ambil versi terbaru Consul dari situs HashiCorp
CONSUL_VERSION=$(curl -s https://releases.hashicorp.com/consul/ | grep -oP 'consul/\K[0-9]+\.[0-9]+\.[0-9]+' | head -1)
# Tampilkan versi yang akan didownload
echo "Consul version terbaru: $CONSUL_VERSION"
# Download file ZIP
curl -O https://releases.hashicorp.com/consul/${CONSUL_VERSION}/consul_${CONSUL_VERSION}_linux_amd64.zip
# Ekstrak dan install
unzip consul_${CONSUL_VERSION}_linux_amd64.zip
sudo mv consul /usr/local/bin/
rm consul_${CONSUL_VERSION}_linux_amd64.zip
# Cek versi terinstal
consul -v
```
### 1.2 Buat direktori konfigurasi Consul (di semua node)
```bash
sudo mkdir -p /etc/consul.d /opt/consul
sudo useradd --system --home /etc/consul.d --shell /bin/false consul
sudo chown -R consul:consul /etc/consul.d /opt/consul
```
### 1.3 Konfigurasi Consul (di semua node)
file ada di /etc/consul.d/consul.hcl
```bash
datacenter = "dc1"
node_name = "nodex"
server = true
bootstrap_expect = 2
data_dir = "/opt/consul"
bind_addr = "0.0.0.0"
advertise_addr = "IP_NODE_PUBLIC"
retry_join = ["IP_NODE_PUBLIC_LAIN"]
ui = true
```
### 1.4 Service Systemd Consul (di semua node)
```bash
[Unit]
Description=Consul
Requires=network-online.target
After=network-online.target

[Service]
User=consul
Group=consul
ExecStart=/usr/local/bin/consul agent -config-dir=/etc/consul.d/
ExecReload=/bin/kill -HUP $MAINPID
KillMode=process
Restart=on-failure

[Install]
WantedBy=multi-user.target
```
### 1.5 Start Consul (di semua node)
```bash
sudo systemctl daemon-reexec
sudo systemctl enable consul
sudo systemctl start consul
```
simpan dengan nama install_latest_vault.sh -> chmod +x install_latest_vault.sh
```bash
#!/bin/bash
# Ambil versi terbaru Vault dari situs HashiCorp
VAULT_VERSION=$(curl -s https://releases.hashicorp.com/vault/ | grep -oP 'vault/\K[0-9]+\.[0-9]+\.[0-9]+' | head -1)
# Tampilkan versi yang akan didownload
echo "Vault version terbaru: $VAULT_VERSION"
# Download file ZIP
curl -O https://releases.hashicorp.com/vault/${VAULT_VERSION}/vault_${VAULT_VERSION}_linux_amd64.zip
# Ekstrak dan install
unzip vault_${VAULT_VERSION}_linux_amd64.zip
sudo mv vault /usr/local/bin/
rm vault_${VAULT_VERSION}_linux_amd64.zip
# Cek versi terinstal
vault -v
```

JIKA DIBUTUHKAN DALAM PENGECEKAN DAN LAIN LAIN NYA
-> consul members
-> consul validate /etc/consul.d/
-> sudo systemctl stop consul kemudian sudo rm -rf /opt/consul/* dan sudo systemctl start consul


### 2. Instalasi & Konfigurasi Vault (di semua node)
### 2.1 Install Vault
```bash
sudo apt update
sudo apt install unzip curl -y
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
