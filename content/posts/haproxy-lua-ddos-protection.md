+++
title = "HAProxy + Lua: Ochrona przed botami i atakami DDoS"
date = "2025-07-22T14:33:00+02:00"
slug = "haproxy-lua-ddos-protection"
draft = false
tags = ["haproxy", "linux", "lua", "ddos"]
featureImage = "/content/images/2025/08/rate-limit-HAProxy-Lua.webp"
+++

## Wstęp

W świecie rosnącego natężenia ruchu automatycznego (botów) i ataków DDoS staje się niezbędne wprowadzenie inteligentnych filtrów już na warstwie L7. W tym wpisie opiszę, jak dzięki prostemu skryptowi w Lua oraz mechanizmom HAProxy zbudowałem jedną z wielu warstw ochrony backendów przed nieporządanym ruchem.

------------------------------------------------------------------------

## Koncepcja rozwiązania

1.  **Wykrywanie „sesji” per IP w oparciu o cookie**\
    - Każda unikalna wartość ciasteczka (`__Secure-app_session`) to jedna sesja,\
    - każde żądanie bez ciasteczka to tzw. „no-cookie” request.
2.  **Limit 150 sesji/IP w oknie 120 s**\
    - Po przekroczeniu tej wartości IP zostaje oznaczone jako podejrzane.
3.  **Akcja po przekroczeniu**\
    - Przekierowanie do backendu z kolejką (queue), lub drop requestu.

------------------------------------------------------------------------

## Skrypt Lua: szczegóły działania

Poniżej pełny kod skryptu, wraz z opisem:

    -- Globalna tabela do przechowywania stanu per IP
    local ip_unique_sessions = {}
    local last_cleanup = os.time()

    -- Funkcja czyszcząca – resetuje całą strukturę co 120 sekund
    local function cleanup()
        local now = os.time()
        if now - last_cleanup >= 120 then
            ip_unique_sessions = {}
            last_cleanup = now
        end
    end

    -- Escapowanie znaków specjalnych w nazwie ciasteczka
    local function escape_pattern(text)
        return text:gsub("([^%w])", "%%%1")
    end

    -- Wyciąganie wartości cookie z nagłówka
    local function get_cookie_value(cookie_header, cookie_name)
        if not cookie_header then
            return nil
        end
        local pattern = escape_pattern(cookie_name) .. "=([^;]+)"
        return cookie_header:match(pattern)
    end

    -- Główna funkcja sprawdzająca limit
    function check_rate_limit(txn)
        -- 1) Czyszczenie przestarzałych danych
        cleanup()

        -- 2) Pobranie IP i nagłówka Cookie
        local client_ip  = txn.f:src() or "unknown"
        local cookie_hdr = txn.f:hdr("Cookie")

        -- 3) Wyciągnięcie wartości ciasteczka
        local cookie_val = get_cookie_value(cookie_hdr, "__Secure-app_session")

        -- 4) Inicjalizacja struktury, jeśli nowe IP
        if not ip_unique_sessions[client_ip] then
            ip_unique_sessions[client_ip] = {
                cookie    = {},   -- set dla unikalnych wartości cookies
                no_cookie = 0     -- licznik requestów bez ciasteczka
            }
        end

        -- 5) Aktualizacja stanu: cookie vs no_cookie
        if cookie_val then
            ip_unique_sessions[client_ip].cookie[cookie_val] = true
        else
            ip_unique_sessions[client_ip].no_cookie = ip_unique_sessions[client_ip].no_cookie + 1
        end

        -- 6) Zliczanie „sesji”
        local count = 0
        for _ in pairs(ip_unique_sessions[client_ip].cookie) do
            count = count + 1
        end
        count = count + ip_unique_sessions[client_ip].no_cookie

        -- 7) Decyzja o przekroczeniu limitu
        if count > 150 then
            return "true"
        else
            return "false"
        end
    end

    -- Rejestracja funkcji jako sample-fetch „rate_limit”
    core.register_fetches("rate_limit", check_rate_limit)

