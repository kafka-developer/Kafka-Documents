# Terraform Resources — Expanded Notes with Commands

## 1. What is a Resource?

A **resource** is any infrastructure object Terraform can create, update, delete, and track.

Examples:

```hcl
resource "aws_instance" "web" {
  ami           = "ami-123456"
  instance_type = "t2.micro"
}

```

Resource examples:

```text
Virtual machines
Virtual networks
Subnets
Security groups
DNS records
Load balancers
Kafka connectors
Kafka topics
Databases
Storage buckets
Kubernetes objects

```

Terraform manages resources through:

```text
configuration file  → desired state
state file          → Terraform’s tracking record
real infrastructure → actual object in cloud/platform

```

----------

## 2. Resource Types Depend on Providers

Terraform itself does not know how to create an Amazon Web Services instance, Kafka topic, Kubernetes deployment, or DNS record.

The **provider** gives Terraform those resource types.

Example:

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

```

Then you can use Amazon Web Services resources:

```hcl
resource "aws_s3_bucket" "logs" {
  bucket = "my-log-bucket"
}

```

Without the provider, Terraform does not understand:

```hcl
aws_s3_bucket
aws_instance
aws_vpc

```

----------

## 3. What is a Provider?

A **provider** is a Terraform plugin that talks to an external platform’s Application Programming Interface.

Examples:

```text
Amazon Web Services provider
Azure provider
Google Cloud Platform provider
Kubernetes provider
Confluent provider
Datadog provider
GitHub provider
VMware provider

```

Provider configuration example:

```hcl
provider "aws" {
  region = "us-east-1"
}

```

Confluent-style example:

```hcl
provider "confluent" {
  cloud_api_key    = var.confluent_cloud_api_key
  cloud_api_secret = var.confluent_cloud_api_secret
}

```

Provider job:

```text
Terraform says: create this resource
Provider says: I know how to call the platform Application Programming Interface
Platform creates/updates/deletes the object
Terraform records result in state

```

----------

## 4. Add a Resource Block

Basic format:

```hcl
resource "<resource_type>" "<local_name>" {
  argument1 = value
  argument2 = value
}

```

Example:

```hcl
resource "aws_instance" "app_server" {
  ami           = "ami-123456"
  instance_type = "t3.micro"
}

```

Resource address:

```bash
aws_instance.app_server

```

That address is used in commands like:

```bash
terraform state show aws_instance.app_server
terraform taint aws_instance.app_server
terraform state rm aws_instance.app_server

```

For `for_each`:

```hcl
resource "aws_instance" "app" {
  for_each = toset(["dev", "qa", "prod"])

  ami           = "ami-123456"
  instance_type = "t3.micro"
}

```

Address examples:

```bash
aws_instance.app["dev"]
aws_instance.app["qa"]
aws_instance.app["prod"]

```

----------

## 5. Resource Arguments and Meta-Arguments

### Resource-specific arguments

These are specific to the resource type.

Example:

```hcl
resource "aws_instance" "web" {
  ami           = "ami-123456"
  instance_type = "t3.micro"
}

```

Here:

```text
ami
instance_type

```

are specific to `aws_instance`.

Kafka-style example:

```hcl
resource "confluent_kafka_topic" "orders" {
  kafka_cluster {
    id = confluent_kafka_cluster.basic.id
  }

  topic_name       = "orders"
  partitions_count = 6
}

```

Here:

```text
topic_name
partitions_count
kafka_cluster

```

are provider/resource-specific.

----------

### Terraform meta-arguments

Meta-arguments are not specific to one provider. Terraform itself uses them to control resource behavior.

Common ones:

```hcl
depends_on
count
for_each
provider
lifecycle

```

Example with `depends_on`:

```hcl
resource "aws_instance" "web" {
  ami           = "ami-123456"
  instance_type = "t3.micro"

  depends_on = [aws_security_group.web_sg]
}

```

Example with `count`:

```hcl
resource "aws_instance" "web" {
  count = 3

  ami           = "ami-123456"
  instance_type = "t3.micro"
}

