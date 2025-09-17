# 🚀 GETTING STARTED GUIDE
## Step-by-Step Setup cho Backend Template

> **Mục tiêu**: Hướng dẫn chi tiết từ A-Z để setup và chạy backend production-ready trong 30 phút.

---

## 📋 **PREREQUISITES**

### **System Requirements**
- **OS**: Linux, macOS, Windows (với WSL2)
- **RAM**: Minimum 8GB, Recommended 16GB
- **CPU**: 2+ cores
- **Storage**: 20GB free space

### **Required Tools**
```bash
# Go (1.21+)
go version  # Should show 1.21+

# Docker & Docker Compose
docker --version
docker-compose --version

# Kubernetes (Optional for production)
kubectl version --client

# Database Tools
psql --version  # PostgreSQL client
redis-cli --version  # Redis client
```

---

## ⚡ **QUICK SETUP - 30 PHÚT**

### **Step 1: Clone & Initialize (5 phút)**

```bash
# 1. Clone template repository
git clone https://github.com/your-org/backend-template.git my-awesome-backend
cd my-awesome-backend

# 2. Initialize Go module
go mod init my-awesome-backend
go mod tidy

# 3. Copy configuration
cp configs/config.example.yaml configs/config.yaml
cp .env.example .env

# 4. Verify structure
tree -L 3
```

### **Step 2: Infrastructure Setup (10 phút)**

```bash
# 1. Start infrastructure services
docker-compose up -d postgres redis

# 2. Wait for services to be ready
docker-compose ps

# 3. Initialize database
make db-migrate-up
# OR manually:
# migrate -path migrations -database "postgres://postgres:password@localhost:5432/app_db?sslmode=disable" up

# 4. Seed initial data (optional)
make db-seed
# OR manually:
# go run scripts/seed.go
```

### **Step 3: Customize Business Logic (10 phút)**

```bash
# 1. Choose your domain template
DOMAIN="ecommerce"  # Options: ecommerce, finance, healthcare, saas, iot, social

# 2. Copy domain files
cp -r templates/domains/$DOMAIN/* src/core/

# 3. Update configuration for your domain
vim configs/config.yaml  # Update app.domain field

# 4. Generate domain-specific code
make generate-domain DOMAIN=$DOMAIN
```

### **Step 4: Run & Test (5 phút)**

```bash
# 1. Build application
go build -o bin/app main.go

# 2. Run locally
./bin/app
# OR with live reload:
# make dev

# 3. Test health endpoint
curl http://localhost:8080/health

# 4. Test API endpoints
curl -X POST http://localhost:8080/api/v1/products \
  -H "Content-Type: application/json" \
  -d '{"name":"Test Product","price":99.99}'
```

---

## 🔧 **DETAILED CONFIGURATION**

### **Environment Variables**

```bash
# .env file
APP_NAME=my-awesome-backend
APP_ENV=development
APP_DEBUG=true

# Database
DB_HOST=localhost
DB_PORT=5432
DB_NAME=app_db
DB_USER=postgres
DB_PASSWORD=password

# Redis
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=

# Security
JWT_ACCESS_SECRET=your-super-secret-access-key
JWT_REFRESH_SECRET=your-super-secret-refresh-key

# External Services
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USERNAME=your-email@gmail.com
SMTP_PASSWORD=your-app-password
```

### **Database Configuration**

```yaml
# configs/config.yaml
database:
  type: "postgres"
  host: "${DB_HOST:localhost}"
  port: ${DB_PORT:5432}
  name: "${DB_NAME:app_db}"
  username: "${DB_USER:postgres}"
  password: "${DB_PASSWORD:password}"
  ssl_mode: "disable"  # Use "require" in production
  max_open_conns: 25
  max_idle_conns: 5
  conn_max_lifetime: "1h"
  
  # Migration settings
  migrations_path: "./migrations"
  auto_migrate: true  # Set to false in production
```

---

## 🎯 **DOMAIN CUSTOMIZATION**

### **1. E-commerce Setup**

```bash
# Copy e-commerce templates
cp -r templates/domains/ecommerce/* src/core/

# Run e-commerce migrations
migrate -path migrations/ecommerce -database $DATABASE_URL up

# Seed e-commerce data
go run scripts/seed_ecommerce.go

# Test e-commerce endpoints
curl -X GET http://localhost:8080/api/v1/products
curl -X GET http://localhost:8080/api/v1/orders
```

### **2. Financial Services Setup**

```bash
# Copy finance templates
cp -r templates/domains/finance/* src/core/

# Run finance migrations
migrate -path migrations/finance -database $DATABASE_URL up

# Seed finance data
go run scripts/seed_finance.go

# Test finance endpoints
curl -X GET http://localhost:8080/api/v1/accounts
curl -X POST http://localhost:8080/api/v1/transactions
```

### **3. Healthcare Setup**

```bash
# Copy healthcare templates
cp -r templates/domains/healthcare/* src/core/

# Run healthcare migrations
migrate -path migrations/healthcare -database $DATABASE_URL up

# Test healthcare endpoints
curl -X GET http://localhost:8080/api/v1/patients
curl -X POST http://localhost:8080/api/v1/appointments
```

---

## 🧪 **TESTING SETUP**

### **Unit Tests**

```bash
# Run all tests
make test

# Run with coverage
make test-coverage

# Run specific package tests
go test ./src/core/usecases/...

# Run with race detection
go test -race ./...
```

### **Integration Tests**

```bash
# Setup test database
make test-db-setup

# Run integration tests
make test-integration

# Run API tests
make test-api
```

### **Load Testing**

