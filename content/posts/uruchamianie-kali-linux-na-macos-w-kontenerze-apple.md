+++
title = "Uruchamianie Kali Linux na macOS w kontenerze Apple"
date = "2025-07-08T16:35:00+02:00"
slug = "uruchamianie-kali-linux-na-macos-w-kontenerze-apple"
draft = false
tags = ["kali", "linux"]
featureImage = "/content/images/2025/08/Kali.png"
+++

Dzięki nowej **konteneryzacji Apple** (ogłoszonej podczas WWDC 2025) możesz teraz bezproblemowo uruchomić **Kali Linux** bezpośrednio na swoim Macu z Apple Silicon. To idealne rozwiązanie dla tych, którzy chcą szybko testować narzędzia bezpieczeństwa bez konieczności stawiania pełnej maszyny wirtualnej.

------------------------------------------------------------------------

## 1. Co to jest Apple Container?

Apple Container to nowy framework wprowadzony w **macOS Sequoia**, umożliwiający izolowane uruchamianie dystrybucji Linuksa na sprzęcie Apple Silicon. Działa to podobnie do WSL2 na Windows, ale w pełni zintegrowane z ekosystemem Apple.

**Kluczowe zalety:**

- Lekka wirtualizacja z natywnym wsparciem dla Apple Silicon
- Integracja z narzędziem Homebrew
- Obsługa standardowego formatu kontenerów Docker

------------------------------------------------------------------------

## 2. Przygotowanie środowiska

### Wymagania

- **macOS Sequoia** (15.x) z Apple Silicon (M1/M2/M3)
- Zainstalowany **Homebrew**

### Instalacja kontenerów Apple

Otwórz Terminal

Zainstaluj CLI kontenerów:\
`brew install --cask container`

Uruchom usługę systemową kontenera:\
`container system start`

------------------------------------------------------------------------

## 3. Uruchomienie Kali Linux w kontenerze

Po przygotowaniu frameworka możesz od razu odpalić obraz Kali Linux z DockerHub:\
`container run --rm -it kalilinux/kali-rolling`

**Wyjaśnienie parametrów:**

- `--rm` – usuwa kontener po zamknięciu
- `-it` – interaktywna sesja terminalowa
- `kalilinux/kali-rolling` – oficjalny obraz Kali Linux

### Montowanie woluminów

Aby uzyskać dostęp do plików z macOS wewnątrz kontenera:

    container run --rm -it \
    --volume $(pwd):/mnt \
    --workdir /mnt \
    docker.io/kalilinux/kali-rolling:latest

- `--volume $(pwd):/mnt` – montuje aktualny katalog do `/mnt`
- `--workdir /mnt` – ustawia katalog roboczy w kontenerze

------------------------------------------------------------------------

## 4. Ograniczenia i znane problemy

Chociaż uruchamianie Kali Linux w kontenerze to duży krok naprzód, warto pamiętać o kilku ograniczeniach:

1.  **Tylko Apple Silicon** – kontenery działają wyłącznie na procesorach M1/M2/M3.
2.  **Brak przejścia sprzętowego** – niektóre narzędzia pentestingowe wymagające bezpośredniego dostępu do urządzeń USB czy GPU mogą nie działać.
3.  **Wczesna faza rozwoju** – framework może mieć błędy, które będą eliminowane w kolejnych aktualizacjach macOS Sequoia.

------------------------------------------------------------------------

## 5. Dlaczego warto?

- 🚀 **Szybkie wdrożenie**: kilka poleceń wystarczy, by mieć działający Kali Linux.
- 🔒 **Bezpieczeństwo**: izolacja kontenera minimalizuje ryzyko wpływu na system hosta.
- 💻 **Wydajność**: lekka wirtualizacja daje niemal natywne osiągi.
- 🔄 **Aktualność**: korzystasz z najnowszego obrazu `kali-rolling`.

------------------------------------------------------------------------

## 6. Podsumowanie

Uruchomienie **Kali Linux w kontenerze na macOS** to prosty sposób na rozszerzenie swojego środowiska pentestowego bez konieczności instalowania pełnej maszyny wirtualnej. Dzięki nowemu frameworkowi Apple Container na Sequoia, użytkownicy Apple Silicon zyskują dostęp do pełnej gamy narzędzi Kali w zaledwie kilka minut.

