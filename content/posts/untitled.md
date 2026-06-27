+++
title = "Kompilacja HAProxy 3.2.3 z obsługą QUIC (OpenSSL 3.1.5) na Debianie 12"
date = "2025-06-01T09:31:00+02:00"
slug = "untitled"
draft = false
tags = ["haproxy"]
featureImage = "/content/images/2025/08/Gemini_Generated_Image_xln6w1xln6w1xln6.webp"
+++

Jeśli zależy Ci na wykorzystaniu pełni możliwości HAProxy, w tym najnowszych funkcji takich jak obsługa protokołu **QUIC**, często konieczna jest kompilacja z kodu źródłowego. Domyślne pakiety HAProxy dostępne w oficjalnych repozytoriach systemów (np. przez `apt install haproxy`) zazwyczaj nie zawierają wsparcia dla QUIC, ponieważ wymaga to użycia niestandardowej biblioteki OpenSSL. Co więcej, obsługa QUIC jest często dostępna w wersjach komercyjnych (HAProxy Enterprise), dlatego ręczna kompilacja jest najlepszym sposobem, aby uzyskać tę funkcjonalność w wersji darmowej.

W tym poradniku pokażę, jak skompilować HAProxy w wersji 3.2.3 na systemie Debian 12, używając OpenSSL z obsługą QUIC. Cały proces podzielimy na dwa główne etapy: kompilację OpenSSL, a następnie kompilację HAProxy.

#### **Etap 1: Kompilacja i instalacja OpenSSL z QUIC**

Zaczynamy od przygotowania OpenSSL z obsługą protokołu QUIC. Domyślna wersja OpenSSL w systemie Debian nie obsługuje tej funkcji, dlatego musimy skompilować ją ręcznie.

**Instalacja niezbędnych narzędzi**

Zanim zaczniemy, upewnij się, że masz zainstalowane podstawowe narzędzia do kompilacji oraz biblioteki deweloperskie, które będą nam potrzebne do skompilowania HAProxy (Lua, PCRE2).Bash

    sudo apt install make build-essential liblua5.4-dev libpcre2-dev

**Pobranie i dekompresja OpenSSL**

Pobierzemy wersję OpenSSL z gałęzi `quictls`, która została zmodyfikowana, aby wspierać QUIC.Bash

    wget https://github.com/quictls/openssl/archive/refs/tags/opernssl-3.1.5-quic1.tar.gz
    tar zxvf opernssl-3.1.5-quic1.tar.gz
    cd opernssl-3.1.5-quic1

**Konfiguracja i kompilacja OpenSSL**

Teraz skonfigurujemy i skompilujemy OpenSSL. Użyjemy wielu opcji, aby zapewnić optymalizację, bezpieczeństwo oraz obsługę QUIC (`enable-tls1_3`). Instalacja docelowo znajdzie się w katalogu `/opt/quictls`, aby nie kolidowała z systemową wersją OpenSSL.Bash

    OPENSSL_OPTS="enable-tls1_3 \
        -g -O3 -fstack-protector-strong -Wformat -Werror=format-security \
        -DOPENSSL_TLS_SECURITY_LEVEL=2 -DOPENSSL_USE_NODELETE -DL_ENDIAN \
        -DOPENSSL_PIC -DOPENSSL_CPUID_OBJ -DOPENSSL_IA32_SSE2 \
        -DOPENSSL_BN_ASM_MONT -DOPENSSL_BN_ASM_MONT5 -DOPENSSL_BN_ASM_GF2m \
        -DSHA1_ASM -DSHA256_ASM -DSHA512_ASM -DKECCAK1600_ASM -DMD5_ASM \
        -DAESNI_ASM -DVPAES_ASM -DGHASH_ASM -DECP_NISTZ256_ASM -DX25519_ASM \
        -DX448_ASM -DPOLY1305_ASM -DNDEBUG -Wdate-time -D_FORTIFY_SOURCE=2 \
    "
    ./config --libdir=lib --prefix=/opt/quictls $OPENSSL_OPTS
    make -j $(nproc)
    make install -j $(nproc)

