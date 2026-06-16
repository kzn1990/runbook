+++
date = 2020-12-25T12:55:37+01:00
description = ""
draft = false
thumbnail = "/content/images/2020/12/microk8s-daemon-proxy.jpg"
slug = "microk8s-fail-snap-microk8s-daemon-proxy-is-not-running"
title = "Microk8s - FAIL: snap.microk8s.daemon-proxy"
tags = ["k8s", "lxd"]
+++


Podczas deployu microk8s w kontenerze LXD może pojawić się problem z daemon-proxy. Wówczas w statusie microk8s zobaczymy:

> FAIL:  Service snap.microk8s.daemon-proxy is not running

W logach samego procesu (_journalctl -u snap.microk8s.daemon-proxy_) widzimy:

> I1130 18:53:44.150679 1 conntrack.go:52] Setting nf_conntrack_max to 1048576I1130 18:53:44.152679 1 conntrack.go:83] Setting conntrack hashsize to 262144error: write /sys/module/nf_conntrack/parameters/hashsize: operation not supported

Problem ten jest związany z ilością połączeń conntrack i jest zależny od ilości rdzeni/wątków procesora. W przypadku małej liczby rdzeni CPU problem może w ogóle nie występować.

Rozwiązaniem jest oczywiście zwiększenie parametru conntrack hashsize na wskazaną w logu powyżej (lub większą) na serwerze nadrzędnym (hoscie):

```
echo 262144 > /sys/module/nf_conntrack/parameters/hashsize
```

### Linki

[Microk8s LXD](https://microk8s.io/docs/lxd)

