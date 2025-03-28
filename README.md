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
## Data Sources
## Variables
## Commands
## Shared State
## Modules
## Workspaces
## Best Practises
## LocalStack
## Checkov
## Infracost
