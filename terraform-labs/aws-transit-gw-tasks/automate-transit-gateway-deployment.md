# Automate Transit gateway deployment

## Task

<figure><img src="../../.gitbook/assets/image (43).png" alt=""><figcaption><p>Task</p></figcaption></figure>

## Diagram

<figure><img src="../../.gitbook/assets/WeekEleven&#x26;Twelve.gif" alt=""><figcaption><p>Diagram 11-12 </p></figcaption></figure>

## File Structure

```basic
.
├── .circleci/
│   └── config.yml
│── terraform.lock.hcl
├── .gitignore
├── ec2.tf
├── provider.tf
├── README.md
├── sg.tf
├── subnet.tf
├── task11_12.pdf
├── terraform.tfvars
├── transitgw.tf
├── variable.tf
└── vpc.tf
```

## Steps

### Configure The Provider and Backend

* Create a file called `provider.tf`
* Copy Code Below

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }

  cloud {
    organization = "DevOps-Kitchen"
    workspaces {
      name = "DevOps-workshop"
    }
  }
}

provider "aws" {
  region = var.AWS_DEFAULT_REGION
}
```

in this code we Configured AWS as our provider and using HCP as backend that save our terraform state file in workspace called `DevOps-workshop`

### Variable Declaration

* Create a file called `variable.tf`&#x20;
* Copy Code Below

```hcl
variable "AWS_ACCESS_KEY_ID" {
  type        = string
  description = "AWS Access Key ID"
}

variable "AWS_SECRET_ACCESS_KEY" {
  type        = string
  description = "AWS Secret Access Key"
  sensitive   = true
}

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

variable "AZs" {
  description = "Subnets AZs"
  type        = list(string)
}
```

In this code created AWS Three main variable `access_key, secret_access_key, and region` we're using them for HCP Workspace access, in environment variable we achieve the tags for all services we will use for reusability, CIDR variable we declare VPCs CIDR  blocks and named it for every VPC we have.

### Variable Initialization

* Create a file called `terraform.tfvars`&#x20;
* Copy code below

```hcl
cidr = [
  {
    cidr = "11.0.0.0/16"
    name = "VPC1"
    }, {
    cidr = "11.0.1.0/24"
    name = "VPC1-private-subnet"
    }, {
    cidr = "12.0.0.0/16"
    name = "VPC2"
    }, {
    cidr = "12.0.1.0/24"
    name = "VPC2-private-subnet"
    }, {
    cidr = "13.0.0.0/16"
    name = "VPC3"
    }, {
    cidr = "13.0.1.0/24"
    name = "VPC3-public-subnet"
  }
]

AZs = ["us-east-1a", "us-east-1b", "us-east-1c"]

environment = "dev"
```

Initialization CIDR, AZs,  and environment variables as previous code

### VPCs Resource Deployment

* Create a file called `vpc.tf`
* Copy below code

```hcl
resource "aws_vpc" "vpc_1" {
  cidr_block = var.cidr[0].cidr
  tags = var.environment
}

resource "aws_vpc" "vpc_2" {
  cidr_block = var.cidr[2].cidr
  tags = var.environment
}

resource "aws_vpc" "vpc_3" {
  cidr_block = var.cidr[4].cidr
  tags = var.environment
}
```

in this code, created three VPCs with three difference CIDR with similar environment tags.

### Subnet Resource Deployment

* Create a file called `subnet.tf`
* Copy below code

```hcl
resource "aws_subnet" "vpc_1_private_subnet" {
  vpc_id            = aws_vpc.vpc_1.id
  cidr_block        = var.cidr[1].cidr
  availability_zone = var.AZs[0]
  tags = {
    Name = "${var.environment}-vpc-1-priv-subnet"
  }
}

resource "aws_route_table" "vpc_1_route_table" {
  vpc_id = aws_vpc.vpc_1.id

  route {
    cidr_block         = "0.0.0.0/0"
    transit_gateway_id = aws_ec2_transit_gateway.forgtech_transit_gw.id
  }
  tags = {
    Name = "${var.environment}-vpc-rtb-1"
  }
}

