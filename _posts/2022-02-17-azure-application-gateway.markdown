---
layout: post
title:  "Azure Application Gateway"
description: Deploy Azure Web Application Firewall with terraform
date:   2022-02-17 20:00:00 +0100
tags: azure terraform
---

## Introduction
I have to be honest about that one. I have weird love-hate relationship with Azure Application Gateway. All in all, I think it is a really great product and you can't go wrong with it. I'd go as far as say that if you need to expose public web services, Application Gateway is a must-have, unless you need multi-region redundancy and scaling then Front Door with Traffic Manager will be your best bet.
I'll show you in this piece, how I deal with the restrictions of the current azurerm provider (and probably the way Azure API is build) for this service. But first, let's dive into what the Application Gateway is.

## What is Application Gateway?
We'll be focusing on the WAFv2 sku throughout this post. So, in advance, sorry if you use other service levels and find discrepancies compared to your setup or experience. 
As usual, you'll find far more details about the product on the [official documentation](https://docs.microsoft.com/en-us/azure/application-gateway/), the objective is to present the service and how to deploy it with terraform.

## Components 
A typical configuration for one site would look like the diagram below.<br>
<br>
![Application Gateway](/pictures/blog-applicationgateway.drawio.png)

> Imagine that you can repeat this pattern for multiple sites (with or without custom probes depending on your needs).

We'll explain below what these components are.

### - Front End IPs and ports
Frontend IPs and ports are the web sockets you expose. In v2 skus, the gateway comes with 2 IPs, one private and one public. Feel free to use both or just one depending on your requirements. 
I haven't tried any exotic ports number, but so far I didn't see any limitation.
**Be aware however that a port number can be associated to only one Frontend IP**. So if you have one listener using 443 with your public frontend, you can't use 443 with your private IP. That restriction led me to sometime deploy 2 application gateways, one for private services and one for public services. 
I found that odd as, to me, it defeats the purpose of having both interfaces...

### - Listeners
Listener are your endpoints to access your services. In this aspect, Application Gateway is more similar to a reverse proxy than a loadbalancer. Just like you would setup numerous virtual hosts on an apache web server, the azure service allows you to setup up to 40 listeners for WAF enabled skus (100 otherwise).

### - Http settings
You will make all your behavioural configuration here. To name a few, this is where you setup protocol (http vs https), cookie affiny, connection draining, etc. This is, also, where you specify on which port your backend is listening. Meaning, you don't have to publicly expose the same port that your service use behind the curtain. It can be useful in some cases, but if you do end-to-end TLS on port 443, you probably listen on tcp 443 on both gateway frontend and backend.

### - Routing rules
Just like shown in the previous diagram, routing rules stitch together the listener with the http settings and the backend pool. We should mention here that the service offers the possibility to route traffic based on path. This can be very useful to manage content or if your web application is microsegmented.

(yes I stole that picture from official documentation ^_^)<br>
![Path Based Routing](/pictures/blog-applicationgatewaypathrouting.drawio.png)

### - Backends
The backends are the compute instances where your service is actually hosted. It can be VMs, VMSS, Web App. It's important to note that backend health is tested against root path, by default. So if your site can't return a valid code (200-399) to the gateway at this address, you will have to setup a custom probe.

### - Probes
When probing your service, you sometimes need specific healthcheck URL and/or ports. You can create custom probes for this. You'll be able to set timeout, intervals, protocol, port, path etc., specific to your service.

### - Web Application Firewall
The big caveat of the default deployment model is that there is one global WAF setting. When you decide to use OWASP 3.0, 3.1 or 3.2 ruleset, custom cyphers and prevention vs detection mode, it applies to all your sites. Which doesn't make much sens because, if you worked with WAF and OWASP ruleset before, you know that you inevitably endup setting up some exceptions. But because you decide to inhibate a few rules for a site doesn't mean you want those same rules disabled for another.

Fortunately Azure presents us the possibility to provision "autonomous" WAF Policies which you can associate with listeners. Actually this is how you do WAF with Azure Front Door.
In this scenario you end up with something looking like this (I removed the backend pool to keep it readable):
![WAF Policy Per Listener](/pictures/blog-applicationgatewaymutipolicy.drawio.png)

You will still have a global WAF configuration, but you can put it in Detection Mode only and manage each site rules on its dedicated WAF Policy. Neat!

