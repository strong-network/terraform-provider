# Strong Network Terraform Provider

This is a guide for the Strong Network Terraform provider. It can be downloaded with the Strong Network installer image and must be placed in the `~/.terraform.d/plugins` directory on your system.
To download the provider, run the Strong Network Installer for the release you are using:
```
docker run -it --rm -v ${PWD}:/strong-network/shared \
              strongnetwork/strong_installer:<VERSION>
```
Then fetch the provider using the command
````
./strong-cli get-terraform
```
The provider binary will then be downloaded to your system.

The provider has to be placed in the local Terraform registry. The full path should look like `~/.terraform.d/plugins/strong.network/strong-network/strong/<VERSION>/<SYSTEM>/strong-terraform-provider`. Replace VERSION (provider version, not Strong Network version) and SYSTEM with the appropriate values (e.g. 1.0.1 and linux_amd64). After placing the provider in the correct directory, run:
```
terraform init -plugin-dir="~/.terraform.d/plugins"
```
To use the Strong Network Terraform provider in your Terraform configuration, you need to add the Strong provider as a `required_provider` in your Terraform configuration file. Here is a code snippet to show how to do this:

```hcl
terraform {
  required_providers {
    strong = {
      version = "1.2.0"
      source  = "strong.network/strong-network/strong"
    }
  }
}

variable "api_token" {
  description = "API token for accessing Strong network"
  type        = string
}

variable "deployment_url" {
  description = "URL for the deployment environment"
  type        = string
}

provider "strong" {
  api_token = var.api_token
  deployment_url = var.deployment_url
}
```
In `.terraformc` add the configuration to retrieve the provider from your locally installed plugins:
```hcl
terraform {
  required_providers {
    strong = {
      version = "1.0.1"
      source  = "strong-network/strong"
    }
  }

  provider_installation {
    filesystem_mirror {
      path    = "/home/developer/.terraform.d/plugins"
      include = ["strong-network/strong"]
    }
    direct {
      exclude = ["strong-network/strong"]
    }
  }
}
```
In `terraform.tfvars` keep the API token and deployment URL variables:
```
api_token = "<TOKEN>"
deployment_url = "https://example.conceptcloud.network"
```

# Usage

Some Strong Network resources can be managed by Terraform. At the moment these are:
* Users
* Organizations
* Projects
* User Groups
* Workspace Templates

Once managed by Terraform, the state of the resource on the platform will match the Terraform configuration file. This means that e.g. changing an organization's name, owner ID, or a Project's member list from the file will enact the corresponding changes on the platform. Removing resources from the file will delete them from the platform as well.

Below are some examples of how to manage these resources.

## Users
```hcl
resource "strong_user" "user1" {
    email = "thor@strong.network"
    full_name = "Thor Michaels"
    identity_provider = 1
    user_type = 3
}

resource "strong_user" "user2" {
    email = "harry@strong.network"
    identity_provider = 1
    user_type = 3
    full_name = "Harry Smith"
}
```

## Organizations
```hcl
resource "strong_user" "org_owner" {
    email = "peter@strong.network"
    full_name = "Peter Project Owner"
    identity_provider = 1
    user_type = 3
}

resource "strong_organization" "organization" {
    name = "Strong Network"
    owner_id = strong_user.org_owner.id
}
```

## Projects
```hcl
resource "strong_user" "organization_owner" {
    email = "olaf@strong.network"
    full_name = "Olaf Organization Owner"
    identity_provider = 1
    user_type = 3
}

resource "strong_organization" "organization" {
    name = "Strong Network"
    owner_id = strong_user.organization_owner.id
}

resource "strong_user" "project_member" {
    email = "daniel@strong.network"
    full_name = "Daniel Developer"
    identity_provider = 1
    user_type = 3
}

resource "strong_user" "project_member_2" {
    email = "derrick@strong.network"
    full_name = "Derrick Developer"
    identity_provider = 1
    user_type = 3
}

resource "strong_project" "project" {
  name = "Frontend Development"
  owner_id = strong_user.organization_owner.id
  organization_id = strong_organization.organization.id

  member {
    id = strong_user.organization_owner.id
    role = "Project Owner"
  }

  member {
    id = strong_user.project_member.id
    role = "Developer"
  }

  member {
    id = strong_user.project_member_2.id
    role = "Developer"
  }
}
```

## User Groups

User groups allow you to organize users into logical groups for easier management.

```hcl
resource "strong_user" "dev1" {
    email = "alice@strong.network"
    full_name = "Alice Developer"
    identity_provider = 1
    user_type = 3
}