resource "aws_route_table_association" "associate_route_table_1_to_subnet" {
  subnet_id      = aws_subnet.vpc_1_private_subnet.id
  route_table_id = aws_route_table.vpc_1_route_table.id
}


resource "aws_subnet" "vpc_2_private_subnet" {
  vpc_id            = aws_vpc.vpc_2.id
  cidr_block        = var.cidr[3].cidr
  availability_zone = var.AZs[1]
  tags = {
    Name = "${var.environment}-vpc-2-priv-subnet"
  }
}

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

resource "aws_route_table_association" "associate_route_table_2_to_subnet" {
  subnet_id      = aws_subnet.vpc_2_private_subnet.id
  route_table_id = aws_route_table.vpc_2_route_table.id
}


resource "aws_subnet" "vpc_3_public_subnet" {
  vpc_id            = aws_vpc.vpc_3.id
  cidr_block        = var.cidr[5].cidr
  availability_zone = var.AZs[2]
  tags = {
    Name = "${var.environment}-vpc-3-pub-subnet"
  }
}

resource "aws_route_table" "vpc_3_public_route_table" {
  vpc_id = aws_vpc.vpc_3.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.vpc_3_internet_gateway.id
  }
  route {
    cidr_block         = var.cidr[0].cidr
    transit_gateway_id = aws_ec2_transit_gateway.forgtech_transit_gw.id
  }
  route {
    cidr_block         = var.cidr[2].cidr
    transit_gateway_id = aws_ec2_transit_gateway.forgtech_transit_gw.id

  }
  tags = {
    Name = "${var.environment}-vpc-rtb-3"
  }
}

resource "aws_route_table_association" "associate_route_table_3_to_subnet" {
  subnet_id      = aws_subnet.vpc_3_public_subnet.id
  route_table_id = aws_route_table.vpc_3_public_route_table.id
}

resource "aws_internet_gateway" "vpc_3_internet_gateway" {
  vpc_id = aws_vpc.vpc_3.id
  tags = {
    Name = "${var.environment}-vpc-igw-3"
  }
}

```

in this code, created three subnets for every VPC, in this task it was required to have Two private subnets for 2 VPCs and One public subnet for 1 VPCs, In the two private subnets, created route table that routes everything to transit gateway, in the public subnet, created routes via Internet gateway, VPC\_1 & VPC\_2 CIDR block through transit gateway (using TGW ID).

## Test our Progress

Let's Test our code until now if its working or not, but first lets Organize our code using `terraform fmt,` then let's check our code syntax using `terraform validate`

<figure><img src="../../.gitbook/assets/image (44).png" alt=""><figcaption></figcaption></figure>

All gooduntil now, let's Check our plan using `terraform plan`&#x20;

<figure><img src="../../.gitbook/assets/image (45).png" alt=""><figcaption></figcaption></figure>

next, use `terraform apply` to apply our code and check our AWS Console.

<figure><img src="../../.gitbook/assets/image (47).png" alt=""><figcaption><p>VPCs Applied</p></figcaption></figure>

<figure><img src="../../.gitbook/assets/image (48).png" alt=""><figcaption><p>Route table &#x26; IGW Applied</p></figcaption></figure>

In the end, use terraform destroy to end all your applied resources.

### Transit Gateway Resource Deployment

* Create a file called `transitgw.tf`&#x20;
* Copy Code below

```hcl
resource "aws_ec2_transit_gateway" "forgtech_transit_gw" {
  description                     = "managing the internet traffic flow-driven from/to the private subnets to the public subnets"
  default_route_table_association = "disable"
}

