# Lab 02 - Using Ansible to Create a GitHub Runner

## Table of Contents

- [Objective](#objective)
- [Guide](#guide)
  - [Step 01: Open a new branch in the repository](#step-01-open-a-new-branch-in-the-repository)
  - [Step 02: Create an Terraform for your GitHub Runner Azure VM](#step-02-create-an-terraform-for-your-github-runner-azure-vm)
  - [Step 03: Create GitHub Actions to deploy the GitHub Runner](#step-03-create-github-actions-to-deploy-the-github-runner)
  - [Step 04: Prepare your repository for Ansible](#step-04-prepare-your-repository-for-ansible)
  - [Step 05: Open a new branch for ansible code](#step-05-open-a-new-branch-for-ansible-code)
  - [Step 06: Set Up Inventory](#step-06-set-up-inventory)
  - [Step 07: Create Ansible Playbook](#step-07-create-ansible-playbook)
  - [Step 08: Create Github Actions to run Ansible Playbook](#step-08-create-github-actions-to-run-ansible-playbook)
  - [Step 09: Check the GitHub Runner](#step-09-check-the-github-runner)
- [Conclusion](#conclusion)

## Objective

Set up a GitHub runner on a target machine using Ansible.

## Guide

### Step 01: Open a new branch in the repository

On your local branch, checkout to `main` branch and pull the latest changes.

```bash
git checkout main
git pull origin main
```

Create a new branch for the changes.

```bash
git checkout -b topic/lab-02-tf
```

### Step 02: Create an Terraform for your GitHub Runner Azure VM

Create a new folder named `github-runner` on existing folder `deploy/terraform`.

```bash
mkdir -p deploy/terraform/github-runner
cd deploy/terraform/github-runner
```

Create a new file named `ssh.tf` and add the following content:

```hcl

resource "azapi_resource_action" "ssh_public_key_gen" {
  type        = "Microsoft.Compute/sshPublicKeys@2022-11-01"
  resource_id = azapi_resource.ssh_public_key.id
  action      = "generateKeyPair"
  method      = "POST"

  response_export_values = ["publicKey", "privateKey"]
}

resource "azapi_resource" "ssh_public_key" {
  type      = "Microsoft.Compute/sshPublicKeys@2022-11-01"
  name      = "<your-prefix>-gh-runner-ssh-key"
  location  = azurerm_resource_group.rg.location
  parent_id = azurerm_resource_group.rg.id
}

output "key_data" {
  value = azapi_resource_action.ssh_public_key_gen.output.publicKey
}

output "pvt_data" {
  value = azapi_resource_action.ssh_public_key_gen.output.privateKey
}
```

Then, create a `main.tf` file and add the following content:

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "<your-prefix>-tfstate-rg"
    storage_account_name = "<your-prefix>tfstatestg"
    container_name       = "tfstate"
    key                  = "terraform.tfstate"
  }

  required_providers {
    azapi = {
      source  = "azure/azapi"
      version = "~>1.0"
    }
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~>3.0"
    }
    random = {
      source  = "hashicorp/random"
      version = "~>3.0"
    }
  }
}

provider "azurerm" {
  use_oidc = true
  features {}
}

resource "azurerm_resource_group" "rg" {
  location = var.resource_group_location
  name     = "<your-prefix>-github-runner-rg"
}

# Create virtual network
resource "azurerm_virtual_network" "my_terraform_network" {
  name                = "<your-prefix>-gh-runner-vnet"
  address_space       = ["10.0.0.0/16"]
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
}

# Create subnet
resource "azurerm_subnet" "my_terraform_subnet" {
  name                 = "<your-prefix>-gh-runner-subnet"
  resource_group_name  = azurerm_resource_group.rg.name
  virtual_network_name = azurerm_virtual_network.my_terraform_network.name
  address_prefixes     = ["10.0.1.0/24"]
}

# Create public IPs
resource "azurerm_public_ip" "my_terraform_public_ip" {
  name                = "<your-prefix>-gh-runner-pip"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  allocation_method   = "Dynamic"
}

# Create Network Security Group and rule
resource "azurerm_network_security_group" "my_terraform_nsg" {
  name                = "<your-prefix>-gh-runner-nsf"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  security_rule {
    name                       = "SSH"
    priority                   = 1001
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "22"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }
}

# Create network interface
resource "azurerm_network_interface" "my_terraform_nic" {
  name                = "<your-prefix>-gh-runner-nic"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  ip_configuration {
    name                          = "my_nic_configuration"
    subnet_id                     = azurerm_subnet.my_terraform_subnet.id
    private_ip_address_allocation = "Dynamic"
    public_ip_address_id          = azurerm_public_ip.my_terraform_public_ip.id
  }
}

# Connect the security group to the network interface
resource "azurerm_network_interface_security_group_association" "example" {
  network_interface_id      = azurerm_network_interface.my_terraform_nic.id
  network_security_group_id = azurerm_network_security_group.my_terraform_nsg.id
}

# Create virtual machine
resource "azurerm_linux_virtual_machine" "my_terraform_vm" {
  name                  = "<your-prefix>-gh-runner-vm"
  location              = azurerm_resource_group.rg.location
  resource_group_name   = azurerm_resource_group.rg.name
  network_interface_ids = [azurerm_network_interface.my_terraform_nic.id]
  size                  = "Standard_DS1_v2"

  os_disk {
    name                 = "myOsDisk"
    caching              = "ReadWrite"
    storage_account_type = "Premium_LRS"
  }

  source_image_reference {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-jammy"
    sku       = "22_04-lts-gen2"
    version   = "latest"
  }

  computer_name  = "hostname"
  admin_username = var.username

  admin_ssh_key {
    username   = var.username
    public_key = azapi_resource_action.ssh_public_key_gen.output.publicKey
  }
}
```

Don't forget to replace the `<your-prefix>` with your own prefix.

Create a `variables.tf` file and add the following content:

```hcl
variable "resource_group_location" {
  type        = string
  default     = "westeurope"
  description = "Location of the resource group."
}

variable "username" {
  type        = string
  description = "The username for the local account that will be created on the new VM."
  default     = "azureadmin"
}
```

And finally create a `outputs.tf` file and add the following content:

```hcl
output "public_ip_address" {
  value = azurerm_linux_virtual_machine.my_terraform_vm.public_ip_address
}
```

### Step 03: Create GitHub Actions to deploy the GitHub Runner

Create a file named `github-runner.yml` on `.github/workflows` folder and add the following content:

```yaml
name: GitHub-Runner

on:
  workflow_dispatch:
  push:
    branches: [ main ]
    paths:
      - '.github/workflows/github-runner.yml'
      - 'deploy/terraform/github-runner/**'

  pull_request:
    branches: [ main ]
    paths:
      - '.github/workflows/github-runner.yml'
      - 'deploy/terraform/github-runner/**'

env:
  ARTIFACT_NAME: "github-runner"

permissions:
  id-token: write
  contents: read

jobs:
  pack:

    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v3

    - uses: actions/upload-artifact@v3
      if: github.event_name != 'pull_request'
      with:
        name: ${{ env.ARTIFACT_NAME }}-iac
        path: deploy/terraform/github-runner

  terraform:
    if: github.event_name != 'pull_request'
    runs-on: ubuntu-latest
    needs: pack

    env:
      ARM_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      ARM_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      ARM_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      ARM_USE_OIDC: true

    steps:
    - uses: actions/download-artifact@v3
      with:
        name: ${{ env.ARTIFACT_NAME }}-iac
        path: ./terraform
    
    - uses: azure/login@v1
      with:
        client-id: ${{ secrets.AZURE_CLIENT_ID }}
        tenant-id: ${{ secrets.AZURE_TENANT_ID }}
        subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

    - uses: hashicorp/setup-terraform@v2
      with:
        terraform_wrapper: false

    - name: terraform init
      run: |
        cd ./terraform
        export
        terraform init -backend-config="key=github-runner.tfstate"

    - name: terraform validate
      run: |
        cd ./terraform
        terraform validate
    
    - name: terraform plan
      id: tfplan
      run: |
        cd ./terraform
        terraform plan

    - name: Terraform Plan Status
      if: steps.tfplan.outcome == 'failure'
      run: exit 1

    - name: terraform apply
      run: |
        cd ./terraform
        terraform apply -auto-approve

    - name: logout
      run: |
        az logout
```

Now you can commit and push the changes to your repository.

```bash
git add -A
git commit -m "add GitHub Runner deployment"
git push origin topic/lab-02
```

Create a pull request to merge the changes to the `main` branch.

Follow the same steps you've made on the previous labs to merge the changes.

Don't forget to delete the branch after the merge.

After merge is complete, a new workflow will be triggered to deploy the GitHub Runner.

Before proceeding to the next steps, make sure the GitHub Runner is deployed successfully in Azure.

### Step 04: Prepare your repository for Ansible

On the GitHub Action that created the GitHub Runner VM, check the logs of the action `terraform apply`.

You should get two important outputs:

- The public IP address of the GitHub Runner VM
- The private key of the SSH key pair

For the private key, let's create a new repository secret named `GH_RUNNER_PRIVATE_KEY` and paste the private key value.

To create a secret, go to your repo and navigate to `Settings > Secrets and variables > Actions`, then click on `New repository secret`.

The public IP will be used on the ansible inventory file.

### Step 05: Open a new branch for ansible code

On your local branch, checkout to `main` branch and pull the latest changes.

```bash
git checkout main
git pull origin main
```

Create a new branch for the changes.

```bash
git checkout -b topic/lab-02-ansible
```

### Step 06: Set Up Inventory

Create a new folder named `ansible` inside the `deploy` folder.

Then create a new folder named `github-runner` inside the `ansible` folder.

```bash
mkdir -p deploy/ansible/github-runner
cd deploy/ansible/github-runner
```

Create a new folder named `inventory` and create a new file named `hosts.yml` inside the `inventory` folder.

Inside the `hosts.yml` file, add the following content:

```yaml
runner:
  hosts:
    github-runner-001:
      ansible_host: <azure-vm-public-ip>
      ansible_ssh_private_key_file: ./vm_key.pem
      ansible_user: azureadmin
```

Replace `<azure-vm-public-ip>` with the public IP address of the GitHub Runner VM.

Now to handle SSH connection, you need to create a file named `ansible.cfg` inside the `github-runner` folder.

Add the following content to the `ansible.cfg` file:

```ini
[defaults]
host_key_checking = False
```

This configuration will disable the SSH host key checking and allow automation using a GitHub Hosted-Runner.

### Step 07: Create Ansible Playbook

Create a new file named `playbook.yml` inside the `github-runner` folder.

```yaml
---
- name: Prepare VM for GitHub Runner
  hosts: runner
  become: true

  tasks:
    - name: Install required packages
      ansible.builtin.apt:
        name: ['git', 'curl', 'jq', 'dotnet-sdk-8.0']
        state: present
      become: true

    - name: Create GitHub Runner user
      ansible.builtin.user:
        name: github-runner
        state: present
        shell: /bin/bash
        createhome: yes
      become: true

    - name: Download GitHub Runner
      ansible.builtin.get_url:
        url: https://github.com/actions/runner/releases/latest/download/actions-runner-linux-x64-2.283.2.tar.gz
        dest: /home/github-runner/actions-runner-linux-x64.tar.gz
        mode: 0644
      become: true

    - name: Extract GitHub Runner
      ansible.builtin.unarchive:
        src: /home/github-runner/actions-runner-linux-x64.tar.gz
        dest: /home/github-runner
        remote_src: yes
      become: true

    - name: Configure GitHub Runner
      ansible.builtin.command: |
        /home/github-runner/config.sh --url https://github.com/theonorg/echo-app --token ${{ GITHUB_TOKEN }}
      become: true

    - name: Install GitHub Runner as a service
      ansible.builtin.systemd:
        name: actions.runner.your_vm_host
        state: started
        enabled: yes
        daemon_reload: yes
        user: github-runner
        exec_start: /home/github-runner/run.sh
      become: true
```

Take a deep look into your playbook and check all tasks that will be performed on your newly created VM.

On first task, you may check that you have an array of packages to be installed. You can update this task to included all needed packages to build and deploy your project.

### Step 08: Create Github Actions to run Ansible Playbook

Create a new file named `github-runner-config.yml` inside the `.github/workflows` folder.

```yaml
name: GitHub-Runner-Config

on:
  workflow_dispatch:
  push:
    branches: [ main ]
    paths:
      - '.github/workflows/github-runner-config.yml'
      - 'deploy/ansible/**'

  pull_request:
    branches: [ main ]
    paths:
      - '.github/workflows/github-runner-config.yml'
      - 'deploy/ansible/**'

env:
  ARTIFACT_NAME: "github-runner-config"

permissions:
  id-token: write
  contents: read

jobs:
  pack:

    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v3

    - uses: actions/upload-artifact@v3
      if: github.event_name != 'pull_request'
      with:
        name: ${{ env.ARTIFACT_NAME }}-ansible
        path: deploy/ansible

  ansible:
    if: github.event_name != 'pull_request'
    runs-on: ubuntu-latest
    needs: pack

    env:
      PVT_KEY: ${{ secrets.VM_PVT_KEY }}

    steps:
    - uses: actions/download-artifact@v3
      with:
        name: ${{ env.ARTIFACT_NAME }}-ansible
        path: ./ansible

    - name: create key file
      run: |
        cd ./ansible
        echo $PVT_KEY > inventory/vm_key.pem
        chmod 400 ./inventory/vm_key.pem
        cat ./inventory/vm_key.pem

    - name: run ansible playbook
      run: |
        cd ./ansible
        ansible-playbook -i inventory/hosts.yml playbook.yml
```

Now you can commit and push the changes to your repository.

```bash
git add -A
git commit -m "add GitHub Runner deployment"
git push origin topic/lab-02
```

Create a pull request to merge the changes to the `main` branch.

Follow the same steps you've made on the previous labs to merge the changes.

Don't forget to delete the branch after the merge.

After merge is complete, a new workflow will be triggered to deploy the GitHub Runner.

### Step 09: Check the GitHub Runner

After the workflow is completed, check the GitHub Runner on your repository.

Go to your repository, then navigate to `Settings > Actions > Runners`.

You should see a new runner with the name of your VM.

## Conclusion

On this lab, you learned how to use Ansible to automate the deployment of a GitHub Runner on an Azure VM.
