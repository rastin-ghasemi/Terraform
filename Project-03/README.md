# Project 3 --- Application Load Balancer (Path‑Based Routing) --- Terraform

In this project, we enhanced our previous AWS infrastructure by adding
an **Application Load Balancer (ALB)** that performs **path‑based
routing** to multiple EC2 instances. Each instance serves a different
part of the application.

## 📋 Infrastructure Summary

| Component | Description |
|-----------|-------------|
| **VPC + Subnets** | Base network for the application |
| **Internet Gateway & Routes** | Allow internet access |
| **3 EC2 Instances** | Each serves a separate path |
| **Application Load Balancer** | Handles HTTP requests and routing |
| **3 Target Groups** | Each path = its own target group |
| **Listener Rules** | Forward traffic based on URL paths |

## 🛣️ URL Path Routing

| URL Path | Target Instance | Description |
|----------|-----------------|-------------|
| `/` | **Instance A** | Homepage service |
| `/images/` | **Instance B** | Images service |
| `/register/` | **Instance C** | Registration service |


## Phase one config our Terraform Cloud (Remote Backend):
We have already configured Terraform Cloud in the previous project, so
here is the short version:

 1. **Login to Terraform cloud**

 2. **terraform login** 

 And Done .

 ## Phase 2 Create our Terraform.tf and config it
 ```bash
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
## config our provider
```bash
provider "aws" {
  region = "us-east-1"
  default_tags {
    tags = {
      Environment = "Test"
      Owner       = "Rastin"
      Project     = "Project-01"
    }
  }
}
```
## Create Our VPc
```bash
data "aws_availability_zones" "available" {}
data "aws_region" "current" {}

resource "aws_vpc" "VPC-ALB" {
  cidr_block = var.vpc_cidr

  tags = {
    Name        = var.vpc_name
    Environment = "demo_environment"
    region      = data.aws_region.current.name
    Terraform   = "true"
  }
}
```
## Create VPC
```bash
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
## Create IGW
```bash
resource "aws_internet_gateway" "internet_gateway" {
  vpc_id = aws_vpc.vpc.id

  tags = {
    Name = "demo_igw"
  }
}
```
## Create Route Tables and Associations
```bash
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

## Now Create Security Group for our ALB
```bash
resource "aws_security_group" "ALB" {
  name        = "vpc-web-${terraform.workspace}"
  vpc_id      = aws_vpc.vpc.id
  description = "Web Traffic For ALB"

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

## Now Lets Create Our EC2
We want to serve requests based on what path they are targeted at. As the diagram above shows, incoming requests can be classified based on whether:

1. They are targeted toward the homepage

2. They are registration requests

3. They are related to images.

Each of the types described above needs to be served separately.

Let’s provision three EC2 instances serving the corresponding requests, as seen in the Terraform configuration below. 

We have used the user_data attribute to supply a script that installs and runs the nginx service. Further, each nginx is configured separately to serve separate paths:

Instance A – responds to root path

Instance B – responds to /images path

Instance C – responds to /register path

## Security Group for our ec2

Note: our EC2 has userdata this is new.
```bash
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

## Instance A
```bash
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"] # Canonical's owner ID

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-*-amd64-server-*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}


