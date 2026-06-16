+++
title = "Puppet przejęty przez Perforce - co się zmieniło i jakie są inne alternatywy dla Puppet?"
date = "2025-08-08T05:01:41+02:00"
slug = "puppet-przejety-przez-perforce-co-sie-zmienilo-i-jakie-sa-inne-alternatywy-dla-puppet"
draft = false
tags = []
featureImage = "/content/images/2025/08/puppet-vs-openvox.png"
+++

Puppet przez lata był jednym z filarów automatyzacji i konfiguracji w dużych środowiskach IT. W 2022 roku projekt został przejęty przez firmę **Perforce**, a pod koniec 2024 roku ogłoszono poważne zmiany w sposobie dystrybucji i licencjonowania.

Najważniejsze z nich to:

- od 2025 roku oficjalne **binarki Puppet** są dostępne wyłącznie w prywatnym repozytorium Perforce,
- aby z nich korzystać, trzeba zaakceptować nową licencję,
- darmowa wersja pozwala na zarządzanie maksymalnie **25 węzłami (agentami)** - powyżej tego progu konieczna jest komercyjna umowa,

kod źródłowy pozostaje otwarty (Apache 2.0), ale tempo publicznych aktualizacji jest wolniejsze niż dotychczas.

Limit 25 węzłów może brzmieć jak coś akceptowalnego dla małych środowisk, ale w praktyce nie wystarczy niemal nikomu - nawet niewielkie firmy łatwo przekraczają tę liczbę, licząc serwery produkcyjne, stagingowe, bazy danych czy load balancery.

------------------------------------------------------------------------

## OpenVox - społecznościowa odpowiedź na zmiany

W odpowiedzi na nowe zasady licencjonowania powstał fork o nazwie **OpenVox**, prowadzony przez społeczność **Vox Pupuli**. Projekt ma na celu utrzymanie otwartej, dostępnej dla wszystkich wersji Puppet, wraz z gotowymi binarkami i bez ograniczeń liczby węzłów.

Trzeba jednak pamiętać, że korzystanie z forka wiąże się z pewnym ryzykiem:

- projekt może zostać porzucony w dowolnym momencie,
- tempo rozwoju i wydawania poprawek może być wolniejsze,
- nie ma gwarancji takiej stabilności i testów, jak w produkcie wspieranym przez firmę,
- mogą pojawić się problemy z kompatybilnością w przyszłości,
- liczba dostępnych modułów i integracji może być mniejsza niż w oryginale.

Dla niektórych organizacji OpenVox może być atrakcyjną opcją, ale w większych, krytycznych środowiskach trzeba dokładnie ocenić, czy ryzyko jest akceptowalne.

------------------------------------------------------------------------

## Jakie są inne alternatywy dla Puppet?

Jeżeli Puppet w nowym modelu licencyjnym nie spełnia Twoich wymagań, na rynku wciąż jest kilka dojrzałych narzędzi do automatyzacji konfiguracji:

- **Ansible -** w pełni open source, bez ograniczeń liczby hostów. Bardzo popularny w DevOpsach i CI/CD, konfiguracje pisane w YAML. Jest on agentless, przez co nie jest do konca alternatywą dla puppeta,
- **Chef** - projekt open source w teorii, ale **darmowy tylko w określonych przypadkach**: użytek osobisty, non-profit w określonych ramach, lub organizacje kontrybuujące do Chef OSS. W każdej innej sytuacji wymagana jest komercyjna licencja,
- **SaltStack (Salt)** - otwarte narzędzie przejęte przez VMware, działające w trybie master/minion lub agentless. Wersja OSS nie ma limitów liczby węzłów, wersja komercyjna dodaje GUI i integracje klasy enterprise.

------------------------------------------------------------------------

## Co to oznacza dla nas

Zmiany w Puppet to jasny sygnał, że nawet długoletnie projekty open source mogą w pewnym momencie zmienić swój model biznesowy. Dla większości firm limit 25 węzłów to realna przeszkoda, wymuszająca dodatkowe koszty lub migrację na inne rozwiązanie.

Warto więc już teraz przyjrzeć się alternatywom - czy to w formie forka OpenVox, czy przejścia na inne narzędzia takie jak Ansible, SaltStack czy Chef - i ocenić, które z nich najlepiej odpowiadają na potrzeby naszej infrastruktury.

