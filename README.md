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
        command = "echo 'backend: \"${aws_instance.backend.public_ip}\"'>/path/to/project/ansible/backend_role/var/file "
       
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
        command = "echo 'database: \"${aws_instance.database.public_ip}\"'>/path/to/project/backend_role/default/file"
       
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
# Directory should look like this 
```
├── index.html
├── main.tf
├── provider.tf
├── terraform.tfstate
├── terraform.tfstate.backup

```
# With the AWS resources successfully provisioned, we can now proceed to configure the EC2 instances using Ansible.
Within the same directory we are going to create a subdirectory named final where all the asnible files are going to be stored such as:
- dynamic ansible inventory
- ansible role
- playbooks
- ansible vault

Before anything we need to create all of our subdirectories for Final 
- create backend_role/database_role directories within final
- Each role directory will have 7 subdirectories that will execute when the main role playbook is executed
```
├── backend_role
│   ├── defaults
│   │   └── main.yml
│   ├── files
│   ├── handlers
│   │   └── main.yml
│   ├── meta
│   ├── tasks
│   │   └── main.yml
│   ├── templates
│   └── vars
│       ├── main.yml
│       └── vault.yml
├── database_role
│   ├── defaults
│   ├── files
│   ├── handlers
│   │   └── main.yml
│   ├── meta
│   ├── tasks
│   │   └── main.yml
│   ├── templates
│   └── vars
│       └── vault.yml


```
# creating dynamic inventory 

```bash
---
plugin: aws_ec2
regions:
  - Region name

keyed_groups:
  # Create groups based on the value of the 'Type' tag
  - key: tags['Type']
    prefix: tag_Type

# Filter instances to include those tagged as either 'backend' or 'database' or any other tags
filters:
  tag:Type:
    - database
    - backend


```

# Create your main ansible role playbookm that will refrence your ansible vault for secured vairbales and be able to execute both backend and database roles automating the configurations of both ec2 instances
When refrencing a role within ansible make sure its the name of the directory where all 7 subdirectories are preent for backend/database role.

role.yml
```bash
---
- name: database server
  hosts: tag_Type_database
  become: true
  vars_files:
   - /path/to/database/vars/ansible.vault/file
  roles:
   - { role: database_role}

- name: backend server
  hosts: tag_Type_backend
  become: true
  vars_files:
   - /path/to/database/vars/ansible.vault/file
  roles:
   - { role: backend_role}


```

#Configure the database_role 
Within database role we will use handles,tasks,vars subdirectories
-tasks/main.yml:
```bash
---
- name: restarting server
  apt:
    update_cache: yes
  
- name: install MySQL
  apt:
    name: 
      - mysql-server
      - mysql-client
      - python3-mysqldb
      - libmysqlclient-dev
  become: true

- name: start mysql 
  systemd:
    name: mysql
    state: started
    enabled: yes
  become: true

- name: mysql secure installation
  become: true
  command: mysql_secure_installation
  args:
    stdin: |
      SET PASSWORD FOR 'root'@'localhost' = PASSWORD('{{ db_password }}');
      DELETE FROM mysql.user WHERE User='';
      DELETE FROM mysql.user WHERE User='root' AND Host NOT IN ('localhost');
      DROP DATABASE IF EXISTS test;
      DELETE FROM mysql.db WHERE Db='test' OR Db='test\\_%';
      FLUSH PRIVILEGES;

- name: create user
  mysql_user:
    name: "{{db_user}}"
    password: "{{db_password}}"
    priv: '*.*:ALL'
    host: '%'
    state: present
  become: true

- name: create Db
  mysql_db:
    name: "{{ db_name }}"
    state: present
  become: true

- name: adjusting mysql config file
  lineinfile:
    path: /etc/mysql/mysql.conf.d/mysqld.cnf
    regexp: '^bind-address'
    line: 'bind-address = 0.0.0.0'
    backup: yes
  become: true
  notify: restart mysql


```
-handlers/main.yml
```bash
---
- name: restart mysql
  systemd: 
   name: mysql
   state: restarted

```


To create a secured file to use your variables you would need to create ansible vault file inside the directory you want to store it in.
vars/vault.yml:
```bash
# once inside the file is created stored your sensative information of database within this file 
ansible-vault create vault.yml

#If you want to edit the information insdie use this command keep in mind it will ask for the password used for the creation of the file you are trying to access
ansible-vault edit secure.yml

```