```

Addresses:

```bash
aws_instance.web[0]
aws_instance.web[1]
aws_instance.web[2]

```

Example with `for_each`:

```hcl
resource "aws_s3_bucket" "buckets" {
  for_each = toset(["logs", "backup", "archive"])

  bucket = "my-${each.key}-bucket"
}

```

Example with `lifecycle`:

```hcl
resource "aws_instance" "web" {
  ami           = "ami-123456"
  instance_type = "t3.micro"

  lifecycle {
    prevent_destroy = true
  }
}

```

Another lifecycle example:

```hcl
lifecycle {
  create_before_destroy = true
}

```

Exam line:

```text
Arguments configure the resource.
Meta-arguments configure Terraform’s behavior around the resource.

```

----------

## 6. Initialize a New Terraform Project

When starting a new Terraform project, run:

```bash
terraform init

```

This does:

```text
Downloads providers
Downloads modules
Initializes backend
Creates .terraform directory
Creates or updates .terraform.lock.hcl

```

Typical workflow:

```bash
terraform init
terraform validate
terraform plan
terraform apply

```

Useful command:

```bash
terraform providers

```

Shows providers used by the configuration.

----------

## 7. Reinitialize After Provider or Module Changes

If you change providers, provider versions, modules, or backend configuration, run:

```bash
terraform init

```

If provider versions changed:

```bash
terraform init -upgrade

```

If backend changed:

```bash
terraform init -reconfigure

```

Example provider version change:

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.50"
    }
  }
}

```

Then:

```bash
terraform init -upgrade

```

Exam line:

```text
init is not only for new projects. You run it again when providers, modules, or backend settings change.

```

----------

## 8. Apply the Configuration

Main command:

```bash
terraform apply

```

Safer workflow:

```bash
terraform plan
terraform apply

```

Save a plan:

```bash
terraform plan -out=tfplan
terraform apply tfplan

```

Destroy:

```bash
terraform destroy

```

Auto-approve:

```bash
terraform apply -auto-approve
terraform destroy -auto-approve

```

But for exam and real production:

```text
plan first
review
then apply

```

----------

## 9. What Happens During Terraform Apply?

Terraform compares three things:

```text
Configuration       = what you want
State file          = what Terraform thinks it manages
Real infrastructure = what actually exists

```

Then it decides what action is needed.

----------

### Case 1 — Resource exists in configuration but not in state

Terraform creates it.

```hcl
resource "aws_s3_bucket" "logs" {
  bucket = "my-log-bucket"
}

```

Command:

```bash
terraform apply

```

Terraform action:

```text
Create bucket
Record bucket in state

```

----------

### Case 2 — Resource exists in state but removed from configuration

Terraform destroys it.

Example:

Before:

```hcl
resource "aws_s3_bucket" "logs" {
  bucket = "my-log-bucket"
}

```

Then you delete/comment out that block.

Run:

```bash
terraform plan

```

Terraform shows:

```text
- destroy aws_s3_bucket.logs

```

Then:

```bash
terraform apply

```

Terraform deletes the real bucket and removes it from state.

Important:

```text
Configuration is the Big Boss.
If configuration says the resource should not exist, Terraform plans to destroy it.

```

----------

### Case 3 — Resource exists but arguments changed

Example:

```hcl
resource "aws_instance" "web" {
  ami           = "ami-123456"
  instance_type = "t3.micro"
}

```

Changed to:

```hcl
resource "aws_instance" "web" {
  ami           = "ami-123456"
  instance_type = "t3.small"
}

```

Run:

```bash
terraform plan

```

Terraform may show:

```text
~ update in-place

```

Then:

```bash
terraform apply

```

Terraform updates the resource and updates state.

----------

### Case 4 — Argument changed but cannot be updated in place

Some changes require replacement.

Example:

```text
Old object must be destroyed
New object must be created

```

Terraform plan shows:

```text
-/+ destroy and then create replacement

```

or:

```text
+/- create replacement and then destroy

```

With lifecycle:

```hcl
lifecycle {
  create_before_destroy = true
}

```

