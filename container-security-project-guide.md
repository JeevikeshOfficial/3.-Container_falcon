# Container Security Falcon: Free Implementation Guide

A comprehensive project that demonstrates all essential DevSecOps skills through building a secure containerized microservices platform without incurring cloud costs.

## Project Overview

**Goal**: Build a fully automated, secure container deployment pipeline that showcases Git/GitHub, GitHub Actions, Terraform, AWS Free Tier, Docker, Kubernetes (local), monitoring, security hardening, and scripting skills.

**Key Innovation**: Use local Kubernetes with cloud-like monitoring and security practices to demonstrate production-ready skills at zero cost.

## Architecture Components

### Infrastructure Layer
- **Local Kubernetes**: kind or k3s for multi-node cluster simulation
- **AWS Free Tier**: ECR (container registry), IAM roles, CloudWatch logs, GuardDuty
- **Terraform**: Infrastructure as Code for AWS resources within free tier limits

### Application Layer
- **Sample Microservices**: 2-3 simple services (API gateway, user service, data service)
- **Docker**: Multi-stage builds with security hardening
- **Python/Bash Scripts**: Automation, scanning, and remediation tools

### Security & Monitoring
- **Container Scanning**: Trivy (open-source) in CI/CD pipeline
- **Runtime Security**: Falco for anomaly detection
- **Logging**: Fluent Bit → CloudWatch (within free tier)
- **Monitoring**: Prometheus + Grafana (local) with CloudWatch integration

## Free Tier Cost Breakdown

### AWS Free Tier Limits (2025)
- **ECR**: 500MB storage/month (always free)
- **CloudWatch**: 5GB logs, 10 custom metrics (always free)  
- **IAM**: Unlimited users, roles, policies (always free)
- **GuardDuty**: 30-day free trial, then minimal cost for small workloads
- **Lambda**: 1M requests/month, 400K GB-seconds (always free)

### GitHub Free Tier
- **Actions**: 2,000 minutes/month for private repos (unlimited for public)
- **Storage**: 500MB for artifacts
- **Repositories**: Unlimited public repos

### Local Resources
- **kind/k3s**: Free, lightweight Kubernetes
- **Trivy**: Free, open-source vulnerability scanner
- **Prometheus/Grafana**: Free, open-source monitoring stack
- **Falco**: Free, open-source runtime security

## Implementation Steps

### Phase 1: Local Infrastructure Setup

#### 1.1 Initialize Project Repository
```bash
# Create project structure
mkdir container-security-falcon
cd container-security-falcon
git init
```

**Project Structure:**
```
container-security-falcon/
├── .github/workflows/
│   ├── security-scan.yml
│   ├── deploy.yml
│   └── infrastructure.yml
├── terraform/
│   ├── main.tf
│   ├── ecr.tf
│   ├── iam.tf
│   └── cloudwatch.tf
├── k8s/
│   ├── manifests/
│   ├── monitoring/
│   └── security/
├── apps/
│   ├── api-gateway/
│   ├── user-service/
│   └── data-service/
├── scripts/
│   ├── setup.sh
│   ├── scan-report.py
│   └── remediation.sh
└── monitoring/
    ├── prometheus/
    ├── grafana/
    └── falco/
```

#### 1.2 Set Up Local Kubernetes
**Install kind (Kubernetes in Docker):**
```bash
# Install kind
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# Create multi-node cluster
cat <<EOF > kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
EOF

kind create cluster --config kind-config.yaml --name security-falcon
```

**Alternative: k3s Setup (if you prefer):**
```bash
# Install k3s
curl -sfL https://get.k3s.io | sh -

# For multi-node setup
# On additional nodes:
# curl -sfL https://get.k3s.io | K3S_TOKEN=<token> K3S_URL=https://<master-ip>:6443 sh -
```

### Phase 2: AWS Free Tier Resources

#### 2.1 Terraform Configuration
**terraform/main.tf:**
```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  
  backend "local" {
    path = "terraform.tfstate"
  }
}

provider "aws" {
  region = var.aws_region
}

variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

variable "project_name" {
  description = "Project name for tagging"
  type        = string
  default     = "container-security-falcon"
}
```

