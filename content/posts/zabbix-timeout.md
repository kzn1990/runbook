+++
date = 2020-07-11T20:54:05+02:00
description = ""
draft = false
thumbnail = "/content/images/2020/07/zabbix-timeout-1.jpg"
slug = "zabbix-timeout"
title = "Zabbix - timeout executing script"

+++


Dzięki agentowi zabbixa możemy uruchamiać zewnętrzne programy/skrypty i przekazywać ich outputu bezpośrednio do serwera zabbixa. Umożliwia to opcja _UserParameter_.Problem może pojawić się, jeżeli skrypt zwraca output po jakimś dłuższym czasie, wtedy zabbix server może nas przywitać błędem:

> Timeout while executing a shell script.

Problem rozwiązuje zwiększenie parametru _Timeout._ Opcję musimy zmienić zarówno po stronie agenta monitorowanego serwera, jak i po stronie samego zabbix-server'a. W plikach _zabbix-agent.conf_ i _zabbix-server.conf_ ustawiamy parametr timeout na maksymalną wartość 30 sekund:

> Timeout 30

Po edycji plików konfiguracyjnych robimy reload agenta i servera.

### Linki

[UserParameter](https://www.zabbix.com/documentation/current/manual/config/items/userparameters)

