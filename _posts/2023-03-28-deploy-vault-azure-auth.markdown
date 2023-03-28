---
layout: post
title:  "Setup Vault for Azure managed identities"
description: Leverage Azure auth method to use Azure Managed identities with Hashicorp Vault
date:   2023-03-28 14:00:00 +0100
tags: vault azure terraform azuread
---

## Table of Content
- [Introduction](#1--introduction)
- [Use Case](#2--use-case) 
- [What's going on when you use azure auth?](#3--whats-going-on-when-you-use-azure-auth) 
- [Azure AD Requirements.. or not?](#4--azure-ad-requirements-or-not)
- [Setup Vault Auth with CLI or Terraform](#5--setup-vault-auth-with-cli-or-terraform)
- [Terraform Azure VM with Vault Agent](#6--terraform-azure-vm-with-vault-agent)
- [Conclusion](#7--conclusion)


## 1- Introduction
Azure provides a very neat identity feature for its workloads: user assigned managed identities (UAI) and system assigned manged identites (SAI).
Such identities are basically special service principals attached to azure resources which allow them to authenticate against Azure AD, thus gain RBAC Roles on other azure resources. The guys from Microsoft will explain it better than I, so you can read more [here](https://learn.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/how-managed-identities-work-vm).
If you're interested to use them to authenticate against vault, you're probably familiar with the concept anyway, so no need to dive deeper. :p

## 2- Use case
What are we looking for here exactly?
If you've been using Azure Keyvault for secret delivery/consumption in Azure before, you probably used UAI or SAI to authorize your workloads to access Keyvault data. Wouldn't it be great to be able to do the same to leverage Vault functionalities?
That is exactly what Azure Auth Method will allow you to do: authenticate your azure workloads, using their UAI or SAI, and consume secret (static, dynamic or EaaS) from Vault.

There are two approach to do that:
- integrate with vault in application code (your dev team has to integrate with vault api)
- delegate said authentication to vault agent (an agent will proxy all vault related task to a local process)

As an ops, I will focus on the second. By using an agent, we can provide all vault features to our application without it needing to know about vault. Like Armon Dagdar says "the applications are blissfully unaware of vault". And that is a great way to switch your secret management to Vault, without touching your code, hence, accelerating adoption.

## 3- What's going on when you use azure auth?
What's happening when you setup your vault agent to use azure auth ?
Well below is a how my messy brain represents the workflow.

![vault agent auth](/pictures/blog-vault-azure-auth.png "auto-auth")

yeah... I know... You probably wouldn't want to live in my head xD

So, let's look closer.

1. We have our VM with its managed identity. The agent on the VM will contact the IMDS (Instance Meta Data Service) endpoint to request a token from Azure AD (JWT), using its managed identity. 
2. Azure AD will return said token which holds a whole lot of metadata such as tenant and subscription, but also information about the resource itself
3. the agent calls the api endpoint for the azure auth using the provided token
4. Vault checks the bearer token with Azure AD
5. Vault provides the agent with a vault token

From there, the agent will renew the token automatically.
If you look at my notes on the left side of the picture, you'll see the usual Vault mecanism:<br>
- For the `azure auth` you will configure one or more roles. These roles can be bound with attributes referenced in the azure ad token metadata, such as resource group name, subscription id, etc.<br>
- Then policies are attached to each role, which will determine what capabilities the vault token, delivered to the agent, holds.

That's how you determine which secrets your Azure VM can consume from Vault. A VM/VMSS will request a certain role, to get the desired policies. But if the bindings requirements are not met (for example VM is in the proper Subscription, but not the required resource group), vault will validate its identity with Azure AD, but will refuse to deliver a token with the requested role. 

The main takeaway is that, just like with oidc to authenticate users, Azure AD is used as a trusted identity provider for vault. It's just that we are not interested in humans this time, but in machines. And that's the very reason it makes sense to take advantage of UAI/SAI since there is nobody to enter creds or validate MFA challenge. The rest is all traditionnal Vault plumbing.


## 4- Azure AD Requirements.. or not?
As we did with OIDC Auth method, we will need a service principal from Azure AD to allow vault to verify managed identities. We will do that by creating an app registration with the proper authorizations on Azure RBAC. 

There is also another possibility if you host your vault server/cluster in Azure: <br>
You can  use an Azure Managed Identity (SAI or UAI), assigned to your vault VMs, and give these identities the proper azure RBAC Authorization.

Whichever of the above you choose here are the required authorizations:
<br>
`Microsoft.Compute/virtualMachines/*/read`<br>
`Microsoft.Compute/virtualMachineScaleSets/*/read`<br>

> Considering this blog is all about Azure and Hashicorp, we'll assume the Vault nodes are hosted in Azure from here, and use Vault VMs managed identities to setup the auth method. <br>
This way we can achieve a *password zero* configuration (!)

## 5- Setup Vault Auth with CLI or Terraform
As usual with Vault there are multiples way to get the job done. You can use the Vault API, the Vault CLI or Terraform. We will have a look at both CLI and Terraform. 

### 5.a- Using Vault CLI (and az CLI)
First, let's create a custom role with the authorization listed in the previous section.
Create a json file (let's call it `myrole.json`) for the role definition:
```json
{
    "Name": "Virtual Machine Reader",
    "IsCustom": true,
    "Description": "Can Read VM and VMSS to get identity - used by VAULT",
    "Actions": [
      "Microsoft.Compute/virtualMachines/*/read",
      "Microsoft.Compute/virtualMachineScaleSets/*/read"
    ],
    "NotActions": [
  
    ],
    "AssignableScopes": [
      "/subscriptions/00000000-0000-0000-0000-000000000000"
    ]
  }
```
Don't forget to customize the `AssignableScopes` for your environment. You can list multiple subscriptions if needed.

Now create the role in Azure with az CLI:
```bash
az role definition create --role-definition ./myrole.json
```

```bash
vaultvm="myVaultVm"
rg="myVaultRg"
principal=$(az vm show --name $vaultvm -g $rg --query "identity.principalId" -o tsv)
mysub=$( az account show --query "id" -o tsv)

az role assignment create --assignee $principal --role "Virtual Machine Reader" --subscription $mysub 
```
Obviously, in production with a 3 or 5 nodes cluster, you'll have to assign the role to each VM identity, if using System Assigned Identity. You can also go the User Assign Identity route and use a single identity for all your VMs, in which case, you have to assign the role to this identity only, and associate this identity to all your nodes.

Now that you're done with the Azure configuration, here is the Vault CLI config:

First, enable the auth method:
```bash
vault auth enable \
  -default-lease-ttl="1h" \
  -description="Auth method for vm/vmss authentication with managed identities" \
  azure 
```

Then you setup the tenant:
```bash
vault write auth/${VAULT_AZURE}/config \
    tenant_id=00000000-0000-0000-0000-000000000000 \ #desired tenant id
    resource=https://management.azure.com/ 
```

Finally you write at least one role:
```bash
vault write auth/azureidentity/role/myapp \
    policies="myapp-policy" \
    bound_subscription_ids=aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa \
    bound_resource_groups=rg-myapp
```

In this example only vm from tenant identified id `00000000-0000-0000-0000-000000000000`, in subscription `aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa` and resource group `rg-myapp`, will be able to request `myapp` role, which will grant capabilities from `myapp` policy.

### 5.b- Using Terraform 
You can of course do the same, using terraform which is my prefered method
(I won't go into the role definition and role assignment part, but you can also do it with terraform, obviously).

```hcl
data "azurerm_client_config" "current" {}

resource "vault_auth_backend" "azure" {
  type = "azure"
  description = "Auth method for vm/vmss authentication with managed identities"
  tune {
      default_lease_ttl = "3600s"
      max_lease_ttl = "3600s"
  }
}

resource "vault_azure_auth_backend_config" "azure" {
  backend       = vault_auth_backend.azure.path
  tenant_id     = data.azurerm_client_config.current.tenant_id
  resource      = "https://management.azure.com/"
}

resource "vault_azure_auth_backend_role" "myapp" {
  backend                         = vault_auth_backend.azure.path
  role                            = "myapp"
  bound_subscription_ids          = [data.azurerm_client_config.current.subscription_id]
  bound_resource_groups           = ["rg-myapp"]
  #token_ttl                       = 60
  #token_max_ttl                   = 120
  token_policies                  = ["myapp-policy"]
}
```

You're done.
At least for this simple example.

## 6- Terraform Azure VM with Vault Agent
A simple way to provision a VM with the vault agent, is to use terraform and either a custom script extension or a cloud-init config using user_data.

### 6.a- Cloud init config for Azure VM

```bash
#cloud-config
cloud_config_modules: 
  - runcmd

package_update: true
package_upgrade: true

packages:
  - jq
  - gpg
  - unzip
  - curl

write_files:
  - path: /etc/vault.d/config.hcl
    content: |
      vault {
        address = "https://vault.mydomain.com:8200"
      }
      auto_auth {
        method "azure" {
          mount_path = "auth/azure"
          config = {
            resource = "https://management.azure.com/"
            role = "myrole"
          }
        }
        sink "file" {
          config = {
            path = "/ramdisk/.vaulttoken"
          }
        }
      }
      cache {
        use_auto_auth_token = true
      }
      listener "tcp" {
        address = "127.0.0.1:8100"
        tls_disable = true
      }
  
  - path: /etc/systemd/system/vault.service
    content: |
      [Unit]
      Description=Vault
      Documentation=https://www.vault.io/
      Requires=network-online.target
      After=network-online.target
      ConditionFileNotEmpty=/etc/vault.d/config.hcl 

      [Service]
      User=vault
      Group=vault
      ProtectSystem=full
      ProtectHome=read-only
      PrivateTmp=yes
      PrivateDevices=yes
      SecureBits=keep-caps
      AmbientCapabilities=CAP_IPC_LOCK
      Capabilities=CAP_IPC_LOCK+ep
      CapabilityBoundingSet=CAP_SYSLOG CAP_IPC_LOCK
      NoNewPrivileges=yes
      ExecStart=/usr/bin/vault agent -config=/etc/vault.d/config.hcl
      ExecReload=/bin/kill --signal HUP $MAINPID 
      KillMode=process 
      KillSignal=SIGINT 
      Restart=on-failure 
      RestartSec=5
      TimeoutStopSec=30
      StartLimitInterval=60
      StartLimitBurst=3
      LimitNOFILE=65536

      [Install]
      WantedBy=multi-user.target

runcmd:
  - wget -O- https://apt.releases.hashicorp.com/gpg | gpg --dearmor | tee /usr/share/keyrings/hashicorp-archive-keyring.gpg
  - echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | tee /etc/apt/sources.list.d/hashicorp.list
  - apt update && sudo apt install vault
  - mkdir /ramdisk
  - mount -t tmpfs -o rw,size=32M tmpfs /ramdisk
  - echo "tmpfs  /ramdisk  tmpfs  rw,size=32M  0   0" | tee -a /etc/fstab
  - adduser vault --shell=/bin/false --no-create-home --disabled-password --gecos GECOS
  - chmod 755 /etc/vault.d/config.hcl
  - chown vault:vault /etc/vault.d/config.hcl
  - systemctl daemon-reload
  - systemctl start vault.service
  - systemctl enable vault.service
  - export VAULT_ADDR=http://127.0.0.1:8100
  - export VAULT_TOKEN=$(sudo cat /ramdisk/.vaulttoken)
```

### 6.b- Create an Azure VM with terraform
This is a really basic config, but if you're here, I guess you already know your way around Azure and Terraform and will probably use your own code, or even module to create the VM.

```hcl
# Storage account for VM diag
data "azurerm_storage_account" "diag" {
  name                     = "mystorageaccount"
  resource_group_name      = "rg-mystorageaccount"
}

# Subnet for the VM NIC
data "azurerm_subnet" "subnet" {
  name                 = "mysubnet"
  virtual_network_name = "myvnet"
  resource_group_name  = "rg-myvnet"
}

# Create Password for vm
resource "random_password" "password" {
  length           = 16
  special          = true
  min_lower        = 1
  min_numeric      = 1
  min_special      = 1
  min_upper        = 1
}

# Create Resource Group
resource "azurerm_resource_group" "rg" {
  name     = "myrg"
  location = "francecentral"
}

# Create NIC for the VM
resource "azurerm_network_interface" "nic" {
  name                = "myvm-nic0"
  resource_group_name = azurerm_resource_group.rg.name
  location            = "francecentral"

  ip_configuration {
    name                          = "internal"
    subnet_id                     = data.azurerm_subnet.subnet.id
    private_ip_address_allocation = "Dynamic"
  }
}

# Create VM
resource "azurerm_linux_virtual_machine" "vm" {
  name                            = "myvm"
  computer_name                   = "myvm"
  resource_group_name             = azurerm_resource_group.rg.name
  location                        = var.location
  size                            = "Standard_D2s_v3"
  admin_username                  = "localadm"
  admin_password                  = random_password.password.result
  disable_password_authentication = "false"

  network_interface_ids = [
    azurerm_network_interface.nic.id,
  ]
  boot_diagnostics {
    storage_account_uri = data.azurerm_storage_account.diag.primary_blob_endpoint
  }

  identity {
    type = "SystemAssigned"
  }

  os_disk {
    name                 = "myvm-OsDisk"
    caching              = "ReadWrite"
    storage_account_type = "Standard_LRS"
    disk_size_gb         = "64"
  }

  user_data = base64encode(file("./cloud-init.yaml"))

  source_image_reference {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-focal"
    sku       = "20_04-lts-gen2"
    version   = "latest"
  }
}
```

## 7- Conclusion
I hope this piece inspired you to try using azure auth for Vault. I really found the integration with Azure to be really good, with either oidc, to authenticate users or azure to authenticate workloads. <br>
Hashicorp also announce to now support azure auth for App Services. I wanted to give it a try, but I honestly lack the time and don't have enough programing language knowledge to make a proper demo.