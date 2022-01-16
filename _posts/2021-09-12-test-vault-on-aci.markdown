---
layout: post
title:  "Hashicorp Vault on ACI"
description: Use Azure Container Instance to test/play with Vault
date:   2021-09-12 22:00:00 +0100
tags: vault azure
---

## Introduction
The moment I heard about Vault, I wanted to try it out. But the although I could use dev mode to test it locally, or even, run a local instance with the file backend, I quickly wanted "more", but without breaking the bank.
Enters Azure Container Instance (ACI) which allows you to run containers in Azure, more or less like Docker as a Service.

## What is ACI
ACI, as stated above, is basically a managed Docker platform. You can set it up with AzCLI, or docker CLI. It even supports running compose applications? You'll find what you need on both [Docker Documentation](https://docs.docker.com/cloud/aci-integration/) and [Azure Documentation](https://docs.microsoft.com/en-us/azure/container-instances/container-instances-quickstart).
What is great with ACI is that you are billed based on CPU/GPU/Memory consumption per second (!). Now, that is an awesome trick for Testing and PoC, but also for simple task automation for example, needing a Container Runtime. In addition, in can be either public or VNet integrated depending on your needs.

By Default, an ACI is "public", no Virtual Network needed, I guess it works just like a Docker bridged network. If you want to persist data and run multiple container images sharing the same lifecycle, as a container group, you get roughly the below components:

![ACI](/pictures/blog-aci.drawio.png)

## The Objective
When I thought about making this little project, I had a few key points in mind:
- Keep cost low
- be able to persist data and configuration
- use terraform to deploy
- project must be self-sufficient - keep it simple !

### - Cost
Since Container Instances (or Container Groups) are per-second billed I can just destroy the instance and redeploy it when I want to play with it. Container Images being immutable, it doesn't matter, I don't save data in the container itself.

### - Data
There are however datas I need to persist. Those being any data written to vault backend, or container configuration file. This is achieved byt using a storage account with file shares mounted as container volumes (as show in the diagram above)

### - Provisioning
I wanted for this project to provision everything using terraform. Terraform has become my go-to provisioning tool, in fact, I terraform everything I can to keep my workflow persistent. So why not try to do that with ACI ?

### - No depedencies
I had to make a few concessions here. Since I want this project to be self-sufficient, I dont want to have to deal with network dependencies like VPN, domain name, certificates etc. Why? Because, among other thinks, I always want to be able to share that kind of project with the community, and not everyone has a domain name, a wildcard SSL certificate or a let's encrypt bot, etc. etc.
So I had to accept the fact that I'll use self-signed certificate for TLS, which is definitely NOT okay for production, but we're just testing here. No VPN nor

## The Architecture
After gathering enough information here is the target architecture:

![Architecture](/pictures/blog-vaultaci.drawio.png)


## Now go have fun !
Everything describe above is available on a public repository [here](https://github.com/nfrappart/lab-vault-on-aci).
I'll be posting other Vautl x Azure Architecture later, such as HA deployement, or Azure AD integration with OIDC auth method.