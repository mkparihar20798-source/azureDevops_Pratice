# azureDevops_Pratice
<br>
This is the first repos where i maintain my azure devops project
SOP for the Azure Devops
this document is created for basic understanding of how the work flow happens in the azure devops
what is Devops--- Devops is a culture and set of practices that brings collabration and development team to automate the task, ensure continous integration and continous devoplement , reduce software deplyoment time , faster time to market, ensure 
<br>
https://learn.microsoft.com/en-us/azure/devops/repos/git/commits?view=azure-devops&tabs=visual-studio-2022
<br>
terrfarom code to deployee a Azure virtual Machine to a host pool by  providing the subsripation 
Create these three files in a folder, then run terraform init && terraform apply.
Provider.tf
terraform {
  required_version = ">= 1.6.0"
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.113" # use a recent version
    }
    azuread = {
      source  = "hashicorp/azuread"
      version = "~> 2.47"
    }
  }
}

provider "azurerm" {
   features {}

  # If you want to pin to a specific subscription ID, set it here or via env var.
  # subscription_id = var.subscription_id
}

# Optional: needed only if you plan AAD group assignments, etc.
<br>
Variable.tf

variable "subscription_id" {
  description = "Azure Subscription ID (optional if you use az login  description = "Azure Subscription ID (optional if you use az login context)"
  type        = string
  default     = null
}

variable "location" {
  description = "Azure region"
  type        = string
  default     = "westus2"
}

variable "rg_name" {
  description = "Existing resource group name"
  type        = string
  default     = "np-corp-avd-hp1-wus2"
}

variable "hostpool_name" {
  description = "AVD Host Pool name"
  type        = string
  default     = "np-corp-avd-hp1-wus2"
}

variable "workspace_name" {
  description = "AVD Workspace name"
  type        = string
  default     = "np-corp-avd-ws-wus2"
}

variable "dag_name" {
  description = "Desktop Application Group name"
  type        = string
  default     = "np-corp-avd-dag-wus2"
}

variable "hostpool_type" {
  description = "Host Pool type: 'Pooled' or 'Personal'"
  type        = string
  default     = "Pooled"
  validation {
    condition     = contains(["Pooled", "Personal"], var.hostpool_type)
    error_message = "hostpool_type must be 'Pooled' or 'Personal'."
  }
}

variable "max_sessions" {
  description = "Maximum sessions per session host (Pooled only)"
  type        = number
  default     = 20
}

variable "registration_token_valid_days" {
  description = "Registration token validity in days"
  type        = number
  default     = 7
}

# Optional tags
variable "tags" {
  description = "Resource tags"
  type        = map(string)
  default     = {
    env     = "prod"
    service = "avd"
    owner   = "np-corp"
  }
<br>
Main.tf

# ------------------------------------------------------------
# Data: Use an existing resource group
# ------------------------------------------------------------
data "azurerm_resource_group" "avd_rg" {
  name = var.rg_name
}

# ------------------------------------------------------------
# AVD Host Pool
# ------------------------------------------------------------
resource "azurerm_virtual_desktop_host_pool" "hp" {
  name                = var.hostpool_name
  resource_group_name = data.azurerm_resource_group.avd_rg.name
  location            = data.azurerm_resource_group.avd_rg.location

  type                 = var.hostpool_type              # "Pooled" or "Personal"
  preferred_app_group_type = "Desktop"                 # Typically "Desktop" for DAGs
  start_vm_on_connect  = true                          # Helpful to start stopped hosts

  # For pooled host pools you can set load balancer and max sessions
  load_balancer_type          = var.hostpool_type == "Pooled" ? "DepthFirst" : null
  maximum_sessions_allowed    = var.hostpool_type == "Pooled" ? var.max_sessions : null

  # Registration token to allow session hosts to join
  registration_info {
    expiration_date = timeadd(timestamp(), format("%dh", var.registration_token_valid_days * 24))
    # Alternatively, specify token_type = "Permanent"; but expiration is recommended.
  }

  tags = var.tags
}

# ------------------------------------------------------------
# Desktop Application Group (DAG)
# ------------------------------------------------------------
resource "azurerm_virtual_desktop_application_group" "dag" {
  name                = var.dag_name
  resource_group_name = data.azurerm_resource_group.avd_rg.name
  location            = data.azurerm_resource_group.avd_rg.location

  type       = "Desktop"
  host_pool_id = azurerm_virtual_desktop_host_pool.hp.id

  tags = var.tags
}

# ------------------------------------------------------------
# Workspace
# ------------------------------------------------------------
resource "azurerm_virtual_desktop_workspace" "ws" {
  name                = var.workspace_name
  resource_group_name = data.azurerm_resource_group.avd_rg.name
  location            = data.azurerm_resource_group.avd_rg.location

  friendly_name = var.workspace_name
  description   = "Workspace for ${var.hostpool_name}"

  tags = var.tags
}

# ------------------------------------------------------------
# Workspace <-> Application Group association
# ------------------------------------------------------------
resource "azurerm_virtual_desktop_workspace_application_group_association" "ws_dag_assoc" {
  workspace_id        = azurerm_virtual_desktop_workspace.ws.id
  application_group_id = azurerm_virtual_desktop_application_group.dag.id
}

# ------------------------------------------------------------
# Useful outputs
# ------------------------------------------------------------
output "hostpool_id" {
  value = azurerm_virtual_desktop_host_pool.hp.id
}

output "registration_token" {
  description = "Use this token when building session hosts (AVD Agent)"
  value       = azurerm_virtual_desktop_host_pool.hp.registration_info[0].token
  sensitive   = true
}

output "workspace_id" {
  value = azurerm_virtual_desktop_workspace.ws.id
}

output "app_group_id" {
  value = azurerm_virtual_desktop_application_group.dag.id
}
``

