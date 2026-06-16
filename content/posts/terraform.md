+++
title = "Terraform - deploy LXD container"
date = "2022-02-23T18:17:47+01:00"
slug = "terraform"
draft = false
tags = []
featureImage = "/content/images/2022/02/terraform-lxd-1.jpg"
+++

Terraform, numer jeden jeżeli chodzi o IaC (***Infrastructure as Code**). W bardzo przyjemny sposób można zarządzać infrastrukturą cloudową jak i* on-premise.\
W tym przypadku opiszę pokrótce jak deployować virtualną maszynę w LXD.

### Konfiguracja

Mamy mało kodu, więc tworzymy wszystko w jednym pliku main.tf:

    terraform {
      required_providers {
        lxd = {
          source = "terraform-lxd/lxd"
        }
      }
    }

    provider "lxd" {
      generate_client_certificates = true
      accept_remote_certificate    = true

      lxd_remote {
        name     = "lxd-1"
        scheme   = "https"
        address  = "lxd.kuzdzal.pl"
        port     = "8443"
        default  = true
      }
    }

    resource "lxd_container" "worker" {
      count     = 3
      remote    = "lxd-1"
      name      = "k8s-${count.index}"
      image     = "ubuntu:20.04"
      ephemeral = false
      type      = "virtual-machine"
      config = {
        "user.access_interface" = "enp5s0"
      }
      limits    = {
        "memory" = "8GB"
        "cpu" = 4
      }
      profiles = ["default"]
      device {
          name       = "root"
          properties = {
              "path" = "/"
              "pool" = "pool_nvme"
              "size" = "25GB"
          }
          type       = "disk"
      }
    }

    output "droplet_ip_addresses" {
      value = {
        for droplet in lxd_container.worker:
        droplet.name => droplet.ipv4_address
      }
    }

### Uruchomienie

Inicjalizacja, czyli pobranie wszystkich wymaganych pluginów (w tym przypadku tylko lxd), zsetupowanie i na końcu zniszczenie zasobów:

    ~ terraform init
    ~ terraform apply -auto-approve
    ~ terraform destroy -auto-approve

### Linki

[Terraform lxd](https://registry.terraform.io/providers/terraform-lxd/lxd/latest/docs).

