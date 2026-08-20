# KONFIGURASI MYSQL CLUSTER INNODB (3 NODE MYSQL + 1 ROUTER)

### Target Arsitektur
```bash
Node1 (192.168.10.11) → MySQL
Node2 (192.168.10.12) → MySQL
Node3 (192.168.10.13) → MySQL
Router (192.168.10.20) → MySQL Router
```

## FASE 0 — Persiapan Jaringan (Semua Node)
> Set hostname tiap node
```bash
sudo hostnamectl set-hostname mysql-node-x
```

> Edit /etc/hosts di SEMUA node (termasuk router)
```bash
ip-node-1   mysql-node-1
ip-node-x   mysql-node-x
ip-node-router   mysql-router-1
```

## FASE 1 — Install MySQL
> Install di semua node
```bash
sudo apt update
sudo apt install mysql-server -y
```

> Cek versi
```bash
mysql --version
```

## FASE 2 — Konfigurasi MySQL
> Lakukan di semua node, sesuaikan server-id (harus uniq di setiap node)
```bash
sudo vi /etc/mysql/mysql.conf.d/mysqld.cnf
```
> Tambahkan/Ubah :
```bash
[mysqld]

server-id=1
bind-address=0.0.0.0

log_bin=mysql-bin
binlog_format=ROW

gtid_mode=ON
enforce_gtid_consistency=ON

transaction_write_set_extraction=XXHASH64
```

> Setelah selesai, lakukan restart
```bash
sudo systemctl restart mysql
```

## FASE 3 Buat User untuk Cluster
> Lakukan di semua node
```bash
mysql -u root -p
```
> Lalu jalankan
```bash
CREATE USER 'clusteradmin'@'%' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON *.* TO 'clusteradmin'@'%' WITH GRANT OPTION;
```

## FASE 4 Install MySQL Shell
> Boleh di node router, atau manapun
```bash
sudo apt install mysql-shell -y
```

> Cek versi:
```bash
mysqlsh --version
```

## FASE 4.1 Konfigurasi Tiap Instance untuk Group Replication
> Dari mesin pada fase 4, jalankan command untuk masuk ke MySQL Shell:
```bash
mysqlsh
```

> Lakukan configureInstance pada masing masing mysql node db
```bash
\connect clusteradmin@{ip-node-1}:3306
dba.configure_instance()
```

> Setelah selesai lakukan restart di semua mysql node
```bash
sudo systemctl restart mysql
```

## FASE 4.2 Membuat Cluster
> Masih di MySQL Shell, koneksi ke mysql node 1 (calon primary):
```bash
\connect clusteradmin@{ip-node-1}:3306
var cluster = dba.create_cluster('myCluster')
```

> Jika dba.createCluster telah dilakukan lalu koneksi tertutup, lakukan command ini
```bash
var cluster = dba.get_cluster('myCluster')
```

## FASE 4.3 Menambahkan MYSQL Node ke Cluster
> Masih dalam session yang sama:
```bash
cluster.add_instance("clusteradmin@{ip-node}:3306")
```
> Lalu akan muncul prompt, pilih C(clone). Ulangi langkah ini di semua mysql node yang akan di join

## FASE 4.4 Verifikasi
> Cek status untuk melihat status cluster
```bash
cluster.status()
```
