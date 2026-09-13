# Java API 设计最佳实践与设计标准

> 本文档聚焦于 Java 环境下 REST API 的设计与最佳实践，适用于 AI Vibe Coding 场景

## 目录

1. [REST API 设计原则](#1-rest-api-设计原则)
2. [URL 设计规范](#2-url-设计规范)
3. [HTTP 方法使用](#3-http-方法使用)
4. [请求与响应格式](#4-请求与响应格式)
5. [错误处理规范](#5-错误处理规范)
6. [API 版本控制](#6-api-版本控制)
7. [安全性设计](#7-安全性设计)
8. [性能优化](#8-性能优化)
9. [API 文档规范](#9-api-文档规范)
10. [Java 特定实现标准](#10-java-特定实现标准)

---

## 1. REST API 设计原则

### 1.1 核心原则

| 原则 | 描述 | 示例 |
|------|------|------|
| 资源导向 | URL 代表资源，而非动作 | `/users` 而非 `/getUsers` |
| 无状态 | 每个请求包含所有必要信息 | 不使用 session 存储状态 |
| 统一接口 | 使用标准 HTTP 方法 | GET 读取、POST 创建 |
| 分层系统 | 客户端无需了解后端架构 | 通过 API Gateway 隐藏内部服务 |

### 1.2 资源命名规范

```java
// ✅ 资源使用名词复数形式
/users
/orders
/products

// ✅ 使用子资源表示关系
/users/123/orders
/products/456/reviews

// ❌ 避免使用动词
/getUsers
/createOrder
/deleteProduct
```

---

## 2. URL 设计规范

### 2.1 URL 结构

```
https://api.example.com/v1/{resource}/{resource_id}/{sub-resource}
```

| 组成部分 | 说明 | 示例 |
|----------|------|------|
| 协议 | HTTPS 必需 | `https://` |
| 域名 | 专用 API 域名 | `api.example.com` |
| 版本 | API 版本号 | `/v1` |
| 资源 | 资源路径 | `/users` |
| 标识符 | 资源 ID | `/123` |
| 子资源 | 关联资源 | `/orders` |

### 2.2 命名规范

```java
// ✅ 使用小写字母
/api/users
/api/user-profiles

// ✅ 使用连字符分隔多词
/user-accounts
/order-items

// ❌ 避免使用下划线
/api/user_accounts

// ❌ 避免使用驼峰命名
/api/userProfiles
```

### 2.3 过滤、排序、分页

```java
// 过滤：使用 query parameters
GET /users?status=active&role=admin

// 排序：使用 sort 参数
GET /users?sort=createdAt,desc

// 分页：使用 page 和 size
GET /users?page=0&size=20
```

---

## 3. HTTP 方法使用

### 3.1 方法对照表

| 方法 | 用途 | 幂等 | 示例 |
|------|------|------|------|
| GET | 读取资源 | 是 | `GET /users/123` |
| POST | 创建资源 | 否 | `POST /users` |
| PUT | 完整更新资源 | 是 | `PUT /users/123` |
| PATCH | 部分更新资源 | 否 | `PATCH /users/123` |
| DELETE | 删除资源 | 是 | `DELETE /users/123` |

### 3.2 方法选择指南

```java
// ✅ 使用 POST 创建新资源
POST /users
Body: { "name": "John", "email": "john@example.com" }

// ✅ 使用 PUT 完整更新（替换）
PUT /users/123
Body: { "name": "John", "email": "john@example.com", "age": 30 }

// ✅ 使用 PATCH 部分更新
PATCH /users/123
Body: { "name": "John Updated" }

// ✅ 使用 GET 查询
GET /users/123
GET /users?page=1&size=10

// ✅ 使用 DELETE 删除
DELETE /users/123
```

---

## 4. 请求与响应格式

### 4.1 请求体格式

```java
// ✅ 使用 JSON 格式
Content-Type: application/json

// 请求体示例
{
    "name": "John Doe",
    "email": "john@example.com",
    "age": 30,
    "roles": ["USER", "ADMIN"]
}
```

### 4.2 响应格式标准

#### 成功响应

```java
// ✅ 统一响应包装格式
{
    "success": true,
    "data": {
        "id": 123,
        "name": "John Doe",
        "email": "john@example.com"
    },
    "message": "User created successfully",
    "timestamp": "2024-01-15T10:30:00Z"
}

// ✅ 列表响应（带分页）
{
    "success": true,
    "data": [
        { "id": 1, "name": "User 1" },
        { "id": 2, "name": "User 2" }
    ],
    "pagination": {
        "page": 0,
        "size": 20,
        "totalElements": 100,
        "totalPages": 5
    },
    "timestamp": "2024-01-15T10:30:00Z"
}
```

#### 响应状态码

| 状态码 | 含义 | 使用场景 |
|--------|------|----------|
| 200 | OK | 成功读取/更新资源 |
| 201 | Created | 成功创建资源 |
| 204 | No Content | 成功删除资源 |
| 400 | Bad Request | 请求格式错误 |
| 401 | Unauthorized | 未认证 |
| 403 | Forbidden | 无权限 |
| 404 | Not Found | 资源不存在 |
| 409 | Conflict | 资源冲突 |
| 500 | Internal Server Error | 服务器错误 |

---

## 5. 错误处理规范

### 5.1 错误响应格式

```java
// ✅ 统一错误响应格式
{
    "success": false,
    "error": {
        "code": "USER_NOT_FOUND",
        "message": "User with ID 123 not found",
        "details": [
            {
                "field": "userId",
                "message": "User ID must be a positive number"
            }
        ]
    },
    "timestamp": "2024-01-15T10:30:00Z"
}
```

### 5.2 Java 异常处理实践

```java
// 统一异常响应 DTO
public record ErrorResponse(
    boolean success,
    ErrorDetail error,
    String timestamp
) {
    public static ErrorResponse of(String code, String message) {
        return new ErrorResponse(
            false,
            new ErrorDetail(code, message, List.of()),
            Instant.now().toString()
        );
    }

    public static ErrorResponse of(String code, String message, List<FieldError> details) {
        return new ErrorResponse(
            false,
            new ErrorDetail(code, message, details),
            Instant.now().toString()
        );
    }

    public record ErrorDetail(
        String code,
        String message,
        List<FieldError> details
    ) {}

    public record FieldError(
        String field,
        String message
    ) {}
}
```

### 5.3 全局异常处理器

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException ex) {
        ErrorResponse error = ErrorResponse.of(
            "RESOURCE_NOT_FOUND",
            ex.getMessage()
        );
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }

    @ExceptionHandler(ValidationException.class)
    public ResponseEntity<ErrorResponse> handleValidation(ValidationException ex) {
        List<FieldError> errors = ex.getBindingResult().getFieldErrors().stream()
            .map(e -> new FieldError(e.getField(), e.getDefaultMessage()))
            .toList();

        ErrorResponse error = ErrorResponse.of(
            "VALIDATION_ERROR",
            "Request validation failed",
            errors
        );
        return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(error);
    }

    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ErrorResponse> handleBusiness(BusinessException ex) {
        ErrorResponse error = ErrorResponse.of(
            ex.getErrorCode(),
            ex.getMessage()
        );
        return ResponseEntity.status(HttpStatus.CONFLICT).body(error);
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneral(Exception ex) {
        ErrorResponse error = ErrorResponse.of(
            "INTERNAL_ERROR",
            "An unexpected error occurred"
        );
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(error);
    }
}
```

---

## 6. API 版本控制

### 6.1 版本控制策略

```java
// ✅ URL 版本控制（推荐）
GET /v1/users
GET /v2/users

// ✅ Header 版本控制
GET /users
Accept-Version: v1

// ❌ 不推荐：查询参数版本
GET /users?version=1
```

### 6.2 版本演进规则

```java
// 版本号规则：MAJOR.MINOR
// - MAJOR: 不兼容的 API 变更
// - MINOR: 向后兼容的新功能

// 向后兼容的变更（可在一个大版本内）
// - 新增可选字段
// - 新增端点
// - 新增响应字段

// 不兼容的变更（需升级大版本）
// - 删除或重命名字段
// - 改变字段类型
// - 改变验证规则
// - 删除端点
```

---

## 7. 安全性设计

### 7.1 认证与授权

```java
// ✅ 使用标准认证机制
// JWT Bearer Token
Authorization: Bearer <token>

// OAuth 2.0
Authorization: Bearer <access_token>
```

### 7.2 安全头部

```java
// 响应添加安全头部
HttpHeaders headers = new HttpHeaders();
headers.add("X-Content-Type-Options", "nosniff");
headers.add("X-Frame-Options", "DENY");
headers.add("X-XSS-Protection", "1; mode=block");
headers.add("Strict-Transport-Security", "max-age=31536000; includeSubDomains");
```

### 7.3 速率限制

```java
// ✅ 实现速率限制
// 响应头
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1609459200

// 超限响应
{
    "success": false,
    "error": {
        "code": "RATE_LIMIT_EXCEEDED",
        "message": "Too many requests. Please try again later."
    }
}
```

### 7.4 输入验证

```java
// ✅ 使用注解进行输入验证
public record CreateUserRequest(
    @NotBlank(message = "Name is required")
    @Size(min = 2, max = 50, message = "Name must be between 2 and 50 characters")
    String name,

    @NotBlank(message = "Email is required")
    @Email(message = "Invalid email format")
    String email,

    @NotBlank(message = "Password is required")
    @Size(min = 8, message = "Password must be at least 8 characters")
    @Pattern(regexp = "^(?=.*[A-Za-z])(?=.*\\d).*$",
             message = "Password must contain letters and numbers")
    String password
) {}
```

---

## 8. 性能优化

### 8.1 缓存策略

```java
// ✅ 使用缓存注解
@Cacheable(value = "users", key = "#id")
public User getUserById(Long id) {
    return userRepository.findById(id).orElseThrow();
}

// ✅ 缓存更新
@CachePut(value = "users", key = "#user.id")
public User updateUser(User user) {
    return userRepository.save(user);
}

// ✅ 缓存清除
@CacheEvict(value = "users", key = "#id")
public void deleteUser(Long id) {
    userRepository.deleteById(id);
}
```

### 8.2 响应压缩

```java
// ✅ 启用响应压缩
// application.yml
server:
  compression:
    enabled: true
    mime-types: application/json,application/xml,text/html
```

### 8.3 分页与字段选择

```java
// ✅ 分页默认值
@GetMapping("/users")
public ResponseEntity<Page<User>> getUsers(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "20") int size,
        @RequestParam(defaultValue = "50") int maxSize) {

    // 限制最大分页大小
    size = Math.min(size, maxSize);
    return ResponseEntity.ok(userService.findAll(PageRequest.of(page, size)));
}

// ✅ 字段选择（Projection）
public interface UserSummary {
    Long getId();
    String getName();
    String getEmail();
}

@GetMapping("/users")
public List<UserSummary> getUsersSummary() {
    return userRepository.findAllProjectedBy(UserSummary.class);
}
```

---

## 9. API 文档规范

### 9.1 OpenAPI 规范示例

```java
// ✅ 使用 OpenAPI 注解
@RestController
@RequestMapping("/api/v1/users")
@Tag(name = "User Management", description = "User CRUD operations")
public class UserController {

    @Operation(summary = "Create a new user", description = "Creates a new user and returns the created resource")
    @ApiResponse(responseCode = "201", description = "User created successfully")
    @ApiResponse(responseCode = "400", description = "Invalid request body")
    @ApiResponse(responseCode = "409", description = "User already exists")
    @PostMapping
    public ResponseEntity<UserResponse> createUser(
            @Valid @RequestBody CreateUserRequest request) {
        User user = userService.createUser(request);
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(UserResponse.from(user));
    }

    @Operation(summary = "Get user by ID", description = "Returns a single user")
    @ApiResponse(responseCode = "200", description = "User found")
    @ApiResponse(responseCode = "404", description = "User not found")
    @GetMapping("/{id}")
    public ResponseEntity<UserResponse> getUser(@PathVariable Long id) {
        User user = userService.findById(id);
        return ResponseEntity.ok(UserResponse.from(user));
    }
}
```

### 9.2 文档访问

```yaml
# SpringDoc OpenAPI 配置
springdoc:
  api-docs:
    path: /api-docs
  swagger-ui:
    path: /swagger-ui.html
    operationsSorter: method
    tagsSorter: alpha
```

---

## 10. Java 特定实现标准

### 10.1 Controller 规范

```java
// ✅ Controller 最佳实践
@RestController
@RequestMapping("/api/v1/users")
@RequiredArgsConstructor
public class UserController {

    private final UserService userService;
    private final UserMapper userMapper;

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public UserResponse create(@Valid @RequestBody CreateUserRequest request) {
        User user = userMapper.toEntity(request);
        User saved = userService.create(user);
        return userMapper.toResponse(saved);
    }

    @GetMapping("/{id}")
    public UserResponse getById(@PathVariable Long id) {
        User user = userService.findByIdOrThrow(id);
        return userMapper.toResponse(user);
    }

    @GetMapping
    public Page<UserResponse> getAll(Pageable pageable) {
        return userService.findAll(pageable).map(userMapper::toResponse);
    }

    @PutMapping("/{id}")
    public UserResponse update(
            @PathVariable Long id,
            @Valid @RequestBody UpdateUserRequest request) {
        User user = userService.findByIdOrThrow(id);
        userMapper.updateEntity(request, user);
        User updated = userService.update(user);
        return userMapper.toResponse(updated);
    }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void delete(@PathVariable Long id) {
        userService.delete(id);
    }
}
```

### 10.2 Service 层规范

```java
// ✅ Service 层最佳实践
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class UserService {

    private final UserRepository userRepository;
    private final EmailService emailService;
    private final CacheService cacheService;

    @Transactional
    public User create(CreateUserRequest request) {
        // 业务验证
        validateUniqueEmail(request.email());

        // 创建实体
        User user = User.builder()
            .name(request.name())
            .email(request.email())
            .status(UserStatus.ACTIVE)
            .build();

        // 保存
        User saved = userRepository.save(user);

        // 异步发送邮件
        emailService.sendWelcomeEmail(saved.getEmail());

        return saved;
    }

    public User findByIdOrThrow(Long id) {
        return userRepository.findById(id)
            .orElseThrow(() -> new UserNotFoundException(id));
    }

    @Cacheable(value = "users", key = "#id")
    public User findById(Long id) {
        return findByIdOrThrow(id);
    }

    @Transactional
    public User update(Long id, UpdateUserRequest request) {
        User user = findByIdOrThrow(id);

        if (request.name() != null) {
            user.setName(request.name());
        }
        if (request.email() != null && !request.email().equals(user.getEmail())) {
            validateUniqueEmail(request.email());
            user.setEmail(request.email());
        }

        return userRepository.save(user);
    }

    @Transactional
    public void delete(Long id) {
        User user = findByIdOrThrow(id);
        user.setStatus(UserStatus.DELETED);
        userRepository.save(user);
    }

    private void validateUniqueEmail(String email) {
        if (userRepository.existsByEmail(email)) {
            throw new EmailAlreadyExistsException(email);
        }
    }
}
```

### 10.3 Repository 规范

```java
// ✅ Repository 最佳实践
public interface UserRepository extends JpaRepository<User, Long> {

    // 方法命名查询（简单场景）
    Optional<User> findByEmail(String email);

    boolean existsByEmail(String email);

    List<User> findByStatus(UserStatus status);

    Page<User> findByStatus(UserStatus status, Pageable pageable);

    // 复杂查询使用 @Query
    @Query("SELECT u FROM User u WHERE u.status = :status " +
           "AND u.createdAt >= :startDate " +
           "ORDER BY u.createdAt DESC")
    List<User> findActiveUsersAfter(
        @Param("status") UserStatus status,
        @Param("startDate") LocalDateTime startDate,
        Pageable pageable
    );

    // 原生查询（需要时）
    @Query(value = "SELECT * FROM users WHERE status = :status " +
           "AND DATE(created_at) = :date", nativeQuery = true)
    List<User> findByStatusAndDate(
        @Param("status") String status,
        @Param("date") LocalDate date
    );

    // Specification 查询（动态查询）
    default Specification<User> withFilters(UserFilter filter) {
        return (root, query, cb) -> {
            List<Predicate> predicates = new ArrayList<>();

            if (filter.name() != null) {
                predicates.add(cb.like(root.get("name"), "%" + filter.name() + "%"));
            }
            if (filter.status() != null) {
                predicates.add(cb.equal(root.get("status"), filter.status()));
            }
            if (filter.startDate() != null) {
                predicates.add(cb.greaterThanOrEqualTo(root.get("createdAt"), filter.startDate()));
            }

            return cb.and(predicates.toArray(new Predicate[0]));
        };
    }
}
```

### 10.4 DTO/Record 使用规范

```java
// ✅ 使用 Java Record 定义 DTO（Java 16+）

// 请求 DTO
public record CreateUserRequest(
    @NotBlank
    @Size(min = 2, max = 50)
    String name,

    @NotBlank
    @Email
    String email,

    @NotBlank
    @Size(min = 8)
    String password
) {}

// 响应 DTO
public record UserResponse(
    Long id,
    String name,
    String email,
    UserStatus status,
    LocalDateTime createdAt,
    LocalDateTime updatedAt
) {
    // 静态工厂方法
    public static UserResponse from(User user) {
        return new UserResponse(
            user.getId(),
            user.getName(),
            user.getEmail(),
            user.getStatus(),
            user.getCreatedAt(),
            user.getUpdatedAt()
        );
    }
}

// 列表响应 DTO
public record PagedResponse<T>(
    List<T> content,
    int page,
    int size,
    long totalElements,
    int totalPages,
    boolean first,
    boolean last
) {
    public static <T> PagedResponse of(Page<T> page) {
        return new PagedResponse<>(
            page.getContent(),
            page.getNumber(),
            page.getSize(),
            page.getTotalElements(),
            page.getTotalPages(),
            page.isFirst(),
            page.isLast()
        );
    }
}
```

---

## 附录：错误码规范

### 错误码命名规则

```
{模块}_{错误类型}_{序号}

例如：
USER_NOT_FOUND_001
USER_EMAIL_DUPLICATE_002
ORDER_PAYMENT_FAILED_001
```

### 常用错误码模板

| 模块 | 错误类型 | 说明 |
|------|----------|------|
| USER | NOT_FOUND | 用户不存在 |
| USER | EMAIL_DUPLICATED | 邮箱已注册 |
| USER | INVALID_PASSWORD | 密码错误 |
| USER | ACCOUNT_DISABLED | 账户已禁用 |
| ORDER | NOT_FOUND | 订单不存在 |
| ORDER | PAYMENT_FAILED | 支付失败 |
| ORDER | CANCELLED | 订单已取消 |
| COMMON | INTERNAL_ERROR | 内部错误 |
| COMMON | VALIDATION_ERROR | 验证错误 |
| COMMON | RATE_LIMIT_EXCEEDED | 超出速率限制 |

---

## 总结

| 分类 | 核心要点 |
|------|----------|
| URL 设计 | 资源导向、名词复数、层次清晰 |
| HTTP 方法 | 正确使用 GET/POST/PUT/PATCH/DELETE |
| 响应格式 | 统一包装、状态码正确 |
| 错误处理 | 统一格式、具体错误码 |
| 版本控制 | URL 路径方式 |
| 安全性 | 认证、验证、加密 |
| 性能 | 缓存、分页、压缩 |
| 文档 | OpenAPI 规范 |

---

## 参考资料

- [RESTful Web API Design - Microsoft](https://learn.microsoft.com/en-us/azure/architecture/best-practices/api-design)
- [OpenAPI Specification](https://swagger.io/specification/)
- [Spring REST Guidelines](https://spring.io/projects/spring-restdocs)
- [RFC 7231 - HTTP/1.1 Semantics and Content](https://tools.ietf.org/html/rfc7231)