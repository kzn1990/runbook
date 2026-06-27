+++
title = "Ghost → Hugo: migracja bloga z K8s na GitHub Pages"
date = "2026-06-22T08:00:00+02:00"
slug = "ghost-to-hugo-migration"
draft = false
tags = ["kubernetes", "hugo", "devops", "self-hosted", "ghost"]
featureImage = "/content/images/2026/06/ghost-hugo-migration.webp"
+++

Pierwszy raz, gdy stawiałem bloga na Kubernetesie, wydawało się to idealnym rozwiązaniem. K8s daje skalowalność, self-healing, deklaratywną konfigurację, GitOps. Do tego dochodzi satysfakcja zbudowania pełnego stosu od zera: klastra, storage'u, monitoringu, CI/CD. To świetny projekt edukacyjny, który uczy wielu rzeczy naraz.

Problem pojawia się dopiero, gdy trzeba to wszystko **utrzymywać na co dzień**. Gdy chcesz dodać jedną usługę, a musisz pisać deployment, service, ingress, PVC. Gdy aktualizacja jednego komponentu ciągnie za sobą łańcuch zmian. Gdy monitoring i storage pochłaniają więcej zasobów niż sama aplikacja.

Dokładnie z takich powodów nastąpiła migracja z Ghosta na Kubernetesie na statyczny generator Hugo z deployem na GitHub Pages.

## Stack „przed" — full-blown K8s

Architektura bloga opartego na Ghostcie w klastrze K8s wyglądała tak:

```
                        K8s Cluster
 +=====================================================+
 |                                                     |
 |              +-------------------+                  |
 |              | HAProxy Ingress   |                  |
 |              +--------+----------+                  |
 |                       |                             |
 |              +--------v----------+                  |
 |              | nginx             |                  |
 |              | (deny /ghost/)    |                  |
 |              +--------+----------+                  |
 |                       |                             |
 |              +--------v----------+                  |
 |              | Varnish           |                  |
 |              | (cache statyczny) |                  |
 |              +--------+----------+                  |
 |                       |                             |
 |         +-------------+-------------+               |
 |         |             |             |               |
 |  +------v------+ +----v----+ +------+------ +       |
 |  |    Ghost     | |  MySQL  | |  cert-mgr   |       |
 |  |   (blog)     | | (baza)  | | + MetalLB   |       |
 |  +--------------+ +---------+ +------+------+       |
 |                                          |          |
 |  +----------+ +----------+ +------------v--------+  |
 |  | ArgoCD   | | OpenEBS  | | VictoriaMetrics     |  |
 |  | (GitOps) | | (storage)| | + Grafana           |  |
 |  +----------+ +----------+ +---------------------+  |
 |                                                     |
 |  Infra: LXD + Terraform + Ansible                   |
 |  Fizyczny serwer: multi-node                        |
 +=====================================================+
```

Kolejność ruchu: **Internet → HAProxy Ingress → nginx → Varnish → Ghost → MySQL**

- **nginx** — robił reverse proxy i blokował miedzy innymi dostęp do `/ghost/` z zewnątrz (panel administracyjny bloga)
- **Varnish** — cache'ował cały front Ghosta. De facto serwowana była statyczna strona HTML
- **Ghost** — generował treści, ale z perspektywy użytkownika był prawie niewidoczny (Varnish serwował cache)
- **MySQL** — baza danych, siedziała w tle na rzadkie aktualizacje

Każdy komponent miał swoje manifesty, swoje PVC, swoje serwisy. Do tego dochodził cały stos pomocniczy:

- **ArgoCD** — GitOps, syncował manifesty z repozytorium
- **OpenEBS** — distributed storage dla jednej bazy MySQL
- **cert-manager + MetalLB** — certyfikaty SSL i load balancing, z wystawionym ingresem HAProxy w sieci zewnetrznej
- **VictoriaMetrics + Grafana** — pełen monitoring dla bloga

### Co było nie tak?

