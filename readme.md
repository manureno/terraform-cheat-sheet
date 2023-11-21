Table of Content
* [Typical Project Structure](#typical-project-structure)
* [Typical Workflow](#typical-workflow)
* [Settings](#settings)
* [Providers](#providers)
  * [Providers Configuration](#provider-configuration)
  * [Lock File](#lock-file)
* [Working With Variables](#working-with-variables)
  * [Using Variables](#using-variables)
  * [Lists](#lists)
  * [Maps](#maps)
* [Publish Output Result](#publish-output-result)
* [Managing State](#managing-state)
  * [State File](#state-file)
  * [Common Commands](#common-commands)
  * [Advanced Commands](#advanced-commands)
* [Targeting Individual Resources](#targeting-individual-resources)
* [Manage Resource Drift](#manage-resource-drift)
* [Troubleshooting](#troubleshooting)
* [Terraform Cloud](#terraform-cloud)
  * [Login to Terraform Cloud](#login-to-terraform-cloud)
  * [Migrate State to Terraform Cloud](#migrate-state-to-terraform-cloud)
  * [Infrastructure Management Workflows](#infrastructure-management-workflows)
  * [Variables](#Variables)

## Typical Project Structure

```
# Root module
LICENSE          -> A License to be used for the project
README.md        -> documentation under Markdown format
terraform.tf     -> set TF version constraint, and provider version constraint
variables.tf     -> declare variables required in the project
main.tf          -> resource description
terraform.tfvars -> declare input variables loaded by default
*.auto.tfvars    -> declare input variables loaded by default
myconfig.tfvars  -> variables input loaded with option -var-file="myconfig.tfvars"
outputs.tf       -> variables available as output
# Sub modules
modules/nestedModuleA
    README.md      -> Documentation under markdown format
    variables.tf   -> declare variables required in the project
    main.tf        -> resource description
    outputs.tf     -> variables available as output
```
Note about Modules  
A _Nested Module_ is a reference to another module from the current module, it can be of two types
* _Child Module_ if it is located externally
* _Sub Module_ if it is embedded in the current workspace

## Typical Workflow
* Create a terraform configuration
* Download providers, create the intermediate working files
```
terraform init
``` 
* Reformat Terraform scripts with standard indentation
```
terraform fmt
```
* Check syntax and consistency vs providers
```
terraform validate
```
* Create a plan, optionally save it in a file
```
terraform plan [-out=path/to/plan.file]
```
* Review and apply the plan, optionally passing a plan file
```
terraform apply [path/to/plan.file]
```
* Destroy infrastructure
```
terraform destroy
```
## Settings
* A configuration block that describes 
  * terraform version requirement
  * provider version requirements
  * Terraform cloud or backend requirement
```
terraform {
  # Terraform version requirement
  # "~> 1.6.1" : allow only the rightmost version to increment (like 1.6.2)
  # ">= 1.6.1" : allow any version superior or equals
  required_version = ">= 1.6.1"
  
  # Provider requirement
  required_providers {
    aws = {
      version = ">= 2.7.0"
      source = "hashicorp/aws"
    }
  }
  
  # Terraform Cloud configuration
  cloud {
    organization = "example_corp"
  }
  
  # As an alternative to TF Cloud, a backend may be used to store remote state
  #backend "http" {
  #  address = "http://mybackend.com/foo"
  #  lock_address = "http://mybackend.com/foo"
  #  unlock_address = "http://mybackend.com/foo"
  #}
}
```
## Providers
### Provider Configuration
* Once imported in the settings block, a provider may need a configuration before being used
  * _Inform the AWS provider to work on the eu-west-3 region :_
```
provider "aws" {
  region  = "eu-west-3"
}
```
* aliases may be given to provider configurations so several 
provider configurations can be used in the same Terraform script.
```
# can be referenced as aws.paris
provider "aws" {
  alias  = "paris"
  region = "eu-west-3"
}
# can be referenced as aws.ireland
provider "aws" {
  alias  = "ireland"
  region = "eu-west-1"
}
```

* Select an alternate provider configuration
```
resource "aws_instance" "foo" {
  provider = aws.paris
  # ...
}
```

### Lock File
* Lock File Change scenarios
  * Dependency on new provider, includes version/contraints/hashes
  * New version of an existing provider
  * New provider package checksums (when new hashes schemes are added in the repository)
  * Provider no longer required 

## Working With Variables
### Using Variables
* Declare variables in a .tf file
```
variable "var_name" {
  description = "variable used for test"
  type        = string
  default     = "test"
}

``` 
* Pass a variable from command line
```
terraform COMMAND -var="variable_name=value"
```

* Initialize variables in a ".tfvars" file
```
...
variable-name      = "value"
...
```
* Pass variables from a file
```
terraform COMMAND -var-file="variables.tfvars"
```

* Pass a variable from an environment variable
```
export TF_VAR_variable_name=value 
terraform COMMAND
```
### Lists
* Declare a list 
```
variable "availability_zone_names" {
  type    = list(string)
  default = ["us-west-1a", "us-west-1c"]
}
```

* Access list values by index
```
output "a_zone" {
  value = var.availability_zone_names[0]
}
```

### Maps
* Declare a Map
```
variable "regions_map" {
  type = map(string)
  default = {
    paris   = "eu-west-3"
    ireland = "eu-west-1"
  }
}
```

* Static access to map values 
```
output "region_ireland" {
  value = var.regions_map["ireland"]
}
```

* Dynamic access to map value
```
output "region_alias" {
  value = lookup(var.regions_map, var.alias, "Unknown alias")
}
```



## Publish Output Result
* Add the desired output in a .tf file _(like output.tf)_
```
output "output_name" {
  description = "ID of the EC2 instance"
  # value references a resource field from the configuration
  value       = aws_instance.app_server.id 
}
```
* List all outputs (after configuration has been applied) 
```
terraform output
```
* List one output (after configuration has been applied) 
```
terraform output output_name
```

## Managing State
### State File  
State file terraform.tfstate contain the infrastructure state as known by Terraform.  
It is a JSON file that should not be edited.   
It contains:
* **Resources**, which can be of type "data", ie a resource managed outside Terraform as a result of the query of a data block, or "managed" ie an actual resource.
* **Dependencies**, which is the dependency tree computed out of the Terraform script.

### Common Commands  
* Show state file :
```
terraform show
```
* List Resources :
```
terraform state list
```
* Override state to explicitely re-create a resource 
```
terraform plan -replace="RESOURCE_NAME"
```
```
terraform apply -replace="RESOURCE_NAME"
```
### Advanced Commands
* Copy a resource from one state to another
```
terraform state mv -state-out=TARGET_STATE_FILE SOURCE_RESOURCE_NAME TARGET_RESOURCE_NAME
```
* Remove a resource from state file
```
terraform state rm RESOURCE_NAME
```
* Import an external resource in the state file
```
terraform import RESOURCE_NAME RESOURCE_ID
```
* Update the state file to match the actual infrastructure
```
terraform apply -refresh-only
```

## Targeting Individual Resources  
In some situations, such as issue during the application of a plan, or complex infrastructure divergence,
there may be a need to manage resource individually.  
NB: This is an advanced usage that need extreme care to end-up with a consistent configuration/state/infrastructure 

* Apply change on a resource
```
terraform apply -target="RESOURCE_NAME"
```

* Destroy a resource
```
terraform destroy -target="RESOURCE_NAME"
```
## Manage Resource Drift
* Check the drift between state and resources
```
terraform plan -refresh-only
```
* Update state file to reflect resource actual state
```
terraform apply -refresh-only
```
* Associate a resource configuration with an existing one (update the state file)
  * RESOURCE_NAME is the name in the .tf file,  RESOURCE_ID is the existing infrastructure provider ID
```
terraform import RESOURCE_NAME RESOURCE_ID
```

## Troubleshooting
4 types of issues
* Language errors in the HCL scripts
* State errors in the state file
* Core errors in the Terraform application itself
* Provider errors


* Format code
```
terraform fmt
```

* Check syntax and configuration in the context of providers expectations
```
terraform validate
```
* Enable logging with levels in (TRACE, DEBUG, INFO, WARN or ERROR)
```
export TF_LOG_CORE=TRACE
```
```
export TF_LOG_PROVIDER=TRACE
```
* Save logs in a file
```
export TF_LOG_PATH=logs.txt
```

## Terraform Cloud
### Login to Terraform Cloud
* Login command will open Terraform Cloud page in a browser in order to generate an authentication token
  * Once logged in, the token is stored in a "credentials.tfrc.json" file (exact location depends on platform), and can 
  be reused for subsequent logins. 
```
terraform login
```

### Migrate State to Terraform Cloud

* Migrating a state in Terraform Cloud requires a "cloud" block be declared 
in the configuration
```
terraform {
  cloud {
    organization = "ORGANIZATION-NAME"
    workspaces {
      name = "learn-terraform-cloud-migrate"
    }
  }
  ...
}
```
* Re-running init once a cloud block has been added will migrate the state in Terraform Cloud
```
terraform init
```
* Check within Terraform Cloud that the state has actually been copied and then delete the local state file.
```
rm terraform.tfstate
```
* Subsequent commands (apply, destroy, etc.) will now rely on remote state 
```
terraform apply
```

### Infrastructure Management Workflows
* 3 workflows can be used in Terraform Cloud
  * CLI  : to manually manage the infrastucture
  * VCS driven : to automate infrastructure management 
  * API driven : to automate with the help of 3rd party tools

* Notes about VCS-driven
  * A Pull Request to "master" branch triggers a "speculative plan" to be reviewed before merge
  * A merge on master branch triggers a run to apply the new configuration 

### Variables
* Variables can be defined either on a per-workspace basis, or across workspaces by grouping
them into reusable "variable sets"
* Variables can be from 2 types
  * Terraform variables
  * Environment variables 

## Deprecated commands
* taint to be replaced with 
```
terraform apply -replace="resource_name" 
```
* refresh to be replace with
```
terraform apply -refresh-only
```

## Other Commands
* Unlock a state file stored on a remote backend
```
terraform force-unlock LOCK_ID
```