resource "strong_user" "dev2" {
    email = "bob@strong.network"
    full_name = "Bob Developer"
    identity_provider = 1
    user_type = 3
}

resource "strong_user_group" "developers" {
    name        = "Development Team"
    description = "All developers in the organization"
    members     = [
        strong_user.dev1.id,
        strong_user.dev2.id
    ]
}
```

You can also reference users by email instead of ID:

```hcl
resource "strong_user_group" "developers" {
    name        = "Development Team"
    description = "All developers in the organization"
    members     = [
        "alice@strong.network",
        "bob@strong.network"
    ]
}
```

### User Group Arguments

| Argument | Type | Required | Description |
|----------|------|----------|-------------|
| `name` | string | Yes | The name of the user group |
| `description` | string | No | A description of the user group |
| `members` | set of strings | No | List of user IDs or emails that are members of this group |

## Workspace Templates

Workspace templates define the configuration for workspaces in a project, including the image, resources, secrets, and security settings.

```hcl
resource "strong_workspace_template" "dev_template" {
    name       = "Development Environment"
    project_id = strong_project.project.id

    version {
        version   = 1
        region_id = 1

        workspace_image {
            id  = "123456"
            tag = "latest"
        }

        workspace_specs {
            cpu    = 4
            memory = 8
            disk   = 50
        }

        workspace_access_items = [0, 2, 3]  # 0=VSCode, 2=Web Terminal, 3=SSH

        clipboard_settings {
            monitor                               = true
            clipboard_restricted                  = false
            clipboard_character_restriction       = 0
            clipboard_restricted_paste            = false
            clipboard_character_restriction_paste = 0
            enable_supervised_copy                = false
        }

        before_startup_script = "echo 'Starting workspace...'"
        after_startup_script  = "echo 'Workspace ready!'"
        default_folder        = "/home/developer/project"

        personal_ssh_identity = true

        # Optional: Inject platform/organization/project secrets
        injected_secrets_as_env  = []
        injected_secrets_as_file = []

        # Optional: Define local secrets
        local_secrets_as_env {
            secret_name = "API_KEY"
            content     = "secret-value"
        }

        # Optional: Network policies
        policy_ids = []

        # Optional: Custom workspace apps
        workspace_apps {
            port     = 8080
            name     = "Web App"
            use_https = true
        }
    }
}
```

### Workspace Template Arguments

| Argument | Type | Required | Description |
|----------|------|----------|-------------|
| `name` | string | Yes | The name of the workspace template |
| `project_id` | int | Yes | The ID of the project this template belongs to |
| `version` | list | Yes | One or more version blocks (see below) |

### Version Block Arguments

| Argument | Type | Required | Description |
|----------|------|----------|-------------|
| `version` | int | Yes | Version number for this template version |
| `region_id` | int | Yes | Region ID where workspaces will be created |
| `workspace_image` | block | Yes | Image configuration (id and tag) |
| `workspace_specs` | block | Yes | Resource specifications (cpu, memory, disk) |
| `workspace_access_items` | list of ints | No | Access methods: 0=VSCode, 2=Web Terminal, 3=SSH |
| `clipboard_settings` | block | No | Clipboard security settings |
| `before_startup_script` | string | No | Script to run before workspace starts |
| `after_startup_script` | string | No | Script to run after workspace starts |
| `default_folder` | string | No | Default folder to open in VSCode |
| `personal_ssh_identity` | bool | No | Enable personal SSH identity |
| `injected_secrets_as_env` | list of ints | No | Secret IDs to inject as environment variables |
| `injected_secrets_as_file` | list of ints | No | Secret IDs to inject as files |
| `local_secrets_as_env` | list of blocks | No | Local secrets as environment variables |
| `local_secrets_as_file` | list of blocks | No | Local secrets as files |
| `policy_ids` | list of strings | No | Network policy IDs to apply |
| `workspace_apps` | list of blocks | No | Custom workspace applications |
| `workspace_schedule` | block | No | Schedule settings (idle_timeout, timeout_outside_schedule) |
