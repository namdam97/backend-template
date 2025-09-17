# 🎯 DOMAIN-SPECIFIC BUSINESS LOGIC EXAMPLES
## Customization Templates for Different Industries

> **Mục tiêu**: Cung cấp templates và examples cụ thể cho các domain khác nhau để customize 10% business logic từ 90% infrastructure có sẵn.

---

## 📋 **CUSTOMIZATION WORKFLOW**

### **Bước 1: Chọn Domain Template**
1. E-commerce Platform
2. Financial Services  
3. Healthcare System
4. SaaS Platform
5. IoT Platform
6. Social Media Platform

### **Bước 2: Copy & Customize**
```bash
# Copy domain template
cp domains/ecommerce/* src/core/domains/
cp domains/ecommerce/usecases/* src/core/usecases/
cp domains/ecommerce/controllers/* src/present/http/controllers/

# Update module imports
find . -name "*.go" -exec sed -i 's/template-module/your-module/g' {} \;
```

### **Bước 3: Database Setup**
```bash
# Run migrations for your domain
migrate -path migrations/ecommerce -database "postgres://..." up
```

---

## 🛒 **1. E-COMMERCE PLATFORM**

### **Domain Models**
```go
// src/core/domains/product.go
package domains

import (
    "context"
    "time"
)

type Product struct {
    ID          string    `json:"id" db:"id"`
    Name        string    `json:"name" db:"name" validate:"required,min=3,max=100"`
    Description string    `json:"description" db:"description" validate:"max=1000"`
    Price       Money     `json:"price" db:"price" validate:"required,min=0"`
    Stock       int       `json:"stock" db:"stock" validate:"min=0"`
    SKU         string    `json:"sku" db:"sku" validate:"required"`
    CategoryID  string    `json:"category_id" db:"category_id" validate:"required"`
    Images      []string  `json:"images" db:"images"`
    Status      string    `json:"status" db:"status" validate:"oneof=active inactive"`
    CreatedAt   time.Time `json:"created_at" db:"created_at"`
    UpdatedAt   time.Time `json:"updated_at" db:"updated_at"`
}

type Money struct {
    Amount   float64 `json:"amount"`
    Currency string  `json:"currency"`
}

type Category struct {
    ID          string    `json:"id" db:"id"`
    Name        string    `json:"name" db:"name" validate:"required"`
    Description string    `json:"description" db:"description"`
    ParentID    *string   `json:"parent_id" db:"parent_id"`
    CreatedAt   time.Time `json:"created_at" db:"created_at"`
}

type ProductRepository interface {
    Create(ctx context.Context, product *Product) error
    GetByID(ctx context.Context, id string) (*Product, error)
    GetBySKU(ctx context.Context, sku string) (*Product, error)
    List(ctx context.Context, filter ProductFilter) ([]*Product, int64, error)
    Update(ctx context.Context, product *Product) error
    Delete(ctx context.Context, id string) error
    UpdateStock(ctx context.Context, id string, quantity int) error
    GetByCategory(ctx context.Context, categoryID string, limit, offset int) ([]*Product, error)
}

type ProductFilter struct {
    CategoryID   string
    MinPrice     float64
    MaxPrice     float64
    Status       string
    SearchQuery  string
    SortBy       string
    SortOrder    string
    Limit        int
    Offset       int
}
```

```go
// src/core/domains/order.go
package domains

import (
    "context"
    "time"
)

type Order struct {
    ID           string      `json:"id" db:"id"`
    CustomerID   string      `json:"customer_id" db:"customer_id" validate:"required"`
    Items        []OrderItem `json:"items" validate:"required,min=1"`
    TotalAmount  Money       `json:"total_amount" db:"total_amount"`
    Status       OrderStatus `json:"status" db:"status"`
    ShippingInfo Address     `json:"shipping_info" db:"shipping_info"`
    PaymentInfo  Payment     `json:"payment_info" db:"payment_info"`
    Notes        string      `json:"notes" db:"notes"`
    CreatedAt    time.Time   `json:"created_at" db:"created_at"`
    UpdatedAt    time.Time   `json:"updated_at" db:"updated_at"`
}

type OrderItem struct {
    ProductID string  `json:"product_id" validate:"required"`
    Quantity  int     `json:"quantity" validate:"required,min=1"`
    Price     Money   `json:"price" validate:"required"`
    Total     Money   `json:"total"`
}

type OrderStatus string

const (
    OrderStatusPending   OrderStatus = "pending"
    OrderStatusConfirmed OrderStatus = "confirmed"
    OrderStatusPaid      OrderStatus = "paid"
    OrderStatusShipped   OrderStatus = "shipped"
    OrderStatusDelivered OrderStatus = "delivered"
    OrderStatusCancelled OrderStatus = "cancelled"
)

type Address struct {
    Street   string `json:"street" validate:"required"`
    City     string `json:"city" validate:"required"`
    State    string `json:"state" validate:"required"`
    ZipCode  string `json:"zip_code" validate:"required"`
    Country  string `json:"country" validate:"required"`
}

type Payment struct {
    Method        string    `json:"method" validate:"required,oneof=card paypal bank_transfer"`
    TransactionID string    `json:"transaction_id"`
    Status        string    `json:"status"`
    ProcessedAt   time.Time `json:"processed_at"`
}

type OrderRepository interface {
    Create(ctx context.Context, order *Order) error
    GetByID(ctx context.Context, id string) (*Order, error)
    GetByCustomerID(ctx context.Context, customerID string, limit, offset int) ([]*Order, error)
    UpdateStatus(ctx context.Context, id string, status OrderStatus) error
    List(ctx context.Context, filter OrderFilter) ([]*Order, int64, error)
    GetOrderStats(ctx context.Context, from, to time.Time) (*OrderStats, error)
}

type OrderFilter struct {
    CustomerID string
    Status     OrderStatus
    DateFrom   time.Time
    DateTo     time.Time
    Limit      int
    Offset     int
}

type OrderStats struct {
    TotalOrders    int64   `json:"total_orders"`
    TotalRevenue   Money   `json:"total_revenue"`
    AverageValue   Money   `json:"average_value"`
    TopProducts    []ProductStat `json:"top_products"`
}

type ProductStat struct {
    ProductID string `json:"product_id"`
    Name      string `json:"name"`
    Quantity  int    `json:"quantity"`
    Revenue   Money  `json:"revenue"`
}
```