resource "aws_ec2_transit_gateway_route_table" "forgtech_tgw_rt" {
  transit_gateway_id = aws_ec2_transit_gateway.forgtech_transit_gw.id
  tags = {
    Name = "${var.environment}-tgw-route-table"
  }
}
```

in this block of code, created Transit Gateway with disabled default route table of TGW because the automatic attachment of other VPCs, it give us more control over Routes in TGW, then created route table for TGW.

```hcl
resource "aws_ec2_transit_gateway_vpc_attachment" "attach_rtb_1_to_tgw" {
  subnet_ids         = [aws_subnet.vpc_1_private_subnet.id]
  transit_gateway_id = aws_ec2_transit_gateway.forgtech_transit_gw.id
  vpc_id             = aws_vpc.vpc_1.id
}

resource "aws_ec2_transit_gateway_route_table_association" "assoc_vpc_1" {
  transit_gateway_attachment_id  = aws_ec2_transit_gateway_vpc_attachment.attach_rtb_1_to_tgw.id
  transit_gateway_route_table_id = aws_ec2_transit_gateway_route_table.forgtech_tgw_rt.id
}

resource "aws_ec2_transit_gateway_vpc_attachment" "attach_rtb_2_to_tgw" {
  subnet_ids         = [aws_subnet.vpc_2_private_subnet.id]
  transit_gateway_id = aws_ec2_transit_gateway.forgtech_transit_gw.id
  vpc_id             = aws_vpc.vpc_2.id
}

resource "aws_ec2_transit_gateway_route_table_association" "assoc_vpc_2" {
  transit_gateway_attachment_id  = aws_ec2_transit_gateway_vpc_attachment.attach_rtb_2_to_tgw.id
  transit_gateway_route_table_id = aws_ec2_transit_gateway_route_table.forgtech_tgw_rt.id
}

resource "aws_ec2_transit_gateway_vpc_attachment" "attach_rtb_3_to_tgw" {
  subnet_ids         = [aws_subnet.vpc_3_public_subnet.id]
  transit_gateway_id = aws_ec2_transit_gateway.forgtech_transit_gw.id
  vpc_id             = aws_vpc.vpc_3.id
}

resource "aws_ec2_transit_gateway_route_table_association" "assoc_vpc_3" {
  transit_gateway_attachment_id  = aws_ec2_transit_gateway_vpc_attachment.attach_rtb_3_to_tgw.id
  transit_gateway_route_table_id = aws_ec2_transit_gateway_route_table.forgtech_tgw_rt.id
}
```

In this code, create attachment (We could say like arrow that point to our VPC) for every VPC, then associated attachment to transit gateway route table for every VPC, so now we have attached every VPC and associated with Transit gateway.

```hcl
resource "aws_ec2_transit_gateway_route" "route_to_vpc_1" {
  destination_cidr_block         = var.cidr[0].cidr
  transit_gateway_route_table_id = aws_ec2_transit_gateway_route_table.forgtech_tgw_rt.id
  transit_gateway_attachment_id  = aws_ec2_transit_gateway_vpc_attachment.attach_rtb_1_to_tgw.id
}

resource "aws_ec2_transit_gateway_route" "route_to_vpc_2" {
  destination_cidr_block         = var.cidr[2].cidr
  transit_gateway_route_table_id = aws_ec2_transit_gateway_route_table.forgtech_tgw_rt.id
  transit_gateway_attachment_id  = aws_ec2_transit_gateway_vpc_attachment.attach_rtb_2_to_tgw.id
}

resource "aws_ec2_transit_gateway_route" "route_to_vpc_3" {
  destination_cidr_block         = var.cidr[4].cidr
  transit_gateway_route_table_id = aws_ec2_transit_gateway_route_table.forgtech_tgw_rt.id
  transit_gateway_attachment_id  = aws_ec2_transit_gateway_vpc_attachment.attach_rtb_3_to_tgw.id
}
```

In this code, created routes that specify where traffic should go if destination of Specific CIDR, in `transit_gateway_route_table_id`  attribute we tell the traffic use transit gateway route table, in  `transit_gateway_attachment_id` attribute we tell the traffic go via specific Attachment.

### Security Group Resource Deployment

* Create a file called `sg.tf`

```hcl
resource "aws_security_group" "forgtech_ec2_sg_vpc_2" {
  name        = "ec2-sg"
  description = "ec2 rules"
  vpc_id      = aws_vpc.vpc_2.id
  tags = {
    Name = "${var.environment}-EC2-SG-2"
  }
}

