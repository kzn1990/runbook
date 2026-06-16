+++
title = "Git - cofnięcie ostatniego commitu"
date = "2020-10-01T08:00:00+02:00"
slug = "git-cofniecie-ostatniego-commitu"
draft = false
tags = ["git"]
featureImage = "/content/images/2020/09/branches-undo-last-commit.jpg"
+++

Przy pracy z gitem nie raz spotkamy się z sytuacją kiedy będziemy musieli wycofać ostatnio dodany commit. Z pomocą przychodzi reset z oderwaniem "głowy" o jeden commit do góry:

``` bash
git reset --soft HEAD~1
```

Flaga --soft pozwala nam na zachowanie zmian w plikach.

``` bash
git reset --hard HEAD~1
```

Flaga --hard usuwa wszelkie zmiany w plikach.

Jeżeli chcemy wycofać się o kilka commitów podajemy id danego commita

``` bash
git reset --hard 0a15a3b9
```