resource "aws_instance" "Instance_A" {
  ami                         = data.aws_ami.ubuntu.id
  instance_type               = "t3.micro"
  subnet_id                   = aws_subnet.public_subnets["public_subnet_1"].id
  security_groups             = [aws_security_group.vpc-ping.id, aws_security_group.ingress-ssh.id, aws_security_group.vpc-web.id]
  associate_public_ip_address = true
  key_name                    = aws_key_pair.generated.key_name

  tags = {
    Name = "Ubuntu EC2 Server- A"
  }

  lifecycle {
    ignore_changes = [security_groups]
  }

  connection {
    user        = "ubuntu"
    private_key = tls_private_key.generated.private_key_pem
    host        = self.public_ip
  }

  provisioner "local-exec" {
    command = "chmod 600 ${local_file.private_key_pem.filename}"
  }

  user_data = <<-EOF
#!/bin/bash
sudo apt-get update
sudo apt-get install -y nginx
sudo systemctl start nginx
sudo systemctl enable nginx
echo '<!doctype html>
<html lang="en"><h1>Home page!</h1></br>
<h3>(Instance A)</h3>
</html>' | sudo tee /var/www/html/index.html
EOF
}
```

## Instance B
```bash
resource "aws_instance" "Instance_B" {
  ami                         = data.aws_ami.ubuntu.id
  instance_type               = "t3.micro"
  subnet_id                   = aws_subnet.public_subnets["public_subnet_1"].id
  security_groups             = [aws_security_group.vpc-ping.id, aws_security_group.ingress-ssh.id, aws_security_group.vpc-web.id]
  associate_public_ip_address = true
  key_name                    = aws_key_pair.generated.key_name

  tags = {
    Name = "Ubuntu EC2 Server- B"
  }

  lifecycle {
    ignore_changes = [security_groups]
  }

  connection {
    user        = "ubuntu"
    private_key = tls_private_key.generated.private_key_pem
    host        = self.public_ip
  }

  provisioner "local-exec" {
    command = "chmod 600 ${local_file.private_key_pem.filename}"
  }
  user_data = <<-EOF
  #!/bin/bash
             sudo apt-get update
             sudo apt-get install -y nginx
             sudo systemctl start nginx
             sudo systemctl enable nginx
             echo '<!doctype html>
             <html lang="en"><h1>Images!</h1></br>
             <h3>(Instance B)</h3>
             </html>' | sudo tee /var/www/html/index.html
             echo 'server {
                       listen 80 default_server;
                       listen [::]:80 default_server;
                       root /var/www/html;
                       index index.html index.htm index.nginx-debian.html;
                       server_name _;
                       location /images/ {
                           alias /var/www/html/;
                           index index.html;
                       }
                       location / {
                           try_files $uri $uri/ =404;
                       }
                   }' | sudo tee /etc/nginx/sites-available/default
             sudo systemctl reload nginx
             EOF
}
```
## Instance C
```bash
resource "aws_instance" "Instance_C" {
  ami                         = data.aws_ami.ubuntu.id
  instance_type               = "t3.micro"
  subnet_id                   = aws_subnet.public_subnets["public_subnet_1"].id
  security_groups             = [aws_security_group.vpc-ping.id, aws_security_group.ingress-ssh.id, aws_security_group.vpc-web.id]
  associate_public_ip_address = true
  key_name                    = aws_key_pair.generated.key_name

  tags = {
    Name = "Ubuntu EC2 Server- C"
  }

  lifecycle {
    ignore_changes = [security_groups]
  }

  connection {
    user        = "ubuntu"
    private_key = tls_private_key.generated.private_key_pem
    host        = self.public_ip
  }

  provisioner "local-exec" {
    command = "chmod 600 ${local_file.private_key_pem.filename}"
  }
  user_data = <<-EOF
  #!/bin/bash
             sudo apt-get update
             sudo apt-get install -y nginx
             sudo systemctl start nginx
             sudo systemctl enable nginx
             echo '<!doctype html>
             <html lang="en"><h1>Register!</h1></br>
             <h3>(Instance C)</h3>
             </html>' | sudo tee /var/www/html/index.html
             echo 'server {
                       listen 80 default_server;
                       listen [::]:80 default_server;
                       root /var/www/html;
                       index index.html index.htm index.nginx-debian.html;
                       server_name _;
                       location /register/ {
                           alias /var/www/html/;
                           index index.html;
                       }
                       location / {
                           try_files $uri $uri/ =404;
                       }
                   }' | sudo tee /etc/nginx/sites-available/default
             sudo systemctl reload nginx
             EOF
}
```

##  Create an ALB Target Group
Target groups – as the name suggests, are used to group compute resources that serve a single responsibility/purpose. 

In this example, resources that are part of each target group serve requests sent on a specific path. The Target Groups are described below:

1. Target Group A – is a group of instances that serves all the incoming requests targeted towards the home page, as well as all those requests that are not served by other target groups.

2. Target Group B – is a group of instances that serves all the incoming requests made on the /images path.

3. Target Group C – is a group of instances that serves all the incoming requests made on the /register path.

```bash
resource "aws_lb_target_group" "my_tg_a" { // Target Group A
 name     = "target-group-a"
 port     = 80
 protocol = "HTTP"
 vpc_id   = aws_vpc.vpc.id
}

resource "aws_lb_target_group" "my_tg_b" { // Target Group B
 name     = "target-group-b"
 port     = 80
 protocol = "HTTP"
 vpc_id   = aws_vpc.vpc.id
}

resource "aws_lb_target_group" "my_tg_c" { // Target Group C
 name     = "target-group-c"
 port     = 80
 protocol = "HTTP"
 vpc_id   = aws_vpc.vpc.id
}
```
Note that we have not configured the path information for Target Groups in the code above. As far as target groups are concerned, recognizing them with paths is a matter of intent, which listener rules and EC2 instances should fulfill.

## Add the ALB Target Group attachment

Add the configuration below to associate EC2 instances with the Target Groups:

1. Instance A :: Target Group A

2. Instance B :: Target Group B

3. Instance C :: Target Group C

```bash

resource "aws_lb_target_group_attachment" "tg_attachment_a" {
 target_group_arn = aws_lb_target_group.my_tg_a.arn
 target_id        = aws_instance.Instance_A.id
 port             = 80
}

