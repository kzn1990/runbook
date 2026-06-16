+++
date = 2020-09-30T08:00:00+02:00
description = ""
draft = false
thumbnail = "/content/images/2020/09/ansible-global-path.jpg"
slug = "ansible-dodanie-sciezki-path"
title = "Ansible - dodanie ścieżki $PATH"
tags = ["ansible"]
+++


Przy budowaniu playbooków często musimy dodać jakąś ścieżkę globalnie do zmiennej PATH. Możemy tego dokonać np. przy użyciu modułu copy:

```bash
  - name: Add new $PATH.
    copy:
      dest: /etc/profile.d/my-path.sh
      content: 'PATH=$PATH:{{ /opt/my/path/ }}'
```

### Linki

[Ansible copy](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/copy_module.html).

