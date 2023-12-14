# Deploying 3-Tier Website with AWS/Ansible/Terraform

This repository provides scripts and configurations to deploy a 3-tier web application using AWS, Ansible, and Terraform.

## Prerequisites

Before deploying the Flask application "TodoList," ensure you have the following and have a brief understading of aws resources:
- AWS account
- VSCode
- Terraform
- Ansible

## Commands to Execute in Terminal

Execute the following commands in the terminal:

```bash
terraform init
terraform plan
terraform apply
ansible-vault create vault.yml
ansible-playbook --ask-vault-pass role.yml -i aws_ec2.yml -u ubuntu --private-key=/path/to/pem/file
```

# Building AWS infrastructure with Terraform
Create two files main.tf/providers.tf within the root directory of the repository
- provider.tf:
```bash
# Enter region associated with your AWS account
provider "aws" {
    region = "Enter Region"
  
}

terraform {
  required_providers {
    aws = {
        source = "hashicorp/aws"
        version = "~> 4.16"
    }
  }
}
# Allows terraform to retrieve data from AWS to get latest ubuntu ami within the specified filters
data "aws_ami" "ubuntu_22" {
    most_recent = true

    filter {
      name = "name"
      values = ["ubuntu/images/hvm-ssd/ubuntu*22*amd64*"]
    }
    filter {
      name = "virtualization-type"
      values = ["hvm"]
    }
    owners = ["amazon"]

}


```
- main.tf:
```bash
# Refrences the defualt vpc that is given to you by aws
resource "aws_default_vpc" "default" {
    tags = {
      Name = "default"
    }
  
}
# Refrences the default Security group that is given to you by aws but also changing the ingress/egress for this deployment
resource "aws_default_security_group" "default" {
  vpc_id = aws_default_vpc.default.id
  ingress {
    description = "http"
    from_port = 80
    to_port = 80
    protocol = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "ssh"
    from_port = 22
    to_port = 22
    protocol = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  ingress {
    description = "myqsl"
    from_port = 3306
    to_port = 3306
    protocol = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port = 0
    to_port = 0
    protocol = "-1"
    cidr_blocks = [ "0.0.0.0/0" ]

  }
  tags = {
    Name = "default"
  }
}


```
# Within the same main.tf file you also need to create a Iam role/policy and attach it to your EC2 instance to give it full access to S3 buckets

 ```bash
# Define the IAM Role
resource "aws_iam_role" "ROLENAME" {
  name = "ROLENAME"

  assume_role_policy = jsonencode({
    Version = "2012-10-17",
    Statement = [
      {
        Action = "sts:AssumeRole",
        Effect = "Allow",
        Principal = {
          Service = "ec2.amazonaws.com",
        },
      },
    ],
  })
}

# Define the IAM Role Policy
resource "aws_iam_policy" "POLICYNAME" {
  name        = "POLICYNAME"
  description = "Policy for EC2 to access S3"

  policy = jsonencode({
    Version = "2012-10-17",
    Statement = [
      {
        Action   = "s3:*",  # Adjust the permissions as needed
        Effect   = "Allow",
        Resource = "*",
      },
    ],
  })
}

# Attach the Role Policy to the Role
resource "aws_iam_policy_attachment" "role_policy_attachment" {
  name       = "role-policy-attachment"
  roles      = [aws_iam_role.ROLENAME.name]
  policy_arn = aws_iam_policy.ec2-s3-fullAccess.arn
}

# Define the IAM Instance Profile
resource "aws_iam_instance_profile" "ec2_profile" {
  name = "ec2_profile"
  role = aws_iam_role.ROLENAME.name
}


```







