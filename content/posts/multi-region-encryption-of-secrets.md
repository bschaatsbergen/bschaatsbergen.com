---
title: "Global KMS keys and secrets in AWS"
date: 2022-01-23T12:19:55+01:00
tags: ["Terraform", "AWS"]
description: "Lets take a look at managing our KMS keys and secrets across multiple regions with Terraform."
---
In this post I want to give you a brief introduction on how to deploy KMS keys and secrets in Secret Manager across multiple regions. We'll do so by making use of replication to minimize waste and prevent repeating ourselves.

## Multi-region KMS key

July last year AWS introduced [multi-region KMS keys](https://docs.aws.amazon.com/kms/latest/developerguide/multi-region-keys-overview.html). A new capability that lets you replicate keys from one region into another. With multi-region keys, you can more easily move encrypted data between regions without having to decrypt and re-encrypt with different keys in each region.

To create a multi-region KMS key in Terraform we simply set `multi_region` to `true` in the `aws_kms_key` resource.

```hcl
resource "aws_kms_key" "mrk" {
  description         = "Multi-region key for Secrets Manager"
  enable_key_rotation = true
  multi_region        = true
}
```

We'll also create an alias for our multi-region KMS key to reduce the added complexity of working with the KMS key ID itself.

```hcl
data "aws_region" "current" {}

resource "aws_kms_alias" "mrk_alias" {
  name          = "alias/${data.aws_region.main.name}-secrets-manager"
  target_key_id = aws_kms_key.mrk.key_id
}
```

I've interpolated the `name` with the name of the region we pointed our `aws` provider to. I usually do so to achieve a bit of consistency in the naming of my cross-region resources.

Now that we have the multi-regional KMS key and KMS key alias deployed in eu-central-1 it's time to create a multi-region replica KMS key.

## Multi-region replica KMS key

A multi-region replica key is a KMS key that has the same key ID and key material as its primary key and related replica keys, but exists in a different region.

A replica key is a fully functional KMS key with it own key policy, grants, alias, tags, and other properties. It is not a copy of or pointer to the primary key or any other key. You can use a replica key even if its primary key and all related replica keys are disabled. You can also convert a replica key to a primary key and a primary key to a replica key.

To create a multi-region replica key we use the `aws_kms_replica_key` resource. We simply point to our parent KMS key that we created earlier and pass a different provider to the resource.

```hcl
resource "aws_kms_replica_key" "euw1_mrk" {
  description = "Multi-region replica key for Secrets Manager in eu-west-1"
  primary_key_arn = aws_kms_key.mrk.arn

  provider = aws.replica
}
```

> Please note the `provider = aws.replica`, it's a second provider I configured in my `provider.tf` that uses an `alias` to point to a different region.

```hcl
provider "aws" {
  region = "eu-central-1"
}

provider "aws" {
  alias = "replica"
  region = "eu-west-1"
}
```

Again, to avoid any added complexity of working with the KMS key ID itself, we'll also create a KMS key alias for the multi-region replica key.

```hcl
resource "aws_kms_alias" "euw1_mrk_alias" {
  name          = "alias/${data.aws_region.replica.name}-secrets-manager"
  target_key_id = aws_kms_replica_key.euw1_mrk.key_id

  provider = aws.replica
}
```

Run `terraform plan` - `apply` and you will see that the same KMS key ID is the same in
both the regions.

## Create a Secret

Great, now that both the multi-region KMS keys are available in their respective regions it's time to play around with it. I'll create a example secret to test the KMS decryption in both regions using that same KMS key ID.

```hcl
resource "aws_secretsmanager_secret" "example" {
  name = "name-of-secret-string"

  kms_key_id = aws_kms_key.mrk.id

  replica {
    kms_key_id = aws_kms_key.euw1_mrk.id
    region     = data.aws_region.replica.name
  }
}

resource "aws_secretsmanager_secret_version" "example" {
  secret_id     = aws_secretsmanager_secret.example.id
  secret_string = "actual-secret-string"
}
```

Also `terraform plan` - `apply` this. To validate if the secret replication was successful, the secret should have a similar `Name` and `KmsKeyId` in the output.

```bash
bruno ~> aws secretsmanager list-secrets --region eu-central-1 | jq '.SecretList[] | .Name, .KmsKeyId'
"name-of-secret-string"
"mrk-4c5359f5ab511673b6471d1df3bf3182"
```

```bash
bruno ~> aws secretsmanager list-secrets --region eu-west-1 | jq '.SecretList[] | .Name, .KmsKeyId'
"name-of-secret-string"
"mrk-4c5359f5ab511673b6471d1df3bf3182"
```

## Decrypt the secret

I created a managed IAM policy that contains a statement which allows me to decrypt the KMS key by its ID.  You could add this managed policy to the role assumed by the workload ran in your primary and secondary region, that will allow you to retrieve the secret from Secrets manager.


```bash
bruno ~> aws secretsmanager get-secret-value --secret-id arn:aws:secretsmanager:eu-west-1:123456789101:secret:name-of-secret-string-vWenRf  --region eu-west-1 | jq .SecretString
"actual-secret-string"
```

To quickly wrap this up, we've covered how you can create multi-region KMS keys and use the multi-region replica KMS key in a different region without managing multiple completely isolated resources across regions. Please note that secrets in Secrets Manager are just one of the many services that can be managed by KMS, I would advice you to fiddle around with multi-region KMS keys in any cross-region architecture.