#Now configure your backend_role files
we are going to be using the subdirectories handlers,tasks,vars
- tasks/main.yml:
```bash
--- 
- name: restarting backend ec2
  apt:
   update_cache: yes

- name: installing git 
  apt:
   name: 
    - git
    - python3
    - python3-pip
    - gunicorn
   state: latest

- name: installing boto3
  ansible.builtin.pip:
   name: boto3
   state: present

- name: installing neccesery files from github repository
  ansible.builtin.git:
   repo: "https://github.com/chandradeoarya/todolist-flask.git"
   dest: "/home/ubuntu/todolist-flask"
   single_branch: yes
   version: env

- name: get rid of index.html from the repository
  ansible.builtin.file:
   state: absent
   path: "/home/ubuntu/todolist-flask/index.html"

- name: download s3 bucket index.html
  aws_s3:
   bucket: bucket-name
   mode: get
   object: "index.html"
   dest: /home/ubuntu/todolist-flask/index.html

- name: update api url
  lineinfile:
   path: /home/ubuntu/todolist-flask/index.html
   regexp: "const API_URL ="
   line: "const API_URL = 'http://{{backend}}'"

- name: create env file
  copy:
   dest: "/home/ubuntu/todolist-flask/.env"
   content: |
    MYSQL_DATABASE_HOST = "{{db_host}}"
    MYSQL_DATABASE_USER = "{{db_user}}"
    MYSQL_DATABASE_PASSWORD = "{{db_password}}"
    MYSQL_DATABASE_PORT = "{{db_port}}"
   mode: '0644'

- name: re-upload s3 bucket 
  aws_s3:
   bucket: bucket-name
   mode: put
   object: "index.html"
   src: /home/ubuntu/todolist-flask/index.html

- name: delete index.html file once its re-uploaded to s3 bucket
  ansible.builtin.file:
   state: absent
   path: "/home/ubuntu/todolist-flask/index.html"

- name: install requirements
  command: pip install -r /home/ubuntu/todolist-flask/requirements.txt

- name: create todolist gunicorn.py file
  ansible.builtin.file:
   path: /home/ubuntu/todolist-flask/gunicorn_config.py
   state: touch
   mode: u+rwx

- name: add configurations to gunicorn
  ansible.builtin.blockinfile:
   path: /home/ubuntu/todolist-flask/gunicorn_config.py
   block: |
    preload_app = True
    bind = "0.0.0.0:80"
    workers = 4
   mode: u+rwx
    

- name: create todolist service file
  ansible.builtin.file:
   path: /etc/systemd/system/todolist.service
   state: touch
   mode: u+rwx

- name: Add content to todolist service file
  ansible.builtin.blockinfile:
   path: /etc/systemd/system/todolist.service
   block: |
    # BEGIN ANSIBLE MANAGED BLOCK
    [Unit]
    Description=Gunicorn instance to serve todolist
    Wants=network.target
    After=syslog.target network-online.target
    [Service]
    Type=simple
    WorkingDirectory=/home/ubuntu/todolist-flask
    ExecStart=/usr/bin/gunicorn todo:app -c /home/ubuntu/todolist-flask/gunicorn_config.py
    Restart=always
    RestartSec=10
    [Install]
    WantedBy=multi-user.target
    # END ANSIBLE MANAGED BLOCK

- name: start
  systemd: 
   name: todolist.service
   state: started
   enabled: yes

- name: restart daemon-reload
  systemd: 
   daemon-reload: yes
  notify: restarting todolist


```

-handlers/main.yml:

```bash
---
- name: restarting todolist
  systemd:
   name: todolist.service
   state: restarted

```

-vars/vault.yml
You can either create a new ansible vault file and give it anoter name and password but for this deployment we actually copied the vault.yml from database_role/vars/vault.yml since the information we need is the same from that file.
```bash
# once inside the file is created stored your sensative information of database within this file 
ansible-vault create vault.yml

#If you want to edit the information insdie use this command keep in mind it will ask for the password used for the creation of the file you are trying to access
ansible-vault edit secure.yml
```
-defaults/main.yml:
```bash
# earlier within the instructions when we created database ec2 instance we ade sure to foward the key:value once it was executed to this location within the terminal so it will auto populate the newest ec2 databse ip
db_host: {{ databse_ec2_IP}}
```


# Now both Terraform and ansible are finaly created and configure we can now activate the playbooks and iy will automatically xocnfigure the ec2 for you
```bash
ansible-playbook -ask-vault-pass role.yml -i aws_ec2.yml -u ubuntu -k --private-key:/path/to/pem/file
#or you can execute it in the main directory where terraform is located
ansible-playbook -ask-vault-pass final/role.yml -i final/aws_ec2.yml -u ubuntu -k --private-key:/path/to/pem/file


```







