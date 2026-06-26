+++
title = "Falco w Kubernetes: runtime security, który naprawdę działa"
date = "2026-06-23T21:30:00+02:00"
slug = "falco-kubernetes-runtime-security"
draft = false
tags = ["kubernetes", "falco", "security", "devops", "sre"]
featureImage = "/content/images/2026/06/falco-kubernetes-runtime-security.png"
+++

## Wstęp

Falco to runtime security monitoring dla Kubernetes. Analizuje system calls w czasie rzeczywistym i alertuje o podejrzanych zachowaniach: nieautoryzowane połączenia sieciowe, dostęp do poufnych plików, nieoczekiwane procesy. Dla mnie to domyślny element każdego nowego klastra K8s — obok monitoringu i logowania.

W tym wpisie: jak wdrożyć Falco w dużym klastrze, jak zintegrować ze Slackiem, jak napisać własną regułę i jak okiełznać domyślne reguły, które w dużej infrastrukturze potrafią być bardzo głośne.

------------------------------------------------------------------------

## Czym jest Falco?

Falco to CNCF project (graduated), który przechwytuje system calls z jądra Linux i porównuje je z regułami bezpieczeństwa. Działa jako DaemonSet w K8s — jeden agent na każdym węźle.

Co wykrywa:

- **Nieautoryzowane połączenia sieciowe** — kontener kontaktujący się z API serverem bez powodu
- **Dostęp do poufnych plików** — odczyt `/etc/shadow`, `/etc/kubernetes/ssl/` przez kontener
- **Nieoczekiwane procesy** — shell uruchomiony w kontenerze, który go nie potrzebuje
- **Modyfikacja logów** — ktoś czyści `/var/log/` z wnętrza kontenera
- **Pakiety surowe** — tworzenie socketów przez kontenery, które tego nie robią

------------------------------------------------------------------------

## Instalacja

Helm chart z oficjalnego repozytorium:

    helm repo add falcosecurity https://falcosecurity.github.io/charts
    helm repo update
    helm install falco falcosecurity/falco --namespace falco --create-namespace

### Struktura wartości

Przy wielu klastrach warto użyć wzorca z plikami wartości:

```
falco/
├── values-shared.yaml          # wspólne dla wszystkich klastrów
├── values-production-1.yaml    # overrides dla prod-1
├── values-production-2.yaml    # overrides dla prod-2
└── values-staging.yaml         # staging
```

Helm automatycznie merguje pliki wartości. Kluczowe ustawienie w `values-shared.yaml`:

    falcosidekick:
      enabled: true
      config:
        slack:
          webhookurl: "https://hooks.slack.com/services/T.../B.../xxx"
          channel: "#security-k8s"
          minimumpriority: "warning"

    customRules:
      rules-custom-rules.yaml: |-
        # tutaj własne reguły i exceptiony

### ArgoCD App-of-Apps

Jeśli używasz ArgoCD, Falco idealnie wpada w wzorzec app-of-apps:

```
apps/
└── falco/
    ├── Chart.yaml
    ├── values-shared.yaml
    └── values-production-1.yaml
```

W `Chart.yaml` jako dependency:

    dependencies:
      - name: falco
        version: 4.20.1
        repository: https://falcosecurity.github.io/charts

ArgoCD synchronizuje automatycznie. Zmiana wartości = redeploy.

------------------------------------------------------------------------

## Integracja ze Slackiem

Falcosidekick to sidecar, który konwertuje alerty Falco na webhooki. Włączenie:

    falcosidekick:
      enabled: true

    falcosidekick-ui:
      enabled: true    # web UI do przeglądania alertów

Konfiguracja Slack:

    falcosidekick:
      config:
        slack:
          webhookurl: "https://hooks.slack.com/services/xxx"
          channel: "#security-k8s"
          minimumpriority: "warning"
          format: "fields"
          messageformat: "Alert: {{ .Rule }}"

Efekt: każde naruszenie reguły trafia na Slacka z czytelnym formatowaniem. Minimum priority `warning` filtruje szum.

------------------------------------------------------------------------

## Problem: domyślne reguły są głośne