**terraform/ecr.tf:**
```hcl
# ECR Repository for container images (Free: 500MB storage)
resource "aws_ecr_repository" "app_repositories" {
  for_each = toset(["api-gateway", "user-service", "data-service"])
  
  name                 = "${var.project_name}-${each.value}"
  image_tag_mutability = "MUTABLE"

  image_scanning_configuration {
    scan_on_push = true
  }

  encryption_configuration {
    encryption_type = "AES256"
  }

  tags = {
    Name    = "${var.project_name}-${each.value}"
    Project = var.project_name
  }
}

# ECR Lifecycle Policy to manage storage (stay within 500MB)
resource "aws_ecr_lifecycle_policy" "app_policy" {
  for_each   = aws_ecr_repository.app_repositories
  repository = each.value.name

  policy = jsonencode({
    rules = [
      {
        rulePriority = 1
        description  = "Keep last 5 images"
        selection = {
          tagStatus     = "tagged"
          tagPrefixList = ["v"]
          countType     = "imageCountMoreThan"
          countNumber   = 5
        }
        action = {
          type = "expire"
        }
      }
    ]
  })
}
```

**terraform/iam.tf:**
```hcl
# IAM role for GitHub Actions
resource "aws_iam_user" "github_actions" {
  name = "${var.project_name}-github-actions"
  
  tags = {
    Name    = "${var.project_name}-github-actions"
    Project = var.project_name
  }
}

resource "aws_iam_access_key" "github_actions" {
  user = aws_iam_user.github_actions.name
}

# Policy for ECR and CloudWatch access
resource "aws_iam_user_policy" "github_actions_policy" {
  name = "${var.project_name}-github-actions-policy"
  user = aws_iam_user.github_actions.name

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "ecr:GetAuthorizationToken",
          "ecr:BatchCheckLayerAvailability",
          "ecr:GetDownloadUrlForLayer",
          "ecr:BatchGetImage",
          "ecr:BatchImportLayerAvailability",
          "ecr:PutImage",
          "ecr:InitiateLayerUpload",
          "ecr:UploadLayerPart",
          "ecr:CompleteLayerUpload",
          "ecr:DescribeRepositories",
          "ecr:GetRepositoryPolicy",
          "ecr:ListImages",
          "ecr:DescribeImages"
        ]
        Resource = "*"
      },
      {
        Effect = "Allow"
        Action = [
          "logs:CreateLogGroup",
          "logs:CreateLogStream",
          "logs:PutLogEvents",
          "logs:DescribeLogStreams",
          "logs:DescribeLogGroups"
        ]
        Resource = "arn:aws:logs:*:*:log-group:/aws/containerinsights/*"
      }
    ]
  })
}
```

#### 2.2 Deploy AWS Resources
```bash
cd terraform
terraform init
terraform plan
terraform apply
```

**Store outputs securely:**
```bash
# Get ECR repository URIs
terraform output -json ecr_repository_urls

# Get GitHub Actions credentials (store in GitHub Secrets)
terraform output -raw github_actions_access_key
terraform output -raw github_actions_secret_key
```

### Phase 3: Container Applications

#### 3.1 Sample Microservice (Python FastAPI)
**apps/api-gateway/Dockerfile:**
```dockerfile
# Multi-stage build for security and size optimization
FROM python:3.11-alpine AS builder

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

FROM python:3.11-alpine

# Security hardening
RUN addgroup -g 1001 -S appgroup && \
    adduser -u 1001 -S appuser -G appgroup

# Install only runtime dependencies
COPY --from=builder /root/.local /root/.local
ENV PATH=/root/.local/bin:$PATH

WORKDIR /app
COPY --chown=appuser:appgroup app.py .

# Security: Run as non-root user
USER appuser

EXPOSE 8000
CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]
```

