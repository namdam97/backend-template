# 🧪 TESTING GUIDE
## Comprehensive Testing Strategy cho Production Backend

> **Mục tiêu**: Hướng dẫn chi tiết về testing strategy từ unit tests đến production monitoring để đảm bảo chất lượng code và system reliability.

---

## 🎯 **TESTING PYRAMID**

```
                    🔺
                  E2E Tests
                 (5% - Slow)
                
            🔺🔺🔺🔺🔺
         Integration Tests  
        (20% - Medium Speed)
        
    🔺🔺🔺🔺🔺🔺🔺🔺🔺
        Unit Tests
    (75% - Fast & Reliable)
```

### **Testing Principles**
- **Fast Feedback**: Unit tests chạy nhanh, feedback tức thì
- **Reliable**: Tests không flaky, kết quả consistent
- **Maintainable**: Tests dễ hiểu và maintain
- **Comprehensive**: Coverage tất cả business logic critical

---

## 🔬 **UNIT TESTING**

### **Testing Framework Setup**

```go
// go.mod
module backend-template

go 1.21

require (
    github.com/stretchr/testify v1.8.4
    github.com/golang/mock v1.6.0
    github.com/DATA-DOG/go-sqlmock v1.5.0
    github.com/go-redis/redismock/v9 v9.2.0
)
```

### **Use Case Testing**

```go
// src/core/usecases/product_usecase_test.go
package usecases_test

import (
    "context"
    "errors"
    "testing"
    "time"

    "github.com/golang/mock/gomock"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/suite"

    "backend-template/src/core/domains"
    "backend-template/src/core/usecases"
    "backend-template/src/mocks"
)

type ProductUseCaseTestSuite struct {
    suite.Suite
    ctrl        *gomock.Controller
    mockRepo    *mocks.MockProductRepository
    mockCache   *mocks.MockCache
    mockLogger  *mocks.MockLogger
    mockMetrics *mocks.MockMetricsService
    useCase     *usecases.ProductUseCase
}

func (suite *ProductUseCaseTestSuite) SetupTest() {
    suite.ctrl = gomock.NewController(suite.T())
    suite.mockRepo = mocks.NewMockProductRepository(suite.ctrl)
    suite.mockCache = mocks.NewMockCache(suite.ctrl)
    suite.mockLogger = mocks.NewMockLogger(suite.ctrl)
    suite.mockMetrics = mocks.NewMockMetricsService(suite.ctrl)
    
    suite.useCase = usecases.NewProductUseCase(
        suite.mockRepo,
        suite.mockCache,
        suite.mockLogger,
        suite.mockMetrics,
    )
}

func (suite *ProductUseCaseTestSuite) TearDownTest() {
    suite.ctrl.Finish()
}

func (suite *ProductUseCaseTestSuite) TestCreateProduct_Success() {
    // Arrange
    product := &domains.Product{
        Name:        "Test Product",
        Description: "Test Description",
        Price: domains.Money{
            Amount:   99.99,
            Currency: "USD",
        },
        Stock:      10,
        SKU:        "TEST-001",
        CategoryID: "cat_123",
        Status:     "active",
    }

    suite.mockRepo.EXPECT().
        GetBySKU(gomock.Any(), product.SKU).
        Return(nil, errors.New("not found")).
        Times(1)

    suite.mockRepo.EXPECT().
        Create(gomock.Any(), gomock.Any()).
        DoAndReturn(func(ctx context.Context, p *domains.Product) error {
            // Verify product fields are set correctly
            assert.NotEmpty(suite.T(), p.ID)
            assert.Equal(suite.T(), product.Name, p.Name)
            assert.Equal(suite.T(), product.SKU, p.SKU)
            assert.NotZero(suite.T(), p.CreatedAt)
            return nil
        }).
        Times(1)

    suite.mockCache.EXPECT().
        Delete(gomock.Any(), gomock.Any()).
        Return(nil).
        Times(1)

    suite.mockLogger.EXPECT().
        Info(gomock.Any(), gomock.Any(), gomock.Any()).
        Times(1)

    suite.mockMetrics.EXPECT().
        RecordBusinessOperation(gomock.Any(), gomock.Any(), gomock.Any(), gomock.Any()).
        Times(1)

    // Act
    err := suite.useCase.CreateProduct(context.Background(), product)

    // Assert
    assert.NoError(suite.T(), err)
    assert.NotEmpty(suite.T(), product.ID)
    assert.NotZero(suite.T(), product.CreatedAt)
}

func (suite *ProductUseCaseTestSuite) TestCreateProduct_DuplicateSKU() {
    // Arrange
    existingProduct := &domains.Product{
        ID:  "existing_id",
        SKU: "TEST-001",
    }
    
    newProduct := &domains.Product{
        SKU: "TEST-001",
    }

    suite.mockRepo.EXPECT().
        GetBySKU(gomock.Any(), newProduct.SKU).
        Return(existingProduct, nil).
        Times(1)

    // Act
    err := suite.useCase.CreateProduct(context.Background(), newProduct)

    // Assert
    assert.Error(suite.T(), err)
    assert.Contains(suite.T(), err.Error(), "already exists")
}

func (suite *ProductUseCaseTestSuite) TestGetProduct_CacheHit() {
    // Arrange
    productID := "prod_123"
    cachedProduct := &domains.Product{
        ID:   productID,
        Name: "Cached Product",
    }

    suite.mockCache.EXPECT().
        Get(gomock.Any(), "product:"+productID).
        Return(cachedProduct, nil).
        Times(1)

    // Act
    result, err := suite.useCase.GetProduct(context.Background(), productID)

    // Assert
    assert.NoError(suite.T(), err)
    assert.Equal(suite.T(), cachedProduct, result)
}

func (suite *ProductUseCaseTestSuite) TestGetProduct_CacheMiss() {
    // Arrange
    productID := "prod_123"
    dbProduct := &domains.Product{
        ID:   productID,
        Name: "DB Product",
    }

    suite.mockCache.EXPECT().
        Get(gomock.Any(), "product:"+productID).
        Return(nil, errors.New("cache miss")).
        Times(1)

    suite.mockRepo.EXPECT().
        GetByID(gomock.Any(), productID).
        Return(dbProduct, nil).
        Times(1)

    suite.mockCache.EXPECT().
        Set(gomock.Any(), "product:"+productID, dbProduct, 1*time.Hour).
        Return(nil).
        Times(1)

    // Act
    result, err := suite.useCase.GetProduct(context.Background(), productID)

    // Assert
    assert.NoError(suite.T(), err)
    assert.Equal(suite.T(), dbProduct, result)
}

func TestProductUseCaseTestSuite(t *testing.T) {
    suite.Run(t, new(ProductUseCaseTestSuite))
}
```

