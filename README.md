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
ansible-playbook --ask-vault-pass role.yml -i aws_ec2.yml -u ubuntu --private-key=/home/epquito/aws_keys/cpdevopsew-us-west-1.pem

