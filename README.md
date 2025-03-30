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
## Commands
## Shared State
## Modules
## Workspaces
## Best Practises
## LocalStack
## Checkov
## Infracost
