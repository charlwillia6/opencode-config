---
name: cloud-deploy-aws-azure-gcp
description: Cloud deployment best practices for AWS, Azure, and GCP. Includes Terraform patterns, security, and modern infrastructure as code practices for 2026.
---

# Cloud Infrastructure: AWS + Azure + GCP (2026)

## Technology Stack

| Cloud Provider | Infrastructure as Code | Container Runtime |
|---------------|------------------------|-------------------|
| **AWS** | Terraform / AWS CDK | ECS / EKS |
| **Azure** | Terraform / Azure ARM | AKS / Container Instances |
| **GCP** | Terraform / Pulumi | GKE / Cloud Run |

## Terraform Architecture

### Project Structure
```
infrastructure/
├── main.tf
├── variables.tf
├── outputs.tf
├── terraform.tfvars.example
├── providers.tf
├── backend.tf
├── modules/
│   ├── vpc/
│   ├── kubernetes/
│   ├── database/
│   └── storage/
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   └── terraform.tfvars
│   ├── staging/
│   └── production/
└── scripts/
    ├── deploy.sh
    └── destroy.sh
```

### Provider Configuration
```hcl
# providers.tf
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }

  backend "s3" {
    bucket         = "myapp-terraform-state"
    key            = "terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}

# AWS Provider
provider "aws" {
  region = var.aws_region
  default_tags {
    tags = {
      Project     = var.project_name
      Environment = var.environment
      ManagedBy   = "Terraform"
    }
  }
}

# Azure Provider
provider "azurerm" {
  features {}
  resource_provider_registration = true
}

# GCP Provider
provider "google" {
  project = var.gcp_project_id
  region  = var.gcp_region
}
```

### VPC Module
```hcl
# modules/vpc/main.tf
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "${var.project_name}-vpc"
  }
}

resource "aws_subnet" "public" {
  count                   = length(var.availability_zones)
  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone       = var.availability_zones[count.index]
  map_public_ip_on_launch = true

  tags = {
    Name        = "${var.project_name}-public-${count.index + 1}"
    Type        = "public"
    Environment = var.environment
  }
}

resource "aws_subnet" "private" {
  count             = length(var.availability_zones)
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index + 10)
  availability_zone = var.availability_zones[count.index]

  tags = {
    Name        = "${var.project_name}-private-${count.index + 1}"
    Type        = "private"
    Environment = var.environment
  }
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "${var.project_name}-igw"
  }
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = {
    Name        = "${var.project_name}-public-rt"
    Type        = "public"
    Environment = var.environment
  }
}

resource "aws_route_table_association" "public" {
  count          = length(var.availability_zones)
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

output "vpc_id" {
  value = aws_vpc.main.id
}

output "public_subnet_ids" {
  value = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  value = aws_subnet.private[*].id
}
```

### Kubernetes Cluster (EKS/AKS/GKE)
```hcl
# modules/kubernetes/main.tf
resource "aws_eks_cluster" "main" {
  name     = var.cluster_name
  role_arn = aws_iam_role.cluster_role.arn
  version  = var.kubernetes_version

  vpc_config {
    subnet_ids              = var.subnet_ids
    endpoint_private_access = true
    endpoint_public_access  = true
    public_access_cidrs     = ["0.0.0.0/0"]
  }

  tags = {
    Name = "${var.project_name}-eks"
  }
}

resource "aws_eks_node_group" "main" {
  cluster_name    = aws_eks_cluster.main.name
  node_group_name = "${var.project_name}-nodes"
  node_role_arn   = aws_iam_role.node_group_role.arn
  subnet_ids      = var.subnet_ids

  scaling_config {
    desired_size = var.desired_capacity
    max_size     = var.max_size
    min_size     = var.min_size
  }

  update_config {
    max_unavailable = 1
  }

  tags = {
    Name = "${var.project_name}-eks-nodes"
  }
}

output "cluster_endpoint" {
  value = aws_eks_cluster.main.endpoint
}

output "cluster_certificate_authority_data" {
  value = aws_eks_cluster.main.certificate_authority[0].data
}

output "cluster_security_group_id" {
  value = aws_eks_cluster.main.vpc_config[0].security_group_ids[0]
}
```

