# CACAO Terraform templates on the CLI

## Overview

This tutorial will guide you through the process of deploying Zero to JupyterHub with Kubernetes on an OpenStack VM using Terraform. 

We will be using a Terraform template for CACAO which has additional configurations.

By the end of this tutorial, you will have a running JupyterHub instance that can be accessed by your users.


## Prerequisites

* Basic understanding of Terraform, Kubernetes, and JupyterHub.
* An OpenStack environment with access to the API.
* Terraform installed on your local machine or VM server.
* `kubectl` and `helm` command-line tools installed on your local machine.
* An SSH key pair to access the VM server running Terraform.

## Installation

Terraform needs a dedicated resource to manage its deployments. 

[:simple-terraform: Terraform](https://developer.hashicorp.com/terraform/downloads){target=_blank} - install Terraform on your server

[:simple-ansible: Ansible](){target=_blank} - 

Optional: [:simple-visualstudiocode: VS Code Terraform Extension](https://marketplace.visualstudio.com/items?itemName=HashiCorp.terraform){target=_blank} - add VS Code extension

??? Tip "Mac OS X Installation"

    Instructions for Mac OS X installation

    If you're on OS X, you can use [brew](https://brew.sh/) to install with the following commands:

    ```bash
    # install terraform -- taken from https://learn.hashicorp.com/tutorials/terraform/install-cli
    brew tap hashicorp/tap && brew install hashicorp/tap/terraform

    # install ansible and jq (for processing terraform's statefile into an ansible inventory)
    brew install ansible jq
    ```

??? Tip "Linux Installation"

    Instructions for Ubuntu 22.04 installation

    ```bash
    wget -O- https://apt.releases.hashicorp.com/gpg | gpg --dearmor | sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg
    echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
    sudo apt update && sudo apt install terraform
    ```

    Install Ansible & J Query

    ```bash
    sudo apt-add-repository ppa:ansible/ansible
    sudo apt update & sudo apt install -y ansible jq
    ```


## CyVerse CACAO Browser UI

[CyVerse's CACAO (Cloud Automation & Continuous Analysis Orchestration)](https://cacao.jetstream-cloud.org/help){target=_blank}

### CyVerse CACAO CLI

### CACAO file structure

```css
terraform-project/
├── main.tf
├── inputs.tf
├── outputs.tf
├── output-ansible-inventory.tf
├── metadata.json
├── terraform.tfvars.example
└── ansible/
    ├── ansible.cfg
    ├── find_connection.yaml
    ├── playbook.yaml      
    └── requirements.yaml
```

**:simple-terraform: Main Configuration File (`main.tf`):** - contains the primary infrastructure resources and configurations for virtual machines, networks, and storage.

**:simple-terraform: Inputs File (`inputs.tf`):** - defines all the input variables used in the configuration. Declare variables with default values or leave them empty for required user input. Include descriptions for each variable to provide context.

**:simple-terraform: Outputs File (`outputs.tf`):** - defines the outputs Terraform displays after applying the Main and Variables configuration. Includes: IP addresses, DNS names, or information for resources.

**:simple-terraform: Provider Configuration File (`provider.tf`):** - 

**:simple-terraform: Modules and Reusable Configurations:** - create separate `.tf` files for reusable modules and configurations. Reuse across multiple projects or within the same project on multiple VMs.

### File Examples

#### :simple-terraform: `main.tf`

#### Start-up the Terraform Server

We need to run `terraform` from somewhere, this should be a permanent instance which is not pre-emptable (suggest a tiny VM on Jetstream2)

Before we start `terraform`, we need to create an SSH key to add to our newly deployed VMs.

??? Tip "Create SSH keys"

    To create an SSH key on an Ubuntu 22.04 terminal, you can follow these steps:

    **Step 1:** Open your terminal and type the following command to generate a new SSH key pair:

    ```bash
    ssh-keygen -t rsa -b 4096
    ```

    **Step 2:** When prompted, press "Enter" to save the key in the default location, or enter a different filename and location to save the key.

    Enter a passphrase to secure your key. This passphrase will be required to use the key later.

    Once the key is generated, you can add it to your SSH agent by running the following command:

    ```bash
    eval "$(ssh-agent -s)"
    ssh-add ~/.ssh/id_rsa
    ```

    **Step 3:** Copy the public key to your remote server by running the following command, replacing "user" and "server" with your username and server address:

    ```bash
    ssh-copy-id <user>@<server-ip>
    ```

    **create_ssh_script.sh:**

    ```bash
    #!/bin/bash

    echo "Generating SSH key pair..."
    ssh-keygen -t rsa -b 4096

    echo "Adding key to SSH agent..."
    eval "$(ssh-agent -s)"
    ssh-add ~/.ssh/id_rsa

    read -p "Enter your remote server username: " username
    read -p "Enter your remote server IP address: " server

    echo "Ready to copy your new public key to a remote server:"

    ssh-copy-id username@server

    echo "SSH key setup complete!"
    ```

    Save the script to a file, make it executable with the following command:

    ```bash
    chmod +x create_ssh_script.sh
    ```

    run it with the following command:

    ```bash
    ./create_ssh_script.sh
    ```

## Running Terraform

**Step 2** Clone the CACAO z2jh repository


```git clone https://gitlab.com/stack0/cacao-tf-jupyterhub.git```

** Step 3** Initialize Terraform in the repository

Change directory to the repo: `cd cacao-tf-jupyterhub`

Run 

```terraform init```

**Step 3** Source the OpenRC shell script

```source *-openrc.sh```

**Step 4** Start Terraform

 `terraform apply -auto-approve`

**Step 5** Suspend Terraform

Update your `terraform.tfvars` file by changing the `power_state` from "active" to "shelved_offloaded"

```yaml
### Set Power Status ###
power_state = "active"
# power_state = "shelved_offloaded"
```

re-run `terraform apply -auto-approve` to update the deployment 

**Step 6** Destroying a Terraform deployment

when done, with your cluster, run `terraform destroy -auto-approve` to take the cluster back down.

#### Editing Terraform Variables file (.tfvars)


??? Tip "Zero to JupyterHub with Kubernetes Deployment"

    Example Terraform variables (.tfvars) file for deploying [Zero to JupyterHub with K3s (Rancher)]()
        
    Based on the [Zero to JupyterHub with Kubernetes](https://z2jh.jupyter.org/en/stable/)

    ```yaml

    ### Set the floating_ip address for DNS redirections ###

    # jupyterhub_floating_ip="149.165.152.223"

    ### Set up the Security Groups (re=used from Horizon) ###
    
    security_groups = ["default","cacao-default","cacao-k8s"]

    ### Give the Deployment a Name (shorter is better) ###

    instance_name = "jh-prod"

    ### Horizon Account User ID and ACCESS-CI Project Account ###
    
    username = "${USER}"
    project = "BIO220085"

    # Horizon SSH key-pair
    keypair = "${USER}-default"

    ### Set the Number of Instances (Master + Workers) ###

    instance_count = 6

    # Storage Volume Size in GiB 
    jh_storage_size=1000

    # Storage Volume directory path
    jh_storage_mount_dir="/home/shared"

    # OS type
    image_name = "Featured-Ubuntu22"

    # Worker VM Flavor
    flavor = "m3.medium"

    # Master VM Flavor (m3.medium or larger is recommended)
    flavor_master = "m3.medium"

    ### Run JupyterHub on Master Node ###
    
    do_jh_singleuser_exclude_master="true"
    
    # Jetstream2 Region
    region = "IU"

    # VM image (root disk) sizes
    root_storage_size = 100
    root_storage_type = "volume"
    
    # Enable GPUs (true/false) - use "true" with "g3" flavors
    do_enable_gpu=false

    ### Zero 2 JupyterHub with K3s Rancher Configuration ###

    jupyterhub_deploy_strategy="z2jh_k3s"

    ### Ansible-based Setup for JupyterHub ###
    
    do_ansible_execution=true

    ### CPU usage per user utilization 10000m = 100% of 1 core ###
    
    jh_resources_request_cpu="3000m"
    
    ### Memory RAM per-user requirement in Gigibytes ###
    
    jh_resources_request_memory="10Gi"
        
    ### Set Power Status ###

    power_state = "active"
    # power_state = "shelved_offloaded"

    ### Pre-Cache images? ###

    do_jh_precache_images=false

    user_data = ""

    ### JupyterHub Configurations ###

    # set the type of authentication, comment out Dummy or OAuth
    # OAuth-type (GitHub Authentication)
    # jupyterhub_oauth2_clientid="02b0a98c5d1547101514"
    # jupyterhub_oauth2_secret="ac658055d848e787c31803afd973a05fbdb98f64"
    
    # Dummy-type Authentication
    jupyterhub_authentication="dummy"

    # set the dummy username password
    jupyterhub_dummy_password="password"

    # Specify the JupyterHub admins
    jupyterhub_admins="${USER}"
    
    # add JupyterHub student account usernames 
    jupyterhub_allowed_users="${USER}, student1, student2, student3, student4"

    ### Select a JupyterHub-ready image ###
    
    #jupyterhub_singleuser_image="harbor.cyverse.org/vice/jupyter/pytorch"

    jupyterhub_singleuser_image="jupyter/datascience-notebook"

    jupyterhub_singleuser_image_tag="latest"

    ```