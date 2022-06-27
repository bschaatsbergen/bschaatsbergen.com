---
title: "The lost love for writing Terraform modules"
date: 2022-06-26T11:10:11+11:00
tags: ["terraform"]
---

The ability to write effectively and elegant Terraform modules has long been revered and rewarded.
Though, I see too many modules that are not written in a way that is easy to understand, they contain a lot of complexity and try to be the "complete package".

I'm writing this post to share my thoughts on the art of writing elegant Terraform modules by using four key principals:

- Do one thing and do it right
- Balance repetition and complexity
- Use industry-standard tooling
- Don't cover what you don't know

Far from your garden-variety style guide, this post covers the art of writing Terraform modules in a nutshell. Writing great modules is a skill, and this post should push you in the right direction to make your modules shimmer on Github.

## Do one thing and do it right

As I mentioned in the introduction, unfortunately it seems that the most popular modules try to be the "complete package" by supporting every single possible scenario. This easily turns your module into a monolithic blob of code, and it's not easy at all to understand what's going on and how to maintain it.

Terraform modules should do nothing more but preventing repetition from an end-user perspective.

**A module should do one thing, and do it right. If you want to do more than one thing, you should create another module.**

This might result into code duplication but this ties in with the idea of modularity and balancing repetition and complexity the right way.

For example, Google Cloud its network and subnetworks are logically isolated from one another. Meaning, you should create a module for a Google Cloud network, and another module for the Google Cloud subnetworks instead of trying to fit them into a singular _"Google Cloud VPC"_ module.

```hcl
module "network" {
  source = "https://github.com/bschaatsbergen/terraform-gcp-network-module"

  name                            = "example-network"
  mtu                             = 1500
  routing_mode                    = "GLOBAL"
  allow_iap_access                = true
  allow_private_google_access     = true
  deny_all_ingress                = true
  deny_all_egress                 = true
  delete_default_routes_on_create = false
}

module "subnet" {
  source = "https://github.com/bschaatsbergen/terraform-gcp-subnetwork-module"

  network       = module.network.id
  name          = "private-ew4-subnet"
  description   = "Subnet in europe-west4"
  region        = "europe-west4"
  ip_cidr_range = "10.0.0.0/19"

  secondary_ip_ranges = [
    {
      range_name    = "ew4-secondary-range-1"
      ip_cidr_range = "10.149.128.0/17"
    },
  ]

  private_ip_google_access = true
  create_nat               = true
}
```

## Balance repetition and complexity

I find this principle the most important when it comes to writing Terraform modules. It's a principle that I still find myself struggling with, and I'm sure that's a good sign.

Trying to fit multiple things in a single piece of Terraform code will make your code bloated -- it introduces conditionals and loops and it's not always easy to understand what's going on.
On the other hand, repeating yourself too many times is a red flag.

You can easily identify complexity in a Terraform module; complex local variables using conditions, multiple nested dynamic blocks, and so on. It's up to you to find that balance point between complexity and repetition, often such complexity is caused because the module does more than one thing.

> "I'm not a big fan of repetition, but I'm not a big fan of complexity either. I think that's a good balance between both." -- Github Copilot generated this quote.

## Use industry-standard tooling

Terraform is part of a rich infrastructure and DevOps ecosystem, there's dozens of tools available to you that can help you produce effective modules.

There's even a industry-wide standard that I recommend you to use, it's composed of various tools that you should already be familiar with.

- pre-commit hooks _(create and use pre-commit hooks with a simpler interface)_
  - terraform fmt _(rewrite Terraform to a canonical format and style)_
  - terraform validate _(checks that a configuration is syntactically valid)_
  - terraform-docs _(generate documentation)_
  - tflint _(pluggable Terraform Linter)_
- checkov _(static-analysis tool with a set of rules)_

I've put the above tools together, similar to how they are used in popular modules, in a template repository: [Terraform Module Template](https://github.com/bschaatsbergen/terraform-module-template).

The reason that I think this is such a important principle to adopt is that it's a way to make your modules understandable by a wide audience. Modules are meant to be maintained and used, not to be kept in the shadows.

## Don't cover what you don't know

I can't stress it enough, but don't try to be that "complete package".

The other day one of my peers asked me why the module I developed didn't cover IAM permissions. I replied that I didn't know how someone would use the module.
Instead of covering IAM permissions in the module using some obscure complex, I simply left it up to the user.

```hcl
module "bucket" {
  source   = "github.com/bschaatsbergen/terraform-gcp-gcs-module"
  name     = "example-bucket"
  location = "EU"
}

resource "google_storage_bucket_iam_binding" "storage_admin" {
  bucket  = module.bucket.name
  role    = "roles/storage.admin"
  members = [
    "user:bschaatsbergen@binx.com",
  ]
}
```

The key take away from this principle is that you should give your users all the possible output they require to make their own decisions. Don't cover what you don't know and don't pretend you know the intention of the user using your module.