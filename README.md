# Terraform DevOps Essentials
## If I'll find any other feature to be useful - it'll surely be added!
## Infrastructure as Code \(IaC\)
IaC is a practice of managing infrastructure \(servers, networks, databases, etc.\) through code rather than manual operations. The infrastructure is defined in configuration files that can be versioned, tested, and automated. It keeps infrastructure consistent in every environment \(e.g. dev, test, prod\), allows teams to collaborate with code rather than manual operations and supports idempotency - infrastructure stays the same after running code multiple times \(no unwanted changes\).

Terraform is an open-source tool by HashiCorp that allows for implementing IaC practise with declarative approach \(code describes what you want to achieve, not how you want to achieve it\). It supports many platforms \(e.g. AWS, Azure, GCP, Kubernetes\), allows for proficient teamwork by sharing infrastructure state, carefully describes infrastructure changes when applying new version and allows for modular architecture by reusable components.
## How it works?
Basicly - write a configuration file \(.tf\) and run appropriate commands to apply your infrastructure! How to do it? See below.
## Providers
Provider is a plugin that integrates Terraform with a specific platform or service \(e.g. AWS, Azure, Kubernetes, GitHub\). It defines a set of resources and data sources that Terraform can manage.\
Providers are responsible for translating HCL configurations to API calls, resource configuration validation, infrastructure lifecycle management, authentication management.\
You can use official providers from HashiCorp \(e.g. hashicorp/aws, hashicorp/azurerm, hashicorp/google\), certified partner providers \(e.g. Datadog, Cloudflare\) and community providers \(e.g. elasticsearch, rabbitmq\).

To define provider in your configuration you must declare which providers are required and from where they should be downloaded.\
To do that, use **required_providers** block:
```
terraform {
  required_providers {
    <provider_name> = {
      source  = <provider_source>
      version = <provider_version>
    }
  }
}
```
After that, you can define provider with **provider** block:
```
provider <provider_name> {
  # Configuration is specific to provider, e.g. for AWS provider
  region     = <region>
  access_key = var.<aws_access_key> # Authentication (usually via environment variables to not hardcode sensitive data)
  secret_key = var.<aws_secret_key>
}
```
To use the same provider but with different configurations use **aliases**, e.g. for different AWS regions:
```
provider "aws" {
  alias  = "europe"
  region = "eu-west-1"
}

provider "aws" {
  alias  = "usa"
  region = "us-east-1"
}

resource "aws_s3_bucket" "bucket" {
  provider = aws.usa
  bucket   = "my-bucket"
}
```
Best practises:
* **Always pin the version** in required_providers to avoid breaking changes \(e.g. ```version = "~> 5.0" # Allows 5.x but not 6.0```\)
* **NEVER HARDCODE CREDENTIALS**, use environment variables or configuration files \(e.g. ~/.aws/credentials\) or dynamic credentials \(e.g. AWS STS\) instead
* **Avoid configuring providers in modules** – pass them via providers in the module call, e.g.
  ```
  module "vpc" {
  source = "./modules/vpc"
  providers = {
    aws = aws.europe  # Use provider with alias
    }
  }
  ```
## Resources
Resource is the fundamental building block of infrastructure in Terraform. It represents a specific object in a cloud or service that you want to create, modify, or delete \(e.g. an virtual server, a managed database, a firewall rule\).\
Each resource defines:
* **Type** - type of resource \(e.g. aws_instance\)
* **Name** - unique name used for references within Terraform
* **Arguments** - resource-specific configuration \(e.g. ami, instance_type for EC2\)
* **Meta-arguments** - special parameters controlling the behavior of the resource \(e.g. count, lifecycle\)
* **Attributes** - output data \(available after creation\), see Outputs

