---
layout: post
title:  "Azure Hub & Spoke with Terraform"
description: Setup your Azure subscription core services
date:   2022-07-08 14:00:00 +0100
tags: terraform azure
---


## Introduction
You all know the 3 basic service levels a public cloud provide: SaaS, PaaS and IaaS. Let's put aside the first one and focus on the two others.<br> 
When you deploy an application in Azure, you can choose to use abstracted infrastructure and manage only your code and its deployment (PaaS) or have a finer control and manage the (virtual) infrastructure yourself (IaaS). Whatever the choice, at some point, you will need to use private network (as in rfc1918), which inevitably leads you to Azure Virtual Networks (aka VNets). The good people at Microsoft have a plethora of documentation, which is good and bad, as it can leave you perplex as to which way to go regarding network architecture.
In this post, we will discuss the `Hub and Spoke` architecture.

## The Objective
The idea is to give you the means to jump start your azure deployment, with some terraform code. Some may think `Landing Zone` but I prefer the term `Subscription Setup` as, to me at least, the landing zone idea includes Governance (with roles, groups, policies etc.) which I don't cover in this project. I find governance to be too specific for each customer, so defintely no `one size fits all` there.
> Full disclaimer here, what you're about to read is geared towards small businesses (in the 100-600 people range). Bigger environment will probably present the usual organizational challenge of having to deal with several teams before validating a target architecture. In which case, the notion of `Landing Zone` really start to make sense.

## The Target Architecture
So now that we are clear on the objective, let's take a look at the architecture. 
We will be going for the following network segmentation:
- Hub VNet (for connectivity and shared services)
- Prd VNet (for production workloads)
- stg VNet (for staging workloads)
- tst VNet (for testing workloads)
- dev VNet (for development workloads)

This is a very classic segmentation.

> Take into account that in this sample, everything is in a single subscription, but with a bit of clever tweaking, you can reuse the same code to deploy in multiple subscriptions (like one subscription per environment).

here goes (click on picture and zoom for larger view):
<a href="/pictures/blog-hubandspoke.drawio.png" target="blank">
    <img 
        src="/pictures/blog-hubandspoke.drawio.png" 
        alt="Hub & Spoke"
    >
</a>

The "transparent" subnets are there for demonstration sake, to show you the kind of services it would make sense to host in your Hub Vnet. But you can deploy them by filling the json accordingly (more on that later).
The Azure Bastion, Azure Firewall and Virtual Network Gateway however, are included in the project and can be deploy by a simple boolean variable.

## What are we deploying ?!
As you can see, we are deploying quite a few ressources. Fortunately, most of them will (almost) not cost you a penny until there's some actual traffic. Of course, this excludes Bastion Hosts, and any type of Gateway, since, like any compute resource, they generate cost.

### *Virtual Networks*
Obviously we are deploying Virtual Networks. They are at the center of this terraform project. Remember, the IP Spaces you choose for your VNets must not be used in any other network you plan to interconnect.
We will be provisionng subnets and VNet peering as well.

### *DNS Zones*
We deploy two type of zone. Public zone, for the services you wish to publish to the public network with certifcates, and private zone, so you can leverage VNet DNS auto-registration. This can be quite usefull for automation.
As a rule of thumb I generally use a subdomain of the public DNS zone for the private zone. 

### *Storage account*
Storage accounts are kind of a swiss army knife, and they come in cheap :)
We use only two here, for the sake of logging.
- one as boot diagnostic for futures VM. That will allow us to use the serial console, which is usefull.
- one as nsg flow log, so that we can keep a bit of networking observability. This is setup in conjunection with network watcher.

### *Keyvault*
If you read my blog, you know I very into Hashicorp Vault. But there are some cases where Azure Keyvault comes in handy. Moreover, if you're just starting your journey to the cloud, you might not be already familiar with vault so going the `native` route with Keyvault seems pretty reasonable ;)

### *Recovery Service Vault*
This is for backup purposes, plain and simple. If you provision VM down the line, and don't already have a backup solution, this is for you. Easy to setup, pay per use model. Not much to say about it.

### *Log Analytics*
Log Analytics Workspace are part of Azure Monitor Stack. They serve many purposes, one of which is to store your logs (hence the name). You can setup up policies to autoconfigure log streaming to a workspace from certain type of resources.
It is also required to use with Automation Accounts etc. VM agents use it to register and interact with Azure Monitor. 5GB of storage ingestion per month is Free.