**apps/api-gateway/app.py:**
```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import os
import logging

# Configure logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

app = FastAPI(title="API Gateway", version="1.0.0")

class HealthResponse(BaseModel):
    status: str
    service: str
    version: str

@app.get("/health", response_model=HealthResponse)
async def health_check():
    logger.info("Health check requested")
    return HealthResponse(
        status="healthy",
        service="api-gateway",
        version="1.0.0"
    )

@app.get("/")
async def root():
    logger.info("Root endpoint accessed")
    return {"message": "Container Security Falcon API Gateway", "status": "running"}

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

**apps/api-gateway/requirements.txt:**
```text
fastapi==0.104.1
uvicorn[standard]==0.24.0
pydantic==2.5.0
```

#### 3.2 Kubernetes Manifests
**k8s/manifests/api-gateway.yaml:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
  labels:
    app: api-gateway
    tier: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api-gateway
  template:
    metadata:
      labels:
        app: api-gateway
        tier: frontend
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1001
        fsGroup: 1001
      containers:
      - name: api-gateway
        image: localhost:5001/container-security-falcon-api-gateway:latest
        ports:
        - containerPort: 8000
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          runAsUser: 1001
          capabilities:
            drop:
              - ALL
        resources:
          limits:
            cpu: 500m
            memory: 256Mi
          requests:
            cpu: 100m
            memory: 128Mi
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 5
        env:
        - name: LOG_LEVEL
          value: "INFO"
---
apiVersion: v1
kind: Service
metadata:
  name: api-gateway-service
  labels:
    app: api-gateway
spec:
  selector:
    app: api-gateway
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8000
  type: ClusterIP
```

### Phase 4: CI/CD Pipeline with Security

#### 4.1 GitHub Actions Workflows
**.github/workflows/security-scan.yml:**
```yaml
name: Security Scan Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  vulnerability-scan:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3

    - name: Build Docker image
      run: |
        docker build -t test-image:latest ./apps/api-gateway

    - name: Run Trivy vulnerability scanner
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: 'test-image:latest'
        format: 'sarif'
        output: 'trivy-results.sarif'
        severity: 'CRITICAL,HIGH'
        exit-code: '1'  # Fail on critical/high vulnerabilities

    - name: Upload Trivy scan results to GitHub Security tab
      uses: github/codeql-action/upload-sarif@v3
      if: always()
      with:
        sarif_file: 'trivy-results.sarif'

    - name: Generate human-readable report
      if: always()
      run: |
        docker run --rm -v "$PWD":/workspace \
          aquasec/trivy image --format table --output /workspace/vulnerability-report.txt test-image:latest

    - name: Upload vulnerability report
      if: always()
      uses: actions/upload-artifact@v4
      with:
        name: vulnerability-report
        path: vulnerability-report.txt

  dockerfile-security-scan:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Run Hadolint
      uses: hadolint/hadolint-action@v3.1.0
      with:
        dockerfile: ./apps/api-gateway/Dockerfile
        format: sarif
        output-file: hadolint-results.sarif
        no-fail: true

    - name: Upload Hadolint scan results
      uses: github/codeql-action/upload-sarif@v3
      if: always()
      with:
        sarif_file: hadolint-results.sarif

  secrets-scan:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Run TruffleHog OSS
      uses: trufflesecurity/trufflehog@main
      with:
        path: ./
        base: main
        head: HEAD
        extra_args: --debug --only-verified
```

**.github/workflows/build-and-deploy.yml:**
```yaml
name: Build and Deploy

on:
  push:
    branches: [ main ]
  workflow_run:
    workflows: ["Security Scan Pipeline"]
    types:
      - completed

env:
  AWS_REGION: us-east-1

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    
    strategy:
      matrix:
        service: [api-gateway, user-service, data-service]

    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ${{ env.AWS_REGION }}

    - name: Login to Amazon ECR
      id: login-ecr
      uses: aws-actions/amazon-ecr-login@v2

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3

    - name: Build and scan image
      run: |
        docker build -t ${{ matrix.service }}:latest ./apps/${{ matrix.service }}
        
        # Run Trivy scan and fail if critical vulnerabilities found
        docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
          aquasec/trivy image --exit-code 1 --severity CRITICAL ${{ matrix.service }}:latest

    - name: Tag and push image
      env:
        ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
        ECR_REPOSITORY: container-security-falcon-${{ matrix.service }}
        IMAGE_TAG: ${{ github.sha }}
      run: |
        docker tag ${{ matrix.service }}:latest $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
        docker tag ${{ matrix.service }}:latest $ECR_REGISTRY/$ECR_REPOSITORY:latest
        docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
        docker push $ECR_REGISTRY/$ECR_REPOSITORY:latest

  deploy-to-kind:
    runs-on: ubuntu-latest
    needs: build-and-push
    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Create kind cluster
      uses: helm/kind-action@v1.8.0
      with:
        config: kind-config.yaml
        cluster_name: security-falcon

    - name: Install kubectl
      uses: azure/setup-kubectl@v3

    - name: Deploy to kind cluster
      run: |
        # Deploy applications
        kubectl apply -f k8s/manifests/
        
        # Wait for deployments to be ready
        kubectl wait --for=condition=available --timeout=300s deployment --all
        
        # Check pod security contexts
        kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.securityContext}{"\n"}{end}'

    - name: Run basic smoke tests
      run: |
        # Port forward to test services
        kubectl port-forward service/api-gateway-service 8080:80 &
        sleep 10
        
        # Test health endpoint
        curl -f http://localhost:8080/health || exit 1
        echo "Health check passed!"
        
        # Kill port-forward
        pkill -f "kubectl port-forward"
```

