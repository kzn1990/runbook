+++
date = 2020-12-08T18:25:32+01:00
description = ""
draft = false
thumbnail = "/content/images/2020/12/docker-filebeat-autodiscover.jpg"
slug = "filebeat-autodiscovery"
title = "Docker - filebeat autodiscovery"
tags = ["docker", "elk", "filebeat"]
+++


Przy pomocy beatów od elasticka i opcji autodiscovery mamy możliwość monitorowania kontenerów z poziomu hosta bez niepotrzebnej ingerencji do środka samych kontenerów. Autodoscoverer podczas startu beat'a skanuje odpalone kontenery i przypisuje im odpowiedni config.

### Konfiguracja filebeata

Do głównej konfiguracji _filebeat.yml_ dodajemy wpis (w tym przypadku dla nginx'a):

```
filebeat.autodiscover:
  providers:
    - type: docker
      templates:
        - condition.contains:
            docker.container.image: nginx
          config:
            - module: nginx
              access:
                input:
                  type: container
                  paths:
                    - "/var/lib/docker/containers/${data.docker.container.id}/*.log"
              error:
                input:
                  type: container
                  paths:
                    - "/var/lib/docker/containers/${data.docker.container.id}/*.log"
```

Filebeat będzie wyszukiwał logów do każdego kontenera zbudowanego z obrazu nginxa i podpinał do niego moduł "nginx".

Parametry, po których możemy wyszukiwać konkretne kontenery, jest oczywiście więcej, oto kilka przykładowych:

```
host
port
docker.container.id
docker.container.name
docker.container.labels
```

Jeżeli używamy **docker-compose** musimy dodać jeszcze wpis o _logging'u_ do configu:

```
nginx:
    image: nginx:alpine
    ports:
      - 80:80
    .
    .some config
    .
    logging:
      driver: "json-file"
      options:
        max-size: 10m
        max-file: "3"
        labels: "production_status"
        env: "os"
```

### Linki

[Autodiscoveredit](https://www.elastic.co/guide/en/beats/filebeat/current/configuration-autodiscover.html#configuration-autodiscover),[Docker-compose logging](https://docs.docker.com/compose/compose-file/#logging).