resource "aws_lb_target_group_attachment" "tg_attachment_b" {
 target_group_arn = aws_lb_target_group.my_tg_b.arn
 target_id        = aws_instance.Instance_B.id
 port             = 80
}

resource "aws_lb_target_group_attachment" "tg_attachment_c" {
 target_group_arn = aws_lb_target_group.my_tg_c.arn
 target_id        = aws_instance.Instance_C.id
 port             = 80
}
```
Use terraform apply on the updated configuration, and make sure the instance placement is as intended in the target groups, as seen in the screenshots below.
##  Create an ALB Listener

Now that we have provisioned the EC2 instances and placed them in appropriate target groups, we are ready to provision the ALB and configure the rules.

The resource aws_lb specified in the Terraform configuration below provisions an ALB. The attribute load_balancer_type specifies the type as “application”. We have also specified the security groups and subnets already created with appropriate routing tables. Even though these few config lines are enough to create an ALB, further settings require more resource blocks. 

```bash
// ALB
resource "aws_lb" "my_alb" {
 name               = "my-alb"
 internal           = false
 load_balancer_type = "application"
 security_groups    = [aws_security_group.vpc-web.id]
 subnets            = [for k, subnet in aws_subnet.public_subnets : subnet.id]

 tags = {
   Environment = "dev"
 }
}
```
For example, to create a Listener, we have used the aws_lb_listener resource block. The load_balancer_arn attribute associates this listener to the ALB provisioned because of the previous configuration. The listener also specifies a default action when none of the paths match. As seen below, the default route is configured to a specific target group A:
```bash
// Listener
resource "aws_lb_listener" "my_alb_listener" {
 load_balancer_arn = aws_lb.my_alb.arn
 port              = "80"
 protocol          = "HTTP"

 default_action {
   type             = "forward"
   target_group_arn = aws_lb_target_group.my_tg_a.arn
 }
}
```
## Manage custom ALB Listener rules

The default_action specified in the Listener resource block above is essentially a default Listener Rule. It currently satisfies the default routing condition. 

We also want more rules to route the requests to Target Groups B and C. Add the configuration below to add corresponding listener rules.
```bash
resource "aws_lb_listener_rule" "rule_b" {
 listener_arn = aws_lb_listener.my_alb_listener.arn
 priority     = 60

 action {
   type             = "forward"
   target_group_arn = aws_lb_target_group.my_tg_b.arn
 }

 condition {
   path_pattern {
     values = ["/images*"]
   }
 }
}

resource "aws_lb_listener_rule" "rule_c" {
 listener_arn = aws_lb_listener.my_alb_listener.arn
 priority     = 40

 action {
   type             = "forward"
   target_group_arn = aws_lb_target_group.my_tg_c.arn
 }

 condition {
   path_pattern {
     values = ["/register*"]
   }
 }
}
```

This is perhaps the most crucial part of this topic. Teams manage various types of workloads on various services offered by AWS, so the efficiency of the system depends on the rules configured in the Listener. 

Two main resource blocks are – action and condition.

## action
This defines the type of routing that needs to be configured. Apart from forward, some other values are redirect, fixed-response, authenticate-cognito, and authenticate-oidc.

1.When a listener rule uses the “Forward” action, the ALB forwards the request to a target group. (This is commonly used.)

2.The “Redirect” action instructs the ALB to respond to the request with a redirect to a different URL.

3. The “Fixed-Response” action allows you to configure the ALB to respond to a request with a fixed HTTP response.

4. The “Authenticate-Cognito” action is used to implement authentication using Amazon Cognito User Pools.

5. The “Authenticate-OIDC” action is used to implement authentication using an OpenID Connect (OIDC) identity provider.

## condition
We know by now that ALB is a Layer 7 load balancer, so the condition block defines conditions in various formats to leverage its capabilities.

1. host_header condition allows you to specify a list of host header patterns to match.

2. http_header condition allows you to match against HTTP headers.

3. Using http_request_method, you can specify a list of HTTP request methods or verbs to match.

4. path_pattern condition involves matching against path patterns in the request URL, which we have used in our example.

5. query_string allows you to match against query strings in the request URL.

6. source_ip allows you to specify a list of source IP CIDR notations to match.

 In the configuration above, we have mentioned the path_pattern in the condition block, which accepts /images* and /register* as path values to be matched, and “forwards” those requests to the corresponding target groups. Apply this updated configuration and verify if the rules are configured as expected.

 ## Finally Test it

 With help of your ALB DNS name Check this urls:
 http://my-alb-914057581.us-east-1.elb.amazonaws.com/images/

 http://my-alb-914057581.us-east-1.elb.amazonaws.com/

 http://my-alb-914057581.us-east-1.elb.amazonaws.com/registers/

 Dont forget "/" At End.