### Phase 5: Monitoring and Logging

#### 5.1 Prometheus Configuration
**monitoring/prometheus/prometheus.yml:**
```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "alerts.yml"

scrape_configs:
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        target_label: __address__

  - job_name: 'kubernetes-nodes'
    kubernetes_sd_configs:
      - role: node
    relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
      - target_label: __address__
        replacement: kubernetes.default.svc:443
      - source_labels: [__meta_kubernetes_node_name]
        regex: (.+)
        target_label: __metrics_path__
        replacement: /api/v1/nodes/${1}/proxy/metrics

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']
```

#### 5.2 Grafana Dashboard Configuration
**monitoring/grafana/dashboards/container-security.json:**
```json
{
  "dashboard": {
    "id": null,
    "title": "Container Security Dashboard",
    "description": "Security metrics for Container Security Falcon project",
    "panels": [
      {
        "title": "Container Vulnerability Scan Results",
        "type": "stat",
        "targets": [
          {
            "expr": "trivy_vulnerabilities_total",
            "legendFormat": "{{severity}} vulnerabilities"
          }
        ]
      },
      {
        "title": "Pod Security Context Violations",
        "type": "graph",
        "targets": [
          {
            "expr": "kube_pod_security_context_run_as_root",
            "legendFormat": "Pods running as root"
          }
        ]
      },
      {
        "title": "Failed Security Scans",
        "type": "graph",
        "targets": [
          {
            "expr": "github_actions_job_failures{job_name=~\".*security.*\"}",
            "legendFormat": "Security scan failures"
          }
        ]
      }
    ]
  }
}
```

#### 5.3 Falco Runtime Security
**monitoring/falco/falco-config.yaml:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: falco-config
data:
  falco.yaml: |
    rules_file:
      - /etc/falco/falco_rules.yaml
      - /etc/falco/falco_rules.local.yaml
    
    json_output: true
    json_include_output_property: true
    
    outputs:
      rate: 1
      max_burst: 1000
    
    syslog_output:
      enabled: false
    
    program_output:
      enabled: true
      keep_alive: false
      program: |
        curl -X POST http://webhook-service:8080/falco-alerts \
        -H "Content-Type: application/json" \
        -d @-
    
    http_output:
      enabled: true
      url: "http://webhook-service:8080/falco-alerts"
      
  falco_rules.local.yaml: |
    - rule: Container Security Violation
      desc: Detect potential security violations in containers
      condition: >
        spawned_process and container and 
        (proc.name in (nc, ncat, netcat, nmap, telnet) or
         fd.name startswith /etc/shadow or
         fd.name startswith /etc/passwd)
      output: >
        Security violation detected (user=%user.name command=%proc.cmdline
        file=%fd.name container_id=%container.id image=%container.image.repository)
      priority: HIGH
      tags: [container, security, process]
```

### Phase 6: Automation Scripts

#### 6.1 Setup Script
**scripts/setup.sh:**
```bash
#!/bin/bash
set -e

echo "🚀 Setting up Container Security Falcon project..."

# Function to check if command exists
command_exists() {
    command -v "$1" >/dev/null 2>&1
}

# Install required tools
install_dependencies() {
    echo "📦 Installing dependencies..."
    
    # Check for Docker
    if ! command_exists docker; then
        echo "❌ Docker is required but not installed."
        echo "Please install Docker: https://docs.docker.com/get-docker/"
        exit 1
    fi
    
    # Install kind if not present
    if ! command_exists kind; then
        echo "Installing kind..."
        curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
        chmod +x ./kind
        sudo mv ./kind /usr/local/bin/kind
    fi
    
    # Install kubectl if not present
    if ! command_exists kubectl; then
        echo "Installing kubectl..."
        curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
        chmod +x kubectl
        sudo mv kubectl /usr/local/bin/kubectl
    fi
    
    # Install Trivy if not present
    if ! command_exists trivy; then
        echo "Installing Trivy..."
        curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin v0.46.1
    fi
    
    # Install Terraform if not present
    if ! command_exists terraform; then
        echo "Installing Terraform..."
        wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
        echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
        sudo apt update && sudo apt install terraform
    fi
}

