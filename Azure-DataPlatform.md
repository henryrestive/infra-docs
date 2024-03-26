# GitOps & Terraform
```
        +--------------------------+       +-------------------------------+
        |                          |       |                               |
        |  Data Platform           |       |  Azure DevOps                 |
        |                          |       |                               |
        |  - Azure Data Factory    |       |  - Pipelines                  |
        |  - Azure Synapse         |       |  - Repositories               |
        |  - Azure Data Lake       |       |  - Builds                     |
        |  - Azure Purview         |       |  - Releases                   |
        |                          |       |  - Artifacts                  |
        |                          |       |  - Boards                     |
        |                          |       |  - Test Plans                 |
        +--------------------------+       +-------------------------------+
                 |                                     |
                 |            Infrastructure           |
                 +-------------------------------------+
```

In this diagram:
- The "Data Platform" consists of various Azure services such as Azure Data Factory, Azure Synapse Analytics, Azure Data Lake, and Azure Purview, which are used for data ingestion, processing, storage, and governance.
- "Azure DevOps" provides a set of integrated tools for managing the software development lifecycle (SDLC), including version control, continuous integration (CI), continuous delivery (CD), project management, and testing.
- Infrastructure resources are managed and provisioned using infrastructure as code (IaC) tools like Terraform, and the configuration and deployment process is orchestrated through Azure DevOps pipelines.
- Azure DevOps pipelines are configured to automate tasks such as deploying infrastructure changes, building and testing code, and deploying applications to various environments.

This diagram illustrates the integration between the Data Platform and Azure DevOps, where infrastructure changes and application deployments are managed and automated using Azure DevOps pipelines.

# Git Repository Structure
---

When structuring a Terraform project for a Data Platform on Azure, it's important to organize your files and directories in a way that promotes modularity, reusability, and maintainability. Here's a suggested folder structure for such a project:

```
data-platform/
├── environments/
│   ├── dev/
│   │   ├── main.tf              # Main configuration for the development environment
│   │   ├── variables.tf         # Variables specific to the development environment
│   │   └── outputs.tf           # Outputs specific to the development environment
│   ├── staging/
│   │   ├── main.tf              # Main configuration for the staging environment
│   │   ├── variables.tf         # Variables specific to the staging environment
│   │   └── outputs.tf           # Outputs specific to the staging environment
│   └── prod/
│       ├── main.tf              # Main configuration for the production environment
│       ├── variables.tf         # Variables specific to the production environment
│       └── outputs.tf           # Outputs specific to the production environment
├── modules/
│   ├── azure-common/
│   │   │   ├── main.tf          # Main configuration for Azure Common Infra: VNet, Resource Group, KeyVault
│   │   │   ├── variables.tf     # Variables for the Azure Common Infra module
│   │   │   └── outputs.tf       # Outputs for the Azure Common Infra module
│   ├── data-factory/
│   │   ├── main.tf              # Main configuration for Azure Data Factory module: DataSet, Linked Service, Pipelines
│   │   ├── variables.tf         # Variables for the Azure Data Factory module
│   │   └── outputs.tf           # Outputs for the Azure Data Factory module
│   ├── synapse/
│   │   ├── main.tf              # Main configuration for Azure Synapse module
│   │   ├── variables.tf         # Variables for the Azure Synapse module
│   │   └── outputs.tf           # Outputs for the Azure Synapse module
│   ├── onelake/
│   │   ├── main.tf              # Main configuration for Azure OneLake module
│   │   ├── variables.tf         # Variables for the Azure OneLake module
│   │   └── outputs.tf           # Outputs for the Azure OneLake module
│   └── pureview/
│       ├── main.tf              # Main configuration for Azure PureView module
│       ├── variables.tf         # Variables for the Azure PureView module
│       └── outputs.tf           # Outputs for the Azure PureView module
├── pipelines/
│   ├── entry-gate/              # Pipeline definitions (yml) of resources under entry gate
│   │   └── ... 
│   ├── master-data/             # Pipeline definitions (yml) of resources under master data
│   │   └── ... 
│   └── control-data-plane/      # Pipeline definitions (yml) of resources under control plane
│       └── ...
├── helper-scripts/              # helper scripts in bash/python 
├── main.tf                      # Main Terraform configuration for the entire data platform
├── variables.tf                 # Global variables for the entire data platform
└── outputs.tf                   # Global outputs for the entire data platform
```

