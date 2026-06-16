+++
date = 2022-08-06T20:56:25+02:00
description = ""
draft = false
thumbnail = ""
slug = "argocd"
title = "ArgoCD - ingress ssl too many redirects"
tags = ["argocd", "k8s", "ingress"]

+++


Jeżeli chcemy użyć ingressa z ArgoCD zetkniemy się z problemem pętli redirectów

```bash
curl -I https://argo.michal.kuzdzal.pl
HTTP/2 307
date: Sat, 06 Aug 2022 20:25:02 GMT
content-type: text/html; charset=utf-8
location: https://argo.michal.kuzdzal.pl/
```

Dzieje się tak ponieważ backend ArgoCD oczekuje, że sam nawiąże połączenie TLS, w innym przypadku zawsze przekierowuje połączenie na HTTPS.

Najprostszym sposobem jest wyłączenie przekierowań:

```bash
kubectl edit deployments.apps argocd-server -n argocd
```

W polu **command** dodajemy opcję **insecure**:

```bash
      - command:
        - argocd-server
        - --insecure
```

Tworzymy na nowo poda:

```bash
kubectl scale -n argocd deployment/argocd-server --replicas=0 && kubectl scale -n argocd deployment/argocd-server --replicas=1
```

Po tym zabiegu wszystko powinno już działać:

```bash
curl -I https://argo.michal.kuzdzal.pl
HTTP/2 200
date: Sat, 06 Aug 2022 20:50:51 GMT
content-type: text/html; charset=utf-8
content-length: 788
accept-ranges: bytes
content-security-policy: frame-ancestors 'self';
x-frame-options: sameorigin
x-xss-protection: 1
strict-transport-security: max-age=15724800; includeSubDomains
```
