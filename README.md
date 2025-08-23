SELAMAT DATANG

Panduan step-by-step untuk setup server fisik menjadi 4 VM dengan IP privat + 1:1 NAT IP publik via DNAT + SNAT di host.

---

# Panduan Instalasi VM dengan 1:1 NAT IP Publik (DNAT + SNAT)

---

## Kondisi Server Fisik

* Processor: Intel Xeon E-2278G (16 Core)
* RAM: 64 GB
* IP Publik Host: 182.3.46.229 (dan blok IP publik tambahan misal 182.3.46.230–233)
* Sistem Operasi: Ubuntu Server (versi 20.04/22.04 rekomendasi)

---

# 1. Install KVM dan Libvirt di Server Fisik

```bash
sudo apt update
sudo apt install -y qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils virtinst virt-manager
```

Cek KVM siap:

```bash
sudo systemctl status libvirtd
virsh list --all
```

---

# 2. Cek dan Setup Jaringan NAT Default di Host

Jaringan `default` biasanya sudah ada di libvirt dan menggunakan NAT.

```bash
virsh net-list --all
```

Jika `default` belum aktif:

```bash
virsh net-start default
virsh net-autostart default
```

`default` biasanya pakai subnet seperti 192.168.122.0/24.

Jika ingin merubah menjadi 172.x.x.x/xx contoh 172.16.100.0/24
Buat file XML konfigurasi baru, misal default.xml
```bash
<network>
  <name>default</name>
  <uuid>generate-a-new-uuid-or omit</uuid>
  <forward mode='nat'/>
  <bridge name='virbr0' stp='on' delay='0'/>
  <ip address='172.16.100.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='172.16.100.2' end='172.16.100.254'/>
    </dhcp>
  </ip>
</network>
```

Definisikan ulang network di libvirt
```bash
virsh net-define default.xml
virsh net-start default
virsh net-autostart default
```

CEK DI HOST NYA
```bash
ip a show virbr0
```
Outputnya harus menunjukkan IP:
```sql
inet 172.16.100.1/24 scope global virbr0
```
---

# 3. Setup IP Publik Alias di Host

Misalnya kamu punya blok IP publik:

| IP           | Keterangan    |
| ------------ | ------------- |
| 182.3.46.229 | IP utama host |
| 182.3.46.230 | Alias IP VM1  |
| 182.3.46.231 | Alias IP VM2  |
| 182.3.46.232 | Alias IP VM3  |
| 182.3.46.233 | Alias IP VM4  |

Edit `/etc/netplan/01-netcfg.yaml`:

```yaml
network:
  version: 2
  ethernets:
    eno1:
      addresses:
        - 182.3.46.229/29
        - 182.3.46.230/29
        - 182.3.46.231/29
        - 182.3.46.232/29
        - 182.3.46.233/29
      gateway4: 182.3.46.225
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
```

Lalu jalankan:

```bash
sudo netplan apply
```

---

# 4. Buat VM via Virt-Install (atau Virt-Manager GUI)

Misal buat VM1:

```bash
sudo virt-install \
  --name vm1 \
  --vcpus 4 \
  --memory 16384 \
  --disk path=/var/lib/libvirt/images/vm1.qcow2,size=100 \
  --os-variant ubuntu20.04 \
  --network network=default,model=virtio \
  --cdrom /var/lib/libvirt/images/ubuntu-20.04.iso \
  --graphics none \
  --console pty,target_type=serial
```

Lakukan juga untuk VM2, VM3, VM4 dengan parameter sesuai kebutuhan.

---

# 5. Konfigurasi IP Privat di VM

Setiap VM akan dapat IP privat otomatis via DHCP dari jaringan `default` (biasanya 192.168.122.x).

Cek IP VM:

```bash
ip a
```

Misal:

| VM  | IP Privat      |
| --- | -------------- |
| VM1 | 192.168.122.10 |
| VM2 | 192.168.122.11 |
| VM3 | 192.168.122.12 |
| VM4 | 192.168.122.13 |

Kalau DHCP tidak memberikan IP yang kamu inginkan, kamu bisa set IP statis di VM (172.x.x.x juga boleh, asal cocok dengan subnet).

---

# 6. Setup 1:1 NAT di Host dengan iptables

Misal mapping IP publik ke IP privat:

| IP Publik    | IP Privat VM   |
| ------------ | -------------- |
| 182.3.46.230 | 192.168.122.10 |
| 182.3.46.231 | 192.168.122.11 |
| 182.3.46.232 | 192.168.122.12 |
| 182.3.46.233 | 192.168.122.13 |

Jalankan perintah ini di host:

```bash
# VM1
iptables -t nat -A PREROUTING -d 182.3.46.230 -j DNAT --to-destination 192.168.122.10
iptables -t nat -A POSTROUTING -s 192.168.122.10 -j SNAT --to-source 182.3.46.230

# VM2
iptables -t nat -A PREROUTING -d 182.3.46.231 -j DNAT --to-destination 192.168.122.11
iptables -t nat -A POSTROUTING -s 192.168.122.11 -j SNAT --to-source 182.3.46.231

# VM3
iptables -t nat -A PREROUTING -d 182.3.46.232 -j DNAT --to-destination 192.168.122.12
iptables -t nat -A POSTROUTING -s 192.168.122.12 -j SNAT --to-source 182.3.46.232

# VM4
iptables -t nat -A PREROUTING -d 182.3.46.233 -j DNAT --to-destination 192.168.122.13
iptables -t nat -A POSTROUTING -s 192.168.122.13 -j SNAT --to-source 182.3.46.233
```

---

# 7. Buka Forwarding di Firewall

Pastikan host menerima trafik forwarding:

```bash
iptables -A FORWARD -d 192.168.122.0/24 -j ACCEPT
iptables -A FORWARD -s 192.168.122.0/24 -j ACCEPT
```

---

# 8. Simpan aturan iptables agar tetap permanen

Install iptables-persistent:

```bash
sudo apt install iptables-persistent
sudo netfilter-persistent save
```

---

# 9. Tes Akses

* Dari luar (internet), ping atau ssh ke `182.3.46.230` akan langsung diarahkan ke VM1
* Dari VM, akses internet lancar menggunakan IP publik via SNAT

---

# 10. (Opsional) Monitoring

Gunakan `virsh list --all` untuk cek VM

Gunakan `ip a` di host dan VM untuk cek IP

Gunakan `iptables -t nat -L -n -v` untuk cek aturan NAT

---
