# 📚 API DOCUMENTATION
## Complete REST API Reference

> **OpenAPI 3.0 Specification**: Comprehensive API documentation với authentication, examples, và testing guides.

---

## 🌐 **API OVERVIEW**

### **Base Information**
- **Base URL**: `https://api.yourdomain.com/api/v1`
- **Protocol**: HTTPS only
- **Format**: JSON
- **Versioning**: URI versioning (`/api/v1`, `/api/v2`)
- **Rate Limiting**: 1000 requests/hour per API key

### **Authentication**
```http
Authorization: Bearer <access_token>
Content-Type: application/json
```

---

## 🔐 **AUTHENTICATION ENDPOINTS**

### **POST /auth/register**
Đăng ký tài khoản mới

**Request:**
```json
{
  "email": "user@example.com",
  "password": "SecurePass123!",
  "first_name": "John",
  "last_name": "Doe",
  "phone": "+84901234567"
}
```

**Response (201):**
```json
{
  "message": "User registered successfully",
  "data": {
    "user": {
      "id": "usr_1234567890",
      "email": "user@example.com",
      "first_name": "John",
      "last_name": "Doe",
      "status": "active",
      "created_at": "2024-01-01T00:00:00Z"
    },
    "tokens": {
      "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
      "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
      "expires_in": 900
    }
  }
}
```

### **POST /auth/login**
Đăng nhập

**Request:**
```json
{
  "email": "user@example.com",
  "password": "SecurePass123!"
}
```

**Response (200):**
```json
{
  "message": "Login successful",
  "data": {
    "user": {
      "id": "usr_1234567890",
      "email": "user@example.com",
      "first_name": "John",
      "last_name": "Doe",
      "role": "user"
    },
    "tokens": {
      "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
      "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
      "expires_in": 900
    }
  }
}
```

### **POST /auth/refresh**
Làm mới access token

