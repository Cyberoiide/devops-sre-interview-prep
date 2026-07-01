# Exercise — Build a minimal secure VPC

This lab defines a small but real secure VPC in **Terraform** — public + private
subnets across two AZs, an internet gateway, a NAT gateway, tiered security
groups, an **S3 gateway endpoint**, and **VPC Flow Logs** — and validates it
with `terraform plan`. A second section builds the same primitives against
**LocalStack** with the AWS CLI so you can inspect route tables and SG rules for
real, for free.

## What's honest about this lab

- **The Terraform is the real, applyable config** and `plan`s with no spend. A
  live `apply` costs money mainly because of the **NAT gateway** (~$0.045/hr +
  data) — it is *not* Free-Tier. Don't leave a NAT gateway running.
- **LocalStack Community emulates the VPC control plane** (VPCs, subnets, route
  tables, IGWs, SGs, NACLs, endpoints) well enough to practice the structure and
  read back the wiring. It does **not** run real instances or route real
  packets, so "does traffic actually flow" needs a real account.
- **VPC Flow Logs** delivery is best verified on a real account (LocalStack
  accepts the resource but won't generate live flow records).

## Prerequisites

- `terraform` >= 1.5 (or `tofu`).
- `docker` + [LocalStack](https://docs.localstack.dev/getting-started/) + the
  `awslocal` wrapper (`pip install awscli-local`).
- AWS CLI v2.

---

## Part A — Define the VPC in Terraform (`plan`-only)

`providers.tf`:

```hcl
terraform {
  required_version = ">= 1.5"
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
  }
}
provider "aws" {
  region = "eu-west-1"
}
```

`main.tf`:

```hcl
data "aws_availability_zones" "available" {
  state = "available"
}

locals {
  azs             = slice(data.aws_availability_zones.available.names, 0, 2)
  public_cidrs    = ["10.0.0.0/24", "10.0.1.0/24"]
  private_cidrs   = ["10.0.10.0/24", "10.0.11.0/24"]
}

resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true
  tags = { Name = "q203-secure-vpc" }
}

########################################
# Subnets: public (per AZ) + private (per AZ)
########################################
resource "aws_subnet" "public" {
  count                   = 2
  vpc_id                  = aws_vpc.main.id
  cidr_block              = local.public_cidrs[count.index]
  availability_zone       = local.azs[count.index]
  map_public_ip_on_launch = true          # public tier gets public IPs
  tags = { Name = "q203-public-${count.index}", Tier = "public" }
}

resource "aws_subnet" "private" {
  count             = 2
  vpc_id            = aws_vpc.main.id
  cidr_block        = local.private_cidrs[count.index]
  availability_zone = local.azs[count.index]
  # No public IPs. No IGW route. This tier is not internet-reachable.
  tags = { Name = "q203-private-${count.index}", Tier = "private" }
}

########################################
# Internet gateway + public route table
########################################
resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id   # public subnets reach the internet
  }
}

resource "aws_route_table_association" "public" {
  count          = 2
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

########################################
# NAT gateway (in a public subnet) + private route table
# Outbound-only internet for the private tier.
########################################
resource "aws_eip" "nat" {
  domain = "vpc"
}

resource "aws_nat_gateway" "nat" {
  allocation_id = aws_eip.nat.id
  subnet_id     = aws_subnet.public[0].id
  depends_on    = [aws_internet_gateway.igw]
}

resource "aws_route_table" "private" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.nat.id     # egress via NAT, no inbound
  }
}

resource "aws_route_table_association" "private" {
  count          = 2
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private.id
}

########################################
# Security groups — tiered, least privilege
########################################
# ALB: allow 443 from the world.
resource "aws_security_group" "alb" {
  name   = "q203-alb"
  vpc_id = aws_vpc.main.id
  ingress {
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

# App: allow 8080 ONLY from the ALB SG (SG-to-SG reference, not a CIDR).
resource "aws_security_group" "app" {
  name   = "q203-app"
  vpc_id = aws_vpc.main.id
  ingress {
    from_port       = 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
  }
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# DB: allow 5432 ONLY from the app SG. No egress to the internet.
resource "aws_security_group" "db" {
  name   = "q203-db"
  vpc_id = aws_vpc.main.id
  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]
  }
  # No 0.0.0.0/0 egress — the DB has no business calling the internet.
}

########################################
# S3 gateway endpoint — S3 traffic stays on the AWS backbone (free)
########################################
resource "aws_vpc_endpoint" "s3" {
  vpc_id            = aws_vpc.main.id
  service_name      = "com.amazonaws.eu-west-1.s3"
  vpc_endpoint_type = "Gateway"
  route_table_ids   = [aws_route_table.private.id]   # inject the S3 prefix route
}

########################################
# VPC Flow Logs -> CloudWatch Logs
########################################
resource "aws_cloudwatch_log_group" "flow" {
  name              = "/vpc/q203/flowlogs"
  retention_in_days = 14
}

data "aws_iam_policy_document" "flow_assume" {
  statement {
    actions = ["sts:AssumeRole"]
    principals {
      type        = "Service"
      identifiers = ["vpc-flow-logs.amazonaws.com"]
    }
  }
}

resource "aws_iam_role" "flow" {
  name               = "q203-flowlogs"
  assume_role_policy = data.aws_iam_policy_document.flow_assume.json
}

resource "aws_iam_role_policy" "flow" {
  role   = aws_iam_role.flow.id
  policy = jsonencode({
    Version = "2012-10-17",
    Statement = [{
      Effect   = "Allow",
      Action   = ["logs:CreateLogStream", "logs:PutLogEvents", "logs:DescribeLogStreams"],
      Resource = "${aws_cloudwatch_log_group.flow.arn}:*"
    }]
  })
}

resource "aws_flow_log" "vpc" {
  vpc_id          = aws_vpc.main.id
  traffic_type    = "ALL"                    # ACCEPT + REJECT
  log_destination = aws_cloudwatch_log_group.flow.arn
  iam_role_arn    = aws_iam_role.flow.arn
}
```

Validate:

```bash
terraform init
terraform validate      # "Success!"
terraform plan
```

**Confirm the security story in the plan:**

1. Private subnets have **no IGW route** — their route table sends `0.0.0.0/0`
   to the **NAT gateway** (outbound only). Public subnets route to the **IGW**.
2. SGs are **tiered and reference each other** (`app` trusts `alb`, `db` trusts
   `app`) instead of using broad CIDRs — least privilege.
3. The `db` SG has **no `0.0.0.0/0` egress**.
4. The **S3 gateway endpoint** is associated with the private route table.
5. **Flow Logs** capture `ALL` traffic to CloudWatch.

---

## Part B — Build the VPC primitives on LocalStack (free)

```bash
localstack start -d

# VPC + one public + one private subnet.
VPC=$(awslocal ec2 create-vpc --cidr-block 10.0.0.0/16 --query Vpc.VpcId --output text)
PUB=$(awslocal ec2 create-subnet --vpc-id $VPC --cidr-block 10.0.0.0/24 --query Subnet.SubnetId --output text)
PRIV=$(awslocal ec2 create-subnet --vpc-id $VPC --cidr-block 10.0.10.0/24 --query Subnet.SubnetId --output text)

# IGW for the public subnet.
IGW=$(awslocal ec2 create-internet-gateway --query InternetGateway.InternetGatewayId --output text)
awslocal ec2 attach-internet-gateway --vpc-id $VPC --internet-gateway-id $IGW

# Public route table: 0.0.0.0/0 -> IGW.
PUBRT=$(awslocal ec2 create-route-table --vpc-id $VPC --query RouteTable.RouteTableId --output text)
awslocal ec2 create-route --route-table-id $PUBRT --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW
awslocal ec2 associate-route-table --route-table-id $PUBRT --subnet-id $PUB

# Private route table: intentionally NO 0.0.0.0/0 -> IGW route.
PRIVRT=$(awslocal ec2 create-route-table --vpc-id $VPC --query RouteTable.RouteTableId --output text)
awslocal ec2 associate-route-table --route-table-id $PRIVRT --subnet-id $PRIV

# Tiered SGs with SG-to-SG references.
ALB=$(awslocal ec2 create-security-group --group-name alb --description alb --vpc-id $VPC --query GroupId --output text)
APP=$(awslocal ec2 create-security-group --group-name app --description app --vpc-id $VPC --query GroupId --output text)
awslocal ec2 authorize-security-group-ingress --group-id $ALB --protocol tcp --port 443 --cidr 0.0.0.0/0
# app trusts ONLY the alb SG on 8080 — not a CIDR.
awslocal ec2 authorize-security-group-ingress --group-id $APP --protocol tcp --port 8080 --source-group $ALB

# S3 gateway endpoint bound to the private route table.
awslocal ec2 create-vpc-endpoint \
  --vpc-id $VPC --service-name com.amazonaws.us-east-1.s3 \
  --route-table-ids $PRIVRT --vpc-endpoint-type Gateway
```

**Inspect what you built — prove the private subnet has no internet route:**

```bash
# Public RT should show a 0.0.0.0/0 -> igw route; private RT should NOT.
awslocal ec2 describe-route-tables --route-table-ids $PUBRT \
  --query 'RouteTables[0].Routes'
awslocal ec2 describe-route-tables --route-table-ids $PRIVRT \
  --query 'RouteTables[0].Routes'      # no 0.0.0.0/0 entry

# The app SG rule references the alb SG (UserIdGroupPairs), not an IP range.
awslocal ec2 describe-security-groups --group-ids $APP \
  --query 'SecurityGroups[0].IpPermissions'
```

Teardown:

```bash
localstack stop
```

---

## Part C — Real-account notes (optional)

- To watch **Flow Logs** actually populate, `apply` on a real account, launch a
  tiny instance, generate traffic, and read `/vpc/q203/flowlogs` in CloudWatch.
- The **NAT gateway bills hourly** even idle — `terraform destroy` it as soon as
  you're done. A cheaper practice alternative for outbound is a NAT *instance*,
  but production uses the managed NAT gateway.
- Verify the S3 gateway endpoint works by `aws s3 ls` from a private-subnet
  instance **with no NAT route** — it still works because traffic goes over the
  endpoint, not the internet.
