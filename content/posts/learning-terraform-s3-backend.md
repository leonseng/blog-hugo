---
title: "Learning Terraform S3 Backend"
date: 2021-06-11T14:52:29+10:00
draft: false
tags: ['aws', 's3', 'terraform']
categories: ['tech']
---

I have had basic experience playing with Terraform to instantiate resources in Kubernetes and AWS, but my previous attempts left me with a thought, how do I implement this at work and scale it up to the team?

Terraform creates a local state file which seems like a pain to share around a team. This is when I found out about remote backends. And this is my attempt to learn [Terraform S3 backend](https://www.terraform.io/docs/language/settings/backends/s3.html).

# Setup

Terraform S3 backend stores the state file in an AWS S3 bucket, and if required, uses DynamoDB to manage state locking (highly recommended when working in a team).

So... to manage infrastructure using Terraform, I first need to deploy the supporting infrastructure 😅

🐔 and 🥚 problem much?

There are guides online to perform this setup using Terraform, but I'll keep it simple for now and do it manually through AWS console.

- Create an S3 bucket
- Create DynamoDB table
- Create a role with policy to interact with S3 and DynamoDB (I called it `terraform-backend-role`)
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:GetObject",
                "dynamodb:PutItem",
                "dynamodb:DeleteItem",
                "dynamodb:GetItem",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:dynamodb:ap-southeast-2:<account ID>:table/TerraformLocks",
                "arn:aws:s3:::ls-terraform-bucket/terraform.tfstate",
                "arn:aws:s3:::ls-terraform-bucket"
            ]
        }
    ]
}
```
- Give the IAM user the appropriate policy to assume the role above.

With all that done, we should be good to go.

# Using Terraform

I will be using the [AWS provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs) to create the EC2 instance.

```
# main.tf
terraform {
  backend "s3" {
    bucket = "ls-terraform-bucket"
    key = "terraform.tfstate"
    region = "ap-southeast-2"
    encrypt = true

    dynamodb_table = "TerraformLocks"

    role_arn = "arn:aws:iam::<account_id>:role/terraform-backend-role"
  }
}

provider "aws" {
  region = "ap-southeast-2"
}

