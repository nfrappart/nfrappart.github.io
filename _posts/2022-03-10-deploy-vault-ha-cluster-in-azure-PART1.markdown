---
layout: post
title:  "Deploy Vault HA in Azure - PART 1"
description: Vault HA with RAFT and Application Gateway in Azure
date:   2022-03-10 22:00:00 +0100
tags: vault azure
---

## Introduction
The first Hashicorp product I learned to use was Terraform (which I still learn btw ha!) . Almost immediately I started to hear/read about Vault. It took me quite some time before I had the time to dive into Vault, which I finally did last year by prepping the Vault Associate Certification. 
*Disclaimer, the Certification isn't that hard, you don't need extensive hours of running the solution in production, but you do need to master the core components, concepts and philosophy*.

## Architecture choices
Vault, as other Hashicorp software, is a single binary. So you can run it on your laptop, in a simple terminal. Is it a production ready environment? Absolutely not! But it is quite a treat to be able to test such high end solution by just typing a single command in your favorite shell.

Here is how you run it locally in dev mode (for testing only - evrything run in memory, nothing persistent):

```bash
vault server -dev
```

But if you want to run it in production, you'll have to make it a bit more reliable than that and we'll get into that. Thankfully, as usual, the [Official Documentation](https://www.vaultproject.io/docs) is very complete.
As an alternative, you could also use my public repo and spin up a vault instance on a container, running on Azure Container Instance (checkout my [previous article](https://nfrappart.github.io/2021/09/12/test-vault-on-aci.html) for more information)

### - Storage backend
You have some choice regarding where Vault will write presistent data. BUT, if you want to have a high availability, all are not usable. Eventually, you will end up, like most people, using either Consul (natural choice when you use it already) or the more recent Integrated Storage based on RAFT.
I might be mistaken but I think Hashicorp actually push people toward the latter. There is even some post on Hashicorp Learn, on how to migrate to Integrated Storage. Probably in this idea to have their product line made to work together, but without having to much adherences. But I may be wrong...

Anyway, if you're starting from the ground up and only deploying Vault, Integrated Storage is the obivous choice:
- HA compatible
- no need for another product (Consul, MySQL, etcd or whatnot... )

Integrated Storage uses RAFT consensus algorythm (just like etcd in kubernetes), to ensure fault tolerance. Minimum required cluster member is 3, which allows 1 member failure.

![RAFT](/pictures/vaultraft.png)
*(This diagram was proudly stolen from official documentation ^_^)*

### - Frontend High Availability (HA)
Vault fundamentaly is a web API. When you have multiple nodes, you must:
- establish which is master
- ensure request are forwarded or redirected to this master (there's actually some subtle differences [here](https://www.vaultproject.io/docs/concepts/ha))

HA works with DNS round robin. However you may not want or be able to use that solution. In many cases, you'll want to resort to Load Balancers.

## Leveraging Azure

A simple 3 node Architecture using some natives Azure solution to ease our work :)
Here the focus is on the Azure resources required to deploy Vault, and how *users* will consume secrets. That's why you find a VNG and WAN users. But of course, the real purpose is to help applications(i.e. machines) consume secrets. Those machines will be in the Vnet or peered Vnet(s) though they aren't showed on the diagram.

![Architecture](/pictures/blog-vaultha.drawio.png)

### - WAF
Since we deploy in Azure, we can go one step further and use Application Gateway, which is a Load Balancer, but also an L7 Firewall (a.k.a a Web Application Firewall).
This Managed Service will bring observability upon incoming trafic, as well as backend probing to determine viable Vault Node.
Actually, there is a dedicated path provided by the application to determine if node is active or standby, at `https://yourvaultclusterFQDN:8200/v1/sys/health`
- if it returns 200 : node is active
- if it returns 429 : node is standby
- anything else indicates an issue with the node

> Actually there are some [other codes](https://www.vaultproject.io/api/system/health) when you use Enterprise Edition which are still healthy status

**HOWEVER** Prepare yourself for some log inspection, because, as you can expect, if you enable *prevention mode* on the WAF, with, say... an OWAP 3.2 template ruleset... You inevitably will get legit trafic blocked by the gateway.

Below is a very simple, yet incredibly useful KQL request to help you list matched rule depending on your listener name.

```bash
AzureDiagnostics
| where Category == "ApplicationGatewayFirewallLog"
| where policyScopeName_s == "<yourListenerName>"
| sort by TimeGenerated

```

### - Azure Keyvault
Both Application Gateway and Vault require a valid SSL Certificate to expose the service, using https. By resorting to Azure Keyvault you kill two birds with one stone:
- since you need the certificate to start Vault, you can't store it *in* Vault (you know, the chiken-and-egg problem ^^)
- it'd be great to be able to keep your certificate in a resilient storage where you can manage access

For the latter, leveraging Azure Managed Identities brings a lot to the table. 

### - Azure Managed Identities
If you don't know yet about those, I encourage you to read more about them [here](https://docs.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/how-managed-identities-work-vm).
In our case we even have the luxury of choosing between 2 flavors for the Vault nodes. We can either enable *system assigned managed identity* per node, which is fully platform managed, or create a *user assigned managed identity* and associate it to all our nodes.

The Fundamental difference will be their lifecycle and the access delegation. The diagram below exhibits these differences. Note that in this cas we use Keyvault Access Policies and not RBAC based policies which may offer more granular access control for other certificates stored in the Keyvault.

**Using one identity per resource** 
(with a system assigned identity for each vm)

![system assigned identities](/pictures/blog-vaulthasystemmanagedidentity.drawio.png)

**Using one identity per resource type** 
(with one user assigned identity for all vault vm)

![system assigned identities](/pictures/blog-vaulthusermanagedidentity.drawio.png)

It will come down to your security and governance strategies. There's not really a better or worse in this case.

Below is the nifty command that allows you to retrieve your identity token so that you can authenticate against azure keyvault from a VM (you'll need to have uploaded the certificated with its key embedded):

```bash
azure_token=$(curl 'http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https%3A%2F%2Fvault.azure.net' \
    -H Metadata:true | \ 
    jq -r '.access_token')

```

You can then query the keyvault from the VM to retrieve the certificate and its key

```bash
# Retrieve the private key
kv=<your_keyvault_name>
certificate=<your_keyvault_certificate_name>

sudo curl https://${kv}.vault.azure.net/secrets/${certificate}/?api-version=2016-10-01 \
    -H "Authorization: Bearer $azure_token" | \
    jq -r ".value" | \
    base64 -d | \
    openssl pkcs12 -nocerts \
    -out ./vaultpkcs.key \
    -nodes -passin pass:

openssl rsa -in vaultpkcs.key -out vault.key

# Retrieve the certificate
sudo curl https://${kv}.vault.azure.net/secrets/${certificate}/?api-version=2016-10-01 \
    -H "Authorization: Bearer $azure_token" | \
    jq -r ".value" | \
    base64 -d | \
    openssl pkcs12 -nokeys \
    -out ./vault.cer -passin pass:

```

### - Log Analytics and Storage Account
Those we use for basic logging and diag settings. For instance, you need to link a storage account to the boot diagnostics if you want to be able to use the serial console on Azure Portal. Then you can use Log Analytics agent to stream your performance counters from your vms to your Log Analytics workspace for use with either Azure Monitor, Azure Sentinel, etc.


### - Azure Availability Zones
Last but not least having 3 nodes and WAFv2 sku Application Gateway, your Vault service will be Zone redundant as long as you set things right.
- 1 VM per Zone
- 1 Data disk per zone, with RAFT dealing with leader election and availability
- WAF minimum instance count higher than 2
- Virtual Network Gateway is active/passive (or you could make it active/active if need be)

![Availability Zones](/pictures/blog-availabilityzones.drawio.png)

## That's a wrap !
We've covered most of the infrastucture side with a few helpfull commands. Next time, we'll look into the terraform code to deploy the cluster and maybe the deployment and configuration of the application gateway (unless we leave it for a part 3).
You can get an idea of how we'll achieve the WAF part by reading the previous post about it [here](https://nfrappart.github.io/2022/02/17/azure-application-gateway.html).

