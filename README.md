## Project one
This Terraform project automates the deployment of a simple web application on AWS, displaying the current time. It builds a foundational network with a VPC, subnets, and gateways, then launches an EC2 instance secured with specific firewall rules. The instance is automatically provisioned with the web server software upon creation via a remote execution script.

In the future, this project will be enhanced to replace the single EC2 instance with a highly available and scalable Auto Scaling Group (ASG) fronted by an Application Load Balancer (ALB). This will ensure the web application is more resilient and can handle increases in traffic. The infrastructure will also be refactored using a dedicated AWS VPC module for better reusability and maintainability.

## Project 02

In Project 2, we enhanced Project One by implementing robust Terraform state management through the configuration and migration of two backend solutions. We first established an S3 backend with versioning, encryption, and DynamoDB state locking for secure local state storage. Subsequently, we migrated to Terraform Cloud to enable enhanced remote operations and team collaboration. Throughout this process, we practiced state migration techniques between backends while maintaining complete infrastructure integrity.
