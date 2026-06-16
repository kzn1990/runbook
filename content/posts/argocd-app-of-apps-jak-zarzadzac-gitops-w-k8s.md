+++
title = "ArgoCD App-of-Apps: Jak efektywnie zarządzać aplikacjami w Kubernetes?"
date = "2025-08-01T14:42:00+02:00"
slug = "argocd-app-of-apps-jak-zarzadzac-gitops-w-k8s"
draft = false
tags = ["k8s", "argocd", "devops", "git", "gitops"]
featureImage = "/content/images/2025/08/argocd-app-of-apps.png"
+++

W świecie **Kubernetes**, zarządzanie rosnącą liczbą aplikacji i ich konfiguracjami staje się wyzwaniem. Tradycyjne podejścia często prowadzą do powielania konfiguracji i trudności w utrzymaniu spójności w różnych środowiskach (**staging** i **production**). Właśnie tutaj z pomocą przychodzi wzorzec **App-of-Apps** (aplikacja aplikacji) w **ArgoCD**, który pozwala na **skalowalne zarządzanie** całym ekosystemem aplikacji z jednego, centralnego miejsca.

Wzorzec **ArgoCD App-of-Apps** polega na tym, że jedna, główna aplikacja (*parent application*) zarządza cyklem życia wielu innych aplikacji (*child applications*). Zamiast konfigurować każdą aplikację z osobna, definiujemy je wszystkie w jednej, nadrzędnej aplikacji, która następnie synchronizuje i monitoruje ich stan. To kluczowy element podejścia **GitOps z ArgoCD**.

------------------------------------------------------------------------

## Architektura i struktura katalogów dla **GitOps z ArgoCD**

Aby w pełni wykorzystać wzorzec App-of-Apps, kluczowa jest przemyślana struktura katalogów w **repozytorium Git**. Umożliwia ona spójne definiowanie konfiguracji dla różnych środowisk.

    .
    ├── app-of-apps
    │   ├── values-production.yaml  # Globalne wartości dla produkcji
    │   └── values-staging.yaml     # Globalne wartości dla stagingu
    ├── apps
    │   ├── cert-manager
    │   │   ├── Chart.yaml          # Definicja Helm chart dla cert-manager
    │   │   ├── values-production.yaml # Wartości dla produkcji
    │   │   └── values-staging.yaml    # Wartości dla stagingu
    │   └── fluent-bit
    │       ├── Chart.yaml          # Definicja Helm chart dla fluent-bit
    │       ├── values-production.yaml # Wartości dla produkcji
    │       └── values-staging.yaml    # Wartości dla stagingu

W tej strukturze:

- **`app-of-apps`**: Katalog zawiera główne pliki `values.yaml` dla każdego środowiska. Te pliki kontrolują, które aplikacje i z jakimi parametrami mają zostać wdrożone w danym środowisku.
- **`apps`**: Katalog, w którym znajdują się definicje (w moim przypadku ***Helm Charts***) każdej z aplikacji (*child applications*). Każda aplikacja ma swój własny podkatalog z plikiem `Chart.yaml` i specyficznymi plikami `values.yaml` dla różnych środowisk.

------------------------------------------------------------------------

## Konfiguracja Helm Chart jako zależności

W naszym przykładzie, **cert-manager** jest wdrażany jako **Helm Chart**, ale zamiast kopiować cały kod, definiujemy go jako **zależność** (dependencies) w pliku `Chart.yaml`:

    apiVersion: v2
    name: cert-manager
    description: Helm chart for cert-manager
    type: application
    version: 0.1.0
    appVersion: 1.0.0
    dependencies:
      - name: cert-manager
        alias: certmanager
        version: 1.16.2
        repository: https://charts.jetstack.io

Dzięki temu podejściu, nasza główna aplikacja **ArgoCD** instaluje `cert-managera` z oficjalnego repozytorium Jetstack. My natomiast, w pliku `values-production.yaml`, możemy łatwo dostosować jego konfigurację za pomocą aliasu `certmanager`:

    certmanager:
      namespace: cert-manager
      crds:
        enabled: true
      replicaCount: 2
      podDisruptionBudget:
        enabled: true

To bardzo elastyczne **zarządzanie aplikacjami**, ponieważ pozwala na dostosowanie konfiguracji zewnętrznego charta bez modyfikowania go bezpośrednio, a jednocześnie utrzymanie wszystkich specyficznych dla środowiska parametrów w jednym miejscu.

------------------------------------------------------------------------

## Podsumowanie i korzyści

Używanie wzorca **App-of-Apps** w **ArgoCD**, w połączeniu z dobrze zorganizowaną strukturą katalogów, przynosi liczne korzyści dla Twojego **zarządzania infrastrukturą**:

- **Standaryzacja**: Konfiguracje dla różnych środowisk są spójne i łatwe do zarządzania.
- **Łatwość zarządzania**: Zmiany wprowadzane są w jednym miejscu, co minimalizuje błędy.
- **Skalowalność**: Dodawanie nowych aplikacji jest proste.
- **Automatyzacja**: ArgoCD automatycznie wykrywa i wdraża zmiany, zapewniając spójność stanu klastra.

