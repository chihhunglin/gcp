# Terraform

## Terraform Fundamentals
terraform
touch instance.tf
```
resource "google_compute_instance" "terraform" {
  project      = "<PROJECT_ID>"
  name         = "terraform"
  machine_type = "n1-standard-1"
  zone         = "us-central1-a"

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-9"
    }
  }

  network_interface {
    network = "default"
    access_config {
    }
  }
}
```

terraform init
terraform plan
terraform apply
terraform show

## Infrastructure as Code with Terraform
touch main.tf
```
terraform {
  required_providers {
    google = {
      source = "hashicorp/google"
    }
  }
}

provider "google" {
  version = "3.5.0"

  project = "<PROJECT_ID>"
  region  = "us-central1"
  zone    = "us-central1-c"
}

resource "google_compute_network" "vpc_network" {
  name = "terraform-network"
}
```
terraform init
terraform apply
terraform show

```
resource "google_compute_instance" "vm_instance" {
  name         = "terraform-instance"
  machine_type = "f1-micro"

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-9"
    }
  }

  network_interface {
    network = google_compute_network.vpc_network.name
    access_config {
    }
  }
}
```

terraform destroy

terraform apply

```
resource "google_compute_address" "vm_static_ip" {
  name = "terraform-static-ip"
}
```
terraform plan

```
  network_interface {
    network = google_compute_network.vpc_network.self_link
    access_config {
      nat_ip = google_compute_address.vm_static_ip.address
    }
  }
```
terraform plan -out static_ip
terraform apply "static_ip"

```
# New resource for the storage bucket our application will use.
resource "google_storage_bucket" "example_bucket" {
  name     = "<UNIQUE-BUCKET-NAME>"
  location = "US"

  website {
    main_page_suffix = "index.html"
    not_found_page   = "404.html"
  }
}

# Create a new instance that uses the bucket
resource "google_compute_instance" "another_instance" {
  # Tells Terraform that this VM instance must be created only after the
  # storage bucket has been created.
  depends_on = [google_storage_bucket.example_bucket]

  name         = "terraform-instance-2"
  machine_type = "f1-micro"

  boot_disk {
    initialize_params {
      image = "cos-cloud/cos-stable"
    }
  }

  network_interface {
    network = google_compute_network.vpc_network.self_link
    access_config {
    }
  }
}
```

terraform plan
terraform apply

### Use terraform taint to tell Terraform to recreate the instance:
terraform taint google_compute_instance.vm_instance
terraform apply

## Interact with Terraform Modules
git clone https://github.com/terraform-google-modules/terraform-google-network
cd terraform-google-network
git checkout tags/v2.5.0 -b v2.5.0

gcloud config list --format 'value(core.project)'
vim variables.tf
```
variable "project_id" {
  description = "The project ID to host the network in"
  default     = "#FILL IN YOUR PROJECT ID HERE"
}
```
cd ~/terraform-google-network/examples/simple_project
terraform init
terraform apply
terraform destroy


cd ~
touch main.tf
mkdir -p modules/gcs-static-website-bucket

cd modules/gcs-static-website-bucket
touch main.tf
touch variables.tf
touch outputs.tf
vim README.md
```
# GCS static website bucket

This module provisions Cloud Storage buckets configured for static website hosting.
```

vim LICENSE
```
Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
```

vim main.tf
```
resource "google_storage_bucket" "bucket" {
  name               = var.name
  project            = var.project_id
  location           = var.location
  storage_class      = var.storage_class
  bucket_policy_only = var.bucket_policy_only
  labels             = var.labels
  force_destroy      = var.force_destroy

  versioning {
    enabled = var.versioning
  }

  dynamic "retention_policy" {
    for_each = var.retention_policy == null ? [] : [var.retention_policy]
    content {
      is_locked        = var.retention_policy.is_locked
      retention_period = var.retention_policy.retention_period
    }
  }

  dynamic "encryption" {
    for_each = var.encryption == null ? [] : [var.encryption]
    content {
      default_kms_key_name = var.encryption.default_kms_key_name
    }
  }

  dynamic "lifecycle_rule" {
    for_each = var.lifecycle_rules
    content {
      action {
        type          = lifecycle_rule.value.action.type
        storage_class = lookup(lifecycle_rule.value.action, "storage_class", null)
      }
      condition {
        age                   = lookup(lifecycle_rule.value.condition, "age", null)
        created_before        = lookup(lifecycle_rule.value.condition, "created_before", null)
        with_state            = lookup(lifecycle_rule.value.condition, "with_state", null)
        matches_storage_class = lookup(lifecycle_rule.value.condition, "matches_storage_class", null)
        num_newer_versions    = lookup(lifecycle_rule.value.condition, "num_newer_versions", null)
      }
    }
  }
}
```

