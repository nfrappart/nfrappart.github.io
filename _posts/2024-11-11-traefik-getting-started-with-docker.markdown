---
layout: post
title:  "Traefik: getting started with docker"
description: Discover the power of traefik for local docker lab
date:   2024-11-11 09:00:00 +0100
tags: traefik docker devops
---

## Table of Content
- [Introduction](#1--introduction)
- [Prerequisites](#2--Prerequisites)
- [Traefik](#3--Traefik)
- [Traefik with Docker](#4--Run-Traefik-locally-with-Docker)
- [SSL with Traefik](#5--Secure-and-expose-an-http-server-with-Traefik)
- [Passthrough](#6--Leverage-Passthrough-for-services-already-in-SSL/TLS)
- [Conclusion](#7--Conclusion)

# 1- Introduction
I recently decided to revisit local labs. The motivation was simple: it's free compared to cloud hosted resources, which makes it more suitable for educational content. Also I happened to get myself without Internet connection (hello airplane!) which renders cloud hosted labs unusable. Finally, working on an ARM based macbook, VM is not really viable anymore.
This particular topic still require Internet connectivity at some point, but not most of the time.

# 2- Prerequisites
In this blog post we will setup traefik for with docker compose, but we want to leverage traefik's capability to generate Let's Encrypt SSL certificates for backend services. This is one of the "magic" features of Traefik, it allows you to expose SSL services without the headache of manual certificate managemet (both issuance and renewal).

To do so I use Azure DNS to which I delegated the management of a public DNS Zone I own. Azure DNS is a supported provider for Traefik, but you can use many other providers, and benefit from its DNS challenge workflow to manage SSL certificates. Tht list of available DNS providers can be found [here](https://doc.traefik.io/traefik/https/acme/). You can also use http challenge, but that will require you to expose a service to the Internet.

- docker-compose
- a Domain Name (with management delegated to Azure DNS)
- an Azure subscription with a public DNS zone for you domain

Since we are going to use Azure DNS for our public domain, we'll need to provide Traefik some credentials to perform DNS challenge. If you're not familiar, one of the way Let's Encrypt CA can verify you own the domain you're requesting a certificate for. It essentially relies on verifying that a TXT record generated at the request is created in the domain zone by its owner, hence proving the user is indeed the owner.
Since we are working from a regular workstation, we will use basic Entra ID service principal and assign it a role.

```bash
# Create SP with permissions on you Azure DNS Zone:
az ad sp create-for-rbac --display-name <your_sp_name> --role "DNS Zone Contributor" --scopes "<your_azure_dns_zone_id>"

```

In the above snippet, you can replace `<your_sp_name>` by whatever name you want to use for your service principal, th role `DNS Zone Contributor` is a built-in role with enough permissions to manage your DNS records, and finally, the scope `"<your_azure_dns_zone_id>"` must be replaced by your Azure DNS Zone id. You can retrieve this id with an Az CLI command like `az network dns zone show --zone-name example.com -g my-rg --query "id"`.

This will output you something like this:

```bash
{
  "appId": "aaaa114c-d001-4486-888a-16d4ad5695f2",
  "displayName": "sp-test",
  "password": "mLp8sfiohoi...",
  "tenant": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}

```

Store safely the password and appId as you will need to be able to provide them to Traefik at startup.

# 3- Traefik
Now, let's take a step back and introduce Traefik, or rather Traefik Proxy I should say.
Traefik Labs is developping multiple product around what we can call "ingress" management, in the broad sense of the term (not specifically Ingress as in Kubernetes).
Traefik proxy is open source and is what we will use here, but Traefik Labs also has Traefik Hub (API Gateway/Management) and Traefik Enterprise, in its portfolio. Traefik Proxy is free, but depending on your needs, other version should be used. Some will of course induce additional costs. But as with anything in the OSS world, I strongly encourage to assess your need and use the more refined paid versions this is what fits your needs. Paying for the full product when your business or your client benefits from it is the best way to support this open source ecosystem, so that we can still use awesome products for free for personal or small businesses use cases.

Here we will use Traefik locally with Docker, but you can use it as an Ingress Controller alternative if you're a Kubernetes user. There is also a very great integration with HashiCorp's scheduler: Nomad.

# 4- Run Traefik locally with Docker
We want to use Traefik to expose our services running in our docker host. The purpose here is to easiily spin-up services on a local machine, while still having SSL and mock a real world infrastructure where your service is hsoted behind some kind of load balancer.

So first, ensure that you have the requirements on your machine, as indicated in the [# 2- Prerequisites](#2--Prerequisites) section.
You will then have to setup multiple files:
- docker-compose.yaml file: to setup docker to pull Traefik image and start it
- traefik.yml file: to setup Traefik's behaviour, like entrypoints, certificate resolvers, logging, api settings, etc.

## 4.1- Prepare Docker folder structure
Because you will need to persist data and will later want to add services to be routed by Traefik, I suggest your create a `docker` folder in which you will create a `traefik` subdolder where you will put everything related to Traefik. Then any other service to be routed/load balanced by Traefik should have its own folder under the `docker` folder. Example:

```bash
docker
├── traefik
│   ├── acme.json
│   ├── certs
│   ├── config
│   │   └── traefik.yml
│   ├── docker-compose.yaml
│   └── log
│       └── traefik.log
│ 
├── serviceA
│   ├── docker-compose.yaml
└── serviceB
    └── docker-compose.yaml

```

In the example above, the `traefik` folder we host every file related to traefik docker instance, either docker related config or traefik related config. The as an example you can see 2 other subfolders un `docker` folderm name `serviceA` and `serviceB` where you'd store config related to hypothetical other docker hosted services.

## 4.2- Docker Compose configuration

## 4.3- Traefik proxy traefik.yaml (or .toml) configuration file

# 5- Secure and expose an http server with Traefik
Demo http router with whoami http service, using DNS challenge

# 6- Leverage Passthrough for services already in SSL/TLS
Demo tcp router

# 7- Conclusion
Wrap up
