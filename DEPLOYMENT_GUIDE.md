# 🚀 DEPLOYMENT GUIDE
## Production-Ready Deployment cho Mọi Môi Trường

> **Mục tiêu**: Hướng dẫn chi tiết deploy backend template lên development, staging, production với best practices.

---

## 🎯 **DEPLOYMENT OVERVIEW**

### **Deployment Environments**
- 🔧 **Development**: Local development và testing
- 🧪 **Staging**: Pre-production testing environment  
- 🚀 **Production**: Live production environment
- 🔄 **DR (Disaster Recovery)**: Backup production environment

### **Deployment Strategies**
- **Blue-Green**: Zero-downtime deployment
- **Canary**: Gradual rollout with monitoring
- **Rolling**: Sequential instance updates
- **A/B Testing**: Feature flag based deployment

---

## 🔧 **DEVELOPMENT ENVIRONMENT**

### **Local Development Setup**

```bash
# 1. Clone and setup
git clone https://github.com/your-org/backend-template.git
cd backend-template
go mod tidy

# 2. Start local infrastructure
docker-compose -f docker-compose.dev.yml up -d

# 3. Run migrations
make db-migrate-up

# 4. Start application with hot reload
make dev
```

### **Docker Compose for Development**

```yaml
# docker-compose.dev.yml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: app_db
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./scripts/init.sql:/docker-entrypoint-initdb.d/init.sql

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data

  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686"
      - "14268:14268"
    environment:
      COLLECTOR_OTLP_ENABLED: true

  mailhog:
    image: mailhog/mailhog
    ports:
      - "1025:1025"
      - "8025:8025"

volumes:
  postgres_data:
  redis_data:
```

### **Development Configuration**

```yaml
# configs/config.dev.yaml
app:
  name: "backend-template-dev"
  environment: "development"
  debug: true
  log_level: "debug"

server:
  host: "0.0.0.0"
  port: 8080
  read_timeout: "30s"
  write_timeout: "30s"

database:
  host: "localhost"
  port: 5432
  name: "app_db"
  username: "postgres"
  password: "password"
  ssl_mode: "disable"
  auto_migrate: true

cache:
  redis:
    host: "localhost"
    port: 6379
    db: 0

monitoring:
  metrics:
    enabled: true
  tracing:
    enabled: true
    jaeger_endpoint: "http://localhost:14268/api/traces"
```

---

## 🧪 **STAGING ENVIRONMENT**

### **AWS ECS Staging Deployment**

```bash
# 1. Build and push image
docker build -t your-registry/backend-template:staging .
docker push your-registry/backend-template:staging

# 2. Update ECS task definition
aws ecs register-task-definition --cli-input-json file://ecs-task-staging.json

# 3. Update service
aws ecs update-service \
  --cluster staging-cluster \
  --service backend-template-staging \
  --task-definition backend-template-staging:LATEST
```

### **ECS Task Definition**

```json
{
  "family": "backend-template-staging",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "executionRoleArn": "arn:aws:iam::123456789:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::123456789:role/ecsTaskRole",
  "containerDefinitions": [
    {
      "name": "backend-template",
      "image": "your-registry/backend-template:staging",
      "portMappings": [
        {
          "containerPort": 8080,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {
          "name": "APP_ENV",
          "value": "staging"
        }
      ],
      "secrets": [
        {
          "name": "DB_PASSWORD",
          "valueFrom": "arn:aws:secretsmanager:region:account:secret:staging/db-password"
        },
        {
          "name": "JWT_SECRET",
          "valueFrom": "arn:aws:secretsmanager:region:account:secret:staging/jwt-secret"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/backend-template-staging",
          "awslogs-region": "us-west-2",
          "awslogs-stream-prefix": "ecs"
        }
      },
      "healthCheck": {
        "command": [
          "CMD-SHELL",
          "curl -f http://localhost:8080/health || exit 1"
        ],
        "interval": 30,
        "timeout": 5,
        "retries": 3,
        "startPeriod": 60
      }
    }
  ]
}
```

### **Terraform for Staging Infrastructure**