**Request:**
```json
{
  "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

**Response (200):**
```json
{
  "data": {
    "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "expires_in": 900
  }
}
```

---

## 🛒 **E-COMMERCE ENDPOINTS**

### **Products API**

#### **GET /products**
Lấy danh sách sản phẩm

**Query Parameters:**
- `category_id` (string): Filter by category
- `min_price` (number): Minimum price filter
- `max_price` (number): Maximum price filter
- `status` (string): active, inactive
- `q` (string): Search query
- `sort_by` (string): name, price, created_at
- `sort_order` (string): asc, desc
- `limit` (number): Items per page (default: 20, max: 100)
- `offset` (number): Skip items (default: 0)

**Response (200):**
```json
{
  "data": [
    {
      "id": "prod_1234567890",
      "name": "iPhone 15 Pro",
      "description": "Latest iPhone with titanium design",
      "price": {
        "amount": 999.99,
        "currency": "USD"
      },
      "stock": 50,
      "sku": "IPHONE15PRO-256GB",
      "category_id": "cat_electronics",
      "images": [
        "https://cdn.example.com/iphone15pro-1.jpg",
        "https://cdn.example.com/iphone15pro-2.jpg"
      ],
      "status": "active",
      "created_at": "2024-01-01T00:00:00Z",
      "updated_at": "2024-01-01T00:00:00Z"
    }
  ],
  "pagination": {
    "total": 150,
    "limit": 20,
    "offset": 0,
    "has_next": true
  }
}
```

#### **POST /products**
Tạo sản phẩm mới (Admin only)

**Request:**
```json
{
  "name": "MacBook Pro M3",
  "description": "Professional laptop with M3 chip",
  "price": 1999.99,
  "currency": "USD",
  "stock": 25,
  "sku": "MACBOOK-PRO-M3-16",
  "category_id": "cat_electronics",
  "images": [
    "https://cdn.example.com/macbook-m3-1.jpg"
  ]
}
```

**Response (201):**
```json
{
  "message": "Product created successfully",
  "data": {
    "id": "prod_0987654321",
    "name": "MacBook Pro M3",
    "description": "Professional laptop with M3 chip",
    "price": {
      "amount": 1999.99,
      "currency": "USD"
    },
    "stock": 25,
    "sku": "MACBOOK-PRO-M3-16",
    "category_id": "cat_electronics",
    "images": ["https://cdn.example.com/macbook-m3-1.jpg"],
    "status": "active",
    "created_at": "2024-01-01T00:00:00Z",
    "updated_at": "2024-01-01T00:00:00Z"
  }
}
```

#### **GET /products/{id}**
Lấy chi tiết sản phẩm

**Response (200):**
```json
{
  "data": {
    "id": "prod_1234567890",
    "name": "iPhone 15 Pro",
    "description": "Latest iPhone with titanium design",
    "price": {
      "amount": 999.99,
      "currency": "USD"
    },
    "stock": 50,
    "sku": "IPHONE15PRO-256GB",
    "category_id": "cat_electronics",
    "images": [
      "https://cdn.example.com/iphone15pro-1.jpg"
    ],
    "status": "active",
    "created_at": "2024-01-01T00:00:00Z",
    "updated_at": "2024-01-01T00:00:00Z"
  }
}
```

### **Orders API**

#### **POST /orders**
Tạo đơn hàng mới

**Request:**
```json
{
  "items": [
    {
      "product_id": "prod_1234567890",
      "quantity": 2,
      "price": {
        "amount": 999.99,
        "currency": "USD"
      }
    }
  ],
  "shipping_info": {
    "street": "123 Main St",
    "city": "Ho Chi Minh City",
    "state": "Ho Chi Minh",
    "zip_code": "70000",
    "country": "Vietnam"
  },
  "notes": "Please deliver in the morning"
}
```

**Response (201):**
```json
{
  "message": "Order created successfully",
  "data": {
    "id": "order_1234567890",
    "customer_id": "usr_1234567890",
    "items": [
      {
        "product_id": "prod_1234567890",
        "quantity": 2,
        "price": {
          "amount": 999.99,
          "currency": "USD"
        },
        "total": {
          "amount": 1999.98,
          "currency": "USD"
        }
      }
    ],
    "total_amount": {
      "amount": 1999.98,
      "currency": "USD"
    },
    "status": "pending",
    "shipping_info": {
      "street": "123 Main St",
      "city": "Ho Chi Minh City",
      "state": "Ho Chi Minh",
      "zip_code": "70000",
      "country": "Vietnam"
    },
    "notes": "Please deliver in the morning",
    "created_at": "2024-01-01T00:00:00Z"
  }
}
```

#### **GET /orders**
Lấy danh sách đơn hàng của user

**Query Parameters:**
- `status` (string): pending, confirmed, paid, shipped, delivered, cancelled
- `date_from` (string): ISO date
- `date_to` (string): ISO date
- `limit` (number): Items per page
- `offset` (number): Skip items

**Response (200):**
```json
{
  "data": [
    {
      "id": "order_1234567890",
      "customer_id": "usr_1234567890",
      "total_amount": {
        "amount": 1999.98,
        "currency": "USD"
      },
      "status": "pending",
      "created_at": "2024-01-01T00:00:00Z",
      "updated_at": "2024-01-01T00:00:00Z"
    }
  ],
  "pagination": {
    "total": 5,
    "limit": 20,
    "offset": 0
  }
}
```

#### **POST /orders/{id}/payment**
Xử lý thanh toán đơn hàng

**Request:**
```json
{
  "method": "card",
  "card_token": "tok_1234567890"
}
```

**Response (200):**
```json
{
  "message": "Payment processed successfully",
  "data": {
    "transaction_id": "txn_1234567890",
    "status": "completed",
    "processed_at": "2024-01-01T00:00:00Z"
  }
}
```

---

## 💰 **FINANCIAL SERVICES ENDPOINTS**

### **Accounts API**

#### **GET /accounts**
Lấy danh sách tài khoản

**Response (200):**
```json
{
  "data": [
    {
      "id": "acc_1234567890",
      "customer_id": "usr_1234567890",
      "account_number": "CHK1234567890",
      "type": "checking",
      "balance": {
        "amount": 5000.00,
        "currency": "USD"
      },
      "status": "active",
      "opened_at": "2024-01-01T00:00:00Z"
    }
  ]
}
```

#### **POST /accounts**
Tạo tài khoản mới

**Request:**
```json
{
  "type": "savings",
  "currency": "USD",
  "initial_deposit": 1000.00
}
```

**Response (201):**
```json
{
  "message": "Account created successfully",
  "data": {
    "id": "acc_0987654321",
    "customer_id": "usr_1234567890",
    "account_number": "SAV0987654321",
    "type": "savings",
    "balance": {
      "amount": 1000.00,
      "currency": "USD"
    },
    "status": "active",
    "opened_at": "2024-01-01T00:00:00Z"
  }
}
```

### **Transactions API**

#### **POST /transactions/transfer**
Chuyển tiền giữa các tài khoản

**Request:**
```json
{
  "from_account_id": "acc_1234567890",
  "to_account_id": "acc_0987654321",
  "amount": {
    "amount": 500.00,
    "currency": "USD"
  },
  "description": "Monthly savings transfer"
}
```

**Response (200):**
```json
{
  "message": "Transfer completed successfully",
  "data": {
    "transaction_id": "txn_1234567890",
    "from_account_id": "acc_1234567890",
    "to_account_id": "acc_0987654321",
    "amount": {
      "amount": 500.00,
      "currency": "USD"
    },
    "status": "processed",
    "description": "Monthly savings transfer",
    "processed_at": "2024-01-01T00:00:00Z"
  }
}
```

#### **GET /accounts/{id}/transactions**
Lấy lịch sử giao dịch

**Query Parameters:**
- `type` (string): deposit, withdrawal, transfer, payment
- `date_from` (string): ISO date
- `date_to` (string): ISO date
- `limit` (number): Items per page
- `offset` (number): Skip items

**Response (200):**
```json
{
  "data": [
    {
      "id": "txn_1234567890",
      "from_account_id": "acc_1234567890",
      "to_account_id": "acc_0987654321",
      "amount": {
        "amount": 500.00,
        "currency": "USD"
      },
      "type": "transfer",
      "status": "processed",
      "description": "Monthly savings transfer",
      "processed_at": "2024-01-01T00:00:00Z",
      "created_at": "2024-01-01T00:00:00Z"
    }
  ],
  "pagination": {
    "total": 25,
    "limit": 20,
    "offset": 0
  }
}
```

---

## 🏥 **HEALTHCARE ENDPOINTS**

### **Patients API**

#### **POST /patients**
Đăng ký bệnh nhân mới

**Request:**
```json
{
  "personal_info": {
    "first_name": "John",
    "last_name": "Doe",
    "date_of_birth": "1990-01-01",
    "gender": "male",
    "phone": "+84901234567",
    "email": "john.doe@example.com",
    "address": {
      "street": "123 Main St",
      "city": "Ho Chi Minh City",
      "state": "Ho Chi Minh",
      "zip_code": "70000",
      "country": "Vietnam"
    }
  },
  "medical_info": {
    "blood_type": "O+",
    "allergies": ["Penicillin", "Peanuts"],
    "conditions": [
      {
        "name": "Hypertension",
        "severity": "mild",
        "diagnosed_at": "2023-01-01T00:00:00Z",
        "status": "active"
      }
    ]
  },
  "insurance": {
    "provider": "Vietnam Social Security",
    "policy_number": "VSS123456789",
    "valid_from": "2024-01-01",
    "valid_to": "2024-12-31"
  },
  "emergency_contact": {
    "name": "Jane Doe",
    "relationship": "spouse",
    "phone": "+84907654321"
  }
}
```

**Response (201):**
```json
{
  "message": "Patient registered successfully",
  "data": {
    "id": "pat_1234567890",
    "personal_info": {
      "first_name": "John",
      "last_name": "Doe",
      "date_of_birth": "1990-01-01",
      "gender": "male",
      "phone": "+84901234567",
      "email": "john.doe@example.com"
    },
    "status": "active",
    "created_at": "2024-01-01T00:00:00Z"
  }
}
```

### **Appointments API**

#### **POST /appointments**
Đặt lịch khám

**Request:**
```json
{
  "patient_id": "pat_1234567890",
  "doctor_id": "doc_1234567890",
  "date_time": "2024-01-15T09:00:00Z",
  "duration": 30,
  "type": "consultation",
  "reason": "Regular checkup"
}
```

**Response (201):**
```json
{
  "message": "Appointment scheduled successfully",
  "data": {
    "id": "appt_1234567890",
    "patient_id": "pat_1234567890",
    "doctor_id": "doc_1234567890",
    "date_time": "2024-01-15T09:00:00Z",
    "duration": 30,
    "type": "consultation",
    "status": "scheduled",
    "reason": "Regular checkup",
    "created_at": "2024-01-01T00:00:00Z"
  }
}
```

#### **GET /appointments**
Lấy danh sách lịch khám

**Query Parameters:**
- `patient_id` (string): Filter by patient
- `doctor_id` (string): Filter by doctor
- `status` (string): scheduled, confirmed, completed, cancelled
- `date_from` (string): ISO date
- `date_to` (string): ISO date

**Response (200):**
```json
{
  "data": [
    {
      "id": "appt_1234567890",
      "patient_id": "pat_1234567890",
      "doctor_id": "doc_1234567890",
      "date_time": "2024-01-15T09:00:00Z",
      "duration": 30,
      "type": "consultation",
      "status": "scheduled",
      "reason": "Regular checkup",
      "created_at": "2024-01-01T00:00:00Z"
    }
  ]
}
```

#### **PUT /appointments/{id}/status**
Cập nhật trạng thái lịch khám

**Request:**
```json
{
  "status": "completed",
  "notes": "Patient is in good health. Recommended annual checkup."
}
```

**Response (200):**
```json
{
  "message": "Appointment status updated successfully",
  "data": {
    "id": "appt_1234567890",
    "status": "completed",
    "notes": "Patient is in good health. Recommended annual checkup.",
    "updated_at": "2024-01-15T09:30:00Z"
  }
}
```

---

## 🔧 **SYSTEM ENDPOINTS**

### **Health Check**

#### **GET /health**
Kiểm tra tình trạng hệ thống

**Response (200):**
```json
{
  "status": "healthy",
  "timestamp": "2024-01-01T00:00:00Z",
  "version": "1.0.0",
  "services": {
    "database": "healthy",
    "redis": "healthy",
    "external_apis": "healthy"
  }
}
```

### **Metrics**

#### **GET /metrics**
Prometheus metrics endpoint

**Response (200):**
```
# HELP http_requests_total Total HTTP requests
# TYPE http_requests_total counter
http_requests_total{method="GET",endpoint="/api/v1/products",status_code="200"} 1234