### **Repository Testing with SQL Mock**

```go
// src/infra/database/product_repository_test.go
package database_test

import (
    "context"
    "database/sql"
    "testing"
    "time"

    "github.com/DATA-DOG/go-sqlmock"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/suite"

    "backend-template/src/core/domains"
    "backend-template/src/infra/database"
)

type ProductRepositoryTestSuite struct {
    suite.Suite
    db   *sql.DB
    mock sqlmock.Sqlmock
    repo domains.ProductRepository
}

func (suite *ProductRepositoryTestSuite) SetupTest() {
    var err error
    suite.db, suite.mock, err = sqlmock.New()
    assert.NoError(suite.T(), err)
    
    suite.repo = database.NewProductRepository(suite.db)
}

func (suite *ProductRepositoryTestSuite) TearDownTest() {
    suite.db.Close()
}

func (suite *ProductRepositoryTestSuite) TestCreate_Success() {
    // Arrange
    product := &domains.Product{
        ID:          "prod_123",
        Name:        "Test Product",
        Description: "Test Description",
        Price: domains.Money{
            Amount:   99.99,
            Currency: "USD",
        },
        Stock:      10,
        SKU:        "TEST-001",
        CategoryID: "cat_123",
        Status:     "active",
        CreatedAt:  time.Now(),
        UpdatedAt:  time.Now(),
    }

    suite.mock.ExpectExec(`
        INSERT INTO products 
        \(id, name, description, price_amount, price_currency, stock, sku, category_id, status, created_at, updated_at\)
        VALUES \(\?, \?, \?, \?, \?, \?, \?, \?, \?, \?, \?\)
    `).WithArgs(
        product.ID,
        product.Name,
        product.Description,
        product.Price.Amount,
        product.Price.Currency,
        product.Stock,
        product.SKU,
        product.CategoryID,
        product.Status,
        sqlmock.AnyArg(),
        sqlmock.AnyArg(),
    ).WillReturnResult(sqlmock.NewResult(1, 1))

    // Act
    err := suite.repo.Create(context.Background(), product)

    // Assert
    assert.NoError(suite.T(), err)
    assert.NoError(suite.T(), suite.mock.ExpectationsWereMet())
}

func (suite *ProductRepositoryTestSuite) TestGetByID_Success() {
    // Arrange
    productID := "prod_123"
    rows := sqlmock.NewRows([]string{
        "id", "name", "description", "price_amount", "price_currency",
        "stock", "sku", "category_id", "status", "created_at", "updated_at",
    }).AddRow(
        "prod_123", "Test Product", "Test Description", 99.99, "USD",
        10, "TEST-001", "cat_123", "active", time.Now(), time.Now(),
    )

    suite.mock.ExpectQuery(`
        SELECT id, name, description, price_amount, price_currency, stock, sku, category_id, status, created_at, updated_at
        FROM products WHERE id = \?
    `).WithArgs(productID).WillReturnRows(rows)

    // Act
    product, err := suite.repo.GetByID(context.Background(), productID)

    // Assert
    assert.NoError(suite.T(), err)
    assert.Equal(suite.T(), "prod_123", product.ID)
    assert.Equal(suite.T(), "Test Product", product.Name)
    assert.NoError(suite.T(), suite.mock.ExpectationsWereMet())
}

func (suite *ProductRepositoryTestSuite) TestGetByID_NotFound() {
    // Arrange
    productID := "nonexistent"
    
    suite.mock.ExpectQuery(`
        SELECT id, name, description, price_amount, price_currency, stock, sku, category_id, status, created_at, updated_at
        FROM products WHERE id = \?
    `).WithArgs(productID).WillReturnError(sql.ErrNoRows)

    // Act
    product, err := suite.repo.GetByID(context.Background(), productID)

    // Assert
    assert.Error(suite.T(), err)
    assert.Nil(suite.T(), product)
    assert.NoError(suite.T(), suite.mock.ExpectationsWereMet())
}

func TestProductRepositoryTestSuite(t *testing.T) {
    suite.Run(t, new(ProductRepositoryTestSuite))
}
```

---

## 🔗 **INTEGRATION TESTING**

### **Database Integration Tests**

