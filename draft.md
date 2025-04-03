# Just Enough Terraform for Coder

## Introduction

This guide provides the essential Terraform knowledge needed to work effectively with Coder templates. Coder uses Terraform to provision and manage development environments, and understanding key Terraform concepts will help you create and customize your own templates.

## What is Terraform?

Terraform is an infrastructure as code (IaC) tool that lets you define and provision infrastructure using a declarative configuration language. In the context of Coder, Terraform is used to define the resources that make up your development environments.

## Basic Terraform Concepts

### 1. Terraform Files and HCL

Terraform configurations are written in HashiCorp Configuration Language (HCL) and typically stored in files with a `.tf` extension. HCL is a structured configuration language that is both human-readable and machine-friendly.

```terraform
# main.tf - A simple Terraform file
```

Key HCL features include:
- Block syntax for defining resources, providers, and other components
- Expressions for referencing variables and performing operations
- Comments using the `#` or `//` symbols
- Labels for identifying resources and blocks (e.g., resource names)

### 2. Providers

Providers are plugins that allow Terraform to interact with cloud providers, APIs, and other services.

```terraform
terraform {
  required_providers {
    coder = {
      source  = "coder/coder"
      version = "~> 0.11.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.20"
    }
  }
}

provider "coder" {
  # Configuration options
}

provider "kubernetes" {
  # Kubernetes provider configuration
}
```

### 3. Resources

Resources are infrastructure objects that Terraform manages, such as Kubernetes pods, volumes, or cloud instances. Each resource has a TYPE and a NAME (these are labels in HCL), forming a unique identifier.

```terraform
resource "kubernetes_pod" "dev_pod" {  # TYPE = kubernetes_pod, NAME = dev_pod
  metadata {
    name = "dev-pod"  # This is an attribute within the metadata block
  }
  
  spec {
    container {
      name  = "dev-container"  # Attribute
      image = "ubuntu:latest"  # Attribute
    }
  }
}
```

### 4. Variables and Locals

Variables allow you to parameterize your configurations, making them more reusable.

```terraform
variable "image" {  # "image" is the label/name of this variable
  description = "Container image to use for the workspace"  # Attribute
  default     = "ubuntu:latest"  # Attribute with default value
  validation {
    condition     = contains(["ubuntu:latest", "debian:stable"], var.image)
    error_message = "Invalid image selected."
  }
}

# Local variables are computed values used within your configuration
locals {
  workspace_name = "coder-${data.coder_workspace.me.owner}-${data.coder_workspace.me.name}"
  common_labels = {
    "app.kubernetes.io/name" = "coder-workspace"
    "coder.workspace_id"     = data.coder_workspace.me.id
  }
}
```

Variables are referenced using `var.name_of_variable` and locals using `local.name_of_local`.

### 5. Outputs

Outputs allow you to expose specific values from your configuration that can be used by Coder.

```terraform
output "ip_address" {  # "ip_address" is the label/name of this output
  value = kubernetes_pod.dev_pod.status.0.pod_ip  # Attribute containing the value to output
}
```

### 6. Data Sources

Data sources allow Terraform to fetch information from external sources.

```terraform
data "coder_workspace" "me" {  # TYPE = coder_workspace, NAME = me
  # This data source doesn't require additional attributes
}
```

## Coder-Specific Concepts

### 1. Coder Agent

The Coder agent is required in each workspace to connect to the Coder platform.

```terraform
resource "coder_agent" "main" {  # "main" is the label for this agent resource
  os   = "linux"  # Attribute
  arch = "amd64"  # Attribute
  
  # Define startup script (attribute with heredoc syntax)
  startup_script = <<-EOT
    # Install development tools
    apt-get update
    apt-get install -y git vim
  EOT
}
```

### 2. Coder Apps

Define applications that can be accessed within the workspace.

```terraform
resource "coder_app" "code_server" {  # "code_server" is the label for this app
  agent_id     = coder_agent.main.id  # Attribute referencing the agent's id
  slug         = "code-server"        # Attribute
  display_name = "VS Code"            # Attribute
  url          = "http://localhost:8080/?folder=/home/coder"
  icon         = "/icon/code.svg"
  subdomain    = false
}
```

