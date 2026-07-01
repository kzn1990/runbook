+++
title = "HAProxy + CAPTCHA: Bot protection bez Enterprise Edition"
date = "2026-06-28T18:00:00+02:00"
slug = "haproxy-captcha-bot-protection"
draft = false
tags = ["haproxy", "security", "captcha", "ddos", "lua"]
featureImage = "/content/images/2026/06/haproxy-captcha-bot-protection.webp"
+++

HAProxy Enterprise ma gotowy moduł Bot Management z CAPTCHA. Ale tę samą funkcjonalność można wdrożyć open-source, bezpośrednio w HAProxy, bez oddzielnych aplikacji w tle. Jak to zrobić i z jakimi problemami się spotkałem - o tym poniżej.

## Co to jest haproxy-protection

Projekt [haproxy-protection](https://github.com/nayumiDEV/haproxy-protection) (fork Basedflare) to zestaw skryptów Lua + map files implementujących Proof-of-Work i CAPTCHA challenge bezpośrednio w HAProxy. Nie potrzeba Redis, Memcache, ani oddzielnego serwisu - wszystko działa wewnątrz proxy.

```
+-------------------+
|    User Request   |
+--------+----------+
         |
         v
+--------+----------+
|   HAProxy L7      |
|   (Lua scripts)   |
+--------+----------+
         |
    [PoW/CAPTCHA check]
         |
         v
+--------+----------+
|   Backend App     |
+-------------------+
```

## Dwa tryby ochrony

### Proof-of-Work (PoW) - automatyczny challenge

Przeglądarka rozwiązuje w tle zadanie matematyczne (SHA256 lub argon2) przez kilkaset milisekund. User widzi stronę z animowanymi kropkami i licznikiem postępu, ale nie musi nic klikać - po rozwiązaniu zadania automatycznie trafia na docelową stronę. Boty bez JavaScript nie potrafią tego rozwiązać (albo kosztuje je to za dużo CPU).

**Zalety:**
- Automatyczne, nie wymaga interakcji usera
- Działa offline, bez zewnętrznych zależności
- Wystarczy na większość przypadków

### CAPTCHA (hCaptcha) - manualna weryfikacja

Widget hCaptcha z checkboxem „I'm not a robot", który wymaga ręcznego kliknięcia.

**Kiedy używać:**
- Dla wrażliwych endpointów (`/login`, `/register`, `/contact-form`)
- Gdy PoW nie wystarczy (uporczywe boty)
- Dla podejrzanych IP (przekroczenie rate limit)

**hCaptcha - darmowy tier:**
- Do 1M zapytań/miesiąc (~30k/dzień)
- Wystarczy na większość serwisów
- Rejestracja na hcaptcha.com, dostajesz Sitekey + Secret

## Konfiguracja per path

Kluczowy plik to `ddos.map` - lookup per host + path:

```
# Cała domena - CAPTCHA
example.com {"m":2}
example.com:8080 {"m":2}

# API - bez challenge'u
example.com/api {"m":0}
example.com:8080/api {"m":0}

# Login - CAPTCHA
example.com/login {"m":2}
example.com:8080/login {"m":2}
```

**Tryby (`m`):**

| Wartość | Znaczenie |
|---------|-----------|
| `m:0` | Brak challenge'u, puszcza do backendu |
| `m:1` | Tylko PoW (automatyczny, bez interakcji usera) |
| `m:2` | PoW + CAPTCHA (manualne kliknięcie) |

Lua sprawdza najpierw `host+path`, potem `host` - można mieć różne ustawienia dla `/api`, `/login`, `/checkout`.

## Stateless cookie - multi-LB bez Redis

To jest kluczowa zaleta tego rozwiązania. Cookie `_basedflare_pow` zawiera wszystko co potrzebne do weryfikacji:

```
user_key#challenge_hash#expiry#answer#hmac_signature
```

HAProxy weryfikuje HMAC-SHA3_256 podpis - **nie ma żadnego wspólnego storage** (Redis, Memcache, DB) między load balancerami.

**Warunek konieczny dla multi-LB:** identyczne sekrety na wszystkich instancjach:
- `HMAC_COOKIE_SECRET`
- `POW_COOKIE_SECRET`
- `CAPTCHA_COOKIE_SECRET`

Reszta konfiguracji (`ddos.map`, `hosts.map`, `POW_TYPE`, `POW_DIFFICULTY`) też musi być taka sama.

**Uwaga na `CHALLENGE_INCLUDES_IP`:**
Domyślnie `true` - cookie jest podpięte pod IP klienta. Jeśli klient ma rotację IP (CDN, VPN), każde IP dostanie nowy challenge. Rozwiązanie: `CHALLENGE_INCLUDES_IP=false` lub LB z proxy protocol zachowującym oryginalne IP.

## Praktyczne wdrożenie - problemy i rozwiązania

### Problem 1: Argon2 C binding crash (exit code 134)

HAProxy crashował z SIGABRT przy każdym requeście challenge'u. Custom C binding `argon2_wrap.c` działał w standalone `lua5.3`, ale crashował w środowisku HAProxy Lua runtime.

**Rozwiązanie:** Przejście na PoW SHA256 (`POW_TYPE=sha256`) - używa wbudowanej funkcji `sha.sha256()` w Lua, zero C bindingów.

**Problem:** Na HTTP przeglądarka nie pozwala na użycie `crypto.subtle.digest()` (wymaga secure context). PoW po prostu wisiał w nieskończoność.

**Rozwiązanie:** Od razu testować na HTTPS (self-signed cert wystarczy na początek). Po `crypto.subtle` działa, SHA256 przechodzi bez problemów.

### Problem 2: Dynamiczne serwery websrv3/websrv4 (503)

Skrypt `register-servers.lua` dodawał przy starcie serwery dynamiczne na podstawie `hosts.map`. Dla `example.com` tworzył `websrv3` i `websrv4` z pustym adresem (`srv_addr=-`). HAProxy przez `balance leastconn` kierował na nie ruch, dostawał timeout 3s i 503.

**Efekt:** Losowo 404 (dobry backend) lub 503 (zły backend).

**Rozwiązanie:** Ustawienie statycznych backendów w konfiguracji HAProxy zamiast dynamicznej rejestracji. Skrypt `register-servers.lua` nie jest potrzebny - wystarczą zdefiniowane serwery w sekcji `backend`.

## Konfiguracja końcowa

```yaml
# docker-compose.yml
services:
  haproxy:
    ports:
      - "8080:80"
      - "443:443"
    environment:
      - POW_TYPE=sha256
      - POW_DIFFICULTY=20
      - CHALLENGE_EXPIRY=43200  # 12h
      - CHALLENGE_INCLUDES_IP=true
      - HCAPTCHA_SITEKEY=20000000-ffff-ffff-ffff-000000000002  # test keys
      - HCAPTCHA_SECRET=0x0000000000000000000000000000000000000000
      - HMAC_COOKIE_SECRET=***  # identyczny na wszystkich LB
      - POW_COOKIE_SECRET=***
      - CAPTCHA_COOKIE_SECRET=***
```

```haproxy
# haproxy.cfg
frontend http-in
    bind *:80
    bind *:443 ssl crt /etc/haproxy/certs/haproxy.pem alpn h2,http/1.1
    
    # Sprawdź czy host jest w ddos.map
    acl ddos_mode_enabled hdr(host),lower,map(/etc/haproxy/map/ddos.map) -m found
    
    # Sprawdź PoW/CAPTCHA
    http-request lua.decide-checks-necessary if !is_excluded !on_bot_check ddos_mode_enabled
    http-request lua.captcha-check if !is_excluded !on_bot_check validate_captcha
    http-request lua.pow-check if !is_excluded !on_bot_check validate_pow
    
    # Redirect na bot-check jeśli trzeba
    http-request redirect location /.basedflare/bot-check?%[capture.req.uri] code 302 \
        if validate_captcha !captcha_passed !on_bot_check ddos_mode_enabled !is_excluded \
        OR validate_pow !pow_passed !on_bot_check ddos_mode_enabled !is_excluded
```

## Co można osiągnąć

**Scenariusz 1: Ochrona całej domeny**
```
example.com {"m":1}  # PoW dla wszystkich
```
Każdy visitor dostaje automatyczny PoW challenge (strona z animowanymi kropkami, rozwiązywana przez JS). Boty bez JS są blokowane.

**Scenariusz 2: Ochrona wrażliwych endpointów**
```
example.com {"m":0}        # Reszta bez challenge'u
example.com/login {"m":2}  # Login z CAPTCHA
example.com/register {"m":2}  # Rejestracja z CAPTCHA
```
Tylko `/login` i `/register` wymagają CAPTCHA. Reszta serwisu działa normalnie.

**Scenariusz 3: Rate-based protection**
Można skonfigurować automatyczne włączanie PoW/CAPTCHA po przekroczeniu progu requestów per IP (np. >30 req/10s → PoW, >100 req/10s → CAPTCHA). Wymaga dodatkowej konfiguracji stick-table w HAProxy.

## Podsumowanie

haproxy-protection pozwala wdrożyć bot protection na HAProxy open-source, bez Enterprise Edition. Kluczowe zalety:

- **Stateless cookie** - nie potrzeba Redis/Memcache, działa na wielu LB
- **Per-path konfiguracja** - różne tryby dla różnych endpointów
- **PoW + CAPTCHA** - elastyczność: automatyczne PoW (strona z kropkami) dla każdego przy pierwszym połączeniu, CAPTCHA tylko przy podejrzanym ruchu
- **hCaptcha free** - do 1M zapytań/miesiąc za darmo

Typowy scenariusz: PoW SHA256 dla każdego usera przy pierwszym połączeniu (niewielkie opóźnienie, ale skuteczne przeciwko botom bez JS), CAPTCHA tylko gdy IP przekracza rate limit lub podejrzane zachowanie. Można to fajnie połączyć z Coraza SPOA (Web Application Firewall) dla dodatkowej warstwy ochrony - ale to temat na osobny wpis.