### Omówienie krok po kroku

1.  **Zmienne globalne**\
    - `ip_unique_sessions`: pusty słownik, kluczem jest IP, wartością tabela z podkluczami `cookie` (set) i `no_cookie` (licznik).\
    - `last_cleanup`: znacznik czasu ostatniego pełnego resetu.
2.  **`cleanup()`**\
    - Wywoływane przy każdym sprawdzeniu – czyści cały stan, jeśli od ostatniego resetu minęło ≥120 s.\
    - Zapobiega niekontrolowanemu wzrostowi zużycia pamięci.
3.  **`escape_pattern(text)`**\
    - Zamienia znaki specjalne w nazwie ciasteczka na escaped (`%`), by `string.match` traktował je dosłownie. Dzięki temu nazwy z podkreśleniami, myślnikami czy kropkami są bezpieczne.
4.  **`get_cookie_value(cookie_header, cookie_name)`**\
    - Przy braku nagłówka `Cookie` od razu zwraca `nil`.\
    - Buduje wzorzec: `<nazwa>=([^;]+)` i zwraca pierwszy dopasowany fragment (wartość ciasteczka).
5.  **Inicjalizacja struktury**\
    - Dla każdego nowego IP tworzymy tabelę z pustym setem ciasteczek i licznikiem 0.
6.  **Aktualizacja stanu**\
    - Jeśli ciasteczko istnieje: dodajemy jego wartość do setu (unikalność gwarantowana),\
    - jeśli nie: inkrementujemy `no_cookie`.
7.  **Zliczanie i próg**\
    - Sumujemy liczbę kluczy w `cookie` + wartość `no_cookie`.\
    - Porównujemy z progiem (150); w razie przekroczenia zwracamy `"true"`.

------------------------------------------------------------------------

## Konfiguracja HAProxy

Konfiguracja haproxy może wyglądać następująco:

    # Zaladowanie skryptu
    lua-load /opt/implix/haproxy/ddos_rate_limit.lua

    # Sprawdzenie rate_limit, z pominięciem whitelisty
    http-request set-var(req.rate_limit_exceeded) lua.rate_limit if !{ src -f /etc/haproxy/whitelist.txt }

    # Ustawienie ddos_rate_exceeded na true jeżeli ip przekroczyło limit
    acl ddos_rate_exceeded var(req.rate_limit_exceeded) -m str true

    # Kierowanie do kolejkującego backendu
    use_backend fpm-queue-backend if ddos_rate_exceeded

    # Można też zdropować takie połączenie
    # http-request deny if ddos_rate_exceeded

W moim przypadku podejrzany ruch trafia na backend z kolejką, gdzie takie żądania czekają na obsłużenie.

Jakie mogą być inne podejścia do obsługi takich połączeń?

- deny - bezpośredni zakończenie połączenia z dowolnym kodem http,
- silent-drop - zamyka połączenie po stronie HAProxy bez żadnej odpowiedzi ani komunikatu do klienta,
- tarpid - podobne do silent dropa, z tą różnicą że połączenie jest przetrzymywane przez socket haproxy.

------------------------------------------------------------------------

## Podsumowanie

HAProxy dzięki skryptom Lua otwiera przed nami niemal nieograniczone możliwości zaawansowanej konfiguracji i zarządzania ruchem. Pokazany wyżej przykład to jedynie wierzchołek góry lodowej – za pomocą Lua możesz zbudować własny, w pełni programowalny load balancer, implementować niestandardowe algorytmy routingu, walidować czy modyfikować nagłówki w locie, a nawet integrować się z zewnętrznymi systemami. To elastyczne podejście pozwala dostosować HAProxy dokładnie do potrzeb Twojej infrastruktury i łatwo rozwijać go o kolejne mechanizmy ochrony czy optymalizacji.