Explanation:

- `environments/`: This directory contains environment-specific configurations for different environments like development, staging, and production.
- `modules/`: This directory contains reusable Terraform modules for different components of the Data Platform such as Data Factory, Synapse, OneLake, and PureView.
- `pipelines/`: This directory contains YML definitions of pipelines.
- `helper-scripts/`: This directory contains helper scripts which could be bash or python which could be used in any step of a pipeline.
- `main.tf`: This file contains the main Terraform configuration for the entire data platform, including resource dependencies and module invocations.
- `variables.tf`: This file contains global variables that can be used across all environments and modules.
- `outputs.tf`: This file contains global outputs that provide information about resources provisioned by the data platform.

This folder structure allows you to easily manage environment-specific configurations, reuse Terraform modules across different environments, and maintain a clear separation of concerns between different components of the data platform.

# Approach 1: VM Agents pool for pipeline
---
           +---------------------+
           | Azure DevOps        |
           | (Azure Pipelines)   |
           +----------+----------+
                      |
             +--------v---------+
             |   Build/Release  |
             |   Pipelines      |
             +--------+---------+
                      |
       +--------------v--------------+
       |                             |
       |  Azure DevOps Agent Pool   |
       |  (Provisioned with Terraform)|
       +--------------+--------------+
                      |
            +---------v---------+
            |    Custom VM     |
            |    (Agent Image)  |
            +-------------------+
1. **Create a Virtual Machine Image**:
   Define a Terraform configuration to create a custom VM image with the necessary tools installed. Below is an example configuration:

   ```hcl
   provider "azurerm" {
     features {}
   }

   resource "azurerm_virtual_machine" "custom_image_vm" {
     name                  = "custom-vm"
     location              = "Australia Southeast"
     resource_group_name   = azurerm_resource_group.data_platform.name
     network_interface_ids = [azurerm_network_interface.data_platform.id]
     vm_size               = "Standard_DS2_v2"

     storage_image_reference {
       publisher = "Canonical"
       offer     = "UbuntuServer"
       sku       = "18.04-LTS"
       version   = "latest"
     }

     os_profile {
       computer_name  = "hostname" # Value from variable
       admin_username = "adminuser" # Value from variable
       admin_password = "Password123!" # Value from variable
     }

     os_profile_linux_config {
       disable_password_authentication = false
     }

     provisioner "remote-exec" {
       inline = [
         "sudo apt-get update",
         "sudo apt-get install -y terraform",
         "curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash",
         "sudo npm install -g azure-devops"
       ]
     }
   }
   ```

   Adjust the configuration according to your requirements, such as specifying the desired VM size, image, credentials, and provisioner commands.

2. **Use the Custom Image in Azure DevOps Agent Pool**:
   Configure your Azure DevOps agent pool to use the custom image created by Terraform. You can specify the custom image in the agent pool configuration using the `agent_image` parameter.

3. **Use the Agent Pool in Pipelines**:
   When defining your pipelines in Azure DevOps, specify the agent pool that uses the custom image. Pipelines running on agents from this pool will have Terraform, Azure CLI, and Azure DevOps CLI available out of the box.

# Approach 2: Containerized agents pool for pipeline
---

```plaintext
           +---------------------+
           | Azure DevOps        |
           | (Azure Pipelines)   |
           +----------+----------+
                      |
             +--------v---------+
             |   Build/Release  |
             |   Pipelines      |
             +--------+---------+
                      |
       +--------------v--------------+
       |                             |
       |  Azure DevOps Agent Pool   |
       |  (Provisioned with Terraform)|
       +--------------+--------------+
                      |
            +---------v---------+
            |    Docker        |
            |    Container     |
            +-------------------+
```

