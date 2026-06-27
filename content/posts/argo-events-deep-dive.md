+++
title = "Argo Events — event-driven automation w Kubernetesie, którego nikt nie zna"
date = "2026-06-24T08:00:00+02:00"
slug = "argo-events-deep-dive"
draft = false
tags = ["kubernetes", "argo", "devops", "event-driven", "automation"]
featureImage = "/content/images/2026/06/argo-events-deep-dive.webp"
+++

Argo CD zna każdy. Argo Workflows — większość. Argo Events? Mało kto. A szkoda, bo to brakujące ogniwo w ekosystemie Argo, które pozwala zamienić Kubernetes w maszynę event-driven.

## Czym jest Argo Events?

Argo Events to framework do automatyzacji workflow'ów w Kubernetesie na podstawie zdarzeń. Coś się dzieje (webhook, S3, Kafka, cron, zmiana zasobu w K8s) → Argo Events to łapie i wyzwala akcję.

```
  Event Source          Event Bus           Sensor            Trigger
 +-------------+      +----------+      +-----------+      +----------+
 | Webhook     | ---> |          | ---> | Filter    | ---> | Argo WF  |
 | S3          | ---> |  NATS    | ---> | Conditions| ---> | K8s Job  |
 | Kafka       | ---> |          | ---> | Parameters| ---> | HTTP     |
 | K8s Resource| ---> |          | ---> |           | ---> | Slack    |
 +-------------+      +----------+      +-----------+      +----------+
```

## Trzy filary

**EventSource** — definiuje źródło zdarzeń. Najpopularniejsze: Webhook, S3, Kafka, Calendar (cron), K8s Resource, Git. Pełna lista 20+ źródeł w [dokumentacji](https://argoproj.github.io/argo-events/eventsources/).

**EventBus** — warstwa transportowa między EventSource a Sensorem. Domyślnie używa NATS — lekkiego brokera komunikatów open-source (CNCF, graduated). NATS trzyma zdarzenia w pamięci i przekazuje je dalej. Można też użyć JetStream (dodatek do NATS) dla trwałych kolejek z persistencją — wtedy zdarzenia przetrwają restart EventBusa. Alternatywnie, jeśli w klastrze jest już Kafka, można użyć jej jako transportu.

**Sensor** — logika: filtracja, warunki (AND/OR na wielu zdarzeniach), parametryzacja, trigger.

## Przykład: Webhook → Argo Workflow

Najprostszy przypadek. Wysyłamy request POST na webhook → uruchamia się workflow.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: EventSource
metadata:
  name: webhook
spec:
  service:
    ports:
      - port: 12000
        targetPort: 12000
  webhook:
    example:
      port: "12000"
      endpoint: /example
      method: POST
```

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Sensor
metadata:
  name: webhook
spec:
  dependencies:
    - name: example-dep
      eventSourceName: webhook
      eventName: example
  triggers:
    - template:
        name: workflow-trigger
        argoWorkflow:
          operation: submit
          source:
            resource:
              apiVersion: argoproj.io/v1alpha1
              kind: Workflow
              metadata:
                generateName: webhook-workflow-
              spec:
                entrypoint: main
                templates:
                  - name: main
                    container:
                      image: alpine:3.18
                      command: [echo]
                      args: ["Triggered by webhook!"]
```

Test:
```bash
curl -X POST http://webhook-eventsource-svc.argo-events:12000/example
```

## Real-world przykład: monitoring backupów

Jedno z praktycznych zastosowań: śledzenie backupów w klastrze K8s i nie tylko.

Problem z backupami w Kubernetesie jest taki, że każdy operator backupu (Velero, Kasten, Trilio, Percona XtraDB Cluster, PostgreSQL operator, itp.) ma swój własny sposób raportowania. Nie zawsze expose'uje metryki Prometheusowe, nie zawsze da się przerobić skrypt. W legacy systemach (cron joby, skrypty bashowe) jest łatwiej — można dodać push metryki do Pushgateway w jednej linijce. Ale w K8s z operatorami to często wymaga dodatkowej warstwy kodu.

Argo Events rozwiązuje ten problem elegancko: nasłuchuje zmian zasobów (K8s Resource EventSource) i pushuje metryki niezależnie od tego, skąd backup pochodzi. Nie trzeba modyfikować żadnego operatora ani dodawać żadnych dodatkowych komponentów do podów.

EventSource nasłuchuje zmian na Jobach (np. backupy bazy danych):
```yaml
apiVersion: argoproj.io/v1alpha1
kind: EventSource
metadata:
  name: job-events
  namespace: argo-events
spec:
  eventBusName: default
  template:
    serviceAccountName: argo-events-sa
  resource:
    backupJobEvents:
      namespace: backups
      group: batch
      version: v1
      resource: jobs
      eventTypes:
        - UPDATE
```

Sensor filtruje zdarzenia i pushuje metryki do systemu monitoringu:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Sensor
metadata:
  name: backup-metrics
  namespace: argo-events
spec:
  eventBusName: default
  template:
    serviceAccountName: argo-events-sa
  dependencies:
    - name: job-change
      eventSourceName: job-events
      eventName: backupJobEvents
      rateLimit:
        unit: Second
        rate: 1
  triggers:
    - template:
        name: push-metrics
        http:
          url: "http://metrics-collector.monitoring:9090/api/put"
          method: POST
          headers:
            Content-Type: "application/json"
          payload:
            - src:
                dependencyName: job-change
                value: "backup.timeline"
              dest: metric
            - src:
                dependencyName: job-change
                dataKey: body.status.succeeded
                value: "0"
              dest: value
            - src:
                dependencyName: job-change
                value: "backup-{{ .Values.env }}"
              dest: tags.backup_name
            - src:
                dependencyName: job-change
                dataKey: body.metadata.name
              dest: tags._backup_id
```

W taki sposób gromadzi się informacje o każdym backupie w systemie — niezależnie czy pochodzi z K8s czy z legacy systemów. Wszystko trafia na jeden dashboard w Grafanie: kiedy backup się zaczyna, kiedy kończy i czy kończy się niepowodzeniem.

## Kiedy używać zamiast workaroundów?

W alternatywnym podejściu trzeba by budować dodatkowe komponenty: webhooki w aplikacjach, custom workery, ręczne skrypty pushujące metryki. To działa, ale wymaga modyfikacji każdego operatora lub aplikacji osobno.

Argo Events daje natywny sposób w Kubernetesie: definiujesz źródło zdarzeń, logikę filtrowania i trigger — i masz event-driven automation bez dotykania kodu aplikacji.

| Cecha | Argo Events | Workaroundy |
|---|---|---|
| Modyfikacja aplikacji | Nie | Tak (webhook/skrypt) |
| Wiele źródeł zdarzeń | 20+ natywnych | Własna implementacja |
| Filtry i warunki | Deklaratywne (YAML) | Kod w aplikacji |
| CloudEvents | Tak | Nie |
| Złożoność operacyjna | Średnia (1 CRD) | Wysoka (per aplikacja) |

## Podsumowanie

Argo Events rozwiązuje realny problem: jak reagować na zdarzenia w Kubernetesie bez pisania własnych controllerów i bez modyfikowania istniejących aplikacji. Nie trzeba od razu wdrażać wszystkiego — wystarczy zacząć od jednego prostego use case'a (np. webhook → workflow) i rozbudowywać w miarę potrzeb.
