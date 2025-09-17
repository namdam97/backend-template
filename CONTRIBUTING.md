# 🤝 CONTRIBUTING GUIDE
## Welcome to Backend Template Community!

> **Chào mừng bạn đến với cộng đồng Backend Template!** Chúng tôi rất hoan nghênh mọi đóng góp từ bug fixes, features mới, documentation improvements đến community support.

---

## 🎯 **HOW TO CONTRIBUTE**

### **Types of Contributions**
- 🐛 **Bug Reports**: Report bugs với detailed information
- ✨ **Feature Requests**: Suggest new features hoặc improvements
- 🔧 **Bug Fixes**: Fix existing bugs
- 🚀 **New Features**: Implement new functionality
- 📚 **Documentation**: Improve docs, examples, tutorials
- 🧪 **Tests**: Add test coverage
- 🎨 **Code Quality**: Refactoring, performance improvements

---

## 🚀 **GETTING STARTED**

### **Development Setup**

```bash
# 1. Fork the repository on GitHub
# 2. Clone your fork
git clone https://github.com/YOUR-USERNAME/backend-template.git
cd backend-template

# 3. Add upstream remote
git remote add upstream https://github.com/original-org/backend-template.git

# 4. Create development branch
git checkout -b feature/your-feature-name

# 5. Install dependencies
go mod tidy

# 6. Setup development environment
make dev-setup

# 7. Run tests to ensure everything works
make test
```

### **Development Workflow**

```bash
# 1. Sync with upstream
git checkout main
git pull upstream main
git push origin main

# 2. Create feature branch
git checkout -b feature/amazing-feature

# 3. Make your changes
# ... code, code, code ...

# 4. Run tests
make test
make lint

# 5. Commit changes
git add .
git commit -m "feat: add amazing feature"

# 6. Push to your fork
git push origin feature/amazing-feature

# 7. Create Pull Request on GitHub
```

---

## 📋 **CODE STANDARDS**

### **Go Code Style**

#### **Formatting**
```bash
# Use gofmt for formatting
gofmt -w .

# Use goimports for imports
goimports -w .

# Use golangci-lint for linting
golangci-lint run
```

#### **Naming Conventions**
```go
// ✅ Good
type UserService struct {
    userRepo UserRepository
    logger   Logger
}

func (s *UserService) CreateUser(ctx context.Context, user *User) error {
    // Implementation
}

// ❌ Bad
type userservice struct {
    UserRepo UserRepository
    Logger   Logger
}

func (s *userservice) createUser(ctx context.Context, user *User) error {
    // Implementation
}
```

#### **Error Handling**
```go
// ✅ Good - Wrap errors with context
func (s *UserService) GetUser(ctx context.Context, id string) (*User, error) {
    user, err := s.userRepo.GetByID(ctx, id)
    if err != nil {
        return nil, fmt.Errorf("failed to get user %s: %w", id, err)
    }
    return user, nil
}

// ❌ Bad - Return raw errors
func (s *UserService) GetUser(ctx context.Context, id string) (*User, error) {
    return s.userRepo.GetByID(ctx, id)
}
```

#### **Context Usage**
```go
// ✅ Good - Always pass context
func (s *UserService) CreateUser(ctx context.Context, user *User) error {
    if err := s.validateUser(ctx, user); err != nil {
        return err
    }
    
    return s.userRepo.Create(ctx, user)
}

// ❌ Bad - Missing context
func (s *UserService) CreateUser(user *User) error {
    return s.userRepo.Create(user)
}
```

