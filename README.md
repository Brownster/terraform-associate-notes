# Terraform Associate – Quick‑Revision Cheat Sheet

---

## 1. Providers

*   **Default provider block not required** – Terraform automatically creates a default provider configuration when it can infer the provider from a resource (e.g., `resource "random_pet" "my_pet" {}`).
*   Add an explicit `provider` block only when you need to set non-default arguments (like region) or create multiple configurations using `alias`.
*   **Who maintains providers?**
    *   Individuals 🧑‍💻 • Community teams 🤝 • HashiCorp 🏢 • Cloud vendors ☁️
*   **Version constraints** are set in the `terraform { required_providers { ... } }` block.
    *   `source = "hashicorp/aws"` – The official global registry address.
    *   `version = ">= 4.0"` or pessimistic `version = "~> 4.0"` (allows `4.x` but not `5.0`).
*   **Provider references:** Since Terraform 0.13+, you only use the provider's **local name** (e.g., `aws`, `azurerm`) in resource blocks.
*   **`.terraform.lock.hcl`** – Created/updated by `terraform init`. It records the exact provider versions and checksums to ensure every team member uses the identical builds. **Commit this file to version control.**

---

## 2. State & Refresh

| Topic | Key Point |
| :--- | :--- |
| **Purpose of State** | To store bindings between objects in your configuration and the real-world resources they manage. |
| **State Drift** | A difference between the real-world infrastructure and the information recorded in the state file. |
| **Implicit Refresh** | `terraform plan` and `terraform apply` automatically check for drift and update the state file with any changes found *before* calculating the plan. |
| **`terraform refresh`** | A legacy command that only detects drift and updates the state file. It is now part of `plan` and `apply`. |
| **Disabling Refresh** | Use `terraform plan -refresh=false` to skip the refresh step, useful for speed when you know no drift has occurred. |
| **Default State File** | `terraform.tfstate` is the default local state file in the working directory. |
| **Previewing Destruction** | **Dry-run:** `terraform plan -destroy`. <br> **Interactive:** `terraform destroy` (prints the same plan, then prompts for confirmation). |

---

## 3. Terraform Enterprise & Cloud (TFE/TFC)

*   **Exclusive Features:**
    *   **Secure Variable Storage:** Centralized, encrypted storage for sensitive variables, with masking in UI/logs.
    *   **Sentinel Policies:** Policy-as-code to enforce security, compliance, and cost controls before an `apply`.
    *   **Private Module Registry:** A private, internal registry for sharing modules across your organization.
    *   **Governance & Auditing:** Centralized run history, user permissions, and audit logs.
*   Features like multi-cloud provisioning, `terraform plan`, and workspaces **also exist in the open-source CLI**.

---

## 4. Functions & Expressions

| Function Type | Valid Examples | Not the Same Type |
| :--- | :--- | :--- |
| **String Functions** | `split()`, `join()`, `chomp()` | `slice()` (for lists) |
| **Collection Functions** | `length()`, `keys()`, `values()`, `element()` | `format()` (for strings) |

**Build a list of IDs from `var.list_of_objects`:**

*   **Splat Expression:** `var.list_of_objects[*].id`
*   **For-Expression:** `[for o in var.list_of_objects : o.id]`

> Note: `[ var.list_of_objects[*].id ]` creates a list-within-a-list. `{ for o in var.list_of_objects : o.name => o.id }` creates a map, not a plain list.

---

## 5. Variables & Locals

*   **Variable Definition:** The only required argument is the variable's name.
    ```hcl
    variable "instance_type" {}
    ```
*   **Variable Precedence (Lowest to Highest):**
    1.  `default` values in configuration
    2.  `*.tfvars` files (alphabetical order)
    3.  `-var-file="<file>"` flag
    4.  `-var="<key>=<value>"` flag
    5.  Environment variables (`TF_VAR_<name>`)
*   **Locals:** Can reference other locals defined earlier in the same block, but circular dependencies are not allowed.
    ```hcl
    locals {
      base_name = "app-${var.env}"
      full_name = "${local.base_name}-web"
    }
    ```

---

## 6. Provisioners