### Database (RDS/Azure SQL/Cloud SQL)
```hcl
# modules/database/main.tf
resource "aws_db_instance" "main" {
  identifier           = var.database_name
  instance_class       = var.instance_class
  allocated_storage    = var.allocated_storage
  
  engine               = "postgres"
  engine_version       = "16.4"
  database_name        = var.database_name
  username             = var.username
  password             = var.password
  
  vpc_security_group_ids = [var.security_group_id]
  db_subnet_group_name   = var.subnet_group_name
  parameter_group_name   = var.parameter_group_name
  
  backup_retention_period = 7
  backup_window           = "03:00-04:00"
  maintenance_window      = "Mon:04:00-Mon:05:00"
  
  deletion_protection     = var.deletion_protection
  skip_final_snapshot     = var.environment == "dev"
  
  storage_type           = "gp3"
  storage_encrypted      = true
  copy_tags_to_snapshot  = true
  
  tags = {
    Name = "${var.project_name}-db"
  }
}

output "endpoint" {
  value = aws_db_instance.main.endpoint
}

output "port" {
  value = aws_db_instance.main.port
}
```

## Container Deployment

### Docker Multi-Stage Build
```dockerfile
# Dockerfile
FROM node:22-alpine AS builder

WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

COPY . .
RUN npm run build

FROM nginx:alpine

COPY --from=builder /app/dist /usr/share/nginx/html
COPY --from=builder /app/nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### Kubernetes Deployment
```yaml
# kubernetes/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  labels:
    app: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myregistry/myapp:latest
        ports:
        - containerPort: 80
        env:
        - name: NODE_ENV
          value: "production"
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 80
  type: LoadBalancer
```

## Security Best Practices

### Secrets Management
```hcl
# Using AWS Secrets Manager
data "aws_secretsmanager_secret_version" "db_credentials" {
  secret_id = aws_secretsmanager_secret.db.name
}

locals {
  db_username = jsondecode(data.aws_secretsmanager_secret_version.db_credentials.secret_string)["username"]
  db_password = jsondecode(data.aws_secretsmanager_secret_version.db_credentials.secret_string)["password"]
}
```

### Network Security
```hcl
# Security group with least privilege access
resource "aws_security_group" "web" {
  name        = "${var.project_name}-web-sg"
  description = "Allow HTTPS inbound traffic"
  vpc_id      = var.vpc_id

  ingress {
    description = "HTTPS from internet"
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

  tags = {
    Name = "${var.project_name}-web-sg"
  }
}
```

### RBAC Configuration
```yaml
# kubernetes/rbac.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: myapp-viewer
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: view
subjects:
- kind: ServiceAccount
  name: myapp
  namespace: default
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp
  namespace: default
```

## CI/CD Pipeline

### GitHub Actions Workflow
```yaml
# .github/workflows/deploy.yml
name: Deploy to Cloud

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
    
    - name: Set up Node.js
      uses: actions/setup-node@v4
      with:
        node-version: '22'
    
    - name: Install dependencies
      run:npm ci
    
    - name: Build application
      run: npm run build
    
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3
    
    - name: Log in to Container Registry
      uses: docker/login-action@v3
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ github.actor }}
        password: ${{ secrets.REGISTRY_TOKEN }}
    
    - name: Build and push Docker image
      uses: docker/build-push-action@v5
      with:
        context: .
        push: true
        tags: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
    
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ${{ vars.AWS_REGION }}
    
    - name: Update Kubernetes deployment
      run: |
        # Update deployment with new image
        kubectl set image deployment/myapp myapp=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
        
        # Wait for rollout
        kubectl rollout status deployment/myapp
      env:
        KUBECONFIG: ${{ secrets.KUBECONFIG }}
