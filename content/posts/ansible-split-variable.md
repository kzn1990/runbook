+++
title = "Ansible - split variable"
date = "2022-07-06T12:51:07+02:00"
slug = "ansible-split-variable"
draft = false
tags = ["ansible", "python"]
featureImage = "/content/images/2022/07/ansible-jinja.jpg"
+++

Czasami w ansiblu może zajść potrzeba żeby "pociąć" jakąś zmienną.\
Jako że ansible == python, to możemy skorzystać z formatowania jinja2.

Przykładowo aby oddzielić samą nazwę użytkownika od maila ze zmiennej AWX awx_user_name:

    {{ awx_user_name.split('@')[0] | lower }}

### Linki

[Jinja](https://jinja.palletsprojects.com/en/3.1.x/).

