+++
title = "Kubernetes - nfs storage"
date = "2021-02-28T21:02:07+01:00"
slug = "kubernetes-nfs-storage"
draft = false
tags = ["k8s"]
featureImage = "/content/images/2021/02/kubernetes-nfs-storage.jpg"
+++

Do domowego laba k8s bardzo fajnie sprawdza się **microk8s**. Jest instalowany przez *snapa,* jego konfiguracja jest banalna i co najważniejsze posiada masę addonów, które ułatwiają odpalenie wiele rzeczy w sekundę. Jednym z takich addonów jest „storage", który utworzy nam storage class w naszym clustrze alokując storage jako *host directory*.\
Przydatne jednak może być podpięcie zewnętrznego storygu jak np. NFS.

### Przygotowanie

Na każdym z nodów clustra musimy zainstalować klienta nfs:

``` bash
apt install nfs-common -y
```

### Deploy NFS jako PV

``` bash
helm install stable/nfs-client-provisioner --set nfs.server=x.x.x.x --set nfs.path=/example/path
```

### Ustawienie domyślnej storage class

``` bash
kubectl patch storageclass nfs-client -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

### Linki

[Helm chart](https://artifacthub.io/packages/helm/moikot/nfs-client-provisioner).