```bash
# Install k6
brew install k6  # macOS
# OR
curl https://github.com/grafana/k6/releases/download/v0.45.0/k6-v0.45.0-linux-amd64.tar.gz -L | tar xvz --strip-components 1

# Run load tests
k6 run tests/load/basic_load_test.js
k6 run tests/load/stress_test.js
```

---

## 🚀 **PRODUCTION DEPLOYMENT**

### **Docker Deployment**

```bash
# Build production image
docker build -t my-awesome-backend:latest .

# Run with production config
docker run -d \
  --name my-backend \
  -p 8080:8080 \
  -e APP_ENV=production \
  -e DB_HOST=your-db-host \
  --restart unless-stopped \
  my-awesome-backend:latest
```

### **Kubernetes Deployment**

```bash
# Update K8s manifests
vim k8s/configmap.yaml
vim k8s/deployment.yaml

# Deploy to cluster
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/configmap.yaml
kubectl apply -f k8s/secrets.yaml
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
kubectl apply -f k8s/ingress.yaml

# Verify deployment
kubectl get pods -n my-backend
kubectl logs -f deployment/my-backend-app -n my-backend
```

### **Cloud Deployment**

#### **AWS EKS**
```bash
# Create EKS cluster
eksctl create cluster --name my-backend --region us-west-2

# Deploy application
kubectl apply -f k8s/aws/
```

#### **Google GKE**
```bash
# Create GKE cluster
gcloud container clusters create my-backend --zone us-central1-a

# Deploy application
kubectl apply -f k8s/gcp/
```

---

## 📊 **MONITORING SETUP**

### **Metrics & Monitoring**

```bash
# Setup Prometheus & Grafana
kubectl apply -f monitoring/prometheus/
kubectl apply -f monitoring/grafana/

# Access Grafana dashboard
kubectl port-forward svc/grafana 3000:3000

# Import dashboard templates
# Visit http://localhost:3000 (admin/admin)
# Import dashboards from monitoring/dashboards/
```

### **Logging**

```bash
# Setup ELK Stack
kubectl apply -f monitoring/elasticsearch/
kubectl apply -f monitoring/logstash/
kubectl apply -f monitoring/kibana/

# Access Kibana
kubectl port-forward svc/kibana 5601:5601
```

### **Tracing**

```bash
# Setup Jaeger
kubectl apply -f monitoring/jaeger/

# Access Jaeger UI
kubectl port-forward svc/jaeger-query 16686:16686
```

---

## 🔧 **DEVELOPMENT WORKFLOW**

### **Daily Development**

```bash
# Start development environment
make dev-start

# Run with hot reload
make dev

# Run tests on file change
make test-watch

# Check code quality
make lint
make format
```

### **Git Workflow**

```bash
# Create feature branch
git checkout -b feature/user-authentication

# Make changes and commit
git add .
git commit -m "feat: add user authentication"

# Push and create PR
git push origin feature/user-authentication
```

### **Code Quality**

```bash
# Install pre-commit hooks
make install-hooks

# Run all quality checks
make quality-check

# Fix formatting issues
make format-fix
```

---

## 🐛 **TROUBLESHOOTING**

### **Common Issues**

#### **Database Connection Issues**
```bash
# Check database status
docker-compose ps postgres

# Check connection
psql "postgres://postgres:password@localhost:5432/app_db"

# Reset database
make db-reset
```

#### **Redis Connection Issues**
```bash
# Check Redis status
docker-compose ps redis

# Test connection
redis-cli -h localhost -p 6379 ping

# Clear Redis cache
redis-cli -h localhost -p 6379 flushall
```

#### **Port Already in Use**
```bash
# Find process using port 8080
lsof -i :8080

# Kill process
kill -9 <PID>

# OR change port in config
vim configs/config.yaml  # Update server.port
```

### **Performance Issues**

```bash
# Check application metrics
curl http://localhost:8080/metrics

# Profile application
go tool pprof http://localhost:8080/debug/pprof/profile

# Check memory usage
go tool pprof http://localhost:8080/debug/pprof/heap
```

---

## 📚 **NEXT STEPS**

### **After Setup**

1. **📖 Read Documentation**
   - `BACKEND_GUIDE.md` - Complete implementation guide
   - `DOMAIN_EXAMPLES.md` - Business logic examples
   - `API_DOCUMENTATION.md` - API reference

2. **🎯 Customize for Your Domain**
   - Define your business entities
   - Implement use cases
   - Create API endpoints
   - Add validation rules

3. **🚀 Production Readiness**
   - Setup monitoring and alerting
   - Configure CI/CD pipeline
   - Implement security measures
   - Performance optimization

### **Learning Resources**

- **Clean Architecture**: Uncle Bob's Clean Architecture principles
- **Domain-Driven Design**: Eric Evans' DDD concepts
- **Go Best Practices**: Effective Go and Go Code Review Comments
- **Kubernetes**: Kubernetes documentation and tutorials
- **Observability**: Prometheus, Grafana, Jaeger guides

---

## 💬 **SUPPORT**

### **Getting Help**

- **📚 Documentation**: Check existing docs first
- **🐛 Issues**: Create GitHub issue for bugs
- **💬 Discussions**: Use GitHub Discussions for questions
- **📧 Enterprise Support**: Contact for enterprise support

### **Contributing**

- **🔧 Bug Fixes**: Submit PRs for bug fixes
- **✨ Features**: Discuss new features first
- **📖 Documentation**: Help improve docs
- **🧪 Testing**: Add more test cases

---

**🎉 Congratulations! Bạn đã setup thành công backend template. Giờ hãy bắt đầu build amazing products! 🚀** 