### *Automation Account*
Another swiss army knife kind of tool. Although there are many functionality I don't use as they use more of a `mutable` approach, but still very usefull.
The most noticeable one is the abaility to run scheduled job, billed by minutes, instead of having to maintain dedicated VMs with cron or scheduled tasks.
But there are loads of other feature like inventory, update management, etc.

### *Azure Bastion (optional)*
If your infrastructure is Cloud only, you will need a way to rdp or ssh into your deployed VMs. The Bastion is here for that. So, even though it induces additionnal costs, I strongly advise to pass the option to `true` and deploy one.
In addition, the Standard sku allow you to use the `Native Client` feature. You can then ssh to target vm from your terminal instead of relying on azure portal.
Here is the az cli command for bash shell (choose auth-type according to your setup):

```bash
bastion_name=
bastion_resource_group_name=
target_vm_resource_group_name=
target_vm_name=

az network bastion ssh \
  -n ${bastion_name} \
  -g ${bastion_resource_group_name} \
  --target-resource-id \
    $(az vm show \
    -g ${target_vm_resource_group_name} \
    -n ${target_vm_name} |jq -r '.id') \
  --auth-type AAD
````

### *Virtual Network Gateway (optional)*
Again, an optional resource. You will need one for 2 main reasons:
- you want Express Route or IPSec connectivity to on-premises network
- you want to enable traffic routing between your spoke VNets

Let's take a moment to discuss the latter.
When you peer 2 VNets (let's say the hub VNet and a spoke VNet), they both inherit a system route in their route table so that traffic can be routed from one VNet to the other. Howerver, if you peer a third VNet to your hub VNet can't communicate with each other unless they are also peered together (full mesh).

![No Gateway Transit](/pictures/blog-hubandspokenotransit.drawio.png)

In our case, not being able to exchange data between production and staging, for instance, can be a blessing. But, if your network model includes other spokes that should be accessible by every other spokes, then you need a way to populate route tables. One way to solve this is to use a Virtual Network Gateway with `Gateway transit` enabled, on you Hub VNet. By doing so, every system route populated in the Hub Vnet system table, is propagated to peered VNets.

![Gateway Transit](/pictures/blog-hubandspoketransit.drawio.png)

### *Azure Firewall (optional)*
Finally, you have the option to deploy an azure firewall, in which case, there is an option in the json to specify if you want your next hop to be azure firewall or not, for each subnet definition.
However, this terraform project does not include any Azure Firewall configuration, apart from its deployment. The reason is to limit terraform plan blast radius. I'll talk about that in a future article, but the idea is to avoid having huge terraform states which are long to process and impact too many ressources at each run (smaller run means smaller risks).
More on Azure Firewall [here](https://docs.microsoft.com/en-us/azure/firewall/).

## About L4-L7 network security
L4 filtering is left to the network security groups. They are declared in the json, in each subnet section. There's a default rule with id 999 in the json, that allow all east-west traffic within each VNet.

You will have to sort out your required flow and define your rules accordingly. NSG will allow the use of service tag (built-in ones from azure such as "Internet" or "VirtualNetwork"), cidr blocks, Application Security Groups or "Any".

If any L7 filtering or scanning is required, you will have to rely on Azure Firewall, Application Gateway or other NVA (Network Virtual Appliances). For example, to use rules based on FQDN, you need Azure Firewall or an NVA, as NSG won't provide such flexibility. 
> I stronly recommand using NSG for east/west filtering and AzureFirewall/NVA for north/south control (or NSG for both). You could theoretically  forward intra-vnet traffic to your firewall in the Hub VNet but the network cost, in both performance and money, is a deal breaker.

## Why are DNS zones important ?
DNS private Zones are relatively simple as long as you stay in a "full" cloud environment. But if you need network hybridation, that's another story (for another post ;)).<br>

This project will make you deploy a private zone, but you will soon need more than one for your environments. When you will need/want to leverage private endpoints (to better control access to your managed services), you will end up with private zones like `privatelink.<managed_service_name.>.<some_microsoft_domain_name>`. This is how Azure returns private IP for a managed service instead of the public one (more on that in the [official documentation](https://docs.microsoft.com/en-us/azure/private-link/private-endpoint-dns)). So it's good pratice to get familiar with them from the get go.<br>

Private DNS Zones are subscription bound, meaning, you could theoretically have the same zone over and over on all your subscriptions, which is obviously bad as workload in each subscription would only get DNS response for the records in its own private zone. That is why the Hub & Spoke model suggest to use the "Hub" for your shared resources. Thus, you share the same private zone for all your connected workloads. This behavior is achieved by [dns virtual network links](https://docs.microsoft.com/en-us/azure/dns/private-dns-virtual-network-links).


## How to use the module:
Below is a tf file sample. But that's not all. The module requires you to provide the parameters in a json file named `networks.json` (see next section)

```hcl

