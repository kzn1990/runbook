+++
title = "Gitlab CI/CD - environment variable"
date = "2022-03-06T20:33:04+01:00"
slug = "gitlab-ci-cd-environment-variable"
draft = false
tags = ["CI/CD", "gitlab", "pipeline"]
featureImage = "/content/images/2022/03/gitlab-ci-environment-variable-1.jpg"
+++

Czasami jest potrzeba przekazania zmiennej env pomiędzy różnymi stagami pipelina. Od wersji 13 gitlaba możemy do tego użyć wbudowanego mechanizmu inherit environment variables. Zapisujemy naszą wartość w pliku *.env* i przekazujemy ją za pomocą mechanizmu **artifacts**:

    build:
      stage: build
      script:
        - echo "COMMIT=true" >> build.env
      artifacts:
        reports:
          dotenv: build.env

    deploy:
      stage: test
      script:
        - echo "$COMMIT"

### Linki

[Inherit environment variables](https://docs.gitlab.com/ee/ci/variables/README.html#inherit-environment-variables).

