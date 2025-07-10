# Terraform Associate – Quick‑Revision Cheat Sheet

## 1. Providers
- **Default provider block not required** – Terraform automatically creates a default provider configuration when it can infer the provider from a resource (e.g., `resource "random_pet" "my_pet" {}`).
- Add an explicit `provider` block only when you need to set non-default arguments (such as region) or when creating multiple configurations with `alias`.
- **Who maintains providers?**
  - Individuals
  - Community teams
  - HashiCorp
  - Cloud vendors
- **Version constraints** are set in the `terraform { required_providers { ... } }` block.
  - `source = "hashicorp/aws"` – the official global registry address.
  - `version = ">= 4.0"` or pessimistic `version = "~> 4.0"` (allows `4.x` but not `5.0`).
- **Provider references:** Since Terraform 0.13+, only the provider's **local name** (e.g., `aws`, `azurerm`) is used in resource blocks.
- **`.terraform.lock.hcl`** – created/updated by `terraform init`. It records the exact provider versions and checksums so that every team member uses identical builds. **Commit this file to version control.**

## 2. State & Refresh
| Topic | Key Point |
| :--- | :--- |
| **Purpose of State** | Stores bindings between objects in your configuration and the real-world resources they manage. |
| **State Drift** | A difference between the real-world infrastructure and the information recorded in the state file. |
| **Implicit Refresh** | `terraform plan` and `terraform apply` automatically check for drift and update the state file with any changes found *before* calculating the plan. |
| **`terraform refresh`** | Legacy command that only detects drift and updates the state file. This logic is now part of `plan` and `apply`. |
| **Disabling Refresh** | Use `terraform plan -refresh=false` to skip the refresh step when you know no drift has occurred. |
| **Default State File** | `terraform.tfstate` in the working directory. |
| **Previewing Destruction** | Dry-run: `terraform plan -destroy`.<br>Interactive: `terraform destroy` (prints the plan and prompts for confirmation). |

## 3. Terraform Enterprise & Cloud (TFE/TFC)
- **Exclusive Features**
  - Secure variable storage
  - Sentinel policies
  - Private module registry
  - Governance & auditing
- Features like multi-cloud provisioning, `terraform plan`, and workspaces also exist in the open-source CLI.

## 4. Functions & Expressions
| Function Type | Valid Examples | Not the Same Type |
| :--- | :--- | :--- |
| **String Functions** | `split()`, `join()`, `chomp()` | `slice()` (for lists) |
| **Collection Functions** | `length()`, `keys()`, `values()`, `element()` | `format()` (for strings) |

**Build a list of IDs from `var.list_of_objects`:**
- **Splat Expression:** `var.list_of_objects[*].id`
- **For Expression:** `[for o in var.list_of_objects : o.id]`

> `[ var.list_of_objects[*].id ]` creates a list-within-a-list. `{ for o in var.list_of_objects : o.name => o.id }` creates a map, not a plain list.

## 5. Variables & Locals
- **Variable Definition:** the only required argument is the variable's name.
  ```hcl
  variable "instance_type" {}
  ```
- **Variable Precedence (lowest to highest):**
  1. `default` values in configuration
  2. `*.tfvars` files (alphabetical order)
  3. `-var-file="<file>"` flag
  4. `-var="<key>=<value>"` flag
  5. Environment variables (`TF_VAR_<name>`)
- **Locals:** can reference other locals defined earlier in the same block, but circular dependencies are not allowed.
  ```hcl
  locals {
    base_name = "app-${var.env}"
    full_name = "${local.base_name}-web"
  }
  ```

## 6. Provisioners
- Provisioners (`remote-exec`, `local-exec`) run scripts on a resource **after it is created** or **before it is destroyed**.
- A `provisioner` block **must** be nested inside a `resource` block.
- They are a **last resort** – prefer provider arguments, user data scripts, or configuration management tools (like Ansible).

## 7. Modules & Registry
- **Module Sources**
  - Public Terraform Registry (`source = "terraform-aws-modules/vpc/aws"`)
  - Git repositories (`source = "git::https://..."`)
  - Local paths (`source = "./modules/webserver"`)
- **Pinning a Version:** use the `version` argument inside the `module` block: `version = "1.0.0"`.
- **Exposing Child Module Outputs:**
  1. Declare an `output` block inside the child module's code.
  2. In the root module, declare another `output` that references the child's output.

