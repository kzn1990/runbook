+++
title = "Docker - backup Bazy MySQL"
date = "2020-07-02T21:11:01+02:00"
slug = "docker-backup-bazy-mysql"
draft = false
tags = ["mysql", "docker"]
featureImage = "/content/images/2020/07/docker-mysqldump-1.jpg"
+++

Wykonanie dumpa bazy danych kontenera mysql z poziomu hosta:

``` bash
docker exec NAZWA_KONTENERA /usr/bin/mysqldump -u root --password=root DATABASE > backup.sql
```

Odtworzenie dumpa:

``` bash
cat backup.sql | docker exec -i NAZWA_KONTENERA /usr/bin/mysql -u root --password=root DATABASE
```

### Linki

[Docker exec](https://docs.docker.com/engine/reference/commandline/exec/),\
[Mysqldump](https://dev.mysql.com/doc/refman/8.0/en/mysqldump.html).

