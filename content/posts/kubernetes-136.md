+++
title = "Kubernetes 1.36 – co trzeba wiedzieć przed aktualizacją klastra"
date = "2026-06-22T06:00:00+02:00"
slug = "kubernetes-136"
draft = false
tags = ["kubernetes", "k8s", "devops"]
featureImage = "/content/images/2026/06/k8s-136-cover.webp"
+++

Kubernetes 1.36 („Haru”) przynosi kilka istotnych zmian, szczególnie w kontekście bezpieczeństwa i zarządzania złożonymi zasobami. Poniżej krótkie podsumowanie tego, co warto wiedzieć przed aktualizacją.

### 1. Koniec z `gitRepo` (Breaking Change!)
Wolumen `gitRepo`, który umożliwiał automatyczne klonowanie repozytoriów do kontenera, został **permanentnie wyłączony**. Nie ma opcji włączenia go z powrotem flagą.

**Jak to wyglądało wcześniej:**
```yaml
volumes:
- name: repo
  gitRepo:
    repository: "https://github.com/moje/repo.git"
```

**Co zamiast tego?**
Najczęściej stosowanym rozwiązaniem jest użycie **init containera**, który wykonuje `git clone` do współdzielonego wolumenu:
```yaml
initContainers:
- name: git-clone
  image: alpine/git
  command: ['git', 'clone', 'https://github.com/moje/repo.git', '/workspace']
  volumeMounts:
  - name: shared-data
    mountPath: /workspace
containers:
- name: app
  image: my-app
  volumeMounts:
  - name: shared-data
    mountPath: /workspace
```

### 2. DRA (Dynamic Resource Allocation) - GA
DRA oficjalnie wchodzi do **General Availability**. Umożliwia precyzyjne żądanie złożonych zasobów sprzętowych (np. GPU z określonymi parametrami) bez konieczności stosowania work-aroundów z poziomu node'ów.

**Przykład żądania dwóch GPU:**
```yaml
resources:
  claims:
  - name: gpu
    resourceClassName: "nvidia.com/gpu"
    count: 2
```

### 3. Mixed Version Proxy -> Beta
Funkcja ta jest teraz w **Beta** i domyślnie włączona. Umożliwia ona węzłowi (Kubelet) z nowszą wersją na transparentne przekierowywanie żądań do starszych API serverów. Jest to szczególnie przydatne w dużych klastrach podczas procesu rolling update.

**Jak to działa w praktyce?**
Załóżmy, że mamy klaster z kilkoma API serverami. Podczas aktualizacji, jeden z nich może już działać na wersji 1.36, a pozostałe jeszcze na 1.35. Dzięki Mixed Version Proxy, Kubelet w wersji 1.36 może przekierować żądanie do starszego API (1.35), które następnie przekazuje je do nowszego, jeśli wymaga tego logika żądania. Dzięki temu nie jest wymagana jednoczesna aktualizacja wszystkich komponentów.

### 4. Service ExternalIPs - Deprecated
Pole `spec.externalIPs` w serwisach zostało oficjalnie deprecated. Wersja 1.43 przewiduje jego całkowite usunięcie. Alternatywą może być użycie LoadBalancer z external-dns lub MetalLB.

**Jak to wyglądało wcześniej:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: ClusterIP
  externalIPs:
  - 192.0.2.1
```

