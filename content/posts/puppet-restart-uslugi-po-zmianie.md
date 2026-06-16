+++
title = "Puppet - restart usługi po zmianie pliku"
date = "2020-10-05T11:53:49+02:00"
slug = "puppet-restart-uslugi-po-zmianie"
draft = false
tags = []
featureImage = "/content/images/2020/10/puppet-restart-uslugi-1.jpg"
+++

W przypadku puppeta sprawa jest bardzo prosta. Najpierw deklarujemy naszą usługę:

``` bash
service { "stunnel4" :
     ensure => "running",
     enable => "true",
     require => Package["stunnel4"],
}
```

Następnie w pliku, którym chcemy striggerować nasz serwis dodajemy parametr 'notify':

``` bash
file { '/etc/stunnel/stunnel.pem': ## {{{
     source      => 'puppet:///modules/lb-hq-cm-ha/stunnel-mysql.pem',
     owner       => root,
     group       => root,
     mode        => '0644',
     notify      => Service['stunnel4'],
} ## }}}
```

### Linki

[Puppet service](https://puppet.com/docs/puppet/5.5/types/service.html).