```hcl
# terraform/staging/main.tf
provider "aws" {
  region = "us-west-2"
}

module "vpc" {
  source = "../modules/vpc"
  
  name = "staging-vpc"
  cidr = "10.1.0.0/16"
  
  azs             = ["us-west-2a", "us-west-2b"]
  private_subnets = ["10.1.1.0/24", "10.1.2.0/24"]
  public_subnets  = ["10.1.101.0/24", "10.1.102.0/24"]
  
  enable_nat_gateway = true
  enable_vpn_gateway = false
  
  tags = {
    Environment = "staging"
  }
}

module "rds" {
  source = "../modules/rds"
  
  identifier = "staging-postgres"
  engine     = "postgres"
  engine_version = "15.4"
  instance_class = "db.t3.micro"
  
  allocated_storage = 20
  storage_encrypted = true
  
  db_name  = "app_db"
  username = "postgres"
  
  vpc_security_group_ids = [module.security_groups.rds_sg_id]
  db_subnet_group_name   = module.vpc.database_subnet_group
  
  backup_retention_period = 7
  backup_window          = "03:00-04:00"
  maintenance_window     = "sun:04:00-sun:05:00"
  
  tags = {
    Environment = "staging"
  }
}

module "elasticache" {
  source = "../modules/elasticache"
  
  cluster_id = "staging-redis"
  engine     = "redis"
  node_type  = "cache.t3.micro"
  
  num_cache_nodes = 1
  port           = 6379
  
  subnet_group_name  = module.vpc.elasticache_subnet_group
  security_group_ids = [module.security_groups.redis_sg_id]
  
  tags = {
    Environment = "staging"
  }
}

module "ecs" {
  source = "../modules/ecs"
  
  cluster_name = "staging-cluster"
  
  service_name = "backend-template-staging"
  task_definition_family = "backend-template-staging"
  
  container_image = "your-registry/backend-template:staging"
  container_port  = 8080
  
  desired_count = 1
  
  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets
  
  security_group_ids = [module.security_groups.ecs_sg_id]
  
  target_group_arn = module.alb.target_group_arn
  
  tags = {
    Environment = "staging"
  }
}
```

---

## 🚀 **PRODUCTION ENVIRONMENT**

### **Kubernetes Production Deployment**

#### **Namespace & ConfigMap**

```yaml
# k8s/production/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: backend-prod
  labels:
    environment: production
---
# k8s/production/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: backend-config
  namespace: backend-prod
data:
  config.yaml: |
    app:
      name: "backend-template-prod"
      environment: "production"
      debug: false
      log_level: "info"
    
    server:
      host: "0.0.0.0"
      port: 8080
      read_timeout: "30s"
      write_timeout: "30s"
      idle_timeout: "120s"
    
    database:
      host: "postgres-prod.cluster-xxx.us-west-2.rds.amazonaws.com"
      port: 5432
      name: "app_db"
      ssl_mode: "require"
      max_open_conns: 50
      max_idle_conns: 10
      conn_max_lifetime: "1h"
    
    cache:
      redis:
        host: "redis-prod.xxx.cache.amazonaws.com"
        port: 6379
        pool_size: 20
        min_idle_conns: 5
    
    monitoring:
      metrics:
        enabled: true
        path: "/metrics"
      tracing:
        enabled: true
        jaeger_endpoint: "http://jaeger-collector:14268/api/traces"
```

#### **Secrets**

```yaml
# k8s/production/secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: backend-secrets
  namespace: backend-prod
type: Opaque
data:
  db-username: cG9zdGdyZXM=  # base64 encoded
  db-password: <base64-encoded-password>
  jwt-access-secret: <base64-encoded-secret>
  jwt-refresh-secret: <base64-encoded-secret>
  redis-password: <base64-encoded-password>
```

#### **Deployment**

