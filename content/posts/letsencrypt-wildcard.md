+++
date = 2020-07-12T19:47:31+02:00
description = ""
draft = false
thumbnail = "/content/images/2020/07/lets-encrypt-wildcard.jpg"
slug = "letsencrypt-wildcard"
title = "Let's encrypt - certyfikat wildcard"

+++


Od marca 2018 r. let's encrypt wprowadził opcję generowania certyfikatów typu wildcard.

### Generowanie certyfikatu

```
certbot certonly --manual --preferred-challenges dns --server https://acme-v02.api.letsencrypt.org/directory --manual-public-ip-logging-ok -d '*.kuzdzal.pl' -d kuzdzal.pl
```

W trakcje operacji zostaniemy poproszeni o dodanie rekordu TXT dla wskazanej domeny.

```
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Plugins selected: Authenticator manual, Installer None
Obtaining a new certificate
Performing the following challenges:
dns-01 challenge for kuzdzal.pl

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Please deploy a DNS TXT record under the name
_acme-challenge.kuzdzal.pl with the following value:

x4MrZ6y-JqFJQRmq_lGi9ReRQHPa1aTC9J2O7wDKzq8

Before continuing, verify the record is deployed.
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Press Enter to Continue
```

### Linki

[Let's encrypt wildcard](https://letsencrypt.org/2017/07/06/wildcard-certificates-coming-jan-2018.html),[Certbot](https://certbot.eff.org/about/).