Terraform tries to create the new resource before destroying the old one.

----------

### Case 5 — State is updated

After apply, Terraform updates the state file.

State file tracks:

```text
resource addresses
resource IDs
attributes
dependencies
metadata

```

Useful commands:

```bash
terraform state list
terraform state show <resource_address>
terraform show

```

Example:

```bash
terraform state list
terraform state show aws_instance.web

```

----------

## 10. Infrastructure Changes Over Time

This is the important section you called out.

As infrastructure changes, you may need to:

```text
Add resources
Remove resources
Move resources into modules
Rename resources
Remove resources from state only
Destroy resources from real infrastructure
Retire resources from cloud/provider

```

These are not the same thing.

----------

## 10.1 Add a Resource

Add a new resource block:

```hcl
resource "aws_s3_bucket" "logs" {
  bucket = "my-log-bucket"
}

```

Run:

```bash
terraform plan
terraform apply

```

Terraform creates the new infrastructure object.

----------

## 10.2 Remove a Resource from Configuration

If you remove the resource block from `.tf` files:

```hcl
# resource "aws_s3_bucket" "logs" {
#   bucket = "my-log-bucket"
# }

```

Run:

```bash
terraform plan

```

Terraform plans:

```text
Destroy the real infrastructure
Remove it from state

```

This is the normal behavior.

----------

## 10.3 Create Modules

A module is a reusable collection of Terraform configuration.

Instead of repeating this everywhere:

```hcl
resource "aws_s3_bucket" "logs" {
  bucket = "app1-logs"
}

```

You can create a module:

```text
modules/s3_bucket/main.tf

```

Inside module:

```hcl
variable "bucket_name" {}

resource "aws_s3_bucket" "this" {
  bucket = var.bucket_name
}

```

Use module:

```hcl
module "logs_bucket" {
  source      = "./modules/s3_bucket"
  bucket_name = "app1-logs"
}

```

Commands:

```bash
terraform init
terraform plan
terraform apply

```

Why `init`?

Because modules must be installed/initialized.

----------

## 10.4 Refactor Resources into Modules

This is where Terraform can get dangerous if you do it wrong.

Suppose you originally had:

```hcl
resource "aws_s3_bucket" "logs" {
  bucket = "app1-logs"
}

```

State address:

```bash
aws_s3_bucket.logs

```

Then you move it into a module:

```hcl
module "logs_bucket" {
  source      = "./modules/s3_bucket"
  bucket_name = "app1-logs"
}

```

New address may become:

```bash
module.logs_bucket.aws_s3_bucket.this

```

Terraform may think:

```text
Old resource removed
New resource added

```

So plan may show:

```text
Destroy aws_s3_bucket.logs
Create module.logs_bucket.aws_s3_bucket.this

```

But you do not want destruction. You only changed Terraform structure.

Use a `moved` block:

```hcl
moved {
  from = aws_s3_bucket.logs
  to   = module.logs_bucket.aws_s3_bucket.this
}

```

Then run:

```bash
terraform plan
terraform apply

```

Terraform updates the state address without destroying the real resource.

Older/manual way:

```bash
terraform state mv aws_s3_bucket.logs module.logs_bucket.aws_s3_bucket.this

```

Exam line:

```text
Refactoring means changing Terraform structure, not necessarily changing real infrastructure.

```

----------

## 10.5 Rename a Resource

Before:

```hcl
resource "aws_s3_bucket" "logs" {
  bucket = "app1-logs"
}

```

Address:

```bash
aws_s3_bucket.logs

```

After rename:

```hcl
resource "aws_s3_bucket" "application_logs" {
  bucket = "app1-logs"
}

```

New address:

```bash
aws_s3_bucket.application_logs

```

Terraform may think:

```text
Destroy old
Create new

```

Use moved block:

```hcl
moved {
  from = aws_s3_bucket.logs
  to   = aws_s3_bucket.application_logs
}

```

Then:

```bash
terraform plan
terraform apply

```

Or manual state move:

```bash
terraform state mv aws_s3_bucket.logs aws_s3_bucket.application_logs

```