```yaml
# k8s/production/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-app
  namespace: backend-prod
  labels:
    app: backend-app
    version: v1
spec:
  replicas: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 2
  selector:
    matchLabels:
      app: backend-app
  template:
    metadata:
      labels:
        app: backend-app
        version: v1
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
    spec:
      serviceAccountName: backend-service-account
      securityContext:
        runAsNonRoot: true
        runAsUser: 65534
        fsGroup: 65534
      containers:
      - name: backend
        image: your-registry/backend-template:v1.0.0
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080
          name: http
          protocol: TCP
        env:
        - name: CONFIG_PATH
          value: "/etc/config/config.yaml"
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: backend-secrets
              key: db-username
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: backend-secrets
              key: db-password
        - name: JWT_ACCESS_SECRET
          valueFrom:
            secretKeyRef:
              name: backend-secrets
              key: jwt-access-secret
        - name: JWT_REFRESH_SECRET
          valueFrom:
            secretKeyRef:
              name: backend-secrets
              key: jwt-refresh-secret
        volumeMounts:
        - name: config-volume
          mountPath: /etc/config
          readOnly: true
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL
      volumes:
      - name: config-volume
        configMap:
          name: backend-config
      imagePullSecrets:
      - name: registry-secret
      nodeSelector:
        kubernetes.io/arch: amd64
      tolerations:
      - key: "node-role.kubernetes.io/master"
        operator: "Exists"
        effect: "NoSchedule"
```

#### **Service & Ingress**

```yaml
# k8s/production/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  namespace: backend-prod
  labels:
    app: backend-app
spec:
  selector:
    app: backend-app
  ports:
  - name: http
    port: 80
    targetPort: 8080
    protocol: TCP
  type: ClusterIP
---
# k8s/production/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: backend-ingress
  namespace: backend-prod
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/rate-limit: "100"
    nginx.ingress.kubernetes.io/rate-limit-window: "1m"
spec:
  tls:
  - hosts:
    - api.yourdomain.com
    secretName: backend-tls
  rules:
  - host: api.yourdomain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: backend-service
            port:
              number: 80
```

#### **HPA & VPA**

```yaml
# k8s/production/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend-hpa
  namespace: backend-prod
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend-app
  minReplicas: 5
  maxReplicas: 50
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
---
# k8s/production/vpa.yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: backend-vpa
  namespace: backend-prod
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend-app
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: backend
      maxAllowed:
        cpu: 2
        memory: 4Gi
      minAllowed:
        cpu: 100m
        memory: 128Mi
```

---

## ☁️ **CLOUD PLATFORM DEPLOYMENTS**

### **AWS EKS Deployment**

```bash
# 1. Create EKS cluster
eksctl create cluster \
  --name production-cluster \
  --version 1.28 \
  --region us-west-2 \
  --nodegroup-name worker-nodes \
  --node-type t3.medium \
  --nodes 3 \
  --nodes-min 1 \
  --nodes-max 10 \
  --managed

# 2. Install AWS Load Balancer Controller
kubectl apply -k "github.com/aws/eks-charts/stable/aws-load-balancer-controller/crds?ref=master"

helm repo add eks https://aws.github.io/eks-charts
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=production-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller

# 3. Deploy application
kubectl apply -f k8s/production/
```

### **Google GKE Deployment**

```bash
# 1. Create GKE cluster
gcloud container clusters create production-cluster \
  --zone us-central1-a \
  --num-nodes 3 \
  --enable-autoscaling \
  --min-nodes 1 \
  --max-nodes 10 \
  --machine-type n1-standard-2 \
  --enable-network-policy \
  --enable-ip-alias

# 2. Get credentials
gcloud container clusters get-credentials production-cluster \
  --zone us-central1-a

# 3. Deploy application
kubectl apply -f k8s/production/
```

### **Azure AKS Deployment**

```bash
# 1. Create resource group
az group create --name backend-prod-rg --location eastus

# 2. Create AKS cluster
az aks create \
  --resource-group backend-prod-rg \
  --name production-cluster \
  --node-count 3 \
  --enable-addons monitoring \
  --generate-ssh-keys \
  --enable-cluster-autoscaler \
  --min-count 1 \
  --max-count 10

# 3. Get credentials
az aks get-credentials \
  --resource-group backend-prod-rg \
  --name production-cluster

# 4. Deploy application
kubectl apply -f k8s/production/
```