## Common Terraform Patterns for Coder Templates

### 1. Kubernetes Pod Workspace

```terraform
resource "kubernetes_pod" "main" {
  metadata {
    name = "coder-${data.coder_workspace.me.owner}-${data.coder_workspace.me.name}"
    namespace = var.workspaces_namespace
    labels = local.common_labels  # Using local variable to apply common labels
  }
  
  spec {
    container {
      name  = "dev"
      image = var.image  # Using variable for image
      
      # Connect the agent to the pod
      command = ["sh", "-c", coder_agent.main.init_script]
      env {
        name  = "CODER_AGENT_TOKEN"
        value = coder_agent.main.token
      }
      
      # Resources
      resources {
        requests = {
          cpu    = "500m"
          memory = "500Mi"
        }
        limits = {
          cpu    = "${var.cpu_limit}"
          memory = "${var.memory_limit}Gi"
        }
      }
      
      # Storage
      volume_mount {
        mount_path = "/home/coder"
        name       = "home-volume"
      }
    }
    
    # Add storage volume
    volume {
      name = "home-volume"
      persistent_volume_claim {
        claim_name = kubernetes_persistent_volume_claim.home.metadata.0.name
      }
    }
  }
}
```

### 2. StorageClass Example

```terraform
resource "kubernetes_persistent_volume_claim" "home" {
  metadata {
    name      = "home-coder-${data.coder_workspace.me.owner}-${data.coder_workspace.me.name}"
    namespace = var.workspaces_namespace
    labels    = local.common_labels  # Using the same local variable for consistent labeling
  }
  
  spec {
    access_modes = ["ReadWriteOnce"]
    
    # Specify storage class
    storage_class_name = "standard-rwo"
    
    resources {
      requests = {
        storage = "${var.home_disk_size}Gi"
      }
    }
  }
}
```

### 3. ImagePullSecret Example

```terraform
locals {
  # Define a local for the pod name to ensure consistency
  pod_name = "coder-${data.coder_workspace.me.owner}-${data.coder_workspace.me.name}"
}

resource "kubernetes_pod" "main" {
  metadata {
    name      = local.pod_name  # Using the local variable
    namespace = var.workspaces_namespace
  }
  
  spec {
    # Add image pull secret
    image_pull_secrets {
      name = "my-registry-credentials"
    }
    
    container {
      name  = "dev"
      image = "private-registry.example.com/my-dev-image:latest"
      
      # Rest of container configuration...
    }
  }
}
```

## Creating a Complete Coder Template

Please use our include templates, such as the AWS K8s template for full functionality: https://github.com/coder/coder/blob/main/examples/templates/aws-linux/main.tf which will install the remote tools, and code-server! The following example will not run without modifications (namespace, pull-secret, storage-class will be different)

Here's an example combining the concepts above:

