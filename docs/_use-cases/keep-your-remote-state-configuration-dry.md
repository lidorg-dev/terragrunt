---
layout: collection-browser-doc
title: Keep your remote state configuration DRY
excerpt: Learn how to create and manage remote state configuration.
tags: ["DRY", "remote"]
order: 101
nav_title: All Use Cases
nav_title_link: /use-cases/
---

## Keep your remote state configuration DRY

  - [Motivation](#motivation)

  - [Filling in remote state settings with Terragrunt](#filling-in-remote-state-settings-with-terragrunt)

  - [Create remote state and locking resources automatically](#create-remote-state-and-locking-resources-automatically)

### Motivation

Terraform supports [remote state storage](https://www.terraform.io/docs/state/remote.html) via a variety of [backends](https://www.terraform.io/docs/backends) that you normally configure in your `.tf` files as follows:

``` hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "frontend-app/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "my-lock-table"
  }
}
```

Unfortunately, the `backend` configuration does not support expressions, variables, or functions. This makes it hard to keep your code [DRY](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself) if you have multiple Terraform modules. For example, consider the following folder structure, which uses different Terraform modules to deploy a backend app, frontend app, MySQL database, and a VPC:

    ├── backend-app
    │   └── main.tf
    ├── frontend-app
    │   └── main.tf
    ├── mysql
    │   └── main.tf
    └── vpc
        └── main.tf

To use remote state with each of these modules, you would have to copy/paste the exact same `backend` configuration into each of the `main.tf` files. The only thing that would differ between the configurations would be the `key` parameter: e.g., the `key` for `mysql/main.tf` might be `mysql/terraform.tfstate` and the `key` for `frontend-app/main.tf` might be `frontend-app/terraform.tfstate`.

To keep your remote state configuration DRY, you can use Terragrunt. You still have to specify the `backend` you want to use in each module, but instead of copying and pasting the configuration settings over and over again into each `main.tf` file, you can leave them blank (this is known as [partial configuration](https://www.terraform.io/docs/backends/config.html#partial-configuration)):

``` hcl
terraform {
  # The configuration for this backend will be filled in by Terragrunt
  backend "s3" {}
}
```

### Filling in remote state settings with Terragrunt

To fill in the settings via Terragrunt, create a `terragrunt.hcl` file in the root folder, plus one `terragrunt.hcl` file in each of the Terraform modules:

    ├── terragrunt.hcl
    ├── backend-app
    │   ├── main.tf
    │   └── terragrunt.hcl
    ├── frontend-app
    │   ├── main.tf
    │   └── terragrunt.hcl
    ├── mysql
    │   ├── main.tf
    │   └── terragrunt.hcl
    └── vpc
        ├── main.tf
        └── terragrunt.hcl

In your **root** `terragrunt.hcl` file, you can define your entire remote state configuration just once in a `remote_state` block (which supports all the same [backend types](https://www.terraform.io/docs/backends/types/index.html) as Terraform), as follows:

``` hcl
remote_state {
  backend = "s3"
  config = {
    bucket         = "my-terraform-state"
    key            = "${path_relative_to_include()}/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "my-lock-table"
  }
}
```

In each of the **child** `terragrunt.hcl` files, such as `mysql/terragrunt.hcl`, you can tell Terragrunt to automatically include all the settings from the root `terragrunt.hcl` file as follows:

``` hcl
include {
  path = find_in_parent_folders()
}
```

The `include` block tells Terragrunt to use the exact same Terragrunt configuration from the `terragrunt.hcl` file specified via the `path` parameter. It behaves exactly as if you had copy/pasted the Terraform configuration from the included file `remote_state` configuration into `mysql/terragrunt.hcl`, but this approach is much easier to maintain\!

The next time you run `terragrunt`, it will automatically configure all the settings in the `remote_state.config` block, if they aren’t configured already, by calling [terraform init](https://www.terraform.io/docs/commands/init.html).

The `terragrunt.hcl` files above use two Terragrunt built-in functions:

  - `find_in_parent_folders()`: This function returns the path to the first `terragrunt.hcl` file it finds in the parent folders above the current `terragrunt.hcl` file. In the example above, the call to `find_in_parent_folders()` in `mysql/terragrunt.hcl` will return `../terragrunt.hcl`. This way, you don’t have to hard code the `path` parameter in every module.

  - `path_relative_to_include()`: This function returns the relative path between the current `terragrunt.hcl` file and the path specified in its `include` block. We typically use this in a root `terragrunt.hcl` file so that each Terraform child module stores its Terraform state at a different `key`. For example, the `mysql` module will have its `key` parameter resolve to `mysql/terraform.tfstate` and the `frontend-app` module will have its `key` parameter resolve to `frontend-app/terraform.tfstate`.

See [the Built-in Functions docs]({{site.baseurl}}/docs/features/built-in-functions/#built-in-functions) for more info.

Check out the [terragrunt-infrastructure-modules-example](https://github.com/gruntwork-io/terragrunt-infrastructure-modules-example) and [terragrunt-infrastructure-live-example](https://github.com/gruntwork-io/terragrunt-infrastructure-live-example) repos for fully-working sample code that demonstrates how to use Terragrunt to manage remote state.

### Rules for merging parent and child configurations

The child `.hcl` file’s `terraform` settings will be merged into the parent file’s `terraform` settings as follows:

  - If an `extra_arguments` block in the child has the same name as an `extra_arguments` block in the parent, then the child’s block will override the parent’s.

      - Specifying an empty `extra_arguments` block in a child with the same name will effectively remove the parent’s block.

  - If an `extra_arguments` block in the child has a different name than `extra_arguments` blocks in the parent, then both the parent and child’s `extra_arguments` will be effective.

      - The child’s `extra_arguments` will be placed *after* the parent’s `extra_arguments` on the terraform command line.

      - Therefore, if a child’s and parent’s `extra_arguments` include `.tfvars` files with the same variable defined, the value from the `.tfvars` file from the child’s `extra_arguments` will be used by terraform.

  - If a `before_hook` or `after_hook` block in the child has the same name as the hook block in the parent, then the child’s block will override the parent’s.

      - Specifying an empty hook block in a child with the same name will effectively remove the parent’s block.

  - If a `before_hook` or `after_hook` block in the child has a different name than hook blocks in the parent, then both the parent and child’s hook blocks will be effective.

  - The `source` field in the child will override `source` field in the parent

Other settings in the child `.hcl` file override the respective settings in the parent.

### Create remote state and locking resources automatically

When you run `terragrunt` with `remote_state` configuration, it will automatically create the following resources if they don’t already exist:

  - **S3 bucket**: If you are using the [S3 backend](https://www.terraform.io/docs/backends/types/s3.html) for remote state storage and the `bucket` you specify in `remote_state.config` doesn’t already exist, Terragrunt will create it automatically, with [versioning](https://docs.aws.amazon.com/AmazonS3/latest/dev/Versioning.html), [server-side encryption](https://docs.aws.amazon.com/AmazonS3/latest/dev/UsingServerSideEncryption.html), and [access logging](https://docs.aws.amazon.com/AmazonS3/latest/dev/ServerLogs.html) enabled.

    In addition, you can let terragrunt tag the bucket with custom tags that you specify in `remote_state.config.s3_bucket_tags`.

  - **DynamoDB table**: If you are using the [S3 backend](https://www.terraform.io/docs/backends/types/s3.html) for remote state storage and you specify a `dynamodb_table` (a [DynamoDB table used for locking](https://www.terraform.io/docs/backends/types/s3.html#dynamodb_table)) in `remote_state.config`, if that table doesn’t already exist, Terragrunt will create it automatically, with [server-side encryption](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/EncryptionAtRest.html) enabled, including a primary key called `LockID`.

    In addition, you can let terragrunt tag the DynamoDB table with custom tags that you specify in `remote_state.config.dynamodb_table_tags`.

  - **GCS bucket**: If you are using the [GCS backend](https://www.terraform.io/docs/backends/types/gcs.html) for remote state storage and the `bucket` you specify in `remote_state.config` doesn’t already exist, Terragrunt will create it automatically, with [versioning](https://cloud.google.com/storage/docs/object-versioning) enabled. For this to work correctly you must also specify `project` and `location` keys in `remote_state.config`, so Terragrunt knows where to create the bucket. You will also need to supply valid credentials using either `remote_state.config.credentials` or by setting the `GOOGLE_APPLICATION_CREDENTIALS` environment variable. If you want to skip creating the bucket entirely, simply set `skip_bucket_creation` to `true` and Terragrunt will assume the bucket has already been created. If you don’t specify `bucket` in `remote_state` then terragrunt will assume that you will pass `bucket` through `-backend-config` in `extra_arguments`.

    We also strongly recommend you enable [Cloud Audit Logs](https://cloud.google.com/storage/docs/access-logs) to audit and track API operations performed against the state bucket.

    In addition, you can let Terragrunt label the bucket with custom labels that you specify in `remote_state.config.gcs_bucket_labels`.

**Note**: If you specify a `profile` key in `remote_state.config`, Terragrunt will automatically use this AWS profile when creating the S3 bucket or DynamoDB table.

**Note**: You can disable automatic remote state initialization by setting `remote_state.disable_init`, this will skip the automatic creation of remote state resources and will execute `terraform init` passing the `backend=false` option. This can be handy when running commands such as `validate-all` as part of a CI process where you do not want to initialize remote state.

The following example demonstrates using an environment variable to configure this option:

``` hcl
remote_state {
  # ...

  disable_init = tobool(get_env("TERRAGRUNT_DISABLE_INIT", "false"))
}
```

### S3-specific remote state settings

For the `s3` backend, the following config options can be used for S3-compatible object stores, as necessary:

``` hcl
remote_state {
  # ...

  skip_bucket_versioning         = true # use only if the object store does not support versioning
  skip_bucket_ssencryption       = true # use only if non-encrypted Terraform State is required and/or the object store does not support server-side encryption
  skip_bucket_accesslogging      = true # use only if the cost for the extra object space is undesirable or the object store does not support access logging
  enable_lock_table_ssencryption = true # use only if non-encrypted DynamoDB Lock Table for the Terraform State is required and/or the NoSQL database service does not support server-side encryption

  shared_credentials_file     = "/path/to/credentials/file"
  skip_credentials_validation = true
  skip_metadata_api_check     = true
  force_path_style            = true
}
```

If you experience an error for any of these configurations, confirm you are using Terraform v0.12.2 or greater.

Further, the config options `s3_bucket_tags`, `dynamodb_table_tags`, `skip_bucket_versioning`, `skip_bucket_ssencryption`, `skip_bucket_accesslogging`, and `enable_lock_table_ssencryption` are only valid for backend `s3`. They are used by terragrunt and are **not** passed on to terraform. See section [Create remote state and locking resources automatically](#create-remote-state-and-locking-resources-automatically).

### GCS-specific remote state settings

For the `gcs` backend, the following config options can be used for GCS-compatible object stores, as necessary:

``` hcl
remote_state {
 # ...

 skip_bucket_versioning = true # use only if the object store does not support versioning

 encryption_key = "GOOGLE_ENCRYPTION_KEY"
}
```

If you experience an error for any of these configurations, confirm you are using Terraform v0.12.0 or greater.

Further, the config options `gcs_bucket_labels` and `skip_bucket_versioning` are only valid for the backend `gcs`. They are used by terragrunt and are **not** passed on to terraform. See section [Create remote state and locking resources automatically](#create-remote-state-and-locking-resources-automatically).
