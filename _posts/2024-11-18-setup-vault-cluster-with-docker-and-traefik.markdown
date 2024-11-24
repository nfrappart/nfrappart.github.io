---
layout: post
title:  "Local Vault cluster with Traefik"
description: The perfect setup to prepare Vault certifications
date:   2024-11-18 09:00:00 +0100
tags: traefik docker devops vault
---

## Table of Content
- [Introduction](#1--introduction)
- [Prerequisites](#2--Prerequisites)
- [How to manage DNS](#3--How-to-resolve-private-services)
- [Vault HA with LB](#4--Vault-HA-behind-load-balancer)
- [Vault and Docker Config](#5--Vault-and-Docker-configuration)
- [Traefik Config](#6--Traefik-for-service-already-using-TLS)
- [Initialize and Unseal](#7--Initialize-and-unseal-Vault)
- [Configure Vault](#8--Caonfigure-Vault)
- [Conclusion](#9--Conclusion)

# 1- Introduction
Last week I published a blog post about Traefik with docker compose to play locally. My ulterior motive was to use it to build a vault cluster to prepare for Vault Operations Professional. But because this would mix two different topics I decided to do a Traefik introduction first, then a specific blog post for Vault cluster. Another reason why I use docker is that, working on ARM based macbook, despite Apple Silicon being around for a few years now, I didn't find a satisfying virtualization solution. So here we are.
We will prepare a basic vault server config for a 3 "node" cluster, all of them in docker and use Traefik as entrypoint to the cluster.

# 2- Prerequisites
All the requirements for this setup are actually describe in last weeks post ["Traefik: getting started with docker"](https://nfrappart.github.io/2024/11/11/traefik-getting-started-with-docker.html).
But here is a recap:
- an public domain managed in an azure DNS public zone
- docker running on local machine
- Traefik running and setup with docker provider

BUT, previous article missed on one entrypoint that we will use for vault, as it focused on regular http/https services. Here is the entrypoint to add in `traefik.yaml`:

```bash
entryPoints:
  vault:
    address: :8200
```

For more details, go back to the blog post referenced above.

> But in addition to the list above, we will also need pre-existing certificates for our Vault nodes. We will not rely on Traefik for SSL certificate lifecycle in this case because we want TLS at the service level, not only at the entrypoint (Traefik) level. We will see below how to achieve this using TLS passthrough.

# 3- How to resolve private services
We are using a public domain to generate SSL certificates using let's encrypt, but we're not actualy exposing these services to the Internet. One method I like to use for private services using public domain, is to rely on a subdomain. Something along the lines of `.private.example.com`. Following the examples of previous blog posts, I'll be using `.docker.example.com`. Of course update all these configurations using your own domain name instead.

By using a subdomain, you can still successfully resolve your public services (if you have any), wherever they are hosted, using a public DNS server. But at the same time, you can safely configure your local subdomain on local DNS servers or even simply using hosts files like we did in the previous blog post.

For the rest of this article, we will consider relying on hosts file. Since we are hosting our services on the local machine we will always get `127.0.0.1` as response from DNS request to a service. In this specific case we are building a Vault cluster, so our `hosts` file should look like this:

```bash
##
# Host Database
#
# localhost is used to configure the loopback interface
# when the system is booting.  Do not change this entry.
##
127.0.0.1	localhost
255.255.255.255	broadcasthost
::1             localhost

127.0.0.1			vault.docker.example.com

```

# 4- Vault HA behind Load Balancer
We want to use Traefik as entrypoint to connect to our Vault cluster, but before we do that, let's look at how Vault behave in HA, nore specificaly behind load balancer. HashiCorp actually provide some enlightening information regarding this, on its [developper.hashicorp.com](https://developer.hashicorp.com/vault/docs/concepts/ha#behind-load-balancers) website.

According to the documentation you have two ways to setup access to a Vault cluster:
- direct access to all nodes
- access through a load balancer

Since we will be behind Traefik it falls into the second category.

This actually has an impact on Vault server config files. There is an attribute `api_addr` that needs to be setup differently depending on which situation you're in (direct access VS load balancer). In our case because we can access Vault nodes only through load balancer (i.e Traefik Proxy container) we have to set the same value for `api_addr`, we'll see this in the next section showing config files.

# 5- Vault and Docker configuration
Now let's have a look at our docker compose and vault server configuration. But first what files do we expect in the docker-compose folder?
Just like in the previous blog post, I dedicated a folder for Vault service, named `vault_cluster` with `docker-compose.yaml` file.
Then I put everything in a `vault` subfolder, tough I could very well have put it at the root of the `vault_cluster` folder...

So in there you can see a `certs` subfolder that is actually shared between all three Vault nodes, then a `config_nodeX` and `data_nodeX` for each node. You got the idea, each of these subfolder will host configuration and data for each node, while all nodes use the same certificate and private key.

> Sharing the certificate is to lighten management a bit. I use SANs in the certificate:<br>
- vault.docker.example.com
- vault1.docker.example.com
- vault2.docker.example.com
- vault3.docker.example.com


```bash
vault_cluster
├── docker-compose.yaml
└── vault
    ├── certs
    │   ├── vault.cer
    │   └── vault.key
    ├── config_node1
    │   └── config.hcl
    ├── config_node2
    │   └── config.hcl
    ├── config_node3
    │   └── config.hcl
    ├── data_node1
    ├── data_node2
    └── data_node3
```

Last important point regarding docker configuration is to create a network named `traefik`. Here's the command: `docker network create traefik`.

## 5.1- Vault server configuration file
Vault config is very basic. 
We have:
- a storage block for raft configuration
- a listener block
- the `api_addr` we discussed before
- the `cluster_addr` which is the specific address of each node
- the `ui` setting to enable web graphical interface

Below are the 3 `config.hcl` files:

```bash
storage "raft" {
  path = "/vault/data"
  node_id = "vault1"
}

listener "tcp" {
  address = "0.0.0.0:8200"
  cluster_address  = "0.0.0.0:8201"
  tls_cert_file = "/vault/certs/vault.cer"
  tls_key_file = "/vault/certs/vault.key"
}

api_addr = "https://vault.docker.example.com:8200"
cluster_addr = "https://vault1.docker.example.com:8201"
ui = true
```

```bash
storage "raft" {
  path = "/vault/data"
  node_id = "vault2"
}

listener "tcp" {
  address = "0.0.0.0:8200"
  cluster_address  = "0.0.0.0:8201"
  tls_cert_file = "/vault/certs/vault.cer"
  tls_key_file = "/vault/certs/vault.key"
}

api_addr = "https://vault.docker.example.com:8200"
cluster_addr = "https://vault2.docker.example.com:8201"
ui = true
```

```bash
storage "raft" {
  path = "/vault/data"
  node_id = "vault3"
}

listener "tcp" {
  address = "0.0.0.0:8200"
  cluster_address  = "0.0.0.0:8201"
  tls_cert_file = "/vault/certs/vault.cer"
  tls_key_file = "/vault/certs/vault.key"
}

api_addr = "https://vault.docker.example.com:8200"
cluster_addr = "https://vault3.docker.example.com:8201"
ui = true
```

As you can see the 3 files are basically identical with the exceptions of `cluster_addr` and `node_id`. All file path will be resolved by the volumes mounted in the `docker-compose.yaml`.

## 5.2- Docker compose configuration file
We'll focus on the docker compose configuration for Vault only. The Traefik labels part will be explained in #6 section.
I's pretty straight forward.
There are 3 services, one for each vault node.
To allow nodes to resolve themselves and not qury DNS outside docker, which would result in not getting any answer, we set the the `hostname` value. I mean, we could use an local DNS server or the hostfile, but we'd have to set it to your host ip, which means it would send traffic to traefik as a result, with no guarantee to reach the desired node. 
By setting the container hostname, docker will manage internal DNS resolution and provide ip address from docker network. Which brings me to the next item: `network`. We specify the `traefik` network we created at the beggining of section #5.

About volumes now, again nothing rocket science here. We need to mount the `config` and `data`, which are specific to each nodes. The `certs` folder however is shared, since we use the same certificates for each nodes, relying on SANs.

```bash
services:
  vault1:
    image: hashicorp/vault:1.18.0
    container_name: vault1
    hostname: vault1.docker.example.com
    networks:
      - traefik
    entrypoint: vault server -config=/vault/config/config.hcl -log-level=DEBUG
    cap_add:
      - IPC_LOCK
    volumes:
      - ./vault/config_node1:/vault/config
      - ./vault/certs:/vault/certs
      - ./vault/data_node1:/vault/data
    restart: always
    expose:
      - "8200/tcp" #vault api port
  
  vault2:
    image: hashicorp/vault:1.18.0
    container_name: vault2
    hostname: vault2.docker.example.com
    networks:
      - traefik
    entrypoint: vault server -config=/vault/config/config.hcl
    cap_add:
      - IPC_LOCK
    volumes:
      - ./vault/config_node2:/vault/config
      - ./vault/certs:/vault/certs
      - ./vault/data_node2:/vault/data
    restart: always
    expose:
      - "8200/tcp" #vault api port

  vault3:
    image: hashicorp/vault:1.18.0
    container_name: vault3
    hostname: vault3.docker.example.com
    networks:
      - traefik
    entrypoint: vault server -config=/vault/config/config.hcl
    cap_add:
      - IPC_LOCK
    volumes:
      - ./vault/config_node3:/vault/config
      - ./vault/certs:/vault/certs
      - ./vault/data_node3:/vault/data
    restart: always
    expose:
      - "8200/tcp" #vault api port

networks:
  traefik:
    external: true
```

# 6- Traefik for service already using TLS
OK, we're almost set. Now the final touch is to enable passthrough for TLS.
For that, we need to add the Traefik labels. Traefik will pick them up because the docker provider is setup and automatically configure itself to expose the service.

The secret sauce here is to use a `tcp` router instead of an `http` one. This gives you the access to the label `passthrough`.

```bash
    labels:
      - "traefik.enable=true"
      - "traefik.tcp.routers.vault1.rule=HostSNI(`vault.docker.example.com`)"
      - "traefik.tcp.routers.vault1.entrypoints=vault"
      - "traefik.tcp.routers.vault1.tls.passthrough=true"
      - "traefik.tcp.services.vault1.loadbalancer.server.port=8200"

```

This bloc is for the `vault1` node, you'll need the same for each node.
here is a quick break down of the labels:
- `traefik.enable` is set to `true` so it says to traefik to enable this service and route traffic to it and will rely on other labels for that 
- ``traefik.tcp.routers.vault1.rule=HostSNI(`vault.docker.example.com`)`` is the rule that tells which SNI should be routed to our vault instances
- `traefik.tcp.routers.vault1.entrypoints=vault` indicates the entrypoint to use. PLease refer to the top section of this blog post where we mention the addition of a `vault` entrypoint.
- `traefik.tcp.routers.vault1.tls.passthrough=true` now, this is our magic trick to tell Traefik not to generate TLS certificate but rather, passthrough and let the backend service presents the certificate on his own
- `traefik.tcp.services.vault1.loadbalancer.server.port=8200` tells Traefik that the backend service's container is listening on port 8200.

# 7- Initialize and unseal Vault
Now that we have all our nodes up and our loadbalancer ready, we need to initialize Vault and unseal it. Refer to documentation to know more.
To do so we will connect to our vault nodes one by one and make them join the same cluster.

Connect to `vault1` using interactive exec command: `docker exec -it vault1 sh`.
Then initialize and unseal `vault1`:

```bash
#Set Vault server
export VAULT_ADDR=https://vault1.docker.example.com:8200

#Initialize Vault
vault operator init
```

You will get 5 unseal keys along with a root token.

You can now unseal the cluster (which currently has only one node) with the `vault operator unseal` command. You will need to enter 3 of the 5 unseal keys, re-entering the `vault operator unseal` command after each valid key entered.
You can then exit the container.

Now connect to vault2" with `docker exec -it vault2 sh`, make it join cluster made with vault1 with `vault operator raft join https://vault1.docker.example.com:8200` and then unseal it too with `vault operator unseal` and provide 3 unseal keys like you did for `vault1`. Exit the container.
Repeat these steps for `vault3`.

You're all set, your cluster is up.

# 8- Configure Vault
There are now multiple ways for you to configure your vault cluster:

- You can now go to the ui at `https://vault.docker.example.com:8200` and login with your root token (until you setup a proper auth method). 
- You can use Vault CLI from your terminal by setting a `VAULT_ADDR` environment varaible to `https://vault.docker.example.com:8200`.
- You can also use Terraform (my favorite)

To play with Terraform you will need to setup your provider. A very simple way to start is to set it like this:

```go
provider "vault" {
  address = var.vault_addr
}

```

`var.vault_addr` will be set as `https://vault.docker.example.com:8200` and you will provide a vault token using environment variable named `VAULT_TOKEN`. You can change this as your setup gets more refined.

# 9- Conclusion
That's it!
We've covered the major steps to have a 3 nodes Vault cluster in docker, using Traefik as entrypoint, but using vault nodes' own certificates. You can now play with UI, CLI or Terraform to setup the cluster, try HA capability by shutting a node etc.
It has to be said that we are in a not ideal configuration for production (but that was never the point), using Vault HA behind a load balancer without direct access to the nodes. Please read more [here](https://developer.hashicorp.com/vault/docs/concepts/ha).