```terraform
terraform {
  required_providers {
    coder = {
      source  = "coder/coder"
      version = "~> 0.11.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.20"
    }
  }
}

provider "kubernetes" {
  # Kubernetes provider configuration will be loaded from environment
}

data "coder_workspace" "me" {
}

variable "workspaces_namespace" {
  description = "Kubernetes namespace where workspace pods will be deployed"
  default     = "coder-workspaces"
}

variable "image" {
  description = "Container image for the workspace"
  default     = "codercom/enterprise-base:ubuntu"
}

variable "cpu_limit" {
  description = "CPU limit for the workspace"
  default     = "1"
}

variable "memory_limit" {
  description = "Memory limit for the workspace (in GB)"
  default     = "2"
}

variable "home_disk_size" {
  description = "Home directory disk size (in GB)"
  default     = "10"
}

# Local variables for reuse across the configuration
locals {
  workspace_name = "coder-${data.coder_workspace.me.owner}-${data.coder_workspace.me.name}"
  labels = {
    "app.kubernetes.io/name"     = "coder-workspace"
    "app.kubernetes.io/instance" = local.workspace_name
    "app.kubernetes.io/part-of"  = "coder"
    "coder.workspace_id"         = data.coder_workspace.me.id
    "coder.workspace_name"       = data.coder_workspace.me.name
    "coder.user_id"              = data.coder_workspace.me.owner_id
    "coder.user_name"            = data.coder_workspace.me.owner
  }
  home_volume_name = "home-volume"
}

resource "coder_agent" "main" {
  os   = "linux"
  arch = "amd64"
  
  startup_script = <<-EOT
    # Install common developer tools
    sudo apt-get update
    sudo apt-get install -y git vim curl wget jq
    
    # Set up dotfiles
    if [ -n "$DOTFILES_URI" ]; then
      git clone $DOTFILES_URI $HOME/dotfiles
      $HOME/dotfiles/install.sh
    fi
  EOT
  
  env = {
    DOTFILES_URI = data.coder_parameter.dotfiles_uri.value != "" ? data.coder_parameter.dotfiles_uri.value : null
  }
}

resource "coder_app" "code_server" {
  agent_id     = coder_agent.main.id
  slug         = "code-server"
  display_name = "VS Code"
  url          = "http://localhost:8080/?folder=/home/coder"
  icon         = "/icon/code.svg"
  subdomain    = false
}

resource "kubernetes_persistent_volume_claim" "home" {
  metadata {
    name      = "home-${local.workspace_name}"  # Using local variable
    namespace = var.workspaces_namespace
    labels    = local.labels  # Using local labels
  }
  
  spec {
    access_modes = ["ReadWriteOnce"]
    storage_class_name = "standard-rwo"
    
    resources {
      requests = {
        storage = "${var.home_disk_size}Gi"
      }
    }
  }
}

resource "kubernetes_pod" "main" {
  metadata {
    name      = local.workspace_name  # Using local variable
    namespace = var.workspaces_namespace
    labels    = local.labels  # Using local labels
  }
  
  spec {
    security_context {
      run_as_user = 1000
      fs_group    = 1000
    }
    
    # Add image pull secret if needed
    image_pull_secrets {
      name = "my-registry-credentials"
    }
    
    container {
      name  = "dev"
      image = var.image
      
      # Connect the agent to the pod
      command = ["sh", "-c", coder_agent.main.init_script]
      env {
        name  = "CODER_AGENT_TOKEN"
        value = coder_agent.main.token
      }
      
      # Resources
      resources {
        requests = {
          cpu    = "500m"
          memory = "500Mi"
        }
        limits = {
          cpu    = "${var.cpu_limit}"
          memory = "${var.memory_limit}Gi"
        }
      }
      
      # Storage
      volume_mount {
        mount_path = "/home/coder"
        name       = local.home_volume_name  # Using local variable
      }
    }
    
    # Add storage volume
    volume {
      name = local.home_volume_name  # Using local variable
      persistent_volume_claim {
        claim_name = kubernetes_persistent_volume_claim.home.metadata.0.name
      }
    }
  }
}

# User parameters
data "coder_parameter" "dotfiles_uri" {
  name        = "dotfiles_uri"
  display_name = "Dotfiles URI"
  description  = "Git URI for your dotfiles repository"
  type        = "string"
  default     = ""
  mutable     = true
}

# Output workspace info
output "workspace_info" {
  value = {
    name = data.coder_workspace.me.name
    owner = data.coder_workspace.me.owner
    pod_name = kubernetes_pod.main.metadata.0.name
  }
}
```

## Best Practices for Coder Templates

1. **Use variables and locals extensively**: Make templates customizable through variables and keep your code DRY with locals.
2. **Use consistent labeling**: Define labels in locals to ensure consistency across resources.
3. **Limit resource usage**: Set appropriate CPU and memory limits to prevent workspaces from consuming too many resources.
4. **Use persistent storage**: Ensure user data persists between workspace restarts.
5. **Security**: Run containers with non-root users and set appropriate Kubernetes security contexts.
6. **Parameterize templates**: Use `coder_parameter` to make templates configurable by end users.
7. **Understand HCL blocks and labels**: Be clear about the structure of HCL blocks, their labels, and how attributes work within blocks.

## Conclusion

This guide has provided the essential Terraform knowledge needed to work with Coder templates. By understanding these concepts, including HCL syntax, blocks, labels, variables, and locals, you can now create, customize, and maintain your own Coder templates for various development environments.

For more advanced templates and examples, explore the [Coder templates repository](https://github.com/coder/coder/tree/main/examples/templates).
