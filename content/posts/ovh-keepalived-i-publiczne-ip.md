+++
date = 2020-07-03T20:38:08+02:00
description = ""
draft = false
thumbnail = "/content/images/2020/07/ovh-keepalived-vrf-2.jpg"
slug = "ovh-keepalived-i-publiczne-ip"
title = "OVH - keepalived i publiczne ip (VRF)"

+++


Keepalived, narzędzie do zapewnienia loadbalancingu i wysokiej dostępności (HA).

### Wstęp

Jak wiadomo w OVH, aby przypisać adresację publiczną, trzeba wygenerować w panelu adres mac, który następnie musimy przypisać serwerowi. Powstaje więc problem, jeżeli chcemy połączyć serwery keepalive'm musimy przypisać obu serwerom te same mac addressy, co może prowadzić do różnych problemów z warstwą L2.Dodatkowo, jeżeli chcemy przypisać kilka publiczny adresów ip, niezbędne będzie zastosowanie VRF'ów (virtual routing and forwarding).

### Tablice routingu

Konfiguracja z systemd, edytujemy plik _/etc/iproute2/rt_tables_ i dodajemy nowe tablice:

```bash
100     net-bgp0
101     net-bgp1
102     net-bgp2
103     net-bgp3
104     net-bgp4
```

### Konfiguracja keepalived

Plik konfiguracyjny _/etc/keepalived/keepalived.conf_:

```bash
vrrp_instance lb-cm {
state BACKUP
interface eth0
virtual_router_id 53
priority 101
advert_int 1
nopreempt # zapobiega flappowaniu adresacji

virtual_routes { # dodanie tablic routingu (default GW)
default via 5.6.187.1 dev eth1
}

virtual_ipaddress { # przypisanie adresacji
5.6.187.16/24 dev eth1
5.6.187.17/24 dev eth2
5.6.187.18/24 dev eth3
5.6.187.20/24 dev eth4
5.6.187.21/24 dev eth5
}

notify /etc/keepalived/keepalivednotify.sh
}
```

### Podmiana maców

W skrypcie notify, oprócz powiadomienia na slacu o zmianie stanów adresacji, możemy umieścić podmianę mac adresów _/etc/keepalived/keepalivednotify.sh_:

```bash
#!/bin/bash
state=$3
echo $3  > /var/run/keepalive.state
case $3 in
MASTER)
echo "Keepalive's bringing up - state $3" |/usr/bin/logger -i -t "Tunnel:"
ip link set eth1 address 02:01:02:83:6a:6c
ip link set eth2 address 02:01:02:40:a6:f1
ip link set eth3 address 02:01:02:7e:fd:eb
ip link set eth4 address 02:01:02:c5:e2:9b
ip link set eth5 address 02:01:02:4d:7f:49
ip r a default via 5.6.187.16 dev eth1 table net-bgp0
ip r a default via 5.6.187.17 dev eth2 table net-bgp1
ip r a default via 5.6.187.18 dev eth3 table net-bgp2
ip r a default via 5.6.187.20 dev eth4 table net-bgp3
ip r a default via 5.6.187.21 dev eth5 table net-bgp4
ip rule add from 5.6.187.16/32 table net-bgp0
ip rule add from 5.6.187.17/32 table net-bgp1
ip rule add from 5.6.187.18/32 table net-bgp2
ip rule add from 5.6.187.20/32 table net-bgp3
ip rule add from 5.6.187.21/32 table net-bgp4
ip -s -s neigh flush all
;;
BACKUP)
echo "Tunnel down - state $3" |/usr/bin/logger -i -t "Tunnel:"
ip rule del from 5.6.187.16/32 table net-bgp0
ip rule del from 5.6.187.17/32 table net-bgp1
ip rule del from 5.6.187.18/32 table net-bgp2
ip rule del from 5.6.187.20/32 table net-bgp3
ip rule del from 5.6.187.21/32 table net-bgp4
ip link set eth1 address 02:01:02:03:04:01
ip link set eth2 address 02:01:02:03:04:02
ip link set eth3 address 02:01:02:03:04:03
ip link set eth4 address 02:01:02:03:04:04
ip link set eth5 address 02:01:02:03:04:05
ip -s -s neigh flush all
;;
*)
echo "Error, unknown state. Tunnel down - state $3" | /usr/bin/logger -i -t "Tunnel:"
ip rule del from 5.6.187.16/32 table net-bgp0
ip rule del from 5.6.187.17/32 table net-bgp1
ip rule del from 5.6.187.18/32 table net-bgp2
ip rule del from 5.6.187.20/32 table net-bgp3
ip rule del from 5.6.187.21/32 table net-bgp4
ip link set eth1 address 02:01:02:03:04:01
ip link set eth2 address 02:01:02:03:04:02
ip link set eth3 address 02:01:02:03:04:03
ip link set eth4 address 02:01:02:03:04:04
ip link set eth5 address 02:01:02:03:04:05
esac
```

### Linki

[Virtual routing and forwarding](https://www.kernel.org/doc/Documentation/networking/vrf.txt),[Keepalived](https://keepalived.org/index.html#).

