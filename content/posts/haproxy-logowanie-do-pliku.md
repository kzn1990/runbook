+++
date = 2020-07-20T17:30:33+02:00
description = ""
draft = false
thumbnail = "/content/images/2020/07/haproxy-logowanie-do-pliku.jpg"
slug = "haproxy-logowanie-do-pliku"
title = "HAProxy - logowanie do pliku"
tags = ["haproxy"]
+++


Haproxy nie umożliwia nam zapisywania logów bezpośrednio do pliku i trzeba w tym celu użyć np. rsysloga. Logi można wysyłać na zdalny serwer logów, lub odpalić taki serwer lokalnie.

### Konfiguracja

Dodajmy konfigurację do rsysloga _/etc/rsyslog.d/49-haproxy.conf_

```
$ModLoad imudp
$UDPServerAddress 127.0.0.1
$UDPServerRun 514
local0.panic,local0.alert,local0.crit -/var/log/haproxy-critical.log
&       ~
local0.error -/var/log/haproxy-error.log
&       ~
:msg, contains, "SSL handshake" /var/log/haproxy-ssl.log
&       ~
:msg, contains, "<BADREQ>"  /var/log/haproxy-badreq.log
&       ~
local0.* -/var/log/haproxy.log
&       ~
```

Konfiguracja _haproxy.conf_

```
global
  log     127.0.0.1   local0
  log     127.0.0.1   local0 info
  log     127.0.0.1   local0 crit
  log     127.0.0.1   local0 alert

defaults
  log global
  log-format %ci:%cp\ [%t]\ %ft\ %b/%s\ %Tq/%Tw/%Tc/%Tr/%Tt\ %ST\ %B\ %CC\ %CS\ %tsc\ %ac/%fc/%bc/%sc/%rc\ %sq/%bq\ %hr\ %hs\ %{+Q}r\ %{+X}o\ %ci:%cp_%fi:%fp_%Ts_%rt:%pid\ %sslv\ %sslc
```

Dodanie reguł logrotat'a _/etc/logrotate.d/haproxy_

```
/var/log/haproxy*.log {
    daily
    rotate 14
    missingok
    notifempty
    compress
    delaycompress
    postrotate
        invoke-rc.d rsyslog rotate >/dev/null 2>&1 || true
    endscript
}
```

Na koniec reload haproxy i rsyslog'a

```bash
systemctl reload rsyslog
systemctl reload haproxy
```

### Linki

[Blog HAProxy](https://www.haproxy.com/blog/introduction-to-haproxy-logging/).