#### **Testing Standards**
```go
// ✅ Good - Table-driven tests
func TestUserService_CreateUser(t *testing.T) {
    tests := []struct {
        name    string
        user    *User
        wantErr bool
    }{
        {
            name: "valid user",
            user: &User{
                Name:  "John Doe",
                Email: "john@example.com",
            },
            wantErr: false,
        },
        {
            name: "invalid email",
            user: &User{
                Name:  "John Doe",
                Email: "invalid-email",
            },
            wantErr: true,
        },
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            err := userService.CreateUser(context.Background(), tt.user)
            if (err != nil) != tt.wantErr {
                t.Errorf("CreateUser() error = %v, wantErr %v", err, tt.wantErr)
            }
        })
    }
}
```

### **Documentation Standards**

#### **Code Comments**
```go
// ✅ Good - Explain why, not what
// retryWithBackoff implements exponential backoff for failed operations
// to avoid overwhelming external services during outages.
func retryWithBackoff(ctx context.Context, operation func() error) error {
    // Implementation
}

// ❌ Bad - Obvious comments
// CreateUser creates a user
func CreateUser(user *User) error {
    // Implementation
}
```

#### **Package Documentation**
```go
// Package usecases contains business logic implementations.
//
// This package implements the application layer of clean architecture,
// orchestrating interactions between domain entities and infrastructure.
package usecases
```

#### **API Documentation**
```go
// CreateProductRequest represents the request to create a new product.
type CreateProductRequest struct {
    Name        string  `json:"name" validate:"required,min=3,max=100"`
    Description string  `json:"description" validate:"max=1000"`
    Price       float64 `json:"price" validate:"required,min=0"`
    SKU         string  `json:"sku" validate:"required"`
}
```

---

## 🔍 **PULL REQUEST PROCESS**

### **Before Creating PR**

```bash
# 1. Ensure tests pass
make test

# 2. Run linting
make lint

# 3. Check code coverage
make test-coverage

# 4. Update documentation if needed
# 5. Add/update tests for new features
# 6. Ensure commit messages follow conventions
```

### **PR Template**

```markdown
## Description
Brief description of what this PR does.

## Type of Change
- [ ] Bug fix (non-breaking change which fixes an issue)
- [ ] New feature (non-breaking change which adds functionality)
- [ ] Breaking change (fix or feature that would cause existing functionality to not work as expected)
- [ ] Documentation update

## Testing
- [ ] Unit tests pass
- [ ] Integration tests pass
- [ ] Manual testing completed
- [ ] Added new tests for new functionality

## Checklist
- [ ] My code follows the code style of this project
- [ ] I have performed a self-review of my own code
- [ ] I have commented my code, particularly in hard-to-understand areas
- [ ] I have made corresponding changes to the documentation
- [ ] My changes generate no new warnings
- [ ] I have added tests that prove my fix is effective or that my feature works
- [ ] New and existing unit tests pass locally with my changes

## Screenshots (if applicable)
Add screenshots to help explain your changes.

## Related Issues
Fixes #(issue number)
```

### **PR Review Process**

1. **Automated Checks**
   - ✅ All tests pass
   - ✅ Code coverage maintained
   - ✅ Linting passes
   - ✅ Security scan passes

2. **Human Review**
   - 👥 At least 1 approving review required
   - 🔍 Code quality review
   - 📚 Documentation review
   - 🧪 Test coverage review

3. **Merge Requirements**
   - ✅ All checks pass
   - ✅ Approved by maintainer
   - ✅ Up-to-date with main branch
   - ✅ Squash and merge preferred

---

## 🐛 **BUG REPORTS**

### **Bug Report Template**

```markdown
---
name: Bug Report
about: Create a report to help us improve
title: '[BUG] '
labels: bug
assignees: ''
---

## Bug Description
A clear and concise description of what the bug is.

## To Reproduce
Steps to reproduce the behavior:
1. Go to '...'
2. Click on '....'
3. Scroll down to '....'
4. See error

## Expected Behavior
A clear and concise description of what you expected to happen.

## Actual Behavior
A clear and concise description of what actually happened.

## Screenshots
If applicable, add screenshots to help explain your problem.

## Environment
- OS: [e.g. Ubuntu 20.04]
- Go Version: [e.g. 1.21.0]
- Application Version: [e.g. v1.2.0]
- Database: [e.g. PostgreSQL 15.0]

## Logs
```
Add relevant log output here
```

## Additional Context
Add any other context about the problem here.
```

