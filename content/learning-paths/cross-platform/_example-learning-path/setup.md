---
# User change
title: "Setup"

weight: 3 # 1 is first, 2 is second, etc.

# Do not modify these elements
layout: "learningpathall"
---

## Setup a computer

There are multiple ways to setup a computer for Learning Path creation. 

Three tools are mandatory:
- A text editor to create and modify markdown files
- [Hugo](https://gohugo.io/) static site generator to review content 
- Git to access the repository and submit contributions

## Fork the repository

You will need a GitHub account to continue. [Sign up](https://github.com/signup) for a new account or use your existing account.

No matter which computer option you choose, the first step is to fork the GitHub repository into your own GitHub account. 

Go to the [repository in GitHub](https://github.com/ArmDeveloperEcosystem/arm-learning-paths) and click `Fork` in the top right area. Follow the prompts to create a new fork. This provides your own copy for you to make changes without impacting the main repository. 

## Setup a development machine 

Three possible ways to setup a development machine are covered below. 

Select **one** that works for you. Please share other ways that work for you!

- A [local computer (Linux, macOS, or Windows)](#setup-a-local-computer)
- A [Gitpod instance](#setup-gitpod) with tools pre-installed (easiest to setup and manage)
- A [remote Linux server](#setup-a-remote-linux-server) via SSH (on your local network or from a Cloud Service Provider)

## Setup a local computer

The first option is to install the required tools on your local computer. 

1. Install a text editor

Any text editor can be used to create and modify the project markdown files. Many developers use [Visual Studio Code](https://code.visualstudio.com/), but any text editor can be used. 

If you already have a text editor installed you can proceed to the next step.

2. Install Git

[Install Git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git) using the documentation for your operating system. 

3. Clone the project repository

Clone your fork of the repository to your local machine (substitute your github account name in the command below). You can copy the path by visiting your fork in GitHub and clicking the Code button.

```bash
git clone https://github.com/<your-github-account-name>/arm-learning-paths
```

You have have a local copy of the repository on your computer. 

4. Install Hugo

Install Hugo so you can see how your changes look before submitting them for review. 

This enables you to run a local web server and see the content just how it will appear when published. 

The easiest way to download Hugo on Linux (Debian/Ubuntu) is using the package manager.

```bash
sudo apt install hugo
```

You can also download Hugo for other operating systems from the [releases page](https://github.com/gohugoio/hugo/releases). Hugo works on all major operating systems and architectures. 

Any recent version of Hugo will work. The **extended version of Hugo is not needed**. 

For even more ways to install Hugo [check the documentation](https://gohugo.io/getting-started/installing).

Check Hugo is installed correctly and check the version by running this command:

```bash
hugo version
```
You are now ready to start developing content. The content creation process consists of editing the markdown files for Learning Paths and viewing the changes on your computer using Hugo.

Run hugo to launch a development version of website on your machine.

```bash
hugo server
```

Hugo server will print a message to connect to port 1313

```console
Web Server is available at //localhost:1313/ (bind address 127.0.0.1)
```

Open a browser and go to [http://localhost:1313](http://localhost:1313)

You are now ready to edit markdown files and view the results on the local Hugo server. 

## Setup Gitpod

The second option is to use Gitpod as a development environment.

[Gitpod](https://www.gitpod.io/) is a cloud development environment (CDE) which makes it easy to create software from any computer. 

Instead of installing tools on your local computer, you can create and modify content directly in Gitpod. The repository is configured to initialize a Gitpod instance with all tools/software you need to start contributing right away. 

Visit your fork of the GitHub project in your browser. 

Install the [Gitpod Chrome Extension](https://chrome.google.com/webstore/detail/gitpod-always-ready-to-co/dodmmooeoklaejobgleioelladacbeki) which installs a Gitpod button in your GitHub projects. 

If you no not want install the browser extension, you can also prefix the URL for your fork of the GitHub project with gitpod.io/# to open the project in Gitpod.

Either way, open the repository in Gitpod. The URL looks like: https://gitpod.io/#github.com/ArmDeveloperEcosystem/arm-learning-paths (replace with the path to your fork).

You can use your GitHub credentials (or select another authentication method) to login to Gitpod and use the Free plan which offers 500 credits (about 50 hours) per month. 

A new project will open in the browser. This is VS Code with the project loaded in your browser.

All of the required tools will be automatically installed by Gitpod during the environment setup. 

A view of the generated site is also opened automatically. The website view is a little hard to see because it's in a sub-window of VS Code.
 
To move the website view into another browser tab, click on the top right corner. The tooltip for this click is “open in browser”.

This gives you a private version of the content in a new tab to review. 

You are now ready to edit markdown files and view the results in the Gitpod environment.

## Setup a remote Linux server

The third option for Learning Path development is to use a remote Linux server, such as an AWS EC2 instance or other cloud virtual machine. The remove Linux server can also be on your local network.

Find the IP address or DNS name of the remote server and make sure you can SSH to the computer. 

Install the same tools as you would on the local computer setup above: 

- Text editor
- Git
- Hugo 

To access the Hugo server on the remote machine, use ssh port forwarding (use port 1313):

```bash
ssh -L 1313:localhost:1313 user@ip-address
```

Clone your fork of the repository to the remote Linux server (substitute your github account name in the command below):
```bash
git clone https://github.com/<your-github-account-name>/arm-learning-paths
```

There are a number of good options for text editors if you don't have a full Linux desktop available on the remote server.

To use Visual Studio Code in the browser on your remote Linux server check the install information for [OpenVSCode Server](/install-tools/openvscode-server/) and [VS Code Tunnels](/install-tools/vscode-tunnels/)

Run hugo to launch a development version of website on your machine.

```bash
hugo server
```

Hugo server will print a message to connect to port 1313

```console
Web Server is available at //localhost:1313/ (bind address 127.0.0.1)
```

Open a browser and go to [http://localhost:1313](http://localhost:1313)

You are now ready to edit markdown files and view the results on the Hugo server. 

The next section will cover how to create and format Learning Paths. 