resource "aws_instance" "terraform-test" {
  ami = "ami-0186908e2fdeea8f3"
  instance_type = "t3.micro"
  availability_zone = "ap-southeast-2a"
}
```

To run Terraform apply, I need to provide it with my AWS authentication details, which I do by exporting the `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` environment variables.

```
$ export AWS_ACCESS_KEY_ID="anaccesskey"
$ export AWS_SECRET_ACCESS_KEY="asecretkey"
$ terraform plan
$ terraform apply
```

I quickly ran into permission errors, but the error message isn't great.

```
╷
│ Error: Error launching source instance: UnauthorizedOperation: You are not authorized to perform this operation. Encoded authorization failure message: <encoded error message>
│       status code: 403, request id: cc586b96-11b7-4a68-a54c-24bf4bce5419
│
│   with aws_instance.terraform-test,
│   on main.tf line 21, in resource "aws_instance" "terraform-test":
│   21: resource "aws_instance" "terraform-test" {
│
```

Enabling debug logs when running `terraform apply` didn't reveal anything useful for this particular error either

```
$ TF_LOG=DEBUG terraform apply
```

A quick Google shows that I can use AWS CLI to [decode the error message](https://docs.aws.amazon.com/cli/latest/reference/sts/decode-authorization-message.html)

```
$ aws sts decode-authorization-message --encoded-message <encoded error message> --query DecodedMessage --output text | jq .
{
  "allowed": false,
  ...
  "context": {
    "action": "ec2:RunInstances",
    "resource": "arn:aws:ec2:ap-southeast-2:<account ID>:instance/*",
    ...
  }
}
```

which tells me that I don't have permission to create an EC2 instance. Of course! Permission to actually perform what I want to do on AWS! I created the necessary role and policy to interact with EC2 (I called it `test-project`, and gave my IAM user permission to assume the role. Once that's been done, I added the `assume_role` attribute to the AWS provider

```
# main.tf
terraform {
  backend "s3" {
    bucket = "ls-terraform-bucket"
    key = "terraform.tfstate"
    region = "ap-southeast-2"
    encrypt = true

    dynamodb_table = "TerraformLocks"

    role_arn = "arn:aws:iam::<account_id>:role/terraform-backend-role"
  }
}

provider "aws" {
  assume_role {
    role_arn = "arn:aws:iam::<account_id>:role/test-project"
  }
  region = "ap-southeast-2"
}

resource "aws_instance" "terraform-test" {
  ami = "ami-0186908e2fdeea8f3"
  instance_type = "t3.micro"
  availability_zone = "ap-southeast-2a"
}
```

With that, `terraform apply` was able to run successfully and created the EC2 instance

```
$ terraform apply

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_instance.terraform-test will be created
  + resource "aws_instance" "terraform-test" {
      + ami                                  = "ami-0186908e2fdeea8f3"
      + arn                                  = (known after apply)
      + associate_public_ip_address          = (known after apply)
      + availability_zone                    = "ap-southeast-2a"
      + cpu_core_count                       = (known after apply)
      + cpu_threads_per_core                 = (known after apply)
      + get_password_data                    = false
      + host_id                              = (known after apply)
      + id                                   = (known after apply)
      + instance_initiated_shutdown_behavior = (known after apply)
      + instance_state                       = (known after apply)
      + instance_type                        = "t3.micro"
      + ipv6_address_count                   = (known after apply)
      + ipv6_addresses                       = (known after apply)
      + key_name                             = (known after apply)
      + outpost_arn                          = (known after apply)
      + password_data                        = (known after apply)
      + placement_group                      = (known after apply)
      + primary_network_interface_id         = (known after apply)
      + private_dns                          = (known after apply)
      + private_ip                           = (known after apply)
      + public_dns                           = (known after apply)
      + public_ip                            = (known after apply)
      + secondary_private_ips                = (known after apply)
      + security_groups                      = (known after apply)
      + source_dest_check                    = true
      + subnet_id                            = (known after apply)
      + tags                                 = {
          + "Name" = "Terraform EC2"
        }
      + tags_all                             = {
          + "Name" = "Terraform EC2"
        }
      + tenancy                              = (known after apply)
      + vpc_security_group_ids               = (known after apply)

      + capacity_reservation_specification {
          + capacity_reservation_preference = (known after apply)

          + capacity_reservation_target {
              + capacity_reservation_id = (known after apply)
            }
        }

      + ebs_block_device {
          + delete_on_termination = (known after apply)
          + device_name           = (known after apply)
          + encrypted             = (known after apply)
          + iops                  = (known after apply)
          + kms_key_id            = (known after apply)
          + snapshot_id           = (known after apply)
          + tags                  = (known after apply)
          + throughput            = (known after apply)
          + volume_id             = (known after apply)
          + volume_size           = (known after apply)
          + volume_type           = (known after apply)
        }

      + enclave_options {
          + enabled = (known after apply)
        }

      + ephemeral_block_device {
          + device_name  = (known after apply)
          + no_device    = (known after apply)
          + virtual_name = (known after apply)
        }

      + metadata_options {
          + http_endpoint               = (known after apply)
          + http_put_response_hop_limit = (known after apply)
          + http_tokens                 = (known after apply)
        }

      + network_interface {
          + delete_on_termination = (known after apply)
          + device_index          = (known after apply)
          + network_interface_id  = (known after apply)
        }

      + root_block_device {
          + delete_on_termination = (known after apply)
          + device_name           = (known after apply)
          + encrypted             = (known after apply)
          + iops                  = (known after apply)
          + kms_key_id            = (known after apply)
          + tags                  = (known after apply)
          + throughput            = (known after apply)
          + volume_id             = (known after apply)
          + volume_size           = (known after apply)
          + volume_type           = (known after apply)
        }
    }

Plan: 1 to add, 0 to change, 0 to destroy.

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes

aws_instance.terraform-test: Creating...
aws_instance.terraform-test: Still creating... [10s elapsed]
aws_instance.terraform-test: Creation complete after 13s [id=i-05e44bf5cfbb98977]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

# Summary

With that, I am now able to use Terraform with the state tracked remotely. Along with state locking, multiple people in the team can access the same state file safely, as long as their IAM users have the correct permission to assume the role to read/write to S3 and DynamoDB.