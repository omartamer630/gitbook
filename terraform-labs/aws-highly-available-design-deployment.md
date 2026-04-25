---
cover: ../.gitbook/assets/1530036013-tf-aws-twitter.avif
coverY: 0
---

# (AWS) Highly available Design deployment

## Task Requirement

<figure><img src="../.gitbook/assets/image (61).png" alt=""><figcaption></figcaption></figure>

## Architecture

<figure><img src="../.gitbook/assets/image (3).png" alt=""><figcaption></figcaption></figure>

## File Structure

```basic
.
├── Highly_available_Design_deployment.pdf
├── README.md
├── app
│   ├── app.py
│   └── prerequisites.sh
├── asg.tf
├── bastionhost.tf
├── gateway.tf
├── images
│   └── Highly_available_Design_deployment.gif
├── loadbalancer.tf
├── provider.tf
├── rds.tf
├── routetable.tf
├── sg.tf
├── subnet.tf
├── terraform.tfvars
├── transitgw.tf
├── variable.tf
└── vpc.tf
```

## Terraform Resources + Code Steps

### Configure AWS Provider

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }

  cloud {
    organization = "terraform_project_01"
    workspaces {
      name = "dev"
    }
  }
}



provider "aws" {
  region = var.AWS_DEFAULT_REGION
}
```

used HCP as `Statefile`  saver (for security and team work things).

### Setup Variables

```hcl
variable "AWS_DEFAULT_REGION" {
  type        = string
  description = "AWS Region"
  default     = "us-east-1"
}

variable "environment" {
  description = "Most Used Variables in All services"
  type        = string
}

variable "cidr" {
  description = "VPCs CIDR blocks"
  type = list(object({
    cidr = string
    name = string
  }))
}
```

will use these variables in code later.

### Set Variables

```hcl
cidr = [
  {
    cidr = "11.0.0.0/16"
    name = "VPC1"
    }, {
    cidr = "12.0.0.0/16"
    name = "VPC2"
    }, {
    cidr = "13.0.0.0/16"
    name = "VPC3"
  }
]

environment = "dev"
```

### VPCs Resource Deployment

```hcl
# VPC_1 Creation
resource "aws_vpc" "vpc_1" {
  cidr_block           = var.cidr[0].cidr
  enable_dns_hostnames = true
  enable_dns_support   = true
  tags = {
    Name = "${var.environment}-vpc-1"
  }
}

# VPC_2 Creation
resource "aws_vpc" "vpc_2" {
  cidr_block           = var.cidr[1].cidr
  enable_dns_hostnames = true
  enable_dns_support   = true
  tags = {
    Name = "${var.environment}-vpc-2"
  }
}

# VPC_3 Creation
resource "aws_vpc" "vpc_3" {
  cidr_block           = var.cidr[2].cidr
  enable_dns_hostnames = true
  enable_dns_support   = true
  tags = {
    Name = "${var.environment}-vpc-3"
  }
}
```

Created Three VPCs,  with enabled DNS hostname and support for flexible communicate.

### Subnets Resource Deployment

```hcl
# VPC_1 Subnets
resource "aws_subnet" "vpc_1_private_subnet_asg_a" {
  vpc_id            = aws_vpc.vpc_1.id
  cidr_block        = cidrsubnet(aws_vpc.vpc_1.cidr_block, 8, 0)
  availability_zone = "${var.AWS_DEFAULT_REGION}a"
  tags = {
    Name = "${var.environment}-vpc-1-priv-subnet-asg-a"
  }
}
resource "aws_subnet" "vpc_1_private_subnet_asg_b" {
  vpc_id            = aws_vpc.vpc_1.id
  cidr_block        = cidrsubnet(aws_vpc.vpc_1.cidr_block, 8, 1)
  availability_zone = "${var.AWS_DEFAULT_REGION}b"
  tags = {
    Name = "${var.environment}-vpc-1-priv-subnet-asg-b"
  }
}

resource "aws_subnet" "vpc_1_private_subnet_rds_a" {
  vpc_id            = aws_vpc.vpc_1.id
  cidr_block        = cidrsubnet(aws_vpc.vpc_1.cidr_block, 8, 2)
  availability_zone = "${var.AWS_DEFAULT_REGION}a"
  tags = {
    Name = "${var.environment}-vpc-1-priv-subnet-rds-a"
  }
}