```go
// tests/integration/database_test.go
package integration_test

import (
    "context"
    "database/sql"
    "testing"
    "time"

    "github.com/golang-migrate/migrate/v4"
    "github.com/golang-migrate/migrate/v4/database/postgres"
    _ "github.com/golang-migrate/migrate/v4/source/file"
    _ "github.com/lib/pq"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/suite"
    "github.com/testcontainers/testcontainers-go"
    "github.com/testcontainers/testcontainers-go/wait"

    "backend-template/src/core/domains"
    "backend-template/src/infra/database"
)

type DatabaseIntegrationTestSuite struct {
    suite.Suite
    db        *sql.DB
    container testcontainers.Container
    repo      domains.ProductRepository
}

func (suite *DatabaseIntegrationTestSuite) SetupSuite() {
    ctx := context.Background()
    
    // Start PostgreSQL container
    req := testcontainers.ContainerRequest{
        Image:        "postgres:15-alpine",
        ExposedPorts: []string{"5432/tcp"},
        Env: map[string]string{
            "POSTGRES_DB":       "testdb",
            "POSTGRES_USER":     "testuser",
            "POSTGRES_PASSWORD": "testpass",
        },
        WaitingFor: wait.ForLog("database system is ready to accept connections"),
    }
    
    container, err := testcontainers.GenericContainer(ctx, testcontainers.GenericContainerRequest{
        ContainerRequest: req,
        Started:          true,
    })
    assert.NoError(suite.T(), err)
    suite.container = container
    
    // Get connection details
    host, err := container.Host(ctx)
    assert.NoError(suite.T(), err)
    
    port, err := container.MappedPort(ctx, "5432")
    assert.NoError(suite.T(), err)
    
    // Connect to database
    dsn := fmt.Sprintf("postgres://testuser:testpass@%s:%s/testdb?sslmode=disable", 
        host, port.Port())
    suite.db, err = sql.Open("postgres", dsn)
    assert.NoError(suite.T(), err)
    
    // Run migrations
    driver, err := postgres.WithInstance(suite.db, &postgres.Config{})
    assert.NoError(suite.T(), err)
    
    m, err := migrate.NewWithDatabaseInstance("file://../../migrations", "postgres", driver)
    assert.NoError(suite.T(), err)
    
    err = m.Up()
    assert.NoError(suite.T(), err)
    
    // Initialize repository
    suite.repo = database.NewProductRepository(suite.db)
}

func (suite *DatabaseIntegrationTestSuite) TearDownSuite() {
    if suite.db != nil {
        suite.db.Close()
    }
    if suite.container != nil {
        suite.container.Terminate(context.Background())
    }
}

func (suite *DatabaseIntegrationTestSuite) SetupTest() {
    // Clean up data before each test
    _, err := suite.db.Exec("TRUNCATE TABLE products CASCADE")
    assert.NoError(suite.T(), err)
}

func (suite *DatabaseIntegrationTestSuite) TestProductCRUD() {
    ctx := context.Background()
    
    // Create
    product := &domains.Product{
        ID:          "prod_integration_test",
        Name:        "Integration Test Product",
        Description: "Product for integration testing",
        Price: domains.Money{
            Amount:   199.99,
            Currency: "USD",
        },
        Stock:      5,
        SKU:        "INT-TEST-001",
        CategoryID: "cat_test",
        Status:     "active",
        CreatedAt:  time.Now(),
        UpdatedAt:  time.Now(),
    }
    
    err := suite.repo.Create(ctx, product)
    assert.NoError(suite.T(), err)
    
    // Read
    retrieved, err := suite.repo.GetByID(ctx, product.ID)
    assert.NoError(suite.T(), err)
    assert.Equal(suite.T(), product.ID, retrieved.ID)
    assert.Equal(suite.T(), product.Name, retrieved.Name)
    assert.Equal(suite.T(), product.SKU, retrieved.SKU)
    
    // Update
    retrieved.Name = "Updated Product Name"
    retrieved.UpdatedAt = time.Now()
    err = suite.repo.Update(ctx, retrieved)
    assert.NoError(suite.T(), err)
    
    updated, err := suite.repo.GetByID(ctx, product.ID)
    assert.NoError(suite.T(), err)
    assert.Equal(suite.T(), "Updated Product Name", updated.Name)
    
    // Delete
    err = suite.repo.Delete(ctx, product.ID)
    assert.NoError(suite.T(), err)
    
    _, err = suite.repo.GetByID(ctx, product.ID)
    assert.Error(suite.T(), err)
}

func TestDatabaseIntegrationTestSuite(t *testing.T) {
    suite.Run(t, new(DatabaseIntegrationTestSuite))
}
```

### **Redis Integration Tests**

```go
// tests/integration/redis_test.go
package integration_test

import (
    "context"
    "testing"
    "time"

    "github.com/go-redis/redis/v8"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/suite"
    "github.com/testcontainers/testcontainers-go"
    "github.com/testcontainers/testcontainers-go/wait"

    "backend-template/src/common/cache"
)

type RedisIntegrationTestSuite struct {
    suite.Suite
    client    *redis.Client
    container testcontainers.Container
    cache     cache.Cache
}

func (suite *RedisIntegrationTestSuite) SetupSuite() {
    ctx := context.Background()
    
    // Start Redis container
    req := testcontainers.ContainerRequest{
        Image:        "redis:7-alpine",
        ExposedPorts: []string{"6379/tcp"},
        WaitingFor:   wait.ForLog("Ready to accept connections"),
    }
    
    container, err := testcontainers.GenericContainer(ctx, testcontainers.GenericContainerRequest{
        ContainerRequest: req,
        Started:          true,
    })
    assert.NoError(suite.T(), err)
    suite.container = container
    
    // Get connection details
    host, err := container.Host(ctx)
    assert.NoError(suite.T(), err)
    
    port, err := container.MappedPort(ctx, "6379")
    assert.NoError(suite.T(), err)
    
    // Connect to Redis
    suite.client = redis.NewClient(&redis.Options{
        Addr: fmt.Sprintf("%s:%s", host, port.Port()),
    })
    
    // Initialize cache
    suite.cache = cache.NewRedisCache(suite.client)
}

func (suite *RedisIntegrationTestSuite) TearDownSuite() {
    if suite.client != nil {
        suite.client.Close()
    }
    if suite.container != nil {
        suite.container.Terminate(context.Background())
    }
}

func (suite *RedisIntegrationTestSuite) SetupTest() {
    // Clean Redis before each test
    suite.client.FlushAll(context.Background())
}

func (suite *RedisIntegrationTestSuite) TestCacheOperations() {
    ctx := context.Background()
    
    // Test Set and Get
    key := "test:key"
    value := "test value"
    ttl := 1 * time.Minute
    
    err := suite.cache.Set(ctx, key, value, ttl)
    assert.NoError(suite.T(), err)
    
    retrieved, err := suite.cache.Get(ctx, key)
    assert.NoError(suite.T(), err)
    assert.Equal(suite.T(), value, retrieved)
    
    // Test Exists
    exists, err := suite.cache.Exists(ctx, key)
    assert.NoError(suite.T(), err)
    assert.True(suite.T(), exists)
    
    // Test Delete
    err = suite.cache.Delete(ctx, key)
    assert.NoError(suite.T(), err)
    
    exists, err = suite.cache.Exists(ctx, key)
    assert.NoError(suite.T(), err)
    assert.False(suite.T(), exists)
}

func (suite *RedisIntegrationTestSuite) TestTTL() {
    ctx := context.Background()
    
    key := "test:ttl"
    value := "test value"
    ttl := 100 * time.Millisecond
    
    err := suite.cache.Set(ctx, key, value, ttl)
    assert.NoError(suite.T(), err)
    
    // Value should exist initially
    retrieved, err := suite.cache.Get(ctx, key)
    assert.NoError(suite.T(), err)
    assert.Equal(suite.T(), value, retrieved)
    
    // Wait for TTL to expire
    time.Sleep(150 * time.Millisecond)
    
    // Value should be expired
    _, err = suite.cache.Get(ctx, key)
    assert.Error(suite.T(), err)
}

func TestRedisIntegrationTestSuite(t *testing.T) {
    suite.Run(t, new(RedisIntegrationTestSuite))
}
```

