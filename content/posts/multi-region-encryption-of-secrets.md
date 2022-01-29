---
title: "Global KMS keys and secrets in AWS"
date: 2022-01-23T12:19:55+01:00
tags: ["Terraform", "AWS"]
draft: true
description: "Lets take a look at managing our KMS keys and secrets across multiple regions with Terraform."
---

I absolutely love encryption and secret management in AWS. Don't get me wrong, it does deserve some love here and there but it's coming together quite well.

In this post I want to give you a brief introduction on how to manage KMS keys and secrets in Secret Manager globally.

## Multi-region KMS key

July last year AWS introduced multi-region keys. A new capability that lets you replicate keys from one region into another. With multi-region keys, you can more easily move encrypted data between regions without having to decrypt and re-encrypt with different keys in each region.

To create a multi-region KMS key in Terraform we simply set `multi_region` to `true` in our `aws_kms_key` resource.

```terraform
resource "aws_kms_key" "main_sm" {
  description         = "Multi-region key for Secrets Manager"
  enable_key_rotation = true
  multi_region        = true
}
```

We'll also create an alias for our KMS key to reduce the added complexity of working with the KMS key ID.

```terraform
data "aws_region" "main" {}

resource "aws_kms_alias" "main_sm" {
  name          = "alias/${data.aws_region.main.name}-secrets-manager"
  target_key_id = aws_kms_key.main_sm.key_id
}
```

I've interpolated the `name` with the name of the region we pointed our `aws` provider to. I usually do so to achieve a bit of consistency in the name of my cross-region resources.

Now that we have the multi-regional KMS key and KMS key alias deployed in eu-central-1 it's time to create a replica key.

## Multi-region replica KMS key

A multi-region replica key is a KMS key that has the same key ID and key material as its primary key and related replica keys, but exists in a different region.

A replica key is a fully functional KMS key with it own key policy, grants, alias, tags, and other properties. It is not a copy of or pointer to the primary key or any other key. You can use a replica key even if its primary key and all related replica keys are disabled. You can also convert a replica key to a primary key and a primary key to a replica key.

To create a multi-region replica key we use the `aws_kms_replica_key` resource. We simply point to our parent KMS key that we created earlier and pass a different provider to the resource.

```terraform
resource "aws_kms_replica_key" "euw1_sm" {
  description = "Multi-region replica key for Secrets Manager in eu-west-1"
  primary_key_arn = aws_kms_key.multi_region_sm.arn

  provider = aws.replica
}
```

> Please note the `provider = aws.replica`, it's a second provider I configured in my `provider.tf` that uses an `alias` to point to a different region.

```terraform
provider "aws" {
  region = "eu-central-1"
}

provider "aws" {
  alias = "replica"
  region = "eu-west-1"
}
```

Again, to avoid added complexity of working with the KMS key ID we'll also create a KMS key alias for the multi-region replica key.

```terraform
resource "aws_kms_alias" "euw1_sm" {
  name          = "alias/${data.aws_region.replica.name}-secrets-manager"
  target_key_id = aws_kms_replica_key.euw1_sm.key_id

  provider = aws.replica
}
```

Now plan and apply, you'll see that the same KMS key ID is in both regions but with a different alias: `eu-central-1-secrets-manager` and `eu-west-1-secrets-manager`.

## Create a Secret

Now that both the multi-region KMS keys are deployed we can start using them/ Lets create a example secret to test the cross-region KMS decryption.

```terraform
resource "aws_secretsmanager_secret" "example" {
  name = "name-of-secret-string"

  kms_key_id = aws_kms_key.main_sm.id

  replica {
    kms_key_id = aws_kms_key.euw1_sm.id
    region     = data.aws_region.replica.name
  }
}

resource "aws_secretsmanager_secret_version" "example" {
  secret_id     = aws_secretsmanager_secret.example.id
  secret_string = "actual-secret-string"
}
```

Also plan and apply this. Lets validate if the replication was successful, we should see the same secret `Name` and `KmsKeyId` in the output.

```sh
bruno ~> aws secretsmanager list-secrets --region eu-central-1 | jq '.SecretList[] | .Name, .KmsKeyId'
"name-of-secret-string"
"mrk-4c5359f5ab511673b6471d1df3bf3182"
```

```sh
bruno ~> aws secretsmanager list-secrets --region eu-west-1 | jq '.SecretList[] | .Name, .KmsKeyId'
"name-of-secret-string"
"mrk-4c5359f5ab511673b6471d1df3bf3182"
```

Great the secret is replicated and uses the same KMS key ID.
> As mentioned earlier, the multi-region replica KMS key ID is similar to its parent KMS key ID.


## Decrypt the secret

Time to decrypt our replicated secret with our multi-region replica key.

> Note, I granted my principal access to call `Decrypt` through the KMS key policy.

```bash
bruno ~> aws secretsmanager get-secret-value --secret-id arn:aws:secretsmanager:eu-west-1:123456789101:secret:name-of-secret-string-vWenRf  --region eu-west-1 | jq .SecretString
"actual-secret-string"
```

Nice that works. We've now covered how you can create multi-region KMS keys and use the key across different regions. Secrets in Secrets Manager are just one of the many supported services out there, you can do the same with EBS, S3, CloudWatch logs, etc.
