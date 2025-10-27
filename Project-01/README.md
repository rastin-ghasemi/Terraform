# Terraform Project: Simple Web App Deployment

In this project, we are going to set up a simple website that displays the current time using **Terraform**.

---

## Step 1: Set Up `terraform.tf`

```hcl
terraform {
  required_version = ">= 1.0.0"

  required_providers {
    aws = {
      source = "hashicorp/aws"
    }
    random = {
      source  = "hashicorp/random"
      version = "3.1.0"
    }
    local = {
      source  = "hashicorp/local"
      version = "2.1.0"
    }
    tls = {
      source  = "hashicorp/tls"
      version = "3.1.0"
    }
  }
}
```

This configuration defines all the main providers we need for our infrastructure.

To list all required providers, run:
```bash
terraform providers
```

### Initialize Terraform
```bash
terraform init
```

### Verify Installed Providers
```bash
terraform version
```

---

## Step 2: Configure the AWS Provider

```hcl
provider "aws" {
  region = "us-east-1"

  default_tags {
    tags = {
      Environment = "Test"
      Owner       = "TFProviders"
      Project     = "Test"
    }
  }
}
```

---

## Step 3: Use Data Blocks

```hcl
data "aws_availability_zones" "available" {}
data "aws_region" "current" {}
```

---

## Step 4: Create a VPC

```hcl
resource "aws_vpc" "vpc" {
  cidr_block = var.vpc_cidr

  tags = {
    Name        = var.vpc_name
    Environment = "demo_environment"
    region      = data.aws_region.current.name
    Terraform   = "true"
  }
}
```

### Variables (`variables.tf`)
```hcl
variable "vpc_cidr" {
  description = "CIDR for the VPC"
  type        = string
  default     = "10.0.0.0/16"
  validation {
    condition     = can(cidrhost(var.vpc_cidr, 0))
    error_message = "Must be a valid CIDR block."
  }
}

variable "vpc_name" {
  description = "Name of the VPC"
  type        = string
  default     = "First VPC"
}
```

---

## Step 5: Create Subnets (Private and Public)

```hcl
# Private subnets
resource "aws_subnet" "private_subnets" {
  for_each          = var.private_subnets
  vpc_id            = aws_vpc.vpc.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, each.value)
  availability_zone = tolist(data.aws_availability_zones.available.names)[each.value]

  tags = {
    Name      = each.key
    Terraform = "true"
  }
}

# Public subnets
resource "aws_subnet" "public_subnets" {
  for_each                = var.public_subnets
  vpc_id                  = aws_vpc.vpc.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 8, each.value + 100)
  availability_zone       = tolist(data.aws_availability_zones.available.names)[each.value]
  map_public_ip_on_launch = true

  tags = {
    Name      = each.key
    Terraform = "true"
  }
}
```

---

## Step 6: Create an Internet Gateway

```hcl
resource "aws_internet_gateway" "internet_gateway" {
  vpc_id = aws_vpc.vpc.id

  tags = {
    Name = "demo_igw"
  }
}
```

---

## Step 7: Create Route Tables and Associations

```hcl
resource "aws_route_table" "public_route_table" {
  vpc_id = aws_vpc.vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.internet_gateway.id
  }

  tags = {
    Name      = "demo_public_rtb"
    Terraform = "true"
  }
}

resource "aws_route_table_association" "public" {
  depends_on     = [aws_subnet.public_subnets]
  route_table_id = aws_route_table.public_route_table.id
  for_each       = aws_subnet.public_subnets
  subnet_id      = each.value.id
}
```

---

## Step 8: Create an SSH Key Pair

```hcl
resource "tls_private_key" "generated" {
  algorithm = "RSA"
}

resource "local_file" "private_key_pem" {
  content  = tls_private_key.generated.private_key_pem
  filename = "MyAWSKey.pem"
}

resource "aws_key_pair" "generated" {
  key_name   = "MyAWSKey"
  public_key = tls_private_key.generated.public_key_openssh

  lifecycle {
    ignore_changes = [key_name]
  }
}
```

---

## Step 9: Create Security Groups

### SSH Access
```hcl
resource "aws_security_group" "ingress-ssh" {
  name   = "allow-all-ssh"
  vpc_id = aws_vpc.vpc.id

  ingress {
    cidr_blocks = ["0.0.0.0/0"]
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

### Web Access
```hcl
resource "aws_security_group" "vpc-web" {
  name        = "vpc-web-${terraform.workspace}"
  vpc_id      = aws_vpc.vpc.id
  description = "Web Traffic"

  ingress {
    description = "Allow HTTP"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "Allow HTTPS"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

### Ping Access
```hcl
resource "aws_security_group" "vpc-ping" {
  name        = "vpc-ping"
  vpc_id      = aws_vpc.vpc.id
  description = "Allow ICMP Ping"

  ingress {
    from_port   = -1
    to_port     = -1
    protocol    = "icmp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

---

## Step 10: Launch an EC2 Instance

```hcl
resource "aws_instance" "ubuntu_server" {
  ami                         = data.aws_ami.ubuntu.id
  instance_type               = "t3.micro"
  subnet_id                   = aws_subnet.public_subnets["public_subnet_1"].id
  security_groups             = [aws_security_group.vpc-ping.id, aws_security_group.ingress-ssh.id, aws_security_group.vpc-web.id]
  associate_public_ip_address = true
  key_name                    = aws_key_pair.generated.key_name

  tags = {
    Name = "Ubuntu EC2 Server"
  }

  lifecycle {
    ignore_changes = [security_groups]
  }

  provisioner "remote-exec" {
    inline = [
      "sudo rm -rf /tmp",
      "sudo git clone https://github.com/hashicorp/demo-terraform-101 /tmp",
      "sudo sh /tmp/assets/setup-web.sh",
    ]

    connection {
      user        = "ubuntu"
      private_key = tls_private_key.generated.private_key_pem
      host        = self.public_ip
    }
  }

  provisioner "local-exec" {
    command = "chmod 600 ${local_file.private_key_pem.filename}"
  }
}
```

---

## Step 11: View EC2 Details

```bash
terraform state show aws_instance.ubuntu_server
```

---

## Step 12: Access the Web App

Visit:
```
http://<your-public-ip>/
```

SSH into the instance:
```bash
ssh ubuntu@<your-public-ip> -i ./MyAWSKey.pem
```

---

## Step 13: Cleanup

When finished, destroy all resources to avoid costs:
```bash
terraform destroy
```

---

✅ **First project completed successfully!**
Next, we will create a **VPC using the AWS module** and deploy an **Auto Scaling Group (ASG)** for our EC2 instances.

**See you in the next lab! 👋**