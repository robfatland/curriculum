## Welcome to Curriculum

These are notes on setting up student environments on the cloud. Right now just Azure using Terraform; more solutions welcome. 
Also please note that the Azure section is missing two important components: First some intelligent Policy design that constrains
SKU access ("SKU" is an execrable term for "which virtual machines you are permitted to spin up") and second some more intelligent
Terraform scripting to delete resource groups at the end of the term. Also highly valuable would be a burn list for charges accrued
by all students at a glance, ideally sent daily to instructors to avoid accidental runaway costs. 


### Azure

This section is about granting measured access to a number of students. 
For the Azure preparation steps described here we use Terraform. 
The necessary commands are issued on a PC from the Windows Subsystem for Linux (WSL) shell.
WSL is installed as part of the PC installation of Visual Studio Code. 
On a non-PC I think you'd just use `bash`. 


* Start a WSL shell...
* Create a directory: `mkdir terraform-azure`
* Install `main.tf` in this directory (see below for template) and edit it to correspond to the student list
* From this same directory execute the command sequence below
    * `az login` and `az account set` aligns the terraform process with the correct location on the Azure cloud
    * the `terraform` commands use Terraform syntax to build the Azure Resource Groups for each student and pre-populate them


Here is the command sequence:


```
az login
az account set --subscription="aaaaaa-aaaa-aaaa-aaaaaaaaaaaa"
terraform -install-autocomplete
terraform init
terraform plan
terraform apply
```

The `apply` step can take some time (tens of minutes) depending on the provisioning load and number of students. 


Here is `main.tf` with just two imaginary students listed:


```
provider "azurerm" {
  features {}
}

data "azurerm_client_config" "current" {}
locals {
  students = [
    "12345678-9abc-def0-0000-000000000000",
"12345678-9abc-def0-0000-000000000001"
  ]
}

/* create a resource group for each student */
resource "azurerm_resource_group" "aml" {
  count    = length(local.students)
  name     = "rg-amlclass-${count.index + 1}"
  location = "West US 2"
}

/* assign the role of Contributor to each student */
resource "azurerm_role_assignment" "aml" {
  count                = length(local.students)
  scope                = azurerm_resource_group.aml[count.index].id
  role_definition_name = "Contributor"
  principal_id         = local.students[count.index]
}

/* assign a monitoring service to each RG called 'application insights' */
resource "azurerm_application_insights" "aml" {
  count               = length(local.students)
  name                = "ai-amlclass-${count.index + 1}"
  location            = azurerm_resource_group.aml[count.index].location
  resource_group_name = azurerm_resource_group.aml[count.index].name
  application_type    = "web"
}

/* assign a key vault to each RG */
resource "azurerm_key_vault" "aml" {
  count               = length(local.students)
  name                = "kv-amlclass-${count.index + 1}"
  location            = azurerm_resource_group.aml[count.index].location
  resource_group_name = azurerm_resource_group.aml[count.index].name
  tenant_id           = data.azurerm_client_config.current.tenant_id
  sku_name            = "standard"
}

/* assign a storage account to each RG */ 
resource "azurerm_storage_account" "aml" {
  count                    = length(local.students)
  name                     = "saamlclass${count.index + 1}"
  location                 = azurerm_resource_group.aml[count.index].location
  resource_group_name      = azurerm_resource_group.aml[count.index].name
  account_tier             = "Standard"
  account_replication_type = "LRS"
}

/* create a machine learning workspace in each student RG */
resource "azurerm_machine_learning_workspace" "aml" {
  count                   = length(local.students)
  name                    = "amlclass${count.index + 1}"
  location                = azurerm_resource_group.aml[count.index].location
  resource_group_name     = azurerm_resource_group.aml[count.index].name
  application_insights_id = azurerm_application_insights.aml[count.index].id
  key_vault_id            = azurerm_key_vault.aml[count.index].id
  storage_account_id      = azurerm_storage_account.aml[count.index].id
  identity {
    type = "SystemAssigned"
  }
}
```
