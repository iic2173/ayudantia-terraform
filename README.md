# Terraform Tutorial: Infrastructure as Code (IaC) with Terraform Step by step

[All of the official documentation for this tutorial can be found here](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)

Terraform is an open-source tool for automating infrastructure management. It allows you to define your infrastructure as code and manage it across various cloud providers (like AWS, Azure, Google Cloud, etc.) and other services.

---

## Prerequisites

1. **Install Terraform**

   You can install Terraform following the [official website](https://developer.hashicorp.com/terraform/install)

   For Ubuntu, use apt:

   ```bash
   wget -O - https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
   echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
   sudo apt update && sudo apt install terraform
   ```

   For macOS, use Homebrew:

   ```bash
   brew tap hashicorp/tap
   brew install hashicorp/tap/terraform
   ```

2. **Cloud provider account**

   This tutorial will use AWS as an example, but you can apply similar concepts to other cloud providers by adjusting the provider configuration.

3. **Install AWS CLI (if using AWS) and configure it or export variables**

   ```bash
   aws configure
   ```

---

## Step 1: Initializing a Terraform Project

Create a new directory for your project:

```bash
mkdir terraform-tutorial
cd terraform-tutorial
```

Inside the directory, create a new file named `main.tf`. This file will contain the configuration code for Terraform.

```bash
touch main.tf
```

---

## Step 2: Define the Provider

A provider is responsible for managing resources for a specific service, like AWS. In this example, we'll use AWS as the provider.
Here is the [documentation for all available providers](https://registry.terraform.io/browse/providers)

Edit `main.tf` and add the following configuration to define the provider:

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}
```

This tells Terraform to use the AWS provider in the `us-east-1` region.

AWS secrets can also be passed here. But for this example we need to export the secrets in the terminal.

---

## Step 3: Create AWS Resources

Now, let's create an AWS EC2 instance. We'll define an AWS EC2 instance resource using Terraform's `aws_instance` resource.

Add the following code in `main.tf`:

```hcl
resource "aws_instance" "my_instance" {
  ami           = "ami-0e2c8caa4b6378d8c"
  instance_type = "t2.micro"

  tags = {
    Name = "TerraformExample"
  }
}
```


**Explanation:**

- `ami`: Amazon Machine Image ID. You'll need to find a valid AMI ID for your region. The selected AMI is the image for Ubuntu 24.04 LTS in us-east-1, [search for AMIs here](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/finding-an-ami.html)
- `instance_type`: The type of EC2 instance, here we're using `t2.micro`, which is eligible for the AWS Free Tier.
- `tags`: Tags for your instance to give it a name and other identifiers.

More details about "aws_instance" resource in [the documentation](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/instance).

---

## Step 4: Initialize the Terraform Project

Before running Terraform, you need to initialize your project. This step downloads the necessary provider plugins.

```bash
terraform init
```

This command sets up your Terraform working directory by initializing the backend, provider plugins, and other dependencies.

---

## Step 5: Plan the Infrastructure Changes

To see what Terraform will do (i.e., which resources it will create), you need to run the `terraform plan` command:

```bash
terraform plan
```

Terraform will display an execution plan, showing what resources it will create or modify. Review the plan to ensure that Terraform is creating the resources as expected.

---

## Step 6: Apply the Infrastructure Changes

Once you're satisfied with the plan, apply the configuration to create the resources in AWS:

```bash
terraform apply
```

Terraform will prompt you for confirmation before applying the changes. Type `yes` to proceed. Terraform will then create the resources defined in `main.tf`.

---

## Step 7: Verify the Resources

Once Terraform finishes, go to the AWS Management Console and check that the EC2 instance has been created in the specified region.

---

## Step 8: Destroy the Resources

To avoid ongoing costs, you should destroy the resources after you're done. Use the `terraform destroy` command to remove everything that Terraform created:

```bash

terraform destroy
```

Terraform will prompt you for confirmation. Type `yes` to destroy the resources.

---

## Step 9: Save State in a Remote Backend

Terraform uses a state file (`terraform.tfstate`) to manage the state of your infrastructure. It's a good practice to store this state file remotely to enable collaboration among team members, ensure security, and prevent data loss. One common approach is to use AWS S3 for remote storage.

Add the following to your `main.tf` file to configure a remote backend with S3:

```hcl
terraform {
  backend "s3" {
    bucket         = "your-terraform-state-bucket"
    key            = "path/to/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-lock-table"  # Optional, for state locking
  }
}
```

**Explanation:**

- `bucket`: The S3 bucket where the state file will be stored. Must be created manually.
- `key`: The path (key) in the bucket for the state file.
- `region`: The AWS region where the bucket is located.
- `encrypt`: Enables server-side encryption of the state file.
- `dynamodb_table`: (Optional) DynamoDB table for state locking, preventing concurrent modifications. It must have key "LockID". Must be created manually.

Detailed usage [here](https://developer.hashicorp.com/terraform/language/backend/s3)

To apply this configuration, initialize the project again with:

```bash
terraform init
```

This reconfigures the backend and migrates the state file to S3.

---

## Step 10: Use Modules and Variables

Terraform modules allow you to organize and encapsulate your configuration into reusable units. You can also use variables to make your configuration more flexible and configurable.

### Create a Module

1. Create a new folder for the module:

   ```bash
   mkdir -p modules/ec2-instance/
   ```

2. Move move the following EC2 configuration from main.tf to a new file modules/ec2-instance/main.tf

   ```hcl
   resource "aws_instance" "my_instance" {
     ami           = "ami-0e2c8caa4b6378d8c"
     instance_type = "t2.micro"

     tags = {
       Name = "TerraformExample"
     }
   }
   ```

3. Create a `variables.tf` file inside `modules/ec2-instance/` to define variables for your module:

   ```hcl
   variable "ami" {
     description = "AMI ID for the EC2 instance"
   }

   variable "instance_type" {
     description = "EC2 instance type"
     default     = "t2.micro"
   }
   ```

4. Update `modules/ec2-instance/main.tf` to use these variables:
   ```hcl
   resource "aws_instance" "my_instance" {
     ami           = var.ami
     instance_type = var.instance_type

     tags = {
       Name = "TerraformExample"
     }
   }
   ```

### Use the Module `./modules/ec2-instance/main.tf` in the root file

Edit `main.tf` in the root directory to use the created module:

```hcl
module "ec2_instance" {
  source        = "./modules/ec2-instance"
  ami           = "ami-0e2c8caa4b6378d8c"
  instance_type = "t2.micro"
}
```

### Explanation

- `source`: Specifies the path to the module's directory.
- `ami` and `instance_type`: Values passed to the module's variables.

This modular approach makes it easier to reuse configurations and manage complex infrastructure.

---

## Step 11: Add a Security Group and Elastic IP

In this step, we'll enhance our configuration by adding a security group and an Elastic IP (EIP) for the EC2 instance.

### Create a Security Group

Add the following resource to `modules/ec2-instance/main.tf`:

```hcl
resource "aws_security_group" "my_sg" {
  name        = "example_security_group"
  description = "Allow inbound traffic on port 22 (SSH) and 443 (HTTPS)"

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"  # All protocols
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

### Associate the Security Group with the EC2 Instance

Update the `aws_instance` resource to use the security group:

```hcl
resource "aws_instance" "my_instance" {
  ami           = var.ami
  instance_type = var.instance_type
  security_groups = [aws_security_group.my_sg.name]

  tags = {
    Name = "TerraformExample"
  }
}
```

### Create an Elastic IP

Add the following resource to allocate, associate an Elastic IP with your EC2 instance and print it in the console so we know the IP:

```hcl
resource "aws_eip" "my_eip" {
  instance = aws_instance.my_instance.id
}

resource "aws_eip_association" "my_eip_association" {
  instance_id   = aws_instance.my_instance.id
  allocation_id = aws_eip.my_eip.id
}

output "elastic_ip" {
  value = aws_eip.my_eip.public_ip
}
```

This configuration creates an Elastic IP and associates it with the EC2 instance.

---

## Step 12: Configure Server Using Scripts

In this step, we will automate the installation of Nginx on the EC2 instance using a shell script. The script will be executed when the instance starts.

### Create the Shell Script

1. Create a new file called `install_nginx.sh` in the `modules/ec2-instance/` directory:

   ```bash
   touch modules/ec2-instance/install_nginx.sh
   ```

2. Add the following content to `install_nginx.sh`:

   ```bash
   #!/bin/bash
   sudo apt install curl gnupg2 ca-certificates lsb-release ubuntu-keyring

   curl https://nginx.org/keys/nginx_signing.key | gpg --dearmor \
       | sudo tee /usr/share/keyrings/nginx-archive-keyring.gpg >/dev/null

   gpg --dry-run --quiet --no-keyring --import --import-options import-show /usr/share/keyrings/nginx-archive-keyring.gpg

   echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] \
   http://nginx.org/packages/ubuntu `lsb_release -cs` nginx" \
       | sudo tee /etc/apt/sources.list.d/nginx.list

   echo -e "Package: *\nPin: origin nginx.org\nPin: release o=nginx\nPin-Priority: 900\n" \
       | sudo tee /etc/apt/preferences.d/99nginx

   sudo apt update
   sudo apt install nginx

   sudo nginx
   ```

This script will:

- Install necessary dependencies.
- Add the Nginx repository.
- Install Nginx and start the service.

### Use the Script in the EC2 Configuration

Update your `aws_instance` resource to execute the script when the EC2 instance launches. Modify the instance configuration to include `user_data`:

```hcl
resource "aws_instance" "my_instance" {
  ami           = var.ami
  instance_type = var.instance_type
  security_groups = [aws_security_group.example_sg.name]

  user_data = file("${path.module}/install_nginx.sh")

  tags = {
    Name = "TerraformExample"
  }
}
```

The `user_data` directive will run the script automatically when the instance is created.

---

## Step 13: Add SSH Key

Manually create a ssh key, then update the EC2 resource with the name of the key

```hcl
resource "aws_instance" "my_instance" {
  ami           = var.ami
  instance_type = var.instance_type
  security_groups = [aws_security_group.example_sg.name]

  key_name               = "name-of-the-key"
  user_data = file("${path.module}/install_nginx.sh")

  tags = {
    Name = "TerraformExample"
  }
}
```

---

## Best Practices and Next Steps

1. **State Management and remote backends**
   Terraform maintains the state of your infrastructure in a state file (`terraform.tfstate`). You should secure this file or store it in remote backends like AWS S3 to prevent data loss. Remote backends also allow multiple team members to collaborate on the same infrastructure.

2. **Modules**
   As your infrastructure grows, you can should your Terraform code by breaking it into reusable modules. This promotes better organization and scalability.

3. **Variables**
   You can use variables to make your configuration more flexible. For example, you could use variables for the region or instance type, instead of hardcoding them.

---

## Learn more

1. [How to pass variables from files and more](https://developer.hashicorp.com/terraform/language/values/variables)

2. [Manage multiple environments with Workspaces](https://developer.hashicorp.com/terraform/language/state/workspaces)

3. [How to manage secrets correctly](https://spacelift.io/blog/terraform-secrets)

4. [Implement other AWS resources following the official documentation](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
