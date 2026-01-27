# Learn to provision AWS resources with Terraform and Docker

Workshop description.

---

## 🛑 Prerequisites

Before beginning this workshop, please ensure your environment is correctly set up by following the instructions in our prerequisites documentation:

➡️ **[Prerequisites guide](https://github.com/daemon-labs-resources/prerequisites)**

### Load Docker images

> [!CAUTION]
> This only works when attending a workshop in person.  
> Due to having a number of people trying to retrieve Docker images at the same time, this allows for a more efficient way.
>
> If you are **NOT** in an in-person workshop, continue to the [workshop](#1-the-foundation), Docker images will be pulled as needed.

Once the facilitator has given you an IP address, open `http://<IP-ADDRESS>:8000` in your browser.

When you see the file listing, download the `workshop-images.tar` file.

> [!WARNING]
> Your browser may block the download initially, when prompted, allow it to download.

Run the following command:

```shell
docker load -i ~/Downloads/workshop-images.tar
```

### Validate Docker images

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
> _Image IDs, created and sizes may vary._

---

## 1. The foundation

**Goal:** Get a working container environment running.

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

### Create the Terraform subdirectory

We keep our Terraform config separate from application code.

```shell
mkdir ./terraform
```

### Create `docker-compose.yaml`

Create this file in the **root** of your project.

```yaml
services:
  terraform:
    image: hashicorp/terraform:1.14
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

## 2. Initialising Terraform

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
services:
  terraform:
    image: hashicorp/terraform:1.14
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

---

### 🎉 Congratulations

You've just ...