# HELP http_request_duration_seconds HTTP request duration
# TYPE http_request_duration_seconds histogram
http_request_duration_seconds_bucket{method="GET",endpoint="/api/v1/products",le="0.1"} 1000
```

---

## 🚨 **ERROR RESPONSES**

### **Standard Error Format**
```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request validation failed",
    "details": [
      {
        "field": "email",
        "message": "Email is required"
      },
      {
        "field": "password",
        "message": "Password must be at least 8 characters"
      }
    ],
    "request_id": "req_1234567890",
    "timestamp": "2024-01-01T00:00:00Z"
  }
}
```

### **HTTP Status Codes**

| Code | Description | Usage |
|------|-------------|-------|
| 200 | OK | Successful GET, PUT, PATCH |
| 201 | Created | Successful POST |
| 204 | No Content | Successful DELETE |
| 400 | Bad Request | Invalid request data |
| 401 | Unauthorized | Missing or invalid authentication |
| 403 | Forbidden | Insufficient permissions |
| 404 | Not Found | Resource not found |
| 409 | Conflict | Resource already exists |
| 422 | Unprocessable Entity | Validation errors |
| 429 | Too Many Requests | Rate limit exceeded |
| 500 | Internal Server Error | Server error |
| 503 | Service Unavailable | Service temporarily down |

### **Common Error Codes**

| Code | Description |
|------|-------------|
| `VALIDATION_ERROR` | Request validation failed |
| `AUTHENTICATION_REQUIRED` | Authentication token required |
| `INVALID_TOKEN` | Invalid or expired token |
| `INSUFFICIENT_PERMISSIONS` | User lacks required permissions |
| `RESOURCE_NOT_FOUND` | Requested resource not found |
| `RESOURCE_ALREADY_EXISTS` | Resource already exists |
| `RATE_LIMIT_EXCEEDED` | Too many requests |
| `EXTERNAL_SERVICE_ERROR` | External service unavailable |
| `DATABASE_ERROR` | Database operation failed |

---

## 🧪 **TESTING THE API**

### **Using cURL**

```bash
# Get access token
ACCESS_TOKEN=$(curl -X POST http://localhost:8080/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"user@example.com","password":"password"}' | \
  jq -r '.data.tokens.access_token')