resource "aws_subnet" "vpc_1_private_subnet_rds_b" {
  vpc_id            = aws_vpc.vpc_1.id
  cidr_block        = cidrsubnet(aws_vpc.vpc_1.cidr_block, 8, 3)
  availability_zone = "${var.AWS_DEFAULT_REGION}b"
  tags = {
    Name = "${var.environment}-vpc-1-priv-subnet-rds-b"
  }
}

# VPC_2 Subnets
resource "aws_subnet" "vpc_2_private_subnet" {
  vpc_id            = aws_vpc.vpc_2.id
  cidr_block        = cidrsubnet(aws_vpc.vpc_2.cidr_block, 8, 0)
  availability_zone = "${var.AWS_DEFAULT_REGION}b"
  tags = {
    Name = "${var.environment}-vpc-2-priv-subnet"
  }
}

# VPC_3 Subnets
resource "aws_subnet" "vpc_3_public_subnet" {
  vpc_id            = aws_vpc.vpc_3.id
  cidr_block        = cidrsubnet(aws_vpc.vpc_3.cidr_block, 8, 0)
  availability_zone = "${var.AWS_DEFAULT_REGION}b"
  tags = {
    Name = "${var.environment}-vpc-3-pub-subnet"
  }
}
resource "aws_subnet" "vpc_3_public_subnet_a" {
  vpc_id            = aws_vpc.vpc_3.id
  cidr_block        = cidrsubnet(aws_vpc.vpc_3.cidr_block, 8, 2)
  availability_zone = "${var.AWS_DEFAULT_REGION}a"
  tags = {
    Name = "${var.environment}-vpc-3-pub-subnet-a"
  }
}
resource "aws_subnet" "vpc_3_private_subnet" {
  vpc_id            = aws_vpc.vpc_3.id
  cidr_block        = cidrsubnet(aws_vpc.vpc_3.cidr_block, 8, 1)
  availability_zone = "${var.AWS_DEFAULT_REGION}b" # Should be same as other Subnet that have NAT GW
  tags = {
    Name = "${var.environment}-vpc-3-private-subnet"
  }
}

resource "aws_subnet" "vpc_3_private_subnet_a" {
  vpc_id            = aws_vpc.vpc_3.id
  cidr_block        = cidrsubnet(aws_vpc.vpc_3.cidr_block, 8, 3)
  availability_zone = "${var.AWS_DEFAULT_REGION}a" # Should be same as other Subnet that have NAT GW
  tags = {
    Name = "${var.environment}-vpc-3-private-subnet_a"
  }
}

```

Created 4 private subnets for VPC 1, Two private subnets in different AZ's for the app and auto-scaling group, and other subnets for RDS and Standby RDS.

Created 1 private subnets for VPC 2,  private subnet to hold EC2 Server.

And finally created 4 Subnets for VPC 3, Two subnet is Public and the other two is Private, <mark style="color:red;">Note: If you have NAT GW in your public subnet you should have the same AZ because of Low latency and Cost</mark> .

### Gateway Resource Deployment

```hcl

resource "aws_internet_gateway" "vpc_3_internet_gateway" {
  vpc_id = aws_vpc.vpc_3.id
  tags = {
    Name = "${var.environment}-vpc-igw-3"
  }
}

resource "aws_eip" "eip_for_natgw" {
  tags = {
    Name = "nat-eip"
  }
}

resource "aws_nat_gateway" "forgtech_pub_vpc_3_nat_gw" {
  allocation_id = aws_eip.eip_for_natgw.id
  subnet_id     = aws_subnet.vpc_3_public_subnet.id

  tags = {
    Name = "${var.environment}-nat"
  }
  depends_on = [aws_internet_gateway.vpc_3_internet_gateway]
}
```

Created Internet Gateway to connect to the internet in VPC 3 as required, Created Elastic IP for NAT GW so it can have IP to connect to the internet, and finally Created NAT GW to connect private Servers to Internet without expose servers IPs.

### Transit  Gateway and Attachment Resource Deployment

```hcl
# Transit GW Creation
resource "aws_ec2_transit_gateway" "forgtech_transit_gw" {
  description                     = "managing the internet traffic flow-driven from/to the private subnets to the public subnets"
  default_route_table_association = "disable"
  dns_support                     = "enable"

}

