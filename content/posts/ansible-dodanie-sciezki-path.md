+++
title = "Ansible - dodanie ścieżki $PATH"
date = "2020-09-30T08:00:00+02:00"
slug = "ansible-dodanie-sciezki-path"
draft = false
tags = ["ansible"]
featureImage = "/content/images/2020/09/ansible-global-path.jpg"
+++

Przy budowaniu playbooków często musimy dodać jakąś ścieżkę globalnie do zmiennej PATH. Możemy tego dokonać np. przy użyciu modułu copy:

``` bash
  - name: Add new $PATH.
    copy:
      dest: /etc/profile.d/my-path.sh
      content: 'PATH=$PATH:{{ /opt/my/path/ }}'
```

### Linki

[Ansible copy](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/copy_module.html).