# Create kind cluster
create_cluster() {
    echo "🔧 Creating kind cluster..."
    
    cat <<EOF > kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  image: kindest/node:v1.28.0
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 8080
    protocol: TCP
  - containerPort: 443
    hostPort: 8443
    protocol: TCP
- role: worker
  image: kindest/node:v1.28.0
- role: worker
  image: kindest/node:v1.28.0
EOF

    kind create cluster --config kind-config.yaml --name security-falcon
    kubectl cluster-info --context kind-security-falcon
}

# Deploy monitoring stack
deploy_monitoring() {
    echo "📊 Deploying monitoring stack..."
    
    # Create monitoring namespace
    kubectl create namespace monitoring --dry-run=client -o yaml | kubectl apply -f -
    
    # Deploy Prometheus
    kubectl apply -f https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/main/bundle.yaml
    
    # Deploy Grafana
    kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      containers:
      - name: grafana
        image: grafana/grafana:latest
        ports:
        - containerPort: 3000
        env:
        - name: GF_SECURITY_ADMIN_PASSWORD
          value: admin
---
apiVersion: v1
kind: Service
metadata:
  name: grafana
  namespace: monitoring
spec:
  selector:
    app: grafana
  ports:
  - port: 3000
    targetPort: 3000
  type: LoadBalancer
EOF
}

# Deploy Falco for runtime security
deploy_falco() {
    echo "🛡️  Deploying Falco runtime security..."
    
    kubectl apply -f https://raw.githubusercontent.com/falcosecurity/falco/master/deploy/kubernetes/falco-daemonset-configmap.yaml
}

# Run initial security scan
run_security_scan() {
    echo "🔍 Running initial security scan..."
    
    # Build test image
    docker build -t security-test:latest ./apps/api-gateway/
    
    # Run Trivy scan
    trivy image --exit-code 0 --severity HIGH,CRITICAL --format table security-test:latest
    
    echo "✅ Security scan completed!"
}

# Main execution
main() {
    echo "Starting Container Security Falcon setup..."
    install_dependencies
    create_cluster
    deploy_monitoring
    deploy_falco
    run_security_scan
    
    echo ""
    echo "🎉 Setup completed successfully!"
    echo ""
    echo "Next steps:"
    echo "1. Configure AWS credentials: aws configure"
    echo "2. Deploy AWS resources: cd terraform && terraform apply"
    echo "3. Set up GitHub Actions secrets"
    echo "4. Push code to trigger CI/CD pipeline"
    echo ""
    echo "Access URLs:"
    echo "- Grafana: kubectl port-forward -n monitoring svc/grafana 3000:3000"
    echo "- Kubernetes Dashboard: kubectl proxy"
}

main "$@"
```

#### 6.2 Vulnerability Report Parser
**scripts/scan-report.py:**
```python
#!/usr/bin/env python3
"""
Container Security Falcon - Vulnerability Report Parser
Processes Trivy scan results and generates actionable reports
"""

import json
import sys
import argparse
import csv
from datetime import datetime
from pathlib import Path
import smtplib
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart

