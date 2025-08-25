---
published: true
title: "Solve Dynamic backend problem with Golang"
cover_image: "https://media2.dev.to/dynamic/image/width=1000,height=420,fit=cover,gravity=auto,format=auto/https%3A%2F%2Fdev-to-uploads.s3.amazonaws.com%2Fuploads%2Farticles%2Ffbaes3u2eqb91de2z8cw.jpeg"
description: "Terraform State file dynamic generation"
tags:
  - concepts
  - terraform
  - automation
  - golang
series: null
canonical_url: null
date: "2022-07-10"
---

As mentioned in a [previous blog](https://awstip.com/terraform-some-gotcha-and-essentials-5e996c11d92f) post, Terraform can store its state file (`.tfstate`) in a remote location. However, the `backend` block requires initialization with `terraform init` and does not support `interpolation`. This means you have to manually populate the backend configuration before running `terraform init` or use the `-backend-config` switch.

```hcl
terraform {
   backend "s3" {
      bucket = "mybucket"
      key    = "path/to/my/key"
      region = "us-east-1"
   }
}
```

This manual process of changing the `key` for every new resource or combination of resources is manageable for small deployments but can become a major headache for larger ones.

To solve this, we can dynamically generate the `backend` block before running `terraform init` (explained here [my previous blog](https://awstip.com/terraform-some-gotcha-and-essentials-5e996c11d92f)) .

![](https://miro.medium.com/v2/resize:fit:720/format:webp/1*Fi_u9dDgaGzZwGn7QwhW8A.png)

---

### **Goal**

The goal is to create a tool that automatically generates the backend block and establishes a **1:1 relationship** between the `tfvars` file and the `tfstate` file. The tool should adhere to the following rules:

- It should accept environment variables for `S3`, `DynamoDB`, and `Region`.
- It should take a `tfvars` file as an argument.
- It should generate the backend configuration using either the environment variables or the absolute path of the `tfvars` file.
- It should be able to run `terraform init`, `terraform plan`, `terraform apply`, and `terraform destroy`.
- It should run `terraform init`, `terraform plan`, and `terraform apply` if the filename ends with `*.tfvars`.
- It should run `terraform plan -destroy` and `terraform destroy` if the filename ends with `*.destroy`.

---

### **Explanation**

Let's call our tool `autotf`. It will have two _sub-commands_: `verify` and `deploy`.

#### **Flow Control and Decision Making**

Actions are performed based on the file extension.

![](https://miro.medium.com/v2/format:webp/1*3HKB5znMYp_thwDVAQ2kBQ.jpeg)

### Building Resources

#### **1. Verify Command**

When you run the following command, `autotf` will automatically generate the backend configuration, initialize Terraform, and then run `terraform plan` or `terraform plan -destroy` based on the file extension.

```bash
export S3Bucket="some-s3-bucket"
export Region="us-east-1"
export DynamoDB="some-dynamodb-table"
./autotf verify stage/s3/autotf-testing01.tfvars
```

`autotf` should automatically generate the backend configuration and initialize `terraform` using it by running `terraform init` and then run terraform plan or `terraform plan -destroy` depending on the file name extension `_.tfvars` or `_.destroy` sequentially.

#### Output

```bash
INFO 2022-03-28 17:50:29 will run terraform init, plan on [autotf-testing01.tfvars]

+------------------+----------------+-------------------------+-----------------------------------+
| Resource Name    | Backend Bucket | TFVars Name             | Backend Key                       |
+------------------+----------------+-------------------------+-----------------------------------+
| autotf-testing01 | autotf-testing | autotf-testing01.tfvars | stage/s3/autotf-testing01.tfstate |
+------------------+----------------+-------------------------+-----------------------------------+

Initializing the backend...

Successfully configured the backend "s3"! Terraform will automatically
use this backend unless the backend configuration changes.

Initializing provider plugins...
- Finding latest version of hashicorp/aws...
- Installing hashicorp/aws v4.8.0...
- Installed hashicorp/aws v4.8.0 (signed by HashiCorp)

Terraform has created a lock file .terraform.lock.hcl to record the provider
selections it made above. Include this file in your version control repository
so that Terraform can guarantee to make the same selections by default when
you run "terraform init" in the future.

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
Acquiring state lock. This may take a few moments...

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_s3_bucket.bucket will be created
  + resource "aws_s3_bucket" "bucket" {
      + acceleration_status                  = (known after apply)
      + acl                                  = (known after apply)
      + arn                                  = (known after apply)
      + bucket                               = "autotf-testing01"
      + bucket_domain_name                   = (known after apply)
      + bucket_regional_domain_name          = (known after apply)
      + cors_rule                            = (known after apply)
      + force_destroy                        = false
      + grant                                = (known after apply)
      + hosted_zone_id                       = (known after apply)
      + id                                   = (known after apply)
      + lifecycle_rule                       = (known after apply)
      + logging                              = (known after apply)
      + object_lock_enabled                  = (known after apply)
      + policy                               = (known after apply)
      + region                               = (known after apply)
      + replication_configuration            = (known after apply)
      + request_payer                        = (known after apply)
      + server_side_encryption_configuration = (known after apply)
      + tags                                 = {
          + "Environment" = "Dev"
          + "Name"        = "autotf-testing01"
        }
      + tags_all                             = {
          + "Environment" = "Dev"
          + "Name"        = "autotf-testing01"
        }
      + versioning                           = (known after apply)
      + website                              = (known after apply)
      + website_domain                       = (known after apply)
      + website_endpoint                     = (known after apply)

      + object_lock_configuration {
          + object_lock_enabled = (known after apply)
          + rule                = (known after apply)
        }
    }

Plan: 1 to add, 0 to change, 0 to destroy.

Changes to Outputs:
  + bucket_name = (known after apply)

────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

Note: You didn't use the -out option to save this plan, so Terraform can't guarantee to take exactly these actions if you run "terraform apply" now.
Releasing state lock. This may take a few moments...
```

#### **2. Deploy Command**

Running the `deploy` command with a `*.tfvars` file will initiate the resource creation process.

```bash
export S3Bucket="some-s3-bucket"
export Region="us-east-1"
export DynamoDB="some-dynamodb-table"
./autotf deploy stage/s3/autotf-testing01.tfvars
```

---

### **Destroying Resources**

Resource destruction follows the same flow control. If the file extension is `*.destroy`, the `verify` command will run `terraform init` and `terraform plan -destroy`, and the `deploy` command will run `terraform init` and `terraform destroy`.

```bash
# Change the file name from *.tfvars to *.destroy
mv stage/s3/autotf-testing01.tfvars stage/s3/autotf-testing01.destroy

# Run the `autotf` deploy command again with the new filename
./autotf deploy stage/s3/autotf-testing01.destroy
```

#### Output

```bash
INFO 2022-03-28 18:13:07 will run terraform init, plan -destory on [autotf-testing01.destroy]

Initializing the backend...

Initializing provider plugins...
- Reusing previous version of hashicorp/aws from the dependency lock file
- Using previously-installed hashicorp/aws v4.8.0

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
Acquiring state lock. This may take a few moments...
aws_s3_bucket.bucket: Refreshing state... [id=autotf-testing01]

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
  - destroy

Terraform will perform the following actions:

  # aws_s3_bucket.bucket will be destroyed
  - resource "aws_s3_bucket" "bucket" {
      - acl                                  = "private" -> null
      - arn                                  = "arn:aws:s3:::autotf-testing01" -> null
      - bucket                               = "autotf-testing01" -> null
      - bucket_domain_name                   = "autotf-testing01.s3.amazonaws.com" -> null
      - bucket_regional_domain_name          = "autotf-testing01.s3.amazonaws.com" -> null
      - cors_rule                            = [] -> null
      - force_destroy                        = false -> null
      - grant                                = [] -> null
      - hosted_zone_id                       = "Z3AQBSTGFYJSTF" -> null
      - id                                   = "autotf-testing01" -> null
      - lifecycle_rule                       = [] -> null
      - logging                              = [] -> null
      - object_lock_enabled                  = false -> null
      - region                               = "us-east-1" -> null
      - replication_configuration            = [] -> null
      - request_payer                        = "BucketOwner" -> null
      - server_side_encryption_configuration = [] -> null
      - tags                                 = {
          - "Environment" = "Dev"
          - "Name"        = "autotf-testing01"
        } -> null
      - tags_all                             = {
          - "Environment" = "Dev"
          - "Name"        = "autotf-testing01"
        } -> null
      - versioning                           = [
          - {
              - enabled    = false
              - mfa_delete = false
            },
        ] -> null
      - website                              = [] -> null
    }

Plan: 0 to add, 0 to change, 1 to destroy.

Changes to Outputs:
  - bucket_name = "autotf-testing01" -> null

────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

Note: You didn't use the -out option to save this plan, so Terraform can't guarantee to take exactly these actions if you run "terraform apply" now.
Releasing state lock. This may take a few moments...
Acquiring state lock. This may take a few moments...
aws_s3_bucket.bucket: Refreshing state... [id=autotf-testing01]

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
  - destroy

Terraform will perform the following actions:

  # aws_s3_bucket.bucket will be destroyed
  - resource "aws_s3_bucket" "bucket" {
      - acl                                  = "private" -> null
      - arn                                  = "arn:aws:s3:::autotf-testing01" -> null
      - bucket                               = "autotf-testing01" -> null
      - bucket_domain_name                   = "autotf-testing01.s3.amazonaws.com" -> null
      - bucket_regional_domain_name          = "autotf-testing01.s3.amazonaws.com" -> null
      - cors_rule                            = [] -> null
      - force_destroy                        = false -> null
      - grant                                = [] -> null
      - hosted_zone_id                       = "Z3AQBSTGFYJSTF" -> null
      - id                                   = "autotf-testing01" -> null
      - lifecycle_rule                       = [] -> null
      - logging                              = [] -> null
      - object_lock_enabled                  = false -> null
      - region                               = "us-east-1" -> null
      - replication_configuration            = [] -> null
      - request_payer                        = "BucketOwner" -> null
      - server_side_encryption_configuration = [] -> null
      - tags                                 = {
          - "Environment" = "Dev"
          - "Name"        = "autotf-testing01"
        } -> null
      - tags_all                             = {
          - "Environment" = "Dev"
          - "Name"        = "autotf-testing01"
        } -> null
      - versioning                           = [
          - {
              - enabled    = false
              - mfa_delete = false
            },
        ] -> null
      - website                              = [] -> null
    }

Plan: 0 to add, 0 to change, 1 to destroy.

Changes to Outputs:
  - bucket_name = "autotf-testing01" -> null
aws_s3_bucket.bucket: Destroying... [id=autotf-testing01]
aws_s3_bucket.bucket: Destruction complete after 0s
Releasing state lock. This may take a few moments...

Destroy complete! Resources: 1 destroyed.
```

---

### **The `autotf` Tool**

The `autotf` tool is available on GitHub. It's a demo to showcase how to solve this common Terraform problem. The control flow is straightforward: for each `verify` and `deploy` command, the backend configuration is dynamically generated from the `tfvars` file, ensuring it's always consistent.

Here is an example of the Terraform code for creating an S3 bucket.

#### **Terraform Code**

```hcl
terraform {
  backend "s3" {}
}

provider "aws" {
  region = var.region
}

resource "aws_s3_bucket" "bucket" {
  bucket = var.name

  tags = {
    Name        = var.name
    Environment = "Dev"
  }
}

variable "region" {
  type = string
}
variable "name" {
  type = string
}

output "bucket_name" {
  value = aws_s3_bucket.bucket.id
}
```

#### **Command Run**

```bash
./autotf verify stage/s3/autotf-testing01.tfvars
```

Using an external tool provides the ability to manage Terraform without manually editing the backend configuration. Generating the backend configuration based on a predefined logic is a much more efficient approach.

![](https://miro.medium.com/v2/format:webp/1*3HKB5znMYp_thwDVAQ2kBQ.jpeg)

Above flow-control explains how the sequence of events will occur for `verify` and `deploy` sub-commands respectively. Because the backend configuration is getting generated dynamically with each `tfvars` file so we don’t have to persist it and it will always be the same as long as we used the same `tfvars` file.

Here is example for verify step for creating a **S3 Bucket**.

#### Terraform Code

```hcl
terraform {
  backend "s3" {}
}

provider "aws" {
  region = var.region
}

resource "aws_s3_bucket" "bucket" {
  bucket = var.name

  tags = {
    Name        = var.name
    Environment = "Dev"
  }
}

variable "region" {
  type = string
}
variable "name" {
  type = string
}

output "bucket_name" {
  value = aws_s3_bucket.bucket.id
}
```

#### Command I Ran

```bash
>>> ./autotf verify stage/s3/autotf-testing01.tfvars
```

#### Output

```bash
INFO 2022-03-28 17:50:29 will run terraform init, plan on [autotf-testing01.tfvars]

+------------------+----------------+-------------------------+-----------------------------------+
| Resource Name    | Backend Bucket | TFVars Name             | Backend Key                       |
+------------------+----------------+-------------------------+-----------------------------------+
| autotf-testing01 | autotf-testing | autotf-testing01.tfvars | stage/s3/autotf-testing01.tfstate |
+------------------+----------------+-------------------------+-----------------------------------+

Initializing the backend...

Successfully configured the backend "s3"! Terraform will automatically
use this backend unless the backend configuration changes.

Initializing provider plugins...
- Finding latest version of hashicorp/aws...
- Installing hashicorp/aws v4.8.0...
- Installed hashicorp/aws v4.8.0 (signed by HashiCorp)

Terraform has created a lock file .terraform.lock.hcl to record the provider
selections it made above. Include this file in your version control repository
so that Terraform can guarantee to make the same selections by default when
you run "terraform init" in the future.

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
Acquiring state lock. This may take a few moments...

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_s3_bucket.bucket will be created
  + resource "aws_s3_bucket" "bucket" {
      + acceleration_status                  = (known after apply)
      + acl                                  = (known after apply)
      + arn                                  = (known after apply)
      + bucket                               = "autotf-testing01"
      + bucket_domain_name                   = (known after apply)
      + bucket_regional_domain_name          = (known after apply)
      + cors_rule                            = (known after apply)
      + force_destroy                        = false
      + grant                                = (known after apply)
      + hosted_zone_id                       = (known after apply)
      + id                                   = (known after apply)
      + lifecycle_rule                       = (known after apply)
      + logging                              = (known after apply)
      + object_lock_enabled                  = (known after apply)
      + policy                               = (known after apply)
      + region                               = (known after apply)
      + replication_configuration            = (known after apply)
      + request_payer                        = (known after apply)
      + server_side_encryption_configuration = (known after apply)
      + tags                                 = {
          + "Environment" = "Dev"
          + "Name"        = "autotf-testing01"
        }
      + tags_all                             = {
          + "Environment" = "Dev"
          + "Name"        = "autotf-testing01"
        }
      + versioning                           = (known after apply)
      + website                              = (known after apply)
      + website_domain                       = (known after apply)
      + website_endpoint                     = (known after apply)

      + object_lock_configuration {
          + object_lock_enabled = (known after apply)
          + rule                = (known after apply)
        }
    }

Plan: 1 to add, 0 to change, 0 to destroy.

Changes to Outputs:
  + bucket_name = (known after apply)

────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

Note: You didn't use the -out option to save this plan, so Terraform can't guarantee to take exactly these actions if you run "terraform apply" now.
Releasing state lock. This may take a few moments...
```

As I mentioned before in [my previous blog](https://awstip.com/terraform-some-gotcha-and-essentials-5e996c11d92f) with the help of some external tool to achieve the dynamic backend configuration generation and we did it. We can even run this tool in `docker` container (How ? [Explained here](https://towardsdev.com/run-golang-app-inside-a-docker-container-8cb6e64ae722)) in our CI System.

---

### **Personal Best Practices**

Here are some rules I personally follow for better Terraform management:

- **Ensure consistency:** The resource name, `tfvars` filename, and `tfstate` filename should be the same to maintain a **1:1 relationship**.
- **Derive names automatically:** For multiple resources forming a single entity, names are automatically derived from the primary resource's name.
- **Keep `tfvars` short:** Use defaults to minimize the size of `tfvars` files.
- **Use ternary operators:** This helps manage both computed and provided values effectively.
- **Make modules self-sufficient:** Design modules to be as nimble and independent as possible.
- **Write validators:** Implement validators to support a "fail-fast" approach.
- **Avoid `depends_on`:** Try to use it as little as possible.

Happy Coding and Terraforming\! 💻
