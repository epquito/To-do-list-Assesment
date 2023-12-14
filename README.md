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
# Now initialize providers.tf
```bash
terraform init
```

# Create main.tf and configure your resources 
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

# Create Your backend and database ec2 instances and attach the iam role
```bash
resource "aws_instance" "backend" {
     ami           = data.aws_ami.ubuntu_22.id
     instance_type = "t2.micro"
     key_name      = "PEMFILE"
     iam_instance_profile = aws_iam_instance_profile.ec2_profile.name  # Use the instance profile
     provisioner "local-exec" {
        command = "echo 'backend: \"${aws_instance.backend.public_ip}\"'>/path/to/project/ansible/role/var/file "
       
     }
     tags = {
        Name = "backend"
        Enviroment = "dev"
        Team = "mobile-app"
        Type = "backend"
        }
}
resource "aws_instance" "database" {
     ami           = data.aws_ami.ubuntu_22.id
     instance_type = "t2.micro"
     key_name      = "PEMFILE"
     provisioner "local-exec" {
        command = "echo 'database: \"${aws_instance.database.public_ip}\"'>/path/to/project/ansible/role/var/file"
       
     }

     tags = {
        Name = "database"
        Enviroment = "dev"
        Team = "mobile-app"
        Type = "database"
        }
}

output "TodoList" {
    value = aws_instance.backend.public_ip
  
}

output "TodoList-db" {
    value = aws_instance.database.public_ip
  
}


```
# Create S3 bucket and object also configure bucket policy.
```bash
resource "aws_s3_bucket" "bucket-name" {
  bucket = "bucket-name"
  website {
    index_document = "index.html"
  }
  tags = {
    Type = "frontend"

  }
}
resource "aws_s3_bucket_object" "index" {
  depends_on = [ aws_s3_bucket.edwin-bucket ]
  bucket       = aws_s3_bucket.edwin-bucket.bucket
  key          = "index.html"
  content_type = "text/html"
  #For this deployment I saved the index.html from the web app to my local machine
  source = "/local/path/to/index.html"
}

resource "aws_s3_bucket_public_access_block" "bucket-name" {
  bucket = aws_s3_bucket.bucket-name.id
  block_public_acls       = false
  block_public_policy     = false
  ignore_public_acls      = false
  restrict_public_buckets = false
}
# Define the Bucket Policy
resource "aws_s3_bucket_policy" "bucket-policy" {
  depends_on = [ aws_instance.backend, aws_instance.database]
  bucket = aws_s3_bucket.bucket-name.id

  policy = jsonencode({
    Version = "2012-10-17",
    Statement = [
      {
        Effect    = "Allow",
        Principal = "*",
        Action    = "s3:GetObject",
        Resource  = "${aws_s3_bucket.bucket-name.arn}/*",
      },
    ],
  })
}

# Define Bucket Ownership Controls
resource "aws_s3_bucket_ownership_controls" "bucket-name" {
  bucket = aws_s3_bucket.bucket-name.id

  rule {
    object_ownership = "BucketOwnerPreferred"
  }
}

# Output the Bucket Website Endpoint
output "bucket_website_endpoint" {
  value = aws_s3_bucket.bucket-name.website_endpoint

}


```
After completing the creation of your AWS architecture structure for your resources, you can now initiate the deployment, and the resources specified in the main.tf file will be provisioned within your AWS account.
```bash
terraform plan
terraform apply
```
# With the AWS resources successfully provisioned, we can now proceed to configure the EC2 instances using Ansible.