**Weryfikacja instalacji OpenSSL**

Sprawdźmy, czy nowa wersja OpenSSL została poprawnie zainstalowana i obsługuje QUIC.Bash

    /opt/quictls/bin/openssl version

Oczekiwana odpowiedź powinna wyglądać tak: `OpenSSL 3.1.5+quic 30 Jan 2024 (Library: OpenSSL 3.1.5+quic 30 Jan 2024)`

#### **Etap 2: Kompilacja HAProxy 3.2.3**

Teraz, gdy mamy gotowy OpenSSL z QUIC, możemy przejść do kompilacji HAProxy, która go wykorzysta.

**Pobranie i dekompresja HAProxy**

Pobierz najnowszą stabilną wersję HAProxy 3.2.3 ze strony projektu.Bash

    wget http://www.haproxy.org/download/3.2/src/haproxy-3.2.3.tar.gz
    tar zxf haproxy-3.2.3.tar.gz
    cd haproxy-3.2.3

**Konfiguracja i kompilacja HAProxy**

W tej sekcji skompilujemy HAProxy, wskazując mu ścieżki do naszego niestandardowego OpenSSL. Musimy również włączyć wsparcie dla QUIC, Lua, PCRE2 oraz innych przydatnych funkcji.Bash

    HAPROXY_CFLAGS="-O3 -g -Wall -Wextra -Wundef -Wdeclaration-after-statement -Wfatal-errors -Wtype-limits -Wshift-negative-value -Wshift-overflow=2 -Wduplicated-cond -Wnull-dereference -fwrapv -Wno-address-of-packed-member -Wno-unused-label -Wno-sign-compare -Wno-unused-parameter -Wno-clobbered -Wno-missing-field-initializers -Wno-cast-function-type -Wno-string-plus-int -Wno-atomic-alignment"
    HAPROXY_OPTS="TARGET=linux-glibc \
        USE_PCRE2=1 USE_PCRE2_JIT=1 \
        USE_PCRE= USE_PCRE_JIT= \
        USE_GETADDRINFO=1 \
        USE_OPENSSL=1 USE_LIBCRYPT=1 \
        USE_LUA=1 \
        USE_PROMEX=1 \
        USE_QUIC=1 \
        USE_EPOLL=1 \
        USE_THREAD=1 \
        USE_NS=1 \
        USE_SLZ=1 USE_ZLIB= \
        LUA_LIB_NAME=lua5.4 \
    "
    echo "/opt/quictls/lib" > /etc/ld.so.conf.d/quictls.conf && ldconfig
    make -j $(nproc) $HAPROXY_OPTS CFLAGS="$HAPROXY_CFLAGS" LDFLAGS="$HAPROXY_LDFLAGS" SSL_INC=/opt/quictls/include SSL_LIB=/opt/quictls/lib all admin/halog/halog

**Ważna uwaga:** Komenda `echo "/opt/quictls/lib" > /etc/ld.so.conf.d/quictls.conf && ldconfig` dodaje ścieżkę do naszych nowo skompilowanych bibliotek OpenSSL do globalnej konfiguracji systemu. To kluczowy krok, który pozwala HAProxy (i innym programom) znaleźć biblioteki SSL/TLS.

**Instalacja HAProxy**

Po udanej kompilacji, zainstaluj binarkę HAProxy w domyślnej lokalizacji.Bash

    sudo make -j $(nproc) install-bin

Gratulacje! Właśnie skompilowałeś HAProxy w wersji 3.2.3 z pełnym wsparciem dla protokołu QUIC na Debianie 12. Teraz możesz skonfigurować HAProxy, aby wykorzystać jego nowe możliwości w Twoim środowisku.

