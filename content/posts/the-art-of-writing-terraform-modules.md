---
title: "The lost art of writing Terraform modules"
date: 2022-07-15T11:11:11+11:00
tags: ["terraform"]
---

The ability to write effectively and elegant Terraform modules has long been revered and rewarded.
And yet to me it seems that many have begun to believe that this talent is somehow obsolete; that the art of writing elegant modules is no longer relevant, that mediocre modules are acceptable in a world crying out for efficiency and accessibility.

I believe otherwise, and I would like to share my thoughts on the art of writing Terraform modules by using **four key principals**:

- Do one thing and do it right - God, I love the Unix
- Balance repetition and complexity
- Use widely adopted tooling
- It doesn't belong in the module

Far from your garden-variety style guide, this post covers the art of writing Terraform modules in a more practical sense. Writing great modules is a skill, and this post should push you in the right direction to make your modules shimmer on Github.

## Do one thing and do it right

Unfortunately it seems that the most popular modules that try to be the "complete package" by supporting every single possible scenario. This easily turns your module into a monolithic blob of code, and it's not always easy to understand what's going on and how to maintain it.

Terraform modules should do nothing more but preventing repetition from an end-user perspective.

**A module should do one thing, and do it right**. If you want to do more than one thing, you should create another module. This might result into code duplication but this ties in with the idea of modularity and being able to balance repetition and complexity the right way.

For example, Google Cloud its network and subnetworks are ideally managed isolated from one another. Meaning, you should create a module for a Google Cloud network, and another module for the Google Cloud subnetworks instead of trying to fit them into a singular _"Google Cloud VPC"_ module.

## Balance repetition and complexity

I find this principle the most important when it comes to writing Terraform modules. It's a principle that I still find myself struggling with, and I'm sure that's a good sign.

Trying to fit multiple things in a single piece of Terraform code will make your code bloated -- it introduces conditionals and loops and it's not always easy to understand what's going on.
On the other hand, repeating yourself too many times is a red flag.

...

## Use widely adopted tooling

Terraform is part of a rich infrastructure and DevOps ecosystem, there's dozens of tools available to you that can help you produce effective modules.

There's even a industry-wide standard that I recommend you to use, it's composed of various tools that you should already be familiar with.

- pre-commit hooks _(create and use pre-commit hooks with a simpler interface)_
  - terraform fmt _(rewrite Terraform to a canonical format and style)_
  - terraform validate _(checks that a configuration is syntactically valid)_
  - terraform-docs _(generate documentation)_
  - tflint _(pluggable Terraform Linter)_
- checkov _(static-analysis tool with a set of rules)_

I've put the above tools together, similar to how they are used in popular modules, in a template repository for you called: [Terraform Module Template](https://github.com/bschaatsbergen/terraform-module-template).

## Don't cover what you don't know

I can't stress it enough but.. write your module in a way that you understand how your end-user will use it.