Falco przyjeżdża z domyślnym zestawem reguł: [falco_rules.yaml](https://github.com/falcosecurity/rules/blob/ec255e68f4cc05ce91d641e36f1855b4435556d5/rules/falco_rules.yaml).

W małym klastrze — kilka alertów dziennie. W dużym klastrze z wieloma operatorami — **setki false positive'ów**.

### Najgłośniejsza reguła: Contact K8S API Server

Reguła wykrywa każde połączenie kontenera z API serverem. Problem: w produkcyjnym klastrze **sporo kontenerów to operatory**, które muszą kontaktować się z API. Przy kilkunastu operatorach masz kilkanaście alertów na minutę.

### Rozwiązanie: exceptiony z audytu

Nie wyłączaj reguły. Dodaj exceptiony oparte na audycie ruchu:

    - rule: Contact K8S API Server From Container
      exceptions:
        - name: known_operators
          fields: [container.image.repository]
          comps: [in]
          values:
            - [quay.io/argoproj/argocd, docker.io/metallb/speaker,
               docker.io/velero/velero, clickhouse/clickhouse-operator,
               ghcr.io/cloudnative-pg/postgresql, docker.io/bitnami/sealed-secrets,
               docker.io/externaldns/externaldns, docker.io/jetstack/cert-manager,
               docker.io/victoriametrics/operator, docker.io/openebs/provisioner]

Każdy exception = konkretny obraz, który wiemy, że musi kontaktować się z API. Reszta dalej alertuje.

### Inne głośne reguły

| Reguła | Problem | Rozwiązanie |
|--------|---------|-------------|
| Redirect STDOUT/STDIN to Network | Operatory baz danych | Exception po `container.name` |
| Read sensitive file untrusted | Host processes | Exception po `container.id = host` |
| Packet socket created | Network operator | Exception po `container.name` |
| Run shell untrusted | Database operator | Exception po `container.image.repository` |
| Clear Log Activities | Legitimate log rotation | Exception po `container.id` |

------------------------------------------------------------------------

## Własna reguła

Domyślne reguły pokrywają 80% przypadków. Ale czasem potrzebujesz czegoś specyficznego.

### Przykład: wykryj dostęp do secrets K8s

```yaml
- rule: Access to K8s Secrets from Non-System Container
  desc: Detect access to Kubernetes secrets from unexpected containers
  condition: >
    open_read and
    container and
    fd.name startswith /var/run/secrets/kubernetes.io/ and
    not container.image.repository in (system_images)
  output: >
    Suspicious secret access
    (user=%user.name command=%proc.cmdline container=%container.name
    image=%container.image.repository file=%fd.name)
  priority: WARNING
  tags: [k8s, secrets, access]
```

### Struktura reguły

    - rule: nazwa           # unikalna, czytelna
      desc: opis            # co wykrywa
      condition: warunek    # kiedy alertować (macro language)
      output: format        # co wyrzucić na slacka
      priority: WARNING     # DEBUG/INFO/WARNING/ERROR/CRITICAL
      tags: [tag1, tag2]    # do filtrowania

### Makra i warunki

Falco ma bogaty język makr:

    container                                    # w kontenerze
    proc.name = nginx                            # proces nginx
    fd.name startswith /etc/                     # dostęp do /etc/
    container.image.repository in (img1, img2)   # z tych obrazów
    spawned                                      # nowy proces
    open_read                                    # otwarcie do odczytu

Można łączyć `and`, `or`, `not`.

------------------------------------------------------------------------

## Shared values dla wielu klastrów

Przy wielu klastrach K8s nie kopiuj konfiguracji. Użyj wzorca:

```yaml
# values-shared.yaml — wspólne dla wszystkich
falcosidekick:
  enabled: true
  config:
    slack:
      webhookurl: "${SLACK_WEBHOOK}"

customRules:
  rules-exceptions.yaml: |
    # exceptiony wspólne dla wszystkich klastrów
    - rule: Contact K8S API Server From Container
      exceptions:
        - name: shared_operators
          fields: [container.image.repository]
          comps: [in]
          values:
            - [argoproj/argocd, metallb/speaker, ...]
```

```yaml
# values-production-1.yaml — tylko override
falco:
  priority: warning

customRules:
  rules-prod1-overrides.yaml: |
    # exceptiony specyficzne dla tego klastra
    - rule: Contact K8S API Server From Container
      exceptions:
        - name: prod1_specific
          fields: [container.image.repository]
          comps: [in]
          values:
            - [specific/operator-for-prod1]
```

Helm merguje automatycznie. Kluczowe: użyj `exceptions: append` w override'ach, żeby exceptiony się łączyły, nie nadpisywały.

------------------------------------------------------------------------

## Podsumowanie

Falco to genialny soft, który znacząco podnosi security klastra K8s. Dla mnie to domyślny element każdego nowego klastra — obok monitoringu, logowania i GitOps.

Kluczowe punkty:

- **Runtime security** — wykrywa anomalie w czasie rzeczywistym, nie tylko na etapie builda
- **Domyślne reguły** — świetna baza, ale w dużym klastrze wymagają exceptionów
- **Operatorzy** — najczęstsze źródło szumu. Audytuj ruch i dodawaj exceptiony
- **Slack integration** — natychmiastowe alerty, zero konfiguracji
- **Shared values** — jeden plik wspólnych exceptionów dla wielu klastrów
- **Własne reguły** — prosty język makr, łatwo dopasować do specyfiki infrastruktury

Instalacja: 10 minut. Konfiguracja exceptionów: kilka dni. Spokój ducha: bezcenne.

------------------------------------------------------------------------

## Linki

- [Falco — oficjalna dokumentacja](https://falco.org/docs/)
- [Falco Rules — domyślne reguły](https://github.com/falcosecurity/rules/blob/ec255e68f4cc05ce91d641e36f1855b4435556d5/rules/falco_rules.yaml)
- [Falcosidekick — integracje](https://github.com/falcosecurity/falcosidekick)
- [Helm Chart — falcosecurity](https://github.com/falcosecurity/charts)
