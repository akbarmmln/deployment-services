# KONFIGURASI MYSQL CLUSTER INNODB (3 NODE MYSQL + 1 ROUTER)

### Target Arsitektur
```bash
Node1 (192.168.10.11) → MySQL
Node2 (192.168.10.12) → MySQL
Node3 (192.168.10.13) → MySQL
Router (192.168.10.20) → MySQL Router
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
> Dari mesin pada fase 4, jalankan MySQL Shell:
```bash
mysqlsh
```