# VPC_1  attachment with TGW
resource "aws_ec2_transit_gateway_vpc_attachment" "attach_rtb_1_to_tgw" {
  subnet_ids         = [aws_subnet.vpc_1_private_subnet_asg_a.id, aws_subnet.vpc_1_private_subnet_asg_b.id]
  transit_gateway_id = aws_ec2_transit_gateway.forgtech_transit_gw.id
  vpc_id             = aws_vpc.vpc_1.id
}

resource "aws_ec2_transit_gateway_route_table_association" "assoc_vpc_1" {
  transit_gateway_attachment_id  = aws_ec2_transit_gateway_vpc_attachment.attach_rtb_1_to_tgw.id
  transit_gateway_route_table_id = aws_ec2_transit_gateway_route_table.forgtech_tgw_rt.id
}

# VPC_2  attachment with TGW

resource "aws_ec2_transit_gateway_vpc_attachment" "attach_rtb_2_to_tgw" {
  subnet_ids         = [aws_subnet.vpc_2_private_subnet.id]
  transit_gateway_id = aws_ec2_transit_gateway.forgtech_transit_gw.id
  vpc_id             = aws_vpc.vpc_2.id
}


resource "aws_ec2_transit_gateway_route_table_association" "assoc_vpc_2" {
  transit_gateway_attachment_id  = aws_ec2_transit_gateway_vpc_attachment.attach_rtb_2_to_tgw.id
  transit_gateway_route_table_id = aws_ec2_transit_gateway_route_table.forgtech_tgw_rt.id
}


# VPC_3  attachment with TGW
resource "aws_ec2_transit_gateway_vpc_attachment" "attach_rtb_3_to_tgw" {
  subnet_ids         = [aws_subnet.vpc_3_private_subnet.id]
  transit_gateway_id = aws_ec2_transit_gateway.forgtech_transit_gw.id
  vpc_id             = aws_vpc.vpc_3.id
}

resource "aws_ec2_transit_gateway_route_table_association" "assoc_vpc_3" {
  transit_gateway_attachment_id  = aws_ec2_transit_gateway_vpc_attachment.attach_rtb_3_to_tgw.id
  transit_gateway_route_table_id = aws_ec2_transit_gateway_route_table.forgtech_tgw_rt.id
}
```

Created Transit to connect the three VPCs together through it like a (Hub).

Created Attachments for every VPC, it's like making a gate in both VPCs and Transit GW.

Created Association for every attachment, it's like to connect the gates together like making a road to travel through.

### Transit Route table and subnets route tables Resource Deployment

```hcl
# VPC_1 Route Table

# VPC 1  RTB and routes
resource "aws_route_table" "vpc_1_route_table" {
  vpc_id = aws_vpc.vpc_1.id

  route {
    cidr_block         = "0.0.0.0/0"
    transit_gateway_id = aws_ec2_transit_gateway.forgtech_transit_gw.id
  }
  tags = {
    Name = "${var.environment}-vpc-1-rtb-1"
  }
}

# Associate RTB 1 To private subnet asg a
resource "aws_route_table_association" "associate_route_table_1_to_vpc_1_private_subnet_asg_a" {
  subnet_id      = aws_subnet.vpc_1_private_subnet_asg_a.id
  route_table_id = aws_route_table.vpc_1_route_table.id
}

# Associate RTB 1 To private subnet asg b
resource "aws_route_table_association" "associate_route_table_1_to_vpc_1_private_subnet_asg_b" {
  subnet_id      = aws_subnet.vpc_1_private_subnet_asg_b.id
  route_table_id = aws_route_table.vpc_1_route_table.id
}

# Associate RTB 1 To private subnet rds a
resource "aws_route_table_association" "associate_route_table_1_to_vpc_1_private_subnet_rds_a" {
  subnet_id      = aws_subnet.vpc_1_private_subnet_rds_a.id
  route_table_id = aws_route_table.vpc_1_route_table.id
}