```Dockerfile
FROM ubuntu:latest

# Install required packages
RUN apt-get update && \
    apt-get install -y curl gnupg && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# Install Terraform CLI
RUN curl -fsSL https://apt.releases.hashicorp.com/gpg | apt-key add - && \
    echo "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main" > /etc/apt/sources.list.d/hashicorp.list && \
    apt-get update && \
    apt-get install -y terraform && \
    terraform --version

# Install Azure CLI
RUN curl -sL https://aka.ms/InstallAzureCLIDeb | bash && \
    az --version

# Install Azure DevOps CLI
RUN curl -sL https://aka.ms/InstallAzureDevOpsCli | bash && \
    azdevops --version

# Set environment variables for service principal authentication (injected from Azure KeyVault through env via AKS yml)
ENV AZURE_CLIENT_ID=<client-id>
ENV AZURE_CLIENT_SECRET=<client-secret>
ENV AZURE_TENANT_ID=<tenant-id>

# Log in to Azure using service principal
RUN az login --service-principal --username $AZURE_CLIENT_ID --password $AZURE_CLIENT_SECRET --tenant $AZURE_TENANT_ID

# Set Azure CLI context
RUN az account set --subscription <subscription-id>

# Run a shell to keep container running
CMD ["/bin/bash"]
```

To instruct an Azure DevOps agent pool to run on an AKS (Azure Kubernetes Service) cluster using Terraform, you would typically follow these steps:

1. Define an AKS cluster in Terraform: Define the AKS cluster resource using the `azurerm_kubernetes_cluster` Terraform provider. You would specify details such as the Kubernetes version, node count, node sizes, networking configurations, etc.

2. Configure the Azure DevOps agent pool to use AKS: When creating or updating the Azure DevOps agent pool resource, you need to specify the `agent_pool_type` as `Kubernetes`. This tells Azure DevOps to use a Kubernetes cluster as the agent pool backend.

3. Link the AKS cluster to the agent pool: You must provide the connection details of the AKS cluster to the Azure DevOps agent pool. This includes the Kubernetes API server URL, the authentication credentials (service principal), and other required configurations.

Here's a basic example of how you might define these resources in Terraform:

```hcl
provider "azurerm" {
  features {}
}

resource "kubernetes_secret" "azure_credentials" {
  metadata {
    name = "azure-credentials"
  }
  data = {
    client-id     = "your-client-id" # value from variables
    client-secret = "your-client-secret" # value from variables
    tenant-id     = "your-tenant-id" # value from variables
  }
}

# Define AKS cluster
resource "azurerm_kubernetes_cluster" "data_platform" {
  name                = "dp-aks-cluster"
  location            = "Australia Southeast"
  resource_group_name = "dp-resources"
  dns_prefix          = "dpaks"
  kubernetes_version  = "1.21.2"
  node_resource_group = "dp-node-resources"

  default_node_pool {
    name            = "default"
    node_count      = 3
    vm_size         = "Standard_DS2_v2"
    enable_auto_scaling = true
    min_count = 1
    max_count = 3
  }

  identity {
    type = "SystemAssigned"
  }

  tags = {
    environment = "Production"
  }
}

# Define Azure DevOps agent pool
resource "azurerm_devops_agent_pool" "data_platform" {
  project_id    = azurerm_devops_project.data_platform.id
  name          = "data_platform-agent-pool"
  agent_pool_type = "Kubernetes"

  kubernetes {
    kube_config {
      host                   = azurerm_kubernetes_cluster.data_platform.kube_config[0].host
      client_certificate     = base64decode(azurerm_kubernetes_cluster.data_platform.kube_config[0].client_certificate)
      client_key             = base64decode(azurerm_kubernetes_cluster.data_platform.kube_config[0].client_key)
      cluster_ca_certificate = base64decode(azurerm_kubernetes_cluster.data_platform.kube_config[0].cluster_ca_certificate)
    }
  }
  container_image = "myregistry.azurecr.io/mydockerimage:latest"
  
}

# Define Azure DevOps project
resource "azurerm_devops_project" "data_platform" {
  name     = "data-platform-project"
  visibility = "public"
  capabilities {
    version_control {
      source_control_type = "Git"
    }
  }
}
```