### - Other useful stuff
Among other things, this managed service can be zone redundant, and can autoscale. It's also a good idea to setup its diagnostic settings to a log analytic workspace. Especially if you plan to use Azure Sentinel dosn the line.
Also since a WAF can be quite severe with threat evaluation and in many case will block legitimate trafic, you can use Log Analytics' KQL to identify matched rules. Below is an query example (juste replace `my_listener_name` with your own setting)

```kql
AzureDiagnostics
| where Category == "ApplicationGatewayFirewallLog"
| where policyScopeName_s == "my_listener_name"
| sort by TimeGenerated

```

Finally, Application Gateway comes with URL Rewrite feature. I tried it to setup HSTS once, and it worked like a charm.

## Can we Terraform it?
Yes we can! But, yes, there is a but, and this is where the love affair starts to crumble. The way I (like to) use Terraform is by splitting my infrastructure code in small chunks. <br>
I organize and separate things by lifecycle and dependancy, and leverage data sources to reference resources provisioned in other terraform projects (or outsite terraform altogether). For instance, I don't see any benefit in provisioning VMs in the same code I use to build my VNets. Networks have very long lifecycles whereas compute tends to have shorter and shorter ones nowadays. But since a NIC requires its subnet information, we can use the azurerm_subnet data source to put the pieces together.<br>

If you follow that logic, it would make sens to provision your listener and backends in the same terraform project that provisions your compute (which essentially is your backend). You know, they kinda share the same lifecycle, right?* <br>
Well you just can't. There are no dedicated azurerm resources for the components of your Application Gateway. You have to setup everything with the gateway itself. Like one big monolith...<br>

> *It should be noted that Azure provides an addon (named AGIC) for its managed Kubernetes Service, that uses Application Gateway as an Ingress Controller, which does exactly what I describe: manage Application Gateway's sites lifecycle together with the backend services. Food for thoughts :)

If you've been reading my other posts (I hope you did !), you know that I like to separate HCL "code" from configuration. So just like we did in the [vm module](https://nfrappart.github.io/2022/02/04/terraform-templating-with-json.html) I use json to template my web services and build locals to provision the resources with terraform and for_each. The big difference here is that I had to use dynamic bloc because of how the Application Gateway works.

This is how the json looks like:

```json
{
    "my":{
        "hostname":"my.domain.com",
        "redirect":false,
        "waf_policy":true,
        "backend":{
            "fqdns":[
                "node1.my.domain",
                "node2.my.domain",
                "node3.my.domain"
            ]
        },
        "listener":{
            "certificate":"domain.com"
        },
        "settings":{
            "cookie_based_affinity":"Disabled",
            "port":"443",
            "protocol":"Https",
            "request_timeout":"60",
            "probe":{
                "path":"/healthcheck",
                "interval":"60",
                "protocol":"https",
                "timeout":"30",
                "threshold":"3",
                "port":"443",
                "match":["200"]
            }
        }
    }
}

```