---

## 🌐 **API TESTING**

### **HTTP API Tests**

```go
// tests/api/product_api_test.go
package api_test

import (
    "bytes"
    "encoding/json"
    "net/http"
    "net/http/httptest"
    "testing"

    "github.com/gin-gonic/gin"
    "github.com/golang/mock/gomock"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/suite"

    "backend-template/src/core/domains"
    "backend-template/src/mocks"
    "backend-template/src/present/http/controllers"
)

type ProductAPITestSuite struct {
    suite.Suite
    router      *gin.Engine
    ctrl        *gomock.Controller
    mockUseCase *mocks.MockProductUseCase
    mockLogger  *mocks.MockLogger
}

func (suite *ProductAPITestSuite) SetupTest() {
    gin.SetMode(gin.TestMode)
    suite.router = gin.New()
    
    suite.ctrl = gomock.NewController(suite.T())
    suite.mockUseCase = mocks.NewMockProductUseCase(suite.ctrl)
    suite.mockLogger = mocks.NewMockLogger(suite.ctrl)
    
    controller := controllers.NewProductController(suite.mockUseCase, suite.mockLogger)
    
    api := suite.router.Group("/api/v1")
    {
        api.POST("/products", controller.CreateProduct)
        api.GET("/products/:id", controller.GetProduct)
        api.GET("/products", controller.SearchProducts)
    }
}

func (suite *ProductAPITestSuite) TearDownTest() {
    suite.ctrl.Finish()
}

func (suite *ProductAPITestSuite) TestCreateProduct_Success() {
    // Arrange
    requestBody := map[string]interface{}{
        "name":        "Test Product",
        "description": "Test Description",
        "price":       99.99,
        "currency":    "USD",
        "stock":       10,
        "sku":         "TEST-001",
        "category_id": "cat_123",
    }
    
    suite.mockUseCase.EXPECT().
        CreateProduct(gomock.Any(), gomock.Any()).
        DoAndReturn(func(ctx context.Context, product *domains.Product) error {
            product.ID = "prod_123"
            return nil
        }).
        Times(1)

    body, _ := json.Marshal(requestBody)
    req := httptest.NewRequest(http.MethodPost, "/api/v1/products", bytes.NewReader(body))
    req.Header.Set("Content-Type", "application/json")
    
    // Act
    w := httptest.NewRecorder()
    suite.router.ServeHTTP(w, req)
    
    // Assert
    assert.Equal(suite.T(), http.StatusCreated, w.Code)
    
    var response map[string]interface{}
    err := json.Unmarshal(w.Body.Bytes(), &response)
    assert.NoError(suite.T(), err)
    
    assert.Equal(suite.T(), "Product created successfully", response["message"])
    
    data := response["data"].(map[string]interface{})
    assert.Equal(suite.T(), "prod_123", data["id"])
    assert.Equal(suite.T(), "Test Product", data["name"])
}

func (suite *ProductAPITestSuite) TestCreateProduct_ValidationError() {
    // Arrange
    requestBody := map[string]interface{}{
        "name": "", // Empty name should fail validation
        "price": -1, // Negative price should fail validation
    }

    body, _ := json.Marshal(requestBody)
    req := httptest.NewRequest(http.MethodPost, "/api/v1/products", bytes.NewReader(body))
    req.Header.Set("Content-Type", "application/json")
    
    // Act
    w := httptest.NewRecorder()
    suite.router.ServeHTTP(w, req)
    
    // Assert
    assert.Equal(suite.T(), http.StatusBadRequest, w.Code)
    
    var response map[string]interface{}
    err := json.Unmarshal(w.Body.Bytes(), &response)
    assert.NoError(suite.T(), err)
    
    assert.Contains(suite.T(), response["error"], "Validation failed")
}

func (suite *ProductAPITestSuite) TestGetProduct_Success() {
    // Arrange
    productID := "prod_123"
    product := &domains.Product{
        ID:          productID,
        Name:        "Test Product",
        Description: "Test Description",
        Price: domains.Money{
            Amount:   99.99,
            Currency: "USD",
        },
        Stock:      10,
        SKU:        "TEST-001",
        CategoryID: "cat_123",
        Status:     "active",
    }
    
    suite.mockUseCase.EXPECT().
        GetProduct(gomock.Any(), productID).
        Return(product, nil).
        Times(1)

    req := httptest.NewRequest(http.MethodGet, "/api/v1/products/"+productID, nil)
    
    // Act
    w := httptest.NewRecorder()
    suite.router.ServeHTTP(w, req)
    
    // Assert
    assert.Equal(suite.T(), http.StatusOK, w.Code)
    
    var response map[string]interface{}
    err := json.Unmarshal(w.Body.Bytes(), &response)
    assert.NoError(suite.T(), err)
    
    data := response["data"].(map[string]interface{})
    assert.Equal(suite.T(), productID, data["id"])
    assert.Equal(suite.T(), "Test Product", data["name"])
}

func (suite *ProductAPITestSuite) TestGetProduct_NotFound() {
    // Arrange
    productID := "nonexistent"
    
    suite.mockUseCase.EXPECT().
        GetProduct(gomock.Any(), productID).
        Return(nil, errors.New("product not found")).
        Times(1)

    req := httptest.NewRequest(http.MethodGet, "/api/v1/products/"+productID, nil)
    
    // Act
    w := httptest.NewRecorder()
    suite.router.ServeHTTP(w, req)
    
    // Assert
    assert.Equal(suite.T(), http.StatusNotFound, w.Code)
}

func TestProductAPITestSuite(t *testing.T) {
    suite.Run(t, new(ProductAPITestSuite))
}
```

