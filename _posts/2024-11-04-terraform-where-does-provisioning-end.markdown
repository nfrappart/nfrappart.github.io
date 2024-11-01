---
layout: post
title:  "Terraform for configuration: where does provisioning end?"
description: Considerations when designing an Infrastructure as Code (IaC) Stack 
date:   2024-11-04 09:00:00 +0100
tags: azure terraform devops
---

## Table of Content
- [Introduction](#1--introduction)
- [Scope](#2--Infrastructure-VS-Application)
- [Skills](#3--know-Your-Audience)
- [Ownership](#4--Organization-and-Governance)
- [AKS Automatic](#5--AKS-Automatic:-Microsoft-may-have-proven-me-right)
- [Conclusion](#8--Conclusion)

# 1- Introduction

In the ever-evolving world of Cloud and DevOps, there is a prevailing notion that Terraform is strictly for provisioning while Ansible handles configuration. Yet, real-world practices tell a more nuanced story: virtual machines (VMs) are sometimes provisioned using Ansible, and Terraform, when using specific providers, extends into configuration management. Where, then, does one draw the line?

As someone who practices Terraform extensively but lacks deep expertise in Ansible, I approach this topic from the perspective of a Terraform specialist. Keep this in mind as you progress through this article.

# 2- Infrastructure VS Application: scope

Nuances between provisioning and configuration are often debated. A common understanding is that provisioning applies to infrastructure while configuration pertains to systems and applications. However, infrastructure itself frequently demands configuration. In Azure, for example, Network Security Groups (NSGs) need detailed rules, and route tables require user-defined routes (UDRs). These tasks clearly fall under configuration.

This prompts the question: Should we provision resources with Terraform and then switch to another tool for their configuration? If that were the case, why do Terraform providers exist for tools like Vault? From a provisioning standpoint, a Vault cluster may just be a collection of VMs, but using the Vault provider, one can configure the cluster comprehensively—from setting up authentication methods to managing secret engines and policies. In fact, all HashiCorp's tools have a provider designed to *configure* them.

# 3- Know Your Audience: skills

Understanding who will use your Terraform code is crucial. Are your consumers experienced Terraform practitioners or software engineers with limited knowledge of the tool? This distinction shapes how you write your IaC. For example it helps you decide how opinionated your modules should be.

For experienced practitioners, you might assume advanced practices, such as modular designs and intricate workflows. On the other hand, if your audience consists of engineers unfamiliar with Terraform, simplicity and thorough documentation become key. 
Kowledge of the underlying infrastructure plays also a key role in your design. Your configuration may leave a lot of customizable options if your consumers are expert in Azure (if this is where you host your infrastructure), or at the other end of the spectrum, you may use a lot of predetermined configration (defaulted/computed values) to abstract as much as possible and make it straightforward. Balancing sophistication and usability ensures your IaC stack is accessible and maintainable by the intended teams.

Finally, you must consider if there are other tools in place to integrate with. This could be a vendor specific UI or configuration language, an agnostic tool like Ansible, or even bespoke scripts managed by your users. In that case you fall into the previously discussed situation where configuration is left outside of terraform and artifices like lifecycles should be used to ensure the tools do not dangeroulsy overlap each others settings.

# 4- Organization and Governance: ownership

The way an organization is structured greatly influences how Terraform is used and where its boundaries lie. For example, if there is a dedicated security team, are they responsible for managing firewall (or NSG) rules, or is this handled by the infrastructure or platform team? This division of responsibilities informs how IaC stacks should be segmented.

Segmenting ownership within the IaC stack promotes a least-privilege model and enhances team autonomy. Each team can manage its own components without needing constant coordination with a central platform team. This segregation not only supports clear lines of responsibility but also reinforces security best practices by aligning IaC segments with team boundaries.

# 5- AKS Automatic: Microsoft may have proven me right

One practical example of how IaC should facilitate complete, ready-to-use infrastructure is the AKS (Azure Kubernetes Service) Automatic offering. Provisioning an AKS cluster shouldn’t end with just the creation of the cluster. Instead, it should be delivered as a fully configured solution, incorporating necessary integrations and configurations that align with the company’s standards.

This might include Vault integration, joining service meshes, or deploying monitoring tools and ingress controllers. The goal is to offer a cluster that teams can use immediately, focusing only on the applications they need to deploy. AKS Automatic embodies this philosophy by simplifying the configuration burden for development and Kubernetes consumer teams.

# 6- Conclusion

The boundary between provisioning and configuration is fluid, shaped by an organization’s structure and the expertise of its teams. While Terraform excels in provisioning, its capabilities often extend into configuration, blurring conventional lines. Understanding your audience, aligning with governance models, and delivering opinionated, ready-to-use infrastructure are key considerations for maximizing the effectiveness of your IaC strategy.
A successful IaC project relies as much on high technical skills as on deep understanding of the dynamics between teams and organization's context. 

