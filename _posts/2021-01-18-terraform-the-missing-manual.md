---
layout: post
title: "Terraform - The Missing Manual"
description: "Infrastructure-as-Code is a principal that drives modern DevOps practice. I discuss the current state of Terraform and provide some basic guidelines/principles regarding how to structure it's usage for your project."
tags: [terraform, aws, hashicorp]
---
![Terrafrom Logo](https://www.terraform.io/assets/images/og-image-large-e60c82fe.png)

# Terraform - The Missing Manual

Infrastructure-as-Code is a principal that drives modern DevOps practice. I discuss the current state of Terraform and provide some basic guidelines/principles regarding how to structure it's usage for your project.

# Tables of Contents
- [Rationale](https://gist.github.com/xirkus/e57cc20fe7fc95694e302887948e9b12#rationale)
- [Background](https://gist.github.com/xirkus/e57cc20fe7fc95694e302887948e9b12#background)
- [Terraform Documentation Gotcha's](https://gist.github.com/xirkus/e57cc20fe7fc95694e302887948e9b12#terraform-documentation-gotchas)
- [Repository Structure](https://gist.github.com/xirkus/e57cc20fe7fc95694e302887948e9b12#repository-structure)
- [Basics](https://gist.github.com/xirkus/e57cc20fe7fc95694e302887948e9b12#basics)
    - [Directives](https://gist.github.com/xirkus/e57cc20fe7fc95694e302887948e9b12#directives)
    - [Environment Variables](https://gist.github.com/xirkus/e57cc20fe7fc95694e302887948e9b12#environment-variables)
    - [Two Stage Deployment](https://gist.github.com/xirkus/e57cc20fe7fc95694e302887948e9b12#two-stage-deployment)
    - [The Module](https://gist.github.com/xirkus/e57cc20fe7fc95694e302887948e9b12#the-module)
        - [Runtime variables with `.tfvars`](https://gist.github.com/xirkus/e57cc20fe7fc95694e302887948e9b12#runtime-variables-in-tfvars)
        - [Use Officially Support Terraform Registry Modules](https://gist.github.com/xirkus/e57cc20fe7fc95694e302887948e9b12#use-officially-supported-terraform-registry-modules)
    - [Terraform Workspaces](https://gist.github.com/xirkus/e57cc20fe7fc95694e302887948e9b12#terraform-workspaces)
        - [Use Terraform Cloud to structure Workspaces and save deployment state](https://gist.github.com/xirkus/e57cc20fe7fc95694e302887948e9b12#use-terraform-cloud-to-structure-workspaces-and-save-deployment-state)
- [DevOps Workflow](https://gist.github.com/xirkus/e57cc20fe7fc95694e302887948e9b12#devops-workflow)
- [Miscellaneous](https://gist.github.com/xirkus/e57cc20fe7fc95694e302887948e9b12#miscellaneous)
- [Conclusion](https://gist.github.com/xirkus/e57cc20fe7fc95694e302887948e9b12#conclusion)
- [TODO](https://gist.github.com/xirkus/e57cc20fe7fc95694e302887948e9b12#todo)

## Rationale

Having used Terraform in the past to deploy a single-tenanted Kubernetes cluster to GCP, I was curious to see how much Terraform had evolved and how well it supported AWS. My deep dive revealed that _to use the cutting edge required significant bleeding_. To save some grief/frustration for others, I've written some guidelines which I believe will answer the following questions:

1. **Where do I start with a new project?**
1. **How do I structure a Terraform repository?**
1. **How do I accomodate different deployment environments?**

To make sense of these guidelines, you should be aware of the following:

- Infrastructure as Code implies that programmatic structure should be applied to the organization of the code, not simply that configuration files should be version controlled.
- Programmatic structure implies the DRY (Don't Repeat Yourself) principle.
- Programmatic structure implies the Single Responsibility principle.
- Programmatic structure implies that code will be literate and self-documenting.
- Programming idioms are used to structure the understanding of how to best leverage Terraform and it's capabilities.
- We adhere to the KISS Principle and avoid adding unnecessary complexity.

Caveats:

1. This is a 80/20 solution, meaning that not all use cases will be covered.
1. This solution assumes a single cloud provider.
1. Assumes that all configuration files will be version controlled and that version control features will be used (such as feature branches for changes).

## Background

- [Evolving Your Infrastructure with Terraform: OpenCredo's 5 Common Terraform Patterns](https://www.hashicorp.com/resources/evolving-infrastructure-terraform-opencredo)
    - The problem with the proposed final solution is that multiple state files for various components of your infrastructure undermine the underlying premise of Terraform - that there is a single source of truth regarding the desired state of your infrastructure as code.
    - In the "Orchestrating Terraform" section, she mention's that there are module ordering dependencies that multi-state Terraform configurations suffer from. Why not use `depends_on` within a single mono-repo?
-  [Keep your Terraform code DRY](https://terragrunt.gruntwork.io/docs/features/keep-your-terraform-code-dry/#motivation)
- Advocates for the use of Terragrunt, but this seems like another layer of abstraction/complexity which violates _KISS_.
- Also requires duplicate config per deployment environment which seems to defeat the purpose of _DRY_.
- [The Role-Profiles Pattern Across Infrastructure as Code](https://medium.com/cloud-technology-solutions/the-role-profiles-pattern-across-infrastructure-as-code-3d8910dcd3c9)
    - interesting perspective with regards to desired structure of infrastructure code, but again relies on Terragrunt.
- [Advantages and Pitfalls of your Infra-as-Code Repo Strategy](https://medium.com/swlh/advantages-and-pitfalls-of-your-infra-as-code-repo-strategy-214c1cb45612) - Another article with some interesting perspectives, but again recommends structuring a multi-tenant repo with directories instead of Workspaces. Workspaces maintain the state of a _specific_ deployment while providing access control on who can make the changes. Relegating this to a git repo's access control seems like a step backwards to me.

> ***These perspectives of advanced Terraform users suggest that _STRUCTURE_ is a fundamental complexity/issue introduced by Terraform. Can these be addressed using _JUST_ Terraform features and providing a "Convention over Configuration" solution?***

## Terraform Documentation Gotcha's

**[Official Terraform Documentation](https://www.terraform.io/docs/index.html)**

There's a wealth of documentation on Terraform provided by Hashicorp, and most users would assume (like myself) that this is good place to start. But here's the problem - the shear _amount of documentation_ does very little to help determine _WHERE_ to get started.

> **Terraform's documentation is like learning to communicate in a foreign language using only a dicitionary as a guide.**

While the specifics are well documented, it's difficult to understand the _CONTEXT_ of how to apply that information without significant experimentation. In other words, the tool ergonomics _SUCK_. Here's a list of missing features that would go a long way to improving the Terraform experience

- scaffolding which suggests an organizational pattern for structuring the project
- clear and actionable error messages
- up-to-date documentation which uses working examples and references/leverages current features (for example, Modules and the Terraform Registry).
- intellisense/autocomplete on defined output variables in the IntelliJ plugin.

This list is far from exhaustive, but it gives you a sense of the number of sharp edges associated with Terraform adoption/usage and the barrier to entry that many face.

## Repository Structure

***TL;DR*** - here's the recommended Terraform organizational structure. What follows is a discussion of basic Terraform concepts and the justifictions as to _WHY_ this structure makes the most sense.

Throughout, I make a number of **[Recommendation]s** as well as point out some **[ANTI-PATTERN]s**, **[WARNING]s**, and **[BUG]s**.

```console
.
README.md
main.tf
outputs.tf
variables.tf
+-- env
    +-- dev.tfvars
    +-- staging.tfvars
    +-- prod.tfvars
+-- aws
    +-- iam
        +-- README.md
        +-- main.tf
        +-- output.tf
        +-- variables.tf
    +-- lambda
        +-- README.md
        +-- main.tf
        +-- output.tf
        +-- variables.tf
    +-- ...
```

## Basics

### Directives

[Official Configuration Documentation](https://www.terraform.io/docs/configuration/index.html)

Terraform configuration consists of files (which end with the `.tf` extension) which contain directives. These directives are used to accomplish the following things:

- [`provider`](https://www.terraform.io/docs/configuration/blocks/providers/index.html): specify the infrastructure provider you want to deploy against
- [`resource`](https://www.terraform.io/docs/configuration/blocks/resources/index.html): specify the desired end state of the resources you want to configure
- [`variable`](https://www.terraform.io/docs/configuration/variables.html): pass around configuration variables between various Terraform components
- [`output`](https://www.terraform.io/docs/configuration/outputs.html): provide output regarding configured resources
- [`module`](https://www.terraform.io/docs/configuration/blocks/modules/index.html): a collection of terraform configuration files
- [`locals`](https://www.terraform.io/docs/configuration/locals.html) : variables defined specifically for a `module` scope

The above directives are _not_ exhaustive, but the ones mentioned will be the most commonly used to set up some basic configuration.

### Environment Variables

**[Assigning values to root module variables](https://www.terraform.io/docs/configuration/variables.html#assigning-values-to-root-module-variables)**

Terraform provides a number of mechanisms to provide input into the executing `terraform` operation. These inputs can be provided through the following methods (listed in order of precedence):

- `export TF_VAR_name=value && terraform _operation_`: provide an shell environment variable to specify the variable
- `terraform -var='name=value' _operation_`: specify a specific variable as an argument to the command
- `terraform -var-file="./path/to/file.tfvars" _operation_`: store specific variable name/value assignments in a `.tfvars` file

### Two-stage Deployment

Terraform is particularly useful for having a two-stage deployment (`terraform plan && terraform apply`). This ensures that only _valid_ configurations can be deployed. Terraform also provides a `terraform validate` command to ensure that the syntax within your module is correct (but I find this less useful that running `terraform plan` directly).

### The Module

> ***The `module` is the fundamental building block of Terraform (not, the `.tf` files themselves). Understanding this is the key to being able to structure your configuration repo.***

A few key points regarding the Module:

1. ***_EVERY_ directory that contains `.tf` files is considered a Module.*** This includes the root directory where Terraform configuration is first specified.
1. ***Modules can be nested.*** (meaning subdirectories off of the root directory are considered child modules.
1. ***Modules are structured in a parent-child hierarchy.***
1. ***Modules do _NOT_ provide inheritance visibility to specifications between parent/child relationships which a few notable exceptions (i.e. `provider`).*** Information that needs to be passed between parent/child/sibling Modules needs to be _specified explicitly_.
1. ***Modules can consist of more than a single `.tf` file.***
1. ***Modules can be user-defined or reference external/public modules such as those available in the Terraform Registry.***

If you think of the Module as analogous to a programming language method or function, then an obvious usage structure emerges regarding the organization of Terraform assets.

|**Programming Concept**|**Terraform Equivalent**|
|-|-|
|function/method        |Module                    |
|parameters             |Input Variables           |
|return value           |Output Variables          |
|local variables        |`locals` specification    |
|method/function calls. |`module` specification    |
|implementation code.   |`resource` specification  |

**The root directory is the function/entry point for `terraform` operations.**

Using this analogy, a logical Module structure becomes apparent.

> **[Recommendation] Place all Module inputs in `variables.tf`**
> This file explicitly declares all required variables for the Module.

> **[Recommendation] Specify all Module outputs in `outputs.tf`**
> This file explicitly declares all values returned by the module associated with newly provisioned assets.

> **[Recommendation] Specify all implementation details in `main.tf`**
> This file contains a Module's implementation specifics which may include the following:
>
> - `provider` specification (usually defined at the _root_ of a Terraform project).
> - `module` specifications (which specify where feature specific implementation details are found/configured).
> - `resource` specifications (which are feature specific implementation configuration).
> - `locals` specifications (local variables used to remove boiler-plate specification variables).

> **[ANTI-PATTERN]: The monolithic `.tf` file**
> A lot of starting tutorials begin with a single monolithic `.tf` file. While initially expedient, this lack of organization
> means Terraform assets with different responsibilities/dependencies remain undifferentiated for the end user. Violating the
> single-responsibility principle leads to unmaintainable code and future technical debt.

> **[ANTI-PATTERN]: The use of unconventional `.tf` file names**
> If you look at the Terraform documentation and official Modules published on the Terraform Registry, an organizational
> convention is used. Each module contains the following three `.tf` files:
>
> - `main.tf`
> - `variables.tf`
> - `outputs.tf`
>
> While it's possible to make a service specific configuration file (i.e. `api_service.tf`), this is _NOT RECOMMENDED_.
>
> Why? By explicitly documenting a Module through the use of these files, you communicate with the end user the intended
> usage of the Module. Inputs are clearly defined within `variables.tf` as well as return values in the `outputs.tf`. All
> implementation specific details are separated into the `main.tf`. This is more readily consumable by end users.

#### Runtime variables in `.tfvars`

A commonly encountered software development pattern requires different deployment environments for developers, staging, and production. The differences between environments can be reflected in things such as:

- the size of the compute asset
- the location of the asset
- access privileges to the asset or any generated artifacts
- the provider profile being used to deploy the infrastructure
- etc.

We make the assumption here that each environment has it's own provider profile configured for the deployment specific functionality in order to follow the principle of _Least Privileged Access_ (i.e. we don't want a single provider profile to manage each of these separate deployment environments to avoid change conflicts). Having separate provisioner profiles also means that developers/QA _CANNOT_ accidentally overwrite the production deployments with newer/unverified changes which may break the application. As per DevOps best practices, production deployments should be automated through the use of profiles which have limited membership/access.

These types of configuration vary in the asset specifics, but not necessarily in the _STRUCTURE_ of the application being deployed. The variants between deployment environments are best encoded in separated `.tfvars` files. In particular, we recommend the following organization:

- `dev.tfvars`
- `staging.tfvars`
- `prod.tfvars`

> **[Recommendation] Use deployment specific resource prefixes to differentiate provisioned assets**
> Provisioned assets should be specific to a deployment type (i.e. the same asset should _NOT_ be used against different  
> deployment stages). To facilitate this separation, deployment specific prefixes should be used for asset creation. This
> ensures a clear separation of access/responsibilities between created assets.
>
> There are a number of features Terraform provides to support this separation:
>
> 1. ***The Workspace feature (which we will discuss in detail later)***. Essentially, different workspaces can be used to
     > preserved the configured state of a deployment. With separate Workspaces, you can avoid conflict between different
     > deployment environments (i.e. changes applied to dev will not automatically be applied to production).
>
> 1. ***Terraform provides the `local.workspace` variable*** to reference the current Workspace within configuration files. If
     > you use the Workspace feature, there is no need to create a user defined `workspace` variable.

#### Use officially supported Terraform Registry Modules

**[Terraform Registry](https://registry.terraform.io)**

It's tempting to begin using Terraform by slapping together some quick `resource` directives, but this is the path to despair. You'll quickly realize that maintaining configuration this way easily leads to duplication and spaghetti.

What's the alternative? Although the official documentation only mentions Terraform Registry in passing, you should strive to use existing modules to configure your infrastructure. In addition to being officially supported and tested, leveraging existing Modules leaves you free to focus on your infrastructure and not the implementation details of standard provider features/capabilities.

While ideally, this is the way _ALL_ users should start, there is no clear guidance on how Terraform Registry Modules should be incorporated into your project. _Even worse_, the Terraform Registry Module documentation is not entirely up-to-date which makes using these modules a frustrating experience at best.

##### Terraform Registry How-to

The following guidance is a combination of project organization and usage which will allow you to leverage the Terraform Registry as a starting/first-class resource. We use the following conventions for our project:

1. **We use a provider subdirectory to contain provider-specific implementations organized as a Module.** This makes a clear distinction between different provider implementations.
1. **For each provider-specific implementation, we follow the above Module structure with regards to organization.**
1. **We separate out provider-specific features by directory.** This means, IAM configuration is separate from asset configuration (for example Lambda configuration).
1. **The goal of this structure is to provide a single-source-of-truth per provider feature, adhere to the Single Responsibility principle, and avoid duplication.** The consequence of this is that features which require multiple configuration (i.e. account/permission configuration as well as asset configuration) will have it's configuration spread across multiple modules. This is intentional; where configuration is dependent on other provider functionality, dependencies can be easily specified between the modules (for example ordering of configuration). This is _much_ more difficult to accomplish if the asset creation mixes a bunch of responsibilities together.
1. **Each user-defined Module in this structure is meant to be reusable (as per the DRY principle).** In other words, required parameters should be passed in as Input variables instead of hard-coded within a Module. These variables _MUST_ be passed into individual `module` blocks and defined within the provider-specific Module's `variables.tf` file. Each Module should be able to be invoked multiple times with configuration specific parameters passed in through Input variables.
1. **Leverage built-in Terraform variables and reduce duplicate interpolation through the use of `locals`.**
1. **Document _EACH_ module with a `README.md` to communicate configuration context to future users that _cannot_ be captured in the configuration files themelves (i.e. reasons why a particular work-around was used).** Optionally, add a link to the official documentation.
1. **Use the Terraform Registry `module` provisioning specification.** Each Module should specify `Provisioning Instructions`. For example, here's the [AWS IAM Module](https://registry.terraform.io/modules/terraform-aws-modules/iam/aws/latest). Use indivudal `resource` directives as a _last resort_.
1. **Run `terraform init` to download the Terraform Registry provided Modules**. These are stored in the `$TERRAFORM_ROOT/.terraform` directory. The underlying structure is a reflection of the organization of your project's `module` structure. The importance of the `.terraform` directory _CANNOT be UNDERESTIMATED_. For each Terraform Registry Module, the complete implementation as well as _EXAMPLES_ are provided. **Use the provided examples to understand how to configure the Module.**

   **Example `.terraform` directory**

   ![.terraform directory](https://user-images.githubusercontent.com/165323/104960156-eb227a00-5990-11eb-9497-4663b7c8b174.png)

> **[Recommendation] The root `main.tf` should have `module` references to user-defined implementation details for specific
> provider features**
>
> The root Module at the top of the Terraform hierarchy, should only contain global configuration and _NOT_ provider specific
> implementation details. These details should be defined in another module which is referenced from the `main.tf` file. For
> example, here's how I have my `main.tf` structured for my project (where AWS is the provider I'm using). Replace the
> `required_providers` with the corresponding provider for your project.
>
> ***${TERRAFORM_ROOT}/main.tf***
> ```
> terraform {
>   // You _CANNOT_ perform variable interpolation within this terraform block!
>   required_providers {
>     ... 
>     aws = {
>       source  = "hashicorp/aws"
>       version = ">= 3.19.0"
>     }
>   }
> ...
> }
> provider "aws" {
>   // These variables are passed in at run time with the use of a .tfvars file
>   region  = var.region 
>   profile = var.aws_profile
>   ...
> }
> locals {
>   // An example of a local variable that is uses a built-in terraform value and reused throughout the main.tf
>   local_variable = "${terraform.variable_name}"
>   ...
> }
> module "aws_iam_user" {
>   source = "./aws/iam"
> 
>   // Input variables provided to the module. These NEED TO BE DEFINED in the module's variables.tf
>   iam_user_var1 = "${local_variable}-iam"
>   iam_user_var2 = "some_value"
>   iam_username = "test"
>   ...
> }
> module "aws_lambda" {
>   source = "./aws/lambda"
>
>   // Input variables provided to the module. These NEED TO BE DEFINED in the module's variables.tf
>   lambda_var1 = "${local_variable}-lambda"
>   lambda_var2 = "another_value"
>   ...
> }
> ...
> ```

> **[Recommendation] Document `module` required/optional/default variables for 3rd party Modules**
> For example, here's my AWS IAM user module specification:
>
> ***${TERRAFORM_ROOT}/aws/iam/main.tf***
> ```console
> module "iam_iam-user" {
> source = "terraform-aws-modules/iam/aws//modules/iam-user"
>  version = "3.6.0"
>
>  # REQUIRED Inputs
>  name = var.iam_username
>
>  # OPTIONAL Inputs
>  force_destroy = true                // default: false
>  password_length = 32                // default: 20
>  password_reset_required = false     // default: true
>  pgp_key = var.keybase_user          // default: ""
>
>  # UNUSED Default Inputs
>  // create_iam_access_key = true
>  // create_iam_user_login_profile = true
>  // create_user = true
>  // path = "/"
>  // permissions_boundary = ""
>  // ssh_key_encoding = "SSH"
>  // ssh_public_key = ""
>  // tags = {}
>  // upload_iam_user_ssh_key = false
> }
> ```

> **[Recommendation] Specify input `variable`s for your user-defined Module in `variables.tf`.** You should have a `variables.tf` file for each Module _even if you don't specify any variables_. It can be parsed by endusers to quickly determine what Input variables are _REQUIRED_ to properly configure the Module.
> For example:
>
> ***$TERRAFORM_ROOT}/aws/iam/variables.tf***
> ```console
> # These variables _SHOULD_ be provided in the parent's module block
>
> variable "iam_username" {}
> variable "variable2" {}
> variable "variable3" {}
> ...
> ```

> **[Recommendation] Define all Module output in the `outputs.tf`**.
> For example:
>
> ***${TERRAFORM_ROOT}/aws/iam/outputs.tf***
> ```console
> output "access_key" {
>   value = module.iam_iam-user.this_iam_access_key_id
> }
>
> output "secret" {
>   value = module.iam_iam-user.this_iam_access_key_encrypted_secret
> }
> 
> output "username" {
>   value = module.iam_iam-user.this_iam_user_name
> }
> 
> output "password" {
>   value = module.iam_iam-user.this_iam_user_login_profile_encrypted_password
> }
> 
> output "lambda_role_arn" {
>   value = aws_iam_role.lambda_role.arn
> }
> ...
> ```
>> ***NOTE:*** Use the `module` references to refer to a Module's defined Output variables.

> **[WARNING]: Variable interpolation _CANNOT_ be used in a number of places.**
> You _CANNOT_ use variable interpolation/expansion in the following circumstances:
>
> 1. In the `terraform` block
> 1. In the `source` parameter of a `module` block
> 1. In the definition of another `variable`. In other words, nested variable interpolation is _NOT_ supported.

> **[WARNING]: The Terraform Registry does _NOT_ correctly document the required variables _NOR_ the correct usage.**
> Use the `examples` provided in the `.terraform` directory when `terraform init` is run.

### Terraform Workspaces

When Terraform executes a `plan`, the state for your infrastructure is stored locally in a `.tfstate` file. If you have multiple deployment environments, this can be problematic as a single state file can be overwritten depending on the environment you are trying to deploy. To avoid this situation, Terraform provides the Workspace feature. Each Workspace maintains a different `.tfstate` file. As a team grows, sharing this state information becomes a high priority.

An additional challenge is that state files contain sensitive information (such as secrets) which may be required for deployments.

Any solution requires both separation of deployment states as well as the ability to share/update state with access control privileges. Tracing of changes to infrastructure can help remediate breaking changes to specific changes introduced by individual devs.

_Enter [Terraform Cloud](https://www.terraform.io/cloud)._

#### Use Terraform Cloud to structure Workspaces and save deployment state

Terraform Cloud is a remote backend state storage service. If either no remote `backend` or Workspace is specified, the following will occur:

- Terraform will use the `default` Workspace.
- Terraform will use a local state backend and store `.tfstate` in the `${TERRAFORM_ROOT}` directory on your machine.

To configure Terraform Cloud as a remote backend with multiple Workspaces, you need to:

1. **Create a Terraform Cloud account.**
1. **Create a Workspace per deployment environment by specifying a Workspace `prefix` as a convention.** For example, you could provision separate workspaces for dev, stage, and prod using the following Workspace naming convention (where `org-` is the organizational prefix):
    - `org-dev`
    - `org-stage`
    - `org-prod`
1. **Configure the `terraform` block in your `${TERRAFORM_ROOT}/main.tf` with a remote backend**:
   ```
   backend "remote" {
     hostname = "app.terraform.io"
     organization = "org"

     workspaces {
       prefix = "org-"
     }
   }
   ```
1. **Run `terraform init` to pick up the changes.** You should see something similar in the output:
   ```
   Initializing the backend...

   Successfully configured the backend "remote"! Terraform will automatically
   use this backend unless the backend configuration changes.

   The currently selected workspace (default) does not exist.
   This is expected behavior when the selected workspace did not have an
   existing non-empty state. Please enter a number to select a workspace:
  
   1. dev
   2. prod
   3. stage

   Enter a value: 

   ```

> **[BUG]: Disable Remote planning per configured workspace.**
> _There is one last CAVEAT to get all of this working._
>
> By default, Terraform Cloud uses the `Remote` Execution Mode when attempting to run `terraform plan`. Unfortunately, this
> does not seem to work with the AWS credentials. While the use of AWS local credentials works with `Local` planning, for some
> reason, it _FAILS_ when the _default_ `Remote` Execution Mode is configured. _This behavior occurs even if the
> `shared_credentials_file` parameter is set in your root Module's `provider` block or if you try setting the ENV_VAR in the
> Terraform Cloud UI for the Workspace._
>
> ***The work-around is to go to each Workspace's > `Settings -> General Settings` and change the Execution Mode to `Local`.***
>
> **Terraform Cloud Workspace Settings:**
>
> ![Workspace Settings -> General](https://user-images.githubusercontent.com/165323/104960154-eb227a00-5990-11eb-925a-a7adf2c86669.png)
>
> **Execution Mode Default Setting:**
>
> ![Execution Mode](https://user-images.githubusercontent.com/165323/104960152-ea89e380-5990-11eb-886b-7e5fb6a3ab82.png)
>
> _The Workstate's state file will be stored on > Terraform Cloud, but the `terraform plan` will run locally_.
>
> Here's an example of the output when `Remote` execution is set for the Terraform Cloud Workspace, but fails:
> ```console
> % terraform plan 
> Running plan in the remote backend. Output will stream here. Pressing Ctrl-C
> will stop streaming the logs, but will not stop the plan running remotely.
> 
> Preparing the remote plan...
> 
> To view this run in a browser, visit:
> https://app.terraform.io/app/scrb/scrb/runs/run-ynhciGC5Dp5CKHgy
>
> Waiting for the plan to start...
> 
> Terraform v0.14.4
> Configuring remote state backend...
> Initializing Terraform configuration...
> 
> Error: No valid credential sources found for AWS Provider.
>         Please see https://terraform.io/docs/providers/aws/index.html for more information on
>         providing credentials for the AWS Provider
>
> on example.tf line 17, in provider "aws":
> 17: provider "aws" {
> ```

> **[BUG]: Misleading error message on misconfiguration.**
> If you forget to specify or use the `workspace.prefix` directive in your configuration, `terraform` operations will fail
> opaquely with the following message:
> ```console
> % terraform workspace list
> workspaces not supported
> ```
> Ideally, a more informative error message would point you at the issue :(

# DevOps Workflow

Now that you've set up your project repository, Workspaces, environment variables, how do you incorporate this into your DevOps pipeline? The following guidance assumes an AWS provider, but should apply to any supported cloud provider.

Here are a few suggestions:

1. **There should be 3 branches in your Terraform project's git repo which correspond to each of the different deployments/Workspaces - `dev`, `stage`, and `prod`.** Having different branches isolates changes from propagating automatically to QA/production environments. ***[Optional - You can use the `main` (historically called `master` branch of your git repo as `prod`].*** In any case, pushing Terraform changes to `stage` or `prod` should be limited to automated/audited processes only. This allows changes to be gated on assurance tests before becoming live on production.
1. **Developers making changes to the infrastructure should _ONLY_ use feature branches off of the `dev` branch.** When the Terraform change is ready, it is merged into the `dev` branch. This is the only branch developer can commit changes to directly and should be enforced in the Terraform git repo.
1. **When changes are made to `dev`, they are pulled into `stage` branch for testing.** Ideally, there are automated tests which verify that there are no breaking changes and that the deployed application passes the required assurance tests/processes before being promoted to production.
1. **Finally, when changes to `stage` have passed, the Terraform changes are merged into `prod`.** This step should be _fully_ automated using a CI/CD system of your choice.

Note that:

- **For each stage, a set of corresponding `.tfvars` is used.** For example, to perform a developer deployment, the `dev.tfvars` are used as an argument to the Terraform operation (i.e. `terraform plan -var-file=./env/dev.tfvars`).
- **It's also recommended that different IAM accounts are associated with each stage.** Following AWS IAM guidelines, there should be a IAM group for developers which give them permissions for creating/deploying development _ONLY_ infrastructure. Each developer should belong to the developer IAM group. For users who are responsible for `stage` environments, another group can be created with the requiste permissions, or alternatively an assumable role. For production deploy, the CI system should be configured with a limited set of credentials to pull the change in the Terraform repository and deploy on production infrastructure.
- **Each deployment environment will have it's own resources (meaning that all assets created within the deployment environment will be _UNIQUE_ to it).** This ensures that environments deployed by users will not accidentally use privileges associated with these environments to update the wrong environment.
- **State files should be stored remotely using Terraform Cloud.** This ensures that there is an audited trail of state changes which can potentially be used to restore previous state if a rollback is required.
- **There's a cost to maintaining different deployment environments due to the duplication of resources.** _The benefit is the clear separation of deployment assets and the privileges required to access/deploy them._

# Miscellaneous

## Why is there a [Keybase.io](https://keybase.io) dependency for the AWS Lambda Module?

If you look closely at the Terraform Registry AWS Lambda Module, you'll note that configuration requires a Keybase.io user. The purpose of this dependency is to be able to use a public PGP key for the purpose of encrypting credentials for the Lambda service. Unfortunately, there doesn't seem to be an alternative (such as having a private/public PGP key to point the configuration towards or other public keyservers which might already contain a public PGP key). To be able to fulfill this dependency requires the following steps:

1. Create a PGP key pair.
1. Create a Keybase.io account.
1. Upload the public key to your Keybase.io account. This will require verification using the private key generated.
1. Once complete, you should be able to refer to the keybase PGP key using the `keybase:username` directive. This will resolve  a public Keybase URL (https://keybase.io/username/pgp_keys.asc) where the public PGP can be downloaded.

# Conclusion

Hopefully, this give you a roadmap to get started on your Terraform journey. There should be enough to avoid the common Terraform pitfalls and provide you a scalable/extensible architecture for your Terraform project.

Comments/feedback are welcome!

# TODO

- [ ] Multi-provider single monolith Terraform repos? Is the ultimate redundancy/resiliency solution to deploy your cloud infrastructure to multiple providers? If so, who is doing this?