### **End-to-End API Tests**

```go
// tests/e2e/product_e2e_test.go
package e2e_test

import (
    "bytes"
    "encoding/json"
    "fmt"
    "net/http"
    "testing"
    "time"

    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/suite"
)

type ProductE2ETestSuite struct {
    suite.Suite
    baseURL     string
    httpClient  *http.Client
    accessToken string
}

func (suite *ProductE2ETestSuite) SetupSuite() {
    suite.baseURL = "http://localhost:8080/api/v1"
    suite.httpClient = &http.Client{
        Timeout: 30 * time.Second,
    }
    
    // Login to get access token
    suite.login()
}

func (suite *ProductE2ETestSuite) login() {
    loginData := map[string]string{
        "email":    "test@example.com",
        "password": "password123",
    }
    
    body, _ := json.Marshal(loginData)
    resp, err := suite.httpClient.Post(
        suite.baseURL+"/auth/login",
        "application/json",
        bytes.NewReader(body),
    )
    assert.NoError(suite.T(), err)
    defer resp.Body.Close()
    
    var loginResponse map[string]interface{}
    err = json.NewDecoder(resp.Body).Decode(&loginResponse)
    assert.NoError(suite.T(), err)
    
    data := loginResponse["data"].(map[string]interface{})
    tokens := data["tokens"].(map[string]interface{})
    suite.accessToken = tokens["access_token"].(string)
}

func (suite *ProductE2ETestSuite) makeAuthenticatedRequest(method, url string, body []byte) (*http.Response, error) {
    req, err := http.NewRequest(method, url, bytes.NewReader(body))
    if err != nil {
        return nil, err
    }
    
    req.Header.Set("Content-Type", "application/json")
    req.Header.Set("Authorization", "Bearer "+suite.accessToken)
    
    return suite.httpClient.Do(req)
}

func (suite *ProductE2ETestSuite) TestProductWorkflow() {
    // 1. Create a product
    productData := map[string]interface{}{
        "name":        "E2E Test Product",
        "description": "Product created in E2E test",
        "price":       149.99,
        "currency":    "USD",
        "stock":       20,
        "sku":         "E2E-TEST-001",
        "category_id": "cat_electronics",
    }
    
    body, _ := json.Marshal(productData)
    resp, err := suite.makeAuthenticatedRequest(
        http.MethodPost,
        suite.baseURL+"/products",
        body,
    )
    assert.NoError(suite.T(), err)
    defer resp.Body.Close()
    
    assert.Equal(suite.T(), http.StatusCreated, resp.StatusCode)
    
    var createResponse map[string]interface{}
    err = json.NewDecoder(resp.Body).Decode(&createResponse)
    assert.NoError(suite.T(), err)
    
    data := createResponse["data"].(map[string]interface{})
    productID := data["id"].(string)
    assert.NotEmpty(suite.T(), productID)
    
    // 2. Get the created product
    resp, err = suite.makeAuthenticatedRequest(
        http.MethodGet,
        suite.baseURL+"/products/"+productID,
        nil,
    )
    assert.NoError(suite.T(), err)
    defer resp.Body.Close()
    
    assert.Equal(suite.T(), http.StatusOK, resp.StatusCode)
    
    var getResponse map[string]interface{}
    err = json.NewDecoder(resp.Body).Decode(&getResponse)
    assert.NoError(suite.T(), err)
    
    retrievedData := getResponse["data"].(map[string]interface{})
    assert.Equal(suite.T(), productID, retrievedData["id"])
    assert.Equal(suite.T(), "E2E Test Product", retrievedData["name"])
    
    // 3. Search for products
    resp, err = suite.makeAuthenticatedRequest(
        http.MethodGet,
        suite.baseURL+"/products?q=E2E Test",
        nil,
    )
    assert.NoError(suite.T(), err)
    defer resp.Body.Close()
    
    assert.Equal(suite.T(), http.StatusOK, resp.StatusCode)
    
    var searchResponse map[string]interface{}
    err = json.NewDecoder(resp.Body).Decode(&searchResponse)
    assert.NoError(suite.T(), err)
    
    products := searchResponse["data"].([]interface{})
    assert.GreaterOrEqual(suite.T(), len(products), 1)
    
    // Find our product in search results
    found := false
    for _, p := range products {
        product := p.(map[string]interface{})
        if product["id"].(string) == productID {
            found = true
            break
        }
    }
    assert.True(suite.T(), found, "Created product should be found in search results")
    
    // 4. Update product stock
    updateData := map[string]interface{}{
        "stock": 15,
    }
    
    updateBody, _ := json.Marshal(updateData)
    resp, err = suite.makeAuthenticatedRequest(
        http.MethodPatch,
        suite.baseURL+"/products/"+productID+"/stock",
        updateBody,
    )
    assert.NoError(suite.T(), err)
    defer resp.Body.Close()
    
    assert.Equal(suite.T(), http.StatusOK, resp.StatusCode)
    
    // 5. Verify stock update
    resp, err = suite.makeAuthenticatedRequest(
        http.MethodGet,
        suite.baseURL+"/products/"+productID,
        nil,
    )
    assert.NoError(suite.T(), err)
    defer resp.Body.Close()
    
    var updatedResponse map[string]interface{}
    err = json.NewDecoder(resp.Body).Decode(&updatedResponse)
    assert.NoError(suite.T(), err)
    
    updatedData := updatedResponse["data"].(map[string]interface{})
    assert.Equal(suite.T(), float64(15), updatedData["stock"])
}

func TestProductE2ETestSuite(t *testing.T) {
    // Skip E2E tests if not running in CI or with E2E flag
    if testing.Short() {
        t.Skip("Skipping E2E tests in short mode")
    }
    
    suite.Run(t, new(ProductE2ETestSuite))
}
```

---

## ⚡ **LOAD TESTING**

### **K6 Load Tests**