### **Bug Investigation Process**

1. **Reproduce**: Try to reproduce the bug locally
2. **Isolate**: Create minimal reproduction case
3. **Analyze**: Identify root cause
4. **Fix**: Implement fix with tests
5. **Verify**: Ensure fix works and doesn't break anything else

---

## ✨ **FEATURE REQUESTS**

### **Feature Request Template**

```markdown
---
name: Feature Request
about: Suggest an idea for this project
title: '[FEATURE] '
labels: enhancement
assignees: ''
---

## Is your feature request related to a problem?
A clear and concise description of what the problem is. Ex. I'm always frustrated when [...]

## Describe the solution you'd like
A clear and concise description of what you want to happen.

## Describe alternatives you've considered
A clear and concise description of any alternative solutions or features you've considered.

## Use Cases
Describe specific use cases for this feature:
1. As a [user type], I want [functionality] so that [benefit]
2. As a [user type], I want [functionality] so that [benefit]

## Acceptance Criteria
- [ ] Criterion 1
- [ ] Criterion 2
- [ ] Criterion 3

## Additional Context
Add any other context or screenshots about the feature request here.

## Implementation Ideas
If you have ideas on how this could be implemented, please share them here.
```

### **Feature Development Process**

1. **Discussion**: Discuss feature in GitHub Issues
2. **Design**: Create design document if needed
3. **Approval**: Get maintainer approval
4. **Implementation**: Implement with tests
5. **Documentation**: Update relevant documentation
6. **Review**: Code review and testing

---

## 🏗️ **ARCHITECTURE DECISIONS**

### **Adding New Domains**

```go
// 1. Create domain models
// src/core/domains/your_domain.go
type YourEntity struct {
    ID        string    `json:"id"`
    Name      string    `json:"name"`
    CreatedAt time.Time `json:"created_at"`
}

type YourRepository interface {
    Create(ctx context.Context, entity *YourEntity) error
    GetByID(ctx context.Context, id string) (*YourEntity, error)
}

// 2. Implement use cases
// src/core/usecases/your_usecase.go
type YourUseCase struct {
    repo   YourRepository
    logger logger.Logger
}

// 3. Create controllers
// src/present/http/controllers/your_controller.go
type YourController struct {
    useCase *usecases.YourUseCase
}

// 4. Add routes
// src/present/http/routes/your_routes.go
func SetupYourRoutes(router *gin.RouterGroup, controller *controllers.YourController) {
    router.POST("/your-entities", controller.Create)
    router.GET("/your-entities/:id", controller.GetByID)
}
```

### **Database Migrations**

```sql
-- migrations/YYYYMMDD_HHMMSS_create_your_table.up.sql
CREATE TABLE your_entities (
    id VARCHAR(255) PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_your_entities_name ON your_entities(name);
```

```sql
-- migrations/YYYYMMDD_HHMMSS_create_your_table.down.sql
DROP INDEX IF EXISTS idx_your_entities_name;
DROP TABLE IF EXISTS your_entities;
```

---

## 🧪 **TESTING CONTRIBUTIONS**

### **Test Requirements**

- **Unit Tests**: All new functions must have unit tests
- **Integration Tests**: New features need integration tests
- **API Tests**: New endpoints need API tests
- **Coverage**: Maintain >80% code coverage

### **Test Structure**

```go
// tests/unit/your_usecase_test.go
package unit_test

import (
    "testing"
    "github.com/stretchr/testify/suite"
)

type YourUseCaseTestSuite struct {
    suite.Suite
    useCase *usecases.YourUseCase
    // mocks and dependencies
}

func (suite *YourUseCaseTestSuite) SetupTest() {
    // Setup before each test
}

func (suite *YourUseCaseTestSuite) TestYourFunction_Success() {
    // Test implementation
}

func TestYourUseCaseTestSuite(t *testing.T) {
    suite.Run(t, new(YourUseCaseTestSuite))
}
```

