---
# User change
title: "Deploy Cluster Automatically(azure)"

weight: 6 # 1 is first, 2 is second, etc.

# Do not modify these elements
layout: "learningpathall"
---

##  Deploy Cluster Automatically 

You can deploy three node Zookeeper cluster, three node Kafka cluster and one client on Azure using Terraform and Ansible. 
In this section, you will deploy Redis as a cache for MySQL on an Azure instance. 

If you are new to Terraform, you should look at [Automate Azure instance creation using Terraform](/learning-paths/server-and-cloud/azure/terraform/) before starting this Learning Path.

## Before you begin

You should have the prerequisite tools installed before starting the Learning Path. 

Any computer which has the required tools installed can be used for this section. The computer can be your desktop or laptop computer or a virtual machine with the required tools. 

You will need an [Azure portal account](https://azure.microsoft.com/en-in/get-started/azure-portal) to complete this Learning Path. Create an account if you don't have one.

Before you begin, you will also need:
- Login to the Azure CLI
- An SSH key pair

The instructions to login to the Azure CLI and create the keys are below.

### Azure authentication

The installation of Terraform on your Desktop/Laptop needs to communicate with Azure. Thus, Terraform needs to be authenticated.

For authentication, follow the [steps from the Terraform Learning Path](/learning-paths/server-and-cloud/azure/terraform#azure-authentication).

### Generate an SSH key-pair

Generate an SSH key-pair (public key, private key) using `ssh-keygen` to use for Azure instance access. To generate the key-pair, follow this [
documentation](/install-guides/ssh#ssh-keys).

{{% notice Note %}} 
If you already have an SSH key-pair present in the `~/.ssh` directory, you can skip this step.
{{% /notice %}}

## Create Azure instances using Terraform

For Azure Arm based instance deployment, the Terraform configuration is broken into three files: `providers.tf`, `variables.tf` and `main.tf`. Here we are creating 7 instances.

Add the following code in `providers.tf` file to configure Terraform to communicate with Azure.
    
```console
terraform {
  required_version = ">=0.12"

  required_providers {
    azurerm = {
      source= "hashicorp/azurerm"
      version = "~>2.0"
    }
    random = {
      source= "hashicorp/random"
      version = "~>3.0"
    }
    tls = {
    source = "hashicorp/tls"
    version = "~>4.0"
    }
  }
}

provider "azurerm" {
  features {}
}
    
```
Create a `variables.tf` file for describing the variables referenced in the other files with their type and a default value.

```console
variable "resource_group_location" {
  default = "eastus2"
  description = "Location of the resource group."
}

variable "resource_group_name_prefix" {
  default = "rg"
  description = "Prefix of the resource group name that's combined with a random ID so name is unique in your Azure subscription."
}
```

Add the resources required to create a virtual machine in `main.tf`.

```console
resource "random_pet" "rg_name" {
  prefix = var.resource_group_name_prefix
}

resource "azurerm_resource_group" "rg" {
  location = var.resource_group_location
  name = random_pet.rg_name.id
}

# Create virtual network
resource "azurerm_virtual_network" "my_terraform_network" {
  name = "myVnet"
  address_space = ["10.1.0.0/16"]
  location = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
}

# Create subnet
resource "azurerm_subnet" "my_terraform_subnet" {
  name = "mySubnet"
  resource_group_name = azurerm_resource_group.rg.name
  virtual_network_name = azurerm_virtual_network.my_terraform_network.name
  address_prefixes = ["10.1.1.0/24"]
}

# Create Public IPs
resource "azurerm_public_ip" "my_terraform_public_ip" {
  name = "myPublicIP${format("%02d", count.index)}-test"
  count= 2
  location = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  allocation_method = "Dynamic"
}

# Create Network Security Group and rule
resource "azurerm_network_security_group" "my_terraform_nsg" {
  name= "myNetworkSecurityGroup"
  location= azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  security_rule {
    name= "SSH"
    priority= 1001
    direction= "Inbound"
    access = "Allow"
    protocol= "Tcp"
    source_port_range= "*"
    destination_port_range = "22"
    source_address_prefix= "*"
    destination_address_prefix = "*"
  }
  security_rule {
    name= "MYSQL"
    priority= 1002
    direction= "Inbound"
    access = "Allow"
    protocol= "Tcp"
    source_port_range= "*"
    destination_port_range = "3306"
    source_address_prefix= "*"
    destination_address_prefix = "*"
  }
}

# Create network interface
resource "azurerm_network_interface" "my_terraform_nic" {
  count= 2
  name= "NIC-${format("%02d", count.index)}-test"
  location= azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  ip_configuration {
    name= "my_nic_configuration"
    subnet_id = azurerm_subnet.my_terraform_subnet.id
    private_ip_address_allocation = "Dynamic"
    public_ip_address_id= azurerm_public_ip.my_terraform_public_ip.*.id[count.index]
  }
}

# Connect the security group to the network interface
resource "azurerm_network_interface_security_group_association" "example" {
  count= 2
  network_interface_id= azurerm_network_interface.my_terraform_nic.*.id[count.index]
  network_security_group_id = azurerm_network_security_group.my_terraform_nsg.id
}

# Generate random text for a unique storage account name
resource "random_id" "random_id" {
  keepers = {
    # Generate a new ID only when a new resource group is defined
    resource_group = azurerm_resource_group.rg.name
  }

  byte_length = 8
}

# Create storage account for boot diagnostics
resource "azurerm_storage_account" "my_storage_account" {
  name = "diag${random_id.random_id.hex}"
  location = azurerm_resource_group.rg.location
  resource_group_name= azurerm_resource_group.rg.name
  account_tier = "Standard"
  account_replication_type = "LRS"
}

# Create virtual machine
resource "azurerm_linux_virtual_machine" "MYSQL_TEST" {
  name= "MYSQL_TEST${format("%02d", count.index + 1)}"
  count= 2
  location= azurerm_resource_group.rg.location
  resource_group_name= azurerm_resource_group.rg.name
  network_interface_ids = [azurerm_network_interface.my_terraform_nic.*.id[count.index]]
  size= "Standard_D2ps_v5"

  os_disk {
    name = "myOsDisk${format("%02d", count.index + 1)}"
    caching= "ReadWrite"
    storage_account_type = "Premium_LRS"
  }

  source_image_reference {
    publisher = "Canonical"
    offer = "0001-com-ubuntu-server-focal"
    sku= "20_04-lts-arm64"
    version= "20.04.202209200"
  }

  computer_name= "myvm"
  admin_username= "ubuntu"
  disable_password_authentication = true

  admin_ssh_key {
    username= "ubuntu"
    public_key = file("~/.ssh/id_rsa.pub")
  }

  boot_diagnostics {
    storage_account_uri = azurerm_storage_account.my_storage_account.primary_blob_endpoint
  }
}
resource "local_file" "inventory" {
    depends_on=[azurerm_linux_virtual_machine.MYSQL_TEST]
    filename = "/tmp/inventory"
    content = <<EOF
[mysql1]
${azurerm_linux_virtual_machine.MYSQL_TEST[0].public_ip_address}
[mysql2]
${azurerm_linux_virtual_machine.MYSQL_TEST[1].public_ip_address}
[all:vars]
ansible_connection=ssh
ansible_user=ubuntu
                EOF
}
```
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
root@ip-172-31-38-39:/home/ubuntu/kf# terraform init

Initializing the backend...

Initializing provider plugins...
- Reusing previous version of hashicorp/aws from the dependency lock file
- Reusing previous version of hashicorp/local from the dependency lock file
- Using previously-installed hashicorp/aws v4.61.0
- Using previously-installed hashicorp/local v2.4.0

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

The output should be similar to:
```output
local_file.inventory: Creating...
local_file.inventory: Creation complete after 0s [id=034d680c1738cdf217a5ad90885e6ec262470127]

Apply complete! Resources: 11 added, 0 changed, 0 destroyed.
```
## Configure three node Zookeeper cluster through Ansible

You can use the same `zookeeper_cluster.yaml` file used in the section, [Deploy Cluster Automatically (AWS)](/learning-paths/server-and-cloud/kafka/kafka_aws.md).

### Ansible Commands

Run the playbook using the  `ansible-playbook` command:
```console
ansible-playbook zookeeper_cluster.yaml -i /tmp/inventory
```
Answer `yes` when prompted for the SSH connection. 

Deployment may take a few minutes. 

The output should be similar to:
```output
root@ip-172-31-38-39:/home/ubuntu/kf# ansible-playbook zookeeper_cluster.yaml -i /tmp/inventory

PLAY [zookeeper1, zookeeper2, zookeeper3] ***********************************************************************************************

TASK [Gathering Facts] ******************************************************************************************************************
The authenticity of host '18.222.137.38 (18.222.137.38)' can't be established.
ED25519 key fingerprint is SHA256:078u3zYJtAMqmZTWI1qGLNTdlp/iDtqgxnrUV7NLjNA.
This key is not known by any other names
The authenticity of host '18.117.144.56 (18.117.144.56)' can't be established.
ED25519 key fingerprint is SHA256:pdmVviF1LTQOrM5t0Pp3yrsL+M3DsJdbYjGtmb5h44c.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? ok: [3.140.201.79]
yes
ok: [18.222.137.38]
yes
ok: [18.117.144.56]

TASK [Update machines and install Java, Zookeeper] **************************************************************************************
changed: [18.222.137.38]
changed: [3.140.201.79]
changed: [18.117.144.56]

TASK [On zookeeper1 instance create Zookeeper directory] ********************************************************************************
skipping: [18.117.144.56]
skipping: [3.140.201.79]
changed: [18.222.137.38]

TASK [On zookeeper1 instance setup confiuration for Zookeeper cluster] ******************************************************************
skipping: [18.117.144.56]
skipping: [3.140.201.79]
changed: [18.222.137.38]

TASK [On zookeeper2 instance create Zookeeper directory] ********************************************************************************
skipping: [18.222.137.38]
skipping: [3.140.201.79]
changed: [18.117.144.56]

TASK [On zookeeper1 instance setup confiuration for Zookeeper cluster] ******************************************************************
skipping: [18.222.137.38]
skipping: [3.140.201.79]
changed: [18.117.144.56]

TASK [On zookeeper3 instance create Zookeeper directory] ********************************************************************************
skipping: [18.222.137.38]
skipping: [18.117.144.56]
changed: [3.140.201.79]

TASK [On zookeeper3 instance setup confiuration for Zookeeper cluster] ******************************************************************
skipping: [18.222.137.38]
skipping: [18.117.144.56]
changed: [3.140.201.79]

TASK [Start Zookeeper server] ***********************************************************************************************************
changed: [18.222.137.38]
changed: [18.117.144.56]
changed: [3.140.201.79]

PLAY RECAP ******************************************************************************************************************************
18.117.144.56              : ok=5    changed=4    unreachable=0    failed=0    skipped=4    rescued=0    ignored=0
18.222.137.38              : ok=5    changed=4    unreachable=0    failed=0    skipped=4    rescued=0    ignored=0
3.140.201.79               : ok=5    changed=4    unreachable=0    failed=0    skipped=4    rescued=0    ignored=0
```
## Configure three node Kafka cluster through Ansible

You can use the same `kafka_cluster.yaml` file used in the section, [Deploy Cluster Automatically (AWS)](/learning-paths/server-and-cloud/kafka/kafka_aws.md). 

### Ansible Commands

Run the playbook using the  `ansible-playbook` command:
```console
ansible-playbook kafka_cluster.yaml -i /tmp/inventory
```
Answer `yes` when prompted for the SSH connection. 

Deployment may take a few minutes. 

The output should be similar to:
```output
root@ip-172-31-38-39:/home/ubuntu/kf# ansible-playbook kafka_cluster.yaml -i /tmp/inventory

PLAY [kafka1, kafka2, kafka3] ***********************************************************************************************************

TASK [Gathering Facts] ******************************************************************************************************************
The authenticity of host '3.19.64.64 (3.19.64.64)' can't be established.
ED25519 key fingerprint is SHA256:3qKhnEt0Js8LL67OAcIDOsTx8qQGRlQlsJ8ZZX9Fz1E.
This key is not known by any other names
The authenticity of host '3.133.93.119 (3.133.93.119)' can't be established.
ED25519 key fingerprint is SHA256:rI7qWf3fDGIkB+EqrPCa18maKeVck951WklPmeAQoXc.
This key is not known by any other names
The authenticity of host '18.222.0.36 (18.222.0.36)' can't be established.
ED25519 key fingerprint is SHA256:+Gv6OFuOLlaIIiQRtRccpmA5vA0wC7SqriwyudB8K34.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
ok: [3.19.64.64]
yes
ok: [3.133.93.119]
yes
ok: [18.222.0.36]

TASK [Update machines and install Java, Kafka] ******************************************************************************************
changed: [18.222.0.36]
changed: [3.133.93.119]
changed: [3.19.64.64]

TASK [On kafka1 instance create log directory] ******************************************************************************************
skipping: [3.133.93.119]
skipping: [18.222.0.36]
changed: [3.19.64.64]

TASK [On kafka1 instance update broker id] **********************************************************************************************
skipping: [3.133.93.119]
skipping: [18.222.0.36]
changed: [3.19.64.64]

TASK [On kafka1 instance uncomment listeners] *******************************************************************************************
skipping: [3.133.93.119]
skipping: [18.222.0.36]
changed: [3.19.64.64]

TASK [On kafka1 instance update zookeeper.connect] **************************************************************************************
skipping: [3.133.93.119]
skipping: [18.222.0.36]
changed: [3.19.64.64]

TASK [On kafka2 instance create log directory] ******************************************************************************************
skipping: [3.19.64.64]
skipping: [18.222.0.36]
changed: [3.133.93.119]

TASK [On kafka2 instance update broker id] **********************************************************************************************
skipping: [3.19.64.64]
skipping: [18.222.0.36]
changed: [3.133.93.119]

TASK [On kafka2 instance uncomment listeners] *******************************************************************************************
skipping: [3.19.64.64]
skipping: [18.222.0.36]
changed: [3.133.93.119]

TASK [On kafka2 instance update zookeeper.connect] **************************************************************************************
skipping: [3.19.64.64]
skipping: [18.222.0.36]
changed: [3.133.93.119]

TASK [On kafka3 instance create log directory] ******************************************************************************************
skipping: [3.19.64.64]
skipping: [3.133.93.119]
changed: [18.222.0.36]

TASK [On kafka3 instance update broker id] **********************************************************************************************
skipping: [3.19.64.64]
skipping: [3.133.93.119]
changed: [18.222.0.36]

TASK [On kafka3 instance uncomment listeners] *******************************************************************************************
skipping: [3.19.64.64]
skipping: [3.133.93.119]
changed: [18.222.0.36]

TASK [On kafka3 instance update zookeeper.connect] **************************************************************************************
skipping: [3.19.64.64]
skipping: [3.133.93.119]
changed: [18.222.0.36]

TASK [Start kafka_server] ***************************************************************************************************************
```
Kafka servers are started on all the three kafka instaces. Make sure this terminal is not closed. Open a new terminal on the same machine and configure the client through Ansible.

After successfully setting up a 3 node Kafka cluster, we can verify it works by creating a topic and storing the events. Follow the steps below to create a topic, write some events into the topic, and then read the events.

## Configure client through Ansible

You can use the same `client.yaml` file used in the section, [Deploy Cluster Automatically (AWS)](/learning-paths/server-and-cloud/kafka/kafka_aws.md).

### Ansible Commands

Run the playbook using the `ansible-playbook` command:
```console
ansible-playbook client.yaml -i /tmp/inventory
```
Answer `yes` when prompted for the SSH connection. 

Deployment may take a few minutes. 

The output should be similar to:
```output
root@ip-172-31-38-39:/home/ubuntu/kf# ansible-playbook client.yaml -i /tmp/inventory

PLAY [client] ***************************************************************************************************************************

TASK [Gathering Facts] ******************************************************************************************************************
The authenticity of host '18.216.149.104 (18.216.149.104)' can't be established.
ED25519 key fingerprint is SHA256:fPpeusF+HmRq8+Rc3erkW7/Je6gTZy/c1iA0eF4Iguo.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
ok: [18.216.149.104]

TASK [Update machines and install Java, Kafka] ******************************************************************************************
changed: [18.216.149.104]

TASK [Create a topic] *******************************************************************************************************************
changed: [18.216.149.104]

PLAY RECAP ******************************************************************************************************************************
18.216.149.104             : ok=3    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```
## Describe the topic created:

To describe the topic created ssh on the client instance and run the following command.
```console
ssh ubuntu@client_ip
cd kafka_node/kafka_2.13-3.2.3
./bin/kafka-topics.sh --topic test-topic --bootstrap-server kf_1_ip:9092,kf_2_ip:9092,kf_3_ip:9092 --describe
```
Replace the `client_ip`, `kf_1_ip`, `kf_2_ip`, `kf_3_ip` with the IP of client, kafka1, kafka2 and kafka3 respectively generated in inventory file present at location `/tmp/inventory`.

The output should be similar to:
```output
ubuntu@ip-172-31-31-117:~/kafka_node/kafka_2.13-3.2.3$ ./bin/kafka-topics.sh --topic test-topic --bootstrap-server 3.19.64.64:9092,3.133.93.119:9092,18.222.0.36:9092 --describe
Topic: test-topic       TopicId: J_G3hNvpTjSXoQ4-gyPKYg PartitionCount: 64      ReplicationFactor: 3    Configs: segment.bytes=1073741824
        Topic: test-topic       Partition: 0    Leader: 2       Replicas: 2,3,1 Isr: 2,3,1
        Topic: test-topic       Partition: 1    Leader: 3       Replicas: 3,1,2 Isr: 3,1,2
        Topic: test-topic       Partition: 2    Leader: 1       Replicas: 1,2,3 Isr: 1,2,3
        Topic: test-topic       Partition: 3    Leader: 2       Replicas: 2,1,3 Isr: 2,1,3
        Topic: test-topic       Partition: 4    Leader: 3       Replicas: 3,2,1 Isr: 3,2,1
        Topic: test-topic       Partition: 5    Leader: 1       Replicas: 1,3,2 Isr: 1,3,2
        Topic: test-topic       Partition: 6    Leader: 2       Replicas: 2,3,1 Isr: 2,3,1
        Topic: test-topic       Partition: 7    Leader: 3       Replicas: 3,1,2 Isr: 3,1,2
        Topic: test-topic       Partition: 8    Leader: 1       Replicas: 1,2,3 Isr: 1,2,3
        Topic: test-topic       Partition: 9    Leader: 2       Replicas: 2,1,3 Isr: 2,1,3
        Topic: test-topic       Partition: 10   Leader: 3       Replicas: 3,2,1 Isr: 3,2,1
        Topic: test-topic       Partition: 11   Leader: 1       Replicas: 1,3,2 Isr: 1,3,2
        Topic: test-topic       Partition: 12   Leader: 2       Replicas: 2,3,1 Isr: 2,3,1
        Topic: test-topic       Partition: 13   Leader: 3       Replicas: 3,1,2 Isr: 3,1,2
        Topic: test-topic       Partition: 14   Leader: 1       Replicas: 1,2,3 Isr: 1,2,3
        Topic: test-topic       Partition: 15   Leader: 2       Replicas: 2,1,3 Isr: 2,1,3
        Topic: test-topic       Partition: 16   Leader: 3       Replicas: 3,2,1 Isr: 3,2,1
        Topic: test-topic       Partition: 17   Leader: 1       Replicas: 1,3,2 Isr: 1,3,2
        Topic: test-topic       Partition: 18   Leader: 2       Replicas: 2,3,1 Isr: 2,3,1
        Topic: test-topic       Partition: 19   Leader: 3       Replicas: 3,1,2 Isr: 3,1,2
        Topic: test-topic       Partition: 20   Leader: 1       Replicas: 1,2,3 Isr: 1,2,3
        Topic: test-topic       Partition: 21   Leader: 2       Replicas: 2,1,3 Isr: 2,1,3
        Topic: test-topic       Partition: 22   Leader: 3       Replicas: 3,2,1 Isr: 3,2,1
        Topic: test-topic       Partition: 23   Leader: 1       Replicas: 1,3,2 Isr: 1,3,2
        Topic: test-topic       Partition: 24   Leader: 2       Replicas: 2,3,1 Isr: 2,3,1
        Topic: test-topic       Partition: 25   Leader: 3       Replicas: 3,1,2 Isr: 3,1,2
        Topic: test-topic       Partition: 26   Leader: 1       Replicas: 1,2,3 Isr: 1,2,3
        Topic: test-topic       Partition: 27   Leader: 2       Replicas: 2,1,3 Isr: 2,1,3
        Topic: test-topic       Partition: 28   Leader: 3       Replicas: 3,2,1 Isr: 3,2,1
        Topic: test-topic       Partition: 29   Leader: 1       Replicas: 1,3,2 Isr: 1,3,2
        Topic: test-topic       Partition: 30   Leader: 2       Replicas: 2,3,1 Isr: 2,3,1
        Topic: test-topic       Partition: 31   Leader: 3       Replicas: 3,1,2 Isr: 3,1,2
        Topic: test-topic       Partition: 32   Leader: 1       Replicas: 1,2,3 Isr: 1,2,3
        Topic: test-topic       Partition: 33   Leader: 2       Replicas: 2,1,3 Isr: 2,1,3
        Topic: test-topic       Partition: 34   Leader: 3       Replicas: 3,2,1 Isr: 3,2,1
        Topic: test-topic       Partition: 35   Leader: 1       Replicas: 1,3,2 Isr: 1,3,2
        Topic: test-topic       Partition: 36   Leader: 2       Replicas: 2,3,1 Isr: 2,3,1
        Topic: test-topic       Partition: 37   Leader: 3       Replicas: 3,1,2 Isr: 3,1,2
        Topic: test-topic       Partition: 38   Leader: 1       Replicas: 1,2,3 Isr: 1,2,3
        Topic: test-topic       Partition: 39   Leader: 2       Replicas: 2,1,3 Isr: 2,1,3
        Topic: test-topic       Partition: 40   Leader: 3       Replicas: 3,2,1 Isr: 3,2,1
        Topic: test-topic       Partition: 41   Leader: 1       Replicas: 1,3,2 Isr: 1,3,2
        Topic: test-topic       Partition: 42   Leader: 2       Replicas: 2,3,1 Isr: 2,3,1
        Topic: test-topic       Partition: 43   Leader: 3       Replicas: 3,1,2 Isr: 3,1,2
        Topic: test-topic       Partition: 44   Leader: 1       Replicas: 1,2,3 Isr: 1,2,3
        Topic: test-topic       Partition: 45   Leader: 2       Replicas: 2,1,3 Isr: 2,1,3
        Topic: test-topic       Partition: 46   Leader: 3       Replicas: 3,2,1 Isr: 3,2,1
        Topic: test-topic       Partition: 47   Leader: 1       Replicas: 1,3,2 Isr: 1,3,2
        Topic: test-topic       Partition: 48   Leader: 2       Replicas: 2,3,1 Isr: 2,3,1
        Topic: test-topic       Partition: 49   Leader: 3       Replicas: 3,1,2 Isr: 3,1,2
        Topic: test-topic       Partition: 50   Leader: 1       Replicas: 1,2,3 Isr: 1,2,3
        Topic: test-topic       Partition: 51   Leader: 2       Replicas: 2,1,3 Isr: 2,1,3
        Topic: test-topic       Partition: 52   Leader: 3       Replicas: 3,2,1 Isr: 3,2,1
        Topic: test-topic       Partition: 53   Leader: 1       Replicas: 1,3,2 Isr: 1,3,2
        Topic: test-topic       Partition: 54   Leader: 2       Replicas: 2,3,1 Isr: 2,3,1
        Topic: test-topic       Partition: 55   Leader: 3       Replicas: 3,1,2 Isr: 3,1,2
        Topic: test-topic       Partition: 56   Leader: 1       Replicas: 1,2,3 Isr: 1,2,3
        Topic: test-topic       Partition: 57   Leader: 2       Replicas: 2,1,3 Isr: 2,1,3
        Topic: test-topic       Partition: 58   Leader: 3       Replicas: 3,2,1 Isr: 3,2,1
        Topic: test-topic       Partition: 59   Leader: 1       Replicas: 1,3,2 Isr: 1,3,2
        Topic: test-topic       Partition: 60   Leader: 2       Replicas: 2,3,1 Isr: 2,3,1
        Topic: test-topic       Partition: 61   Leader: 3       Replicas: 3,1,2 Isr: 3,1,2
        Topic: test-topic       Partition: 62   Leader: 1       Replicas: 1,2,3 Isr: 1,2,3
        Topic: test-topic       Partition: 63   Leader: 2       Replicas: 2,1,3 Isr: 2,1,3
```
## Run the producer client to write events into the created topic:

Run the following command in the same client terminal and folder where the topic was created and replace `kf_1_ip`, `kf_2_ip`, `kf_3_ip` with the IP of kafka1, kafka2 and kafka3 respectively generated in inventory file present at location `/tmp/inventory`.
```console
./bin/kafka-console-producer.sh --topic test-topic --bootstrap-server kf_1_ip:9092,kf_2_ip:9092,kf_3_ip:9092
```
The output should be similar to:
```output
ubuntu@ip-172-31-31-117:~/kafka_node/kafka_2.13-3.2.3$ ./bin/kafka-console-producer.sh --topic test-topic --bootstrap-server 3.19.64.64:9092,3.133.93.119:9092,18.222.0.36:9092
>This is the first message written on producer
>
```
## Run the consumer client to read all the events created:

Open a new terminal on the client machine and run the consumer client using the following command.
```console
ssh ubuntu@client_ip
cd kafka_node/kafka_2.13-3.2.3
./bin/kafka-console-consumer.sh --topic test-topic --bootstrap-server kf_1_ip:9092,kf_2_ip:9092,kf_3_ip:9092
```
Replace the `client_ip`, `kf_1_ip`, `kf_2_ip`, `kf_3_ip` with the IP of client, kafka1, kafka2 and kafka3 respectively generated in inventory file present at location `/tmp/inventory`.

The output should be similar to:
```output
ubuntu@ip-172-31-31-117:~/kafka_node/kafka_2.13-3.2.3$ ./bin/kafka-console-consumer.sh --topic test-topic --bootstrap-server 3.19.64.64:9092,3.133.93.119:9092,18.222.0.36:9092
This is the first message written on producer
```
Write a message into the producer client terminal and press enter. You should see the same message appear on consumer client terminal. 