### **Use Cases**
```go
// src/core/usecases/product_usecase.go
package usecases

import (
    "context"
    "fmt"
    "time"
    
    "your-module/src/core/domains"
    "your-module/src/common/logger"
)

type ProductUseCase struct {
    productRepo  domains.ProductRepository
    categoryRepo domains.CategoryRepository
    cache        cache.Cache
    logger       logger.Logger
    metrics      *metrics.MetricsService
}

func NewProductUseCase(
    productRepo domains.ProductRepository,
    categoryRepo domains.CategoryRepository,
    cache cache.Cache,
    logger logger.Logger,
    metrics *metrics.MetricsService,
) *ProductUseCase {
    return &ProductUseCase{
        productRepo:  productRepo,
        categoryRepo: categoryRepo,
        cache:        cache,
        logger:       logger,
        metrics:      metrics,
    }
}

func (uc *ProductUseCase) CreateProduct(ctx context.Context, product *domains.Product) error {
    start := time.Now()
    defer func() {
        uc.metrics.RecordBusinessOperation("product", "create", time.Since(start), true)
    }()
    
    // Business validation
    if err := uc.validateProduct(ctx, product); err != nil {
        return fmt.Errorf("product validation failed: %w", err)
    }
    
    // Check if SKU already exists
    existingProduct, err := uc.productRepo.GetBySKU(ctx, product.SKU)
    if err == nil && existingProduct != nil {
        return fmt.Errorf("product with SKU %s already exists", product.SKU)
    }
    
    // Generate ID
    product.ID = generateProductID()
    product.CreatedAt = time.Now()
    product.UpdatedAt = time.Now()
    
    // Save to repository
    if err := uc.productRepo.Create(ctx, product); err != nil {
        uc.logger.Error(ctx, err, "Failed to create product")
        return fmt.Errorf("failed to create product: %w", err)
    }
    
    // Invalidate cache
    uc.invalidateProductCache(ctx, product.CategoryID)
    
    uc.logger.Info(ctx, "Product created successfully", 
        logger.String("product_id", product.ID),
        logger.String("sku", product.SKU))
    
    return nil
}

func (uc *ProductUseCase) GetProduct(ctx context.Context, id string) (*domains.Product, error) {
    // Try cache first
    cacheKey := fmt.Sprintf("product:%s", id)
    if cached, err := uc.cache.Get(ctx, cacheKey); err == nil {
        if product, ok := cached.(*domains.Product); ok {
            return product, nil
        }
    }
    
    // Get from repository
    product, err := uc.productRepo.GetByID(ctx, id)
    if err != nil {
        uc.logger.Error(ctx, err, "Failed to get product", 
            logger.String("product_id", id))
        return nil, fmt.Errorf("product not found: %w", err)
    }
    
    // Cache the result
    go uc.cache.Set(context.Background(), cacheKey, product, 1*time.Hour)
    
    return product, nil
}

func (uc *ProductUseCase) SearchProducts(ctx context.Context, filter domains.ProductFilter) ([]*domains.Product, int64, error) {
    // Validate filter
    if filter.Limit <= 0 || filter.Limit > 100 {
        filter.Limit = 20
    }
    
    products, total, err := uc.productRepo.List(ctx, filter)
    if err != nil {
        uc.logger.Error(ctx, err, "Failed to search products")
        return nil, 0, fmt.Errorf("failed to search products: %w", err)
    }
    
    uc.logger.Info(ctx, "Products searched successfully",
        logger.Int("count", len(products)),
        logger.Int64("total", total))
    
    return products, total, nil
}

func (uc *ProductUseCase) UpdateStock(ctx context.Context, productID string, quantity int) error {
    if quantity < 0 {
        return fmt.Errorf("stock quantity cannot be negative")
    }
    
    if err := uc.productRepo.UpdateStock(ctx, productID, quantity); err != nil {
        uc.logger.Error(ctx, err, "Failed to update stock")
        return fmt.Errorf("failed to update stock: %w", err)
    }
    
    // Invalidate cache
    cacheKey := fmt.Sprintf("product:%s", productID)
    uc.cache.Delete(ctx, cacheKey)
    
    return nil
}

func (uc *ProductUseCase) validateProduct(ctx context.Context, product *domains.Product) error {
    if product.Price.Amount <= 0 {
        return fmt.Errorf("product price must be greater than 0")
    }
    
    if product.Stock < 0 {
        return fmt.Errorf("product stock cannot be negative")
    }
    
    // Validate category exists
    if _, err := uc.categoryRepo.GetByID(ctx, product.CategoryID); err != nil {
        return fmt.Errorf("invalid category ID: %s", product.CategoryID)
    }
    
    return nil
}

func generateProductID() string {
    return "prod_" + generateRandomString(10)
}
```

```go
// src/core/usecases/order_usecase.go
package usecases

import (
    "context"
    "fmt"
    "time"
)

type OrderUseCase struct {
    orderRepo    domains.OrderRepository
    productRepo  domains.ProductRepository
    customerRepo domains.CustomerRepository
    paymentSvc   PaymentService
    eventBus     EventBus
    logger       logger.Logger
}

func NewOrderUseCase(
    orderRepo domains.OrderRepository,
    productRepo domains.ProductRepository,
    customerRepo domains.CustomerRepository,
    paymentSvc PaymentService,
    eventBus EventBus,
    logger logger.Logger,
) *OrderUseCase {
    return &OrderUseCase{
        orderRepo:    orderRepo,
        productRepo:  productRepo,
        customerRepo: customerRepo,
        paymentSvc:   paymentSvc,
        eventBus:     eventBus,
        logger:       logger,
    }
}

func (uc *OrderUseCase) CreateOrder(ctx context.Context, order *domains.Order) error {
    // Validate customer exists
    if _, err := uc.customerRepo.GetByID(ctx, order.CustomerID); err != nil {
        return fmt.Errorf("invalid customer ID: %s", order.CustomerID)
    }
    
    // Validate and calculate order
    if err := uc.validateAndCalculateOrder(ctx, order); err != nil {
        return fmt.Errorf("order validation failed: %w", err)
    }
    
    // Check stock availability
    if err := uc.checkStockAvailability(ctx, order.Items); err != nil {
        return fmt.Errorf("stock check failed: %w", err)
    }
    
    // Generate order ID
    order.ID = generateOrderID()
    order.Status = domains.OrderStatusPending
    order.CreatedAt = time.Now()
    order.UpdatedAt = time.Now()
    
    // Create order
    if err := uc.orderRepo.Create(ctx, order); err != nil {
        uc.logger.Error(ctx, err, "Failed to create order")
        return fmt.Errorf("failed to create order: %w", err)
    }
    
    // Reserve stock
    if err := uc.reserveStock(ctx, order.Items); err != nil {
        // Rollback order creation
        uc.orderRepo.Delete(ctx, order.ID)
        return fmt.Errorf("failed to reserve stock: %w", err)
    }
    
    // Publish order created event
    event := &OrderCreatedEvent{
        OrderID:    order.ID,
        CustomerID: order.CustomerID,
        Amount:     order.TotalAmount,
        CreatedAt:  order.CreatedAt,
    }
    uc.eventBus.Publish(ctx, "order.created", event)
    
    uc.logger.Info(ctx, "Order created successfully",
        logger.String("order_id", order.ID),
        logger.String("customer_id", order.CustomerID))
    
    return nil
}

func (uc *OrderUseCase) ProcessPayment(ctx context.Context, orderID string, paymentInfo domains.Payment) error {
    order, err := uc.orderRepo.GetByID(ctx, orderID)
    if err != nil {
        return fmt.Errorf("order not found: %w", err)
    }
    
    if order.Status != domains.OrderStatusPending {
        return fmt.Errorf("order is not in pending status")
    }
    
    // Process payment
    result, err := uc.paymentSvc.ProcessPayment(ctx, PaymentRequest{
        OrderID: orderID,
        Amount:  order.TotalAmount,
        Method:  paymentInfo.Method,
    })
    if err != nil {
        return fmt.Errorf("payment processing failed: %w", err)
    }
    
    // Update order status
    if result.Success {
        order.Status = domains.OrderStatusPaid
        order.PaymentInfo = domains.Payment{
            Method:        paymentInfo.Method,
            TransactionID: result.TransactionID,
            Status:        "completed",
            ProcessedAt:   time.Now(),
        }
    } else {
        order.Status = domains.OrderStatusCancelled
        // Release reserved stock
        uc.releaseStock(ctx, order.Items)
    }
    
    order.UpdatedAt = time.Now()
    
    if err := uc.orderRepo.UpdateStatus(ctx, orderID, order.Status); err != nil {
        return fmt.Errorf("failed to update order status: %w", err)
    }
    
    // Publish payment event
    event := &OrderPaymentProcessedEvent{
        OrderID:       orderID,
        Success:       result.Success,
        TransactionID: result.TransactionID,
        ProcessedAt:   time.Now(),
    }
    uc.eventBus.Publish(ctx, "order.payment_processed", event)
    
    return nil
}

func (uc *OrderUseCase) validateAndCalculateOrder(ctx context.Context, order *domains.Order) error {
    if len(order.Items) == 0 {
        return fmt.Errorf("order must have at least one item")
    }
    
    var total float64
    for i, item := range order.Items {
        // Get product to verify price
        product, err := uc.productRepo.GetByID(ctx, item.ProductID)
        if err != nil {
            return fmt.Errorf("product not found: %s", item.ProductID)
        }
        
        // Verify price
        if item.Price.Amount != product.Price.Amount {
            return fmt.Errorf("price mismatch for product %s", item.ProductID)
        }
        
        // Calculate item total
        order.Items[i].Total = domains.Money{
            Amount:   item.Price.Amount * float64(item.Quantity),
            Currency: item.Price.Currency,
        }
        
        total += order.Items[i].Total.Amount
    }
    
    order.TotalAmount = domains.Money{
        Amount:   total,
        Currency: "USD", // Default currency
    }
    
    return nil
}

func generateOrderID() string {
    return "order_" + generateRandomString(10)
}
```