---

## 🔄 **CI/CD PIPELINE**

### **GitHub Actions Workflow**

```yaml
# .github/workflows/deploy.yml
name: Deploy to Production

on:
  push:
    tags:
      - 'v*'

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: testdb
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
      redis:
        image: redis:7
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 6379:6379
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Set up Go
      uses: actions/setup-go@v4
      with:
        go-version: '1.21'
    
    - name: Run tests
      run: |
        go test -v -race -coverprofile=coverage.out ./...
      env:
        DATABASE_URL: postgres://postgres:postgres@localhost:5432/testdb?sslmode=disable
        REDIS_URL: redis://localhost:6379
    
    - name: Upload coverage
      uses: codecov/codecov-action@v3

  security-scan:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    
    - name: Run Trivy vulnerability scanner
      uses: aquasecurity/trivy-action@master
      with:
        scan-type: 'fs'
        scan-ref: '.'
        format: 'sarif'
        output: 'trivy-results.sarif'
    
    - name: Upload Trivy scan results
      uses: github/codeql-action/upload-sarif@v2
      with:
        sarif_file: 'trivy-results.sarif'

  build:
    needs: [test, security-scan]
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
      image-digest: ${{ steps.build.outputs.digest }}
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3
    
    - name: Log in to Container Registry
      uses: docker/login-action@v3
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}
    
    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v5
      with:
        images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
        tags: |
          type=ref,event=tag
          type=sha,prefix={{branch}}-
    
    - name: Build and push
      id: build
      uses: docker/build-push-action@v5
      with:
        context: .
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        cache-from: type=gha
        cache-to: type=gha,mode=max
        platforms: linux/amd64,linux/arm64

  deploy-staging:
    needs: build
    runs-on: ubuntu-latest
    environment: staging
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: us-west-2
    
    - name: Deploy to EKS staging
      run: |
        aws eks update-kubeconfig --name staging-cluster
        
        # Update image in deployment
        kubectl set image deployment/backend-app \
          backend=${{ needs.build.outputs.image-tag }} \
          -n backend-staging
        
        # Wait for rollout
        kubectl rollout status deployment/backend-app \
          -n backend-staging --timeout=600s

  smoke-tests:
    needs: deploy-staging
    runs-on: ubuntu-latest
    
    steps:
    - name: Run smoke tests
      run: |
        # Health check
        curl -f https://staging-api.yourdomain.com/health
        
        # Basic API test
        response=$(curl -s -o /dev/null -w "%{http_code}" \
          https://staging-api.yourdomain.com/api/v1/health)
        
        if [ $response != "200" ]; then
          echo "Smoke test failed"
          exit 1
        fi

  deploy-production:
    needs: [build, smoke-tests]
    runs-on: ubuntu-latest
    environment: production
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: us-west-2
    
    - name: Deploy to EKS production
      run: |
        aws eks update-kubeconfig --name production-cluster
        
        # Blue-Green deployment using Argo Rollouts
        kubectl patch rollout backend-rollout \
          -p '{"spec":{"template":{"spec":{"containers":[{"name":"backend","image":"${{ needs.build.outputs.image-tag }}"}]}}}}' \
          -n backend-prod
        
        # Wait for rollout
        kubectl argo rollouts get rollout backend-rollout \
          -n backend-prod --watch
```

### **ArgoCD GitOps Deployment**

```yaml
# argocd/applications/backend-prod.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: backend-prod
  namespace: argocd
spec:
  project: default
  
  source:
    repoURL: https://github.com/your-org/backend-template
    targetRevision: HEAD
    path: k8s/production
  
  destination:
    server: https://kubernetes.default.svc
    namespace: backend-prod
  
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

---

## 📊 **MONITORING & OBSERVABILITY**

### **Prometheus Configuration**

```yaml
# monitoring/prometheus/config.yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
- job_name: 'backend-app'
  kubernetes_sd_configs:
  - role: pod
    namespaces:
      names:
      - backend-prod
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

rule_files:
- "alerts.yml"