resource "aws_vpc_security_group_ingress_rule" "forgtech-ec2-ingress_vpc_2" {
  security_group_id = aws_security_group.forgtech_ec2_sg_vpc_2.id
  ip_protocol       = "tcp"
  from_port         = 22
  to_port           = 22
  cidr_ipv4         = "0.0.0.0/0"
}
resource "aws_vpc_security_group_egress_rule" "forgtech-ec2-egress_vpc_2" {
  security_group_id = aws_security_group.forgtech_ec2_sg_vpc_2.id
  ip_protocol       = "-1"
  cidr_ipv4         = "0.0.0.0/0"
}


resource "aws_security_group" "forgtech_ec2_sg_vpc_3" {
  name        = "ec2-sg"
  description = "ec2 rules"
  vpc_id      = aws_vpc.vpc_3.id
  tags = {
    Name = "${var.environment}-EC2-SG-2"
  }
}

resource "aws_vpc_security_group_ingress_rule" "forgtech-ec2-ingress_vpc_3" {
  security_group_id = aws_security_group.forgtech_ec2_sg_vpc_3.id
  ip_protocol       = "tcp"
  from_port         = 22
  to_port           = 22
  cidr_ipv4         = "0.0.0.0/0"
}
resource "aws_vpc_security_group_egress_rule" "forgtech-ec2-egress_vpc_3" {
  security_group_id = aws_security_group.forgtech_ec2_sg_vpc_3.id
  ip_protocol       = "-1"
  cidr_ipv4         = "0.0.0.0/0"
}

```

in this code, we use SSH ports so we can access and test our task

### EC2 Resource Deployment

* Create a file called `ec2.tf`&#x20;

```hcl
# Create EC2 for VPC-2
resource "aws_instance" "ec2_vpc_2" {
  ami                    = "ami-0a3c3a20c09d6f377"
  instance_type          = "t2.micro"
  subnet_id              = aws_subnet.vpc_2_private_subnet.id
  vpc_security_group_ids = [aws_security_group.forgtech_ec2_sg_vpc_2.id]
  key_name               = "forgtech-keypair"

  tags = {
    Name = "${var.environment}-EC2-2"
  }
}


# Create EC2 for VPC-3
resource "aws_instance" "ec2_vpc_3" {
  ami                         = "ami-0a3c3a20c09d6f377"
  instance_type               = "t2.micro"
  associate_public_ip_address = true
  subnet_id                   = aws_subnet.vpc_3_public_subnet.id
  vpc_security_group_ids      = [aws_security_group.forgtech_ec2_sg_vpc_3.id]
  key_name                    = "forgtech-keypair"

  tags = {
    Name = "${var.environment}-EC2-3"
  }
}
```

in this code, created Two ec2 in two difference subnets ( Private , public ) to check our transit gateway.

## Final Terraform Test &#x20;

<figure><img src="../../.gitbook/assets/image (49).png" alt=""><figcaption><p>VPCs Applied</p></figcaption></figure>

<figure><img src="../../.gitbook/assets/image (50).png" alt=""><figcaption><p>Subnets Applied</p></figcaption></figure>

<figure><img src="../../.gitbook/assets/image (51).png" alt=""><figcaption><p>Route table Applied</p></figcaption></figure>

<figure><img src="../../.gitbook/assets/image (52).png" alt=""><figcaption><p>VPC_3 routes Applied</p></figcaption></figure>

<figure><img src="../../.gitbook/assets/image (53).png" alt=""><figcaption><p>TGW Applied</p></figcaption></figure>

<figure><img src="../../.gitbook/assets/image (54).png" alt=""><figcaption><p>TGW Attachments Applied</p></figcaption></figure>

<figure><img src="../../.gitbook/assets/image (55).png" alt=""><figcaption><p>TGW Route table Applied</p></figcaption></figure>

<figure><img src="../../.gitbook/assets/image (56).png" alt=""><figcaption><p>EC2 Applied</p></figcaption></figure>

<figure><img src="../../.gitbook/assets/image (57).png" alt=""><figcaption><p>EC2_VPC_3 Connected</p></figcaption></figure>

<figure><img src="../../.gitbook/assets/image (58).png" alt=""><figcaption><p>EC2_VPC_2 Couldn't connect (Because it's private subnet)</p></figcaption></figure>

Used `scp -i "forgtech-keypair.pem" /home/spectre/devops/forgtech-keypair.pem ec2-user@54.84.247.131:/home/ec2-user/ forgtech-keypair.pem`  Command to transfer EC2\_VPC\_2 Key to EC2\_VPC\_3&#x20;

<figure><img src="../../.gitbook/assets/image (59).png" alt=""><figcaption><p>Key transferred and Could connect to EC2_VPC_2  VIA EC2_VPC_3</p></figcaption></figure>

## CircleCI Configuration

* Create a file called .circleci/config.yml

```yaml
executors:
  terraform:
    docker:
      - image: hashicorp/terraform:1.5.0