# Associate RTB 1 To private subnet rds b
resource "aws_route_table_association" "associate_route_table_1_to_vpc_1_private_subnet_rds_b" {
  subnet_id      = aws_subnet.vpc_1_private_subnet_rds_b.id
  route_table_id = aws_route_table.vpc_1_route_table.id
}

# VPC_2 Route Table

# VPC 2 RTB and routes
resource "aws_route_table" "vpc_2_route_table" {
  vpc_id = aws_vpc.vpc_2.id

  route {
    cidr_block         = "0.0.0.0/0"
    transit_gateway_id = aws_ec2_transit_gateway.forgtech_transit_gw.id
  }
  tags = {
    Name = "${var.environment}-vpc-rtb-2"
  }
}

# Associate RTB 2 To vpc 2 private subnet 
resource "aws_route_table_association" "associate_route_table_2_to_subnet" {
  subnet_id      = aws_subnet.vpc_2_private_subnet.id
  route_table_id = aws_route_table.vpc_2_route_table.id
}

# VPC_3 Route Table

# VPC_3 Public RTB and routes
resource "aws_route_table" "vpc_3_public_route_table" {
  vpc_id = aws_vpc.vpc_3.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.vpc_3_internet_gateway.id
  }
  route {
    cidr_block         = aws_vpc.vpc_1.cidr_block
    transit_gateway_id = aws_ec2_transit_gateway.forgtech_transit_gw.id
  }
  route {
    cidr_block         = aws_vpc.vpc_2.cidr_block
    transit_gateway_id = aws_ec2_transit_gateway.forgtech_transit_gw.id

  }
  tags = {
    Name = "${var.environment}-vpc-rtb-3"
  }
}

# Associate Public RTB to vpc 3 public subnet 
resource "aws_route_table_association" "associate_route_table_3_to_subnet" {
  subnet_id      = aws_subnet.vpc_3_public_subnet.id
  route_table_id = aws_route_table.vpc_3_public_route_table.id
}

resource "aws_route_table_association" "associate_route_table_3_to_subnet_a" {
  subnet_id      = aws_subnet.vpc_3_public_subnet_a.id
  route_table_id = aws_route_table.vpc_3_public_route_table.id
}

# VPC_3 private RTB and routes
resource "aws_route_table" "vpc_3_private_route_table" {
  vpc_id = aws_vpc.vpc_3.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.forgtech_pub_vpc_3_nat_gw.id
  }

  tags = {
    Name = "${var.environment}-vpc-3-private-rtb-3"
  }
}

# Associate Private RTB to vpc 3 private subnet 
resource "aws_route_table_association" "associate_vpc_3_private_subnet" {
  subnet_id      = aws_subnet.vpc_3_private_subnet.id
  route_table_id = aws_route_table.vpc_3_private_route_table.id
}

resource "aws_route_table_association" "associate_vpc_3_private_subnet_a" {
  subnet_id      = aws_subnet.vpc_3_private_subnet_a.id
  route_table_id = aws_route_table.vpc_3_private_route_table.id
}

# Transit GW Routes

resource "aws_ec2_transit_gateway_route_table" "forgtech_tgw_rt" {
  transit_gateway_id = aws_ec2_transit_gateway.forgtech_transit_gw.id
  tags = {
    Name = "${var.environment}-tgw-route-table"
  }
}

# Transit GW VPC_1 Route
resource "aws_ec2_transit_gateway_route" "route_to_vpc_1" {
  destination_cidr_block         = aws_vpc.vpc_1.cidr_block
  transit_gateway_route_table_id = aws_ec2_transit_gateway_route_table.forgtech_tgw_rt.id
  transit_gateway_attachment_id  = aws_ec2_transit_gateway_vpc_attachment.attach_rtb_1_to_tgw.id
}

# Transit GW VPC_2 Route

resource "aws_ec2_transit_gateway_route" "route_to_vpc_2" {
  destination_cidr_block         = aws_vpc.vpc_2.cidr_block
  transit_gateway_route_table_id = aws_ec2_transit_gateway_route_table.forgtech_tgw_rt.id
  transit_gateway_attachment_id  = aws_ec2_transit_gateway_vpc_attachment.attach_rtb_2_to_tgw.id
}