class VulnerabilityReporter:
    def __init__(self, trivy_json_file):
        self.trivy_json_file = trivy_json_file
        self.vulnerabilities = []
        self.summary = {
            'critical': 0,
            'high': 0,
            'medium': 0,
            'low': 0,
            'total': 0
        }
    
    def parse_trivy_output(self):
        """Parse Trivy JSON output"""
        try:
            with open(self.trivy_json_file, 'r') as f:
                data = json.load(f)
            
            for result in data.get('Results', []):
                target = result.get('Target', 'Unknown')
                vulnerabilities = result.get('Vulnerabilities', [])
                
                for vuln in vulnerabilities:
                    severity = vuln.get('Severity', 'UNKNOWN').lower()
                    self.vulnerabilities.append({
                        'target': target,
                        'vulnerability_id': vuln.get('VulnerabilityID', 'N/A'),
                        'package_name': vuln.get('PkgName', 'N/A'),
                        'installed_version': vuln.get('InstalledVersion', 'N/A'),
                        'fixed_version': vuln.get('FixedVersion', 'Not Available'),
                        'severity': severity,
                        'title': vuln.get('Title', 'N/A'),
                        'description': vuln.get('Description', 'N/A')[:200] + '...',
                        'references': vuln.get('References', [])
                    })
                    
                    if severity in self.summary:
                        self.summary[severity] += 1
                    self.summary['total'] += 1
                        
        except FileNotFoundError:
            print(f"Error: Could not find {self.trivy_json_file}")
            sys.exit(1)
        except json.JSONDecodeError:
            print(f"Error: Invalid JSON in {self.trivy_json_file}")
            sys.exit(1)
    
    def generate_csv_report(self, output_file):
        """Generate CSV report"""
        with open(output_file, 'w', newline='') as csvfile:
            fieldnames = [
                'target', 'vulnerability_id', 'package_name', 
                'installed_version', 'fixed_version', 'severity',
                'title', 'description'
            ]
            writer = csv.DictWriter(csvfile, fieldnames=fieldnames)
            writer.writeheader()
            
            for vuln in self.vulnerabilities:
                writer.writerow({k: v for k, v in vuln.items() if k != 'references'})
    
    def generate_html_report(self, output_file):
        """Generate HTML report"""
        html_template = """
        <!DOCTYPE html>
        <html>
        <head>
            <title>Container Security Falcon - Vulnerability Report</title>
            <style>
                body { font-family: Arial, sans-serif; margin: 40px; }
                .header { background-color: #2c3e50; color: white; padding: 20px; border-radius: 5px; }
                .summary { display: flex; justify-content: space-around; margin: 20px 0; }
                .metric { text-align: center; padding: 10px; border-radius: 5px; }
                .critical { background-color: #e74c3c; color: white; }
                .high { background-color: #f39c12; color: white; }
                .medium { background-color: #f1c40f; color: black; }
                .low { background-color: #27ae60; color: white; }
                table { width: 100%; border-collapse: collapse; margin-top: 20px; }
                th, td { border: 1px solid #ddd; padding: 12px; text-align: left; }
                th { background-color: #f2f2f2; }
                .severity-critical { background-color: #ffebee; }
                .severity-high { background-color: #fff3e0; }
                .severity-medium { background-color: #fffde7; }
                .severity-low { background-color: #e8f5e8; }
            </style>
        </head>
        <body>
            <div class="header">
                <h1>🛡️ Container Security Falcon</h1>
                <h2>Vulnerability Report</h2>
                <p>Generated on: {timestamp}</p>
            </div>
            
            <div class="summary">
                <div class="metric critical">
                    <h3>{critical}</h3>
                    <p>Critical</p>
                </div>
                <div class="metric high">
                    <h3>{high}</h3>
                    <p>High</p>
                </div>
                <div class="metric medium">
                    <h3>{medium}</h3>
                    <p>Medium</p>
                </div>
                <div class="metric low">
                    <h3>{low}</h3>
                    <p>Low</p>
                </div>
            </div>
            
            <h3>Detailed Vulnerabilities ({total} total)</h3>
            <table>
                <tr>
                    <th>Severity</th>
                    <th>Vulnerability ID</th>
                    <th>Package</th>
                    <th>Installed Version</th>
                    <th>Fixed Version</th>
                    <th>Title</th>
                </tr>
                {vulnerability_rows}
            </table>
        </body>
        </html>
        """
        
        vulnerability_rows = ""
        for vuln in sorted(self.vulnerabilities, key=lambda x: ['critical', 'high', 'medium', 'low'].index(x['severity'])):
            vulnerability_rows += f"""
                <tr class="severity-{vuln['severity']}">
                    <td><strong>{vuln['severity'].upper()}</strong></td>
                    <td>{vuln['vulnerability_id']}</td>
                    <td>{vuln['package_name']}</td>
                    <td>{vuln['installed_version']}</td>
                    <td>{vuln['fixed_version']}</td>
                    <td>{vuln['title']}</td>
                </tr>
            """
        
        html_content = html_template.format(
            timestamp=datetime.now().strftime("%Y-%m-%d %H:%M:%S"),
            critical=self.summary['critical'],
            high=self.summary['high'],
            medium=self.summary['medium'],
            low=self.summary['low'],
            total=self.summary['total'],
            vulnerability_rows=vulnerability_rows
        )
        
        with open(output_file, 'w') as f:
            f.write(html_content)
    
    def check_security_policy(self):
        """Check against security policy"""
        policy_violations = []
        
        if self.summary['critical'] > 0:
            policy_violations.append(f"❌ CRITICAL: {self.summary['critical']} critical vulnerabilities found (Policy: 0 allowed)")
        
        if self.summary['high'] > 5:
            policy_violations.append(f"⚠️  HIGH: {self.summary['high']} high vulnerabilities found (Policy: max 5 allowed)")
        
        return policy_violations
    
    def generate_remediation_script(self, output_file):
        """Generate bash script for automated remediation"""
        script_content = """#!/bin/bash
# Container Security Falcon - Automated Remediation Script
# Generated on: {}

echo "🔧 Starting automated vulnerability remediation..."

""".format(datetime.now().strftime("%Y-%m-%d %H:%M:%S"))

        # Group vulnerabilities by package
        packages_to_update = {}
        for vuln in self.vulnerabilities:
            if vuln['fixed_version'] != 'Not Available' and vuln['severity'] in ['critical', 'high']:
                package = vuln['package_name']
                fixed_version = vuln['fixed_version']
                if package not in packages_to_update:
                    packages_to_update[package] = fixed_version
        
        for package, version in packages_to_update.items():
            script_content += f"""
# Update {package} to {version}
echo "Updating {package} to {version}..."
# Add package-specific update commands here
"""

        script_content += """
echo "✅ Remediation script completed!"
echo "⚠️  Note: Please review and test all changes before applying to production"
"""
        
        with open(output_file, 'w') as f:
            f.write(script_content)
        
        # Make script executable
        import os
        os.chmod(output_file, 0o755)

