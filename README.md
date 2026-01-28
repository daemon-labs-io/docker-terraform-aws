# Learn to provision AWS resources with Terraform and Docker

Learn how to architect and deploy AWS resources locally using Terraform, Docker, and LocalStack. 
This hands-on workshop covers the end-to-end lifecycle of Infrastructure as Code (IaC) without the need for an AWS account or cloud costs.

---

## 🛑 Prerequisites

Before beginning this workshop, please ensure your environment is correctly set up by following the instructions in our prerequisites documentation:

➡️ **[Prerequisites guide](https://github.com/daemon-labs-resources/prerequisites)**

### ⚠️ In-person workshop prerequisites

<details>
<summary>If you are in an in-person workshop, expand this section.</summary>

> [!CAUTION]  
> This only works when attending a workshop in person.  
> Due to having a number of people trying to retrieve Docker images at the same time, this allows for a more efficient way.
>
> If you are **NOT** in an in-person workshop, continue to the next step, Docker images will be pulled as needed.

Once the facilitator has given you an IP address, open `http://<IP-ADDRESS>:8000` in your browser.

When you see the file listing, download the `workshop-images.tar` file.

> [!WARNING]
> Your browser may block the download initially, when prompted, allow it to download.

Run the following command:

```shell
docker load -i ~/Downloads/workshop-images.tar
```

<!-- ### Validate Docker images -->

Run the following command:

```shell
docker images
```

> [!NOTE]
> You should now see three images listed.
>
> ```shell
> $ docker images
> IMAGE                      ID             DISK USAGE   CONTENT SIZE   EXTRA
> amazon/aws-cli:latest      71a8f2297aa9        463MB             0B    U   
> hashicorp/terraform:1.14   fad9352aeadb        123MB             0B    U   
> localstack/localstack:4    badd0f2d7269       1.21GB             0B    U   
> ```
>
> _This output is using Rancher Desktop, Docker Desktop and Docker Engine may differ slightly._  
> _Some values may vary._

</details>

### Create project folder

Create a new folder for your project:

```shell
mkdir -p ~/Documents/daemon-labs/docker-terraform-aws
```

> [!NOTE]
> You can either create this via a terminal window or your file explorer.

### Open the new folder in your code editor

> [!TIP]
> If you are using VSCode, we can now do everything from within the code editor.  
> You can open the terminal pane via Terminal -> New Terminal.

---

## 1. Setup the "fake cloud"

**Goal:** Establish the project foundation.

### Create an environment variables file

Create a `.env` file in the **root** of your project and add the following:

```dotenv
AWS_ACCESS_KEY_ID=test
AWS_SECRET_ACCESS_KEY=test
AWS_DEFAULT_REGION=us-east-1
AWS_ENDPOINT_URL=http://localstack:4566
```

### Create the Docker Compose file

Create a `docker-compose.yaml` file in the **root** of your project and add the following:

```yaml
services:
  localstack:
    image: localstack/localstack:4
    networks:
      default:
        aliases:
          - localstack
          - 000000000000.localstack
          - terraform.localstack
  aws:
    image: amazon/aws-cli
    depends_on:
      localstack:
        condition: service_healthy
    env_file:
      - ./.env
```

> [!NOTE]
> The LocalStack image comes with a predefined healthcheck, by adding the `depends_on` to our AWS CLI container, 
> it means it won't do anything until LocalStack is ready.

---

## 2. Identity verification

**Goal:** Confirm connectivity and explore the simulated AWS account.

Run the following command:

```shell
docker compose run --rm aws --version
```

> [!NOTE]
> You should see a response that starts with `aws-cli/2` which is followed by `Python/3` and lastly some `Linux` distribution details.
> 
> As this is the first time we have used our stack and we have the dependency on LocalStack, it will take slightly longer.  
> After the first time, if we run this again it will be much quicker.

Run the following command:

```shell
docker compose run --rm aws sts get-caller-identity
```

You should see the following response:

```json
{
    "UserId": "AKIAIOSFODNN7EXAMPLE",
    "Account": "000000000000",
    "Arn": "arn:aws:iam::000000000000:root"
}
```

> [!NOTE]
> The JSON response shows "fake" user details which resemble those you would get on an actual AWS account.

---

## 3. Initialising Terraform

**Goal:** Initialise the IaC engine.

### Create the Terraform subdirectory

We keep our Terraform config separate from application code.

```shell
mkdir ./terraform
```

### Add Terraform to Docker Compose

Open the `docker-compose.yaml` file and under `services` append the following:

```yaml
  terraform:
    image: hashicorp/terraform:1.14
    depends_on:
      localstack:
        condition: service_healthy
    env_file:
      - ./.env
    working_dir: /terraform
    volumes:
      - ./terraform:/terraform
```

> [!NOTE]
> There are scenarios where Terraform needs more than just Terraform, for example: Python.  
> In these scenarios we would add a `build` attribute and create a `Dockerfile` to extend the base Terraform image.

### Image check

Run the following command:

```shell
docker compose run --rm terraform version
```

> [!NOTE]
> The output should start with `Terraform v1.14` followed by the latest patch version.

> [!TIP]
> The `--rm` flag tells Docker to remove the container once it has finished/completed.  
> If you run `docker compose ps -a` you'll currently see nothing is listed, if you were to not use `--rm` you would see one container listed.

Run the following command:

```shell
docker compose run --rm terraform init
```

You'll see the following response:

```text
Terraform initialized in an empty directory!

The directory has no Terraform configuration files. You may begin working
with Terraform immediately by creating Terraform configuration files.
```

> [!NOTE]
> Terraform actually defines a [Style Guide](https://developer.hashicorp.com/terraform/language/style) around a lot of different areas.  
> Where possible, always try to follow it.

### Define the required versions and provider

Create a `terraform/terraform.tf` file and add the following:

```hcl
terraform {
  required_version = "~> 1.14"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}
```

Create a `terraform/providers.tf` file and add the following:

```hcl
provider "aws" {
  default_tags {
    tags = {
      provisioner = "Terraform"
    }
  }
}
```

> [!TIP]
> By definind the `default_tags` you will see that Terraform will automatically add these tags to every applicable resource.

Run the following command:

```shell
docker compose run --rm terraform init
```

You'll see the following response:

```text
Initializing the backend...
Initializing provider plugins...
- Finding hashicorp/aws versions matching "~> 6.0"...
- Installing hashicorp/aws v6.28.0...
- Installed hashicorp/aws v6.28.0 (signed by HashiCorp)
Terraform has created a lock file .terraform.lock.hcl to record the provider
selections it made above. Include this file in your version control repository
so that Terraform can guarantee to make the same selections by default when
you run "terraform init" in the future.

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
```

> [!NOTE]
> Notice how the `terraform/.terraform.lock.hcl` file is automatically created on your host machine due to the volume mount.  
> Also, the version `hashicorp/aws` might vary from above, but it will always start with `v6`.

Lastly, run the following command:

```shell
docker compose run --rm terraform plan
```

You should see the following output:

```text
No changes. Your infrastructure matches the configuration.

Terraform has compared your real infrastructure against your configuration and found no differences, so no changes are needed.
```

---

## 4. Provision resources

**Goal:** Deploy your first resource and solve environment quirks.

### Resolve a LocalStack quirk

Open the `docker-compose.yaml` file in the root of the project and add this `environment` section to the Terraform container:

```yaml
environment:
  TF_VAR_s3_use_path_style: true
```

Create a `terraform/variables.tf` file and add the following:

```hcl
variable "s3_use_path_style" {
  description = "Set to true for LocalStack"
  type        = bool
  default     = false
}
```

Open the `terraform/providers.tf` file and update the provider to the following:

```hcl
provider "aws" {
  s3_use_path_style = var.s3_use_path_style

  default_tags {
    tags = {
      provisioner = "Terraform"
    }
  }
}
```

### Create an S3 bucket

Create a `terraform/main.tf` file and add the following:

```hcl
resource "aws_s3_bucket" "workshop_bucket" {
  bucket = "daemon-labs-workshop-bucket"
}

resource "aws_s3_bucket_public_access_block" "workshop_bucket" {
  bucket = aws_s3_bucket.workshop_bucket.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

Create a `terraform/outputs.tf` file and add the following:

```hcl
output "workshop_bucket_arn" {
  value = aws_s3_bucket.workshop_bucket.arn
}
```

Run the following command:

```shell
docker compose run --rm terraform apply
```

> [!NOTE]
> Terraform will print out the `plan` and prompt you to accept the changes.

Type `yes` and pressing enter and you'll then see the following output:

```text
aws_s3_bucket.workshop_bucket: Creating...
aws_s3_bucket.workshop_bucket: Creation complete after 0s [id=daemon-labs-workshop-bucket]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.

Outputs:

workshop_bucket_arn = "arn:aws:s3:::daemon-labs-workshop-bucket"
```

> [!NOTE]
> Notice how Terraform has printed out an `Outputs` section with our bucket ARN, 
> it's worth noting that Terraform is clever enough to not print out sensitive data here.  
> If we wanted to "play safe" could run `plan` first, but for our purposes, 
> because there is the prompt we can go straight to this.

### Verify the bucket exists

Run the following command:

```shell
docker compose run --rm aws s3 ls
```

You'll see something similar to the following response:

```text
2026-01-28 20:26:31 daemon-labs-workshop-bucket
```

---

## 5. Scaling with modules

**Goal:** Upgrade to professional, community-maintained code.

### Add and load the S3 module

Open the `terraform/main.tf` file and append the following:

```hcl
module "s3_bucket" {
  source  = "terraform-aws-modules/s3-bucket/aws"
  version = "~> 5.0"

  bucket = "daemon-labs-workshop-module-bucket"
}
```

> [!NOTE]
> Modules work in a similar way to providers and need downloading before they can be used.

Run the following command:

```shell
docker compose run --rm terraform init
```

You'll see something similar to the following response:

```text
Initializing the backend...
Initializing modules...
Downloading registry.terraform.io/terraform-aws-modules/s3-bucket/aws 5.10.0 for s3_bucket...
- s3_bucket in .terraform/modules/s3_bucket
Initializing provider plugins...
- Reusing previous version of hashicorp/aws from the dependency lock file
- Using previously-installed hashicorp/aws v6.28.0

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
```

### Applying to module

Open `terraform/outputs.tf` and append the following:

```hcl
output "workshop_module_bucket_arn" {
  value = module.s3_bucket.s3_bucket_arn
}
```

Run the following command:

```shell
docker compose run --rm terraform apply -auto-approve
```

> [!WARNING]
> We're living dangerously here by using the `-auto-approve` flag,
> this is not recommended on a day-to-day or real life scenario.

You'll then see the following output:

```text
module.s3_bucket.aws_s3_bucket.this[0]: Creating...
module.s3_bucket.aws_s3_bucket.this[0]: Creation complete after 0s [id=daemon-labs-workshop-module-bucket]
module.s3_bucket.aws_s3_bucket_public_access_block.this[0]: Creating...
module.s3_bucket.aws_s3_bucket_public_access_block.this[0]: Creation complete after 0s [id=daemon-labs-workshop-module-bucket]

Apply complete! Resources: 2 added, 0 changed, 0 destroyed.

Outputs:

workshop_bucket_arn = "arn:aws:s3:::daemon-labs-workshop-bucket"
workshop_module_bucket_arn = "arn:aws:s3:::daemon-labs-workshop-module-bucket"
```

---

## 6. Cleanup

**Goal:** Destroy resources, remove containers and reclaim disk space.

Since we are done with the workshop, let's remove the resources we created.

Run the following command:

```
docker compose run --rm terraform destroy
```

> [!NOTE]
> By doing this first, we make sure our state file has been updated accordingly.

Run the following command:

```shell
docker compose ps -a
```

> [!NOTE]
> Even though they're not all running, we still have this images sitting there doing nothing.

Run the following command:

```shell
docker compose images
```

> [!NOTE]
> We also have these images which are taking up resources on our machine.

Run the following command:

```shell
docker compose down --rmi all
```

> [!NOTE]
> This stops all services, removes the containers/networks, and deletes all images used by this project.

---

## 🎉 Congratulations

You've successfully gone from an empty folder to a fully orchestrated local cloud!
By completing this workshop, you have mastered:

- ☁️ **Local Cloud Simulation:** Using LocalStack to emulate AWS without the costs, risks, or credentials.
- 🏗️ **Infrastructure as Code:** Managing the full lifecycle of resources using Terraform init, plan, and apply.
- 🐳 **Environment Orchestration:** Using Docker Compose to bridge the gap between your local code and the simulated cloud.
- 📦 **Modular Architecture:** Leveraging Community Modules to deploy complex, best-practice infrastructure in seconds.