# Use token in subsequent requests
curl -X GET http://localhost:8080/api/v1/products \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json"
```

### **Using Postman**

1. Import the OpenAPI spec: `openapi.yaml`
2. Set up environment variables:
   - `base_url`: `http://localhost:8080/api/v1`
   - `access_token`: Your JWT token
3. Use `{{base_url}}` and `{{access_token}}` in requests

### **Using HTTPie**

```bash
# Login and save token
http POST localhost:8080/api/v1/auth/login \
  email=user@example.com password=password

# Use token (replace with actual token)
http GET localhost:8080/api/v1/products \
  Authorization:"Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
```

---

## 📊 **RATE LIMITING**

### **Limits by Endpoint Type**

| Endpoint Type | Rate Limit | Window |
|---------------|------------|--------|
| Authentication | 10 requests | 1 minute |
| Read Operations | 1000 requests | 1 hour |
| Write Operations | 100 requests | 1 hour |
| File Uploads | 10 requests | 1 hour |

### **Rate Limit Headers**

```http
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1640995200
```

---

## 🔒 **SECURITY**

### **Authentication Methods**
- **JWT Bearer Tokens**: For API access
- **API Keys**: For server-to-server communication
- **OAuth 2.0**: For third-party integrations

### **Security Headers**
```http
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
X-XSS-Protection: 1; mode=block
Strict-Transport-Security: max-age=31536000; includeSubDomains
```

