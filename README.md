# Implementasi MySQL Cluster dengan Load Balancer menggunakan ProxySQL

## 1. Setting Up Machine Menggunakan Vagrant
Dibawah ini merupakan pembagian Arsitektur dan IP:

No | HostName |    IP    | Keterangan  |
---|----------|----------|-------------|
1  |clusterdb1|192.168.33.11|ndb-Manager|
2 |clusterdb2|192.168.33.12|Node 1 dan API 1|
3 |clusterdb3|192.168.33.13|Node 2 dan API 2|
4 |clusterdb4|192.168.33.14|Load Balance(ProxySQL)|

Pertama siapkan konfigurasi vagranfile seperti yang dibawah untuk membuat node. kali ini kita akan membuat 4 node yaitu 1 ndb-manager, 1 load balance(Proxy), 2 node dan 2 service API 

```
Vagrant.configure("2") do |config|
  (1..4).each do |i|
    config.vm.define "clusterdb#{i}" do |node|
      node.vm.hostname = "clusterdb#{i}"
      node.vm.box = "bento/ubuntu-18.04"
      node.vm.network "private_network", ip: "192.168.33.1#{i}"

      # Opsional. Edit sesuai dengan nama network adapter di komputer
      # node.vm.network "public_network", bridge: "Qualcomm Atheros QCA9377 Wireless Network Adapter"
      
      node.vm.provider "virtualbox" do |vb|
        vb.name = "clusterdb#{i}"
        vb.gui = false
        vb.memory = "512"
      end

      node.vm.provision "shell", path: "provision/bootstrap.sh", privileged: false
    end
  end
end
```
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
![Ss]
Tambahkan aturan untuk mengizinkan koneksi masuk lokal dari kedua node data
```
$ sudo ufw allow from 192.168.33.12
$ sudo ufw allow from 192.168.33.13
```
Maka akan ada output:
![Ss](https://github.com/afrianmc/mysql-cluster/blob/master/screenshot/local%20connection%20from%20both%20data%20node.PNG)

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

## 4. Verifikasi Installasi Service Node
Login pada service node dan donwload packagenya
```
$ wget https://dev.mysql.com/get/Downloads/MySQL-Cluster-7.6/mysql-cluster_7.6.6-1ubuntu16.04_amd64.deb-bundle.tar
```
Buat direktori dan masuk ke direktori tersebut
```
$ mkdir install
$ cd install
```
Ekstrak package hasil download
```
$ tar -xvf mysql-cluster_7.6.6-1ubuntu16.04_amd64.deb-bundle.tar -C
```
Update dan install depedency
```
$ sudo apt update
$ sudo apt install libaio1 libmecab2
```
Instal masing-masing package 
```
$ sudo dpkg -i mysql-common_7.6.6-1ubuntu16.04_amd64.deb
$ sudo dpkg -i mysql-cluster-community-client_7.6.6-1ubuntu16.04_amd64.deb
$ sudo dpkg -i mysql-client_7.6.6-1ubuntu16.04_amd64.deb
$ sudo dpkg -i mysql-cluster-community-server_7.6.6-1ubuntu16.04_amd64.deb
```
Saat instalasi mysql cluster comunity server, maka akan muncul prompt untuk memasukan password root. masukan password vagrant

Instal mysql server
```
$ sudo dpkg -i mysql-server_7.6.6-1ubuntu16.04_amd64.deb
```
tambahkan kode berikut pada konfigurasi di /etc/mysql/my.cnf
```
[mysqld]
# Options for mysqld process:
ndbcluster                         

[mysql_cluster]
# Options for NDB Cluster processes:
ndb-connectstring=192.168.33.11
```
restart mysql server
```
$ sudo systemctl restart mysql
```
enable kan service mysql server
```
$ sudo systemctl enable mysql
```
### ** Lakukan step diatas pada semua service **
Verifikasi service node

jalankan command
```
$ mysql -u root -p 
```
Maka akan muncul tampilan
```
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 3
Server version: 5.7.22-ndb-7.6.6 MySQL Cluster Community Server (GPL)

Copyright (c) 2000, 2018, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql>
```
Kemudian cek
```
mysql > SHOW ENGINE NDB STATUS \G
```
## 5. Instalasi ProxySQL
Download instalasi ProxySQL, Jalankan command pada server Proxy untuk mendownload
```
$ cd /tmp
$ curl -OL https://github.com/sysown/proxysql/releases/download/v1.4.4/proxysql_1.4.4-ubuntu16_amd64.deb
```
Install package yang sudah didownload
```
$ sudo dpkg -i proxysql_*
```
Untuk menghubungkan Service API dengan ProxySQL maka diperlukan instalasi mysql-client
```
$ sudo apt-get update
$ sudo apt-get install mysql-client
```
Aktifkan layanan ProxySQL 
```
$ sudo systemctl start proxysql
```
Cek apakah sudah berhasil berjalan atau belum
```
$ sudo systemctl status proxysql
```
** Setting password Admin ProxySQL, secara default awal password nya adalah admin **
```
$ mysql -u admin -p -h 127.0.0.1 -P 6032 --prompt='ProxySQLAdmin> '
```
Ganti password dengan password yang diinginkan
```
ProxySQLAdmin > UPDATE global_variables SET variable_value='admin:passwordbaru' WHERE variable_name='admin-admin_credentials';
ProxySQLAdmin > LOAD ADMIN VARIABLES TO RUNTIME;
ProxySQLAdmin > SAVE ADMIN VARIABLES TO DISK;
```
## 6. Konfigurasi Monitoring di Service API
Download file sql addition yang sudah disiapkan dengan command pada service API
```
$ curl -OL https://gist.github.com/lefred/77ddbde301c72535381ae7af9f968322/raw/5e40b03333a3c148b78aa348fd2cd5b5dbb36e4d/addition_to_sys.sql
```
Buat konfigurasi dalam file sql kedalam service
```
$ mysql -u root -p < addition_to_sys.sql
```
Login ke MySQL
```
$ mysql -u root -p
```
Buat user bernama monitor
```
CREATE USER 'monitor'@'%' IDENTIFIED BY 'monitorpassword';
GRANT SELECT on sys.* to 'monitor'@'%';
FLUSH PRIVILEGES;
```
### ** Lakukan step diatas pada semua service API**
## 7. Konfigurasi monitoring pada ProxySQL
```
ProxySQLAdmin > UPDATE global_variables SET variable_value='monitor' WHERE variable_name='mysql-monitor_username';
ProxySQLAdmin > LOAD MYSQL VARIABLES TO RUNTIME;
ProxySQLAdmin > SAVE MYSQL VARIABLES TO DISK;
```
Daftarkan service API kedalam ProxySQL
```
ProxySQLAdmin > INSERT INTO mysql_group_replication_hostgroups (writer_hostgroup, backup_writer_hostgroup, reader_hostgroup, offline_hostgroup, active, max_writers, writer_is_also_reader, max_transactions_behind) VALUES (2, 4, 3, 1, 1, 3, 1, 100);
ProxySQLAdmin > INSERT INTO mysql_servers(hostgroup_id, hostname, port) VALUES (2, '192.168.31.104', 3306);
ProxySQLAdmin > INSERT INTO mysql_servers(hostgroup_id, hostname, port) VALUES (2, '192.168.31.105', 3306);

ProxySQLAdmin > LOAD MYSQL SERVERS TO RUNTIME;
ProxySQLAdmin > SAVE MYSQL SERVERS TO DISK;
```
Cek apakah Service API sudah terhubung dengan ProxySQL
```
ProxySQLAdmin > SELECT hostgroup_id, hostname, status FROM runtime_mysql_servers;
```

## ** Buat User MySQL pada Service API untuk dapat diakses melalui ProxySQL **
Login pada server service API
```
service1 > mysql -u root -p
mysql > CREATE USER 'mysqlcluster'@'%' IDENTIFIED BY 'vagrant';
mysql > GRANT ALL PRIVILEGES on clustertest.* to 'mysqlcluster'@'%';
mysql > FLUSH PRIVILEGES;
mysql > exit;
```

## ** Buat User MySQL pada ProxySQL agar dapat diakses melalui aplikasi **
```
ProxySQLAdmin > INSERT INTO mysql_users(username, password, default_hostgroup) VALUES ('mysqlcluster', 'vagrant', 2);
ProxySQLAdmin > LOAD MYSQL USERS TO RUNTIME;
ProxySQLAdmin > SAVE MYSQL USERS TO DISK;
```

## Referensi
https://www.digitalocean.com/community/tutorials/how-to-create-a-multi-node-mysql-cluster-on-ubuntu-18-04
https://www.digitalocean.com/community/tutorials/how-to-use-proxysql-as-a-load-balancer-for-mysql-on-ubuntu-16-04