alerting:
  alertmanagers:
  - static_configs:
    - targets:
      - alertmanager:9093
```

### **Grafana Dashboards**

```json
{
  "dashboard": {
    "title": "Backend Application Metrics",
    "panels": [
      {
        "title": "Request Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(http_requests_total[5m])",
            "legendFormat": "{{method}} {{endpoint}}"
          }
        ]
      },
      {
        "title": "Response Time",
        "type": "graph",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))",
            "legendFormat": "95th percentile"
          }
        ]
      },
      {
        "title": "Error Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(http_requests_total{status_code=~\"5..\"}[5m])",
            "legendFormat": "5xx errors"
          }
        ]
      }
    ]
  }
}
```

---

## 🚨 **DISASTER RECOVERY**

### **Backup Strategy**

```bash
#!/bin/bash
# scripts/backup.sh

# Database backup
pg_dump "postgres://user:pass@host:5432/db" > backup_$(date +%Y%m%d_%H%M%S).sql

# Upload to S3
aws s3 cp backup_*.sql s3://your-backup-bucket/database/

# Redis backup
redis-cli --rdb dump.rdb
aws s3 cp dump.rdb s3://your-backup-bucket/redis/

# Clean old backups (keep 30 days)
find . -name "backup_*.sql" -mtime +30 -delete
aws s3 ls s3://your-backup-bucket/database/ | \
  awk '$1 < "'$(date -d '30 days ago' '+%Y-%m-%d')'" {print $4}' | \
  xargs -I {} aws s3 rm s3://your-backup-bucket/database/{}
```

### **Disaster Recovery Plan**

1. **RTO (Recovery Time Objective)**: 4 hours
2. **RPO (Recovery Point Objective)**: 1 hour
3. **Backup frequency**: Every 6 hours
4. **Cross-region replication**: Enabled
5. **Automated failover**: Configured

---

## 🔒 **SECURITY CHECKLIST**

### **Production Security Requirements**

- [ ] **Network Security**
  - [ ] VPC with private subnets
  - [ ] Security groups configured
  - [ ] WAF enabled
  - [ ] DDoS protection

- [ ] **Application Security**
  - [ ] HTTPS only
  - [ ] Security headers configured
  - [ ] Input validation
  - [ ] SQL injection protection

- [ ] **Authentication & Authorization**
  - [ ] JWT token validation
  - [ ] Role-based access control
  - [ ] API rate limiting
  - [ ] Session management

- [ ] **Data Protection**
  - [ ] Database encryption at rest
  - [ ] Encryption in transit
  - [ ] Secrets management
  - [ ] Data masking

- [ ] **Compliance**
  - [ ] GDPR compliance
  - [ ] Audit logging
  - [ ] Data retention policies
  - [ ] Regular security scans

---

## 📈 **PERFORMANCE OPTIMIZATION**

### **Production Performance Checklist**

- [ ] **Application Level**
  - [ ] Connection pooling optimized
  - [ ] Caching strategy implemented
  - [ ] Query optimization
  - [ ] Async processing

- [ ] **Infrastructure Level**
  - [ ] Auto-scaling configured
  - [ ] Load balancing optimized
  - [ ] CDN implemented
  - [ ] Database read replicas

- [ ] **Monitoring**
  - [ ] APM tools configured
  - [ ] Performance alerts set
  - [ ] Capacity planning done
  - [ ] SLA monitoring

---

## 📞 **SUPPORT & MAINTENANCE**

### **Production Support**

- **24/7 Monitoring**: Automated alerts and on-call rotation
- **Incident Response**: Defined escalation procedures
- **Change Management**: Controlled deployment process
- **Documentation**: Runbooks and troubleshooting guides

### **Maintenance Schedule**

- **Daily**: Health checks, log review
- **Weekly**: Performance review, capacity planning
- **Monthly**: Security updates, dependency updates
- **Quarterly**: Disaster recovery testing, architecture review

---

**🚀 Congratulations! Bạn đã có đầy đủ kiến thức để deploy backend template lên production một cách an toàn và hiệu quả! 🚀** 