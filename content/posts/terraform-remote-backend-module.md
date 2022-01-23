---
title: "AWS remote backend module"
date: 2022-01-20T12:19:55+01:00
tags: ["Terraform", "AWS"]
description:
---

Everytime I have to setup a remote backend in AWS I end up getting slightly more annoyed. I'm either copy pasting code from previous projects (where I manage to improve it after every copy-paste) or I deploy a CloudFormation stack that I forked a while ago from a colleague. It was time to take matters into my owns hands, why not create a Terraform module for this? After all, these remote backends are not rocket science.

## Piecing something together

After fiddling around a bit I managed to put the module together quite fast. Most of my time went to thinking of how I could keep every user happy.

[github.com/binxio/terraform-aws-remote-state-module](https://github.com/binxio/terraform-aws-remote-state-module).


## Features

- S3 cross-region replication
- Forced encryption at rest and in-transit using AES-256
- Denies object deletion
- Point in time recovery

## Usage

Before you start, I recommend to always isolate the state of the remote backend and not mix it with other resources. It's preferred that you create a folder dedicated to the remote backend.

```bash
└── remote-backend
     ├── main.tf
     ├── provider.tf
     └── variables.tf
```

Lets get started, create a `main.tf` file and copy-paste the below code there.

```terraform
provider "aws" {
  region  = "eu-central-1"
}

provider "aws" {
  alias  = "replica"
  region = "eu-west-1"
}

module "remote_backend" {
  source              = "github.com/binxio/terraform-aws-remote-state-module"
  bucket_name         = "my-remote-state-bucket"
  dynamodb_table_name = "my-state-lock-table"
  tags = {
    "Key" = "Value"
  }
  providers = {
    aws         = aws
    aws.replica = aws.replica
  }
}
```

Once that's done, run `terraform plan` followed by a `terraform apply`, this will create the S3 buckets, DynamodDB table and required IAM resources.

Now that the resources are created, you're stuck with a `terraform.tfstate` file locally. Luckily we can migrate that into our S3 bucket and get rid of the local files.

Time to migrate, create a `provider.tf` file and copy-paste the below code there.

```terraform
terraform {
  backend "s3" {
    bucket         = "my-remote-state-bucket"
    dynamodb_table = "my-state-lock-table"
    key            = "your/state/path"
    region         = "eu-central-1"
    encrypt        = true
  }
}
```

Run `terraform init -migrate-state` followed by a `yes`. This will migrate the `terraform.tfstate` to the S3 bucket we just created (your remote backend).

You can now safely delete the local `terraform.tfstate` and `terraform.tfstate.backup`.

Using this module you should be able to deploy a remote backend in less than 30 seconds, and if you would like to have something changed you can [open a PR or issue](https://github.com/binxio/terraform-aws-remote-state-module).