```javascript
// tests/load/basic_load_test.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Counter, Rate, Trend } from 'k6/metrics';

// Custom metrics
const errorCount = new Counter('errors');
const errorRate = new Rate('error_rate');
const responseTime = new Trend('response_time');

// Test configuration
export const options = {
  stages: [
    { duration: '2m', target: 100 }, // Ramp up to 100 users
    { duration: '5m', target: 100 }, // Stay at 100 users
    { duration: '2m', target: 200 }, // Ramp up to 200 users
    { duration: '5m', target: 200 }, // Stay at 200 users
    { duration: '2m', target: 0 },   // Ramp down to 0 users
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'], // 95% of requests must be below 500ms
    http_req_failed: ['rate<0.02'],   // Error rate must be below 2%
    error_rate: ['rate<0.02'],
  },
};

// Test data
const BASE_URL = 'http://localhost:8080/api/v1';
let authToken = '';

export function setup() {
  // Login to get auth token
  const loginPayload = JSON.stringify({
    email: 'loadtest@example.com',
    password: 'password123',
  });

  const loginResponse = http.post(`${BASE_URL}/auth/login`, loginPayload, {
    headers: { 'Content-Type': 'application/json' },
  });

  if (loginResponse.status === 200) {
    const responseBody = JSON.parse(loginResponse.body);
    authToken = responseBody.data.tokens.access_token;
  }

  return { authToken };
}

export default function (data) {
  const headers = {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${data.authToken}`,
  };

  // Test scenarios with different weights
  const scenario = Math.random();
  
  if (scenario < 0.7) {
    // 70% - Read operations (GET products)
    testGetProducts(headers);
  } else if (scenario < 0.9) {
    // 20% - Search operations
    testSearchProducts(headers);
  } else {
    // 10% - Write operations (POST products)
    testCreateProduct(headers);
  }

  sleep(1);
}

function testGetProducts(headers) {
  const response = http.get(`${BASE_URL}/products`, { headers });
  
  const success = check(response, {
    'GET products status is 200': (r) => r.status === 200,
    'GET products response time < 200ms': (r) => r.timings.duration < 200,
    'GET products has data': (r) => {
      const body = JSON.parse(r.body);
      return body.data && Array.isArray(body.data);
    },
  });

  if (!success) {
    errorCount.add(1);
    errorRate.add(1);
  } else {
    errorRate.add(0);
  }
  
  responseTime.add(response.timings.duration);
}

function testSearchProducts(headers) {
  const searchQueries = ['phone', 'laptop', 'book', 'shoes', 'camera'];
  const query = searchQueries[Math.floor(Math.random() * searchQueries.length)];
  
  const response = http.get(`${BASE_URL}/products?q=${query}&limit=20`, { headers });
  
  const success = check(response, {
    'Search products status is 200': (r) => r.status === 200,
    'Search products response time < 300ms': (r) => r.timings.duration < 300,
    'Search products has pagination': (r) => {
      const body = JSON.parse(r.body);
      return body.pagination && typeof body.pagination.total === 'number';
    },
  });

  if (!success) {
    errorCount.add(1);
    errorRate.add(1);
  } else {
    errorRate.add(0);
  }
  
  responseTime.add(response.timings.duration);
}

function testCreateProduct(headers) {
  const productData = {
    name: `Load Test Product ${Math.random().toString(36).substr(2, 9)}`,
    description: 'Product created during load testing',
    price: Math.floor(Math.random() * 1000) + 10,
    currency: 'USD',
    stock: Math.floor(Math.random() * 100) + 1,
    sku: `LT-${Math.random().toString(36).substr(2, 9).toUpperCase()}`,
    category_id: 'cat_electronics',
  };

  const response = http.post(`${BASE_URL}/products`, JSON.stringify(productData), { headers });
  
  const success = check(response, {
    'Create product status is 201': (r) => r.status === 201,
    'Create product response time < 500ms': (r) => r.timings.duration < 500,
    'Create product returns ID': (r) => {
      const body = JSON.parse(r.body);
      return body.data && body.data.id;
    },
  });

  if (!success) {
    errorCount.add(1);
    errorRate.add(1);
  } else {
    errorRate.add(0);
  }
  
  responseTime.add(response.timings.duration);
}

export function teardown(data) {
  // Cleanup if needed
  console.log('Load test completed');
}
```

### **Stress Test**

```javascript
// tests/load/stress_test.js
import http from 'k6/http';
import { check } from 'k6';

export const options = {
  stages: [
    { duration: '1m', target: 100 },   // Ramp up
    { duration: '2m', target: 100 },   // Stay at 100
    { duration: '1m', target: 500 },   // Spike to 500
    { duration: '2m', target: 500 },   // Stay at 500
    { duration: '1m', target: 1000 },  // Spike to 1000
    { duration: '2m', target: 1000 },  // Stay at 1000
    { duration: '1m', target: 0 },     // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<1000'], // More lenient for stress test
    http_req_failed: ['rate<0.05'],    // Allow 5% error rate
  },
};

const BASE_URL = 'http://localhost:8080/api/v1';

export default function () {
  const response = http.get(`${BASE_URL}/health`);
  
  check(response, {
    'Health check status is 200': (r) => r.status === 200,
    'Health check response time < 1s': (r) => r.timings.duration < 1000,
  });
}
```

---

## 🛡️ **SECURITY TESTING**

### **Authentication Tests**

```go
// tests/security/auth_test.go
package security_test

import (
    "bytes"
    "encoding/json"
    "net/http"
    "net/http/httptest"
    "testing"
    "time"

    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/suite"

    "backend-template/src/common/crypto/jwt"
)

type SecurityTestSuite struct {
    suite.Suite
    jwtService *jwt.Service
}

func (suite *SecurityTestSuite) SetupTest() {
    suite.jwtService = jwt.NewService(
        "test-access-secret",
        "test-refresh-secret",
        15*time.Minute,
        7*24*time.Hour,
    )
}