Nie to, że nie działało. Działało. Problem polegał na proporcjach:

- **Varnish cacheował cały front** — Ghost generował treści, Varnish je cacheował, a reszta stosu siedziała w tyle. Cały ten aparat utrzymywał **statyczną stronę HTML**.
- **Złożoność codziennego zarządzania** — nowa wersja Ghosta? Aktualizacja manifestów. Zmiana certyfikatu? Cert-manager. Problem ze storage'm? OpenEBS debugging.
- **Zużycie zasobów** — monitoring, storage, GitOps reconciliation loops — wszystko to pochłaniało zasoby, które mogłyby służyć czemuś innemu.
- **Security** — wycinanie `/ghost/` z zewnątrz, aktualizacje CVE, zarządzanie sekretami.

Budowanie tego pierwszy raz to świetna zabawa. Bawienie się ArgoCD, testowanie jak Varnish cache'uje response, wystawianie MetalLB w trybie L2 — to wszystko dostarcza frajdy i daje ogromną satysfakcję. Ale gdy trzeba to utrzymywać miesiącami, a chcesz po prostu pisać artykuły, złożoność zaczyna przewyższać korzyści.

## Stack „po" — Hugo + GitHub Pages

Nowy setup:

```
        Lokalny komputer
 +------------------------------+
 |  edytor -> markdown          |
 |  hugo server (podgląd local) |
 +--------------+---------------+
                |
                | git push
                v
        GitHub Actions
 +------------------------------+
 |  hugo --gc --minify          |
 |  deploy do GitHub Pages      |
 +--------------+---------------+
                |
                v
     GitHub Pages + Cloudflare
 +------------------------------+
 |  statyczny HTML/CSS/JS       |
 |  CDN, cache, SSL             |
 |  zero roboty                 |
 +------------------------------+
```

**Co zniknęło z bloga:**
- Ghost, MySQL, Varnish, nginx
- VictoriaMetrics, Grafana (dla bloga)
- Zarządzanie sekretami, aktualizacje, monitoring

**ArgoCD, OpenEBS, cert-manager, MetalLB** — dalej działają w klastrze, ale blog już z nich nie korzysta.

**Co zostało:**
- Markdown → commit → deploy = koniec pracy

## Porównanie

| Aspekt | K8s + Ghost | Hugo + Pages |
|---|---|---|
| Koszt utrzymania | Wysoki (kilka godzin/mies.) | Minimalny (kilka min/tydz.) |
| Złożoność | Wiele komponentów, wiele punktów awarii | Jeden statyczny generator |
| Deploy | ArgoCD sync + debug manifestów | `git push` |
| Bezpieczeństwo | Wycinanie paneli, aktualizacje CVE | Brak attack surface (statyczny HTML) |
| Wydajność | Varnish cacheował statykę | Natywnie statyczny |
| Koszt serwera | Fizyczny multi-node K8s | Darmowe (GitHub Pages) |

## Kiedy K8s ma sens dla bloga?

- Do nauki K8s — jak najbardziej. To świetny projekt edukacyjny.
- Do dużych platform z wieloma serwisami — tam HAmigrowanie z K8s ma sens.

## Kiedy nie ma sensu?

- Gdy blog to statyczna strona (a prawie każdy blog jest statyczny)
- Gdy utrzymanie pochłania więcej czasu niż pisanie treści
- Gdy cache'owanie i tak serwuje statykę

## Podsumowanie

Czasami mniej znaczy więcej. Przejście z K8s na statyczny generator to nie regres — to optymalizacja. Mniej komponentów, mniej awarii, mniej roboty = więcej czasu.

Czy to oznacza całkowite pozbycie się K8s z homelabu? Nie. K8s dalej tam jest i dalej obsługuje inne serwisy. Ale zmniejsza się jego udział — nie jest już z automatu domyślnym punktem deployu dla każdej nowej rzeczy. Pojawia się świadoma decyzja: czy ta usługa naprawdę potrzebuje K8s, czy wystarczy coś prostszego?
