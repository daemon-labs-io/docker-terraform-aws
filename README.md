# Learn to provision AWS resources with Terraform and Docker

Workshop description.

---

## 🛑 Prerequisites

Before beginning this workshop, please ensure your environment is correctly set up by following the instructions in our prerequisites documentation:

➡️ **[Prerequisites guide](https://github.com/daemon-labs-resources/prerequisites)**

<details>
<summary>⚠️ In-person workshop prerequisites</summary>

<!-- ### Load Docker images -->

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

## 1. Setup the "Fake Cloud"

**Goal:** Spin up a fully configured local AWS environment.

### Create an environment variables file

Create a `.env` file in the **root** of your project.

```dotenv
AWS_ACCESS_KEY_ID=test
AWS_SECRET_ACCESS_KEY=test
AWS_DEFAULT_REGION=us-east-1
AWS_ENDPOINT_URL=http://localstack:4566
```

### Create `docker-compose.yaml`

Create this file in the **root** of your project.

```yaml
services:
  localstack:
    image: localstack/localstack:4
    # @todo do we need the ports?
    # ports:
    #   - 4566:4566
    #   - 4510-4559:4510-4559
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

### Confirm the AWS CLI is working as expected

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
> The JSON response shows the "fake" user details you injected via environment variables.

---

## 2. The Terraform foundation

**Goal:** Get a working container environment running.

### Create the Terraform subdirectory

We keep our Terraform config separate from application code.

```shell
mkdir ./terraform
```

### Add Terraform to `docker-compose.yaml`

Open the `docker-compose.yaml` and append the following:

```yaml
  terraform:
    image: hashicorp/terraform:1.14
    depends_on:
      localstack:
        condition: service_healthy
    env_file:
      - ./.env
  #   environment:
  #     TF_VAR_s3_use_path_style: true
    working_dir: /terraform
    volumes:
      - ./terraform:/terraform
```

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

---

## 3. Initialising Terraform

**Goal:** Getting the basics in place.

> [!NOTE]
> Terraform actually defines a [Style Guide](https://developer.hashicorp.com/terraform/language/style) around a lot of different areas.  
> Where possible, always try to follow it.

### Define the required versions

Create a `terraform.tf` file and add the following:

```hcl
terraform {
  required_version = "~> 1.14"
}
```

Run the following command:

```shell
docker compose run --rm terraform init
```

You'll be comfronted with the following response:

```text
Terraform initialized in an empty directory!

The directory has no Terraform configuration files. You may begin working
with Terraform immediately by creating Terraform configuration files.
```

Open the `docker-compose.yaml` file and update to the following:

```yaml
  terraform:
    image: hashicorp/terraform:1.14
    depends_on:
      localstack:
        condition: service_healthy
    env_file:
      - ./.env
  #   environment:
  #     TF_VAR_s3_use_path_style: true
    working_dir: /terraform
    volumes:
      - ./terraform:/terraform
```

Run the following command again:

```shell
docker compose run --rm terraform init
```

Now you'll see the following response:

```text
Initializing the backend...
Initializing provider plugins...

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
```

Open the `terraform.tf` file and update to the following:

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

Run the following command again:

```shell
docker compose run --rm terraform init
```

Now you'll see the following response:

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
<!-- 
## 4. Provisioning resources

**Goal**: TBC

Create `main.tf` and add the following:

```hcl
resource "aws_s3_bucket" "this" {
  bucket = "daemon-labs-bucket-example"
}
```

Run the following command:

```shell
docker compose run --rm terraform plan
```

> [!NOTE]
> At this stage you should now see Terraform wanting to add one resource.

Run the following command, when prompted type `yes` and press enter:

```shell
docker compose run --rm terraform apply
```

> [!NOTE]
> We've now hit another error, we need to configure the provider.

Create `variables.tf` and add the following:

```hcl
variable "s3_use_path_style" {
  description = "Set to true for LocalStack"
  type        = bool
  default     = false
}
```

Create `providers.tf` and add the following:

```hcl
provider "aws" {
  s3_use_path_style = var.s3_use_path_style
}
```

Open `docker-compose.yaml` and add a new environment variable for the Terraform container:

```yaml

TF_VAR_s3_use_path_style: true
```

Run the following command, when prompted type `yes` and press enter:

```shell
docker compose run --rm terraform apply
```

You should now see the following output:

```text
aws_s3_bucket.this: Creating...
aws_s3_bucket.this: Creation complete after 0s [id=daemon-labs-bucket-example]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

---

## 5. Introducing modules

Open the `main.tf` file and append the following:

```hcl
module "s3_bucket" {
  source = "terraform-aws-modules/s3-bucket/aws"
  version = "~> 5.0"

  bucket = "daemon-labs-bucket-module-example"
}
```

Run the following command:

```shell
docker compose run --rm terraform init
```

You should see the following output:

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

Run the following command:

```shell
docker compose run --rm terraform plan
```

> [!NOTE]
> Notice how despite "just" creating a bucket, the module wants to add two resources instead of one.

Run the following command, when prompted type `yes` and press enter:

```shell
docker compose run --rm terraform apply
```

You should now see the following output:

```text
module.s3_bucket.aws_s3_bucket.this[0]: Creating...
module.s3_bucket.aws_s3_bucket.this[0]: Creation complete after 0s [id=daemon-labs-bucket-module-example]
module.s3_bucket.aws_s3_bucket_public_access_block.this[0]: Creating...
module.s3_bucket.aws_s3_bucket_public_access_block.this[0]: Creation complete after 0s [id=daemon-labs-bucket-module-example]

Apply complete! Resources: 2 added, 0 changed, 0 destroyed.
```

---

## 6. Cleanup

Goal: Destroy resources, remove containers and reclaim disk space.

Since we are done with the workshop, let's remove the resources we created.

Runn the following command:

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

### 🎉 Congratulations

You've just successfully:
- Containerised your entire Cloud Engineering toolset (Terraform, AWS CLI).
- Simulated a complete AWS environment locally using LocalStack.
- Debugged complex infrastructure networking issues (DNS, Service Discovery).
- Refactored raw infrastructure code into a reusable Module.
- You have now provisioned AWS resources without needing an AWS account, internet access, or a credit card! -->
