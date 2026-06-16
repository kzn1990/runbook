+++
date = 2022-03-29T06:30:54+02:00
description = ""
draft = false
thumbnail = "https://images.unsplash.com/photo-1600333527715-cc7f665bc332?crop=entropy&cs=tinysrgb&fit=max&fm=jpg&ixid=MnwxMTc3M3wwfDF8c2VhcmNofDI1fHxzbmFrZXxlbnwwfHx8fDE2NDg1MzUzNTc&ixlib=rb-1.2.1&q=80&w=2000"
slug = "python-lokalny-skrypt-na-zdalnej-maszynie-ssh"
title = "Python - lokalny skrypt na zdalnej maszynie (ssh)"

+++


Czasem jest taka potrzeba uruchomienia jakiegoś lokalnego skryptu/programu na zdalnym hoście. Jeżeli nie chcemy kopiować i umieszczać na hoście tego skryptu możemy użyć ssh:

```
ssh user@host python3 < script.py
```

Natomiast jeżeli chcemy uruchomić skrypt z dodatkowymi parametrami:

```
ssh user@host python3 -u - --parametr arg1 < script.py
```