```

in this code, making docker image for terraform

```yaml
jobs:
  terraform-plan:
    executor: terraform
    steps:
      - checkout
      - run:
          name: Install AWS CLI
          command: |
            apk add --no-cache python3 py3-pip && pip3 install awscli
      - run:
          name: Configure AWS CLI
          command: |
            aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID
            aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY
            aws configure set region $AWS_DEFAULT_REGION
      - run:
          name: Install jq
          command: apk add --no-cache jq
      - run:
          name: Initialize Terraform
          command: terraform init
      - run:
          name: Terraform Plan
          command: terraform plan -out=tfplan
      - run:
          name: Review Plan Output
          command: |
            terraform show -json tfplan | jq '.resource_changes[] | {type: .type, name: .name, change: .change}' > plan_output.json
            cat plan_output.json

  terraform-apply:
    executor: terraform
    steps:
      - checkout
      - run:
          name: Initialize Terraform
          command: terraform init
      - run:
          name: Terraform Apply
          command: terraform apply -auto-approve

  terraform-destroy:
    executor: terraform
    steps:
      - checkout
      - run:
          name: Initialize Terraform
          command: terraform init
      - run:
          name: Terraform Destroy
          command: terraform destroy -auto-approve
```

in this code, create three jobs that has execute in docker container that has terraform initilazed&#x20;

1. terraform-plan job -> clone our repo by using built-in function checkout, then install AWS CLI so we can run our terraform code, install JQ to filter tfplan and can read our terraform plan, then using terraform init to initialize terraform, then terraform plan step and redirected in tfplan file to read our plan, then review plan by using terrafom show and some filters with jq.
2. terraform-apply -> uses terraform apply -auto-approve&#x20;
3. terraform-destroy-> uses terraform destroy-auto-approve&#x20;

```yaml
workflows:
  terraform-workflow:
    jobs:
      - terraform-plan
      - hold-apply:
          type: approval
          requires:
            - terraform-plan
      - terraform-apply:
          requires:
            - terraform-plan
            - hold-apply
      - hold-destroy:
          type: approval
          requires:
            - terraform-apply
      - terraform-destroy:
          requires:
            - terraform-apply
            - hold-destroy
```

in this code, organize our jobs, so in first terraform-plan job will execute, then apply hold strategy to ready the plan content, then approve or cancel terraform plan, if approved terraform-apply will run else it will exit, and same for terraform-destroy.

## CircleCI Pipeline Test

<figure><img src="../../.gitbook/assets/image (13).png" alt=""><figcaption><p>CircleCI &#x26; Terraform Applied</p></figcaption></figure>

**You can check whole Task in here (** [**Github** ](https://github.com/omartamer630/transitgateway-circleci)**)**

## Conclustion

Created Two VPC with Two Private Subnets that routes all traffic to TransitGW, and One VPC with One Public Subnet that routes to Internate gateway and  routes VPC\_1  & VPC\_2 via TransitGW, Then we managed CircleCI to automate all of this.