### **Data Validation**
- Input sanitization
- SQL injection prevention
- XSS protection
- CSRF protection

---

## 🌐 **WEBHOOKS**

### **Webhook Events**

| Event | Description |
|-------|-------------|
| `order.created` | New order created |
| `order.paid` | Order payment completed |
| `user.registered` | New user registration |
| `appointment.scheduled` | New appointment scheduled |

### **Webhook Payload Example**

```json
{
  "event": "order.created",
  "data": {
    "id": "order_1234567890",
    "customer_id": "usr_1234567890",
    "total_amount": {
      "amount": 999.99,
      "currency": "USD"
    },
    "status": "pending",
    "created_at": "2024-01-01T00:00:00Z"
  },
  "timestamp": "2024-01-01T00:00:00Z",
  "signature": "sha256=1234567890abcdef..."
}
```

---

## 📝 **CHANGELOG**

### **v1.2.0 (2024-01-15)**
- Added healthcare endpoints
- Improved error handling
- Added rate limiting

### **v1.1.0 (2024-01-01)**
- Added financial services endpoints
- Enhanced authentication
- Added webhook support

### **v1.0.0 (2023-12-01)**
- Initial API release
- E-commerce endpoints
- Basic authentication

---

## 📞 **SUPPORT**

### **API Support**
- **Documentation**: This document
- **OpenAPI Spec**: `openapi.yaml`
- **Postman Collection**: `api-collection.json`
- **Issues**: GitHub Issues
- **Email**: api-support@yourdomain.com

### **SLA**
- **Uptime**: 99.9%
- **Response Time**: < 200ms (95th percentile)
- **Support Response**: < 24 hours

---

**🚀 Happy coding! Build amazing applications with our API! 🚀** 