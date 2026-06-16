+++
title = "Ansible - instalacja pakietu jeśli nie istnieje"
date = "2020-09-29T10:41:48+02:00"
slug = "ansible"
draft = false
tags = ["ansible"]
featureImage = "/content/images/2020/09/ansible-install-package-if-not-installed.jpg"
+++

Exit code przydać nam się mogą w ansiblu do sprawdzania prostych warunków. W tym przypadku do zweryfikowania czy mamy zainstalowany pakiet repozytoriów puppetlabs.

    - name: "Check if puppet is installed"  
      shell: dpkg-query -W puppet5-release
      register: is_puppet_installed
      failed_when: no
      changed_when: no
      
    - name: "Download puppetlabs package"
      get_url:  
        url: https://apt.puppetlabs.com/puppet5-release-{{   ansible_distribution_release }}.deb
        dest: /tmp/puppet5-release-{{ ansible_distribution_release }}.deb
      when: is_puppet_installed.rc == 1

    - name: "Install puppet repository"
      apt: deb="/tmp/puppet5-release-{{ ansible_distribution_release }}.deb"   when: is_puppet_installed.rc == 1

### Linki

[Ansible error handling](https://docs.ansible.com/ansible/latest/user_guide/playbooks_error_handling.html)

