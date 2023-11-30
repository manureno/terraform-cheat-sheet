Table of Content
* [Typical Project Structure](#typical-project-structure)
* [Typical Workflow](#typical-workflow)
  * [Retrieve Modules](#retrieve-modules) 
* [Settings](#settings)
* [Providers](#providers)
  * [Providers Configuration](#provider-configuration)
  * [Lock File](#lock-file)
  * [Manually Locking Providers](#manually-locking-providers)
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
* [Data Sources](#data-sources)
  * [Retrieve Data](#retrieve-data)
  * [Use Data](#use-data)
* [Modules](#modules) 
* [Troubleshooting](#troubleshooting)
* [CLI Workspaces](#cli-workspaces)
* [Backends](#backends)
  * [Local Backend](#local-backend)
* [Terraform Cloud](#terraform-cloud)
  * [Cloud Workspaces](#cloud-workspaces)
  * [Login to Terraform Cloud](#login-to-terraform-cloud)
  * [Migrate State to Terraform Cloud](#migrate-state-to-terraform-cloud)
  * [Infrastructure Management Workflows](#infrastructure-management-workflows)
  * [Variables](#Variables)
* [Sentinel](#Sentinel)
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
# Sub modules (aka nested modules)
modules/nestedModuleA
    README.md      -> Documentation under markdown format
    variables.tf   -> declare variables required in the project
    main.tf        -> resource description
    outputs.tf     -> variables available as output
```
Terraform configuration are most of the time expressed using the HCL language in ".tf" files. JSON can be used as 
an alternative in files named ".tf.json".

Note about Modules  
* _Root Module_ : Resources defined in the .tf files in the main working directory.
* _Nested Module_ : Module part of a configuration, located under ./modules
* _Child Module_ :  A module that is called by anothrr module is called a child module

## Typical Workflow
* Create a Terraform configuration
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
_NB : plan, apply and destroy analyze resource dependencies with a parallel processing of 10 which can be 
increased or decreased:_
```
terraform [plan|apply|destroy] -parallelism=n
```

* The init command expects to find a configuration in its current directory, it can also be used from 
an empty directory by specifying the location of the root module :
```
terraform init -from-module path/to/root/module
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
  # NB: Provider block may be left empty
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
* Provider versions required by a configuration is computed during th init and stored in a .terraform.lock.hcl file
* Lock File Change scenarios
  * Dependency on new provider, includes version/contraints/hashes
  * New version of an existing provider
  * New provider package checksums (when new hashes schemes are added in the repository)
  * Provider no longer required 

### Manually Locking Providers
Provider versions are usually locked as a side effect of terraform init.  
There are some situations, such as using alternative providers installation (such as filesystem or network mirror) that 
may require to lock provider versions manually.  
This can be done with :
```
terraform providers lock
```

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
* Variable types : string, number, bool, list, set, map
  * _NB: When no type is specified, Terraform accepts any type_
  

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
* Show the details of a given resource 
```
terraform state show RESOURCE_TYPE.RESOURCE_NAME
```
* Override state to explicitly re-create a resource 
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

## Data Sources
### Retrieve Data
Data sources allow Terraform to use information defined outside of Terraform
Each provider may offer data sources alongside its set of resource types.
* Find the latest available AMI that is tagged with Component = web:
```
data "aws_ami" "web" {
  filter {
    name   = "state"
    values = ["available"]
  }
  filter {
    name   = "tag:Component"
    values = ["web"]
  }
  most_recent = true
}
```
### Use Data
Results from Data Source queries can be re-injected in resource definitions
* Create an EC2 instance using the latest web AMI ID retrieved from a Data Source call: 
```
resource "aws_instance" "web" {
  ami           = data.aws_ami.web.id
  instance_type = "t1.micro"
}
```

## Modules
Module : a group of related resources

* Reference module source from public terraform registry : ```<NAMESPACE>/<NAME>/<PROVIDER>```  
Ex: use the vpc module from the aws provider :
```
module "my_vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "3.18.1"
  
  cidr = var.vpc_cidr
  ...
}
```

* Reference a private registry module source : ```<HOSTNAME>/<NAMESPACE>/<NAME>/<PROVIDER>```
```   
module "vpc" {
  source  = "app.terraform.io/example_corp/vpc/aws"
  version = "1.0.4"
}
``` 

### Retrieve Modules

* Module retrieval is a part of the initialization executed by
```
terraform init
```
* Modules can also be retrieved or upgraded explicitly without all the other init steps using :
```
terraform get
```
* Once the module downloaded, they can be upgraded with either init or get 
```
terraform init -upgrade
```
```
terraform get -upgrade
```

## Troubleshooting
4 types of issues
* Language errors in the HCL scripts
* State errors in the state file
* Core errors in the Terraform application itself
* Provider errors


* Reformat code in the current dir according to Terraform conventions (2 space indent, etc).  
_NB: use -recursive to process sub-directories_
```
terraform fmt [-recursive]
```
* Reformat code and output the diff
```
terraform fmt -diff
```
* Check the format without modifying the code 
```
terraform fmt -check
```

* Validate syntax and configuration
```
terraform validate
```
* Validate syntax and configuration and generate a machine-readable Json output, 
useful for example in a CI/CD workflow
```
terraform validate -json
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
## CLI Workspaces
CLI workspaces isolate multiple state files in the same working directory.  
_NB: The Terraform CLI does not require to create CLI workspaces._  
* List all workspaces
```
terraform workspace list
```
* Create a workspace
```
terraform workspace new WORKSPACE_NAME
```
* Change the current workspace
```
terraform workspace select WORKSPACE_NAME
```
* Workspaces are stored in the terraform.tfstate.d directory
## Backends
Backend is where Terraform store the state file.  
A single backend should be used by a Terraform config, it is configured as part of the "terraform init" phase.
A backend block cannot refer to Environment or Input variables, instead they can be configured through -backend-config option:
```
terraform init -backend-config="KEY=VALUE"
```
```
terraform init -backend-config=/path/to/config/file
```
### Local Backend
A local backend is used by default to store the state file in the current directory
* Local backend can be configuredin te settings block
```
terraform {
  backend "local" {
    path = "relative/path/to/terraform.tfstate"
    workspace_dir = "relative/path/to/non-default/workspaces"
  }
}

```
## Terraform Cloud
### Login to Terraform Cloud
* Login command will open Terraform Cloud page in a browser in order to generate an authentication token
  * Once logged in, the token is stored in a "credentials.tfrc.json" file (exact location depends on platform), and can 
  be reused for subsequent logins. 
```
terraform login
```

### Cloud Workspaces

Terraform Cloud workspaces are required: they represent all of the collections of infrastructure in an organization.  
You cannot manage resources in Terraform Cloud without creating at least one workspace.

_NB: They differ in that from CLI workspaces._

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


## Sentinel
Sentinel is Policy as Code tool that allows to write policies in a high-level language
and benefit from proven software development best practices such as versioning, automation, testing.  
_NB : It is available in Terraform Enterprise._

* Sentinel rules are applied after terraform plan and before terraform apply

* 3 enforcement levels can be defined to implement pass/fail behavior
  * Advisory
  * Soft Mandatory
  * Hard Mandatory