To define resource use **resource** block:
```
resource <type> <name> {
  # Arguments
  <argument1> = <value1>
  <argument2> = <value2>

  # Meta-arguments (optional)
  depends_on  = [resource.<resource_name>]      # forces creation order
  count       = <instance_number>               # creates multiple, identical resources
  for_each    = var.<items>                     # creates resource from map/set by iterating over them
  provider    = <provider_alias>                # alternative provider using alias
  lifecycle {                                   # lifecycle policies
    prevent_destroy = <true_or_false>           # blocks accidental deletion
    ignore_changes  = [<tag1>, <tag2>, ...]     # ignore tag changes \(e.g. after manual editing\)
    create_before_destroy = <true_or_false>     # zero downtime replacement
  }                             
}
```
Dependencies are automatically detected by Terraform but you can force them, e.g.
```
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "example" {
  vpc_id     = aws_vpc.main.id # explicit dependency
  cidr_block = "10.0.1.0/24"
}
```
To construct repeated nested block arguments dynamically in resource-type blocks use **dynamic** block:
```
resource <type> <name> {
  # body of the resource block

  dynamic  <label>  {                            # specifies the kind of repeated nested block to generate
    for_each = <complex_value_to_iterate_over>   # specifies the complex value (common collections used are either list or map) to iterate over
    iterator = <iterator_name>                   # optional, specifies a name that represents the current element of the complex value being iterated over

    content {
      # body of the dynamic block generated, e.g.
      from_port   = <iterator_name>.<from_port>
      to_port     = <iterator_name>.<to_port>
      protocol    = <protocol>
    }
  }
}
```
Best practises:
* **Use unique resource names**
* **Place reusable resources in modules**
* **Use variables and outputs**
* **Always remember about shared state**
* **Use ```terraform plan``` before applying**
## Data Sources
Data Source is a mechanism that allows you to read information about existing resources \(created manually or by other Terraform code\) without managing them which is crucial when integrating with manually managed or managed by other systems resources with your IaC code.\
They are read-only and fetched during execution of ```terraform apply```.

To define data source use **data** block:
```
data <provider_type> <name> {
  # Search arguments
  <argument1> = <value1>
  <argument2> = <value2>

  # Search filters
  filter {
    name   = <key>
    values = [<value1>, <value2>]
  }
}
```
To use data source in resources refer to it via data.\<provider_type\>.\<name\>.\<attribute\>, e.g.
```
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id  # use AMI from data source
  instance_type = "t3.micro"
}
```
Dependencies are automatically detected by Terraform, if data source is dependent on some resource, Terraform will create this resource first.

Best practises:
* **Use Data Sources instead of hardcoding values and for dynamic resource search**
* **Use ```try()```** to catch errors
## Variables
Variables are used to parameterize the configuration, they allow for:
* dynamically passing values \(e.g. cloud region, instance size\)
* avoiding hardcoding sensitive data \(API keys, passwords\)
* creating reusable modules and environments \(dev/prod\)

Variables consist of:
* **type** - variable type, they are divided on simple \(e.g. string, number, bool\) and complex \(list, set, map, object, any - avoid any if possible\)
* **default** - optional, default value \(unless other specified\)
* **description** - documentation
* **sensitive** - hides value in outputs \(e.g. passwords\)
* **validation** - value validation rules

Define your variables in **variables.tf** using **variable** block:
```
variable <name> {
  type        = <type>
  default     = <default_value>
  description = <description>
  sensitive   = <true_or_false>
  
  validation {
    condition     = <condition>
    error_message = <error_message>
  }
}
```

There are several ways to pass variable values to configuration files:
* **Use ```-var=<name>=<value>``` option in commands**
* **Use .tfvars file with ```-var-file=<var_file>``` option in commands**
* **Use environment variables in TF_VAR_\<variable_name\> format**
* **Use default values from variable declarations**\
First 2 methods have highest priority, then third, then fourth.

To reference variable in configuration files use ```var.<variable_name>```.
## Outputs
Outputs are a mechanism for sharing read-only information about the deployed infrastructure with other modules and resources, users \(e.g. using ```terraform apply```\), or external systems \(e.g. CI/CD\).

To define output use **output** block:
```
output <name> {
  value       = <expression>                    # required, specifies information to export
  description = <documentation>                 # optional
  sensitive   = <true_or_false>                 # optional, false as default
  depends_on  = [resource.<resource_name>]      # forces creation order
}
```
To reference output use ```module.<module_name>.<output_name>```.

Best practises:
* **Use outputs to link modules**
* **Export only what is really needed**
## Shared State
State \(stored in the terraform.tfstate file\) is a map of the resources managed by Terraform, containing:
* mappings of resources defined in code to real objects in the cloud or other platform
* metadata \(e.g. IP addresses, configurations\)
* dependencies between resources

