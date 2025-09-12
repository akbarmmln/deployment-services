# konfigurasi Vault Cluster dengan raf
node 1 : 47.237.117.191 node 2 : 47.237.117.192 node 3 : 47.237.117.193 (sebagai load balancer nya)

# 1. Instalasi & Konfigurasi Vault (di semua node)
# 2. Buat direktori dan user Vault
    sudo mkdir -p /etc/vault.d /var/lib/vault/data
    sudo useradd --system --home /etc/vault.d --shell /bin/false vault
    sudo chown -R vault:vault /etc/vault.d /var/lib/vault/data

# 3. Konfigurasi Vault Raft (di semua node)
    -> /etc/vault/config.hcl
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
4.  Service systemd Vault
5.  Start vault
6.  Unseal Vault
7.  Join Vault (jika ada tambahan node)
    case: Node 3 (baru): 47.237.117.193
    langkah 1, 2 dan 3, pada langkah ke-3 Jangan isi retry_join di sini, karena kita akan join secara manual
    lalu joinkan
    ```bash
    vault operator raft join http:<LEADER_NODE_IP>:8200
    ```
8.  Cek Status Cluster
    vault operator raft list-peers
NOTED JIKA ADA SECURITY GROUP ATAU FIREWALL AKTIF, PERLU DILAKUKAN PENYESUAIN TERHADAP PORT BERIKUT: 8200, 8300, 8301, 8500, 8600, 8302
journalctl -u vault.service -n 50 --no-pager