### **Controllers**
```go
// src/present/http/controllers/product_controller.go
package controllers

import (
    "net/http"
    "strconv"
    
    "github.com/gin-gonic/gin"
    "github.com/go-playground/validator/v10"
    
    "your-module/src/core/domains"
    "your-module/src/core/usecases"
    "your-module/src/common/logger"
)

type ProductController struct {
    productUC *usecases.ProductUseCase
    validator *validator.Validate
    logger    logger.Logger
}

func NewProductController(
    productUC *usecases.ProductUseCase,
    logger logger.Logger,
) *ProductController {
    return &ProductController{
        productUC: productUC,
        validator: validator.New(),
        logger:    logger,
    }
}

type CreateProductRequest struct {
    Name        string   `json:"name" validate:"required,min=3,max=100"`
    Description string   `json:"description" validate:"max=1000"`
    Price       float64  `json:"price" validate:"required,min=0"`
    Currency    string   `json:"currency" validate:"required,oneof=USD EUR GBP"`
    Stock       int      `json:"stock" validate:"min=0"`
    SKU         string   `json:"sku" validate:"required"`
    CategoryID  string   `json:"category_id" validate:"required"`
    Images      []string `json:"images"`
}

type ProductResponse struct {
    ID          string    `json:"id"`
    Name        string    `json:"name"`
    Description string    `json:"description"`
    Price       domains.Money `json:"price"`
    Stock       int       `json:"stock"`
    SKU         string    `json:"sku"`
    CategoryID  string    `json:"category_id"`
    Images      []string  `json:"images"`
    Status      string    `json:"status"`
    CreatedAt   string    `json:"created_at"`
    UpdatedAt   string    `json:"updated_at"`
}

func (ctrl *ProductController) CreateProduct(c *gin.Context) {
    var req CreateProductRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        ctrl.logger.Error(c.Request.Context(), err, "Invalid request body")
        c.JSON(http.StatusBadRequest, gin.H{
            "error": "Invalid request body",
            "details": err.Error(),
        })
        return
    }
    
    if err := ctrl.validator.Struct(&req); err != nil {
        ctrl.logger.Error(c.Request.Context(), err, "Request validation failed")
        c.JSON(http.StatusBadRequest, gin.H{
            "error": "Validation failed",
            "details": err.Error(),
        })
        return
    }
    
    product := &domains.Product{
        Name:        req.Name,
        Description: req.Description,
        Price: domains.Money{
            Amount:   req.Price,
            Currency: req.Currency,
        },
        Stock:      req.Stock,
        SKU:        req.SKU,
        CategoryID: req.CategoryID,
        Images:     req.Images,
        Status:     "active",
    }
    
    if err := ctrl.productUC.CreateProduct(c.Request.Context(), product); err != nil {
        ctrl.logger.Error(c.Request.Context(), err, "Failed to create product")
        c.JSON(http.StatusInternalServerError, gin.H{
            "error": "Failed to create product",
            "details": err.Error(),
        })
        return
    }
    
    response := &ProductResponse{
        ID:          product.ID,
        Name:        product.Name,
        Description: product.Description,
        Price:       product.Price,
        Stock:       product.Stock,
        SKU:         product.SKU,
        CategoryID:  product.CategoryID,
        Images:      product.Images,
        Status:      product.Status,
        CreatedAt:   product.CreatedAt.Format("2006-01-02T15:04:05Z"),
        UpdatedAt:   product.UpdatedAt.Format("2006-01-02T15:04:05Z"),
    }
    
    c.JSON(http.StatusCreated, gin.H{
        "message": "Product created successfully",
        "data":    response,
    })
}

func (ctrl *ProductController) GetProduct(c *gin.Context) {
    id := c.Param("id")
    if id == "" {
        c.JSON(http.StatusBadRequest, gin.H{"error": "Product ID is required"})
        return
    }
    
    product, err := ctrl.productUC.GetProduct(c.Request.Context(), id)
    if err != nil {
        ctrl.logger.Error(c.Request.Context(), err, "Failed to get product")
        c.JSON(http.StatusNotFound, gin.H{"error": "Product not found"})
        return
    }
    
    response := &ProductResponse{
        ID:          product.ID,
        Name:        product.Name,
        Description: product.Description,
        Price:       product.Price,
        Stock:       product.Stock,
        SKU:         product.SKU,
        CategoryID:  product.CategoryID,
        Images:      product.Images,
        Status:      product.Status,
        CreatedAt:   product.CreatedAt.Format("2006-01-02T15:04:05Z"),
        UpdatedAt:   product.UpdatedAt.Format("2006-01-02T15:04:05Z"),
    }
    
    c.JSON(http.StatusOK, gin.H{"data": response})
}

func (ctrl *ProductController) SearchProducts(c *gin.Context) {
    var filter domains.ProductFilter
    
    // Parse query parameters
    if categoryID := c.Query("category_id"); categoryID != "" {
        filter.CategoryID = categoryID
    }
    
    if minPrice := c.Query("min_price"); minPrice != "" {
        if price, err := strconv.ParseFloat(minPrice, 64); err == nil {
            filter.MinPrice = price
        }
    }
    
    if maxPrice := c.Query("max_price"); maxPrice != "" {
        if price, err := strconv.ParseFloat(maxPrice, 64); err == nil {
            filter.MaxPrice = price
        }
    }
    
    if status := c.Query("status"); status != "" {
        filter.Status = status
    }
    
    if query := c.Query("q"); query != "" {
        filter.SearchQuery = query
    }
    
    if sortBy := c.Query("sort_by"); sortBy != "" {
        filter.SortBy = sortBy
    }
    
    if sortOrder := c.Query("sort_order"); sortOrder != "" {
        filter.SortOrder = sortOrder
    }
    
    if limit := c.Query("limit"); limit != "" {
        if l, err := strconv.Atoi(limit); err == nil && l > 0 {
            filter.Limit = l
        }
    }
    if filter.Limit == 0 {
        filter.Limit = 20
    }
    
    if offset := c.Query("offset"); offset != "" {
        if o, err := strconv.Atoi(offset); err == nil && o >= 0 {
            filter.Offset = o
        }
    }
    
    products, total, err := ctrl.productUC.SearchProducts(c.Request.Context(), filter)
    if err != nil {
        ctrl.logger.Error(c.Request.Context(), err, "Failed to search products")
        c.JSON(http.StatusInternalServerError, gin.H{"error": "Failed to search products"})
        return
    }
    
    var responses []ProductResponse
    for _, product := range products {
        responses = append(responses, ProductResponse{
            ID:          product.ID,
            Name:        product.Name,
            Description: product.Description,
            Price:       product.Price,
            Stock:       product.Stock,
            SKU:         product.SKU,
            CategoryID:  product.CategoryID,
            Images:      product.Images,
            Status:      product.Status,
            CreatedAt:   product.CreatedAt.Format("2006-01-02T15:04:05Z"),
            UpdatedAt:   product.UpdatedAt.Format("2006-01-02T15:04:05Z"),
        })
    }
    
    c.JSON(http.StatusOK, gin.H{
        "data": responses,
        "pagination": gin.H{
            "total":  total,
            "limit":  filter.Limit,
            "offset": filter.Offset,
        },
    })
}
```

