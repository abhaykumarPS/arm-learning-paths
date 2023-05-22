---
# User change
title: "Deploy a single node of Spark"

weight: 2 # 1 is first, 2 is second, etc.

# Do not modify these elements
layout: "learningpathall"
---

##  Deploy a single node of Spark 
Apache Spark is an open-source, distributed processing system used for big data workloads. It utilizes in-memory caching and optimized query execution for fast queries against data of any size. Simply we can say, Spark is a fast and general engine for large-scale data processing.

You can deploy Spark on AWS Graviton processors using Terraform. 
In this topic, you will deploy Spark on a single AWS EC2 instance. 
If you are new to Terraform, you should look at [Automate AWS EC2 instance creation using Terraform](/learning-paths/server-and-cloud/aws/terraform/) before starting this Learning Path.

## Before you begin

You should have the prerequisite tools installed before starting the Learning Path. 

Any computer which has the required tools installed can be used for this section. The computer can be your desktop or laptop computer or a virtual machine with the required tools. 

You will need an [AWS account](https://portal.aws.amazon.com/billing/signup?nc2=h_ct&src=default&redirect_url=https%3A%2F%2Faws.amazon.com%2Fregistration-confirmation#/start) to complete this Learning Path. Create an account if you don't have one.

Before you begin you will also need:
- An AWS access key ID and secret access key. 
- An SSH key pair

The instructions to create the keys are below.

### Acquire AWS Access Credentials 

The installation of Terraform on your desktop or laptop needs to communicate with AWS. Thus, Terraform needs to be able to authenticate with AWS.

To generate and configure the Access key ID and Secret access key, follow this [documentation](/install-guides/aws_access_keys).
### Generate an SSH key-pair

Generate the SSH key-pair (public key, private key) using `ssh-keygen` to use for AWS EC2 access. To generate the key-pair, follow this [guide](/install-guides/ssh#ssh-keys).

{{% notice Note %}}
If you already have an SSH key-pair present in the `~/.ssh` directory, you can skip this step.
{{% /notice %}}

## Create an AWS EC2 instance using Terraform

Using a text editor, save the code below to in a file called `main.tf`

Scroll down to see the information you need to change in `main.tf`

```console

// instance creation
provider "aws" {
  region = "us-east-2"
}
resource "aws_instance" "Spark_TEST" {
  ami           = "ami-0ca2eafa23bc3dd01"
  instance_type = "t4g.small"
  security_groups= [aws_security_group.Terraformsecurity.name]
  key_name = aws_key_pair.deployer.key_name
 
  tags = {
    Name = "Spark_TEST"
  }
}
resource "aws_default_vpc" "main" {
  tags = {
    Name = "main"
  }
}
resource "aws_security_group" "Terraformsecurity" {
  name        = "Terraformsecurity"
  description = "Allow TLS inbound traffic"
  vpc_id      = aws_default_vpc.main.id

ingress {
    description      = "TLS from VPC"
    from_port        = 8888
    to_port          = 8888
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
}
ingress {
    description      = "TLS from VPC"
    from_port        = 80
    to_port          = 80
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
    Name = "Terraformsecurity"
  }
 }
output "Master_public_IP" {
  value = [aws_instance.Spark_TEST.public_ip]
}
 resource "aws_key_pair" "deployer" {
         key_name   = "id_rsa"
         public_key = file("~/.ssh/id_rsa.pub")
  }
// Generate inventory file
resource "local_file" "inventory" {
    depends_on= [aws_instance.Spark_TEST]
    filename = "/tmp/inventory"
    content = <<EOF
          [db_master]
          ${aws_instance.Spark_TEST.public_ip}         
          [all:vars]
          ansible_connection=ssh
          ansible_user=ubuntu
          EOF
}
```

Make the changes listed below in `main.tf` to match your account settings.

1. In the `provider` section, update value to use your preferred AWS region.

2. (optional) In the `aws_instance` section, change the ami value to your preferred Linux distribution. The AMI ID for Ubuntu 22.04 on Arm in the us-east-2 region is `ami-0f9bd9098aca2d42b ` No change is needed if you want to use Ubuntu AMI in us-east-1. The AMI ID values are region specific and need to be changed if you use another AWS region. Use the AWS EC2 console to find an AMI ID or refer to [Find a Linux AMI](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/finding-an-ami.html).

{{% notice Note %}}
The instance type is t4g.small. This is an Arm-based instance and requires an Arm Linux distribution.
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

The public IP address will be different, but the output should be similar to:

```output
Apply complete! Resources: 5 added, 0 changed, 0 destroyed.

Outputs:

Master_public_IP = [
  "13.58.17.252",
]

```
## Configure Spark manually
**SSH to the instance**

Login to the deployed instance, using SSH to the public IP of the AWS EC2 instance.

``` console
ssh ubuntu@Master_public_IP
```
**Installation of required dependencies on AWS EC2 instance**

For deploying Spark on AWS graviton2, we need to to install below tools and dependencies on our EC2 instance:
```console
sudo apt-get update
sudo apt-get upgrade -y
sudo apt install python3-pip -y
pip3 install jupyter 
sudo apt-get install default-jre -y 
sudo apt-get install scala -y
pip3 install py4j
pip3 install findspark
pip3 install pyspark
 ```
 **Install and extract Spark binary**
 
 We can install Spark binary on our EC2 instance using below commands:
 ```console
 sudo wget https://archive.apache.org/dist/spark/spark-3.2.2/spark-3.2.2-bin-hadoop2.7.tgz
 sudo tar -zxvf spark-3.2.2-bin-hadoop2.7.tgz
```
**Config Jupyter notebook**

Jupyter comes with Anaconda, but we will need to configure it in order to use it through EC2 and connect with SSH. Generate a configuration file for Jupyter using:
```console
 jupyter notebook --generate-config
```
**Create Certifications**

We can also create certifications for our connections in the form of .pem files. 
Perform the following:
```console
 mkdir cert/
 cd cert/
 sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout mycert.pem -out mycert.pem
```
{{% notice Note %}}
You’ll get asked some general questions after running that last line. Just fill them out with some general information.
{{% /notice %}}

**Change the file owner or directory permissions**

To give read, write and execute permissions for everyone:
```console
sudo chown $USER:$USER /home/ubuntu/cert/mycert.pem
sudo chmod 777 /home/ubuntu/cert/mycert.pem  
```
**Edit Configuration File**

Next, we need to finish editing the Jupyter Configuration file we created earlier. Change directory to:

```console
cd ~/.jupyter/
```
Then we will use any editor to edit the file. Here we are uisng visual editor(vi).Type:
```console
vi jupyter_notebook_config.py
```
You should see a bunch of commented Python code, this is where you can either uncomment lines or add in your own (things such as adding password protection are an option here). We will keep things simple.

Press i on your keyboard to activate -INSERT-. Then at the top of the file type:
```console
c.NotebookApp.certfile = u'/home/ubuntu/cert/mycert.pem'
c.NotebookApp.ip = '*' 
c.NotebookApp.port = 8888 
c.NotebookApp.open_browser = False
 ```
Once you’ve typed/pasted this code in your config file, save your file and quit the editor.

Lastly, create a dummy file(.json) for checking whether Spark is working or not:
 
 ```console
{"Country":"Singapore"}
{"Country":"India","Capital":"New Delhi"}
{"Country":"UK","Capital":"London","Population":"78M"}
```
Now follow [this](/learning-paths/server-and-cloud/spark/spark_deployment#check-that-jupyter-notebook-is-working-with-spark) to Check that Jupyter Notebook is working with spark.

{{% notice Note %}} Follow the above mentioned steps for configuring Spark manually or follow the below ansible steps for configuration of Spark in AWS EC2 instance. {{% /notice %}}

## Configure Spark by Ansible
Using a text editor, save the code below to a file called `spark.yaml`. It will install the Spark and the required dependencies. This is the YAML file for the Ansible playbook.
```console
---
- name: spark config
  hosts: all
  become: true
  become_user: root
  become_method: sudo

  tasks:
    - name: Update the Machine & Install Dependencies
      shell: sudo apt-get update
    - name: Update apt repo and cache on all Debian/Ubuntu boxes
      apt:  upgrade=yes update_cache=yes force_apt_get=yes cache_valid_time=3600
      become: true
    - name: Install Python pip & Python package
      apt: name={{ item }} update_cache=true state=present force_apt_get=yes
      with_items:
      - python3-pip
      become: true
    - name: Install jupyter
      shell: pip3 install jupyter
    - name: Install Java and related Dependencies
      shell: sudo apt-get install -y default-jre default-jdk vim wget
    - name: Install scala
      shell: sudo apt-get install -y scala
    - name: Update the Machine & Install Dependencies
      shell: pip3 install py4j
    - name: Install openssl
      shell: sudo apt install openssl -y
    - name: Install spark
      shell: pip3 install findspark
    - name: Install and extract spark binary
      shell: |
              sudo wget https://archive.apache.org/dist/spark/spark-3.2.2/spark-3.2.2-bin-hadoop2.7.tgz
              sudo tar -zxvf spark-3.2.2-bin-hadoop2.7.tgz
    - name: configure jupyter notebook
      shell: |
              jupyter notebook --generate-config
    - name: Change file path and assigned priveledges
      shell: |
              sudo mv /root/.jupyter /home/ubuntu/
              sudo chown $USER:$USER /home/ubuntu/.jupyter/
              sudo chmod 777 /home/ubuntu/.jupyter/
    - name: Create directory and config Jupyter notebook
      shell: |
              mkdir cert
              cd cert
              sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout mycert.pem -out mycert.pem -subj "/C=GB/ST=London/L=London/O=Global Security/OU=IT Department/CN=example.com/E=abh@gmail.com"
              sudo chown $USER:$USER /home/ubuntu/cert/mycert.pem
              sudo chmod 777 /home/ubuntu/cert/mycert.pem  
    - name: Edit Configuration File
      lineinfile:
        path: /home/ubuntu/.jupyter/jupyter_notebook_config.py
        line: |
               c.NotebookApp.certfile = u'/home/ubuntu/cert/mycert.pem'
               c.NotebookApp.ip = '*'
               c.NotebookApp.port = 8888
               c.NotebookApp.open_browser = False
        insertafter: c = get_config()  #noqa
    - name: Creating a dummy .json file
      copy:
        dest: /home/ubuntu/test.json
        content: |
             {"Country":"Singapore"}
             {"Country":"India","Capital":"New Delhi"}
             {"Country":"UK","Capital":"london","Population":"78M"}

```
## Ansible Commands
Run the playbook using the `ansible-playbook` command:
```console
ansible-playbook spark.yaml -i /tmp/inventory
```
Answer `yes` when prompted for the SSH connection.

Deployment may take a few minutes.

The output should be similar to:
```output
root@ip-172-31-38-39:/home/ubuntu/spark# ansible-playbook spark.yaml -i /tmp/inventor

PLAY [spark config] ***********************************************************************************************************************

TASK [Gathering Facts] ********************************************************************************************************************
ok: [18.119.110.6]

TASK [Update the Machine & Install Dependencies] ******************************************************************************************
changed: [18.119.110.6]

TASK [Update apt repo and cache on all Debian/Ubuntu boxes] *******************************************************************************
ok: [18.119.110.6]

TASK [Install Python pip & Python package] ************************************************************************************************
changed: [18.119.110.6] => (item=python3-pip)

TASK [Install jupyter] ********************************************************************************************************************
changed: [18.119.110.6]

TASK [Install Java and related Dependencies] **********************************************************************************************
changed: [18.119.110.6]

TASK [Install scala] **********************************************************************************************************************
changed: [18.119.110.6]

TASK [Update the Machine & Install Dependencies] ******************************************************************************************
changed: [18.119.110.6]

TASK [Install openssl] ********************************************************************************************************************
changed: [18.119.110.6]

TASK [Install spark] **********************************************************************************************************************
changed: [18.119.110.6]

TASK [Install and extract spark binary] ***************************************************************************************************
changed: [18.119.110.6]

TASK [configure jupyter notebook] *********************************************************************************************************
changed: [18.119.110.6]

TASK [Change file path and assigned priveledges] ******************************************************************************************
changed: [18.119.110.6]

TASK [Create directory and config Jupyter notebook] ***************************************************************************************
changed: [18.119.110.6]

TASK [Edit Configuration File] ************************************************************************************************************
changed: [18.119.110.6]

TASK [Creating a json file] ***************************************************************************************************************
changed: [18.119.110.6]

PLAY RECAP ********************************************************************************************************************************
18.119.110.6               : ok=16   changed=14   unreachable=0    failed=0    skipped=0    rescued=0    ignored=0


```

## Check that Jupyter Notebook is working with spark
First, Log in to the node using SSH:

```console
ssh ubuntu@Public_ip_of_node
```

Now everything is set up to launch the Juptyer notebook with Spark! Run:
```console
jupyter notebook
```
The below interface will be shown in the terminal after applying `jupyter notebook` command:

![image](https://github.com/abhaykumarPS/arm-learning-paths/assets/92078754/b56893c2-6ce4-4b05-b00c-4aab2b845565)

You’ll see an output saying that a jupyter notebook is running at all IP addresses at port 8888. Go to a web browser and type Public DNS of Amazon EC2 instance followed by:8888. It should be in the form:

```console
https://<Public_DNS_Of_Your_Ec2_instance>:8888/?token=f5d44e8faa19f9b08bd3820e700666f176e75920cc5571a3

```
After putting that into the browser, an interface as shown below will be seen:

![spark1](https://github.com/abhaykumarPS/arm-learning-paths/assets/92078754/f714bd0e-ff11-4446-af87-61f7e58f8824)

Now, click on the new and then on Python3 (ipykernel) for accessing the jupyter notebook:

![spark2](https://github.com/abhaykumarPS/arm-learning-paths/assets/92078754/50fd6d68-77e9-4e83-b270-1c4efb7c6351)

Now, run the below code line by line in jupyter notebook:

```console
import findspark
findspark.init('/home/ubuntu/spark-3.2.2-bin-hadoop2.7') 
from pyspark.sql import SparkSession 
spark = SparkSession.builder.appName('myApp').getOrCreate() 
df = spark.read.json('test.json') 
df.show()
```
Below is the interface of jupyter notebook:

![image](https://github.com/abhaykumarPS/arm-learning-paths/assets/92078754/771d3b28-5246-425b-8b0a-665d65663b7e)

You have successfully deployed Spark on an AWS EC2 instance running Graviton processors.
### Clean up resources

Run `terraform destroy` to delete all resources created.

```console
terraform destroy
```

