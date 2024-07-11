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
      version = "1.0.1"
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

Some Strong Network resources can be managed by Terraform. At the moment these are
* Users
* Organizations
* Projects

Once managed by Terraform, the state of the resource on the platform will match the Terraform configuration file. This means that e.g. changing an organization's name, owner ID, or a Project's member list from the file will enact the corresponding changes on the platform. Removing resources from the file will delete them from the platform as well.

Below are some examples of how to manage these resources.

## Users
```
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
```
resource "strong_user" "user1" {
    email = "thor@strong.network"
    full_name = "Peter Project Owner"
    identity_provider = 1
    user_type = 3
}

resource "strong_organization" "organization" {
    name = "Strong Network"
    owner_id = strong_user.user1.id
}
```

## Projects
```
resource "strong_user" "organization_owner" {
    email = "thor@strong.network"
    full_name = "Olaf Organization Owner"
    identity_provider = 1
    user_type = 3
}

resource "strong_organization" "organization" {
    name = "Strong Network"
    owner_id = strong_user.user1.id
}

resource "strong_user" "project_member" {
    email = "thor@strong.network"
    full_name = "Daniel Developer"
    identity_provider = 1
    user_type = 3
}

resource "strong_user" "project_member_2" {
    email = "thor@strong.network"
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
