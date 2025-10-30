## Project one
This Terraform project automates the deployment of a simple web application on AWS, displaying the current time. It builds a foundational network with a VPC, subnets, and gateways, then launches an EC2 instance secured with specific firewall rules. The instance is automatically provisioned with the web server software upon creation via a remote execution script.

In the future, this project will be enhanced to replace the single EC2 instance with a highly available and scalable Auto Scaling Group (ASG) fronted by an Application Load Balancer (ALB). This will ensure the web application is more resilient and can handle increases in traffic. The infrastructure will also be refactored using a dedicated AWS VPC module for better reusability and maintainability.