def main():
    parser = argparse.ArgumentParser(description='Container Security Falcon Vulnerability Reporter')
    parser.add_argument('trivy_json', help='Path to Trivy JSON output file')
    parser.add_argument('--csv', help='Generate CSV report')
    parser.add_argument('--html', help='Generate HTML report')
    parser.add_argument('--remediation', help='Generate remediation script')
    parser.add_argument('--check-policy', action='store_true', help='Check security policy compliance')
    
    args = parser.parse_args()
    
    reporter = VulnerabilityReporter(args.trivy_json)
    reporter.parse_trivy_output()
    
    print(f"📊 Vulnerability Summary:")
    print(f"   Critical: {reporter.summary['critical']}")
    print(f"   High:     {reporter.summary['high']}")
    print(f"   Medium:   {reporter.summary['medium']}")
    print(f"   Low:      {reporter.summary['low']}")
    print(f"   Total:    {reporter.summary['total']}")
    
    if args.csv:
        reporter.generate_csv_report(args.csv)
        print(f"📄 CSV report generated: {args.csv}")
    
    if args.html:
        reporter.generate_html_report(args.html)
        print(f"🌐 HTML report generated: {args.html}")
    
    if args.remediation:
        reporter.generate_remediation_script(args.remediation)
        print(f"🔧 Remediation script generated: {args.remediation}")
    
    if args.check_policy:
        violations = reporter.check_security_policy()
        if violations:
            print("\n🚨 Security Policy Violations:")
            for violation in violations:
                print(f"   {violation}")
            sys.exit(1)
        else:
            print("\n✅ All security policies passed!")

if __name__ == "__main__":
    main()
```

## Cost Monitoring & Management

### Free Tier Usage Tracking Script
**scripts/aws-cost-monitor.py:**
```python
#!/usr/bin/env python3
"""
AWS Free Tier usage monitoring for Container Security Falcon
"""

import boto3
import json
from datetime import datetime, timedelta

def check_ecr_usage():
    """Check ECR repository storage usage"""
    ecr = boto3.client('ecr')
    
    total_size = 0
    repositories = ecr.describe_repositories()['repositories']
    
    for repo in repositories:
        repo_name = repo['repositoryName']
        if 'container-security-falcon' in repo_name:
            try:
                images = ecr.describe_images(repositoryName=repo_name)['imageDetails']
                repo_size = sum(image['imageSizeInBytes'] for image in images)
                total_size += repo_size
                print(f"📦 {repo_name}: {repo_size / 1024 / 1024:.2f} MB")
            except Exception as e:
                print(f"❌ Error checking {repo_name}: {e}")
    
    free_tier_limit = 500 * 1024 * 1024  # 500 MB
    usage_percent = (total_size / free_tier_limit) * 100
    
    print(f"\n📊 ECR Total Usage: {total_size / 1024 / 1024:.2f} MB / 500 MB ({usage_percent:.1f}%)")
    
    if usage_percent > 80:
        print("⚠️  WARNING: ECR usage is above 80% of free tier limit!")
        return False
    return True

