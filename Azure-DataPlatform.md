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

# Git Repoitory Structure
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

# Agents pool for pipeline
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
     location              = "East US"
     resource_group_name   = azurerm_resource_group.example.name
     network_interface_ids = [azurerm_network_interface.example.id]
     vm_size               = "Standard_DS2_v2"

     storage_image_reference {
       publisher = "Canonical"
       offer     = "UbuntuServer"
       sku       = "18.04-LTS"
       version   = "latest"
     }

     os_profile {
       computer_name  = "hostname"
       admin_username = "adminuser"
       admin_password = "Password123!"
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

This approach allows you to use Terraform to provision the VMs and install the required tools directly, eliminating the need for a separate tool like Packer. Adjust the Terraform configuration as needed for your specific requirements and environment.