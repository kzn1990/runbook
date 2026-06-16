+++
date = 2022-07-06T12:51:07+02:00
description = ""
draft = false
thumbnail = "/content/images/2022/07/ansible-jinja.jpg"
slug = "ansible-split-variable"
title = "Ansible - split variable"
tags = ["ansible"]

+++


Czasami w ansiblu może zajść potrzeba żeby "pociąć" jakąś zmienną.
Jako że ansible == python, to możemy skorzystać z formatowania jinja2.

Przykładowo aby oddzielić samą nazwę użytkownika od maila ze zmiennej AWX awx_user_name:

```bash
{{ awx_user_name.split('@')[0] | lower }}
```

### Linki

[Jinja](https://jinja.palletsprojects.com/en/3.1.x/).