```

## Cloud Provider Specific Features

### AWS S3 Static Website
```hcl
resource "aws_s3_bucket" "website" {
  bucket = var.bucket_name
  
  tags = {
    Name = "${var.project_name}-website"
  }
}

resource "aws_s3_bucket_public_access_block" "website" {
  bucket = aws_s3_bucket.website.id

  block_public_acls       = false
  block_public_policy     = false
  ignore_public_acls      = false
  restrict_public_buckets = false
}

resource "aws_s3_bucket_configuration" "website" {
  bucket = aws_s3_bucket.website.id
  
  website {
    index_document = "index.html"
    error_document = "error.html"
  }
}
```

### Azure App Service
```hcl
resource "azurerm_service_plan" "main" {
  name                = "${var.project_name}-plan"
  location            = var.location
  resource_group_name = var.resource_group_name
  os_type             = "Linux"
  sku_name            = "P1v2"
}

resource "azurerm_linux_web_app" "main" {
  name                = var.app_name
  location            = var.location
  resource_group_name = var.resource_group_name
  service_plan_id     = azurerm_service_plan.main.id

  site_config {
    application_stack {
      docker_image_name = "${var.container_registry}.azurecr.io/myapp:latest"
    }
  }
}
```

### GCP Cloud Run
```hcl
resource "google_cloud_run_service" "main" {
  name     = var.service_name
  location = var.region

  template {
    spec {
      containers {
        image = var.image_url
        ports {
          container_port = 8080
        }
      }
    }
  }

  traffic {
    percent         = 100
    latest_revision = true
  }
}
```

## Environment Management

### Different Environments
```hcl
# environments/development/main.tf
module "vpc" {
  source                   = "../../modules/vpc"
  project_name             = var.project_name
  environment              = "development"
  vpc_cidr                 = "10.0.0.0/16"
  availability_zones       = ["us-east-1a", "us-east-1b"]
}

module "database" {
  source              = "../../modules/database"
  project_name        = var.project_name
  environment         = "development"
  database_name       = "myapp_dev"
  instance_class      = "db.t4g.micro"
  allocated_storage   = 20
  deletion_protection = false
}
```

### Terraform Variables
```hcl
# variables.tf
variable "project_name" {
  description = "Name of the project"
  type        = string
}

variable "environment" {
  description = "Environment (dev, staging, production)"
  type        = string
  validation {
    condition     = contains(["dev", "staging", "production"], var.environment)
    error_message = "Environment must be dev, staging, or production."
  }
}

variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

variable "kubernetes_version" {
  description = "Kubernetes version"
  type        = string
  default     = "1.29"
}
```

## Common Patterns & Anti-Patterns

### Do
✅ Use modules for reusable infrastructure components  
✅ Implement state locking with DynamoDB backend  
✅ Always enable encryption for databases and storage  
✅ Use least privilege IAM roles  
✅ Implement secrets management for credentials  
✅ Tag all resources for organization and billing  
✅ Use Terraform plan before apply in CI/CD  

### Don't
❌ Store secrets in Terraform variables  
❌ Hardcode IPs or CIDR blocks  
❌ Skip state locking in production  
❌ Mix environments in same state file  
❌ Forget to clean up resources after development  
❌ Use random resource names in production  

## references

* [Terraform Documentation](https://developer.hashicorp.com/terraform/docs)
* [AWS Documentation](https://docs.aws.amazon.com)
* [Azure Documentation](https://learn.microsoft.com/azure)
* [GCP Documentation](https://cloud.google.com/docs)

## 2026 Best Practices

* **Terraform 1.8+**: Use for improved plugin management
* **HCL2 Best Practices**: Use modern HCL syntax
* **IaC Security**: Scan infrastructure code for vulnerabilities
* **GitOps**: Use Git workflows for infrastructure changes
* **Cloud-Native**: Leverage managed services where possible