# Transit GW VPC_3 Route

resource "aws_ec2_transit_gateway_route" "route_to_vpc_3_priv_sub" {
  destination_cidr_block         = aws_subnet.vpc_3_private_subnet.cidr_block
  transit_gateway_route_table_id = aws_ec2_transit_gateway_route_table.forgtech_tgw_rt.id
  transit_gateway_attachment_id  = aws_ec2_transit_gateway_vpc_attachment.attach_rtb_3_to_tgw.id
}

resource "aws_ec2_transit_gateway_route" "route_to_vpc_3_pub_sub" {
  destination_cidr_block         = "0.0.0.0/0"
  transit_gateway_route_table_id = aws_ec2_transit_gateway_route_table.forgtech_tgw_rt.id
  transit_gateway_attachment_id  = aws_ec2_transit_gateway_vpc_attachment.attach_rtb_3_to_tgw.id
}
```

Created One Route table for VPC 1, routes accept any  traffic from/to transit GW, then associate every subnet in VPC 1 to this one route table.

Created One Route table for VPC 2, routes accept any  traffic from/to transit GW, then associate every subnet in VPC 2 to this one route table.

Created Two Route Table for VPC 3, first one  Public RTB that routes for 3 things

1. Routes to Internet GW
2. Routes to VPC 1 through Transit GW
3. Routes to VPC 2 through Transit GW

{% hint style="success" %}
now we Created public rtb in vpc 3 that accept traffic from internet and vpc 1 & 2, we will use IGW to get info from internet by NAT
{% endhint %}

then associate public subnet to the public RTB, second  one Private RTB that routes only for NAT GW, then associate private subnet to the private RTB.

{% hint style="success" %}
So now we Created private rtb in vpc 3 that take any traffic and pass it to NAT GW in public subnet
{% endhint %}

Created Transit GW Route table.

Then created  route to VPC 1, by adding VPC 1 CIDER block and transit GW Table ID and the expected gate of VPC 1 ID.

created  route to VPC 2, by adding VPC 1 CIDER block and transit GW Table ID and the expected gate of VPC 2 ID.

created  route to VPC 1, by adding VPC 3 CIDER block and transit GW Table ID and the expected gate of VPC 3 ID.

created route\_to\_vpc\_3\_pub\_sub and adding CIDR "0.0.0.0/0" and the expected gate of VPC 3 ID.

{% hint style="danger" %}
This route essentially handles all **other traffic** that might need to go outside VPC 3.
{% endhint %}

### Security Group Resource Deployment

```hcl
# VPC 1 Security group
resource "aws_security_group" "vpc_1_security_group" {
  vpc_id = aws_vpc.vpc_1.id

  # add HTTP ingress rule 
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # Add SSH ingress rule
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"] # Replace with a more restrictive CIDR if needed
  }

  # Add SSH ingress rule
  ingress {
    from_port   = -1
    to_port     = -1
    protocol    = "icmp"
    cidr_blocks = ["0.0.0.0/0"] # Replace with a more restrictive CIDR if needed
  }
  # Add RDS Postgres ingress rule
  ingress {
    from_port   = 5432
    to_port     = 5432
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"] # Replace with a more restrictive CIDR if needed
  }
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "VPC_1_SG"
  }

}
# VPC 2 Security group
resource "aws_security_group" "vpc_2_security_group" {
  vpc_id = aws_vpc.vpc_2.id

  # Add SSH ingress rule
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"] # Replace with a more restrictive CIDR if needed
  }

  # Add SSH ingress rule
  ingress {
    from_port   = -1
    to_port     = -1
    protocol    = "icmp"
    cidr_blocks = ["0.0.0.0/0"] # Replace with a more restrictive CIDR if needed
  }
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "VPC_2_SG"
  }

}

