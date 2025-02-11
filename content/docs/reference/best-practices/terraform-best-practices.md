---
title: "Terraform Best Practices"
description: "Our opinionated best-practices for Terraform"
slug: "terraform-best-practices"
tags:
  - "Best Practices"
  - "Terraform"
categories:
  - "Community Resources"
  - "Terraform"
---

![Terraform](/assets/08bcd99-terraform.png)

These are the *opinionated* best-practices we follow at Cloud Posse. They are inspired by years of experience writing terraform
and borrow on the many other helpful resources like those by [HashiCorp](https://www.terraform.io/docs/cloud/guides/recommended-practices/index.html).

See our general [Best Practices](/category/best-practices/) which also apply to Terraform.

# Language

## Use indented `HEREDOC` syntax

Using `<<-EOT` (as opposed to `<<EOT` without the `-`) ensures the code can be indented inline with the other code in the project.
Note that `EOT` can be any uppercase string (e.g. `CONFIG_FILE`)


```
block {
  value = <<-EOT
  hello
    world
  EOT
}
```

## Do not use `HEREDOC` for JSON, YAML or IAM Policies

There are better ways to achieve the same outcome using terraform interpolations or resources

For JSON, use a combination of a `local` and the [`jsonencode`](https://www.terraform.io/docs/configuration/functions/jsonencode.html) function.

For YAML, use a combination of a `local` and the [`yamlencode`](https://www.terraform.io/docs/configuration/functions/yamlencode.html) function.

For IAM Policy Documents, use the native [`iam_policy_document`](https://www.terraform.io/docs/providers/aws/d/iam_policy_document.html) resource.

## Do not use long `HEREDOC` configurations

Use instead the [`template_file`](https://www.terraform.io/docs/providers/template/d/file.html) resource and move the configuration to a separate template file.


## Use terraform linting

Linting helps to ensure a consistent code formatting, improves code quality and catches common errors with syntax.

Run `terraform fmt` before committing all code. Use a `pre-commit` hook to do this automatically. See [Terraform Tips & Tricks](/reference/best-practices/terraform-tips-tricks.md)

## Use proper datatype

Using proper datatypes in terraform makes it easier to validate inputs and document usage.

* Use `null` instead of empty strings (`""`)
* Use `bool` instead of strings or integers for binary true/false
* Use `string` for freeform text
* Use `object` sparingly as it makes it harder to document and validate
*

Note, in *HCLv1*, it was recommended to use strings for all booleans. This is no longer a best practice with HCLv2.
Read more:  https://www.terraform.io/docs/configuration-0-11/variables.html#booleans

## Use CIDR math interpolation functions for network calculations

This reduces the barrier to entry for others to contribute and reduces likelihood of human error. We have a number
of terraform modules as well to aide in the subnet calculations since there are multiple strategies depending on the use-case.

- https://github.com/cloudposse/terraform-aws-dynamic-subnets
- https://github.com/cloudposse/terraform-aws-multi-az-subnets
- https://github.com/cloudposse/terraform-aws-named-subnets

Read more: https://www.terraform.io/docs/configuration/interpolation.html#cidrsubnet-iprange-newbits-netnum-

## Use `.editorconfig` in all repos for consistent whitespace

Every mainstream IDE supports plugins for the `.editorconfig` standard to make it easier to enforce whitespace consistency.

We recommend adopting the whitespace convention of a particular language or project.

This is the standard `.editorconfig` that we use.

```ini
# Override for Makefile
[{Makefile, makefile, GNUmakefile, Makefile.*}]
indent_style = tab
indent_size = 4

[*.yaml]
indent_style = space
indent_size = 2

[*.sh]
indent_style = tab
indent_size = 4

[*.{tf,tfvars,tpl}]
indent_size = 2
indent_style = space
```

## Use miminum version pinning on all providers

Terraform's providers are constantly in flux. It is hard to know if the module you write today will work with older versions of the provider APIs, and usually not worth the effort to find out.  For that reason we want to advertise the minimum version of the provider we have tested with.

While it is possible that future versions may introduce breaking changes, our experience has been that this is no longer likely. Furthermore, in a library of code such as what Cloud Posse publishes, we cannot test updated providers to find out about problems if we have placed upper limits on versions. Therefore for all our modules, we only place lower limits on versions.

In your root modules, you may want to include upper limits or pin to exact versions to avoid suprises. It is a trade-off between stability and ease of staying current you will have to evaluate for your own situation.

## Use `locals` to baptize opaque resource IDs

Using `locals` makes code more descriptive and maintainable. Rather than using complex expressions as parameters to some terraform resource,
instead move that expression to a `local` and reference the `local` in the resource.

# Variables

## Use upstream module or provider variable names where applicable

When writing a module that accepts `variable` inputs, make sure to use the same names as the upstream to avoid confusion and ambiguity.

## Use all lower-case with underscores as separators

Avoid introducing any other syntaxes commonly found in other languages such as CamelCase or pascalCase. For consistency we want all variables
to look uniform. This is also inline with the [HashiCorp naming conventions](https://www.terraform.io/docs/extend/best-practices/naming.html).

## Use positive variable names to avoid double negatives

All `variable` inputs that enable/disable a setting should be formatted `...._enabled` (e.g. `encryption_enabled`). It is acceptable for
default values to be either `false` or `true`.

## Use feature flags to enable/disable functionality

All modules should incorporate feature flags to enable or disable functionality. All feature flags should end in `_enabled` and should be of type `bool`.

## Use description field for all inputs

All `variable` inputs need a `description` field. When the field is provided by an upstream provider (e.g. `terraform-aws-provider`), use same wording as the upstream docs.

## Use sane defaults where applicable

Modules should be as turnkey as possible. The `default` value should ensure the most secure configuration (E.g. with encryption enabled).

## Use variables for all secrets with no `default` value

All `variable` inputs for secrets must never define a `default` value. This ensures that `terraform` is able to validate user input.
The exception to this is if the secret is optional and will be generated for the user automatically when left `null` or `""` (empty).

# Outputs

## Use description field for all outputs

All outputs must have a `description` set. The `description` should be based on (or adapted from) the upstream terraform provider where applicable.
Avoid simply repeating the variable name as the output `description`.

## Use well-formatted snake case output names

Avoid introducing any other syntaxes commonly found in other languages such as CamelCase or pascalCase. For consistency we want all variables
to look uniform. It also makes code more consistent when using outputs together with terraform [`remote_state`](https://www.terraform.io/docs/providers/terraform/d/remote_state.html) to access those settings from across modules.

## Never output secrets

Secrets should never be outputs of modules. Rather, they should be written to secure storage such as AWS Secrets Manager, AWS SSM Parameter Store with KMS encryption, or S3 with KMS encryption at rest. Our preferred mechanism on AWS is using SSM Parameter Store. Values written to SSM are easily
retrieved by other terraform modules, or even on the command-line using tools like [chamber](https://github.com/segmentio/chamber) by Segment.io.

We are very strict about this in "root" modules (or the top-most module), because these sensitive outputs are easily leaked in CI/CD pipelines (see [`tfmask`](https://github.com/cloudposse/tfmask) for masking secrets in output only as a last resort). We are less sensitive to this in modules that are typically nested inside of other modules.

Rather than outputting a secret, you may output plain text indicating where the secret is stored, for example `RDS master password is in SSM parameter /rds/master_password`. You may also want to have another output just for the key for the secret in the secret store, so the key is available to other programs which may be able to retrvieve the value given the key.

## Use symmetrical names

We prefer to keep terraform outputs symmetrical as much as possible with the upstream resource or module, with exception of prefixes. This reduces the amount of entropy in the code or possible ambiguity, while increasing consistency. Below is an example of what **not* to do. The expected output name is `user_secret_access_key`. This is because the other IAM user outputs in the upstream module are prefixed with `user_`, and then we should borrow the upstream's output name of `secret_access_key` to become `user_secret_access_key` for consistency.

![Terraform outputs should be symmetrical](/assets/terraform-outputs-should-be-symmetrical.png)

# State

## Use remote state

## Use Terraform to create state bucket

This requires a two-phased approach, whereby you first provision the bucket without the remote state enabled. Then enable remote state (e.g. `s3 {}`) and import remote state by simply rerunning `terraform init`. We recommend this strategy because it promotes using the best tool for the job and makes it easier to define requirements and use consistent tooling.

Using the [`terraform-aws-tfstate-backend`](https://github.com/cloudposse/terraform-aws-tfstate-backend) module it is easy to provision state buckets.

## Use backend with support for state locking

We recommend using the S3 backend with DynamoDB for state locking.

**Pro Tip:** Using the [`terraform-aws-tfstate-backend`](https://github.com/cloudposse/terraform-aws-tfstate-backend) this can be easily implemented.

https://www.terraform.io/docs/backends/types/s3.html

## Use strict enforcement of version of `terraform` CLI

Terraform state is incompatible between versions of CLI. We suggest using a container to promote the use of identical tooling for all devs.

**Pro Tip:**  Use [`geodesic`](https://github.com/cloudposse/geodesic) to manage all `terraform` interactions

## Use `terraform` CLI to set backend parameters

Promote the reusability of a root module across accounts by avoiding hardcoded backend requirements. Instead, use the Terraform CLI to set the current context.

```hcl
terraform {
  required_version = ">= 0.12.26"

  backend "s3" {}
}
```

Future versions of Terraform are usually at least mostly compatible of previous versions, and we want to be able to test the modules with future versions to find out. Therefore, do not place an upper limit on the `required_version` like `~>0.12.26` or `>= 0.13, < 0.15`. Always use `>=` and enforce Terraform versions in your environment by controlling which  CLI you use.

## Use encrypted S3 bucket with versioning, encryption and strict IAM policies

We recommend not commingling state in the same bucket. This could cause the state to get overridden or compromised. Note, the state contains cached values of all outputs. Wherever possible, keep stages 100% isolated with physical barriers (separate buckets, separate organizations)

**Pro Tip:** Using the [`terraform-aws-tfstate-backend`](https://github.com/cloudposse/terraform-aws-tfstate-backend) to easily provision buckets for each stage.

## Use Versioning on State Bucket

## Use Encryption at Rest on State Bucket

## Use `.gitignore` to exclude terraform state files, state directory backups and core dumps

```
.terraform
.terraform.tfstate.lock.info
*.tfstate
*.tfstate.backup
```

## Use `.dockerigore` to exclude terraform statefiles from builds

Example:

```
**/.terraform*
```

# Root Module

# Data Sources

# Resources

# AWS Specific

# Naming Conventions

## Use a programmatically consistent naming convention

All resource names (E.g. things provisioned on AWS) must follow a consistent convention. The reason this is so important is
that modules are frequently composed inside of other modules. Enforcing consistency increases the likelihood that modules can
invoke other modules without colliding on resource names.

To enforce consistency, we require that all modules use the [`terraform-null-label`](https://github.com/cloudposse/terraform-null-label) module.
With this module, users have the ability to change the way resource names are generated such as by changing the order of parameters or the delimiter.
While the module is opinionated on the parameters, it's proved invaluable as a mechanism for generating consistent resource names.

# DNS Infrastructure

## Use lots of DNS zones

Never mingle DNS from different stages or environments in the same zone.

## Delegate DNS zones across account boundaries

Delegate each AWS account a DNS zone for which it is authoritative.

## Distinguish between branded domains and service discovery domains

Service discovery domains are what services use to discover each other. These are seldom if ever used by end-users. There should only
be one service discovery domain, but there may be many zones delegated from that domain.

Branded domains are the domains that users use to access the services. These are determined by products, marketing, and business use-cases.
There may be many branded domains pointing to a single service discovery domain. The architecture of the branded domains won't mirror the
service discovery domains.

# Module Design

## Small Opinionated Modules

We believe that modules should do one thing very well. But in order to do that, it requires being opinionated on the design.
Simply wrapping terraform resources for the purposes of modularizing code is not that helpful. Implementing a specific use-case
of those resource is more helpful.

## Composable Modules

Write all modules to be easily composable into other modules. This is how we're able to achieve economies of scale and stop
re-inventing the same patterns over and over again.

## Use `variable` inputs

Modules should accept as many parameters as possible. Avoid using inputs of `type = object` since they are harder to document. Of course,
this is not a hard rule and sometimes objects just make the most sense. Just be weary of the ability for tools like [`terraform-docs`](https://github.com/segmentio/terraform-docs) to be able to
generate meaningful documentation.

# Module Usage

## Use Terraform registry format with exact version numbers

There are many ways to express a module's source. Our convention is to use Terraform registry syntax with an explicit version.

```
  source  = "cloudposse/label/null"
  version = "0.22.0"
```

The reason to pin to an explicit version rather than a range like `>= 0.22.0` is that any update is capable of breaking something. Any changes to your infrastructure should be implemented and reviewed under your control, not blindly automatic based on when you deployed it.

:::info

Prior to Terraform v0.13, our convention was to use the pure git url:

```hcl
source = "git::https://github.com/cloudposse/terraform-null-label.git?ref=tags/0.16.0"
```

Note that the `ref` always points explicitly to a `tags` pinned to a specific version. Dropping the `tags/` qualifier means it could be a branch or a tag; we prefer to be explicit.

:::
