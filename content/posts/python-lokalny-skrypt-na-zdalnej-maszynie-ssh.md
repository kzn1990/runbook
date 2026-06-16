+++
title = "Python - lokalny skrypt na zdalnej maszynie (ssh)"
date = "2022-03-29T06:30:54+02:00"
slug = "python-lokalny-skrypt-na-zdalnej-maszynie-ssh"
draft = false
tags = ["python", "ssh"]
featureImage = "/content/images/external/python-snake.jpg"
+++

Czasem jest taka potrzeba uruchomienia jakiegoś lokalnego skryptu/programu na zdalnym hoście. Jeżeli nie chcemy kopiować i umieszczać na hoście tego skryptu możemy użyć ssh:

    ssh user@host python3 < script.py

Natomiast jeżeli chcemy uruchomić skrypt z dodatkowymi parametrami:

    ssh user@host python3 -u - --parametr arg1 < script.py

Oczywiście nic nie stoi na przeszkodzie aby odpalić w ten sposób każdy inny skrypt bashowy, pearlowy itp.

