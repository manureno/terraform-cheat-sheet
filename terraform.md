
## Typical Terraform Project Structure

```
# Root module
LICENSE        -> A License to be used for the project
readme.md      -> documentation under Markdown format
terraform.tf   -> set TF version constraint, and provider version constraint
variables.tf   -> declare variables required in the project
main.tf        -> resource description
default.tfvars -> declare variables default values
outputs.tf     -> variables available as output
# Sub modules
modules/terraform_PROVIDER_NAME
    variables.tf   -> declare variables required in the project
    main.tf        -> resource description
    outputs.tf     -> variables available as output
```
Note about Modules  
A _Nested Module_ is a reference to another module from the current module, it can be of two types
* _Child Module_ if it is located externally
* _Sub Module_ if it is embedded in the current workspace

## Pass Variables to Terraform
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

##Terraform State
###State File
State file contain the infrastructure state as known by Terraform.  
It is a JSON file that should not be edited.   
It contains:
* **Resources**, which can be of type "data", ie a resource managed outside Terraform as a result of the query of a data block, or "managed" ie an actual resource.
* **Dependencies**, which is the dependency tree computed out of the Terraform script.

###Common Commands
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
terraform apply -replace RESOURCE_NAME
```
###Advanced Commands
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
terraform refresh
```

##Targeting Individual Resources
In some situations, such as issue during the application of a plan, or complex infrastructure divergence,
there ay be a need to manage resource individually.
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

* Format code to check syntax errors
```
terraform fmt
```

* Check configuration in the context of providers expectations
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
### Login to Terrafrom Cloud
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
* Re-running init once a cloud block has been added will migrate the stte in Terraform Cloud
```
terraform init
```
* Check in Terraform Cloud that the state has actually been copied and then delete the local state file.
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
  * Teraform variables
  * Environment variables 