*   Provisioners (`remote-exec`, `local-exec`) are used to run scripts on a resource **after it is created** or **before it is destroyed**.
*   A `provisioner` block **must** be nested inside a `resource` block.
*   They are a **last resort**. Prefer provider arguments, user data scripts, or dedicated configuration management tools (like Ansible) whenever possible.

---

## 7. Modules & Registry

*   **Module Sources:** Modules can be sourced from:
    *   Public Terraform Registry (`source = "terraform-aws-modules/vpc/aws"`)
    *   Git repositories (`source = "git::https://...`")
    *   Local paths (`source = "./modules/webserver"`)
*   **Pinning a Version:** Use the `version` argument inside the `module` block: `version = "1.0.0"`.
*   **Exposing Child Module Outputs:**
    1.  Declare an `output` block **inside the child module's code** (`modules/webserver/outputs.tf`).
    2.  In the root module, declare another `output` that references the child's output.
        ```hcl
        # In root/main.tf
        output "instance_ip" {
          value = module.webserver.public_ip
        }
        ```

---

## 8. Best-Practice Workflow

*   The standard workflow follows the sequence: `init` → `validate` → `plan` → `apply`.
*   Use a **CI/CD pipeline** with version control (like Git) to automate this workflow. This ensures:
    *   **Repeatability:** Every change is executed consistently.
    *   **Peer Review:** Changes are reviewed via pull requests.
    *   **Audit Trail:** Git history tracks every infrastructure change.

---

## 9. CLI Flags & Syntax Nuggets

| Task | Command / Syntax |
| :--- | :--- |
| Initialize & Upgrade Providers | `terraform init -upgrade` |
| Check Syntax & Validate Config | `terraform validate` |
| Check Code Formatting | `terraform fmt -check -recursive` |
| Save Execution Plan | `terraform plan -out=tfplan` |
| Apply a Saved Plan | `terraform apply tfplan` |
| Create a Destroy Plan | `terraform plan -destroy` |
| Taint a Resource (force replacement) | `terraform taint <address>` |
| Replace a Resource (v1.1+) | `terraform apply -replace="<address>"` |
| Pass Sensitive Variable (in CI/CD) | `-var="secret=${SECRET_VAR}"` or `-var-file="secure.tfvars"` |
| Import Existing Resource | `terraform import <address> <provider-id>` |
| List Resources in State | `terraform state list` |
| Show Specific Resource State | `terraform state show <address>` |
| Remove Resource from State (Orphan) | `terraform state rm <address>` |
| Enable Debug Logging | `TF_LOG=DEBUG terraform plan` (set `TF_LOG_PATH` for file output) |

---

## 10. Miscellaneous Facts

*   **Resource Address:** `resource "TYPE" "NAME" {}` is referenced as `TYPE.NAME` (e.g., `aws_instance.web`).
*   **Data Source Reference:** `data.<TYPE>.<NAME>.<ATTRIBUTE>` (e.g., `data.aws_ami.ubuntu.id`).
*   **Backend Configuration:** The `backend` block is nested inside the top-level `terraform` block. One backend configuration maps to exactly **one** remote workspace.
*   **Partial Backend Configuration:** Avoid hard-coding credentials in the `backend` block. Omit them and provide them dynamically during `init` using the `-backend-config` flag.
*   **Migrating State:** To change backends, update the `backend` block in your code and run `terraform init`. Terraform will detect the change and prompt you to migrate the state.

---

## 11. Resource `count` & `for_each`

*   **`count`:** Creates a **list** of resources. Instances are accessed with a zero-based integer index.
    *   *Reference:* `aws_instance.web[0].id`
*   **`for_each`:** Creates a **map** of resources. Requires a `map` or a `set of strings`. Instances are accessed with the map/set key.
    *   *Reference:* `aws_instance.web["us-east-1a"].id`
*   **Splat Expressions (`*`)**
    *   Works with `count`: `aws_instance.web[*].id` → returns a list of IDs.
    *   **Does not work directly** with `for_each` for attributes.
    *   *Correct `for_each` equivalent:* Use a `for` expression (`[for instance in aws_instance.web : instance.id]`) or the `values()` function (`values(aws_instance.web)[*].id`).
*   **Repeated Nested Blocks:** Use a splat expression to get a list of attributes from all nested blocks of the same type.
    *   `aws_instance.example.ebs_block_device[*].volume_id` → returns a list of all attached volume IDs.
