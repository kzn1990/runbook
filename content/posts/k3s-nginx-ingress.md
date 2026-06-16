+++
date = 2022-01-05T20:58:00+01:00
description = ""
draft = false
thumbnail = "/content/images/2022/02/k3s-nginx-ingress.jpg"
slug = "k3s-nginx-ingress"
title = "k3s - nginx ingress"
tags = ["k8s", "k3s", "nginx", "ingress"]
+++


Domyślnym ingressem w k3s jest traefik. Osobiście wolne korzystać z nginx'a więc opiszę pokrótce jak zdeploywać go w clustrze k3s'a.

### Instalacja k3s

Zaczniemy od instalacji samego k3s z wyłączonym traefikiem:

```
curl -sfL https://get.k3s.io | sh -s - --disable traefik
```

### Deploy nginx ingress

Z zainstalowanym 'clustrem' jesteśmy gotowi do instalacji nginx ingress:

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/baremetal/deploy.yaml
```

Zmieniamy network na lokalny aby wystawić ingress na 'zewnątrz':

```
cat > ingress.yaml <<EOF 
spec:
  template:
    spec:
      hostNetwork: true
EOF

kubectl patch deployment ingress-nginx-controller -n ingress-nginx --patch "$(cat ingress.yaml)"
```

### Testy deploy'u

W tym momencie na localhoscie powinniśmy mieć już wystawiony porty http/s. Możemy to sprawdzić curlem:

```
➜  ~ curl https://localhost -k
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx</center>
</body>
```

### Deploy przykładowego nginxa

Utwórzmy zatem przykładowy deployment i wystawmy go za pomocą ingressu:

```
cat > deploy.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      # manage pods with the label app: nginx
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-svc
            port:
              number: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
spec:
  ports:
    - name: http
      port: 80
  selector:
    app: nginx
EOF

kubectl apply -f deploy.yaml
```

Ważne tutaj jest dodanie w ingress'ie **_kubernetes.io/ingress.class: nginx_**. Inaczej nasz ingress nie wystawi naszego servicu. Jeżeli nie chcemy tego robić musimy wyedytować naszą ingressclass i ustawić go jako default:

```
kubectl edit ingressclass nginx
  annotations:
    ingressclass.kubernetes.io/is-default-class: "true"
```

### Linki

[Nginx ingress](https://kubernetes.github.io/ingress-nginx/)

