+++
title = "CI/CD - drone k8s deploy"
date = "2021-12-14T13:36:00+01:00"
slug = "cd-cd-drone-k8s-deploy"
draft = false
tags = ["k8s", "pipeline", "CI/CD"]
featureImage = "/content/images/2022/02/Drone-deploy-k8s.jpg"
+++

Drone jest fajnym i lekkim toolem do Continuous Integration i Continuous Delivery/Deployment. Jak przy jego po mocy zdeployować coś do clustra k8s?\
Z pomocą przychodzi projekt dostępny na [GitHub'ie](https://github.com/sinlead/drone-kubectl).

### Przygotowanie

Stwórzmy pipeline, który utworzy nam testowy deployment gotowy do dalszej pracy:

    # drone 1.0 syntax
    kind: pipeline
    name: deploy

    steps:
      - name: deploy
        image: sinlead/drone-kubectl
        settings:
          kubernetes_server:
            from_secret: k8s_server
          kubernetes_cert:
            from_secret: k8s_cert
          kubernetes_token:
            from_secret: k8s_token
        commands:
          - kubectl create namespace deploy-test

Kiedy mamy już przygotowany pipeline, to musimy dodać dla danego projektu w **Drone** odpowiednie secrets'y:

![](/content/images/2022/02/Drone_secrets.png)

Secret'y możną wyciągnąć z clustra za pomocą komend:

    kubectl get secret
    NAME                                TYPE                                  DATA   AGE
    default-token-lcknp                 kubernetes.io/service-account-token   3      10d
    ...

    kubectl config view -o jsonpath='{range .clusters[*]}{.name}{"\t"}{.cluster.server}{"\n"}{end}'
    kubectl get secret default-token-lcknp -o jsonpath='{.data.token}' | base64 --decode && echo
    kubectl get secret default-token-lcknp -o jsonpath='{.data.ca\.crt}' && echo

Jeżeli wszystko poszło dobrze po odpaleniu pipelina w logach consoli powinniśmy zobaczyć nowo utworzony namespace, który jest gotowy do dalszych prac:

![](/content/images/2022/02/obraz_2022-02-06_143135.png)

Działamy tutaj na uprawnieniach *cluster-admin* i oczywiście w celach labowych nie ma w tym nic złego. Do nadawani odpowiednich uprawnień w clustrze polecam zapoznać się z rozwiązaniem ***permission-manager**.*

### Linki

[Drone](https://www.drone.io),\
[Permission-manager](https://github.com/sighupio/permission-manager).

