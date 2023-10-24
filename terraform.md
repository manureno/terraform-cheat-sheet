
### Typical Terraform Project Structure

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

### Pass Variables to Terraform
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