Shared state means that terraform.tfstate file is stored remotely \(e.g. AWS S3\) and not locally. This allows for synchronizing the work of team members, eliminating conflicts and inconsistencies by storing the current state.

To define remote storage for shared state in configuration use **backend** block, e.g. AWS S3 + DynamoDB configuration:
```
terraform {
  backend <name> {
    bucket         = <bucket_name>
    key            = <path_to_state_file>
    region         = <region>
    dynamodb_table = <table_name>          # DynamoDB table for locking
    encrypt        = <true_or_false>       # state encryption
  }
}
```

To prevent user's race condition \(using ```terraform apply``` simultaneously, which could corrupt the state\) use locking. E.g. when using AWS S3 + DynamoDB for remote state storage Terraform inserts record in DynamoDB for the duration of the operation that locks state changes.

Best practises:
* **ALWAYS use remote state storage**
* **Encrypt state to prevent sensitive data being compromised**
* **Isolate different environments and components \(VPC, databases\) state files** by configuring different backends for them
* **Use state versioning** to be able to roll back previous states
## Commands
Essential commands:
* **terraform init** - initializes working directory, downloads the required providers \(e.g. AWS\) and modules, configures the backend for the state
  * **terraform init -upgrade** - updates provider version
  * **terraform init -backend-config** - backend configuration \(e.g. for remote state\)
* **terraform plan** - generates an execution plan – shows what will be added/changed/destroyed without applying any changes
  * **terraform plan -out=\<plan_file\>** - saves plan to a file \(e.g. tfplan\) to use later in apply
  * **terraform plan -var=\<name\>=\<value\>** - passes variables
  * **terraform plan -var-file=\<var_file\>** - specifies variable file
  * **terraform plan -destroy** - generates a plan for the destruction of infrastructure
* **terraform apply** - applies changes in infrastructure
  * **terraform apply \<plan\>** - applies previously generated plan
  * **terraform apply -auto-approve** - skips interactive confirmation
  * **terraform apply -var-file=\<var_file\>** - specifies variable file
  * **terraform apply -parallelism=\<parallel_operations_number\>** - limits parallel operations, default number is 10
* **terraform destroy** - destroys all resources defined in the configuration
* **terraform state** - state management commands
  * **terraform state list** - displays a list of all resources in the state
  * **terraform state show \<resource_name\>** - shows details of a specified resource
  * **terraform state mv \<old_name\> \<new_name\>** - changes the name of a resource in the state
  * **terraform state rm \<resource_name\>** - removes resource from a state without destroying it in the infrastructure
  * **terraform state pull** - pulls and displays the raw state
* **terraform refresh** - synchronizes state with real infrastructure
* **terraform get** - downloads and updates modules
  * **terraform get -update** - forces modules update
* **terraform validate** - checks the syntax of .tf files \(useful in CI/CD\)
* **terraform fmt** - automatically formats configuration files to standard style
  * **terraform fmt -recursive** - formats subdirectories too
  * **terraform fmt -diff** - shows changes before committing
  * **terraform fmt -check** - checks if files are formatted \(useful in CI/CD\)
* **terraform console** - launches an interactive expression testing console
* **terraform workspace** - workspaces management commands
  * **terraform workspace new <name>** - creates a new workspace
  * **terraform workspace select <name>** - switches to existing workspace
  * **terraform workspace delete <name>** - removes workspace
  * **terraform workspace list** - displays a list of all workspaces
* **terraform import \<resource_name\> \<external_id\>** - imports an existing resource \(e.g. created manually\) to the state
* **terraform output** - displays output values \(e.g. from outputs.tf\)
  * **terraform output -json** - formats output to JSON
* **terraform taint \<resource_name\>** - marks specified objects in the state as tainted \(resources will be deleted and recreated on the next apply\)

Best practises:
* **Use plan before apply** to avoid unpleasant surprises
* **NEVER change the state manually** - use the commands provided for this
## Modules
Module is a self-contained, reusable component that groups related resources, variables, and outputs. It can be thought of as a "function" in programming.

Example module directory structure:
```
modules/
└── <module_name>/
    ├── main.tf          # main resource configuration
    ├── variables.tf     # input variables
    ├── outputs.tf
    └── README.md
```