---

## 📚 **DOCUMENTATION CONTRIBUTIONS**

### **Documentation Types**

- **API Documentation**: OpenAPI specs, examples
- **User Guides**: Step-by-step tutorials
- **Developer Docs**: Architecture, patterns, best practices
- **Code Examples**: Working examples for different domains

### **Documentation Standards**

```markdown
# Title
Brief description of what this document covers.

## Prerequisites
What users need before following this guide.

## Step-by-step Instructions
1. Clear, numbered steps
2. Code examples with explanations
3. Expected outputs

## Troubleshooting
Common issues and solutions.

## Next Steps
What to do after completing this guide.
```

---

## 🎖️ **RECOGNITION**

### **Contributors**
All contributors are recognized in:
- README.md contributors section
- GitHub contributors graph
- Release notes acknowledgments
- Special recognition for significant contributions

### **Maintainer Path**
Active contributors may be invited to become maintainers:
1. **Consistent Quality Contributions**: Regular, high-quality PRs
2. **Community Involvement**: Help with issues, reviews, discussions
3. **Domain Expertise**: Deep knowledge in specific areas
4. **Leadership**: Guide other contributors, make architectural decisions

---

## 📞 **COMMUNICATION**

### **Channels**

- **GitHub Issues**: Bug reports, feature requests
- **GitHub Discussions**: General questions, ideas
- **Discord**: Real-time chat (link in README)
- **Email**: security@yourdomain.com for security issues

### **Code of Conduct**

We follow the [Contributor Covenant Code of Conduct](CODE_OF_CONDUCT.md):

- **Be Respectful**: Treat everyone with respect
- **Be Inclusive**: Welcome people of all backgrounds
- **Be Constructive**: Provide helpful feedback
- **Be Patient**: Help newcomers learn

---

## 🎯 **DEVELOPMENT PRIORITIES**

### **Current Focus Areas**

1. **Performance Optimization**
   - Database query optimization
   - Caching improvements
   - Load testing

2. **Security Enhancements**
   - Security audit findings
   - Vulnerability patches
   - Security testing

3. **Developer Experience**
   - Better error messages
   - Improved documentation
   - Development tooling

4. **New Domain Support**
   - IoT domain templates
   - Social media templates
   - Real-time features

### **Help Wanted**

Looking for contributors in these areas:
- 📊 **Monitoring & Observability**: Grafana dashboards, alerts
- 🔒 **Security**: Security testing, audit fixes
- 📱 **Mobile Support**: Mobile-friendly APIs
- 🌐 **Internationalization**: Multi-language support
- 🤖 **AI/ML**: ML pipeline improvements

---

## 📋 **RELEASE PROCESS**

### **Version Naming**
- **Major**: Breaking changes (v2.0.0)
- **Minor**: New features (v1.1.0)
- **Patch**: Bug fixes (v1.0.1)

### **Release Checklist**
- [ ] All tests pass
- [ ] Documentation updated
- [ ] Migration guides (if breaking)
- [ ] Release notes written
- [ ] Security review completed

---

## 🙏 **THANK YOU**

Thank you for contributing to Backend Template! Your contributions help make this project better for everyone. Whether you're fixing a typo, adding a feature, or helping other users, every contribution matters.

### **Quick Links**
- [Code of Conduct](CODE_OF_CONDUCT.md)
- [Security Policy](SECURITY.md)
- [Issue Templates](.github/ISSUE_TEMPLATE/)
- [PR Template](.github/PULL_REQUEST_TEMPLATE.md)

---

**🚀 Happy Contributing! Let's build amazing backends together! 🚀** 