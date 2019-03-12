# Implementasi MySQL Cluster dengan Load Balancer menggunakan ProxySQL

## 1. Setting Up Machine Menggunakan Vagrant
## 2. Instalasi dan Konfigurasi Cluster Manager

Login ke Cluster Manager dan download ```.deb``` file:
```
$ cd ~
$ wget https://dev.mysql.com/get/Downloads/MySQL-Cluster-7.6/mysql-cluster-community-management-server_7.6.6-1ubuntu18.04_amd64.deb  
```
Install  ```ndb_mgmd``` menggunakan ```dpkg```:
```
$ sudo dpkg -i mysql-cluster-community-management-server_7.6.6-1ubuntu18.04_amd64.deb
```
Buat direktori untuk menyimpan hasil konfigurasi NDB Manager
```
$ sudo mkdir /var/lib/mysql-cluster
```
Edit file konfigurasi
```
$ sudo nano /var/lib/mysql-cluster/config.ini
```
Copy file pada direktori
```
[ndbd default]
# Options affecting ndbd processes on all data nodes:
NoOfReplicas=2  

[ndb_mgmd]
# Management process options:
hostname=192.168.33.11 
datadir=/var/lib/mysql-cluster 

[ndbd]
hostname=192.168.33.12 
NodeId=2            
datadir=/usr/local/mysql/data   

[ndbd]
hostname=192.168.33.13 
NodeId=3            
datadir=/usr/local/mysql/data  

[mysqld]
# SQL node options:
hostname=192.168.33.12

[mysqld]
hostname=192.168.33.13
```
Jalankan ndb_mgmd
```
$ sudo ndb_mgmd -f /var/lib/mysql-cluster/config.ini
```
Maka akan ada output:
[gambar]
Sebelum menjalankan service manager, kill dahulu service yang sedang berjalan 
```
$ sudo pkill -f ndb_mgmd
```
Edit file konfigurasi management
```
$ sudo nano /etc/systemd/system/ndb_mgmd.service
```
Copy file pada direktori
```
[Unit]
Description=MySQL NDB Cluster Management Server
After=network.target auditd.service

[Service]
Type=forking
ExecStart=/usr/sbin/ndb_mgmd -f /var/lib/mysql-cluster/config.ini
ExecReload=/bin/kill -HUP $MAINPID
KillMode=process
Restart=on-failure

[Install]
WantedBy=multi-user.target
```
Save dan exit, lalu jalankan daemon mode dan enable kan service yang sudah dibuat
```
$ sudo systemctl daemon-reload
$ sudo systemctl enable ndb_mgmd
```
Jalankan service
```
$ sudo systemctl start ndb_mgmd
```
Cek status 
```
$ sudo systemctl status ndb_mgmd
```
Maka akan ada output:
[gambar]
Tambahkan aturan untuk mengizinkan koneksi masuk lokal dari kedua node data
```
$ sudo ufw allow from 192.168.33.12
$ sudo ufw allow from 192.168.33.13
```
Maka akan ada output:
[gambar]

## 3. Instalasi dan Konfigurasi Data Nodes 
Login ke Data Nodes dan download ```.deb``` file:
```
$ cd ~
$ wget https://dev.mysql.com/get/Downloads/MySQL-Cluster-7.6/mysql-cluster-community-data-node_7.6.6-1ubuntu18.04_amd64.deb 
```
Sebelum menginstall data nodes, install terlebih dahulu ```libclass-methodmaker-perl```
```
$ sudo apt update
$ sudo apt install libclass-methodmaker-perl
```
Setelah itu install data node menggunakan ```dpkg```
```
$ sudo dpkg -i mysql-cluster-community-data-node_7.6.6-1ubuntu18.04_amd64.deb
```
Buat konfigurasi data nodes dari MySQL standard location
```
$ sudo nano /etc/my.cnf
```
Copy file pada direktori kemudian save dan exit
```
[mysql_cluster]
# Options for NDB Cluster processes:
ndb-connectstring=192.168.33.11 
```
Sebelum membuat daemon, buat sebuah direktori pada node kemudian jalankan
```
$ sudo mkdir -p /usr/local/mysql/data
$ sudo ndbd
```
Maka akan keluar output
[gambar]
Tambahkan aturan untuk mengizinkan koneksi dari Cluster Manager dan Data Nodes lainnya
```
$ sudo ufw allow from 192.168.33.11
$ sudo ufw allow from 192.168.33.13
```
Output
[gambar]
Sebelum menjalankan service manager, kill dahulu ```ndbd``` yang sedang berjalan 
```
$ sudo pkill -f ndbd
```
Buat konfigurasi
```
$ sudo nano /etc/systemd/system/ndbd.service
```
Copy file pada direktori
```
[Unit]
Description=MySQL NDB Data Node Daemon
After=network.target auditd.service

[Service]
Type=forking
ExecStart=/usr/sbin/ndbd
ExecReload=/bin/kill -HUP $MAINPID
KillMode=process
Restart=on-failure

[Install]
WantedBy=multi-user.target
```
Reload konfigurasi sistem manager menggunakan ```daemon-reload```
```
$ sudo systemctl daemon-reload
```
Hubungkan dan buat data node daemon memulai reboot, mulai service
```
$ sudo systemctl enable ndbd
$ sudo systemctl start ndbd
```
Cek running pada NDB Cluster Management
```
$ sudo systemctl status ndbd
```
Maka akan keluar output
[gambar]

## 4. Konfigurasi 