----------

## 10.6 Remove a Resource from State Without Destroying Infrastructure

This is very important.

Command:

```bash
terraform state rm <resource_address>

```

Example:

```bash
terraform state rm aws_s3_bucket.logs

```

What happens:

```text
Terraform forgets the resource
Real infrastructure remains
Resource is removed from state

```

Terraform does not call the cloud/provider delete Application Programming Interface.

Use this when:

```text
You no longer want Terraform to manage the object
You imported something by mistake
You are handing management to another Terraform state
You are splitting state files
You want to stop tracking but keep the infrastructure

```

Important danger:

If the resource block still exists in configuration after `state rm`, then next plan may show:

```text
Create new resource

```

because Terraform sees:

```text
Configuration wants it
State does not have it
So Terraform wants to create it

```

Correct pattern if you want Terraform to forget but keep infra:

```bash
terraform state rm aws_s3_bucket.logs

```

Then also remove/comment the resource from configuration.

----------

## 10.7 Destroy a Resource

Destroy means:

```text
Delete real infrastructure
Remove it from state

```

Destroy everything:

```bash
terraform destroy

```

Destroy one resource:

```bash
terraform destroy -target=aws_s3_bucket.logs

```

Safer:

```bash
terraform plan -destroy
terraform destroy

```

Important:

```text
terraform destroy deletes actual infrastructure.
terraform state rm does not delete actual infrastructure.

```

Clean distinction:

Action

Real Infrastructure

Terraform State

Remove from config + apply

Destroyed

Removed

`terraform state rm`

Kept

Removed

`terraform destroy`

Destroyed

Removed

`terraform state mv`

Kept

Address changed

`moved` block

Kept

Address changed

----------

## 10.8 Retire a Resource from Cloud Provider or State File

There are two meanings of “retire.”

### Retire from real infrastructure

You want the object gone.

Use:

```bash
terraform destroy -target=<resource_address>

```

or remove it from configuration and run:

```bash
terraform apply

```

Result:

```text
Real object deleted
State updated

```

----------

### Retire from Terraform management only

You want the object to remain, but Terraform should stop managing it.

Use:

```bash
terraform state rm <resource_address>

```

Then remove it from configuration.

Result:

```text
Real object remains
Terraform forgets it

```

----------

## 11. Data Sources

Many providers allow Terraform to read existing infrastructure without creating it.

That is a **data source**.

Resource:

```hcl
resource "aws_s3_bucket" "logs" {
  bucket = "my-log-bucket"
}

```

This creates/manages a bucket.

Data source:

```hcl
data "aws_ami" "latest" {
  most_recent = true

  owners = ["amazon"]
}

```

This reads an existing Amazon Machine Image.

Use data source values:

```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.latest.id
  instance_type = "t3.micro"
}

```

Difference:

Terraform Block

Purpose

`resource`

Create/manage infrastructure

`data`

Read existing infrastructure

Exam line:

```text
Resources change infrastructure.
Data sources only read information from existing infrastructure.

```

----------

# Core Commands Summary

```bash
terraform init
terraform init -upgrade
terraform init -reconfigure
terraform validate
terraform fmt
terraform plan
terraform plan -out=tfplan
terraform apply
terraform apply tfplan
terraform destroy
terraform state list
terraform state show <resource>
terraform state rm <resource>
terraform state mv <old_address> <new_address>
terraform import <resource_address> <real_resource_id>
terraform show
terraform providers

```

----------

# Final Mental Model

```text
Configuration = desired infrastructure
State         = Terraform’s memory
Provider      = plugin that talks to platform API
Resource      = object Terraform manages
Data source   = object Terraform reads
Apply         = makes real infrastructure match configuration

```

Most important exam distinction:

```text
Remove from configuration + apply = destroy real object.
terraform state rm = forget object, keep real object.
terraform destroy = delete real object and remove from state.
moved block/state mv = rename or move tracking without destroying.

```
<!--stackedit_data:
eyJoaXN0b3J5IjpbMjI5MzEzMzc5LC0yNDc5OTAxODVdfQ==
-->