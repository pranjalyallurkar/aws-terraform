# aws-terraform


###  IAM
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}

provider "aws" {
  region = "ap-south-1"
}

# Create IAM User
resource "aws_iam_user" "my_user" {
  name = "terraform-practice-user"

  tags = {
    Name        = "Terraform-IAM-User"
    Environment = "Dev"
  }
}

# Attach Read-Only S3 Policy
resource "aws_iam_user_policy_attachment" "s3_readonly" {
  user = aws_iam_user.my_user.name

  policy_arn = "arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess"
}

output "iam_username" {
  value = aws_iam_user.my_user.name
}

output "attached_policy" {
  value = aws_iam_user_policy_attachment.s3_readonly.policy_arn
}



### S3


terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}

# AWS Provider
provider "aws" {
  region = "ap-south-1"
}

# Create S3 Bucket
resource "aws_s3_bucket" "my_bucket" {
  bucket = "pranjal-terraform-demo-2026-unique123"

  tags = {
    Name        = "Terraform-S3-Bucket"
    Environment = "Dev"
  }
}

# Block public access
resource "aws_s3_bucket_public_access_block" "my_bucket_access" {
  bucket = aws_s3_bucket.my_bucket.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# Enable versioning
resource "aws_s3_bucket_versioning" "my_bucket_versioning" {
  bucket = aws_s3_bucket.my_bucket.id

  versioning_configuration {
    status = "Enabled"
  }
}

# Output bucket name
output "bucket_name" {
  description = "S3 bucket name"
  value       = aws_s3_bucket.my_bucket.id
}

# Output bucket ARN
output "bucket_arn" {
  description = "S3 bucket ARN"
  value       = aws_s3_bucket.my_bucket.arn
}


### VPC 

terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}

provider "aws" {
  region = "ap-south-1"
}

# Create VPC
resource "aws_vpc" "my_vpc" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = "My-Terraform-VPC"
  }
}

# Create Public Subnet
resource "aws_subnet" "public_subnet" {
  vpc_id                  = aws_vpc.my_vpc.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "ap-south-1a"
  map_public_ip_on_launch = true

  tags = {
    Name = "My-Public-Subnet"
  }
}

# Internet Gateway
resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.my_vpc.id

  tags = {
    Name = "My-Internet-Gateway"
  }
}

# Route Table
resource "aws_route_table" "public_rt" {
  vpc_id = aws_vpc.my_vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }

  tags = {
    Name = "My-Public-Route-Table"
  }
}

# Associate Route Table with Subnet
resource "aws_route_table_association" "public_assoc" {
  subnet_id      = aws_subnet.public_subnet.id
  route_table_id = aws_route_table.public_rt.id
}

output "vpc_id" {
  value = aws_vpc.my_vpc.id
}

output "subnet_id" {
  value = aws_subnet.public_subnet.id
}


### EC2

terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}

# AWS Provider
provider "aws" {
  region = "ap-south-1"
}

# Find the latest Ubuntu 24.04 LTS AMI
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd-gp3/ubuntu-noble-24.04-amd64-server-*"]
  }

  filter {
    name   = "architecture"
    values = ["x86_64"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

# Default VPC
data "aws_vpc" "default" {
  default = true
}

# Security Group
resource "aws_security_group" "ubuntu_sg" {
  name        = "ubuntu-terraform-sg"
  description = "Allow SSH access to Ubuntu EC2"
  vpc_id      = data.aws_vpc.default.id

  ingress {
    description = "SSH access"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "Ubuntu-Terraform-SG"
  }
}

# Create EC2 Instance
resource "aws_instance" "ubuntu_vm" {
  ami                         = data.aws_ami.ubuntu.id
  instance_type               = "t3.micro"
  vpc_security_group_ids      = [aws_security_group.ubuntu_sg.id]
  associate_public_ip_address = true

  root_block_device {
    volume_size           = 8
    volume_type           = "gp3"
    delete_on_termination = true
  }

  tags = {
    Name        = "Ubuntu-Free-Tier-VM"
    Environment = "Dev"
  }
}

# Output Instance ID
output "instance_id" {
  description = "EC2 instance ID"
  value       = aws_instance.ubuntu_vm.id
}

# Output Public IP
output "public_ip" {
  description = "Public IP address of the EC2 instance"
  value       = aws_instance.ubuntu_vm.public_ip
}


