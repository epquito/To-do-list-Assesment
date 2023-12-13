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







