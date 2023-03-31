---
# User change
title: "Deploy Cluster Automatically"

weight: 2 # 1 is first, 2 is second, etc.

# Do not modify these elements
layout: "learningpathall"
---

##  Deploy seven instances(three for Zookeeper cluster, three for Kafka cluster and one for Client) 

You can deploy three node Zookeeper cluster, three node Kafka cluster and one client on AWS Graviton processors using Terraform and Ansible. 
 
If you are new to Terraform, you should look at [Automate AWS EC2 instance creation using Terraform](/learning-paths/server-and-cloud/aws-terraform/terraform/) before starting this Learning Path.

## Before you begin

You should have the prerequisite tools installed before starting the Learning Path. 

Any computer which has the required tools installed can be used for this section. The computer can be your desktop or laptop computer or a virtual machine with the required tools. 

You will need an [AWS account](https://portal.aws.amazon.com/billing/signup?nc2=h_ct&src=default&redirect_url=https%3A%2F%2Faws.amazon.com%2Fregistration-confirmation#/start) to complete this Learning Path. Create an account if you don't have one.

Before you begin you will also need:
- An AWS access key ID and secret access key. 
- An SSH key pair

The instructions to create the keys are below.

### Generate the SSH key-pair

Generate the SSH key-pair (public key, private key) using `ssh-keygen` to use for AWS EC2 access. To generate the key-pair, follow this [documentation](/install-guides/ssh#ssh-keys).

{{% notice Note %}}
If you already have an SSH key-pair present in the `~/.ssh` directory, you can skip this step.
{{% /notice %}}

### Generate and configure Access keys (Access key ID and Secret access key)

The installation of Terraform on your desktop or laptop needs to communicate with AWS. Thus, Terraform needs to be able to authenticate with AWS.

To generate and configure the Access key ID and Secret access key, follow this [documentation](/install-guides/aws_access_keys).

## Create seven AWS EC2 instance using Terraform

Using a text editor, save the code below to in a file called `main.tf`

Scroll down to see the information you need to change in `main.tf`

```console
provider "aws" {
  region = "us-east-2"
  access_key  = "xxxxxxxxxxxxxxx"
  secret_key   = "xxxxxxxxxxxxxxx"
}
  resource "aws_instance" "ABHISHEK_KAFKA_TEST" {
  count         = "7"
  ami           = "ami-0ca2eafa23bc3dd01"
  instance_type = "t4g.small"
  security_groups= [aws_security_group.Terraformsecurity500.name]
  key_name = "kafka"
  tags = {
    Name = "ABHISHEK_KAFKA_TEST"
  }
}
resource "aws_default_vpc" "main" {
  tags = {
    Name = "main"
  }
}
resource "aws_security_group" "Terraformsecurity500" {
  name        = "Terraformsecurity500"
  description = "Allow TLS inbound traffic"
  vpc_id      = aws_default_vpc.main.id

  ingress {
    description      = "TLS from VPC"
    from_port        = 2181
    to_port          = 2181
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
}


  ingress {
    description      = "TLS from VPC"
    from_port        = 9092
    to_port          = 9092
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
}


  ingress {
    description      = "TLS from VPC"
    from_port        = 2888
    to_port          = 2888
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
}


  ingress {
    description      = "TLS from VPC"
    from_port        = 3888
    to_port          = 3888
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
}

  ingress {
    description      = "TLS from VPC"
    from_port        = 22
    to_port          = 22
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
  }
  egress {
    from_port        = 0
    to_port          = 0
    protocol         = "-1"
    cidr_blocks      = ["0.0.0.0/0"]
  }

  tags = {
    Name = "Terraformsecurity500"
  }
}
resource "local_file" "inventory" {
    depends_on=[aws_instance.ABHISHEK_KAFKA_TEST]
    filename = "./hosts"
    content = <<EOF
[zookeeper1]
${aws_instance.ABHISHEK_KAFKA_TEST[0].public_ip}
[zookeeper2]
${aws_instance.ABHISHEK_KAFKA_TEST[1].public_ip}
[zookeeper3]
${aws_instance.ABHISHEK_KAFKA_TEST[2].public_ip}
[kafka1]
${aws_instance.ABHISHEK_KAFKA_TEST[3].public_ip}
[kafka2]
${aws_instance.ABHISHEK_KAFKA_TEST[4].public_ip}
[kafka3]
${aws_instance.ABHISHEK_KAFKA_TEST[5].public_ip}
[client1]
${aws_instance.ABHISHEK_KAFKA_TEST[6].public_ip}

[all:vars]
ansible_connection=ssh
ansible_user=ubuntu
                EOF
}

resource "aws_key_pair" "deployer" {
        key_name   = "kafka"
        public_key = "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDffAvwWmMiVdXZ+uu0OoqjueUySSQwXYkYV3YZp6svQDYyPvIIkLQZB+qaFyBaoLIDVHriDDP6xg1Oc/YteNg93NjeZ6WIOx2a5ONbZCsmVXqC+vv/Z4xhjpDpnwUVB0jG+et7PLHCp7qUrCu60bFRuI4F4EXydFt4RWJHtxIf1OglfJITnAiC305KjTjwrOEsi/0L6qV98Y2abo8X3Fxrd0sf4uOLYiy53Ia2xEGvD8tmwKId2Kd8cymopCxe/F2kJISG0AydYdRzR2dx4xE9RmuZ8GtpGUSCs6hQgDNNQJ8INNQJ32gUHL2AAtGS0UkJvTsR1Hk6WyCccdeV1K+b/YBUQs2U74yivxMLYJ5ngzeERYWpLPAxOood506gjeZj1sxn8olJAQ0nfRuPYnub5P2F4IjZCQLu4S1M0YyivwlpzTL5X+ZboK2DT3uAVJ/Qs/OtvWbk74v+FHKIwGfhunIBfflJ8tLFt177w+LAcomydV+SbI/Y1a/DS5NPxMk= root@ip-172-31-38-39"
}

```

Make the changes listed below in `main.tf` to match your account settings.

1. In the `provider` section, update all 3 values to use your preferred AWS region and your AWS access key ID and secret access key.

2. (optional) In the `aws_instance` section, change the ami value to your preferred Linux distribution. The AMI ID for Ubuntu 22.04 on Arm in the us-east-1 region is `ami-0f9bd9098aca2d42b ` No change is needed if you want to use Ubuntu AMI in us-east-1. The AMI ID values are region specific and need to be changed if you use another AWS region. Use the AWS EC2 console to find an AMI ID or refer to [Find a Linux AMI](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/finding-an-ami.html).

{{% notice Note %}}
The instance type is t4g.small. This an an Arm-based instance and requires an Arm Linux distribution.
{{% /notice %}}

3. in the `local_file` section, change the `filename` to be the path to your current directory.

The hosts file is automatically generated and does not need to be changed, change the path to the location of the hosts file.


## Terraform Commands

Use Terraform to deploy the `main.tf` file.

### Initialize Terraform

Run `terraform init` to initialize the Terraform deployment. This command downloads the dependencies required for AWS.

```console
terraform init
```
    
The output should be similar to:

```output
Initializing the backend...

Initializing provider plugins...
- Finding latest version of hashicorp/local...
- Finding latest version of hashicorp/aws...
- Installing hashicorp/local v2.4.0...
- Installed hashicorp/local v2.4.0 (signed by HashiCorp)
- Installing hashicorp/aws v4.58.0...
- Installed hashicorp/aws v4.58.0 (signed by HashiCorp)

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
```

### Create a Terraform execution plan

Run `terraform plan` to create an execution plan.

```console
terraform plan
```

A long output of resources to be created will be printed. 

### Apply the Terraform execution plan

Run `terraform apply` to apply the execution plan and create all AWS resources: 

```console
terraform apply
```      

Answer `yes` to the prompt to confirm you want to create AWS resources. 


## Configure three node Zookeeper cluster through Ansible

Install the Zookeeper and the required dependencies. 

Using a text editor, save the code below to in a file called `zookeeper_cluster.yaml`. This is the YAML file for the Ansible playbook. 

```console
- hosts: zookeeper1, zookeeper2, zookeeper3
  become: True
  vars:
    - zk_1_ip: "3.134.244.225"
    - zk_2_ip: "18.117.161.107"
    - zk_3_ip: "3.17.129.114"
  tasks:

  - name: Update machines and install Java, Zookeeper
    shell: |
           apt update
           apt install -y default-jdk
           mkdir Zookeeper_node
           cd Zookeeper_node/ && wget https://dlcdn.apache.org/zookeeper/zookeeper-3.8.0/apache-zookeeper-3.8.0-bin.tar.gz && tar -xzf apache-zookeeper-3.8.0-bin.tar.gz && cd apache-zookeeper-3.8.0-bin

  - name: On zookeeper1 instance create Zookeeper directory
    when: "'zookeeper1' in group_names"
    shell: mkdir /tmp/zookeeper && echo 1 >> /tmp/zookeeper/myid
    
  - name: On zookeeper1 instance setup confiuration for Zookeeper cluster
    when: "'zookeeper1' in group_names"
    copy:
      dest: "/home/ubuntu/Zookeeper_node/apache-zookeeper-3.8.0-bin/conf/zoo.cfg"
      content: |
        tickTime=2000
        dataDir=/tmp/zookeeper
        clientPort=2181
        maxClientCnxns=60
        initLimit=10
        syncLimit=5
        4lw.commands.whitelist=*
        server.1=0.0.0.0:2888:3888
        server.2={{zk_2_ip}}:2888:3888
        server.3={{zk_3_ip}}:2888:3888

  - name: On zookeeper2 instance create Zookeeper directory
    when: "'zookeeper2' in group_names"
    shell: mkdir /tmp/zookeeper && echo 2 >> /tmp/zookeeper/myid
  - name: On zookeeper1 instance setup confiuration for Zookeeper cluster
    when: "'zookeeper2' in group_names"
    copy:
      dest: "/home/ubuntu/Zookeeper_node/apache-zookeeper-3.8.0-bin/conf/zoo.cfg"
      content: |
        tickTime=2000
        dataDir=/tmp/zookeeper
        clientPort=2181
        maxClientCnxns=60
        initLimit=10
        syncLimit=5
        4lw.commands.whitelist=*
        server.1={{zk_1_ip}}:2888:3888
        server.2=0.0.0.0:2888:3888
        server.3={{zk_3_ip}}:2888:3888
        
  - name: On zookeeper3 instance create Zookeeper directory
    when: "'zookeeper3' in group_names"
    shell: mkdir /tmp/zookeeper && echo 3 >> /tmp/zookeeper/myid

  - name: On zookeeper3 instance setup confiuration for Zookeeper cluster
    when: "'zookeeper3' in group_names"
    copy:
      dest: "/home/ubuntu/Zookeeper_node/apache-zookeeper-3.8.0-bin/conf/zoo.cfg"
      content: |
        tickTime=2000
        dataDir=/tmp/zookeeper
        clientPort=2181
        maxClientCnxns=60
        initLimit=10
        syncLimit=5
        4lw.commands.whitelist=*
        server.1={{zk_1_ip}}:2888:3888
        server.2={{zk_2_ip}}:2888:3888
        server.3=0.0.0.0:2888:3888
        
  - name: Start Zookeeper server
    command: /home/ubuntu/Zookeeper_node/apache-zookeeper-3.8.0-bin/bin/zkServer.sh start
        
```

Provide the IP of zookeeper changes are required to the file.

### Ansible Commands

Substitute your private key name, and run the playbook using the  `ansible-playbook` command:

```console
ansible-playbook playbook.yaml -i hosts
```

Answer `yes` when prompted for the SSH connection. 

Deployment may take a few minutes. 

The output should be similar to:

```output
PLAY [all] *********************************************************************

TASK [Gathering Facts] *********************************************************
The authenticity of host '3.84.22.24 (3.84.22.24)' can't be established.
ED25519 key fingerprint is SHA256:igz06Iz4YiilC08oFy8E5E78KCaJzYIthVpt1zhq9KM.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
ok: [3.84.22.24]

TASK [Update the Machine & Install PostgreSQL] *********************************
changed: [3.84.22.24]

TASK [Update apt repo and cache on all Debian/Ubuntu boxes] ********************
changed: [3.84.22.24]

TASK [Install Python pip & Python package] *************************************
changed: [3.84.22.24] => (item=python3-pip)

TASK [Find out if PostgreSQL is initialized] ***********************************
ok: [3.84.22.24]

TASK [Start and enable services] ***********************************************
ok: [3.84.22.24] => (item=postgresql)

PLAY RECAP *********************************************************************
3.84.22.24                 : ok=6    changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```

## Connect to the database 

Execute the steps below to try out PostgreSQL.

1. Connect to the database using SSH to the public IP of the AWS EC2 instance. 

```console
ssh ubuntu@<public-IP-address>
```

2. Log in to postgres using the commands:

```console
cd ~postgres/
sudo su postgres -c psql
```

This will enter the PostgreSQL command prompt.

```console
psql (15.2 (Ubuntu 15.2-1.pgdg22.04+1))
Type "help" for help.

postgres=# 
```

3. Create a new database:

```console
create database testdb;
```

The output will be:

```output
CREATE DATABASE
```

4. List all databases: 

```console
 \l
```

The output will be:

```output
                                             List of databases
   Name    |  Owner   | Encoding | Collate |  Ctype  | ICU Locale | Locale Provider |   Access privileges   
-----------+----------+----------+---------+---------+------------+-----------------+-----------------------
 postgres  | postgres | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            | 
 template0 | postgres | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            | =c/postgres          +
           |          |          |         |         |            |                 | postgres=CTc/postgres
 template1 | postgres | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            | =c/postgres          +
           |          |          |         |         |            |                 | postgres=CTc/postgres
 testdb    | postgres | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            | 
(4 rows)
```

5. Switch to the new databases:

```console
 \c testdb;
```

The output confirms you have changed to the new database:

```output
You are now connected to database "testdb" as user "postgres".
```

6. Create a new table in the database:

```console
CREATE TABLE company ( emp_name VARCHAR, emp_dpt VARCHAR);
```

The output will be:

```output
CREATE TABLE
```

7. Display the database tables: 

```console
 \dt;
```

The tables will be printed:

```console
          List of relations
 Schema |  Name   | Type  |  Owner   
--------+---------+-------+----------
 public | company | table | postgres
(1 row)
```

8. Insert data into the table:

```console
INSERT INTO company VALUES ('Herry', 'Development'), ('Tom', 'Testing'),('Ankit', 'Sales'),('Manoj', 'HR'),('Noy', 'Presales');
```

The output will be:

```output
INSERT 0 5
```

9. Print the contents of the table:

```console
select * from company;
```

The output will be:

```output
 emp_name |   emp_dpt   
----------+-------------
 Herry    | Development
 Tom      | Testing
 Ankit    | Sales
 Manoj    | HR
 Noy      | Presales
(5 rows)
```

You have successfully installed PostgreSQL on an AWS EC2 instance running Graviton processors. 

### Clean up resources

Run `terraform destroy` to delete all resources created.

```console
terraform destroy
```

Continue the Learning Path to create a multi-node PostgreSQL deployment. 
