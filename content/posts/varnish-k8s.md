+++
date = 2022-02-01T22:15:00+01:00
description = ""
draft = false
thumbnail = "/content/images/2022/02/k8s-varnish.jpg"
slug = "varnish-k8s"
title = "Varnish - cache serwer w k8s"
tags = ["k8s", "varnish"]
+++


Varnish, świetny serwer cache'u umożliwiający bardzo zaawansowaną konfigurację. Większość serwerów cdn dostępnych w internecie wykorzystuje właśnie varnisha na backendzie do serwowania statycznych danych. Osobiście użyłem go jako front przed ghostem, o którego de facto oparty jest ten blog.

### Pliki konfiguracyjne

Konfiguracja samego varnisha _default.vcl_:

```
vcl 4.0;

backend default {
  .host = "blog-mk:2368";
}

sub vcl_recv {
  if (req.url ~ "/(admin|p|ghost)/") {
           return (pass);
  }
  unset req.http.cookie;
}

sub vcl_backend_response {
    if (beresp.http.content-type ~ "text/plain|text/css|application/json|application/x-javascript|text/xml|application/xml|application/xml+rss|text/javascript") {
        set beresp.do_gzip = true;
        set beresp.http.cache-control = "public, max-age=1209600";
    }
    set beresp.ttl = 1w;
}
```

Deployment _varnish-dep.yaml:_

```
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: blog-varnish-dep
  labels:
    app: blog-varnish
spec:
  replicas: 1
  selector:
    matchLabels:
      app: blog-varnish
  template:
    metadata:
      labels:
        app: blog-varnish
    spec:
      containers:
      - name: varnish
        image: varnish:6.4
        imagePullPolicy: Always
        ports:
        - containerPort: 80
        volumeMounts:
        - mountPath: /etc/varnish/default.vcl
          name: varnish-config
          subPath: default.vcl
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 3
          periodSeconds: 3
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
      volumes:
      - name: varnish-config
        configMap:
          name: varnish-config
          items:
            - key: default.vcl
              path: default.vcl
```

Service _varnish-svc.yaml_:

```
---
kind: Service
apiVersion: v1
metadata:
  name: varnish-svc
spec:
  selector:
    app: blog-varnish
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

Ingress _varnish-ing.yaml_:

```
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: public
  name: blog-stage
spec:
  rules:
  - host: michal.kuzdzal.pl
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: varnish-svc
            port:
              number: 80
```

### Deploy

```
kubectl create configmap varnish-config --from-file=default.vcl -n blog-stage
kubectl apply -f varnish-ing.yaml -f varnish-svc.yaml -f varnish-dep.yaml
```

### Podsumowanie

W moim przypadku zastosowanie varnisha zwiększyło możliwości przetwarzania req/s mojego clustra prawie **dziesięciokrotnie**. Wszystko oparte jest tutaj o dosyć słabe wydajnościowo _free-tier_ z **oracle-cloud'a**. Natomiast jeżeli połączymy całość z cloudflarem, to można uzyskać więcej niż zadawalające rezultaty.

### Linki

[Varnish](https://varnish-cache.org),[Oracle free-tier.](https://www.oracle.com/pl/cloud/free/)