# VPC 3 Security group
resource "aws_security_group" "vpc_3_security_group" {
  vpc_id = aws_vpc.vpc_3.id

  # add HTTP ingress rule 
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # Add SSH ingress rule
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"] # Replace with a more restrictive CIDR if needed
  }

  # Add SSH ingress rule
  ingress {
    from_port   = -1
    to_port     = -1
    protocol    = "icmp"
    cidr_blocks = ["0.0.0.0/0"] # Replace with a more restrictive CIDR if needed
  }
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "VPC_3_SG"
  }

}
```

Created Security group for VPC 1,&#x20;

* Ingress rules: Allowed HTTP so i can see the app in port 80, Allowed SSH so i can securely check the app logs, Allowed ICMP to check connections, Allowed RDS postgress port the app can connect to Database.
* egress rule: Allowed all traffic to go out.

Created Security group for VPC 2,&#x20;

* Ingress rules:Allowed SSH so i can securely check ec2 logs, Allowed ICMP to check connection.
* egress rule: Allowed all traffic to go out.

Created Security group for VPC 3,&#x20;

* Ingress rules: Allowed HTTP so i can see the app in port 80, Allowed SSH so i can securely check the app logs.
* egress rule: Allowed all traffic to go our.

{% hint style="info" %}
SSH allowed so i can connect to any EC2 in every VPC.
{% endhint %}

### EC2 Resource Deployment

```hcl
# Generate a new key pair for the jumper to access private instances
resource "tls_private_key" "private_key_ec2s" {
  algorithm = "RSA"
  rsa_bits  = 2048
}
resource "aws_key_pair" "private_key_ec2s_pair" {
  key_name   = "private_key_ec2s"
  public_key = tls_private_key.private_key_ec2s.public_key_openssh
}
# Bastion Host for VPC 3 for Testing propose
resource "aws_instance" "bastion_host" {
  ami                         = "ami-0182f373e66f89c85"
  instance_type               = "t2.micro"
  subnet_id                   = aws_subnet.vpc_3_public_subnet.id
  vpc_security_group_ids      = [aws_security_group.vpc_3_security_group.id]
  associate_public_ip_address = true
  key_name                    = "forgtech-keypair"

  user_data = <<-EOF
              #!/bin/bash
              mkdir -p /home/ec2-user/.ssh
              echo '${tls_private_key.private_key_ec2s.private_key_pem}' > /home/ec2-user/.ssh/private_key_ec2s.pem
              chmod 600 /home/ec2-user/.ssh/private_key_ec2s.pem
              chown ec2-user:ec2-user /home/ec2-user/.ssh/private_key_ec2s.pem
              EOF
  tags = {
    Name = "${var.environment}-bastion-host"
  }
}
# Bastion Host for VPC 2 for Testing propose
resource "aws_instance" "ec2_vpc_2" {
  ami                         = "ami-0182f373e66f89c85"
  instance_type               = "t2.micro"
  subnet_id                   = aws_subnet.vpc_2_private_subnet.id
  vpc_security_group_ids      = [aws_security_group.vpc_2_security_group.id]
  key_name                    = aws_key_pair.private_key_ec2s_pair.key_name

  user_data = <<-EOF
              #!/bin/bash
              mkdir -p /home/ec2-user/.ssh
              echo '${tls_private_key.private_key_ec2s.private_key_pem}' > /home/ec2-user/.ssh/private_key_ec2s.pem
              chmod 600 /home/ec2-user/.ssh/private_key_ec2s.pem
              chown ec2-user:ec2-user /home/ec2-user/.ssh/private_key_ec2s.pem
              EOF

  tags = {
    Name = "${var.environment}-ec2-vpc-2"
  }
}
```

Created Key pair and make it download in bastion host so i can have SSH Connection to other EC2s easily. (in both VPC 3 & 2)

### RDS Resource Deployment

```hcl
resource "aws_db_instance" "forgtech-rds-postgresql" {
  identifier                  = "forgtech-postgres-db"
  allocated_storage           = 20
  storage_encrypted           = true
  engine                      = "postgres"
  engine_version              = "15.4"
  instance_class              = "db.t3.micro"
  apply_immediately           = true
  publicly_accessible         = false # default is false
  multi_az                    = true  # using stand alone DB
  skip_final_snapshot         = true  # after deleting RDS aws will not create snapshot 
  copy_tags_to_snapshot       = true  # default = false
  db_subnet_group_name        = aws_db_subnet_group.db-attached-subnet.id
  vpc_security_group_ids      = [aws_security_group.vpc_1_security_group.id]
  username                    = "omartamer"
  manage_master_user_password = true  # manage password using secret manager service
  auto_minor_version_upgrade  = false # default = false
  allow_major_version_upgrade = true  # default = true
  backup_retention_period     = 7     # default value is 7
  delete_automated_backups    = true  # default = true
  blue_green_update {
    enabled = true # Blue Depolyment (Prod) , Green Deployment (Staging) 
  }
  tags = {
    Name = "${var.environment}-forgtech-rds-posgress"
  }
}

