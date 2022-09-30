---
layout: post
title:  "Terraform workflow and lifecycle"
description: My 2cts about IaC management
date:   2021-10-03 08:00:00 +0100
tags: terraform
---

> Disclaimer:<br>
I am by no means a software engineer. Just an Ops who's been sailing the devops seas, trying to make to most out of it (still working on it ^^)
This is an opinionated view infrastructure lifecycle and infrastructure code management

## Introduction
It’s been around for quite some time now, in application development, we’re getting away from monoliths in favor of micro services. It’s more than a trend, it’s the (not-so)new standard.<br>
At the same time, with the rise of cloud computing, we changed the way we manage infrastructure. More specifically Infrastructure as Code bring a lot to the table. Now concepts like cattle, immutability, idempotency are proving extremely valuable for your infrastructure deployment reliability. What lessons learnt in software development can we apply to IaC?



## GitOps
It may seem obvious nowadays but, version control systems (vcs), like git, are the gold standard for any code production. So if we want to benefit from dev’s practice in our ops world, needless to say that gitops is the way to go.
Thus, in order to be effective as a team, and reliable/repeatable as a platform we will:
- use IaC (Terraform) to provision infrastructure
- use Git to save/share/version our code
- test our infrastructure code before submitting it to merge
- use automation (CI/CD) to actually deploy said coded resources

That said, GitOps does not limit itself to infrastructure provisioning. GitOps suggest that every bit of OPS configuration and setup shoud be "GITed". This helping you deal with configuration drift, cost and capacity planning, inventory, regression issues etc.

## Segmentation
As we transpose dev workflow to infrastructure provisioning I find it quite logical to be faced with the same caveats. 
The first to come to my mind is segmentation. In introduction, I talked about the aging monolith model. Anyone who’s had to face a monolith application migration project, knows the challenges it involves. So don’t go that road with you IaC.<br>
You could be tempted to write a big project with everything inside, especially if you’re doing greenfield in a go-to-cloud project. But in the long run, you will get into trouble.<br>
Think of *blast radius*. <br>
If all your infrastructure is deployed and maintained in the same terraform project, each and every modification will:
- potentially have undesired effect on other resources
- make your terraform plans and apply longer and longer
- make your Git workflow a nightmare as your teams and project scale up
- halt your whole deployment capabilities while you need to deal with drift issues
- ...

## Lifecycle
Then, if having one big repository is bad, what do you recommand? How do you suggest we segment our infrastructure?

One word: LIFECYCLE

You should always think of your infratructure lifecycle. Network resources tend to have far longer life than virtual machines, a container is by design, ephemeral, etc. <br>
My recommandation is to use this idea of lifecycle to separate your IaC projects into smaller chunks. I'll probably get mixed feelings for saying this but I even recommand using one repository "per lifecycle group". For instance, you should be able to `terraform destroy` the web tier of an 3-tiers application, without touching your business or data tiers. Makes sense doesn't it?

Let's elaborate on this example. You have a 3-tiers application with multiple virtual machines for each tier (yes nowadays containers should be better alternative, but let's stick with VM for the sake of the example). Your dev team is delivering a new version of the web front end which is deployed on your web tier VMs. Now if you want to Canary your deployment, you will want to provide new VMs for your new version to be deployed (because remember, we love immutable infrastructure ^^) and load balance between your current version and your new version. <br>
What are your options regarding your infrastructure code management?
- *What about having one repo with all you tiers?*<br>
Actually, it might work for a small project but as you scale, problems will appear. For instance, you need to upgrade your tf provider to leverage some features for your new web vms, but this provider deprecates configuration used on your other tiers vm. Hello Yak Shaving...
- *What about having one repo with a subfolder by tier?*<br>
Seems better! Now, your terraform "run cycles" will be separated between your tiers making it more flexible. You can roll out your new web VMs without worrying about other tiers providers/modules dependencies. However, if you need to tag/version your infrastructure code (for roll back capability or compatibility matrix for instance), this may become a hindrance in some occurences. You can't rollback*  one tier without rolling back all the other tiers.<br>
*I'm talking about infrastructure verions here, not software.
- *What about having one repo per tier?*<br>
Same advantages as above and also more flexibility regarding infrastructure versions rolling. But, on the other hand, that means having hell of a lot more repositories for your whole infrastructure because your projects are smaller.

There is no silver bullet, you will probably be better of mixing the two last options. 
Although it may seem that you'll be multiplying similar infrastructure code (in this example: VMs), with the use of modules and meta-arguments (like hcl for_each) you should be able to keep it rather DRY.

## About environments
A note regarding dealing with environments.

## Paradygm shift