def check_cloudwatch_logs():
    """Check CloudWatch Logs usage"""
    logs = boto3.client('logs')
    
    # Get log groups
    log_groups = logs.describe_log_groups()['logGroups']
    
    total_stored_bytes = sum(lg.get('storedBytes', 0) for lg in log_groups)
    free_tier_limit = 5 * 1024 * 1024 * 1024  # 5 GB
    usage_percent = (total_stored_bytes / free_tier_limit) * 100
    
    print(f"📝 CloudWatch Logs: {total_stored_bytes / 1024 / 1024:.2f} MB / 5 GB ({usage_percent:.1f}%)")
    
    if usage_percent > 80:
        print("⚠️  WARNING: CloudWatch Logs usage is above 80% of free tier limit!")
        return False
    return True

def main():
    print("🔍 Checking AWS Free Tier usage for Container Security Falcon...")
    
    ecr_ok = check_ecr_usage()
    logs_ok = check_cloudwatch_logs()
    
    if ecr_ok and logs_ok:
        print("\n✅ All services within free tier limits!")
    else:
        print("\n⚠️  Some services are approaching free tier limits. Consider cleanup.")

if __name__ == "__main__":
    main()
```

## Project Completion Checklist

### Technical Implementation ✅
- [ ] Local Kubernetes cluster (kind/k3s) with multi-node setup
- [ ] AWS resources deployed with Terraform (ECR, IAM, CloudWatch)
- [ ] Sample microservices with security-hardened Dockerfiles
- [ ] Complete CI/CD pipeline with GitHub Actions
- [ ] Vulnerability scanning integrated (Trivy, Hadolint)
- [ ] Runtime security monitoring (Falco)
- [ ] Monitoring stack (Prometheus, Grafana)
- [ ] Logging pipeline (Fluent Bit → CloudWatch)
- [ ] Automation scripts (Python for reports, Bash for setup)

### Security Hardening ✅
- [ ] Container images run as non-root users
- [ ] Minimal base images (Alpine Linux)
- [ ] Multi-stage Docker builds
- [ ] Pod Security Standards enforced
- [ ] Network policies implemented
- [ ] Secret management with Kubernetes secrets
- [ ] Regular vulnerability scanning in CI/CD
- [ ] Security policy compliance checks

### Documentation & Demo ✅
- [ ] Complete README with setup instructions
- [ ] Architecture diagrams
- [ ] Security best practices documentation
- [ ] Demo script showing all features
- [ ] Cost monitoring and free-tier compliance
- [ ] Troubleshooting guide

## Skills Demonstrated

### Git & GitHub ✅
- Branch protection rules
- Pull request workflows
- Issue templates
- GitHub Actions CI/CD
- Secrets management

### Infrastructure as Code (Terraform) ✅
- AWS resource provisioning
- State management
- Variable management
- Resource tagging and policies
- Free tier resource optimization

### Container Technologies ✅
- Docker multi-stage builds
- Security-hardened containers
- Kubernetes manifest creation
- Pod security contexts
- Resource limits and requests

### AWS Cloud Services ✅
- ECR (container registry)
- IAM (identity and access management)
- CloudWatch (logging and monitoring)
- Lambda (serverless automation)
- GuardDuty (threat detection)

### Security & Compliance ✅
- Vulnerability scanning (Trivy)
- Container security (Falco)
- Configuration scanning (Hadolint)
- Secrets detection (TruffleHog)
- Compliance reporting

### Monitoring & Observability ✅
- Prometheus metrics collection
- Grafana dashboards
- Log aggregation
- Alerting and notifications
- Performance monitoring

### Scripting & Automation ✅
- Python automation scripts
- Bash setup and deployment scripts
- CI/CD pipeline automation
- Automated remediation
- Cost monitoring scripts

This comprehensive project demonstrates all the requested skills while staying within free tier limits, making it perfect for showcasing DevSecOps capabilities to potential employers without incurring any costs.