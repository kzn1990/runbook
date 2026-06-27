+++
title = "Terraform i GitOps w GitLab CI/CD"
date = "2025-07-10T12:31:00+02:00"
slug = "terraform-with-gitops"
draft = false
tags = ["terraform", "git", "gitlab"]
featureImage = "/content/images/2025/08/Terraform_i_GitOps_w_GitLab_CI-CD.webp"
+++

Dziś opiszę, jak zintegrować **Terraform** z **GitLab CI/CD**, aby stworzyć wydajny i niezawodny proces GitOps.

#### **Po co Terraform i GitOps?**

**Terraform** pozwala opisywać infrastrukturę jako kod (**IaC**). Dzięki temu możesz ją wersjonować, łatwo powielać i audytować. **GitOps** dodaje do tego proces CI/CD, w którym każda zmiana w kodzie jest automatycznie wdrażana, eliminując ręczne modyfikacje i minimalizując ryzyko błędów. Trzymanie stanu w gitlabi pomoże też uniknąć potencjalnych problemów i konfliktów.

------------------------------------------------------------------------

### **Struktura GitLab CI/CD**

Nasze CI/CD podzielimy na trzy główne etapy: `init`, `plan` i `apply`. Dodatkowo, stan Terraform (tzw. **State**) przechowamy bezpiecznie w GitLab, co uprości zarządzanie i wyeliminuje potencjalne problemy z przechowywaniem stanu w różnych miejscach.

Poniżej przedstawiam, jak powinien wyglądać plik `.gitlab-ci.yml`:

    stages:
      - validate
      - plan
      - apply

    variables:
      TF_ADDRESS: "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}"
      STATES: "terraform/state/${TF_STATE_NAME}"
      TERRAFORM_INIT: "terraform init \
                      -backend-config=address=${TF_ADDRESS}/${STATES} \
                      -backend-config=lock_address=${TF_ADDRESS}/${STATES}/lock \
                      -backend-config=unlock_address=${TF_ADDRESS}/${STATES}/lock \
                      -backend-config=username=${TF_USERNAME} \
                      -backend-config=password=${TF_PASSWORD} \
                      -backend-config=lock_method=POST \
                      -backend-config=unlock_method=DELETE \
                      -backend-config=retry_wait_min=5"
      TF_STATE_NAME: "nazwa-projektu"
      
    .terraform-base:
      image:
        name: "registry.gitlab.com/gitlab-org/terraform-images/stable:latest"
      before_script:
        - eval $TERRAFORM_INIT

    validate:
      stage: validate
      extends:
        - .terraform-base
      script:
        - terraform validate

    plan:
      stage: plan
      extends:
        - .terraform-base
      script:
        - terraform plan -out=plan.tf
      artifacts:
        paths:
          - plan.tf

    apply:
      stage: apply
      extends:
        - .terraform-base
      script:
        - terraform apply -input=false plan.tf
      rules:
        - if: '$CI_COMMIT_BRANCH == "main"'
          when: manual

Sam pipline prezentuje się następująco:

![](/content/images/2025/08/image.webp)

------------------------------------------------------------------------

#### **Skąd wziąć zmienne?**

- **`CI_API_V4_URL`** i **`CI_PROJECT_ID`**: Te zmienne zawierają adres URL do API oraz unikalny identyfikator projektu (np. CI_API_V4_URL=[https://git.example.com/api/v4/](https://git.int.clickmeeting.com/api/v4/projects/246/terraform/state)\
  CI_PROJECT_ID=210),
- **`TF_STATE_NAME`**: dowolna nazwa stanu, która będzie widoczna w gitalabie,
- **`TF_USERNAME`** i **`TF_PASSWORD`**: Te zmienne są kluczowe do uwierzytelniania w API GitLab, aby potok mógł bezpiecznie zarządzać stanem Terraform. Najbezpieczniej jest stworzyć **Project Access Token** lub **Personal Access Token** z odpowiednimi uprawnieniami (zakres `api`), a następnie zapisać go jako **zmienną środowiskową** w ustawieniach CI/CD projektu w GitLab.

------------------------------------------------------------------------

#### **Podsumowanie**

Integracja Terraform z GitLab CI/CD daje nam potężne narzędzie do zarządzania infrastrukturą w sposób w pełni zautomatyzowany, bezpieczny i zgodny z najlepszymi praktykami GitOps. Taki proces pozwala na szybsze i bardziej niezawodne wdrażanie, minimalizując błędy i zapewniając pełną transparentność zmian.