Then you'd iterate for as many site as you'd like (well in the limit of 40 listeners because we're using WAFv2 sku). You should be able to correlate the attributes you see here, with the components I described in the previous section.<br>
Since I wanted to be able to redirect http to https, I added a "flag" for it that I simply use as a condition in the hcl code. For now, the way I did it, there is a constraint on the listener naming. For it to work you need to create both http and https service in the json, and you have to use the same name and append `_80` to the http one. <br>
Then terraform we create a redirection to the listener, like so:

```json
{
    "my-other":{
        "hostname":"my-other.domain.com",
        "redirect":false,
        "waf_policy":true,
        "backend":{
            "fqdns":[
                "node1.my.domain"
            ]
        },
        "listener":{
            "certificate":"domain.com"
        },
        "settings":{
            "cookie_based_affinity":"Disabled",
            "port":"443",
            "protocol":"Https",
            "request_timeout":"60",
            "probe":{}
        }
    },
    "my-other_80":{
        "hostname":"my-other.domain.com",
        "redirect":true,
        "waf_policy":false,
        "backend":{},
        "listener":{},
        "settings":{
            "cookie_based_affinity":"Disabled",
            "port":"80",
            "protocol":"Http",
            "request_timeout":"60",
            "probe":{}
        }
    }
}
```
Also note that the way it is coded, every listener will be https unless http_settings port is 80 in which case the listener will be http. This is a limitation of my code, not the product.
It's not very elegant but I fulfills my current needs.
<br>
A very important detail you probably notice is the use of SSL certificate. Since we're in Azure we can leverage native solutions like Keyvault to store sensitive information like the SSL certificate. To make it work you have to create a *user assigned managed identity* and assign it to the application gateway. Then, create an access policy in the keyvault to authorize the gateway to retrieve the certificate.
<br>
As for the WAF policies, the same "flag" principle is used with the attribute `waf_policy` which accept true or false. There is a lifecycle on the waf policy to ensure Terraform will just provisions it and is done with it. This allows me manage the rules outside terraform  which, for now, is more convenient for me.

Now onto the terraform code:

```hcl
locals {
  prefix = substr(join("", regexall("[a-z]+", lower(var.customerName))), 0, 3)
  appGw = jsondecode(file("appgw.json"))
  appGwIP = cidrhost(tostring(data.azurerm_subnet.appGwSubnet.address_prefixes[0]),4)
  frontendPort = toset(distinct([ for k,v in local.appGw : v.settings.port ]))
  backend = { for k,v in local.appGw : k=>v.backend if v.redirect == false}
  settings = { for k,v in local.appGw : k=>merge({hostname=v.hostname},v.settings) if v.redirect == false }
  listener = { for k,v in local.appGw : k=>merge(v.listener,{hostname=v.hostname,frontend_port="Port_${v.settings.port}",waf=v.waf_policy}) }
  probe = { for k,v in local.settings : k=>v.probe if v.probe != {} }
  redirect = { for k,v in local.appGw : k=>v if v.redirect == true }
}


# Tenant and sub config
data "azurerm_client_config" "current" {}

########################################################
# Adapat this section for your existing infrastructure #
########################################################

# Subnet for the application gateway
data "azurerm_subnet" "appGwSubnet" {
  name                 = "appGwSubnet" 
  virtual_network_name = "myVnet" 
  resource_group_name  = "myVnetRg" 
}

# Keyvault Policy for certificate access
data "azurerm_key_vault" "kv" {
  name                = "myKeyvault"
  resource_group_name = "myKeyvaultRg"
}
########################################################

resource "azurerm_resource_group" "rgAppGw" {
  name = "rgAppGw"
  location = "francecentral"
  tags = {
    ProvisioningDate = timestamp()
    ProvisioningMode = "Terraform"
  }
  lifecycle {
    ignore_changes = [
      tags["ProvisioningDate"]
    ]
  }
}

# Managed identity for App Gw to access keyvault
resource "azurerm_user_assigned_identity" "appGwIdentity" {
  resource_group_name = data.azurerm_resource_group.rgAppGw.name
  location            = data.azurerm_resource_group.rgAppGw.location
  name = "appGwIdentity"
}

resource "azurerm_key_vault_access_policy" "appGwIdentityKvPolicy" {
  key_vault_id = data.azurerm_key_vault.kv.id
  tenant_id    = data.azurerm_client_config.current.tenant_id
  object_id    = azurerm_user_assigned_identity.appGwIdentity.principal_id

  secret_permissions = [
    "get",
  ]

  certificate_permissions = [
    "get",
    "getissuers",
    "list",
    "listissuers",
  ]
}

# public IP for private Application Gateway
resource "azurerm_public_ip" "appGwPub0" {
  name                = "appGwPub0"
  resource_group_name = azurerm_resource_group.rgAppGw.name
  location            = azurerm_resource_group.rgAppGw.location
  allocation_method   = "Static"
  sku                 = "standard"
  tags = {
    ProvisioningDate = timestamp()
    ProvisioningMode = "Terraform"
  }
  lifecycle {
    ignore_changes = [
      tags["ProvisioningDate"],
    ]
  }
}
resource "azurerm_web_application_firewall_policy" "policies" {
  for_each = { for k,v in local.listener : k=>v if v.waf==true }
  name                = "${each.key}-policy"
  resource_group_name = azurerm_resource_group.rgAppGw.name
  location            = azurerm_resource_group.rgAppGw.location

  policy_settings {
    enabled                     = true
    mode                        = "Prevention"
    request_body_check          = true
    file_upload_limit_in_mb     = 100
    max_request_body_size_in_kb = 128
  }

  managed_rules {

    managed_rule_set {
      type    = "OWASP"
      version = "3.2"
    }
  }
  lifecycle {
    ignore_changes = [
      managed_rules,custom_rules
    ]
  }
}

resource "azurerm_application_gateway" "appGw" {
  name                = "appGw"
  resource_group_name = azurerm_resource_group.rgAppGw.name
  location            = azurerm_resource_group.rgAppGw.location
  zones               = ["1", "2","3"]

  sku {
    name     = "WAF_v2"
    tier     = "WAF_v2"
  }

  autoscale_configuration {
      min_capacity = 2
      max_capacity = 3
  }

  gateway_ip_configuration {
    name      = "appGwGatewayIp"
    subnet_id = data.azurerm_subnet.appGwSubnet.id
  }

  dynamic "frontend_port" {
    for_each = local.frontendPort
    content {
      name = "Port_${frontend_port.value}"
      port = frontend_port.value
    }
  }

  frontend_ip_configuration {
    name                 = "appGw-frontend-pub"
    public_ip_address_id = azurerm_public_ip.appGwPub0.id
  }

  frontend_ip_configuration {
    name                 = "appGwFrontend"
    subnet_id = data.azurerm_subnet.appGwSubnet.id
    private_ip_address_allocation = "static"
    private_ip_address = local.appGwIP
  }

  identity {
    type = "UserAssigned"
    identity_ids = [azurerm_user_assigned_identity.appGwIdentity.id]
  }

  waf_configuration {
    enabled          = "true"
    firewall_mode    = "Detection"
    rule_set_type    = "OWASP"
    rule_set_version = "3.2"
  }

  ssl_policy {
    policy_type = "Custom"
    cipher_suites = [
      "TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384",
      "TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256",
      "TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA", 
    ]
    min_protocol_version = "TLSv1_2"
  }


  ssl_certificate {
    name = var.certName
    key_vault_secret_id = "${data.azurerm_key_vault.kv.vault_uri}secrets/${var.certName}"#data.azurerm_key_vault.kv.id
  }
  
  dynamic "backend_address_pool" {
    for_each = local.backend

    content {
      name  = "${backend_address_pool.key}_backend"
      fqdns = backend_address_pool.value.fqdns
    }
  }

  dynamic "backend_http_settings" {
    for_each = local.settings
    content {
      name                  = "${backend_http_settings.key}_settings"
      cookie_based_affinity = "Disabled"
      port                  = backend_http_settings.value.port
      protocol              = backend_http_settings.value.protocol
      request_timeout       = backend_http_settings.value.request_timeout
      probe_name            = backend_http_settings.value.probe != {} ? "${backend_http_settings.key}_probe" : null
      host_name             = backend_http_settings.value.hostname
    }
  }

  dynamic "http_listener" {
    for_each = local.listener
    content {
      name                           = "${http_listener.key}_listener"
      frontend_ip_configuration_name = "appGwFrontend"
      frontend_port_name             = http_listener.value.frontend_port
      protocol                       = http_listener.value.frontend_port != "Port_80" ? "Https" : "Http"
      host_name                      = http_listener.value.hostname
      ssl_certificate_name           = http_listener.value.frontend_port != "Port_80" ? http_listener.value.certificate : null
    }
  }

  dynamic "redirect_configuration" {
    for_each = local.redirect
    content {
      name = "${redirect_configuration.key}_redirect_https"
      redirect_type = "Permanent"
      target_listener_name = "${trim(redirect_configuration.key,"_80")}_listener"
      include_path = "true"
      include_query_string = "true"
    }
  }

  dynamic "request_routing_rule" {
    for_each = local.appGw

    content {
      name                       = "${request_routing_rule.key}_rule"
      rule_type                  = "Basic"
      http_listener_name         = "${request_routing_rule.key}_listener"
      backend_address_pool_name  = request_routing_rule.value.redirect == false ? "${request_routing_rule.key}_backend" : null
      backend_http_settings_name = request_routing_rule.value.redirect == false ? "${request_routing_rule.key}_settings" : null
      redirect_configuration_name = request_routing_rule.value.redirect == true ? "${request_routing_rule.key}_redirect_https" : null
    }
  }

  dynamic "probe" {
    for_each = local.probe
    content {
      name = "${probe.key}_probe"
      path = probe.value.path
      interval = probe.value.interval
      port = probe.value.port
      protocol = probe.value.protocol
      timeout = probe.value.timeout
      unhealthy_threshold = probe.value.threshold
      pick_host_name_from_backend_http_settings = true
      match {
        status_code = probe.value.match
      }
    }
  }

  depends_on = [
    azurerm_key_vault_access_policy.appGwIdentityKvPolicy
  ]
}

```

That's it folks!
I hope this helps you with your own project wether you use this code as is for test or customize your own, learning from it.

Cheers!