resource "aws_db_subnet_group" "db-attached-subnet" {
  name = "forgtech-db-subnet-group"
  subnet_ids = [
    "${aws_subnet.vpc_1_private_subnet_rds_a.id}",
    "${aws_subnet.vpc_1_private_subnet_rds_b.id}"
  ]
  tags = {
    Name = "${var.environment}-db-subnets"
  }
}
```

Created RDS  with specific Requirment as the task and password managed by Secret Manager Service

Created Subnet group for db so i can include the primary AZ and the standby AZ

### Load Balancer Resource Deployment

```hcl
# Internal Load Balancer for VPC 1
resource "aws_lb" "internal_lb" {
  name                       = "dev-internal-alb"
  internal                   = true
  load_balancer_type         = "network"
  security_groups            = [aws_security_group.vpc_1_security_group.id]
  subnets                    = [aws_subnet.vpc_1_private_subnet_asg_a.id, aws_subnet.vpc_1_private_subnet_asg_b.id]
  enable_deletion_protection = false

  tags = {
    Name = "${var.environment}-internal"
  }
}

resource "aws_lb_target_group" "internal_tg" {
  name        = "internal-tg"
  port        = 80
  protocol    = "TCP"
  vpc_id      = aws_vpc.vpc_1.id
  target_type = "instance"

  health_check {
    port     = 80
    protocol = "HTTP"
    path     = "/"
  }

  tags = {
    Name = "${var.environment}-internal-tg"
  }
}

resource "aws_lb_listener" "internal_listener" {
  load_balancer_arn = aws_lb.internal_lb.arn
  port              = 80
  protocol          = "TCP"
  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.internal_tg.arn
  }
}

# Public Load Balancer

resource "aws_lb" "public_lb" {
  name               = "public-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.vpc_3_security_group.id]
  subnets            = [aws_subnet.vpc_3_public_subnet.id, aws_subnet.vpc_3_public_subnet_a.id]
  enable_deletion_protection = false

}

resource "aws_lb_target_group" "nginx_target_group" {
  name     = "nginx-target-group"
  port     = 80
  protocol = "HTTP"
  vpc_id   = aws_vpc.vpc_3.id
  target_type = "instance"
  health_check {
    protocol = "HTTP"
    port     = 80
    path     = "/" 
  }
}

