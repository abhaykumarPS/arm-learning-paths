---
# User change
title: "Deploy Cluster Automatically (GCP)"

weight: 6 # 1 is first, 2 is second, etc.

# Do not modify these elements
layout: "learningpathall"
---

##  Deploy Cluster Automatically 

You can deploy three node Zookeeper cluster, three node Kafka cluster and one client on Google Cloud using Terraform and Ansible. 
 
If you are new to Terraform, you should look at [Automate GCP instance creation using Terraform](/learning-paths/server-and-cloud/gcp/terraform/) before starting this Learning Path.

## Before you begin

You should have the prerequisite tools installed before starting the Learning Path. 

Any computer which has the required tools installed can be used for this section. The computer can be your desktop or laptop computer or a virtual machine with the required tools. 

You will need a [Google Cloud account](https://console.cloud.google.com/?hl=en-au) to complete this Learning Path. Create an account if you don't have one.

Before you begin you will also need:
- Login to the Google Cloud CLI 
- An SSH key pair

The instructions to login to the Google Cloud CLI and create the keys are below.

## Generate an SSH key pair

Generate an SSH key pair (public key, private key) using `ssh-keygen` to use for Arm VMs access. To generate the key pair, follow this [guide](/install-guides/ssh#ssh-keys).

{{% notice Note %}}
If you already have an SSH key pair present in the `~/.ssh` directory, you can skip this step.
{{% /notice %}}


## Acquire GCP Access Credentials

The installation of Terraform on your Desktop/Laptop needs to communicate with GCP. Thus, Terraform needs to be authenticated.

To obtain GCP user credentials, follow this [guide](/install-guides/gcp_login).

## Create seven GCP EC2 instance using Terraform

Using a text editor, save the code below to in a file called `main.tf`

Scroll down to see the information you need to change in `main.tf`
```console
provider "google" {
  project = "project_id"
  region = "us-central1"
  zone = "us-central1-a"
}

resource "google_compute_instance" "KAFKA_TEST" {
  name         = "kafkatest-${count.index+1}"
  count        = "7"
  machine_type = "t2a-standard-1"

  boot_disk {
    initialize_params {
      image = "ubuntu-os-cloud/ubuntu-2204-lts-arm64"
    }
  }

  network_interface {
    network = "default"
    access_config {
      // Ephemeral public IP
    }
  }
  metadata = {
     ssh-keys = "ubuntu:${file("~/.ssh/id_rsa.pub")}"
  }
}

resource "google_compute_firewall" "rules" {
  project     = "project_id"
  name        = "my-firewall-rule"
  network     = "default"
  description = "Open ssh connection and kafka port"
  source_ranges = ["0.0.0.0/0"]
  
  allow {
    protocol = "icmp"
  }

  allow {
    protocol  = "tcp"
    ports     = ["22", "2181", "3888", "2888", "9092"]
  }
}

resource "google_compute_network" "default" {
  name = "test-network"
}
resource "local_file" "inventory" {
    depends_on=[google_compute_instance.KAFKA_TEST]
    filename = "/tmp/inventory"
    content = <<EOF
[zookeeper1]
${google_compute_instance.KAFKA_TEST[0].network_interface.0.access_config.0.nat_ip}
[zookeeper2]
${google_compute_instance.KAFKA_TEST[1].network_interface.0.access_config.0.nat_ip}
[zookeeper3]
${google_compute_instance.KAFKA_TEST[2].network_interface.0.access_config.0.nat_ip}
[kafka1]
${google_compute_instance.KAFKA_TEST[3].network_interface.0.access_config.0.nat_ip}
[kafka2]
${google_compute_instance.KAFKA_TEST[4].network_interface.0.access_config.0.nat_ip}
[kafka3]
${google_compute_instance.KAFKA_TEST[5].network_interface.0.access_config.0.nat_ip}
[client]
${google_compute_instance.KAFKA_TEST[6].network_interface.0.access_config.0.nat_ip}

[all:vars]
ansible_connection=ssh
ansible_user=ubuntu
zk_1_ip=${google_compute_instance.KAFKA_TEST[0].network_interface.0.access_config.0.nat_ip}
zk_2_ip=${google_compute_instance.KAFKA_TEST[1].network_interface.0.access_config.0.nat_ip}
zk_3_ip=${google_compute_instance.KAFKA_TEST[2].network_interface.0.access_config.0.nat_ip}
kf_1_ip=${google_compute_instance.KAFKA_TEST[3].network_interface.0.access_config.0.nat_ip}
kf_2_ip=${google_compute_instance.KAFKA_TEST[4].network_interface.0.access_config.0.nat_ip}
kf_3_ip=${google_compute_instance.KAFKA_TEST[5].network_interface.0.access_config.0.nat_ip}
                EOF
}

```

In the `provider` and `google_compute_firewall` sections, update the `project_id` with your value.

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

You can use the same `zookeeper_cluster.yaml` file used in the section, [Deploy Cluster Automatically (AWS)](https://github.com/abhisheknishantpuresoftware/arm-learning-paths/blob/Testing/learning-paths/server-and-cloud/kafka/kafka_aws.md#configure-three-node-zookeeper-cluster-through-ansible).

### Ansible Commands

Run the playbook using the  `ansible-playbook` command:
```console
ansible-playbook zookeeper_cluster.yaml -i /tmp/inventory
```
Answer `yes` when prompted for the SSH connection. 

Deployment may take a few minutes. 

The output should be similar to the output attached in the section, [Deploy Cluster Automatically (AWS)](https://github.com/abhisheknishantpuresoftware/arm-learning-paths/blob/Testing/learning-paths/server-and-cloud/kafka/kafka_aws.md#configure-three-node-zookeeper-cluster-through-ansible).

## Configure three node Kafka cluster through Ansible

You can use the same `kafka_cluster.yaml` file used in the section, [Deploy Cluster Automatically (AWS)](https://github.com/abhisheknishantpuresoftware/arm-learning-paths/blob/Testing/learning-paths/server-and-cloud/kafka/kafka_aws.md#configure-three-node-zookeeper-cluster-through-ansible). 

### Ansible Commands

Run the playbook using the  `ansible-playbook` command:
```console
ansible-playbook kafka_cluster.yaml -i /tmp/inventory
```
Answer `yes` when prompted for the SSH connection. 

Deployment may take a few minutes. 

The output should be similar to the output attached in the section, [Deploy Cluster Automatically (AWS)](https://github.com/abhisheknishantpuresoftware/arm-learning-paths/blob/Testing/learning-paths/server-and-cloud/kafka/kafka_aws.md#configure-three-node-zookeeper-cluster-through-ansible).

Kafka servers are started on all the three kafka instaces. Make sure this terminal is not closed. Open a new terminal on the same machine and configure the client through Ansible.

After successfully setting up a 3 node Kafka cluster, we can verify it works by creating a topic and storing the events. Follow the steps below to create a topic, write some events into the topic, and then read the events.

## Configure client through Ansible

You can use the same `client.yaml` file used in the section, [Deploy Cluster Automatically (AWS)](https://github.com/abhisheknishantpuresoftware/arm-learning-paths/blob/Testing/learning-paths/server-and-cloud/kafka/kafka_aws.md#configure-three-node-zookeeper-cluster-through-ansible).

### Ansible Commands

Run the playbook using the `ansible-playbook` command:
```console
ansible-playbook client.yaml -i /tmp/inventory
```
Answer `yes` when prompted for the SSH connection. 

Deployment may take a few minutes. 

The output should be similar to the output attached in the section, [Deploy Cluster Automatically (AWS)](https://github.com/abhisheknishantpuresoftware/arm-learning-paths/blob/Testing/learning-paths/server-and-cloud/kafka/kafka_aws.md#configure-three-node-zookeeper-cluster-through-ansible).

## Describe the topic created:

To describe the topic created ssh on the client instance and run the following command.
```console
ssh ubuntu@client_ip
cd kafka_node/kafka_2.13-3.2.3
./bin/kafka-topics.sh --topic test-topic --bootstrap-server kf_1_ip:9092,kf_2_ip:9092,kf_3_ip:9092 --describe
```
Replace the `client_ip`, `kf_1_ip`, `kf_2_ip`, `kf_3_ip` with the IP of client, kafka1, kafka2 and kafka3 respectively generated in inventory file present at location `/tmp/inventory`.

The output should be similar to the output attached in the section, [Deploy Cluster Automatically (AWS)](https://github.com/abhisheknishantpuresoftware/arm-learning-paths/blob/Testing/learning-paths/server-and-cloud/kafka/kafka_aws.md#configure-three-node-zookeeper-cluster-through-ansible).

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