### **Database Migrations**
```sql
-- migrations/001_create_products.up.sql
CREATE TABLE IF NOT EXISTS products (
    id VARCHAR(255) PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    price_amount DECIMAL(10,2) NOT NULL,
    price_currency VARCHAR(3) NOT NULL DEFAULT 'USD',
    stock INTEGER NOT NULL DEFAULT 0,
    sku VARCHAR(255) UNIQUE NOT NULL,
    category_id VARCHAR(255) NOT NULL,
    images TEXT[], -- PostgreSQL array
    status VARCHAR(50) NOT NULL DEFAULT 'active',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_products_category_id ON products(category_id);
CREATE INDEX idx_products_sku ON products(sku);
CREATE INDEX idx_products_status ON products(status);
CREATE INDEX idx_products_price ON products(price_amount);

-- migrations/002_create_orders.up.sql
CREATE TABLE IF NOT EXISTS orders (
    id VARCHAR(255) PRIMARY KEY,
    customer_id VARCHAR(255) NOT NULL,
    total_amount_amount DECIMAL(10,2) NOT NULL,
    total_amount_currency VARCHAR(3) NOT NULL DEFAULT 'USD',
    status VARCHAR(50) NOT NULL,
    shipping_info JSONB,
    payment_info JSONB,
    notes TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS order_items (
    id SERIAL PRIMARY KEY,
    order_id VARCHAR(255) NOT NULL REFERENCES orders(id),
    product_id VARCHAR(255) NOT NULL,
    quantity INTEGER NOT NULL,
    price_amount DECIMAL(10,2) NOT NULL,
    price_currency VARCHAR(3) NOT NULL DEFAULT 'USD',
    total_amount DECIMAL(10,2) NOT NULL,
    total_currency VARCHAR(3) NOT NULL DEFAULT 'USD'
);

CREATE INDEX idx_orders_customer_id ON orders(customer_id);
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_orders_created_at ON orders(created_at);
CREATE INDEX idx_order_items_order_id ON order_items(order_id);
```

---

## 💰 **2. FINANCIAL SERVICES**

### **Domain Models**
```go
// src/core/domains/account.go
package domains

import (
    "context"
    "time"
)

type Account struct {
    ID            string      `json:"id" db:"id"`
    CustomerID    string      `json:"customer_id" db:"customer_id" validate:"required"`
    AccountNumber string      `json:"account_number" db:"account_number" validate:"required"`
    Type          AccountType `json:"type" db:"type" validate:"required"`
    Balance       Money       `json:"balance" db:"balance"`
    Currency      string      `json:"currency" db:"currency" validate:"required"`
    Status        string      `json:"status" db:"status" validate:"oneof=active inactive frozen closed"`
    OpenedAt      time.Time   `json:"opened_at" db:"opened_at"`
    CreatedAt     time.Time   `json:"created_at" db:"created_at"`
    UpdatedAt     time.Time   `json:"updated_at" db:"updated_at"`
}

type AccountType string

const (
    AccountTypeChecking AccountType = "checking"
    AccountTypeSavings  AccountType = "savings"
    AccountTypeCredit   AccountType = "credit"
    AccountTypeLoan     AccountType = "loan"
)

type Transaction struct {
    ID              string            `json:"id" db:"id"`
    FromAccountID   *string           `json:"from_account_id" db:"from_account_id"`
    ToAccountID     *string           `json:"to_account_id" db:"to_account_id"`
    Amount          Money             `json:"amount" db:"amount" validate:"required"`
    Type            TransactionType   `json:"type" db:"type" validate:"required"`
    Status          TransactionStatus `json:"status" db:"status"`
    Description     string            `json:"description" db:"description"`
    Reference       string            `json:"reference" db:"reference"`
    Metadata        map[string]string `json:"metadata" db:"metadata"`
    ProcessedAt     *time.Time        `json:"processed_at" db:"processed_at"`
    CreatedAt       time.Time         `json:"created_at" db:"created_at"`
}

type TransactionType string

const (
    TransactionTypeDeposit    TransactionType = "deposit"
    TransactionTypeWithdrawal TransactionType = "withdrawal"
    TransactionTypeTransfer   TransactionType = "transfer"
    TransactionTypePayment    TransactionType = "payment"
    TransactionTypeFee        TransactionType = "fee"
    TransactionTypeInterest   TransactionType = "interest"
)

type TransactionStatus string

const (
    TransactionStatusPending   TransactionStatus = "pending"
    TransactionStatusProcessed TransactionStatus = "processed"
    TransactionStatusFailed    TransactionStatus = "failed"
    TransactionStatusCancelled TransactionStatus = "cancelled"
)

type AccountRepository interface {
    Create(ctx context.Context, account *Account) error
    GetByID(ctx context.Context, id string) (*Account, error)
    GetByAccountNumber(ctx context.Context, accountNumber string) (*Account, error)
    GetByCustomerID(ctx context.Context, customerID string) ([]*Account, error)
    UpdateBalance(ctx context.Context, id string, balance Money) error
    UpdateStatus(ctx context.Context, id string, status string) error
    List(ctx context.Context, filter AccountFilter) ([]*Account, int64, error)
}

type TransactionRepository interface {
    Create(ctx context.Context, transaction *Transaction) error
    GetByID(ctx context.Context, id string) (*Transaction, error)
    GetByAccountID(ctx context.Context, accountID string, limit, offset int) ([]*Transaction, error)
    UpdateStatus(ctx context.Context, id string, status TransactionStatus) error
    List(ctx context.Context, filter TransactionFilter) ([]*Transaction, int64, error)
    GetAccountBalance(ctx context.Context, accountID string) (Money, error)
}

type AccountFilter struct {
    CustomerID string
    Type       AccountType
    Status     string
    Limit      int
    Offset     int
}

type TransactionFilter struct {
    AccountID   string
    Type        TransactionType
    Status      TransactionStatus
    DateFrom    time.Time
    DateTo      time.Time
    MinAmount   float64
    MaxAmount   float64
    Limit       int
    Offset      int
}
```

