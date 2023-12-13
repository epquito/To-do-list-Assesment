
Deploying 3-Tier website with
AWS/Ansible/Terraform.

•	For The deployment of the flask application “TodoList” we are going to need the following:
o	Aws account
o	Vscode
o	Operating system
o	Terraform 
o	Ansible
•	Commands use in terminal to execute terraform/ansible
o	Terraform init
o	Terraform plan
o	Terraform apply
o	Ansible-vault create vault.yml
o	Ansible-playbook –ask-vault-pass role.yml -i aws_ec2.yml -u ubuntu –private-key=/home/epquito/aws_keys/cpdevopsew-us-west-1.pem
1.	Building AWS infrastructure with Terraform with Vscode
a.	Create your AWS resource in Vscode using Terraform.
b.	Create two files provider.tf and main.tf
c.	Provider:
provider "aws" {
    region = "us-west-1"
  
}

terraform {
  required_providers {
    aws = {
        source = "hashicorp/aws"
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
d.	Inside this file you reference your AWS and Region your account is associated with. 
e.	AWS AMI data is basically going to retieve the latest ubuntu server ami available in my region.
2.	Once you finished the provider.tf you need to start on your main AWS infrastructure.
a.	Inside Maint.tf you will need the following resources:
b.	AWS Default VPC, AWS Default SG, IAM ROLE, AWS EC2, S3, S3 BUCKET,S3 BUCKET POLICY,
c.	The following Block of code is going to show the AWS VPC, AWS SG, with specific ingres ports being open.  
resource "aws_default_vpc" "default" {
    tags = {
      Name = "default"
    }
  
}
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
3.	Once Done You will need to create Your IAM ROLE that will have a policy allowing the ec2 to have full access to S3 buckets associated with your account.
a.	Also need yo attach the policy to the role and define a IAM instance profile to later be attached to the specific ec2 you want to have access to the s3
# Define the IAM Role
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

# Define the IAM Role Policy
resource "aws_iam_policy" "ec2-s3-fullAccess" {
  name        = "ec2-s3-fullAccess"
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
  roles      = [aws_iam_role.edwinRole.name]
  policy_arn = aws_iam_policy.ec2-s3-fullAccess.arn
}

# Define the IAM Instance Profile
resource "aws_iam_instance_profile" "ec2_profile" {
  name = "ec2_profile"
  role = aws_iam_role.edwinRole.name
}

4.	Next step is to Create Both EC2 One for Backend and the other for Database
a.	The ec2 is going to reference back to your AWS AMI for the latest ubuntu server ami in your region
b.	Select the instance type “t2.micro”
c.	Select the Pem key 
d.	Associate the instance profile if needed
e.	And also use the builtin function called provisioner “local-exec” to run a command that will receive the EC2 public IPV4 and storing it into a variable that will be inside a file that’s being routed to another location within the directory 
f.	Also the EC@ is going to be tagged with specific values to be refence later when needed to run playbooks using ansible

resource "aws_instance" "TodoList" {
     ami           = data.aws_ami.ubuntu_22.id
     instance_type = "t2.micro"
     key_name      = "cpdevopsew-us-west-1"
     iam_instance_profile = aws_iam_instance_profile.ec2_profile.name  # Use the instance profile
     provisioner "local-exec" {
        command = "echo 'TodoList: \"${aws_instance.TodoList.public_ip}\"'>/home/epquito/Terraform/todo-list-flask-mysql-app/final/backend_role/vars/main.yml "
       
     }
     tags = {
        Name = "TodoList"
        Enviroment = "dev"
        Team = "mobile-app"
        Type = "backend"
        }
}
resource "aws_instance" "TodoList-db" {
     ami           = data.aws_ami.ubuntu_22.id
     instance_type = "t2.micro"
     key_name      = "cpdevopsew-us-west-1"
     provisioner "local-exec" {
        command = "echo 'db_host: \"${aws_instance.TodoList-db.public_ip}\"'>/home/epquito/Terraform/todo-list-flask-mysql-app/final/backend_role/defaults/main.yml "
       
     }

     tags = {
        Name = "TodoList-db"
        Enviroment = "dev"
        Team = "mobile-app"
        Type = "database"
        }
}

output "TodoList" {
    value = aws_instance.TodoList.public_ip
  
}

output "TodoList-db" {
    value = aws_instance.TodoList.public_ip
  
}


5.	Now for the last peace of the terraform configuration is going to be Creating your S3 resource that will stored your frontend index.html file making it the front end of your application. 
a.	Within the configuration of the S3 bucket 
b.	Create bucket
c.	Create bucket object
d.	Configure/change the Bucket public access default settings 
e.	Create bucket json policy and attach it to your S3
f.	Bucket policy should allow the user to get the object with no issue 
resource "aws_s3_bucket" "edwin-bucket" {
  bucket = "edwin-bucket123"
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

  source = "/home/epquito/Terraform/todo-list-flask-mysql-app/index.html"
}

resource "aws_s3_bucket_public_access_block" "edwin-bucket" {
  bucket = aws_s3_bucket.edwin-bucket.id
  block_public_acls       = false
  block_public_policy     = false
  ignore_public_acls      = false
  restrict_public_buckets = false
}
# Define the Bucket Policy
resource "aws_s3_bucket_policy" "edwin-bucket-policy" {
  depends_on = [ aws_instance.TodoList, aws_instance.TodoList-db]
  bucket = aws_s3_bucket.edwin-bucket.id

  policy = jsonencode({
    Version = "2012-10-17",
    Statement = [
      {
        Effect    = "Allow",
        Principal = "*",
        Action    = "s3:GetObject",
        Resource  = "${aws_s3_bucket.edwin-bucket.arn}/*",
      },
    ],
  })
}

# Define Bucket Ownership Controls
resource "aws_s3_bucket_ownership_controls" "edwin-bucket" {
  bucket = aws_s3_bucket.edwin-bucket.id

  rule {
    object_ownership = "BucketOwnerPreferred"
  }
}

# Output the Bucket Website Endpoint
output "bucket_website_endpoint" {
  value = aws_s3_bucket.edwin-bucket.website_endpoint

}


6.	Once All parts been configure you can use the command 
a.	Terraform plan
b.	Terraform apply
7.	If the terraform commands run without issue you can start on creating your ansible playbooks/dynamic inventory/ ansible roles
a.	Create another directory called Final or anything you want 
b.	Inside the final directory you will create your dynamic inventory aws_ec2.yml
c.	Inside the dynamic inventory playbook you will adentify your plugin,region,keyed_groups,filters.
---
plugin: aws_ec2
regions:
  - us-west-1

keyed_groups:
  # Create groups based on the value of the 'Type' tag
  - key: tags['Type']
    prefix: tag_Type

# Filter instances to include those tagged as either 'backend' or 'database'
filters:
  tag:Type:
    - database
    - backend

8.	Once dynamic inventory is created you will need to create a playbook that will call upon the roles backend_role and database_role and will trigger them to run there own playbooks/tasks in order which the role.yml is configured
---
- name: database server
  hosts: tag_Type_database
  become: true
  vars_files:
   - /home/epquito/Terraform/todo-list-flask-mysql-app/final/database_role/vars/vault.yml
  roles:
   - { role: database_role}

- name: backend server
  hosts: tag_Type_backend
  become: true
  vars_files:
   - /home/epquito/Terraform/todo-list-flask-mysql-app/final/backend_role/vars/vault.yml
  roles:
   - { role: backend_role}
9.	Now you have create two other directories that are named the same as the roles and inside the directories you create you need to make 6 directories called 

a.	Defaults
i.	Main.yml
b.	 Files
i.	Index.html
c.	 Handlers
i.	Main.yml
d.	 Meta
i.	Main.yml.bak
e.	 Templates
i.	Vhost.tpl
f.	 Tasks
i.	Main.yml
g.	 Vars
i.	Main.yml
ii.	Vault.yml
10.	 Once all these directories are created for each role you can now add your playbooks in the respected spot 
 
11.	WE can now start on the backend role 
a.	Inside we are going to use the handlers,tasks,vars directory
b.	Tasks:	
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
   bucket: edwin-bucket123
   mode: get
   object: "index.html"
   dest: /home/ubuntu/todolist-flask/index.html

- name: update api url
  lineinfile:
   path: /home/ubuntu/todolist-flask/index.html
   regexp: "const API_URL ="
   line: "const API_URL = 'http://{{TodoList}}'"

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
   bucket: edwin-bucket123
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

12.	The playbook does these specific configurations:
a.	Updates the ubuntu server
b.	Installes the dependencies
c.	Retrieves repository where the application resides from branch env
d.	Gets rid of the index.html already inside the repository
e.	Downloads the index.html inside the s3 bucket
f.	We change a specific line inside the index.html to reference our ec2 ip stored in default directory
g.	We create our .env file inside the application directory in the ec2 with all the database information stored in vars directory in vault.yml
h.	Re upload s3 index.html from ec2 to s3 with the updated changes to api url
i.	We now install requirements.txt 
j.	Create a gunicorn file that will start up the application -. Bind it to a specific ip/port and make it from single thread to muilti thread
k.	Once the guniconr file is created we will create a service file inside the ec2 at /etc/cyctem/system
l.	We will start the service file
m.	Then do a daemon-restart that will notify the Handler directory and restart the todolist service file
13.	 Now we go on to the database_role
a.	We will use the handlers,tasks,vars directory 
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
b.	Inside the tasks the playbook will update the server
c.	Install the depenecies for mysql
d.	Start mysql servers
e.	It will do a mysql secure installation
i.	We are using args and stdin to execute mysql commands to set root password and to delete any existing default users except the root and tables 
f.	Create mysql user and give it al rpidliges
g.	Create db 
h.	Changing mywl config file to allow remote access by binding the address to anyone using 0.0.0.0
i.	We notify handlers with the last task being dine by using notify 
14.	Vars hold are vault.yml which encrypt our db information
15.	 Once everything is done go back to the final directory and run this command 
a.	Ansible-playbook –ask-vault-pass role.yml -i aws_ec2.yml -u ubuntu –private-key= path to pem file
16.	This command will execute both roles in order and run the playbooks inside 
