# terraformomno
terraform script for aws
1. AWS პროვაიდერის კონფიგურაცია
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.0"
    }
  }
}

provider "aws" {
  access_key = "..." # aws-ის access Key
  secret_key = "..." # aws-ის secret Key
  region     = "us-east-1" # რეგიონი, სადაც რესურსები შეიქმნება
}
2. VPC-ს  შექმნა
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = "main-vpc"
  }
}
3. საჯარო და პირადი სუბნეტების შექმნა
resource "aws_subnet" "public" {
  count                   = 2
  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet(aws_vpc.main.cidr_block, 8, count.index)
  availability_zone       = element(data.aws_availability_zones.available.names, count.index)
  map_public_ip_on_launch = true

  tags = {
    Name = "public-subnet-${count.index}"
  }
}
იქმნება 2 საჯარო სუბნეტი, რომლებიც საჯაროდ ხელმისაწვდომია ინტერნეტიდან
resource "aws_subnet" "private" {
  count             = 2
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(aws_vpc.main.cidr_block, 8, count.index + 2)
  availability_zone = element(data.aws_availability_zones.available.names, count.index)

  tags = {
    Name = "private-subnet-${count.index}"
  }
}
4. Security Group შექმნა
resource "aws_security_group" "eks_sg" {
  name        = "eks-security-group"
  description = "EKS security group"
  vpc_id      = aws_vpc.main.id
}
5. IAM როლების შექმნა
resource "aws_iam_role" "eks_cluster_role" {
  name = "eks-cluster-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = {
        Service = "eks.amazonaws.com"
      }
      Action = "sts:AssumeRole"
    }]
  })
}
იქმნება iam როლი eks კლასტერისთვის რომელიც საშუალებას აძლევს eks-ს იმოქმედოს aws-ზე
resource "aws_iam_role_policy_attachment" "eks_cluster_policy" {
  role       = aws_iam_role.eks_cluster_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSClusterPolicy"
}
6. eks კლასტერის შექმნა
resource "aws_eks_cluster" "eks" {
  name     = "my-eks-cluster"
  role_arn = aws_iam_role.eks_cluster_role.arn

  vpc_config {
    subnet_ids              = aws_subnet.public[*].id
    endpoint_public_access  = true
    endpoint_private_access = true
  }

  enabled_cluster_log_types = ["api", "audit", "authenticator"]

  depends_on = [
    aws_iam_role_policy_attachment.eks_cluster_policy
  ]
}
7. fargate პროფილის შექმნა
resource "aws_iam_role" "eks_pod_execution_role" {
  name = "eks-pod-execution-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = {
        Service = "eks-fargate-pods.amazonaws.com"
      }
      Action = "sts:AssumeRole"
    }]
  })
}
იქმნება iam როლი რომელიც გამოიყენება fargate-ზე გაშვებული პოდებისთვი
იქმნება fargate profile, რომელიც საშუალებას აძლევს kubernetes პოდებს, გაშვდნენ fargate-ზე
ნეიმსფეისი >>> test-app  მხოლოდ ამ ნეიმსფეისში გაშვებული პოდები იმუშავებენ fargate-ზე
resource "aws_eks_fargate_profile" "fargate_profile" {
  cluster_name           = aws_eks_cluster.eks.name
  fargate_profile_name   = "fargate-profile"
  pod_execution_role_arn = aws_iam_role.eks_pod_execution_role.arn
  subnet_ids             = aws_subnet.private[*].id

  selector {
    namespace = "test-app"
  }

  depends_on = [
    aws_iam_role_policy_attachment.eks_fargate_policy
  ]
}
8.  outputs
output "cluster_name" {
  value = aws_eks_cluster.eks.name
}

output "cluster_endpoint" {
  value = aws_eks_cluster.eks.endpoint
}

output "fargate_profile_name" {
  value = aws_eks_fargate_profile.fargate_profile.fargate_profile_name
}
ამ სკრიპტის გაშვების შემდეგ, გამოსავალად მიიღებთ:

eks კლასტერის სახელს

eks api endpoints-ს

fargate profile-ს სახელს