### **Use Cases**
```go
// src/core/usecases/account_usecase.go
package usecases

import (
    "context"
    "fmt"
    "time"
)

type AccountUseCase struct {
    accountRepo     domains.AccountRepository
    transactionRepo domains.TransactionRepository
    customerRepo    domains.CustomerRepository
    eventBus        EventBus
    logger          logger.Logger
    metrics         *metrics.MetricsService
}

func NewAccountUseCase(
    accountRepo domains.AccountRepository,
    transactionRepo domains.TransactionRepository,
    customerRepo domains.CustomerRepository,
    eventBus EventBus,
    logger logger.Logger,
    metrics *metrics.MetricsService,
) *AccountUseCase {
    return &AccountUseCase{
        accountRepo:     accountRepo,
        transactionRepo: transactionRepo,
        customerRepo:    customerRepo,
        eventBus:        eventBus,
        logger:          logger,
        metrics:         metrics,
    }
}

func (uc *AccountUseCase) CreateAccount(ctx context.Context, account *domains.Account) error {
    // Validate customer exists
    customer, err := uc.customerRepo.GetByID(ctx, account.CustomerID)
    if err != nil {
        return fmt.Errorf("invalid customer ID: %w", err)
    }
    
    // Business rules validation
    if err := uc.validateAccountCreation(ctx, account, customer); err != nil {
        return fmt.Errorf("account validation failed: %w", err)
    }
    
    // Generate account number
    account.AccountNumber = uc.generateAccountNumber(account.Type)
    account.ID = generateAccountID()
    account.Status = "active"
    account.Balance = domains.Money{Amount: 0, Currency: account.Currency}
    account.OpenedAt = time.Now()
    account.CreatedAt = time.Now()
    account.UpdatedAt = time.Now()
    
    // Create account
    if err := uc.accountRepo.Create(ctx, account); err != nil {
        uc.logger.Error(ctx, err, "Failed to create account")
        return fmt.Errorf("failed to create account: %w", err)
    }
    
    // Publish account created event
    event := &AccountCreatedEvent{
        AccountID:     account.ID,
        CustomerID:    account.CustomerID,
        AccountNumber: account.AccountNumber,
        Type:          account.Type,
        CreatedAt:     account.CreatedAt,
    }
    uc.eventBus.Publish(ctx, "account.created", event)
    
    uc.logger.Info(ctx, "Account created successfully",
        logger.String("account_id", account.ID),
        logger.String("customer_id", account.CustomerID),
        logger.String("account_number", account.AccountNumber))
    
    return nil
}

func (uc *AccountUseCase) Transfer(ctx context.Context, fromAccountID, toAccountID string, amount domains.Money, description string) error {
    // Validate accounts
    fromAccount, err := uc.accountRepo.GetByID(ctx, fromAccountID)
    if err != nil {
        return fmt.Errorf("from account not found: %w", err)
    }
    
    toAccount, err := uc.accountRepo.GetByID(ctx, toAccountID)
    if err != nil {
        return fmt.Errorf("to account not found: %w", err)
    }
    
    // Business rules validation
    if err := uc.validateTransfer(ctx, fromAccount, toAccount, amount); err != nil {
        return fmt.Errorf("transfer validation failed: %w", err)
    }
    
    // Create transaction
    transaction := &domains.Transaction{
        ID:            generateTransactionID(),
        FromAccountID: &fromAccountID,
        ToAccountID:   &toAccountID,
        Amount:        amount,
        Type:          domains.TransactionTypeTransfer,
        Status:        domains.TransactionStatusPending,
        Description:   description,
        CreatedAt:     time.Now(),
    }
    
    if err := uc.transactionRepo.Create(ctx, transaction); err != nil {
        return fmt.Errorf("failed to create transaction: %w", err)
    }
    
    // Process transfer (this could be async)
    if err := uc.processTransfer(ctx, transaction, fromAccount, toAccount); err != nil {
        // Update transaction status to failed
        uc.transactionRepo.UpdateStatus(ctx, transaction.ID, domains.TransactionStatusFailed)
        return fmt.Errorf("transfer processing failed: %w", err)
    }
    
    return nil
}

func (uc *AccountUseCase) processTransfer(ctx context.Context, transaction *domains.Transaction, fromAccount, toAccount *domains.Account) error {
    // Update balances
    newFromBalance := domains.Money{
        Amount:   fromAccount.Balance.Amount - transaction.Amount.Amount,
        Currency: fromAccount.Balance.Currency,
    }
    
    newToBalance := domains.Money{
        Amount:   toAccount.Balance.Amount + transaction.Amount.Amount,
        Currency: toAccount.Balance.Currency,
    }
    
    // Update from account
    if err := uc.accountRepo.UpdateBalance(ctx, fromAccount.ID, newFromBalance); err != nil {
        return fmt.Errorf("failed to update from account balance: %w", err)
    }
    
    // Update to account
    if err := uc.accountRepo.UpdateBalance(ctx, toAccount.ID, newToBalance); err != nil {
        // Rollback from account balance
        uc.accountRepo.UpdateBalance(ctx, fromAccount.ID, fromAccount.Balance)
        return fmt.Errorf("failed to update to account balance: %w", err)
    }
    
    // Update transaction status
    processedAt := time.Now()
    transaction.ProcessedAt = &processedAt
    transaction.Status = domains.TransactionStatusProcessed
    
    if err := uc.transactionRepo.UpdateStatus(ctx, transaction.ID, domains.TransactionStatusProcessed); err != nil {
        uc.logger.Error(ctx, err, "Failed to update transaction status")
    }
    
    // Publish transfer completed event
    event := &TransferCompletedEvent{
        TransactionID: transaction.ID,
        FromAccountID: *transaction.FromAccountID,
        ToAccountID:   *transaction.ToAccountID,
        Amount:        transaction.Amount,
        ProcessedAt:   processedAt,
    }
    uc.eventBus.Publish(ctx, "transfer.completed", event)
    
    uc.logger.Info(ctx, "Transfer completed successfully",
        logger.String("transaction_id", transaction.ID),
        logger.String("from_account", *transaction.FromAccountID),
        logger.String("to_account", *transaction.ToAccountID),
        logger.Float64("amount", transaction.Amount.Amount))
    
    return nil
}

func (uc *AccountUseCase) validateTransfer(ctx context.Context, fromAccount, toAccount *domains.Account, amount domains.Money) error {
    // Check account status
    if fromAccount.Status != "active" {
        return fmt.Errorf("from account is not active")
    }
    
    if toAccount.Status != "active" {
        return fmt.Errorf("to account is not active")
    }
    
    // Check currency match
    if fromAccount.Currency != amount.Currency {
        return fmt.Errorf("currency mismatch: account currency %s, transfer currency %s", 
            fromAccount.Currency, amount.Currency)
    }
    
    // Check sufficient balance
    if fromAccount.Balance.Amount < amount.Amount {
        return fmt.Errorf("insufficient balance: available %f, requested %f", 
            fromAccount.Balance.Amount, amount.Amount)
    }
    
    // Check minimum transfer amount
    if amount.Amount <= 0 {
        return fmt.Errorf("transfer amount must be positive")
    }
    
    // Check maximum transfer limit (business rule)
    maxTransferLimit := 10000.0 // $10,000
    if amount.Amount > maxTransferLimit {
        return fmt.Errorf("transfer amount exceeds maximum limit of %f", maxTransferLimit)
    }
    
    return nil
}

func (uc *AccountUseCase) generateAccountNumber(accountType domains.AccountType) string {
    prefix := map[domains.AccountType]string{
        domains.AccountTypeChecking: "CHK",
        domains.AccountTypeSavings:  "SAV",
        domains.AccountTypeCredit:   "CRD",
        domains.AccountTypeLoan:     "LON",
    }
    
    return prefix[accountType] + generateRandomString(10)
}
```