func (suite *SecurityTestSuite) TestJWTTokenSecurity() {
    // Test token generation
    tokens, err := suite.jwtService.GenerateTokenPair("user123", "test@example.com", "user")
    assert.NoError(suite.T(), err)
    assert.NotEmpty(suite.T(), tokens.AccessToken)
    assert.NotEmpty(suite.T(), tokens.RefreshToken)
    
    // Test token validation
    claims, err := suite.jwtService.ValidateAccessToken(tokens.AccessToken)
    assert.NoError(suite.T(), err)
    assert.Equal(suite.T(), "user123", claims.UserID)
    assert.Equal(suite.T(), "test@example.com", claims.Email)
    
    // Test expired token
    expiredService := jwt.NewService(
        "test-access-secret",
        "test-refresh-secret",
        -1*time.Hour, // Expired
        7*24*time.Hour,
    )
    
    expiredTokens, err := expiredService.GenerateTokenPair("user123", "test@example.com", "user")
    assert.NoError(suite.T(), err)
    
    _, err = suite.jwtService.ValidateAccessToken(expiredTokens.AccessToken)
    assert.Error(suite.T(), err)
    
    // Test tampered token
    tamperedToken := tokens.AccessToken + "tampered"
    _, err = suite.jwtService.ValidateAccessToken(tamperedToken)
    assert.Error(suite.T(), err)
}

func (suite *SecurityTestSuite) TestPasswordSecurity() {
    // Test password requirements
    testCases := []struct {
        password string
        valid    bool
    }{
        {"weak", false},           // Too short
        {"password", false},       // No numbers/special chars
        {"Password1", false},      // No special chars
        {"Password1!", true},      // Valid
        {"VeryLongPassword123!", true}, // Valid long password
    }
    
    for _, tc := range testCases {
        valid := validatePassword(tc.password)
        assert.Equal(suite.T(), tc.valid, valid, 
            "Password %s should be valid=%v", tc.password, tc.valid)
    }
}

func validatePassword(password string) bool {
    if len(password) < 8 {
        return false
    }
    
    hasUpper := false
    hasLower := false
    hasDigit := false
    hasSpecial := false
    
    for _, char := range password {
        switch {
        case char >= 'A' && char <= 'Z':
            hasUpper = true
        case char >= 'a' && char <= 'z':
            hasLower = true
        case char >= '0' && char <= '9':
            hasDigit = true
        case char >= 33 && char <= 126:
            hasSpecial = true
        }
    }
    
    return hasUpper && hasLower && hasDigit && hasSpecial
}

func TestSecurityTestSuite(t *testing.T) {
    suite.Run(t, new(SecurityTestSuite))
}
```

### **SQL Injection Tests**

```go
// tests/security/sql_injection_test.go
package security_test

import (
    "context"
    "testing"

    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/suite"

    "backend-template/src/core/domains"
)

type SQLInjectionTestSuite struct {
    suite.Suite
    repo domains.ProductRepository
}

func (suite *SQLInjectionTestSuite) SetupSuite() {
    // Setup test database
    // suite.repo = setupTestDatabase()
}

func (suite *SQLInjectionTestSuite) TestSQLInjectionPrevention() {
    ctx := context.Background()
    
    // Test malicious input in search
    maliciousInputs := []string{
        "'; DROP TABLE products; --",
        "' OR '1'='1",
        "' UNION SELECT * FROM users --",
        "<script>alert('xss')</script>",
        "../../etc/passwd",
    }
    
    for _, input := range maliciousInputs {
        filter := domains.ProductFilter{
            SearchQuery: input,
            Limit:       10,
        }
        
        // Should not cause errors or return unexpected results
        products, total, err := suite.repo.List(ctx, filter)
        
        // Should handle malicious input gracefully
        assert.NoError(suite.T(), err, "Malicious input should not cause errors: %s", input)
        assert.GreaterOrEqual(suite.T(), int(total), 0, "Total should be non-negative")
        assert.NotNil(suite.T(), products, "Products should not be nil")
    }
}

func TestSQLInjectionTestSuite(t *testing.T) {
    suite.Run(t, new(SQLInjectionTestSuite))
}
```

---

## 📊 **PERFORMANCE TESTING**

### **Benchmark Tests**

```go
// tests/performance/benchmark_test.go
package performance_test

import (
    "context"
    "testing"

    "backend-template/src/core/domains"
    "backend-template/src/core/usecases"
)

func BenchmarkProductUseCase_CreateProduct(b *testing.B) {
    // Setup
    useCase := setupProductUseCase()
    
    product := &domains.Product{
        Name:        "Benchmark Product",
        Description: "Product for benchmark testing",
        Price: domains.Money{
            Amount:   99.99,
            Currency: "USD",
        },
        Stock:      10,
        CategoryID: "cat_123",
        Status:     "active",
    }
    
    b.ResetTimer()
    
    // Benchmark
    for i := 0; i < b.N; i++ {
        product.SKU = fmt.Sprintf("BENCH-%d", i)
        err := useCase.CreateProduct(context.Background(), product)
        if err != nil {
            b.Fatal(err)
        }
    }
}

func BenchmarkProductUseCase_GetProduct(b *testing.B) {
    // Setup
    useCase := setupProductUseCase()
    productID := "prod_benchmark"
    
    // Create test product
    product := &domains.Product{
        ID:          productID,
        Name:        "Benchmark Product",
        SKU:         "BENCH-GET",
        CategoryID:  "cat_123",
    }
    useCase.CreateProduct(context.Background(), product)
    
    b.ResetTimer()
    
    // Benchmark
    for i := 0; i < b.N; i++ {
        _, err := useCase.GetProduct(context.Background(), productID)
        if err != nil {
            b.Fatal(err)
        }
    }
}

func BenchmarkProductUseCase_SearchProducts(b *testing.B) {
    // Setup
    useCase := setupProductUseCase()
    
    filter := domains.ProductFilter{
        SearchQuery: "benchmark",
        Limit:       20,
        Offset:      0,
    }
    
    b.ResetTimer()
    
    // Benchmark
    for i := 0; i < b.N; i++ {
        _, _, err := useCase.SearchProducts(context.Background(), filter)
        if err != nil {
            b.Fatal(err)
        }
    }
}

// Parallel benchmarks
func BenchmarkProductUseCase_GetProduct_Parallel(b *testing.B) {
    useCase := setupProductUseCase()
    productID := "prod_parallel"
    
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            _, err := useCase.GetProduct(context.Background(), productID)
            if err != nil {
                b.Fatal(err)
            }
        }
    })
}
```

---

## 📈 **TEST AUTOMATION**

### **Makefile for Testing**

```makefile
# Makefile
.PHONY: test test-unit test-integration test-api test-e2e test-load test-security test-coverage

