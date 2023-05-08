---
# User change
title: "Deploy a single node of Spark"

weight: 2 # 1 is first, 2 is second, etc.

# Do not modify these elements
layout: "learningpathall"
---

##  Deploy a single node of Spark 

You can deploy Spark on AWS Graviton processors using Terraform. 

In this topic, you will deploy Sparek on a single AWS EC2 instance. 

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

Generate an SSH key-pair (public key, private key) using `ssh-keygen` to use for AWS EC2 access: 

```console
ssh-keygen -f aws_key -t rsa -b 2048 -P ""
```

You should now have your AWS access keys and your SSH keys in the current directory.

## Create an AWS EC2 instance using Terraform

Using a text editor, save the code below to in a file called `main.tf`

Scroll down to see the information you need to change in `main.tf`

```console

// instance creation
provider "aws" {
  region = "us-east-2"
}
resource "aws_instance" "Spark_TEST" {
  ami           = "ami-0f9bd9098aca2d42b"
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
    from_port        = 8080
    to_port          = 8080
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
}
ingress {
    description      = "TLS from VPC"
    from_port        = 9000
    to_port          = 9000
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
    from_port        = 0
    to_port          = 0
    protocol         = "-1"
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
    depends_on= [aws_instance.PSQL_TEST]
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

The public IP address will be different, but the output should be similar to:

```output
Apply complete! Resources: 5 added, 0 changed, 0 destroyed.

Outputs:

Master_public_IP = [
  "13.58.17.252",
]

```
### Configure Spark
**Login to the deployed instance, using SSH to the public IP of the AWS EC2 instance..**

``` console
ssh ubuntu@Master_public_IP
```
**Installation of required dependencies on AWS EC2 instance.**

For deploying spark on aws graviton2, we need to install below tools and depencies on our ec2 instance
```console
      sudo apt-get update 
      sudo apt install python3-pip 
      pip3 install jupyter 
      sudo apt-get install default-jre 
      sudo apt-get install scala 
      pip3 install py4j
      pip3 install findspark
      pip3 install pyspark
 ```
 **Install and extract spark binary:**
 
 We need to install spark binary on aur ec2 instance by followed below command .
 ```console
       sudo wget https://archive.apache.org/dist/spark/spark-3.2.2/spark-3.2.2-bin-hadoop2.7.tgz
       sudo tar -zxvf spark-3.2.2-bin-hadoop2.7.tgz
```
**config Jupyter notebook:**

Jupyter comes with Anaconda, but we will need to configure it in order to use it through EC2 and connect with SSH. Go ahead and generate a configuration file for Jupyter using:
```console
      jupyter notebook --generate-config
```
**Create Certifications:**

We can also create certifications for our connections in the form of .pem files. Perform the following:
```console
      cd cert/
      pip3 install findspark
      jupyter notebook --generate-config 
      sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout mycert.pem -out mycert.pem
```
You’ll get asked some general questions after running that last line. Just fill them out with some general information.

**Edit Configuration File:**

Next we need to finish editing the Jupyter Configuration file we created earlier. Change directory to:

``console
cd ~/.jupyter/
```
Then we will use visual editor (vi) to edit the file. Type:
```console
vi jupyter_notebook_config.py
``
You should see a bunch of commented Python code, this is where you can either uncomment lines or add in your own (things such as adding password protection are an option here). We will keep things simple.

Press i on your keyboard to activate -INSERT-. Then at the top of the file type:
```console
      c.NotebookApp.certfile = u'/home/ubuntu/certs/mycert.pem'
      c.NotebookApp.ip = '*' 
      c.NotebookApp.port = 8888 
      c.NotebookApp.open_browser = False
 ```
 Once you’ve typed/pasted this code in your config file, press Esc to stop inserting. Then type a colon : and then type wq to write and quit the editor.
 
**Check that Jupyter Notebook is working**

You should now have everything set up to launch Juptyer notebook with Spark! Run:
```console
jupyter notebook
```

You’ll see an output saying that a jupyter notebook is running at all ip addresses at port 8888. Go to your own web browser (Google Chrome suggested) and type in your Public DNS for your Amazon EC2 instance followed by :8888. It should be in the form:

```console
https://ec2-xx-xx-xxx-xxx.us-west-2.compute.amazonaws.com:8888
```
After putting that into your browser you’ll probably get a warning of an untrusted certificate, go ahead and click through that and connect anyway, you trust the site. (Hopefully, after all you are the one that made it!)

You should be able to see Jupyter Notebook running on you EC2 instance. Great!Use Crtl-C in your EC2 Ubuntu console to kill the Jupyter Notebook process. Clear the console with clear

Then as previously done, connect through your browser again to your instance’s Jupyter Notebook. Launch a new notebook and in a notebook cell type:

```console
from pyspark import SparkContext
sc = SparkContext()
```

If that works, you’re all done!
### Clean up resources

Run `terraform destroy` to delete all resources created.

```console
terraform destroy
```

Continue the Learning Path to create a multi-node PostgreSQL deployment. 