Modules can have different sources:
* **Local** - simple, no versioning, use with ```source = "./modules/<module_name>"```
* **Terraform Registry** - public, ready-made solutions, with versioning, e.g. use with ```source = "terraform-aws-modules/<path_to_module>"```
* **Git** - fully controlled and versioned, e.g. use with ```source = "git::https://github.com/<user_or_organization>/<repo_name>.git//modules/<module_name>?ref=<module_version>"```
* **S3/GCS/HTTP** - fully controlled and versioned, e.g. use with ```source = "s3::https://s3-<region>.amazonaws.com/<bucket-name>/modules/<module_name>.zip"```

To use modules use **module** block in main configuration:
```
module <module_name> {
  source = <module_source>
  version = <module_version>
  
  # Input variables values
  <variable1>         = <value1>
  <variable2>         = [<value2>, <value3>]
}

# Using module output example
resource <type> <name> {
  subnet_id = module.prod_vpc.public_subnet_ids[0]  # "public_subnet_ids" module output "public_subnet_ids"
}
```

Best practises:
* **Design single responsibility modules**
* **Isolate state for each module**
## Workspaces
Workspace is a mechanism that allows you to store separate states \(tfstate\) and variable values for the same configuration. This allows for isolating different environments without duplicating code.

Use earlier described commands to manage workspaces.

You can use workspace-specific values of variables to prepare proper environments.\
Secure production workspace from accidental deletion.\
If the environments are too different, use separate configurations rather that workspaces.
## LocalStack
LocalStack is described [here](https://github.com/WallyS02/AWS-DevOps-Essentials?tab=readme-ov-file#localstack).

To use Terraform with LocalStack use provider configuration:
```
provider "aws" {
  region                      = "us-east-1"  # LocalStack requires region but ignores it
  access_key                  = "mock"       # dummy credentials
  secret_key                  = "mock"
  skip_credentials_validation = true         # skipping credential validation
  skip_requesting_account_id  = true
  skip_metadata_api_check     = true
  s3_use_path_style           = true         # required for LocalStack

  endpoints {
    s3      = "http://localhost:4566"
    dynamodb = "http://localhost:4566"
    # Add endpoints for other services
  }
}
```
With provider configured as shown above, deploy infrastructure with Terraform as normally.
## Checkov
Checkov is an open-source static code analysis tool \(SAST\) for IaC which:
* detects security vulnerabilities and misconfigurations in Terraform configuration files
* checks compliance with best practices and standards \(CIS, GDPR, HIPAA, PCI-DSS\)
* integrates with CI/CD tools

Checkov analyzes .tf or .tfplan files and compares configuration with over 2,500 built-in rules. After analysis it returns PASSED/FAILED check list with description and documentation url.

To use Checkov:
1. Install Checkov with ```pip install checkov```, install Python and Pip as prerequisite
2. Scan Terraform directory with ```checkov -d /path/to/terraform``` or Terraform plan with ```checkov -f tfplan.json```
3. See possible vulnerabilities detected by Checkov and fix them

You can define custom rules with YAML or Python, e.g.
```
metadata:
  name: ENSURE_LAMBDA_IN_VPC
  id: CUSTOM_001
category: NETWORKING
scope:
  provider: aws
  resource: aws_lambda_function
definition:
  cond_type: attribute
  resource_types: ["aws_lambda_function"]
  attribute: "vpc_config"
  operator: not_equals
  value: null
```
## Infracost
Infracost is an open-source tool for estimating the costs of cloud IaC which:
* predicts monthly AWS/Azure/GCP resource costs based on Terraform configuration files
* helps avoiding financial surprises after deployment
* integrates with CI/CD tools

Infracost parses .tf or .tfplan files to identificate resources and using official AWS/Azure/GCP pricing APIs returns a detailed monthly cost broken down by service.

To use Infracost:
1. Install Infracost with ```curl -fsSL https://raw.githubusercontent.com/infracost/infracost/master/scripts/install.sh | sh```
2. Configure free API key with ```infracost register```
3. To parse Terraform directory use ```cd /path/to/terraform``` and ```infracost breakdown --path .``` to analyse costs of current configuration
4. See cost raport

To check how code changes will affect costs \(e.g. in a pull request\) use ```infracost diff --path . --compare-to main```.