---

## 🏥 **3. HEALTHCARE SYSTEM**

### **Domain Models**
```go
// src/core/domains/patient.go
package domains

import (
    "context"
    "time"
)

type Patient struct {
    ID             string         `json:"id" db:"id"`
    PersonalInfo   PersonalInfo   `json:"personal_info" db:"personal_info"`
    MedicalInfo    MedicalHistory `json:"medical_info" db:"medical_info"`
    Insurance      InsuranceInfo  `json:"insurance" db:"insurance"`
    EmergencyContact Contact      `json:"emergency_contact" db:"emergency_contact"`
    Status         string         `json:"status" db:"status" validate:"oneof=active inactive"`
    CreatedAt      time.Time      `json:"created_at" db:"created_at"`
    UpdatedAt      time.Time      `json:"updated_at" db:"updated_at"`
}

type PersonalInfo struct {
    FirstName   string    `json:"first_name" validate:"required"`
    LastName    string    `json:"last_name" validate:"required"`
    DateOfBirth time.Time `json:"date_of_birth" validate:"required"`
    Gender      string    `json:"gender" validate:"oneof=male female other"`
    Phone       string    `json:"phone" validate:"required"`
    Email       string    `json:"email" validate:"email"`
    Address     Address   `json:"address"`
}

type MedicalHistory struct {
    BloodType     string              `json:"blood_type"`
    Allergies     []string            `json:"allergies"`
    Medications   []CurrentMedication `json:"medications"`
    Conditions    []MedicalCondition  `json:"conditions"`
    Surgeries     []Surgery           `json:"surgeries"`
    Vaccinations  []Vaccination       `json:"vaccinations"`
}

type CurrentMedication struct {
    Name        string `json:"name"`
    Dosage      string `json:"dosage"`
    Frequency   string `json:"frequency"`
    StartDate   time.Time `json:"start_date"`
    EndDate     *time.Time `json:"end_date,omitempty"`
    PrescribedBy string `json:"prescribed_by"`
}

type MedicalCondition struct {
    Name        string     `json:"name"`
    Severity    string     `json:"severity"`
    DiagnosedAt time.Time  `json:"diagnosed_at"`
    Status      string     `json:"status"`
    Notes       string     `json:"notes"`
}

type Surgery struct {
    Name        string    `json:"name"`
    Date        time.Time `json:"date"`
    Surgeon     string    `json:"surgeon"`
    Hospital    string    `json:"hospital"`
    Notes       string    `json:"notes"`
}

type Vaccination struct {
    Name         string    `json:"name"`
    Date         time.Time `json:"date"`
    Administrator string   `json:"administrator"`
    LotNumber    string    `json:"lot_number"`
    NextDue      *time.Time `json:"next_due,omitempty"`
}

type InsuranceInfo struct {
    Provider    string `json:"provider"`
    PolicyNumber string `json:"policy_number"`
    GroupNumber  string `json:"group_number"`
    ValidFrom    time.Time `json:"valid_from"`
    ValidTo      time.Time `json:"valid_to"`
}

type Contact struct {
    Name         string `json:"name"`
    Relationship string `json:"relationship"`
    Phone        string `json:"phone"`
    Email        string `json:"email"`
}

type Appointment struct {
    ID          string            `json:"id" db:"id"`
    PatientID   string            `json:"patient_id" db:"patient_id" validate:"required"`
    DoctorID    string            `json:"doctor_id" db:"doctor_id" validate:"required"`
    DateTime    time.Time         `json:"date_time" db:"date_time" validate:"required"`
    Duration    int               `json:"duration" db:"duration"` // minutes
    Type        AppointmentType   `json:"type" db:"type" validate:"required"`
    Status      AppointmentStatus `json:"status" db:"status"`
    Reason      string            `json:"reason" db:"reason"`
    Notes       string            `json:"notes" db:"notes"`
    CreatedAt   time.Time         `json:"created_at" db:"created_at"`
    UpdatedAt   time.Time         `json:"updated_at" db:"updated_at"`
}

type AppointmentType string

const (
    AppointmentTypeCheckup     AppointmentType = "checkup"
    AppointmentTypeConsultation AppointmentType = "consultation"
    AppointmentTypeFollowUp    AppointmentType = "follow_up"
    AppointmentTypeEmergency   AppointmentType = "emergency"
    AppointmentTypeSurgery     AppointmentType = "surgery"
)

type AppointmentStatus string

const (
    AppointmentStatusScheduled  AppointmentStatus = "scheduled"
    AppointmentStatusConfirmed  AppointmentStatus = "confirmed"
    AppointmentStatusInProgress AppointmentStatus = "in_progress"
    AppointmentStatusCompleted  AppointmentStatus = "completed"
    AppointmentStatusCancelled  AppointmentStatus = "cancelled"
    AppointmentStatusNoShow     AppointmentStatus = "no_show"
)

type PatientRepository interface {
    Create(ctx context.Context, patient *Patient) error
    GetByID(ctx context.Context, id string) (*Patient, error)
    GetByEmail(ctx context.Context, email string) (*Patient, error)
    Update(ctx context.Context, patient *Patient) error
    List(ctx context.Context, filter PatientFilter) ([]*Patient, int64, error)
    UpdateMedicalHistory(ctx context.Context, patientID string, history MedicalHistory) error
}

type AppointmentRepository interface {
    Create(ctx context.Context, appointment *Appointment) error
    GetByID(ctx context.Context, id string) (*Appointment, error)
    GetByPatientID(ctx context.Context, patientID string, from, to time.Time) ([]*Appointment, error)
    GetByDoctorID(ctx context.Context, doctorID string, from, to time.Time) ([]*Appointment, error)
    UpdateStatus(ctx context.Context, id string, status AppointmentStatus) error
    List(ctx context.Context, filter AppointmentFilter) ([]*Appointment, int64, error)
    CheckAvailability(ctx context.Context, doctorID string, dateTime time.Time, duration int) (bool, error)
}

type PatientFilter struct {
    SearchQuery string
    Status      string
    Limit       int
    Offset      int
}

type AppointmentFilter struct {
    PatientID string
    DoctorID  string
    Type      AppointmentType
    Status    AppointmentStatus
    DateFrom  time.Time
    DateTo    time.Time
    Limit     int
    Offset    int
}
```

