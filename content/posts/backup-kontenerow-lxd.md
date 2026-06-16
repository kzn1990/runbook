+++
date = 2020-09-22T20:27:02+02:00
description = ""
draft = false
thumbnail = "/content/images/2020/09/backup-lxd.jpg"
slug = "backup-kontenerow-lxd"
title = "Backup kontenerów LXD"
tags = ["lxd", "backup", "lxc"]
+++


LXD to tak naprawdę nakładka (api) na kontenery LXC.Backup kontenerów możemy robić na kilka sposobów.

### Snapshot

Snapshot to tak naprawdę nic innego, jak rsync file systemu danego kontenera. Snapshot nie zapisuje konfiguracji kontenera!

```bash
lxc snapshot nazwaKontenera nazwaSnapshotu
lxc restore nazwaKontenera nazwaSnapshotu
```

### Export

Eksport kontenera zapisuje do pliku tarball cały file system kontenera, wraz ze wszystkimi snapshotami, oraz jego konfiguracją.

```bash
lxc export nazwaKontenera /sciezka/do/pliku.tar
lxc import /sciezka/do/pliku.tar
```

### Copy

Można również skopiować dany kontener obok lokalnie lub na inny host LXD

```bash
lxd copy nazwaKontenera nowyKontener
```

### Rsync

Możemy po prostu użyć starego dobrego rsynca do backupu kontenerów/snapshotów.

```bash
rsync -arhvP /var/lib/lxc /sciezka/do/backupu
```

### Skrypt

Prosty skrypt backupujący wszystkie kontenery na danym hoscie

```bash
#!/bin/bash
for x in $(lxc list -c n --format csv); do
   lxc export "${x}" "/opt/backups/lxd/${i}-backup-$(date +'%d-%m-%Y').tar"
done
```

### Linki

[Projekt LXD](https://linuxcontainers.org/lxd/introduction/),[Backup lxd by redhat](https://lxd.readthedocs.io/en/latest/backup/).