provider "azurerm" {
  features {}
}
terraform {
  required_providers {
    azurerm = {
      version = "3.10.0"
    }
  }
}

data "azurerm_client_config" "current" {}

module "subsetup" {
  source = "github.com/nfrappart/terraform-az-modules/azTerraSubscriptionSetup?ref=v1.0.0"
  customerName        = "natedemo"          #required.
  privDomain          = "priv.natedemo.fr"  #required.
  pubDomain           = "natedemo.fr"       #required.
  deployVngIpsec      = false        #optional. Defaults to false.
  vngIpsecSku         = "VpnGw1AZ"   #optional. Defaults to VpnGw1AZ.
  vngErSku            = "ErGw1AZ"    #optional. Defaults to ErGw1AZ.
  deployVngEr         = false        #optional. Defaults to false.
  deployAzureBastion  = false        #optional. Defaults to false.
  deployAzureFirewall = false        #optional. Defaults to false.
  myTags = {                         #optional. Defaults to empty map.
    "provisionedBy" = "Nate",
    "usage" = "demo"
  }
}

# Most module ressources are available as output. 
# To use them, you need to specify the instance name (refer to the names you define in the json file)
# Then specify the usual resource attribute
# Example:
output "GatewaySubnet" {
  value = module.subsetup.subnet["GatewaySubnet"].address_prefixes
}