### **Use Cases**
```go
// src/core/usecases/appointment_usecase.go
package usecases

import (
    "context"
    "fmt"
    "time"
)

type AppointmentUseCase struct {
    appointmentRepo domains.AppointmentRepository
    patientRepo     domains.PatientRepository
    doctorRepo      domains.DoctorRepository
    notificationSvc NotificationService
    eventBus        EventBus
    logger          logger.Logger
}

func NewAppointmentUseCase(
    appointmentRepo domains.AppointmentRepository,
    patientRepo domains.PatientRepository,
    doctorRepo domains.DoctorRepository,
    notificationSvc NotificationService,
    eventBus EventBus,
    logger logger.Logger,
) *AppointmentUseCase {
    return &AppointmentUseCase{
        appointmentRepo: appointmentRepo,
        patientRepo:     patientRepo,
        doctorRepo:      doctorRepo,
        notificationSvc: notificationSvc,
        eventBus:        eventBus,
        logger:          logger,
    }
}

func (uc *AppointmentUseCase) ScheduleAppointment(ctx context.Context, appointment *domains.Appointment) error {
    // Validate patient exists
    patient, err := uc.patientRepo.GetByID(ctx, appointment.PatientID)
    if err != nil {
        return fmt.Errorf("patient not found: %w", err)
    }
    
    // Validate doctor exists and availability
    doctor, err := uc.doctorRepo.GetByID(ctx, appointment.DoctorID)
    if err != nil {
        return fmt.Errorf("doctor not found: %w", err)
    }
    
    // Business rules validation
    if err := uc.validateAppointmentScheduling(ctx, appointment, patient, doctor); err != nil {
        return fmt.Errorf("appointment validation failed: %w", err)
    }
    
    // Check doctor availability
    available, err := uc.appointmentRepo.CheckAvailability(ctx, appointment.DoctorID, 
        appointment.DateTime, appointment.Duration)
    if err != nil {
        return fmt.Errorf("failed to check availability: %w", err)
    }
    if !available {
        return fmt.Errorf("doctor is not available at the requested time")
    }
    
    // Generate appointment ID and set defaults
    appointment.ID = generateAppointmentID()
    appointment.Status = domains.AppointmentStatusScheduled
    appointment.CreatedAt = time.Now()
    appointment.UpdatedAt = time.Now()
    
    // Create appointment
    if err := uc.appointmentRepo.Create(ctx, appointment); err != nil {
        uc.logger.Error(ctx, err, "Failed to create appointment")
        return fmt.Errorf("failed to create appointment: %w", err)
    }
    
    // Send notifications
    go uc.sendAppointmentNotifications(context.Background(), appointment, patient, doctor, "scheduled")
    
    // Publish appointment scheduled event
    event := &AppointmentScheduledEvent{
        AppointmentID: appointment.ID,
        PatientID:     appointment.PatientID,
        DoctorID:      appointment.DoctorID,
        DateTime:      appointment.DateTime,
        Type:          appointment.Type,
        ScheduledAt:   appointment.CreatedAt,
    }
    uc.eventBus.Publish(ctx, "appointment.scheduled", event)
    
    uc.logger.Info(ctx, "Appointment scheduled successfully",
        logger.String("appointment_id", appointment.ID),
        logger.String("patient_id", appointment.PatientID),
        logger.String("doctor_id", appointment.DoctorID))
    
    return nil
}

func (uc *AppointmentUseCase) RescheduleAppointment(ctx context.Context, appointmentID string, newDateTime time.Time) error {
    // Get existing appointment
    appointment, err := uc.appointmentRepo.GetByID(ctx, appointmentID)
    if err != nil {
        return fmt.Errorf("appointment not found: %w", err)
    }
    
    // Check if appointment can be rescheduled
    if appointment.Status != domains.AppointmentStatusScheduled && 
       appointment.Status != domains.AppointmentStatusConfirmed {
        return fmt.Errorf("appointment cannot be rescheduled (status: %s)", appointment.Status)
    }
    
    // Check doctor availability for new time
    available, err := uc.appointmentRepo.CheckAvailability(ctx, appointment.DoctorID, 
        newDateTime, appointment.Duration)
    if err != nil {
        return fmt.Errorf("failed to check availability: %w", err)
    }
    if !available {
        return fmt.Errorf("doctor is not available at the new requested time")
    }
    
    // Update appointment
    oldDateTime := appointment.DateTime
    appointment.DateTime = newDateTime
    appointment.UpdatedAt = time.Now()
    
    if err := uc.appointmentRepo.Update(ctx, appointment); err != nil {
        return fmt.Errorf("failed to update appointment: %w", err)
    }
    
    // Send notifications
    patient, _ := uc.patientRepo.GetByID(ctx, appointment.PatientID)
    doctor, _ := uc.doctorRepo.GetByID(ctx, appointment.DoctorID)
    go uc.sendAppointmentNotifications(context.Background(), appointment, patient, doctor, "rescheduled")
    
    // Publish appointment rescheduled event
    event := &AppointmentRescheduledEvent{
        AppointmentID: appointment.ID,
        OldDateTime:   oldDateTime,
        NewDateTime:   newDateTime,
        RescheduledAt: appointment.UpdatedAt,
    }
    uc.eventBus.Publish(ctx, "appointment.rescheduled", event)
    
    return nil
}

func (uc *AppointmentUseCase) CompleteAppointment(ctx context.Context, appointmentID string, notes string) error {
    appointment, err := uc.appointmentRepo.GetByID(ctx, appointmentID)
    if err != nil {
        return fmt.Errorf("appointment not found: %w", err)
    }
    
    if appointment.Status != domains.AppointmentStatusInProgress {
        return fmt.Errorf("appointment is not in progress")
    }
    
    // Update appointment
    appointment.Status = domains.AppointmentStatusCompleted
    appointment.Notes = notes
    appointment.UpdatedAt = time.Now()
    
    if err := uc.appointmentRepo.UpdateStatus(ctx, appointmentID, domains.AppointmentStatusCompleted); err != nil {
        return fmt.Errorf("failed to update appointment status: %w", err)
    }
    
    // Publish appointment completed event
    event := &AppointmentCompletedEvent{
        AppointmentID: appointment.ID,
        PatientID:     appointment.PatientID,
        DoctorID:      appointment.DoctorID,
        CompletedAt:   appointment.UpdatedAt,
        Notes:         notes,
    }
    uc.eventBus.Publish(ctx, "appointment.completed", event)
    
    return nil
}

func (uc *AppointmentUseCase) validateAppointmentScheduling(ctx context.Context, appointment *domains.Appointment, patient *domains.Patient, doctor *domains.Doctor) error {
    // Check if appointment is in the future
    if appointment.DateTime.Before(time.Now()) {
        return fmt.Errorf("appointment cannot be scheduled in the past")
    }
    
    // Check if appointment is within business hours
    hour := appointment.DateTime.Hour()
    if hour < 8 || hour > 18 { // 8 AM to 6 PM
        return fmt.Errorf("appointments can only be scheduled between 8 AM and 6 PM")
    }
    
    // Check if appointment is on a weekday
    weekday := appointment.DateTime.Weekday()
    if weekday == time.Saturday || weekday == time.Sunday {
        return fmt.Errorf("appointments cannot be scheduled on weekends")
    }
    
    // Check minimum advance notice (24 hours)
    if appointment.DateTime.Sub(time.Now()) < 24*time.Hour {
        return fmt.Errorf("appointments must be scheduled at least 24 hours in advance")
    }
    
    // Check patient status
    if patient.Status != "active" {
        return fmt.Errorf("patient account is not active")
    }
    
    // Check doctor availability status
    if doctor.Status != "active" {
        return fmt.Errorf("doctor is not available")
    }
    
    // Check appointment duration
    if appointment.Duration <= 0 || appointment.Duration > 480 { // Max 8 hours
        return fmt.Errorf("invalid appointment duration")
    }
    
    return nil
}

func (uc *AppointmentUseCase) sendAppointmentNotifications(ctx context.Context, appointment *domains.Appointment, patient *domains.Patient, doctor *domains.Doctor, action string) {
    // Send email to patient
    patientEmail := NotificationRequest{
        To:      patient.PersonalInfo.Email,
        Subject: fmt.Sprintf("Appointment %s", action),
        Body:    uc.generatePatientNotificationBody(appointment, doctor, action),
        Type:    "email",
    }
    uc.notificationSvc.Send(ctx, patientEmail)
    
    // Send SMS to patient if phone available
    if patient.PersonalInfo.Phone != "" {
        patientSMS := NotificationRequest{
            To:   patient.PersonalInfo.Phone,
            Body: uc.generatePatientSMSBody(appointment, doctor, action),
            Type: "sms",
        }
        uc.notificationSvc.Send(ctx, patientSMS)
    }
    
    // Send notification to doctor
    doctorEmail := NotificationRequest{
        To:      doctor.Email,
        Subject: fmt.Sprintf("Appointment %s", action),
        Body:    uc.generateDoctorNotificationBody(appointment, patient, action),
        Type:    "email",
    }
    uc.notificationSvc.Send(ctx, doctorEmail)
}

func generateAppointmentID() string {
    return "appt_" + generateRandomString(10)
}
```

