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
  resource "aws_instance" "KAFKA_TEST" {
  count         = "7"
  ami           = "ami-0ca2eafa23bc3dd01"
  instance_type = "t4g.small"
  security_groups= [aws_security_group.Terraformsecurity500.name]
  key_name = aws_key_pair.deployer.key_name  
  tags = {
    Name = "KAFKA_TEST"
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
    depends_on=[aws_instance.KAFKA_TEST]
    filename = "/tmp/inventory"
    content = <<EOF
[zookeeper1]
${aws_instance.KAFKA_TEST[0].public_ip}
[zookeeper2]
${aws_instance.KAFKA_TEST[1].public_ip}
[zookeeper3]
${aws_instance.KAFKA_TEST[2].public_ip}
[kafka1]
${aws_instance.KAFKA_TEST[3].public_ip}
[kafka2]
${aws_instance.KAFKA_TEST[4].public_ip}
[kafka3]
${aws_instance.KAFKA_TEST[5].public_ip}
[client]
${aws_instance.KAFKA_TEST[6].public_ip}

[all:vars]
ansible_connection=ssh
ansible_user=ubuntu
                EOF
}

resource "aws_key_pair" "deployer" {
        key_name   = "id_rsa"
        public_key = file("~/.ssh/id_rsa.pub")
}

```

Make the changes listed below in `main.tf` to match your account settings.

1. In the `provider` section, update all 3 values to use your preferred AWS region and your AWS access key ID and secret access key.

2. (optional) In the `aws_instance` section, change the ami value to your preferred Linux distribution. The AMI ID for Ubuntu 22.04 on Arm in the us-east-1 region is `ami-0f9bd9098aca2d42b ` No change is needed if you want to use Ubuntu AMI in us-east-1. The AMI ID values are region specific and need to be changed if you use another AWS region. Use the AWS EC2 console to find an AMI ID or refer to [Find a Linux AMI](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/finding-an-ami.html).

{{% notice Note %}}
The instance type is t4g.small. This an an Arm-based instance and requires an Arm Linux distribution.
{{% /notice %}}

The inventory file is automatically generated and does not need to be changed.

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
    - zk_1_ip: "zookeeper1_ip"
    - zk_2_ip: "zookeeper2_ip"
    - zk_3_ip: "zookeeper3_ip"
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

Replace zookeeper1_ip, zookeeper2_ip, zookeeper3_ip with the IP of zookeeper1, zookeeper2 and zookeeper3 respectively generated in inventory file present at location `/tmp/inventory`.

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

## Configure three node Kafka cluster through Ansible

Install the Kafka and the required dependencies. 

Using a text editor, save the code below to in a file called `kafka_cluster.yaml`. This is the YAML file for the Ansible playbook. 

```console
- hosts: kafka1, kafka2, kafka3
  become: True
  vars:
    - zk_1_ip: "zookeeper1_ip"
    - zk_2_ip: "zookeeper2_ip"
    - zk_3_ip: "zookeeper3_ip"
  tasks:
  - name: Update machines and install Java, Kafka
    shell: |
           apt update
           apt install -y default-jdk
           mkdir kafka_node
           cd kafka_node/ && wget https://dlcdn.apache.org/kafka/3.2.3/kafka_2.13-3.2.3.tgz && tar -xzf kafka_2.13-3.2.3.tgz && cd kafka_2.13-3.2.3

  - name: On kafka1 instance create log directory
    when: "'kafka1' in group_names"
    shell: mkdir /tmp/kafka-logs

  - name: On kafka1 instance update broker id
    when: "'kafka1' in group_names"
    lineinfile:
      path: /home/ubuntu/kafka_node/kafka_2.13-3.2.3/config/server.properties
      regexp: '^(.*)broker.id(.*)$'
      line: 'broker.id=1'
      backrefs: yes

  - name: On kafka1 instance uncomment listeners
    when: "'kafka1' in group_names"
    lineinfile:
      path: /home/ubuntu/kafka_node/kafka_2.13-3.2.3/config/server.properties
      regexp: '^(.*)#listeners=PLAINTEXT://:9092(.*)$'
      line: 'listeners=PLAINTEXT://:9092'
      backrefs: yes

  - name: On kafka1 instance update zookeeper.connect
    when: "'kafka1' in group_names"
    lineinfile:
      path: /home/ubuntu/kafka_node/kafka_2.13-3.2.3/config/server.properties
      regexp: '^(.*)zookeeper.connect=localhost:2181(.*)$'
      line: 'zookeeper.connect={{zk_1_ip}}:2181,{{zk_2_ip}}:2181,{{zk_3_ip}}:2181'
      backrefs: yes

  - name: On kafka2 instance create log directory
    when: "'kafka2' in group_names"
    shell: mkdir /tmp/kafka-logs

  - name: On kafka2 instance update broker id
    when: "'kafka2' in group_names"
    lineinfile:
      path: /home/ubuntu/kafka_node/kafka_2.13-3.2.3/config/server.properties
      regexp: '^(.*)broker.id(.*)$'
      line: 'broker.id=2'
      backrefs: yes
  - name: On kafka2 instance uncomment listeners
    when: "'kafka2' in group_names"
    lineinfile:
      path: /home/ubuntu/kafka_node/kafka_2.13-3.2.3/config/server.properties
      regexp: '^(.*)#listeners=PLAINTEXT://:9092(.*)$'
      line: 'listeners=PLAINTEXT://:9092'
      backrefs: yes

  - name: On kafka2 instance update zookeeper.connect
    when: "'kafka2' in group_names"
    lineinfile:
      path: /home/ubuntu/kafka_node/kafka_2.13-3.2.3/config/server.properties
      regexp: '^(.*)zookeeper.connect=localhost:2181(.*)$'
      line: 'zookeeper.connect={{zk_1_ip}}:2181,{{zk_2_ip}}:2181,{{zk_3_ip}}:2181'
      backrefs: yes


  - name: On kafka3 instance create log directory
    when: "'kafka3' in group_names"
    shell: mkdir /tmp/kafka-logs

  - name: On kafka3 instance update broker id
    when: "'kafka3' in group_names"
    lineinfile:
      path: /home/ubuntu/kafka_node/kafka_2.13-3.2.3/config/server.properties
      regexp: '^(.*)broker.id(.*)$'
      line: 'broker.id=3'
      backrefs: yes

  - name: On kafka3 instance uncomment listeners
    when: "'kafka3' in group_names"
    lineinfile:
      path: /home/ubuntu/kafka_node/kafka_2.13-3.2.3/config/server.properties
      regexp: '^(.*)#listeners=PLAINTEXT://:9092(.*)$'
      line: 'listeners=PLAINTEXT://:9092'
      backrefs: yes

  - name: On kafka3 instance update zookeeper.connect
    when: "'kafka3' in group_names"
    lineinfile:
      path: /home/ubuntu/kafka_node/kafka_2.13-3.2.3/config/server.properties
      regexp: '^(.*)zookeeper.connect=localhost:2181(.*)$'
      line: 'zookeeper.connect={{zk_1_ip}}:2181,{{zk_2_ip}}:2181,{{zk_3_ip}}:2181'
      backrefs: yes

  - name: Start kafka_server
    command: /home/ubuntu/kafka_node/kafka_2.13-3.2.3/bin/kafka-server-start.sh /home/ubuntu/kafka_node/kafka_2.13-3.2.3/config/server.properties

        
```

Replace zookeeper1_ip, zookeeper2_ip, zookeeper3_ip with the IP of zookeeper1, zookeeper2 and zookeeper3 respectively generated in inventory file present at location `/tmp/inventory`.

After successfully setting up a 3 node Kafka cluster, we can verify it works by creating a topic and storing the events. Follow the steps below to create a topic, write some events into the topic, and then read the events.

## Configure client through Ansible

Install the Kafka and the required dependencies. 

Using a text editor, save the code below to in a file called `client.yaml`. This is the YAML file for the Ansible playbook. 

```console
- hosts: client1
  become: true
  vars:
    - kf_1_ip: "18.221.58.152"
    - kf_2_ip: "3.21.19.58"
    - kf_3_ip: "18.117.95.223"
  tasks:

  - name: Update machines and install Java, Kafka
    shell: |
           apt update
           apt install -y default-jdk
           mkdir kafka_node
           cd kafka_node/ && wget https://dlcdn.apache.org/kafka/3.2.3/kafka_2.13-3.2.3.tgz && tar -xzf kafka_2.13-3.2.3.tgz && cd kafka_2.13-3.2.3

  - name: Create a topic
    command: /home/ubuntu/kafka_node/kafka_2.13-3.2.3/bin/kafka-topics.sh --create --topic test-topic --bootstrap-server {{kf_1_ip}}:9092,{{kf_2_ip}}:9092,{{kf_3_ip}}:9092 --replication-factor 3 --partitions 64

```

## Describe the topic created:

Run in the same client terminal where the topic was created.

```console

./bin/kafka-topics.sh --topic test-topic --bootstrap-server kf_1_ip:9092,kf_2_ip:9092,kf_3_ip:9092 --describe

```
![describe_command (3)](https://user-images.githubusercontent.com/66300308/196909519-20eb720c-e43a-4eb7-abfa-405d4f931417.png)

## Run the producer client to write events into the created topic:

Run in the same client terminal where the topic was created.

```console

./bin/kafka-console-producer.sh --topic test-topic --bootstrap-server kf_1_ip:9092,kf_2_ip:9092,kf_3_ip:9092

```

![producer_message](https://user-images.githubusercontent.com/66300308/196901191-bc9944cb-8bd9-4d0c-8096-00b1a5a23452.png)

## Run the consumer client to read all the events created:

Open a new terminal on the client machine to run the consumer client.

```console

./bin/kafka-console-consumer.sh --topic test-topic --bootstrap-server kf_1_ip:9092,kf_2_ip:9092,kf_3_ip:9092

```
![consumer_message](https://user-images.githubusercontent.com/66300308/196900853-e8ca972c-6154-4a5c-92ac-3af92410ba95.png)

Write a message into the producer client terminal and press enter. You should see the same message appear on consumer client terminal. 

[<-- Return to Learning Path](/content/en/cloud/kafka/#sections)