```

## Explaining Configuration file: networks.json
The json file has a predefined structure like below. A full example is provided in the next section.
> the `networks.json` file must be in the same directory as the tf file calling the module.

The Vnet part looks like this:
```json
{
  "hub":{
    "address_space":["10.10.0.0/16"],
    "dns_servers":[],
    "peerings":{
      (redacted...)
    },
    "subnets":{
      (redacted...)
    }
  },
  "spoke1":{
    (redacted...)
  },
  "spoke2":{
    (redacted...)
  }
}
```
The keys will be used to name your VNets. You can add as many bloc as you want as long as they hold all the necessary attributes.

`dns_server` attribute can be an empty list, in which case the VNet will use Azure default DNS.
`peerings`can be an empty map, but since the whole idea is to build a Hub&Spoke, you will obviously want to fill them.

Herre is how it looks:
```json
{
  (redacted...)
    "peerings":{
      "peerHubToSpoke1":{
        "resource_group_name":"rgCore",
        "remote_vnet":"spoke1",
        "subscription_id":"<your_sub_id>",
        "allow_virtual_network_access":"true",
        "allow_forwarded_traffic":"true",
        "allow_gateway_transit":"true",
        "use_remote_gateways":"false"
      },
      "peerHubToSpoke2":{
        "resource_group_name":"rgCore",
        "remote_vnet":"spoke1",
        "subscription_id":"<your_sub_id>",
        "allow_virtual_network_access":"true",
        "allow_forwarded_traffic":"true",
        "allow_gateway_transit":"true",
        "use_remote_gateways":"false"
      }
    }
  (redacted...)
}
```
Choose your options depending on your needs, but you can probably use them as above if you're starting. 

Don't forget to fill in your remote vnet `subscription_id`.
> subscription id is a trick to allow you to peer Vnets in different subscriptions. If all your infrastructure is in the same subscription, then use the same id everywhere for all your peerings

Keys will be used as name for your peerings.

`resource_group_name` is the resource group where your remote VNet is located.

For your Hub VNet, you will need as many peerings as spoke Vnets you have. On the other hand, in a spoke VNet you will most likely only need a peering toward hub VNet like so:

```json
{
  (redacted...)
    "peerings":{
      "peerSpoke1ToHub":{
        "resource_group_name":"rgCore",
        "remote_vnet":"hub",
        "subscription_id":"<your_sub_id>",
        "allow_virtual_network_access":"true",
        "allow_forwarded_traffic":"true",
        "allow_gateway_transit":"false",
        "use_remote_gateways":"false"
      }
    }
  (redacted...)
} 
```

The Subnet attribute is obviously the most complete one, see a redacted version here:
```json
{
  (redacted...)
    "subnets":{
      "adminHubSubnet":{
        "subnetAddressPrefix":["10.10.0.0/25"],
        "nsg":"true",
        "delegation":"none",
        "customTags":{
          (redacted...)
        },
        "inbound":{
          "999":{
            "name":"tempAllowVnetInBound",
            "access":"Allow",
            "protocol":"*",
            "source_port_range":"*",
            "destination_port_range":"*",
            "source_address_prefix":"VirtualNetwork",
            "destination_address_prefix":"VirtualNetwork"
          },
          "1000":{
            "name":"PermitIcmp",
            "access":"Allow",
            "protocol":"Icmp",
            "source_port_range":"*",
            "destination_port_range":"*",
            "source_address_prefix":"VirtualNetwork",
            "destination_address_prefix":"VirtualNetwork"
          },
          "4095":{
            "name":"PermitAzureLB",
            "access":"Allow",
            "protocol":"*",
            "source_port_range":"*",
            "destination_port_range":"*",
            "source_address_prefix":"AzureLoadBalancer",
            "destination_address_prefix":"*"
          },
          "4096":{
            "name":"DenyANY",
            "access":"Deny",
            "protocol":"*",
            "source_port_range":"*",
            "destination_port_range":"*",
            "source_address_prefix":"*",
            "destination_address_prefix":"*"
          }
        },
        "outbound":{},
        "routesToFirewall":{}
      },
      "AzureFirewallSubnet":{
        "subnetAddressPrefix":["10.10.0.128/26"],
        "nsg":"false",
        "delegation":"none",
        "customTags":{
          (redacted...)
        },
        "routesToFirewall":
          "Internet":"0.0.0.0/0"
        }
      }
    }
  (redacted...)
}
```
Let's dive into what we have here.

In the `subnet` bloc, the key will be used as your subnet name.

The `subnetAddressPrefix`is obviously required as your CIDR bloc for said subnet.

`nsg` attribute is a flag, if set to true, a Network Security Group will be provisioned and associated with the subnet. In that case, you **have to** provide `Inbound` and `Outbound` attribute which will define your NSG rules. Both of these can be empty maps if you don't know yet what rule to apply

`delegation` is for the case your subnet is to be delegated to a managed service, like some managed services (like flexible servers or container instances).

> In the sample below, I create `AzureBastionSubnet` and the Inbound/Outbound section provide all the required rules for secured Bastion Service.

`customTags` is a simple map to add your personalized default tags.

`routesToFirewall` is a map which, when not empty, will populate UDR and Route Table for the related subnet. This attribute will use keys as udr name in the format `subnetNameToKeyname`, and value as destination network (value must be a valid CIDR block), using Azure Firewall as next hop IP.

**OBVIOUSLY** to use this, you need to set the input variable `deployAzureFirewall` to `true` and provide an `AzureFirewallSubnet` in you subnet bloc (preferably in your hub VNet).

## Conclusion
To wrap up this post, let's take seat back and recap what we discussed:
- We can setup the core services and network architecture using the terraform project presented here. It's available on my [github](https://github.com/nfrappart/azTerraSubscriptionSetup)
- We separate code and configuration between `.tf` files and `.json` file
- The configration file must be named `networks.json`
- The structure behind the json file structure is explained above, block per block
- The repo can be used as a reusable module as shown in the example above
- You can alternatively clone the repo, customize it, and run it as a root module within your project



## Example with hub and one spoke:
```json
{
  "hub":{
    "address_space":["10.10.0.0/16"],
    "dns_servers":[],
    "peerings":{
      "peerHubToSpoke1":{
        "resource_group_name":"rgCore",
        "remote_vnet":"spoke1",
        "subscription_id":"00000000-0000-0000-0000-000000000000",
        "allow_virtual_network_access":"true",
        "allow_forwarded_traffic":"true",
        "allow_gateway_transit":"true",
        "use_remote_gateways":"false"
      }
    },
    "subnets":{
      "adminHubSubnet":{
        "subnetAddressPrefix":["10.10.0.0/25"],
        "nsg":"true",
        "delegation":"none",
        "customTags":{
          "Contact":"",
          "Comment":""
        },
        "inbound":{
          "999":{
            "name":"tempAllowVnetInBound",
            "access":"Allow",
            "protocol":"*",
            "source_port_range":"*",
            "destination_port_range":"*",
            "source_address_prefix":"VirtualNetwork",
            "destination_address_prefix":"VirtualNetwork"
          },
          "1000":{
            "name":"PermitIcmp",
            "access":"Allow",
            "protocol":"Icmp",
            "source_port_range":"*",
            "destination_port_range":"*",
            "source_address_prefix":"VirtualNetwork",
            "destination_address_prefix":"VirtualNetwork"
          },
          "4095":{
            "name":"PermitAzureLB",
            "access":"Allow",
            "protocol":"*",
            "source_port_range":"*",
            "destination_port_range":"*",
            "source_address_prefix":"AzureLoadBalancer",
            "destination_address_prefix":"*"
          },
          "4096":{
            "name":"DenyANY",
            "access":"Deny",
            "protocol":"*",
            "source_port_range":"*",
            "destination_port_range":"*",
            "source_address_prefix":"*",
            "destination_address_prefix":"*"
          }
        },
        "outbound":{},
        "nextHopFirewall":"false",
        "routesToFirewall":{}
      },
      "AzureFirewallSubnet":{
        "subnetAddressPrefix":["10.10.0.128/26"],
        "nsg":"false",
        "delegation":"none",
        "customTags":{
          "Contact":"",
          "Comment":""
        },
        "nextHopFirewall":"false",
        "routesToFirewall":{}
      },
      "GatewaySubnet":{
        "subnetAddressPrefix":["10.10.0.192/27"],
        "nsg":"false",
        "delegation":"none",
        "customTags":{
          "Contact":"",
          "Comment":""
        },
        "nextHopFirewall":"false",
        "routesToFirewall":{}
      },
      "AzureBastionSubnet":{
        "subnetAddressPrefix":["10.10.0.224/27"],
        "nsg":"true",
        "delegation":"none",
        "customTags":{
          "Contact":"",
          "Comment":""
        },
        "inbound":{
          "1000":{
            "name":"PermitIcmp",
            "access":"Allow",
            "protocol":"Icmp",
            "source_port_range":"*",
            "destination_port_range":"*",
            "source_address_prefix":"VirtualNetwork",
            "destination_address_prefix":"VirtualNetwork"
          },
          "1001":{
            "name":"PermitGatewayManagerInbound",
            "access":"Allow",
            "protocol":"tcp",
            "source_port_range":"*",
            "destination_port_range":"443",
            "source_address_prefix":"GatewayManager",
            "destination_address_prefix":"*"
          },
          "1002":{
            "name":"PermitBastionCommunication8080Inbound",
            "access":"Allow",
            "protocol":"tcp",
            "source_port_range":"*",
            "destination_port_range":"8080",
            "source_address_prefix":"VirtualNetwork",
            "destination_address_prefix":"VirtualNetwork"
          },
          "1003":{
            "name":"PermitBastionCommunication5701Inbound",
            "access":"Allow",
            "protocol":"tcp",
            "source_port_range":"*",
            "destination_port_range":"5701",
            "source_address_prefix":"VirtualNetwork",
            "destination_address_prefix":"VirtualNetwork"
          },
          "1004":{
            "name":"PermitBastionHttpsInbound",
            "access":"Allow",
            "protocol":"tcp",
            "source_port_range":"*",
            "destination_port_range":"443",
            "source_address_prefix":"Internet",
            "destination_address_prefix":"*"
          },
          "4095":{
            "name":"PermitAzureLbInbound",
            "access":"Allow",
            "protocol":"*",
            "source_port_range":"*",
            "destination_port_range":"443",
            "source_address_prefix":"AzureLoadBalancer",
            "destination_address_prefix":"*"
          },
          "4096":{
            "name":"DenyAnyInbound",
            "access":"Deny",
            "protocol":"*",
            "source_port_range":"*",
            "destination_port_range":"*",
            "source_address_prefix":"*",
            "destination_address_prefix":"*"
          }
        },
        "outbound":{
          "1000":{
            "name":"PermitSsh",
            "access":"Allow",
            "protocol":"tcp",
            "source_port_range":"*",
            "destination_port_range":"22",
            "source_address_prefix":"*",
            "destination_address_prefix":"VirtualNetwork"
          },
          "1001":{
            "name":"PermitRdp",
            "access":"Allow",
            "protocol":"tcp",
            "source_port_range":"*",
            "destination_port_range":"3389",
            "source_address_prefix":"*",
            "destination_address_prefix":"VirtualNetwork"
          },
          "1002":{
            "name":"PermitAzureCloud",
            "access":"Allow",
            "protocol":"tcp",
            "source_port_range":"*",
            "destination_port_range":"443",
            "source_address_prefix":"*",
            "destination_address_prefix":"AzureCloud"
          },
          "1003":{
            "name":"PermitGetSessionInformation",
            "access":"Allow",
            "protocol":"tcp",
            "source_port_range":"*",
            "destination_port_range":"*",
            "source_address_prefix":"*",
            "destination_address_prefix":"Internet"
          },
          "1004":{
            "name":"PermitBastionCommunication8080Outbound",
            "access":"Allow",
            "protocol":"tcp",
            "source_port_range":"*",
            "destination_port_range":"8080",
            "source_address_prefix":"VirtualNetwork",
            "destination_address_prefix":"VirtualNetwork"
          },
          "1005":{
            "name":"PermitBastionCommunication5701Outbound",
            "access":"Allow",
            "protocol":"tcp",
            "source_port_range":"*",
            "destination_port_range":"5701",
            "source_address_prefix":"VirtualNetwork",
            "destination_address_prefix":"VirtualNetwork"
          },
          "1006":{
            "name":"InternetOutbound",
            "access":"Allow",
            "protocol":"tcp",
            "source_port_range":"*",
            "destination_port_range":"*",
            "source_address_prefix":"*",
            "destination_address_prefix":"Internet"
          },
          "4095":{
            "name":"PermitAzureLbOutbound",
            "access":"Allow",
            "protocol":"*",
            "source_port_range":"*",
            "destination_port_range":"*",
            "source_address_prefix":"AzureLoadBalancer",
            "destination_address_prefix":"*"
          },
          "4096":{
            "name":"DenyAnyOutbound",
            "access":"Deny",
            "protocol":"*",
            "source_port_range":"*",
            "destination_port_range":"*",
            "source_address_prefix":"*",
            "destination_address_prefix":"*"
          }
        },
        "nextHopFirewall":"false",
        "routesToFirewall":{}
      }
    }
  },
  "spoke1":{
    "address_space":["10.11.0.0/16"],
    "dns_servers":[],
    "peerings":{
      "peerSpoke1ToHub":{
        "resource_group_name":"rgCore",
        "remote_vnet":"hub",
        "subscription_id":"00000000-0000-0000-0000-000000000000",
        "allow_virtual_network_access":"true",
        "allow_forwarded_traffic":"true",
        "allow_gateway_transit":"false",
        "use_remote_gateways":"false"
      }
    },
    "subnets":{
      "adminSpoke1Subnet":{
        "subnetAddressPrefix":["10.11.0.0/25"],
        "nsg":"true",
        "delegation":"none",
        "customTags":{
          "Contact":"",
          "Comment":""
        },
        "inbound":{
          "999":{
            "name":"tempAllowVnetInBound",
            "access":"Allow",
            "protocol":"*",
            "source_port_range":"*",
            "destination_port_range":"*",
            "source_address_prefix":"VirtualNetwork",
            "destination_address_prefix":"VirtualNetwork"
          },
          "1000":{
            "name":"PermitIcmp",
            "access":"Allow",
            "protocol":"Icmp",
            "source_port_range":"*",
            "destination_port_range":"*",
            "source_address_prefix":"VirtualNetwork",
            "destination_address_prefix":"VirtualNetwork"
          },
          "4095":{
            "name":"PermitAzureLB",
            "access":"Allow",
            "protocol":"*",
            "source_port_range":"*",
            "destination_port_range":"*",
            "source_address_prefix":"AzureLoadBalancer",
            "destination_address_prefix":"*"
          },
          "4096":{
            "name":"DenyANY",
            "access":"Deny",
            "protocol":"*",
            "source_port_range":"*",
            "destination_port_range":"*",
            "source_address_prefix":"*",
            "destination_address_prefix":"*"
          }
        },
        "outbound":{},
        "nextHopFirewall":"false",
        "routesToFirewall":{}
      },
      "aksSpoke1Subnet":{
        "subnetAddressPrefix":["10.11.252.0/22"],
        "nsg":"true",
        "delegation":"none",
        "customTags":{
          "Contact":"",
          "Comment":""
        },
        "inbound":{
          "999":{
            "name":"tempAllowVnetInBound",
            "access":"Allow",
            "protocol":"*",
            "source_port_range":"*",
            "destination_port_range":"*",
            "source_address_prefix":"VirtualNetwork",
            "destination_address_prefix":"VirtualNetwork"
          },
          "1000":{
            "name":"PermitIcmp",
            "access":"Allow",
            "protocol":"Icmp",
            "source_port_range":"*",
            "destination_port_range":"*",
            "source_address_prefix":"VirtualNetwork",
            "destination_address_prefix":"VirtualNetwork"
          },
          "4095":{
            "name":"PermitAzureLB",
            "access":"Allow",
            "protocol":"*",
            "source_port_range":"*",
            "destination_port_range":"*",
            "source_address_prefix":"AzureLoadBalancer",
            "destination_address_prefix":"*"
          },
          "4096":{
            "name":"DenyANY",
            "access":"Deny",
            "protocol":"*",
            "source_port_range":"*",
            "destination_port_range":"*",
            "source_address_prefix":"*",
            "destination_address_prefix":"*"
          }
        },
        "outbound":{},
        "nextHopFirewall":"false",
        "routesToFirewall":{}
      },
      "pgsqlSpoke1Subnet":{
        "subnetAddressPrefix":["10.11.251.128/25"],
        "nsg":"true",
        "delegation":"Microsoft.DBforPostgreSQL/flexibleServers",
        "customTags":{
          "Contact":"",
          "Comment":"Delegated to PgSQL Flexible Servers"
        },
        "inbound":{
          "999":{
            "name":"tempAllowVnetInBound",
            "access":"Allow",
            "protocol":"*",
            "source_port_range":"*",
            "destination_port_range":"*",
            "source_address_prefix":"VirtualNetwork",
            "destination_address_prefix":"VirtualNetwork"
          },
          "1000":{
            "name":"PermitIcmp",
            "access":"Allow",
            "protocol":"Icmp",
            "source_port_range":"*",
            "destination_port_range":"*",
            "source_address_prefix":"VirtualNetwork",
            "destination_address_prefix":"VirtualNetwork"
          },
          "4095":{
            "name":"PermitAzureLB",
            "access":"Allow",
            "protocol":"*",
            "source_port_range":"*",
            "destination_port_range":"*",
            "source_address_prefix":"AzureLoadBalancer",
            "destination_address_prefix":"*"
          },
          "4096":{
            "name":"DenyANY",
            "access":"Deny",
            "protocol":"*",
            "source_port_range":"*",
            "destination_port_range":"*",
            "source_address_prefix":"*",
            "destination_address_prefix":"*"
          }
        },
        "outbound":{},
        "nextHopFirewall":"false",
        "routesToFirewall":{}
      },
      "privateEndpointSpoke1Subnet":{
        "subnetAddressPrefix":["10.11.250.0/24"],
        "nsg":"true",
        "delegation":"none",
        "customTags":{
          "Contact":"",
          "Comment":"Reserved to PaaS Private Endpoints"
        },
        "inbound":{
          "999":{
            "name":"tempAllowVnetInBound",
            "access":"Allow",
            "protocol":"*",
            "source_port_range":"*",
            "destination_port_range":"*",
            "source_address_prefix":"VirtualNetwork",
            "destination_address_prefix":"VirtualNetwork"
          },
          "1000":{
            "name":"PermitIcmp",
            "access":"Allow",
            "protocol":"Icmp",
            "source_port_range":"*",
            "destination_port_range":"*",
            "source_address_prefix":"VirtualNetwork",
            "destination_address_prefix":"VirtualNetwork"
          },
          "4095":{
            "name":"PermitAzureLB",
            "access":"Allow",
            "protocol":"*",
            "source_port_range":"*",
            "destination_port_range":"*",
            "source_address_prefix":"AzureLoadBalancer",
            "destination_address_prefix":"*"
          },
          "4096":{
            "name":"DenyANY",
            "access":"Deny",
            "protocol":"*",
            "source_port_range":"*",
            "destination_port_range":"*",
            "source_address_prefix":"*",
            "destination_address_prefix":"*"
          }
        },
        "outbound":{},
        "nextHopFirewall":"false",
        "routesToFirewall":{}
      }
    }
  }
}
```