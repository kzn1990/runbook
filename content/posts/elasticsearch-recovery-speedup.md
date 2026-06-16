+++
title = "Przyśpieszenie rolling restart i recovery shardów w Elasticsearch"
date = "2025-08-12T13:19:19+02:00"
slug = "elasticsearch-recovery-speedup"
draft = false
tags = ["elasticsearch"]
featureImage = "/content/images/2025/08/elasticsearch-speedup-recovery.png"
+++

Czasem trzeba zrobić rolling restart w klastrze Elasticsearch - wymienić sprzęt, zaktualizować wersję lub przenieść shardy na nowy węzeł. Bez odpowiedniego przygotowania proces może trwać długo i generować dużo niepotrzebnego ruchu.\
Poniżej opisuję, co robię, żeby przyspieszyć restart i ograniczyć ruch shardów.

------------------------------------------------------------------------

### Wyłączanie alokacji shardów podczas restartu

Żeby uniknąć niekontrolowanego przemieszczania shardów w trakcie restartu węzła, wyłączam alokację replik. Klaster wtedy nie zaczyna od razu przesuwać danych, co pozwala na kontrolowany proces restartu.

    PUT _cluster/settings 
    {
      "persistent": {    "cluster.routing.allocation.enable": "primaries"  }
    }

Po zakończeniu restartu przywracam domyślne ustawienie:

    PUT _cluster/settings
    {
      "persistent": {    "cluster.routing.allocation.enable": null  }
    }

------------------------------------------------------------------------

### Opóźnienie reakcji klastra (delayed timeout)

Domyślnie Elasticsearch od razu po utracie węzła zaczyna rebalansować shardy, co generuje zbędny ruch, jeśli węzeł szybko wróci. Zwiększam timeout, żeby dać czas na ewentualny powrót węzła.

`PUT _all/_settings {  "index.unassigned.node_left.delayed_timeout": "5m"}`

------------------------------------------------------------------------

### Flush przed restartem

Wykonuję flush, żeby zapisać segmenty na dysk i przyspieszyć późniejsze odzyskiwanie shardów.

`POST /_flush`

------------------------------------------------------------------------

### Restart node po node

Restart wykonuję pojedynczo dla każdego węzła:

1.  Wyłączam alokację shardów (patrz wyżej)
2.  Robię flush
3.  Restartuję węzeł
4.  Przywracam alokację shardów (`null`)
5.  Czekam na status klastra **green**
6.  Powtarzam dla kolejnego węzła

------------------------------------------------------------------------

### Tuning recovery shardów

Jeśli infrastruktura pozwala, zwiększam liczbę równoczesnych recovery i limit przepustowości, żeby skrócić czas odtwarzania shardów.

    PUT _cluster/settings 
    {  
      "persistent": {
        "cluster.routing.allocation.node_concurrent_recoveries": 15,
        "indices.recovery.max_bytes_per_sec": "2000mb"  
      }
    }

