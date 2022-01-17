---
layout: post
title:  "Start Terraforming Azure"
description: Terraform with remote backend in Azure
date:   2021-09-01 12:10:00 +0100
tags: terraform azure
---

## Introduction
If you're here, you probably know what terraform is and want to use it for your Azure Environment. If so, then you know what the terraform [state](https://www.terraform.io/language/state) file is, and how vital it is for your infrastructure. It's basically the souce of truth specifying what should exist in the target environment. For more on Terraform and the State, checkout the official documentation, the guys at Hashicorp are doing a great job at keeping it up-to-date and well organized ([Offical Documentation](https://www.terraform.io/intro)).

Here is a diagram exhibiting state sollicitation during a typical Terraform workflow:

![Terraform Workflow](/pictures/blog-terraformworkflow.drawio.png)

The big picture here is that you can go quite far on your own using local state, but at some point you'll need to use a remote backend, which leads to the next question.

## Why remote backend ?
Saying the state is the source of truth for your infrastructure lets you grasp it's importance. As such, you'll need to:
- ensure it is highly available
- have it backed up
- secure its access

But apart from the obvious reasons above, why would you use a remote backend? Very simple: you probably don't work alone. Since Terraform asks the state about what is deployed, each time you run it, you need to share it with everyone involving in "coding" your infrastructure. 
That also means you have to manage your code with a Version Control System like git (but that's another topic).

![Remote Backend in Azure](/pictures/blog-terraformremotebackend.drawio.png)

> So instead of working locally and moving state file to a remote backend later, let's configure Terraform to use remote backend from the get go!

## Azurerm remote backend
Now that we are in an agreement that remote backend is the way to go, let's check out what is available to us.
Once again, the [Offical Documentation](https://www.terraform.io/language/settings/backends) gives a good look at our options. You can actually store your backend in a Consul cluster, in an AWS bucket, and quite a few others. But since we're here looking at working on Azure, we'll use `azurerm` which is a blob in a [Storage Account](https://docs.microsoft.com/en-us/azure/storage/common/storage-account-overview).

## Setting up resources
How do we make that work?
Well you'll need a few things: a storage account, obviously, to store your state file, a service principal so that terraform uses it's own identity and not yours, and preferably a keyvault, to store some secrets and avoid having them in clear text in the terraform .tf/.tfvars files.

You will also be required to have adequat roles on your Azure AD Tenant and target Azure Subscription. Az CLI is also required for most of this procedure, although you can do it through the portal. Every command I'll give you here is for *bash shell*. It won't work on powershell.


```bash
# Log into Azure
az login -o none

# Select your target subscription, replace with your own subscription id
az account set -s 0000000-0000-0000-0000-000000000000

# Create Resource Group
RESOURCE_GROUP_NAME=myRg
RESOURCE_GROUP_LOCATION=westeurope
az group create \
    -l $RESOURCE_GROUP_LOCATION \
    -n $RESOURCE_GROUP_NAME \
    -o none

# Create Storage Account
STORAGE_ACCOUNT_NAME=mySa$RANDOM
az storage account create \
    --name $STORAGE_ACCOUNT_NAME \
    --resource-group $RESOURCE_GROUP_NAME \
    --sku Standard_LRS \
    --encryption-services blob\
    --min-tls-version TLS1_2 \
    --location $RESOURCE_GROUP_LOCATION \
    -o none

# Retrieve Storage Account access key 
STORAGE_ACCOUNT_KEY=$(az storage account keys list \
    --resource-group $RESOURCE_GROUP_NAME \
    --account-name $STORAGE_ACCOUNT_NAME \
    --query [0].value -o tsv)

# Create Blob Container named `terraform` in storage account
az storage container create \
    --account-name $STORAGE_ACCOUNT_NAME \
    --name terraform \
    --account-key $STORAGE_ACCOUNT_KEY \
    -o none

```

Your backend, per se, is ready. But now you need an identity to authenticate terraform, and a way to securely store the credentials for this identity.
Let's create the service principal first!

```bash
# Create SP with Contributor Role
SERVICE_PRINCIPAL_NAME=sp-terraform-auth

# the output is stored in a variable so that the credential do not appear as clear text
mapfile -t SP_TERRAFORM< <(az ad sp create-for-rbac \
    --name $SERVICE_PRINCIPAL_NAME \
    --role Contributor \
    --query "[appId,password]" \
    -o tsv)
SP_TERRAFORM_ID=${SP_TERRAFORM[0]}
SP_TERRAFORM_SECRET=${SP_TERRAFORM[1]}
```

Let's create a keyvault to store these sensitive ids:

```bash
# Keyvault name must be globally unique and less thant 24 characters.
KV_NAME=mykv$RANDOM
az keyvault create \
    --name $KV_NAME \
    --location $RESOURCE_GROUP_LOCATION \
    --resource-group $RESOURCE_GROUP_NAME \
    -o none

# Add permission for sp to keyvault
az keyvault set-policy \
    --name $KV_NAME \
    --object-id $SP_TERRAFORM_ID \
    --secret-permissions backup delete get list purge recover restore set \
    --key-permissions backup create decrypt delete encrypt get import list purge recover restore sign unwrapKey update verify wrapKey  \
    --certificate-permissions backup create delete deleteissuers get getissuers import list listissuers managecontacts manageissuers purge recover restore setissuers update \
    -o none

# Register service principal secret to keyvault
az keyvault secret set \
    --name $SERVICE_PRINCIPAL_NAME-secret \
    --vault-name $KV_NAME \
    --value $SP_TERRAFORM_SECRET \
    -o none

# Register service principal id to keyvault
az keyvault secret set \
    --name $SERVICE_PRINCIPAL_NAME-id \
    --vault-name $KV_NAME \
    --value $SP_TERRAFORM_ID \
    -o none

```

That was a bit of work, but you're done !
Now you just need to add a `backend` block in your terraform provider configuration in your .tf files.

## Configure provider in TF code
Provider configuration is well documented on Hashicorp website. But here is what you should have in your `main.tf` file (or any other .tf file in your project folder for that matter):

```hcl
provider "azurerm" {
  features {}
 }
terraform {
  required_providers {
    azurerm = {}
  }
  backend "azurerm" {
    storage_account_name = "<my_storage_account_name>" #replace with the value stored previously in $STORAGE_ACCOUNT_NAME
    container_name       = "terraform"
    key                  = "terraform.tfstate"
  }
}
```

## We've got you covered ;)
There is far more litterature on Internet about how to setup your remote backend using Azure now, than there was when I started out. So to ease things up, I made a nifty bash script that does all you need on the cloud platform side, and even generate a `main.tf` file with the proper configuration.
Everything is available in a public github repository : [Azure TF Init Project](https://github.com/nfrappart/terraform-azure-init)

Note: 
Contrary to the guide above, the script will have you store your storage account key in your keyvault. You can actually query the access key directly if your credentials allow it (if you're contributor or owner of the subscription).
Storing the secret in the keyvault is not a bad practice but if for some reason you need to rotate the key, you'll have to update your keyvault. Or you can also manage storage account keys from keyvault directly (there are some documentations out there ^^).
