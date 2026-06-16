+++
date = 2020-06-29T16:34:08+02:00
description = ""
draft = false
thumbnail = "/content/images/2020/07/xtradb-cluster.jpg"
slug = "xtradb-cluster"
title = "XtraDB Cluster"

+++


Żeby zapewnić HA cluster powinien składać się, z nieparzystej liczby nodów (3, 5, 7...).  Samo XtraDB jest forkiem MySQL Galera.

### Wstęp

Poniżej przykładowa konfiguracja dla dwóch nodów (niezalecana):xtradb-1 - 10.1.100.101xtradb-2 - 10.1.100.102Konfiguracja zakłada komunikację po adresach IP (samo resolvowanie dnsów przy bazach danych, jest niezalecane ze względu na performance). Dodanie kolejnych nodów przebiega analogicznie.

### Instalacja (deb/ubuntu)

```bash
apt-get update && apt-get install lsb-release gnupg
wget https://repo.percona.com/apt/percona-release_latest.$(lsb_release -sc)_all.deb
dpkg -i percona-release_latest.$(lsb_release -sc)_all.deb
apt-get update && apt-get install percona-xtradb-cluster-57 -y
```

### Konfiguracja

Przed dalszym etapem wymagane jest zatrzymanie baz na wszystkich nodach:

```bash
systemctl stop mysql.service
```

Podnoszenie clustra zawsze wygląda tak samo. Przygotowujemy jeden z nodów, którego będziemy bootstrapować, następnie podłączać do niego pozostałe nody.Na każdym z nodów edytujemy plik: _/etc/mysql/percona-xtradb-cluster.conf.d/wsrep.cnf_

> [mysqld]wsrep_provider=/usr/lib/galera3/libgalera_smm.sowsrep_cluster_address=gcomm://10.1.100.101,10.1.100.102binlog_format=ROWdefault_storage_engine=InnoDBwsrep_slave_threads= 8wsrep_log_conflictsinnodb_autoinc_lock_mode=2wsrep_node_address=10.1.100.101wsrep_cluster_name=XtraDB_Clusterwsrep_node_name=node-1pxc_strict_mode=ENFORCINGwsrep_sst_method=xtrabackup-v2wsrep_sst_auth=”sstuser:YourPassword”

Na pozostałych nodach zmieniamy tylko _wsrep_node_name_ i _wsrep_node_address_

### Inicjalizacja clustra

```bash
/etc/init.d/mysql bootstrap-pxc
```

Dodanie użytkownika API wsrep:

```bash
mysql -p -e "CREATE USER 'sstuser'@'localhost' IDENTIFIED BY 'YourPassword';GRANT PROCESS, RELOAD, LOCK TABLES, REPLICATION CLIENT ON *.* TO 'sstuser'@'localhost';FLUSH PRIVILEGES;"
```

### Dodanie kolejnego noda

Po edycji pliku konfiguracyjnego i podmianie node_name i node_address, podłączamy host poprzez wystartowanie bazy

```bash
systemctl start mysql.service
```

### Status XtraDB Cluster

Status można sprawdzić z poziomu mysql'a:

```bash
mysql -p -e "show status like 'wsrep%’;"
```

### Linki

[Dokumentacja percony](https://www.percona.com/doc/percona-xtradb-cluster/5.7/index.html),[Bootstrap noda](https://www.percona.com/doc/percona-xtradb-cluster/LATEST/bootstrap.html),[API wsrep](https://galeracluster.com/library/documentation/architecture.html),[MySQL Galera](https://galeracluster.com/products/).