resource "aws_lb_listener" "public_listener" {
  load_balancer_arn = aws_lb.public_lb.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.nginx_target_group.arn
  }
}
```

Created Two load balancer, One  Internal Load balancer for the application in VPC 1, and one Public Load balancer for the user.

### Auto-Scaling Group Resource Deployment

````hcl
# NGINX Configurations in ASG
resource "aws_launch_template" "nginx_proxy" {
  name_prefix   = "app-launch-template"
  image_id      = "ami-0182f373e66f89c85"
  instance_type = "t2.micro"
  key_name      = aws_key_pair.private_key_ec2s_pair.key_name
  network_interfaces {
    security_groups = [aws_security_group.vpc_3_security_group.id]
  }

  user_data = base64encode(<<-EOF
      #!/bin/bash
      sudo yum update -y
      sudo yum install -y nginx

      # Start and enable Nginx service
      sudo systemctl start nginx
      sudo systemctl enable nginx

      # Add server_names_hash_bucket_size to nginx.conf
      echo 'server_names_hash_bucket_size 128;' | sudo tee -a /etc/nginx/nginx.conf

      # Configure Nginx proxy settings
      echo 'server {
          listen 80;

          server_name ${aws_lb.public_lb.dns_name}:80; # Replace with your actual public-facing ALB DNS

          location / {
              proxy_pass http://${aws_lb.internal_lb.dns_name};  # Internal ALB DNS name or NLB
              proxy_set_header Host \$host;
              proxy_set_header X-Real-IP \$remote_addr;
              proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
              proxy_set_header X-Forwarded-Proto \$scheme;
          }
      }' | sudo tee /etc/nginx/conf.d/proxy.conf

      # Test Nginx configuration and reload
      sudo nginx -t
      sudo systemctl reload nginx

      EOF
  )
}

resource "aws_autoscaling_group" "asg_for_proxy_servers" {
  launch_template {
    id      = aws_launch_template.nginx_proxy.id
    version = "$Latest"
  }
  max_size            = 3
  min_size            = 2
  desired_capacity    = 2
  vpc_zone_identifier = [aws_subnet.vpc_3_private_subnet.id, aws_subnet.vpc_3_private_subnet_a.id]
  target_group_arns   = [aws_lb_target_group.nginx_target_group.id]
  tag {
    key                 = "Name"
    value               = "ASG_NGNINX_instance"
    propagate_at_launch = true
  }
  lifecycle {
    ignore_changes = [desired_capacity] # ignores any manual and automated change in ASG
  }
}

# EC2 Configurations in ASG
resource "aws_launch_template" "ec2s_app" {
  name_prefix   = "app-launch-template"
  image_id      = "ami-0182f373e66f89c85"
  instance_type = "t2.micro"
  key_name      = aws_key_pair.private_key_ec2s_pair.key_name
  network_interfaces {
    security_groups = [aws_security_group.vpc_1_security_group.id]
  }

  user_data = base64encode(<<-EOF
      sudo yum install git
      git clone https://github.com/omartamer630/Highly_available_Design_deployment.git
      cd Highly_available_Design_deployment/app/
      sudo bash prerequisites.sh
      nohup sudo python3 app.py > app.log 2>&1 &

      EOF
  )
}

resource "aws_autoscaling_group" "asg_private_subnets" {
  launch_template {
    id      = aws_launch_template.ec2s_app.id
    version = "$Latest"
  }
  max_size            = 3
  min_size            = 2
  desired_capacity    = 2
  vpc_zone_identifier = [aws_subnet.vpc_1_private_subnet_asg_a.id, aws_subnet.vpc_1_private_subnet_asg_b.id]
  target_group_arns   = [aws_lb_target_group.internal_tg.arn]
  tag {
    key                 = "Name"
    value               = "ASG_instance"
    propagate_at_launch = true
  }
  lifecycle {
    ignore_changes = [desired_capacity] # ignores any manual and automated change in ASG
  }
}

```
````

Created templat to nginx server to be proxy for Public ALB

Created ASG Service and attach it to VPC 3.

Created template so i can use it in multiple EC2 that the ASG Service will create in high load.

Created ASG Service and attach it to VPC 1 as Required.

{% hint style="danger" %}
Using Ignore\_changes to tell terraform to ingore any manual changes, its useful because some times you need to change things manually so when you make terraform apply it can be reconfigure.
{% endhint %}

## Check Code

<figure><img src="../.gitbook/assets/image (63).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (64).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (65).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (66).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (67).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (68).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (69).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (72).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (70).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (71).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (73).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (75).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (2).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (76).png" alt=""><figcaption></figcaption></figure>

Connected to Bastion host and pinged one server from VPC 1

<figure><img src="../.gitbook/assets/image (77).png" alt=""><figcaption></figcaption></figure>

Connected to one server in VPC 1 from Bastion host

<figure><img src="../.gitbook/assets/image (1).png" alt=""><figcaption></figcaption></figure>

and Finally internal Load Balancer can be accessed through nginx proxy and Public Application Load balancer.&#x20;



You can Check whole Task in here ([github](https://github.com/omartamer630/Highly_available_Design_deployment))

#### **That's it, quick and simple! 🚀 I hope this guide has sparked some ideas, and I’d love to hear your thoughts. Thanks!**
