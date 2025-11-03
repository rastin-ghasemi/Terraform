# 🧩 Project 2 — Terraform Remote and Standard Backend Configuration

In this project, we enhance our previous infrastructure by implementing **two key backend configurations** in Terraform:

1. **S3 Standard Backend**
2. **Terraform Cloud (Remote) Backend**

These configurations help securely store and share the Terraform state file, improve team collaboration, and enable state locking to prevent conflicts.

---

## 🚀 Phase 1: Configure the Backend

This phase focuses on mastering Terraform state management by configuring both backend types and performing a **state migration**.

At the end of this phase, we will practice how to migrate state files between different backends.  
> ⚠️ Terraform supports only **one active backend** at a time.

---

## ☁️ S3 Standard Backend

We start by creating an **S3 bucket** and a **DynamoDB table** for state locking.

### Step 1: Create the S3 Bucket
```bash
aws s3api create-bucket   --bucket my-terraform-state-bucket-$(date +%s)   --region us-east-1
```

### Step 2: Enable Versioning and Encryption
```bash
aws s3api put-bucket-versioning   --bucket my-terraform-state-bucket-123456789   --versioning-configuration Status=Enabled

aws s3api put-bucket-encryption   --bucket my-terraform-state-bucket-1761834673   --server-side-encryption-configuration '{
      "Rules": [
          {
              "ApplyServerSideEncryptionByDefault": {
                  "SSEAlgorithm": "AES256"
              }
          }
      ]
  }'
```
> Optionally, you can enable KMS encryption for stronger security.

### Step 3: Create DynamoDB Table for State Locking
```bash
aws dynamodb create-table   --table-name terraform-state-locking   --attribute-definitions AttributeName=LockID,AttributeType=S   --key-schema AttributeName=LockID,KeyType=HASH   --provisioned-throughput ReadCapacityUnits=5,WriteCapacityUnits=5   --table-class STANDARD

aws dynamodb wait table-exists --table-name terraform-state-locking
```

### Step 4: Configure Bucket Policy (Optional but Recommended)
```bash
cat > bucket-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EnforceTLS",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::my-terraform-state-bucket-123456789",
        "arn:aws:s3:::my-terraform-state-bucket-123456789/*"
      ],
      "Condition": {
        "Bool": {"aws:SecureTransport": "false"}
      }
    },
    {
      "Sid": "RequireEncryption",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::my-terraform-state-bucket-123456789/*",
      "Condition": {
        "StringNotEquals": {"s3:x-amz-server-side-encryption": "AES256"}
      }
    }
  ]
}
EOF

aws s3api put-bucket-policy   --bucket my-terraform-state-bucket-123456789   --policy file://bucket-policy.json
```

### Step 5: Update `terraform.tf`
```hcl
terraform {
  required_version = ">= 1.0.0"
  backend "s3" {
    bucket         = "my-terraform-state-bucket-1761834673"
    key            = "test/aws_infra"
    region         = "us-east-1"
    dynamodb_table = "terraform-state-locking"
    encrypt        = true
  }

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

### Step 6: Test the S3 Backend
1. Create some test resources.  
2. Check versioning in the AWS Console.  
3. Verify DynamoDB state locks.  
4. Confirm state file presence in the S3 bucket.

Once confirmed, destroy all resources to prepare for the next step.

---

## ☁️ Terraform Cloud (Remote Backend)

### Step 1: Create a Terraform Cloud Account
Go to [Terraform Cloud](https://app.terraform.io/session)  
Create an **account**, **organization**, and **workspace**.

### Step 2: Login via CLI
```bash
terraform login
```
This command opens your browser for token generation.  
Copy and paste the token in your terminal — now you are logged in.

### Step 3: Update `terraform.tf` for Remote Backend
```hcl
terraform {
  required_version = ">= 1.0.0"
  backend "remote" {
    hostname     = "app.terraform.io"
    organization = "Home-ghasemi"

    workspaces {
      name = "my-aws-app"
    }
  }

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

### Step 4: Reconfigure Terraform
```bash
terraform init -reconfigure
```

If you encounter a credential error such as:
```
Error: No valid credential sources found
```
You must **set your AWS credentials as environment variables** inside **Terraform Cloud**.

Once that’s done, your Terraform project is fully operational with a remote backend!

---

## ✅ Summary

- Implemented **S3 Standard backend** with versioning and encryption.  
- Enabled **DynamoDB state locking** to prevent concurrent changes.  
- Configured **Terraform Cloud backend** for remote collaboration.  
- Practiced **state migration** using `terraform init -reconfigure` and `-migrate-state`.

---

## 🧑‍💻 Author
**Rastin Ghasemi**  
DevOps Engineer | Cloud & Kubernetes Enthusiast  
🔗 [GitHub Profile](https://github.com/rastin-ghasemi)