---

## 🚀 **IMPLEMENTATION WORKFLOW**

### **Step 1: Choose Your Domain**
```bash
# Select domain template
DOMAIN="ecommerce"  # or "finance", "healthcare", "saas", "iot", "social"

# Copy domain files
cp -r domains/$DOMAIN/* src/core/
```

### **Step 2: Customize Configuration**
```yaml
# configs/config.yaml - Add domain-specific config
app:
  name: "my-${DOMAIN}-backend"
  domain: "${DOMAIN}"

business:
  rules:
    max_order_value: 10000  # E-commerce specific
    min_account_balance: 0  # Finance specific
    appointment_advance_hours: 24  # Healthcare specific
```

### **Step 3: Database Setup**
```bash
# Run domain-specific migrations
migrate -path migrations/$DOMAIN -database $DATABASE_URL up

# Seed initial data
go run scripts/seed_${DOMAIN}_data.go
```

### **Step 4: Update Bootstrap**
```go
// src/bootstrap/domain.go
func ProvideDomainServices(
    // ... existing providers
) {
    // Add domain-specific providers
    switch config.App.Domain {
    case "ecommerce":
        return ProvideEcommerceServices(...)
    case "finance":
        return ProvideFinanceServices(...)
    case "healthcare":
        return ProvideHealthcareServices(...)
    }
}
```

### **Step 5: Test & Deploy**
```bash
# Test domain-specific functionality
go test ./src/core/usecases/...

# Run locally
go run main.go

# Deploy to production
kubectl apply -f k8s/
```

---

## 📝 **CUSTOMIZATION CHECKLIST**

### ✅ **Domain Models**
- [ ] Define core entities
- [ ] Add validation rules
- [ ] Create repository interfaces
- [ ] Define domain events

### ✅ **Business Logic**
- [ ] Implement use cases
- [ ] Add business rules validation
- [ ] Create domain services
- [ ] Handle error scenarios

### ✅ **API Layer**
- [ ] Create controllers
- [ ] Define request/response DTOs
- [ ] Add input validation
- [ ] Setup routes

### ✅ **Data Layer**
- [ ] Create database migrations
- [ ] Implement repositories
- [ ] Add indexes for performance
- [ ] Setup data seeding

### ✅ **Integration**
- [ ] External service adapters
- [ ] Event handlers
- [ ] Background jobs
- [ ] Notification services

### ✅ **Testing**
- [ ] Unit tests for use cases
- [ ] Integration tests for repositories
- [ ] API tests for controllers
- [ ] End-to-end tests

---

## 🎯 **NEXT STEPS**

1. **Chọn domain template** phù hợp với business của bạn
2. **Copy và customize** domain models, use cases, controllers
3. **Setup database** với migrations và seed data
4. **Test thoroughly** với unit và integration tests
5. **Deploy to production** với K8s manifests

**Với templates này, bạn có thể customize business logic cho bất kỳ domain nào trong vài giờ thay vì vài tuần! 🚀** 