## 8. Best-Practice Workflow
- Follow the sequence: `init` → `validate` → `plan` → `apply`.
- Use a CI/CD pipeline with version control to automate this workflow.
  - **Repeatability:** every change is executed consistently.
  - **Peer Review:** changes are reviewed via pull requests.
  - **Audit Trail:** Git history tracks every infrastructure change.

## 9. CLI Flags & Syntax Nuggets
| Task | Command / Syntax |
| :--- | :--- |
| Initialize & upgrade providers | `terraform init -upgrade` |
| Check syntax & validate config | `terraform validate` |
| Check code formatting | `terraform fmt -check -recursive` |
| Save execution plan | `terraform plan -out=tfplan` |
| Apply a saved plan | `terraform apply tfplan` |
| Create a destroy plan | `terraform plan -destroy` |
| Taint a resource (force replacement) | `terraform taint <address>` |
| Replace a resource (v1.1+) | `terraform apply -replace="<address>"` |
| Pass sensitive variable (in CI/CD) | `-var="secret=${SECRET_VAR}"` or `-var-file="secure.tfvars"` |
| Import existing resource | `terraform import <address> <provider-id>` |
| List resources in state | `terraform state list` |
| Show specific resource state | `terraform state show <address>` |
| Remove resource from state | `terraform state rm <address>` |
| Enable debug logging | `TF_LOG=DEBUG terraform plan` (set `TF_LOG_PATH` for file output) |

## 10. Miscellaneous Facts
- **Resource Address:** `resource "TYPE" "NAME" {}` is referenced as `TYPE.NAME` (e.g., `aws_instance.web`).
- **Data Source Reference:** `data.<TYPE>.<NAME>.<ATTRIBUTE>` (e.g., `data.aws_ami.ubuntu.id`).
- **Backend Configuration:** the `backend` block is nested inside the top-level `terraform` block. One backend configuration maps to exactly **one** remote workspace.
- **Partial Backend Configuration:** avoid hard‑coding credentials in the `backend` block. Omit them and provide them dynamically during `init` with `-backend-config` flags.
- **Migrating State:** to change backends, update the `backend` block and run `terraform init`. Terraform will prompt you to migrate the state.

## 11. Resource `count` & `for_each`
- **`count`** creates a **list** of resources. Access instances with a zero-based index.
  - Example: `aws_instance.web[0].id`
- **`for_each`** creates a **map** of resources. Requires a `map` or a `set of strings`.
  - Example: `aws_instance.web["us-east-1a"].id`
- **Splat Expressions (`*`)**
  - Works with `count`: `aws_instance.web[*].id` → list of IDs.
  - Does not work directly with `for_each` attributes.
  - Use a `for` expression (`[for inst in aws_instance.web : inst.id]`) or the `values()` function (`values(aws_instance.web)[*].id`).
- **Repeated Nested Blocks**
  - Use a splat expression to get a list of attributes from all nested blocks of the same type.
  - Example: `aws_instance.example.ebs_block_device[*].volume_id`.

## 12. Command Reference
### Initialization & Backend Migration
- `terraform init` – bootstraps the working directory and automatically migrates state when backend configuration changes (e.g., local → S3).

### Planning & Applying Changes
- `terraform plan` – previews changes without applying.
- `terraform apply` – applies the changes to your infrastructure.
- `terraform plan -destroy` – shows what would be destroyed (preview before deletion).
- `terraform destroy` – shows and then deletes resources after confirmation.

### State Sync & Replacements
- `terraform refresh` – updates the state file to reflect manual changes or deletions.
- `terraform apply -replace=<resource>` – marks a resource for replacement in a single step.

### Importing & Managing External Resources
- `terraform import <address> <id>` – brings an existing resource under Terraform management.

### Other Useful Commands
- `terraform validate` – validates HCL syntax.
- `terraform fmt` – formats your code.
- `terraform output` – displays output values.
- `terraform state rm <resource>` – removes a resource from state without destroying it.

## 13. Quick Reference Table
| Goal | Command |
| :--- | :--- |
| Initialize or migrate backend | `terraform init` |
| Preview infrastructure changes | `terraform plan` |
| Apply infrastructure changes | `terraform apply` |
| Preview deletion of all resources | `terraform plan -destroy` |
| Destroy resources after review | `terraform destroy` |
| Refresh state to match reality | `terraform refresh` |
| Force replace specific resource | `terraform apply -replace=resource.address` |
| Import existing resource | `terraform import <resource.address> <external-resource-id>` |
| Remove resource from tracking | `terraform state rm <resource.address>` |
| Format & validate code | `terraform fmt`, `terraform validate` |
| View configured outputs | `terraform output` |