vim variables.tf
```
variable "name" {
  description = "The name of the bucket."
  type        = string
}

variable "project_id" {
  description = "The ID of the project to create the bucket in."
  type        = string
}

variable "location" {
  description = "The location of the bucket."
  type        = string
}

variable "storage_class" {
  description = "The Storage Class of the new bucket."
  type        = string
  default     = null
}

variable "labels" {
  description = "A set of key/value label pairs to assign to the bucket."
  type        = map(string)
  default     = null
}


variable "bucket_policy_only" {
  description = "Enables Bucket Policy Only access to a bucket."
  type        = bool
  default     = true
}

variable "versioning" {
  description = "While set to true, versioning is fully enabled for this bucket."
  type        = bool
  default     = true
}

variable "force_destroy" {
  description = "When deleting a bucket, this boolean option will delete all contained objects. If false, Terraform will fail to delete buckets which contain objects."
  type        = bool
  default     = false
}

variable "iam_members" {
  description = "The list of IAM members to grant permissions on the bucket."
  type = list(object({
    role   = string
    member = string
  }))
  default = []
}

variable "retention_policy" {
  description = "Configuration of the bucket's data retention policy for how long objects in the bucket should be retained."
  type = object({
    is_locked        = bool
    retention_period = number
  })
  default = null
}

variable "encryption" {
  description = "A Cloud KMS key that will be used to encrypt objects inserted into this bucket"
  type = object({
    default_kms_key_name = string
  })
  default = null
}

variable "lifecycle_rules" {
  description = "The bucket's Lifecycle Rules configuration."
  type = list(object({
    # Object with keys:
    # - type - The type of the action of this Lifecycle Rule. Supported values: Delete and SetStorageClass.
    # - storage_class - (Required if action type is SetStorageClass) The target Storage Class of objects affected by this Lifecycle Rule.
    action = any

    # Object with keys:
    # - age - (Optional) Minimum age of an object in days to satisfy this condition.
    # - created_before - (Optional) Creation date of an object in RFC 3339 (e.g. 2017-06-13) to satisfy this condition.
    # - with_state - (Optional) Match to live and/or archived objects. Supported values include: "LIVE", "ARCHIVED", "ANY".
    # - matches_storage_class - (Optional) Storage Class of objects to satisfy this condition. Supported values include: MULTI_REGIONAL, REGIONAL, NEARLINE, COLDLINE, STANDARD, DURABLE_REDUCED_AVAILABILITY.
    # - num_newer_versions - (Optional) Relevant only for versioned objects. The number of newer versions of an object to satisfy this condition.
    condition = any
  }))
  default = []
}
```

vim outputs.tf
```
output "bucket" {
  description = "The created storage bucket"
  value       = google_storage_bucket.bucket
}
```

###Return to the main.tf in your root module and add a reference to the new module:
```
module "gcs-static-website-bucket" {
  source = "./modules/gcs-static-website-bucket"

  name       = var.name
  project_id = var.project_id
  location   = "us-east1"

  lifecycle_rules = [{
    action = {
      type = "Delete"
    }
    condition = {
      age        = 365
      with_state = "ANY"
    }
  }]
}
```

touch outputs.tf
```
output "bucket-name" {
  description = "Bucket names."
  value       = "module.gcs-static-website-bucket.bucket"
}
```

touch variables.tf
```
variable "project_id" {
  description = "The ID of the project in which to provision resources."
  type        = string
  default     = "#FILL IN YOUR PROJECT ID HERE"

}

variable "name" {
  description = "Name of the buckets to create."
  type        = string
  default     = "#FILL IN YOUR (UNIQUE) BUCKET NAME HERE"
}
```

terraform init

terraform apply

cd ~
curl https://raw.githubusercontent.com/hashicorp/learn-terraform-modules/master/modules/aws-s3-static-website-bucket/www/index.html > index.html
curl https://raw.githubusercontent.com/hashicorp/learn-terraform-modules/blob/master/modules/aws-s3-static-website-bucket/www/error.html > error.html

gsutil mb gs://YOUR-BUCKET-NAME
gsutil cp *.html gs://YOUR-BUCKET-NAME

gsutil rm -r gs://YOUR-BUCKET-NAME
terraform destroy

## Managing Terraform State
wget https://releases.hashicorp.com/terraform/0.14.8/terraform_0.14.8_linux_386.zip
unzip terraform_0.14.8_linux_386.zip
sudo mv ~/terraform /usr/local/bin/
terraform -v
touch main.tf
gcloud config list --format 'value(core.project)'
```
provider "google" {
  project     = "# REPLACE WITH YOUR PROJECT ID"
  region      = "us-central-1"
}

resource "google_storage_bucket" "test-bucket-for-state" {
  name        = "# REPLACE WITH YOUR PROJECT ID"
  location    = "US"
  uniform_bucket_level_access = true
}
```
```
terraform {
  backend "local" {
    path = "terraform/state/terraform.tfstate"
  }
  required_version = "~> 0.14"
}
```

terraform init
terraform apply
terraform show
```
terraform {
  backend "gcs" {
    bucket  = "# REPLACE WITH YOUR BUCKET NAME"
    prefix  = "terraform/state"
  }
}
```
terraform init
terraform refresh 
terraform show

## Automating Infrastructure on Google Cloud with Terraform: Challenge Lab
ref: https://github.com/apichlinski/GCPTerraformChallenge-GSP345.git

