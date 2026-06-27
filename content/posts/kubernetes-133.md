+++
title = "In-place Pod Resize i Sidecary w Kubernetes 1.33 – Co to zmienia?"
date = "2025-07-28T13:18:00+02:00"
slug = "kubernetes-133"
draft = false
tags = ["kubernetes", "k8s"]
featureImage = "/content/images/2025/08/k8s-pod-resize-sidecar.webp"
+++

### TL;DR

Kubernetes 1.33 to kamień milowy, który znacząco podnosi elastyczność i bezpieczeństwo platformy. Kluczowe nowości to stabilne, natywne **kontenery sidecar**, które upraszczają złożone architektury, oraz funkcja **in-place Pod resize**, umożliwiająca zmianę alokacji zasobów (CPU i pamięci) w działających kontenerach bez konieczności ich restartu. Dodatkowo, domyślnie włączone **przestrzenie nazw użytkowników** podnoszą poziom izolacji, czyniąc Kubernetes jeszcze potężniejszym narzędziem dla każdego dewelopera i administratora.

------------------------------------------------------------------------

## 1. Kubernetes 1.33 – Ewolucja, a nie rewolucja

Kubernetes nie przestaje się rozwijać, a każda nowa wersja to krok w stronę doskonalszej orkiestracji kontenerów. Wersja 1.33, najnowsze wydanie, skupia się na **ewolucji platformy**, dostarczając funkcji, na które społeczność czekała od dawna. W artykule przybliżamy dwie najważniejsze nowości, które mogą zmienić sposób projektowania i zarządzania aplikacjami w chmurze.

------------------------------------------------------------------------

## 2. Natywne kontenery sidecar: Prostsze zarządzanie złożonymi architekturami

Wzorzec **sidecar**, polegający na umieszczaniu dodatkowych kontenerów pomocniczych (np. do logowania czy monitorowania) obok głównej aplikacji w tym samym Podzie, zawsze był popularny. Jednak jego implementacja w starszych wersjach Kubernetes bywała problematyczna.

**Jak to działało wcześniej?** Wcześniej, gdy startował Pod, nie było gwarancji kolejności uruchamiania kontenerów. Często zdarzało się, że główny kontener startował przed sidecarem. Prowadziło to do nieprzewidywalnych zachowań i błędów, szczególnie podczas rolowania wdrożeń, ponieważ aplikacja mogła próbować komunikować się z sidecarem, który jeszcze nie był gotowy. Wymagało to stosowania skomplikowanych mechanizmów i skryptów, aby wymusić poprawną kolejność.

**Jak to działa teraz w Kubernetes 1.33?** Dzięki natywnemu wsparciu dla sidecarów, proces został całkowicie zmieniony. Kubernetes implementuje sidecary jako specjalną klasę **`init containers` z `restartPolicy: Always`**. To gwarantuje, że:

1.  Najpierw startują sidecary i muszą być gotowe.
2.  Dopiero później startuje główny kontener aplikacji.
3.  Sidecary działają przez cały cykl życia Podu i kończą działanie dopiero po głównym kontenerze.

![](/content/images/2025/08/sidecar_before_133.webp)

Ta fundamentalna zmiana w cyklu życia Podu eliminuje wcześniejsze problemy i sprawia, że złożone architektury z sidecarami są o wiele bardziej niezawodne i łatwiejsze w utrzymaniu.

------------------------------------------------------------------------

## 3. Zmiana rozmiaru Podów w miejscu (in-place Pod resize): Elastyczność bez przestojów

Dotychczas, zmiana zasobów (CPU lub pamięci RAM) w działającym kontenerze wymagała jego restartu/rolloutu. Było to szczególnie uciążliwe dla **aplikacji stanowych** (sts), gdzie przerwa w działaniu jest niepożądana, lub problematyczna.

W Kubernetes 1.33 ten problem zostaje rozwiązany dzięki funkcji **`in-place Pod resize`**.

### Jak to działa?

Zamiast restartować Pod, możesz teraz dynamicznie zmienić alokację zasobów dla działającego kontenera. Nowa funkcja pozwala na modyfikację pola `spec.containers[*].resources` w specyfikacji Podu, które teraz reprezentuje **pożądane zasoby**.

Aby dokonać takiej zmiany, możesz użyć nowego sub-zasobu `resize` za pomocą `kubectl`:

`kubectl edit pod <pod-name> --subresource resize`

Kubelet na węźle hostującym Pod próbuje dostosować zasoby w locie. Status operacji jest widoczny w `status.containerStatuses[*].resources`, co daje pełny wgląd w postępy.

### Dlaczego `in-place Pod resize` to game-changer?

Ta funkcja ma kluczowe znaczenie, zwłaszcza dla **aplikacji stanowych (StatefulSets)**, gdzie zachowanie ciągłości działania i stanu jest priorytetem. Zmiana zasobów nie wymaga restartu, co jest kluczowe dla aplikacji wrażliwych na przerwy. Oprócz tego:

- **Zwiększona wydajność:** Możesz dostosowywać zasoby do rzeczywistego obciążenia, co prowadzi do oszczędności i lepszego wykorzystania zasobów klastra.
- **Szybsze skalowanie:** Możliwość szybkiego zwiększenia zasobów na start, a następnie ich zmniejszenia (tzw. vertical autoscaling), to ogromna korzyść dla aplikacji o zmiennym profilu obciążenia.

------------------------------------------------------------------------

## 4. Inne istotne zmiany: Domyślne User Namespaces

Kubernetes 1.33 domyślnie włącza **przestrzenie nazw użytkowników (User Namespaces)**. To kolejny ważny krok w kierunku poprawy bezpieczeństwa. Funkcja ta pozwala na uruchamianie procesów wewnątrz kontenera z niższymi uprawnieniami niż te na poziomie systemu hosta. Dzięki temu, nawet w przypadku luki w zabezpieczeniach kontenera, ryzyko eskalacji uprawnień na poziomie systemu jest mniejsz.

------------------------------------------------------------------------

## 5. Podsumowanie

Kubernetes 1.33 dostarcza kluczowych usprawnień, które podnoszą jakość i efektywność pracy z kontenerami. **Stabilne sidecary** upraszczają zarządzanie złożonymi aplikacjami, a **dynamiczna zmiana rozmiaru Podów** daje niezrównaną elastyczność, szczególnie w przypadku aplikacji stanowych (sts).