# Run all tests
test: test-unit test-integration test-api

# Unit tests
test-unit:
	@echo "Running unit tests..."
	go test -v -race -short ./src/...

# Integration tests
test-integration:
	@echo "Running integration tests..."
	go test -v -tags=integration ./tests/integration/...

# API tests
test-api:
	@echo "Running API tests..."
	go test -v -tags=api ./tests/api/...

# End-to-end tests
test-e2e:
	@echo "Running E2E tests..."
	@echo "Starting test environment..."
	docker-compose -f docker-compose.test.yml up -d
	@sleep 10
	go test -v -tags=e2e ./tests/e2e/...
	docker-compose -f docker-compose.test.yml down

# Load tests
test-load:
	@echo "Running load tests..."
	k6 run tests/load/basic_load_test.js

# Security tests
test-security:
	@echo "Running security tests..."
	go test -v -tags=security ./tests/security/...

# Test coverage
test-coverage:
	@echo "Running tests with coverage..."
	go test -v -race -coverprofile=coverage.out ./src/...
	go tool cover -html=coverage.out -o coverage.html
	@echo "Coverage report generated: coverage.html"

# Benchmark tests
test-benchmark:
	@echo "Running benchmark tests..."
	go test -bench=. -benchmem ./tests/performance/...

# Clean test artifacts
test-clean:
	rm -f coverage.out coverage.html
	docker-compose -f docker-compose.test.yml down -v
```

### **GitHub Actions Test Workflow**

```yaml
# .github/workflows/test.yml
name: Test Suite

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Set up Go
      uses: actions/setup-go@v4
      with:
        go-version: '1.21'
    
    - name: Cache Go modules
      uses: actions/cache@v3
      with:
        path: ~/go/pkg/mod
        key: ${{ runner.os }}-go-${{ hashFiles('**/go.sum') }}
    
    - name: Run unit tests
      run: make test-unit
    
    - name: Generate coverage report
      run: make test-coverage
    
    - name: Upload coverage to Codecov
      uses: codecov/codecov-action@v3
      with:
        file: ./coverage.out

  integration-tests:
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
    
    - name: Run integration tests
      run: make test-integration
      env:
        DATABASE_URL: postgres://postgres:postgres@localhost:5432/testdb?sslmode=disable
        REDIS_URL: redis://localhost:6379

  api-tests:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Set up Go
      uses: actions/setup-go@v4
      with:
        go-version: '1.21'
    
    - name: Run API tests
      run: make test-api

  load-tests:
    runs-on: ubuntu-latest
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Install k6
      run: |
        sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys C5AD17C747E3415A3642D57D77C6C491D6AC1D69
        echo "deb https://dl.k6.io/deb stable main" | sudo tee /etc/apt/sources.list.d/k6.list
        sudo apt-get update
        sudo apt-get install k6
    
    - name: Run load tests
      run: make test-load
```

---

## 📊 **TEST REPORTING**

### **Test Report Generation**

```go
// tests/utils/reporter.go
package utils

import (
    "encoding/json"
    "fmt"
    "os"
    "time"
)

type TestReport struct {
    Timestamp    time.Time     `json:"timestamp"`
    Environment  string        `json:"environment"`
    TestSuite    string        `json:"test_suite"`
    TotalTests   int           `json:"total_tests"`
    PassedTests  int           `json:"passed_tests"`
    FailedTests  int           `json:"failed_tests"`
    SkippedTests int           `json:"skipped_tests"`
    Duration     time.Duration `json:"duration"`
    Coverage     float64       `json:"coverage"`
    Details      []TestDetail  `json:"details"`
}

type TestDetail struct {
    Name     string        `json:"name"`
    Status   string        `json:"status"`
    Duration time.Duration `json:"duration"`
    Error    string        `json:"error,omitempty"`
}

func GenerateTestReport(report TestReport) error {
    reportJSON, err := json.MarshalIndent(report, "", "  ")
    if err != nil {
        return err
    }
    
    filename := fmt.Sprintf("test-report-%s.json", 
        time.Now().Format("2006-01-02-15-04-05"))
    
    return os.WriteFile(filename, reportJSON, 0644)
}
```

---

## 🎯 **TESTING BEST PRACTICES**

### **Test Organization**
- **Arrange-Act-Assert**: Cấu trúc rõ ràng cho mọi test
- **Test Isolation**: Mỗi test độc lập, không phụ thuộc lẫn nhau
- **Descriptive Names**: Tên test mô tả rõ behavior được test
- **Single Responsibility**: Mỗi test chỉ test một behavior

### **Mock Strategy**
- **Mock External Dependencies**: Database, APIs, file system
- **Verify Interactions**: Kiểm tra calls đến dependencies
- **Use Dependency Injection**: Dễ dàng inject mocks
- **Avoid Over-Mocking**: Chỉ mock khi cần thiết

### **Test Data Management**
- **Test Fixtures**: Dữ liệu test chuẩn và reusable
- **Factory Pattern**: Tạo test data dynamically
- **Database Seeding**: Setup data cho integration tests
- **Cleanup**: Dọn dẹp data sau mỗi test

### **Performance Testing**
- **Baseline Metrics**: Establish performance baselines
- **Load Patterns**: Test realistic load patterns
- **Resource Monitoring**: Monitor CPU, memory, DB connections
- **Gradual Load Increase**: Avoid sudden load spikes

---

## 📞 **CONTINUOUS TESTING**

### **Test Pipeline Integration**
- **Pre-commit Hooks**: Run tests before commit
- **CI/CD Integration**: Automated testing in pipeline
- **Quality Gates**: Block deployment if tests fail
- **Test Environment Management**: Consistent test environments

### **Monitoring & Alerting**
- **Test Results Dashboard**: Visualize test trends
- **Flaky Test Detection**: Identify unstable tests
- **Performance Regression Alerts**: Alert on performance degradation
- **Coverage Tracking**: Monitor code coverage trends

---

**🧪 Với comprehensive testing strategy này, bạn có thể đảm bảo chất lượng code và system reliability ở mọi level! 🚀** 