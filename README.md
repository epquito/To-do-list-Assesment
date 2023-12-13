Deploying 3-Tier Website with AWS/Ansible/Terraform
This repository provides scripts and configurations to deploy a 3-tier web application using AWS, Ansible, and Terraform.

Prerequisites
Before deploying the Flask application "TodoList," ensure you have the following:

AWS account
VSCode
Operating system
Terraform
Ansible
Commands to Execute in Terminal
Execute the following commands in the terminal:

bash
Copy code
terraform init
terraform plan
terraform apply
ansible-vault create vault.yml
ansible-playbook --ask-vault-pass role.yml -i aws_ec2.yml -u ubuntu --private-key=/path/to/aws_keys/cpdevopsew-us-west-1.pem
Building AWS Infrastructure with Terraform in VSCode
1. Create provider.tf and main.tf
provider.tf:

hcl
Copy code
provider "aws" {
    region = "us-west-1"
}
main.tf:

hcl
Copy code
terraform {
    required_providers {
        aws = {
            source  = "hashicorp/aws"
            version = "~> 4.16"
        }
    }
}

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
2. AWS Infrastructure in main.tf
Define AWS resources such as VPC, security groups, IAM roles, EC2 instances, S3 bucket, etc.

Example Block:

hcl
Copy code
resource "aws_default_vpc" "default" {
    tags = {
      Name = "default"
    }
}
3. IAM Role for EC2 Access to S3
Define IAM roles, policies, and instance profiles for EC2 instances to access S3 buckets.

Example Block:

hcl
Copy code
resource "aws_iam_role" "edwinRole" {
  name = "edwinRole"

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
4. Creating EC2 Instances
Create EC2 instances for the backend and database, referencing the Ubuntu server AMI, and configure necessary settings.

Example Block:

hcl
Copy code
resource "aws_instance" "TodoList" {
    ami           = data.aws_ami.ubuntu_22.id
    instance_type = "t2.micro"
    key_name      = "cpdevopsew-us-west-1"
    # Other configurations...
}
5. Creating S3 Resource for Frontend
Create an S3 bucket to store the frontend index.html file and configure bucket policies.

Example Block:

hcl
Copy code
resource "aws_s3_bucket" "edwin-bucket" {
    bucket = "edwin-bucket123"
    # Other configurations...
}
Ansible Playbooks
Create a dynamic inventory file aws_ec2.yml in the final directory.
yaml
Copy code
plugin: aws_ec2
regions:
  - us-west-1
keyed_groups:
  - key: tags['Type']
    prefix: tag_Type
filters:
  tag:Type:
    - database
    - backend
Create a playbook role.yml in the final directory to call backend and database roles.

Create roles for backend and database with necessary directories and files.

Running the Playbooks
Execute the Ansible playbook with the following command:

bash
Copy code
ansible-playbook --ask-vault-pass role.yml -i aws_ec2.yml -u ubuntu --private-key=/path/to/aws_keys/cpdevopsew-us-west-1.pem
This command deploys both roles in the specified order.

Backend Role Playbook
Execute various tasks to configure the backend server, install dependencies, retrieve the Git repository, update configurations, and start the application.

Example Block:

yaml
Copy code
- name: restarting backend ec2
  apt:
   update_cache: yes
# Other tasks...
Database Role Playbook
Configure the database server, install MySQL, secure the installation, create a user, database, and adjust MySQL configuration for remote access.

Example Block:

yaml
Copy code
- name: install MySQL
  apt:
    name: 
      - mysql-server
      - mysql-client
      - python3-mysqldb
      - libmysqlclient-dev
  become: true
# Other tasks...
